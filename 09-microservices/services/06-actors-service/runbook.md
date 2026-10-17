# Runbook — actors-service

> Estado: 🟢 Estable | Última actualización: 2026-06-20

Servicio fuente de verdad sobre instructores, aprendices y empresas del proceso formativo SENA. Gestiona contratos, competencias, disponibilidad, etapas productivas y visitas empresariales.

---

## Healthcheck

| Endpoint | Respuesta esperada | SLO |
|----------|--------------------|-----|
| `GET /health` | `200 { "status": "ok" }` | < 200 ms |
| `GET /health/db` | `200` cuando `actors_db` (PostgreSQL) acepta conexiones | < 500 ms |

**Verificación manual:**

```bash
curl -sf http://actors-service:8006/health | jq .
curl -sf http://actors-service:8006/health/db | jq .
```

**SLO de disponibilidad:** 99% mensual (máximo ~7.3 h de downtime/mes).

**Nota operativa:** `scheduling-service` consulta instructores síncronamente al crear sesiones. Una degradación de este servicio bloquea la creación de horarios en tiempo real.

---

## Variables de entorno requeridas

| Variable | Descripción | Ejemplo |
|----------|-------------|---------|
| `DATABASE_URL` | Cadena de conexión PostgreSQL a `actors_db` | `postgresql://user:pass@host:5432/actors_db` |
| `DATABASE_POOL_MIN` | Conexiones mínimas en el pool | `2` |
| `DATABASE_POOL_MAX` | Conexiones máximas en el pool | `10` |
| `IAM_SERVICE_URL` | URL base de `iam-service` para validación de JWT | `http://iam-service:8001` |
| `ACADEMIC_MANAGEMENT_SERVICE_URL` | URL base de `academic-management-service` | `http://academic-management-service:8003` |
| `REFERENCE_DATA_SERVICE_URL` | URL base de `reference-data-service` | `http://reference-data-service:8002` |
| `KAFKA_BROKERS` | Lista de brokers Kafka | `kafka:9092` |
| `KAFKA_CONSUMER_GROUP` | Grupo de consumo para eventos entrantes | `actors-service-cg` |
| `KAFKA_TOPIC_PUBLISH` | Topic de publicación de eventos propios | `actors-events` |
| `KAFKA_TOPIC_IAM` | Topic de eventos de IAM a consumir | `iam-events` |
| `KAFKA_TOPIC_ACADEMIC` | Topic de eventos académicos a consumir | `academic-events` |
| `JWT_SECRET` | Clave de verificación de tokens JWT | (secreto) |
| `PORT` | Puerto de escucha del servicio | `8006` |
| `LOG_LEVEL` | Nivel de logging | `info` |
| `ALERT_WEBHOOK_URL` | Webhook para alertas operativas | `https://hooks.slack.com/...` |

**Variables no requeridas:** Redis no es utilizado por este servicio.

---

## Alertas críticas

