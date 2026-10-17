# Patrones de comunicación

> Estado: 🟡 En progreso | Última actualización: 2026-08-26
> Autor: Jesús Ariel González Bonilla (PM + Arquitecto) | Equipo: Arquitectura

Define cómo se comunican los 10 microservicios backend entre sí: cuándo usar comunicación
**síncrona** (petición/respuesta) y cuándo **asíncrona** (eventos), y qué mecanismos de
resiliencia acompañan a cada estilo.

> **Nota de alcance:** los servicios usan stacks polyglot según ADR-006, pero estas reglas de comunicación son agnósticas de lenguaje. RabbitMQ/AMQP es el transporte asíncrono y los contratos públicos se gobiernan desde `shared-contracts`.

## Principio rector

> Síncrono solo cuando el llamador **necesita la respuesta para continuar**; en cualquier otro
> caso, evento asíncrono.

La comunicación síncrona crea **acoplamiento de disponibilidad**: si el servicio llamado está
caído, el llamador falla. La asíncrona desacopla el tiempo y la disponibilidad, a costa de
**consistencia eventual**. La elección se rige por las [reglas de frontera](./service-boundary-rules.md)
(R05–R08).

## Cómo elegir

| Pregunta | Si la respuesta es… | Patrón |
|----------|--------------------|--------|
| ¿El llamador necesita el resultado para responder al usuario ahora? | Sí | **Síncrono (REST)** |
| ¿Es una validación puntual sobre datos que pertenecen a otro servicio? | Sí | **Síncrono (REST)**, contando para el límite de 2 (R06) |
| ¿Es notificar que algo ocurrió (una mutación ya confirmada)? | Sí | **Asíncrono (evento)** |
| ¿El consumidor puede procesarlo con segundos de retraso sin romper el negocio? | Sí | **Asíncrono (evento)** |
| ¿Se consulta un dato de otro servicio con muy alta frecuencia? | Sí | **Read model local** poblado por eventos (ver [ADR-002](../05-architecture/decisions/records/ADR-002-scheduling-read-models.md)) |

## Síncrona (REST)

Comunicación petición/respuesta sobre **HTTP/REST**. Se usa para operaciones que requieren
respuesta inmediata y para validaciones puntuales contra el servicio dueño de un dato.

**Cuándo usarla**

- Autenticación/autorización: todos los servicios validan el JWT contra `iam-service`
  (dependencia `auth`, no cuenta para el límite de 2 — ver [dependency-map.md](./dependency-map.md)).
- Validaciones que deben ser frescas en el momento exacto: p. ej. `scheduling-service` valida
  la ficha y sus competencias contra `academic-management-service` al crear el borrador, y
  reverifica el estado fresco de instructores/ambientes justo antes de `publish`
  (ver [ADR-002](../05-architecture/decisions/records/ADR-002-scheduling-read-models.md)).
- Lecturas de catálogos que el llamador necesita resolver en línea.

**Reglas de diseño (agnósticas de lenguaje)**

| Regla | Detalle |
|-------|---------|
| Contrato explícito | Cada endpoint tiene contrato versionado (`contract.md` del componente). Compatibilidad hacia atrás ≥ 1 versión (R11). |
| Límite de cadena | Una petición no encadena más de **2** llamadas síncronas aguas abajo (R06, excluyendo `auth` a IAM). Ver antipatrón "sync en cascada" en [service-boundary-rules.md](./service-boundary-rules.md). |
| Sin acceso a BD ajena | Nunca se consulta la BD de otro servicio; solo su API pública (R02). |
| Errores estándar | Respuesta de error uniforme `{ error_code, message, trace_id }`. |
| Idempotencia de escritura | Las operaciones de escritura expuestas aceptan una clave de idempotencia cuando el reintento es posible. |
| Propagación de trazas | Se propaga `trace_id`/`correlation_id` en cada salto para trazabilidad extremo a extremo. |

### Resiliencia en llamadas síncronas

Toda llamada REST saliente se protege con tres mecanismos combinados. Los valores son
**objetivos de diseño**; se afinan por endpoint al construir el servicio.

| Mecanismo | Propósito | Objetivo de diseño |
|-----------|-----------|--------------------|
| **Timeout** | Acota cuánto se espera; evita que un servicio lento bloquee al llamador. | Timeout de conexión y de lectura explícitos por dependencia (p. ej. objetivo global de validación de horario < 2 s ⇒ timeouts individuales en cientos de ms). |
| **Retry** | Reintenta fallos **transitorios** (timeout, 503, reset). | Reintentos limitados (p. ej. 2–3) **solo** sobre operaciones idempotentes, con **backoff exponencial + jitter**. Nunca reintentar errores 4xx de negocio. |
| **Circuit breaker** | Corta las llamadas cuando el destino falla de forma sostenida; evita el fallo en cascada. | Se abre al superar un umbral de fallos; en estado abierto responde rápido con fallback o error controlado; pasa a *half-open* para probar recuperación. |

**Degradación / fallback.** Cuando una dependencia síncrona no crítica no responde, el servicio
degrada de forma controlada en vez de fallar por completo (p. ej. usar el read model local como
respaldo — ver [ADR-002](../05-architecture/decisions/records/ADR-002-scheduling-read-models.md)).
Las validaciones **críticas** (las que preceden a `publish`) no admiten degradación: si no se
puede verificar, la operación se rechaza.

## Asíncrona (eventos)

Comunicación por **publicación/consumo de eventos** a través del broker AMQP (RabbitMQ,
[ADR-001](../05-architecture/decisions/records/ADR-001-message-broker.md)). Un servicio publica
un evento cuando ocurre una mutación relevante; los interesados lo consumen sin que el
publicador los conozca ni los espere.

**Cuándo usarla**

