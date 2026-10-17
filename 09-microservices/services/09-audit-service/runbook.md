# Runbook — audit-service

> Estado: 🟢 Estable | Última actualización: 2026-10-11

---

## Healthcheck

El `audit-worker` es un consumidor RabbitMQ; su estado se determina por la profundidad de su cola durable, la tasa de ack y el crecimiento de `audit.dlq`. La API de consulta de solo lectura es un componente independiente.

| Metrica | Herramienta | Umbral saludable | Umbral de alerta |
|---------|-------------|-----------------|-----------------|
| Profundidad de cola durable | RabbitMQ management | < 100 mensajes | > 500 mensajes |
| Tiempo desde el último ack | Logs/métricas del worker | < 60 s | > 5 min |
| Tasa de INSERT exitosos en `audit_record` | Metricas de BD | > 0 inserts/min (si hay trafico) | 0 inserts por > 5 min con cola no vacia |

**Verificación de cola (RabbitMQ):** inspeccionar la cola cuyo nombre es `BROKER_CONSUMER_GROUP` y `audit.dlq` en RabbitMQ Management. La profundidad debe mantenerse cercana a cero o decrecer cuando hay consumidor activo.

**Verificacion alternativa via BD:**

```bash
# Confirmar que se estan recibiendo eventos recientes
psql $DATABASE_URL -c "
  SELECT source_service, count(*) AS eventos_ultima_hora
  FROM audit_record
  WHERE received_at > NOW() - INTERVAL '1 hour'
  GROUP BY source_service
  ORDER BY eventos_ultima_hora DESC;
"
```

Si hay trafico en el sistema y esta consulta devuelve cero filas por un servicio activo, el worker esta caido o hay un problema de conectividad al broker.

---

## Variables de entorno requeridas

El `audit-worker` es el unico componente desplegable del servicio. Una variable faltante impide el arranque.

| Variable | Descripcion | Ejemplo |
|----------|-------------|---------|
| `DATABASE_URL` | Cadena de conexion a `audit_db` | `postgresql://user:pass@host:5432/audit_db` |
| `MESSAGE_BROKER_URL` | URL del broker de mensajes | `amqp://user:pass@broker:5672` |
| `AUDIT_EVENTS_EXCHANGE` | Exchange topic global | `audit-events` |
| `BROKER_TOPIC_PATTERN` | Binding wildcard AMQP | `#` |
| `BROKER_CONSUMER_GROUP` | Nombre de la cola durable compartida | `audit-worker` |
| `WORKER_MAX_RETRIES` | Reintentos antes de enviar a DLQ | `3` |
| `WORKER_RETRY_DELAY_MS` | TTL/backoff de `audit.retry` en ms | `5000` |
| `LOG_LEVEL` | Nivel de log (`debug`, `info`, `warn`, `error`) | `info` |
| `SERVICE_ENV` | Entorno de ejecucion | `production` |
| `DB_POOL_MAX_CONNECTIONS` | Tamano maximo del pool de conexiones | `10` |

> No hay variables de entorno para SMTP ni servicios externos. El `audit-service` no llama a ningun otro servicio. Si una variable adicional aparece en el entorno, es probablemente un error de configuracion.

---

## Alertas criticas

### P0 — Base de datos no responde

**Condicion:** timeout de conexion a `audit_db` > 5 s, o el worker no puede ejecutar INSERT.

**Impacto:** los eventos del sistema dejan de persistirse. La cola del broker acumula mensajes. No hay perdida de datos si el broker tiene retencion suficiente (verificar configuracion de retencion del broker).

**Accion inmediata:**
1. Verificar estado del contenedor/pod de PostgreSQL.
2. Intentar conexion directa: `psql $DATABASE_URL -c "SELECT 1"`.
3. Revisar espacio en disco de la BD (la tabla `audit_record` crece continuamente).
4. Revisar logs del worker — si reporta `connection refused` o `too many clients`, el problema es BD o red.
5. Si la BD esta sana, reiniciar el worker para forzar reconexion del pool.
6. Escalar con DBA si la BD no responde en 10 min.

---

### P1 — Consumer lag creciendo (eventos no siendo persistidos)

**Condicion:** el lag del consumer group `audit-worker` crece sostenidamente por mas de 5 min, o el worker no reporta actividad de commit de offset.

