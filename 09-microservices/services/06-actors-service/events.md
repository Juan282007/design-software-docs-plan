# Eventos — actors-service

> Estado: 🟢 Estable | Última actualización: 2026-06-20

---

## Eventos publicados

Topic: **`actors-events`**

### `actors.instructor.assigned`

Instructor habilitado para dictar una competencia en una ficha de formación.

| Campo | Tipo | Descripción |
|-------|------|-------------|
| `instructor_id` | `uuid` | Identificador del instructor en actors-service |
| `competency_id` | `uuid` | Identificador interno de la competencia |
| `competency_code` | `string` | Código oficial de la competencia (ej. `220501046`) |
| `ficha_id` | `uuid` | Ficha de caracterización a la que se asigna |
| `assigned_by` | `uuid` | `user_id` del coordinador que realizó la asignación |
| `assigned_at` | `datetime` | ISO 8601 — momento de la asignación |

**Ejemplo de payload:**

```json
{
  "instructor_id": "a1b2c3d4-0001-0000-0000-000000000000",
  "competency_id": "e5f6a7b8-0002-0000-0000-000000000000",
  "competency_code": "220501046",
  "ficha_id": "c9d0e1f2-0003-0000-0000-000000000000",
  "assigned_by": "f3a4b5c6-0004-0000-0000-000000000000",
  "assigned_at": "2026-06-20T10:00:00Z"
}
```

**Consumidores:** `audit-service`, `monitoring-service`, `scheduling-service`

---

### `actors.learner.enrolled`

Aprendiz matriculado en una ficha de caracterización.

| Campo | Tipo | Descripción |
|-------|------|-------------|
| `learner_id` | `uuid` | Identificador del aprendiz en actors-service |
| `ficha_id` | `uuid` | Ficha en la que se matricula |
| `ficha_number` | `string` | Número de ficha visible (ej. `2879431`) |
| `stage` | `enum` | `LECTIVA` \| `PRODUCTIVA` |
| `enrollment_date` | `date` | Fecha efectiva de matrícula (`YYYY-MM-DD`) |
| `enrolled_by` | `uuid` | `user_id` del funcionario que registró la matrícula |
| `enrolled_at` | `datetime` | ISO 8601 — momento del registro en el sistema |

**Ejemplo de payload:**

```json
{
  "learner_id": "b2c3d4e5-0010-0000-0000-000000000000",
  "ficha_id": "c9d0e1f2-0003-0000-0000-000000000000",
  "ficha_number": "2879431",
  "stage": "LECTIVA",
  "enrollment_date": "2026-06-20",
  "enrolled_by": "f3a4b5c6-0004-0000-0000-000000000000",
  "enrolled_at": "2026-06-20T10:15:00Z"
}
```

**Consumidores:** `audit-service`, `monitoring-service`

---

### `actors.productive_stage.started`

Etapa productiva iniciada: el aprendiz comienza su práctica en una empresa.

| Campo | Tipo | Descripción |
|-------|------|-------------|
| `productive_stage_id` | `uuid` | Identificador de la etapa productiva |
| `learner_id` | `uuid` | Aprendiz que inicia la etapa |
| `company_id` | `uuid` | Empresa receptora |
| `company_name` | `string` | Nombre legal de la empresa |
| `supervisor_instructor_id` | `uuid` | Instructor de seguimiento asignado |
| `start_date` | `date` | Fecha de inicio (`YYYY-MM-DD`) |
| `planned_end_date` | `date` | Fecha de finalización estimada (`YYYY-MM-DD`) |
| `total_hours_required` | `integer` | Total de horas de etapa requeridas por el programa |
| `created_by` | `uuid` | `user_id` que registró la etapa |
| `created_at` | `datetime` | ISO 8601 — momento de creación del registro |

**Ejemplo de payload:**

```json
{
  "productive_stage_id": "d4e5f6a7-0020-0000-0000-000000000000",
  "learner_id": "b2c3d4e5-0010-0000-0000-000000000000",
  "company_id": "e5f6a7b8-0030-0000-0000-000000000000",
  "company_name": "Tecnologías del Valle S.A.S.",
  "supervisor_instructor_id": "a1b2c3d4-0001-0000-0000-000000000000",
  "start_date": "2026-07-01",
  "planned_end_date": "2026-12-31",
  "total_hours_required": 880,
  "created_by": "f3a4b5c6-0004-0000-0000-000000000000",
  "created_at": "2026-06-20T11:00:00Z"
}
```

**Consumidores:** `audit-service`, `monitoring-service`

---

### `actors.company_visit.completed`

Visita de seguimiento del instructor a la empresa registrada como completada.

| Campo | Tipo | Descripción |
|-------|------|-------------|
| `visit_id` | `uuid` | Identificador de la visita |
| `productive_stage_id` | `uuid` | Etapa productiva a la que pertenece la visita |
| `learner_id` | `uuid` | Aprendiz visitado |
| `company_id` | `uuid` | Empresa visitada |
| `instructor_id` | `uuid` | Instructor que realizó la visita |
| `visit_date` | `date` | Fecha de la visita (`YYYY-MM-DD`) |
| `visit_number` | `integer` | Número secuencial de visita dentro de la etapa (1, 2, 3…) |
| `visit_type` | `enum` | `SEGUIMIENTO` \| `EVALUACION_FINAL` |
| `learner_performance` | `enum` | `SATISFACTORIO` \| `EN_RIESGO` \| `INSATISFACTORIO` |
| `has_improvement_plan` | `boolean` | `true` si se generó plan de mejoramiento en la visita |
| `recorded_at` | `datetime` | ISO 8601 — momento en que el instructor cerró el registro |