- Propagar mutaciones que no requieren respuesta inmediata (R07): auditoría, seguimiento,
  notificaciones.
- Poblar read models locales (ADR-002).
- Fan-out: `audit-service` consume **todos** los topics; `monitoring-service` consume los
  eventos de negocio relevantes (ver [event-catalog.md](./event-catalog.md)).

### Topología en el broker

- Un **topic exchange** por dominio de eventos (`scheduling-events`, `academic-events`, …).
- `audit-service` se suscribe con routing key `#` (wildcard) al exchange de fan-out global;
  **nunca** se le llama síncronamente (R08).
- El **orden por ficha** se preserva con routing keys que incluyen `ficha_id` y una cola con
  consumidor único por partición lógica cuando el orden importa
  ([ADR-001](../05-architecture/decisions/records/ADR-001-message-broker.md)).

### Envelope estándar

Todos los eventos usan el mismo envelope (ver [`_template/service/events.md`](./_template/service/events.md)). Para eventos de Academic, el schema canónico es [`07-api/contracts/schemas/event-envelope.schema.json`](../07-api/contracts/schemas/event-envelope.schema.json); prohíbe propiedades adicionales en el envelope y deja `correlation_id` como opcional:

```json
{
  "event_id": "uuid-v4",
  "event_type": "<servicio>.<entidad>.<accion>",
  "version": "1.0",
  "timestamp": "2026-01-01T00:00:00Z",
  "source_service": "<nombre-servicio>",
  "payload": {},
  "correlation_id": "uuid-v4"
}
```

Los campos propios del evento van dentro de `payload`. No se usan `schema_version`, `occurred_at`, `data` ni `source_port` como campos de envelope.

### Garantías de entrega

| Garantía | Cómo se cumple |
|----------|---------------|
| **At-least-once** | El broker reintenta hasta recibir `ack` del consumidor. Un evento puede llegar **más de una vez**. |
| **Idempotencia** | Cada consumidor deduplica por `event_id` (registro de eventos procesados). Procesar dos veces el mismo `event_id` no produce efecto adicional. Requisito para tolerar el at-least-once. |
| **Orden** | Solo garantizado dentro de una partición lógica (por `ficha_id`) cuando se configura consumidor único. Entre particiones no hay orden global. |
| **Dead Letter Queue** | Decisión abierta para Academic: falta acordar si se usará una DLQ común (`academic-events.dlq`) o una DLQ por consumidor, con su retención y reprocesamiento. |

### Patrón Outbox (publicación confiable)

Publicar un evento **y** confirmar la transacción de negocio de forma atómica es el problema del
*dual write*. Se resuelve con el **patrón Outbox**:

1. En la **misma transacción** de PostgreSQL que muta el estado de negocio, el servicio inserta
   el evento en una tabla `outbox` local. O ambos se confirman, o ninguno.
2. Un proceso relay (worker) lee filas pendientes de `outbox` y las publica en el broker.
3. Tras el `ack` del broker, marca la fila como publicada.

Esto garantiza que **no se pierde** ningún evento aunque el broker esté caído en el momento de la
escritura, y que **nunca** se publica un evento de una transacción que se revirtió. Combinado con
la idempotencia del consumidor (dedup por `event_id`), entrega semántica **efectivamente una vez**
sobre un transporte at-least-once.

> **Diseño previsto — no implementado por Docs.** Docs especifica el patrón, pero no implementa ni
> demuestra el Outbox. La tabla `outbox` y el relay se materializan en cada servicio publicador.
> Para Academic, el publicador documentado usa reintentos `1s → 4s → 16s` (tres reintentos tras
> el intento original) para publicar hacia el broker; eso es distinto del retry de consumo.

### Resiliencia en el consumo de eventos

| Mecanismo | Detalle |
|-----------|---------|
| **Ack manual** | El consumidor confirma (`ack`) solo tras procesar con éxito; si falla, `nack` y el broker reintenta. |
| **Reintentos con backoff** | Reintentos limitados con backoff antes de enviar a DLQ; en Academic queda abierta la política de consumo por consumidor. |
| **DLQ + reproceso** | Los mensajes en DLQ se inspeccionan y se reprocesan tras corregir la causa; retención y mecanismo de reprocesamiento quedan abiertos para Academic. |
| **Dedup por `event_id`** | Neutraliza los duplicados inherentes al at-least-once. |
| **Poblado inicial de read models** | Un servicio nuevo bootstrapea su read model vía snapshot del servicio dueño, y luego se mantiene por eventos (ADR-002). |

## Resumen: síncrono vs. asíncrono

| Criterio | Síncrono (REST) | Asíncrono (evento) |
|----------|-----------------|--------------------|
| Acoplamiento | Fuerte (disponibilidad) | Débil |
| Consistencia | Inmediata | Eventual |
| Uso típico | Validación puntual, lectura en línea, auth | Propagar mutaciones, auditoría, seguimiento, read models |
| Falla si el destino está caído | Sí (mitigar con timeout/breaker/fallback) | No (el broker retiene) |
| Límite de diseño | ≤ 2 por operación (R06) | Sin límite; fan-out libre |
| Entrega | 1 respuesta | At-least-once + idempotencia |

## Referencias

- [ADR-001 — Selección del broker de mensajes](../05-architecture/decisions/records/ADR-001-message-broker.md)
- [ADR-002 — Read models para reducir dependencias síncronas](../05-architecture/decisions/records/ADR-002-scheduling-read-models.md)
- [ADR-003 — Estrategia de object storage](../05-architecture/decisions/records/ADR-003-object-storage.md)
- [dependency-map.md](./dependency-map.md)
- [service-boundary-rules.md](./service-boundary-rules.md)
- [event-catalog.md](./event-catalog.md)
- [service-catalog.md](./service-catalog.md)