**Impacto:** los eventos de auditoria se acumulan en el broker sin persistirse. Dependiendo del TTL del broker, los mensajes mas antiguos pueden expirar antes de ser procesados si la caida es prolongada.

**Accion inmediata:**
1. Verificar que el pod del `audit-worker` este corriendo.
2. Revisar logs del worker por errores de conexion al broker o a la BD.
3. Si el worker esta caido, reiniciarlo — procesa el backlog automaticamente.
4. Confirmar que el broker tiene retencion suficiente para el backlog acumulado.
5. Monitorear que el lag empiece a bajar tras el reinicio.

```bash
# Verificar que el worker esta consumiendo activamente
# El LAG debe reducirse entre dos consultas consecutivas
kafka-consumer-groups.sh --bootstrap-server <broker>:9092 \
  --group audit-worker --describe

sleep 30

kafka-consumer-groups.sh --bootstrap-server <broker>:9092 \
  --group audit-worker --describe
```

---

### P1 — DLQ con mensajes acumulados

**Condicion:** la dead letter queue del `audit-worker` tiene mensajes > 0.

**Impacto:** eventos que no pudieron persistirse tras los reintentos configurados. Riesgo de gaps en el log de auditoria.

**Accion inmediata:**
1. Inspeccionar los mensajes de la DLQ para identificar la causa del fallo.
2. Si el fallo fue transitorio (BD temporalmente no disponible), reinyectar los mensajes (ver procedimiento abajo).
3. Si el fallo es por schema invalido o evento malformado, escalar al equipo de desarrollo del servicio fuente.

---

### P2 — Advertencia de espacio en disco (retencion 7 anos)

**Condicion:** el espacio disponible en el tablespace de `audit_db` cae por debajo del 20%, o el crecimiento proyectado supera la capacidad en < 90 dias.

**Impacto:** si el disco se llena, la BD no puede recibir nuevos INSERT y el `audit-worker` falla con errores de escritura, desencadenando la alerta P0.

**Accion inmediata:**
1. Verificar espacio disponible:

```sql
-- Tamano de la tabla principal y sus particiones
SELECT
  relname AS tabla,
  pg_size_pretty(pg_total_relation_size(oid)) AS tamano_total,
  pg_size_pretty(pg_relation_size(oid)) AS tamano_datos
FROM pg_class
WHERE relname LIKE 'audit_record%'
ORDER BY pg_total_relation_size(oid) DESC;
```

2. Verificar si las particiones de mas de 2 anos han sido movidas a cold storage segun el calendario de archivado.
3. Si no hay archivado automatico configurado, iniciar el proceso de archivado manual (ver `runbooks/audit-cold-restore.md`).
4. Escalar con DBA e infraestructura para expansion de disco si el archivado no libera espacio suficiente a tiempo.

---

## Procedimientos comunes

### Verificar gaps en la secuencia de eventos (eventos faltantes)

El `audit_record` no tiene una secuencia numerica propia, pero los `event_id` son UUID v4 generados por el servicio origen. La forma de detectar gaps es verificar que todos los eventos de un tipo especifico en un rango temporal esten presentes.

**Metodo 1 — Verificar cobertura por servicio en un rango de tiempo:**

```sql
-- Eventos recibidos por servicio en el ultimo dia
SELECT
  source_service,
  event_type,
  count(*) AS total_eventos,
  min(event_occurred_at) AS primer_evento,
  max(event_occurred_at) AS ultimo_evento
FROM audit_record
WHERE received_at BETWEEN '<fecha_inicio>' AND '<fecha_fin>'
GROUP BY source_service, event_type
ORDER BY source_service, event_type;
```

**Metodo 2 — Detectar ventanas de tiempo sin actividad para un servicio especifico:**

```sql
-- Ventanas de mas de 10 minutos sin eventos de un servicio durante horario laboral
SELECT
  date_trunc('hour', received_at) AS hora,
  count(*) AS eventos
FROM audit_record
WHERE source_service = '<nombre-servicio>'
  AND received_at BETWEEN '<fecha_inicio>' AND '<fecha_fin>'
GROUP BY hora
ORDER BY hora;
-- Una hora con 0 eventos en horario laboral es sospechosa
```

**Metodo 3 — Correlacionar con eventos del servicio origen:**