| Alerta | Condición | Severidad | Acción inmediata |
|--------|-----------|-----------|------------------|
| BD no responde | Timeout en `actors_db` > 5 s o `/health/db` retorna 5xx | P0 | Ver [Falla de base de datos](#falla-de-base-de-datos) |
| Instructores no encontrados masivamente | Error 404 masivo desde `scheduling-service` al consultar instructores activos | P1 | Verificar integridad: `SELECT COUNT(*) FROM instructor WHERE is_active = true` |
| Mensaje en DLQ de actors-events | Mensaje en `actors-events.dlq` sin procesar > 5 min | P1 | Revisar DLQ; ver [Mensajes en Dead Letter Queue](#mensajes-en-dead-letter-queue) |
| Desincronización con iam-service | Evento `iam.user.created` no procesado; actor sin registro en actors-service | P1 | Verificar consumer group `actors-service-cg` en Kafka |
| Competencias marcadas `requires_review` en masa | Lote de `competency_assignment.requires_review = true` sin atender > 24 h | P2 | Notificar coordinadores para revisión manual |
| Latencia alta en `/instructors/available` | P95 > 300 ms | P2 | Verificar índice `competency_id WHERE is_active = true`; ver [Consulta de disponibilidad lenta](#consulta-de-disponibilidad-lenta) |
| Servicio no responde | `/health` retorna 5xx o sin respuesta > 1 min | P0 | Reiniciar pod/contenedor; revisar logs de arranque |

---

## Procedimientos comunes

### Transferir manualmente un aprendiz a una ficha diferente

Usar cuando el aprendiz fue matriculado en la ficha incorrecta o cuando se aprueba un traslado formal.

**Precondición:** Confirmar que existe la ficha destino en `academic-management-service`.

```sql
-- 1. Identificar el registro de inscripción activo del aprendiz
SELECT id, learner_id, ficha_id, stage, enrollment_date, is_current
FROM learner_ficha_enrollment
WHERE learner_id = '<LEARNER_UUID>'
  AND is_current = true;

-- 2. Cerrar la inscripción actual (dentro de una transacción)
BEGIN;

UPDATE learner_ficha_enrollment
SET
    is_current      = false,
    completion_date = CURRENT_DATE,
    transfer_reason = 'Traslado manual por operaciones — <MOTIVO>'
WHERE learner_id   = '<LEARNER_UUID>'
  AND is_current   = true;

-- 3. Crear la nueva inscripción en la ficha destino
INSERT INTO learner_ficha_enrollment
    (id, learner_id, ficha_id, stage, enrollment_date, is_current, transfer_reason, created_at)
VALUES
    (gen_random_uuid(),
     '<LEARNER_UUID>',
     '<FICHA_DESTINO_UUID>',
     'LECTURE',        -- ajustar a INDUCTION o PRODUCTIVE según corresponda
     CURRENT_DATE,
     true,
     'Traslado desde ficha <FICHA_ORIGEN_UUID>',
     NOW());

-- 4. Registrar en bitácora de actividad
INSERT INTO activity_log
    (id, actor_type, actor_id, event_type, description, previous_value, new_value, recorded_by, recorded_at)
VALUES
    (gen_random_uuid(),
     'LEARNER',
     '<LEARNER_UUID>',
     'FICHA_TRANSFERRED',
     'Traslado manual de ficha por operaciones',
     '<FICHA_ORIGEN_UUID>',
     '<FICHA_DESTINO_UUID>',
     NULL,   -- NULL = evento del sistema; reemplazar por UUID del operador si disponible
     NOW());

COMMIT;
```

**Verificar resultado:**

```sql
SELECT id, ficha_id, stage, enrollment_date, is_current
FROM learner_ficha_enrollment
WHERE learner_id = '<LEARNER_UUID>'
ORDER BY created_at DESC
LIMIT 3;
```

**Post-acción:** Notificar a `scheduling-service` si el aprendiz tenía sesiones programadas en la ficha anterior. Publicar el evento `actors.learner.enrolled` manualmente si el sistema no lo dispara automáticamente.

---

### Reactivar un instructor desactivado

Usar cuando un instructor fue marcado inactivo por error o cuando regresa al servicio activo tras una ausencia.

**Precondición:** Verificar con el área de talento humano que la reactivación está autorizada y que existe un contrato vigente o nuevo contrato a crear.

```sql
-- 1. Verificar estado actual del instructor
SELECT id, full_name, is_active, updated_at
FROM instructor
WHERE id = '<INSTRUCTOR_UUID>';

-- 2. Verificar contratos existentes
SELECT id, contract_type, start_date, end_date, is_current
FROM instructor_contract
WHERE instructor_id = '<INSTRUCTOR_UUID>'
ORDER BY start_date DESC;

-- 3. Reactivar instructor y marcar contrato como vigente (transacción)
BEGIN;

UPDATE instructor
SET
    is_active  = true,
    updated_at = NOW()
WHERE id = '<INSTRUCTOR_UUID>';

-- Si el contrato anterior sigue vigente, reactivarlo
UPDATE instructor_contract
SET is_current = true
WHERE instructor_id = '<INSTRUCTOR_UUID>'
  AND id = '<CONTRACT_UUID>';
-- Si se debe crear un contrato nuevo, usar INSERT en lugar del UPDATE anterior.

-- Reactivar asignaciones de competencia que quedaron en SUSPENDIDA
-- (las asignaciones se suspenden automáticamente al procesar iam.user.deactivated)
UPDATE competency_assignment
SET is_active = true
WHERE instructor_id = '<INSTRUCTOR_UUID>'
  AND is_active = false;
-- ATENCIÓN: revisar que solo se reactiven las competencias válidas; excluir las revocadas.

-- Registrar en bitácora
INSERT INTO activity_log
    (id, actor_type, actor_id, event_type, description, previous_value, new_value, recorded_by, recorded_at)
VALUES
    (gen_random_uuid(),
     'INSTRUCTOR',
     '<INSTRUCTOR_UUID>',
     'STATUS_CHANGED',
     'Reactivación manual por operaciones',
     'is_active=false',
     'is_active=true',
     NULL,
     NOW());

COMMIT;
```

**Verificar resultado:**

```sql
SELECT i.id, i.full_name, i.is_active,
       COUNT(ca.id) FILTER (WHERE ca.is_active = true) AS competencias_activas
FROM instructor i
LEFT JOIN competency_assignment ca ON ca.instructor_id = i.id
WHERE i.id = '<INSTRUCTOR_UUID>'
GROUP BY i.id, i.full_name, i.is_active;
```

**Post-acción:** Confirmar con `iam-service` que el usuario correspondiente también fue reactivado (`is_active = true` en `iam-service`). Si no, la reactivación en actors-service no tendrá efecto funcional porque el JWT seguirá siendo rechazado.

---

## Escenarios de falla

### Falla de base de datos

**Síntoma:** `/health/db` retorna 503; todas las peticiones al API retornan 500; logs muestran `connection timeout` o `too many connections`.

**Diagnóstico:**

```bash
# Verificar conectividad desde el pod del servicio
psql "$DATABASE_URL" -c "SELECT 1;"

# Revisar conexiones activas en la BD
psql "$DATABASE_URL" -c "
  SELECT count(*), state
  FROM pg_stat_activity
  WHERE datname = 'actors_db'
  GROUP BY state;"
```

**Acciones:**

1. Si el problema es agotamiento de conexiones (`too many connections`): reducir `DATABASE_POOL_MAX` y reiniciar el servicio. Verificar si hay conexiones zombies con `pg_terminate_backend`.
2. Si PostgreSQL no responde: escalar a DBA de guardia. El servicio puede reiniciarse pero no recuperará hasta que la BD esté disponible.
3. Si la BD está disponible pero el servicio no conecta: verificar `DATABASE_URL` en las variables de entorno del pod.

---

### Mensajes en Dead Letter Queue

**Síntoma:** Alerta de mensaje en `actors-events.dlq`; actor no fue creado tras evento `iam.user.created`; o cambio de estado de competencia no se reflejó.

**Diagnóstico:**

```bash
# Leer mensaje(s) en la DLQ (con kafka-console-consumer o kafkacat)
kafka-console-consumer \
  --bootstrap-server "$KAFKA_BROKERS" \
  --topic actors-events.dlq \
  --from-beginning \
  --max-messages 10
```

**Acciones:**

1. Identificar el `event_type` y `event_id` del mensaje fallido.
2. Determinar la causa: payload malformado, BD no disponible en el momento del procesamiento, referencia externa inválida.
3. Si fue un fallo transitorio (BD no disponible): republicar el mensaje al topic original desde la DLQ. La lógica de idempotencia usa `event_id` para evitar duplicados.
4. Si el payload es inválido: corregir manualmente el estado en la BD según el procedimiento correspondiente y descartar el mensaje de la DLQ.

---

### Consulta de disponibilidad lenta

**Síntoma:** `GET /instructors/available?competency_id=<UUID>` supera 300 ms P95; scheduling-service reporta timeouts al crear sesiones.

**Diagnóstico:**

```sql
-- Verificar que el índice parcial está siendo usado
EXPLAIN ANALYZE
SELECT i.id, i.full_name
FROM instructor i
JOIN competency_assignment ca ON ca.instructor_id = i.id
WHERE ca.competency_id = '<COMPETENCY_UUID>'
  AND ca.is_active = true
  AND i.is_active = true;
```

**Acciones:**

1. Si el índice no está siendo usado: ejecutar `ANALYZE competency_assignment;` y `ANALYZE instructor;`.
2. Si el índice está corrompido: `REINDEX INDEX CONCURRENTLY idx_competency_assignment_active;`
3. Si la tabla tiene millones de filas sin vaciar: ejecutar `VACUUM ANALYZE competency_assignment;`

---

### Desincronización de actor con iam-service

**Síntoma:** Usuario existe en `iam-service` pero no tiene registro en `instructor` ni en `learner`; el API retorna 404 al intentar acceder por `user_id`.

**Causa probable:** El evento `iam.user.created` no fue procesado (fallo transitorio, servicio caído al momento del evento, mensaje expiró en la DLQ).

**Acción:** Crear el registro de actor manualmente a partir de los datos disponibles en `iam-service`, luego registrar el evento en `activity_log` con `event_type = 'MANUAL_SYNC'`. No reinventar el `user_id`; usar exactamente el UUID que proviene de `iam-service`.

---

## Escalamiento

| Nivel | Condición para escalar | A quién |
|-------|------------------------|---------|
| **L1 — Operaciones** | Servicio caído; latencia alta; mensajes en DLQ; fallo en healthcheck | Equipo de guardia de plataforma |
| **L2 — Ingeniería backend** | Corrupción de datos en `instructor` o `learner`; fallo en migración de BD; comportamiento inesperado tras deploy | Equipo backend responsable del servicio |
| **L3 — DBA** | Problemas de rendimiento en PostgreSQL; corrupción de índices; necesidad de `pg_terminate_backend` masivo; fallo en replicación | DBA de guardia |
| **L4 — Dominio** | Decisiones sobre reglas de negocio: si se puede o no reactivar un instructor, si un traslado de ficha es válido, conflictos de competencias | Coordinador académico o área de talento humano |

**Canal de alertas:** Configurar `ALERT_WEBHOOK_URL` con el webhook del canal de operaciones en Slack/Teams.

**Retención de PII:** Datos de `instructor` y `learner` deben eliminarse 5 años después de su desvinculación o graduación. Coordinar con el equipo de privacidad antes de cualquier borrado masivo.