**Ejemplo de payload:**

```json
{
  "visit_id": "f6a7b8c9-0040-0000-0000-000000000000",
  "productive_stage_id": "d4e5f6a7-0020-0000-0000-000000000000",
  "learner_id": "b2c3d4e5-0010-0000-0000-000000000000",
  "company_id": "e5f6a7b8-0030-0000-0000-000000000000",
  "instructor_id": "a1b2c3d4-0001-0000-0000-000000000000",
  "visit_date": "2026-09-15",
  "visit_number": 2,
  "visit_type": "SEGUIMIENTO",
  "learner_performance": "SATISFACTORIO",
  "has_improvement_plan": false,
  "recorded_at": "2026-09-15T16:45:00Z"
}
```

**Consumidores:** `audit-service`, `monitoring-service`

---

## Eventos consumidos

### `iam.user.created`

**Fuente:** `iam-service` — topic `iam-events`

**Acción:** actors-service crea o vincula el registro de actor según el campo `actor_type` del payload. Ver [Sincronización con iam-service](#sincronización-con-iam-service) para el detalle completo del mapeo.

---

### `iam.user.deactivated`

**Fuente:** `iam-service` — topic `iam-events`

**Acción:** actors-service localiza el registro de instructor o aprendiz cuyo `user_id` coincide y lo marca como inactivo (`active = false`). El registro se preserva íntegro para trazabilidad histórica; no se elimina. Las asignaciones de competencia y fichas activas del instructor pasan a estado `SUSPENDIDA` hasta resolución manual por parte del coordinador.

---

### `academic.competency.updated`

**Fuente:** `academic-management-service` — topic `academic-events`

**Acción:** actors-service revisa todas las asignaciones de competencia (`asignacion_competencia`) que referencian la `competency_id` modificada. Si el cambio afecta el código, nombre o nivel de la competencia, las asignaciones vigentes se marcan con `requires_review = true` para que el coordinador confirme o revoque la habilitación del instructor. No se desactivan automáticamente — el instructor permanece habilitado hasta revisión explícita.

**Contrato recibido:** `academic.competency.updated` usa el envelope común de Academic (`event_id`, `event_type`, `version`, `timestamp`, `source_service`, `payload` y `correlation_id` opcional). La implementación del consumidor de Actors depende de su HU propia; esta documentación solo registra la integración esperada.

---

## Formato de envelope

Todos los eventos siguen el envelope estándar del ecosistema:

```json
{
  "event_id": "uuid-v4",
  "event_type": "actors.instructor.assigned",
  "version": "1.0",
  "timestamp": "2026-06-20T10:00:00Z",
  "source_service": "actors-service",
  "correlation_id": "uuid-v4",
  "payload": { }
}
```

| Campo | Descripción |
|-------|-------------|
| `event_id` | UUID v4 generado por actors-service en el momento de publicar. Idempotency key para consumidores. |
| `event_type` | Nombre del evento en formato `<servicio>.<entidad>.<accion>`. |
| `version` | Versión del contrato de payload. Incremento mayor si hay breaking changes. |
| `timestamp` | Momento de publicación en UTC, ISO 8601. |
| `source_service` | Siempre `actors-service` para eventos de este catálogo. |
| `correlation_id` | Propagado desde el request HTTP que originó la acción. Permite rastrear la cadena de llamadas en los logs. |

---

## Política de reintentos

- **Broker:** Kafka (o equivalente configurado en el entorno).
- **Topic de mensajes fallidos:** `actors-events.dlq` (Dead Letter Queue).
- **Reintentos en consumo:** 3 intentos con backoff exponencial (1 s → 4 s → 16 s) antes de enviar el mensaje a la DLQ.
- **Retención en DLQ:** 7 días. Pasado ese plazo el mensaje expira.
- **Procesamiento idempotente:** Los consumidores deben usar `event_id` como clave de idempotencia. actors-service garantiza que cada evento tiene un `event_id` único; los consumidores son responsables de deduplicar en su propio almacén.
- **Alertas:** Un mensaje en `actors-events.dlq` debe generar una alerta en `monitoring-service` dentro de los 5 minutos siguientes a su llegada.

---

## Sincronización con iam-service

Cuando `iam.user.created` llega con `actor_type=INSTRUCTOR`, actors-service crea o vincula el registro instructor usando `user_id` como clave de join. Si `actor_type=LEARNER`, crea o vincula el registro aprendiz de la misma forma. El `user_id` del JWT sirve como join key en todas las operaciones posteriores: cualquier servicio que llame a actors-service con un token válido puede derivar el `instructor_id` o `learner_id` correspondiente a partir del `user_id` embebido en el JWT, sin necesidad de una llamada de resolución separada.

Los campos que actors-service copia del evento `iam.user.created` al registro de actor son:

| Campo en `iam.user.created` | Campo en actors-service |
|-----------------------------|-------------------------|
| `user_id` | `user_id` (join key) |
| `email` | `email` |
| `first_name` | `first_name` |
| `last_name` | `last_name` |
| `actor_type` | determina la entidad destino (`instructor` o `learner`) |

El resto de los campos del perfil de actor (número de documento, tipo de contrato, competencias, etc.) se completan mediante la API de actors-service una vez creado el registro base.