Si se sospecha que un evento especifico no fue auditado, buscar por `event_id` o por `correlation_id`:

```sql
SELECT id, event_id, event_type, source_service, actor_id,
       event_occurred_at, received_at
FROM audit_record
WHERE event_id = '<event_uuid>'
   OR payload->>'correlation_id' = '<correlation_uuid>';
```

Si el evento no aparece, puede ser que:
- El `audit-worker` estaba caido cuando se publico (verificar lag historico del broker).
- El evento fue a la DLQ y no se reinyecto.
- El servicio origen no publico el evento por un bug.

---

### Consultar el log de auditoria por actor

Usado para investigaciones de seguridad, auditorias regulatorias o soporte.

```sql
-- Todas las acciones de un actor en un rango de fechas
SELECT
  ar.event_type,
  ar.source_service,
  ar.entity_type,
  ar.entity_id,
  ar.event_occurred_at,
  ar.received_at,
  ar.payload
FROM audit_record ar
WHERE ar.actor_id = '<actor_uuid>'
  AND ar.event_occurred_at BETWEEN '<fecha_inicio>' AND '<fecha_fin>'
ORDER BY ar.event_occurred_at ASC;
```

```sql
-- Historial de cambios de una entidad especifica
SELECT
  ar.event_type,
  ar.source_service,
  ar.actor_id,
  ar.event_occurred_at,
  ar.payload
FROM audit_record ar
WHERE ar.entity_type = '<tipo>'   -- ej: 'ClassSession', 'Learner'
  AND ar.entity_id = '<entity_uuid>'
ORDER BY ar.event_occurred_at ASC;
```

> Estas consultas pueden ser lentas si el rango de fechas es amplio. Usar los indices disponibles `(actor_id, received_at)` y `entity_id` asegura planes de ejecucion eficientes.

---

### Reinyectar eventos desde la DLQ

Usado cuando mensajes fallidos deben recuperarse despues de que la causa raiz fue resuelta.

```bash
# RabbitMQ / AMQP — reinyectar mensajes de audit.dlq al exchange global
# IMPORTANTE: verificar primero que la causa raiz (BD, schema) esta resuelta
rabbitmqadmin get queue=audit.dlq count=500 requeue=false
rabbitmqadmin publish exchange=audit-events routing_key=<routing-key-origen> \
  payload='<mensaje_original>' properties='{"content_type":"application/json"}'
```

> El `audit-worker` es idempotente: `INSERT INTO audit_record ON CONFLICT (event_id) DO NOTHING`. Reinyectar eventos ya persistidos no genera duplicados.

Verificar resultado:

```sql
-- Confirmar que los event_ids esperados estan en audit_record
SELECT event_id, event_type, received_at
FROM audit_record
WHERE event_id IN ('<uuid1>', '<uuid2>', '<uuid3>');
```

---

### Archivar particiones antiguas a cold storage

El proceso de archivado libera espacio en disco manteniendo la retencion de 7 anos. Las particiones de mas de 2 anos se mueven a cold storage.

Ver procedimiento detallado en: `runbooks/audit-cold-restore.md` (pendiente de crear).

Resumen del proceso:
1. Identificar particiones con `received_at` de mas de 2 anos.
2. Exportar la particion a formato Parquet o dump SQL comprimido.
3. Subir al cold storage (S3 Glacier / Azure Archive).
4. Verificar integridad del archivo en cold storage.
5. Detach de la particion en PostgreSQL (sin DROP — la retencion legal es 7 anos).

> **Regla critica:** no ejecutar DROP PARTITION antes de que la particion tenga 7 anos de antiguedad. La normativa SENA requiere 7 anos de retencion del log de auditoria.

---

## Escenarios de falla

### Escenario 1 — audit-worker caido; broker acumula mensajes

**Sintoma:** lag del consumer group `audit-worker` crece; no hay INSERT en `audit_record` durante el tiempo de caida.

**Comportamiento esperado:** el broker retiene los mensajes. Ningun evento se pierde mientras el broker tenga suficiente retencion configurada.

**Recuperacion:**
1. Reiniciar el `audit-worker`.
2. El worker procesa el backlog desde el ultimo offset confirmado.
3. Monitorear que el lag baje a cero.
4. Verificar que los eventos del periodo de caida esten en `audit_record`.

