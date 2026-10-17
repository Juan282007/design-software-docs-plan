# Modelo de datos — notification-service

> Estado: 🟡 En progreso | Última actualización: 2026-08-26
> Naming: entidades y atributos en inglés (HALT-DB-NAMING)

## Convenciones

Aplican las reglas de [06-data/modeling-conventions.md](../../../06-data/modeling-conventions.md): UUID, `TIMESTAMPTZ`, constraints con nombre, enums técnicos como `VARCHAR + CHECK`, propiedad exclusiva por servicio e idempotencia por claves únicas.

**BD lógica:** `notification_db` · **schema:** `notification` · **motor:** PostgreSQL 16.

## Entidades

### `sent_notification`

Registro de una solicitud/entrega de notificación.

| Campo | Tipo | Nullable | Descripción |
|-------|------|----------|-------------|
| `id` | UUID | No | PK. |
| `recipient_id` | UUID | No | Identificador lógico del destinatario. |
| `recipient_email` | VARCHAR(255) | No para EMAIL | PII; dirección resuelta para entrega por email. |
| `channel` | VARCHAR(20) | No | `EMAIL` / `IN_APP`. |
| `subject` | VARCHAR(300) | No | Asunto/título. |
| `body_summary` | TEXT | Sí | Cuerpo renderizado/resumen persistido según política. No debe copiarse a logs. |
| `send_status` | VARCHAR(20) | No | `PENDING` / `SENT` / `FAILED`. |
| `failure_reason` | TEXT | Sí | Motivo técnico sanitizado cuando falla. |
| `template_id` | UUID | Sí | FK local → `notification_template.id`. |
| `source_service` | VARCHAR(100) | Sí | Servicio que originó la solicitud/evento. |
| `source_event_id` | UUID | Sí | ID del evento fuente para idempotencia. |
| `sent_at` | TIMESTAMPTZ | Sí | Momento de entrega efectiva. |
| `created_at` | TIMESTAMPTZ | No | Momento de creación. |

**CHECKs:**

```sql
CHECK (channel IN ('EMAIL', 'IN_APP'))
CHECK (send_status IN ('PENDING', 'SENT', 'FAILED'))
```

**Idempotencia:** `source_event_id` no nulo debe tener índice único parcial. Una redelivery AMQP del mismo evento no puede causar doble entrega.

```sql
CREATE UNIQUE INDEX uq_sent_notification_source_event_id
ON notification.sent_notification (source_event_id)
WHERE source_event_id IS NOT NULL;
```

### `notification_template`

Plantilla reutilizable por evento/caso de uso.

| Campo | Tipo | Nullable | Descripción |
|-------|------|----------|-------------|
| `id` | UUID | No | PK. |
| `code` | VARCHAR(100) | No | Código único, p. ej. `ALERT_TRIGGERED`. |
| `channel` | VARCHAR(20) | No | `EMAIL` / `IN_APP`. |
| `subject_template` | TEXT | No | Plantilla de asunto. |
| `body_template` | TEXT | No | Plantilla de cuerpo. |
| `is_active` | BOOLEAN | No | Habilitación lógica. |
| `created_at` | TIMESTAMPTZ | No | Auditoría. |
| `updated_at` | TIMESTAMPTZ | No | Auditoría. |

**CHECK:** `channel IN ('EMAIL', 'IN_APP')`.

Plantillas iniciales alineadas con los eventos implementados:

| Código | Evento |
|--------|--------|
| `ALERT_TRIGGERED` | `monitoring.alert.triggered` |
| `SCHEDULE_PUBLISHED` | `scheduling.schedule.published` |

### `outbox`

Tabla técnica del patrón Outbox. El relay publica filas pendientes y marca `published_at` después del `ack` del broker.

| Campo | Tipo | Nullable | Descripción |
|-------|------|----------|-------------|
| `id` | UUID | No | PK. |
| `event_id` | UUID | No | ID del evento de salida. |
| `event_type` | VARCHAR(150) | No | En v1: `notification.notification.sent`. |
| `payload` | JSONB | No | Payload de integración. |
| `created_at` | TIMESTAMPTZ | No | Creación. |
| `published_at` | TIMESTAMPTZ | Sí | Null mientras está pendiente. |

## Relaciones

```text
notification_template 1 ─── 0..N sent_notification

sent_notification
  └── source_event_id  (referencia lógica; sin FK cross-service)

sent_notification SENT
  └── misma transacción ──> outbox
```

No existen FKs hacia `monitoring_db`, `scheduling_db`, `actors_db` ni ninguna base de datos ajena.

## Índices mínimos

| Tabla | Índice | Motivo |
|-------|--------|--------|
| `sent_notification` | UNIQUE parcial (`source_event_id`) | Idempotencia async. |
| `sent_notification` | (`recipient_id`, `created_at DESC`) | Bandeja/consulta por destinatario cuando se implemente listado. |
| `sent_notification` | (`send_status`, `created_at`) | Operación/reintentos. |
| `notification_template` | UNIQUE (`code`) | Resolución por código. |
| `outbox` | (`published_at`, `created_at`) WHERE `published_at IS NULL` | Relay eficiente. |

## Retención y PII

- `sent_notification`: objetivo inicial de retención **1 año**, sujeto a validación normativa.
- `outbox`: puede purgarse/archivarse después de publicación conforme a política operativa.
- `recipient_email`: PII; acceso restringido y no expuesto en logs.
- El servicio de auditoría registra el evento de integración, no el contenido completo del mensaje.

## Propiedad

`notification-service` es el **único escritor** de estas tablas. Monitoring deja de poseer `sent_notification` a partir de ADR-005.

## Referencias

- [README del servicio](./README.md)
- [Eventos](./events.md)
- [ADR-005](../../../05-architecture/decisions/records/ADR-005-microservices-topology-and-transversal-capabilities.md)
- [Matriz de ownership](../../data-ownership-matrix.md)