**Tiempo esperado de recuperacion:** depende del tamano del backlog. Un worker sano procesa varios miles de eventos por minuto. Para backlog < 10,000 eventos: < 5 min.

**Riesgo de perdida de datos:** ninguno, si el broker tiene retencion > tiempo de caida del worker.

---

### Escenario 2 — Disco de audit_db lleno

**Sintoma:** el `audit-worker` reporta errores de INSERT (`ERROR: could not extend file`); lag crece; todos los servicios pueden verse afectados si comparten infraestructura de BD.

**Recuperacion:**
1. Accion inmediata: liberar espacio ejecutando archivado de particiones antiguas (ver procedimiento arriba).
2. Si no hay particiones archivables de forma inmediata, escalar con infraestructura para expansion de disco.
3. Una vez restaurado el espacio, el worker reinicia el procesamiento desde el ultimo offset confirmado.
4. Investigar por que el proceso de archivado automatico no previno la situacion.

---

### Escenario 3 — Evento con schema invalido llega al worker

**Sintoma:** el worker reporta errores de deserializacion; el mensaje se mueve a la DLQ despues de los reintentos.

**Comportamiento esperado:** el worker no debe fallar completamente por un mensaje malformado. Debe mover el mensaje a la DLQ y continuar con el siguiente.

**Accion:**
1. Inspeccionar el mensaje en la DLQ para identificar el servicio origen y el tipo de evento.
2. Reportar al equipo de desarrollo del servicio origen para corregir el schema.
3. Si el mensaje contiene datos validos con formato incorrecto, puede procesarse manualmente:

```sql
-- Insercion manual de un evento con schema corregido
INSERT INTO audit_record (id, event_id, event_type, source_service,
  actor_id, entity_type, entity_id, payload, event_occurred_at, received_at)
VALUES (
  gen_random_uuid(),
  '<event_id_original>',
  '<event_type>',
  '<source_service>',
  '<actor_id_o_null>',
  '<entity_type>',
  '<entity_id>',
  '<payload_json>'::jsonb,
  '<event_occurred_at>',
  NOW()
)
ON CONFLICT (event_id) DO NOTHING;
```

---

### Escenario 4 — Necesidad de restaurar eventos de cold storage para auditoria legal

**Sintoma / Solicitud:** equipo legal o auditor externo requiere eventos de mas de 2 anos de antiguedad.

**Proceso:**
1. Identificar el rango de fechas y servicio fuente requerido.
2. Localizar la particion en cold storage correspondiente.
3. Restaurar la particion a un tablespace temporal o exportar los datos directamente desde cold storage.
4. Ejecutar las consultas de auditoria sobre los datos restaurados.
5. Documentar el acceso en el registro de accesos a auditoria.

Ver: `runbooks/audit-cold-restore.md` para instrucciones detalladas de restauracion.

---

## Escalamiento

| Nivel | Condicion | Responsable | Accion |
|-------|-----------|-------------|--------|
| L1 — Operaciones | Consumer lag creciendo, worker caido | On-call de turno | Reiniciar worker; verificar lag; ejecutar procedimientos de este runbook |
| L2 — Infraestructura | BD no responde, disco lleno, broker inestable | Equipo de infraestructura | Diagnóstico de capa de datos, expansion de disco, archivado de particiones |
| L3 — Desarrollo | Mensajes en DLQ por schema invalido, bug de idempotencia, duplicados en audit_record | Equipo de desarrollo del servicio fuente | Correccion de schema del evento publicado; no se modifica audit-service |
| L4 — Legal / Cumplimiento | Solicitud de auditoria legal, investigacion de seguridad, retencion en riesgo | Equipo legal + DBA | Proceso formal de acceso a datos de auditoria con registro de accesos |

**SLO del servicio:** 99% de disponibilidad mensual del `audit-worker` (capacidad de consumir eventos). Budget de error: 7.2 h/mes.

**Retencion legal:** 7 anos desde la fecha de emision del evento (`event_occurred_at`). Ninguna fila puede eliminarse antes de cumplir este periodo. Cualquier solicitud de borrado anticipado requiere aprobacion del equipo legal.

**Contactos de escalamiento:** definidos en el directorio interno del equipo (pendiente de completar con nombres y canales).
