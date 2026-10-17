# Eventos — training-environment-service

> Estado: 🟢 Estable | Última actualización: 2026-06-20

---

## Eventos publicados

Topic: `environment-events`

| Evento | Descripción | Consumidores conocidos |
|--------|-------------|------------------------|
| `environment.environment.created` | Ambiente físico nuevo registrado en el sistema | `audit-service`, `scheduling-service` |
| `environment.availability.changed` | Regla de disponibilidad agregada, eliminada o modificada | `audit-service`, `scheduling-service` |
| `environment.maintenance.started` | Período de mantenimiento creado; bloquea el ambiente | `audit-service`, `scheduling-service` |
| `environment.reservation.created` | Reserva confirmada sobre un ambiente | `audit-service` |

---

### `environment.environment.created`

Publicado cuando un ambiente físico (aula, laboratorio, taller) queda registrado y disponible para asignación.

```json
{
  "event_id": "a1b2c3d4-0000-0000-0000-000000000001",
  "event_type": "environment.environment.created",
  "version": "1.0",
  "timestamp": "2026-06-20T08:00:00Z",
  "source_service": "training-environment-service",
  "correlation_id": "f9e8d7c6-0000-0000-0000-000000000099",
  "payload": {
    "environment_id": "uuid-ambiente",
    "name": "Laboratorio de Redes 302",
    "environment_type_id": "uuid-tipo",
    "training_center_id": "uuid-centro",
    "capacity": 30,
    "is_active": true,
    "created_by": "uuid-usuario",
    "created_at": "2026-06-20T08:00:00Z"
  }
}
```

| Campo | Tipo | Descripción |
|-------|------|-------------|
| `environment_id` | UUID | ID del ambiente recién creado |
| `name` | string | Nombre descriptivo del ambiente |
| `environment_type_id` | UUID | FK → `environment_type` (CLASSROOM, LAB, WORKSHOP…) |
| `training_center_id` | UUID | Referencia externa → `reference-data-service` |
| `capacity` | integer | Número máximo de personas |
| `is_active` | boolean | `true` cuando el ambiente está operativo |
| `created_by` | UUID | Usuario que realizó el registro (iam-service) |
| `created_at` | ISO 8601 | Timestamp de creación |

---

### `environment.availability.changed`

Publicado cuando se agrega, elimina o modifica una regla de disponibilidad (`availability_rule`). `scheduling-service` debe re-sincronizar su read model al recibir este evento (ver ADR-002).

```json
{
  "event_id": "a1b2c3d4-0000-0000-0000-000000000002",
  "event_type": "environment.availability.changed",
  "version": "1.0",
  "timestamp": "2026-06-20T09:00:00Z",
  "source_service": "training-environment-service",
  "correlation_id": "f9e8d7c6-0000-0000-0000-000000000099",
  "payload": {
    "environment_id": "uuid-ambiente",
    "change_type": "RULE_ADDED",
    "affected_days": [1, 2, 3, 4, 5],
    "effective_from": "2026-07-01",
    "changed_by": "uuid-usuario",
    "changed_at": "2026-06-20T09:00:00Z"
  }
}
```

| Campo | Tipo | Valores | Descripción |
|-------|------|---------|-------------|
| `environment_id` | UUID | — | Ambiente cuya disponibilidad cambió |
| `change_type` | enum | `RULE_ADDED` \| `RULE_REMOVED` \| `RULE_MODIFIED` | Naturaleza del cambio |
| `affected_days` | integer[] | 1=Lunes … 7=Domingo | Días de la semana afectados por el cambio |
| `effective_from` | DATE | ISO 8601 date | Fecha desde la que aplica el cambio |
| `changed_by` | UUID | — | Usuario que realizó la modificación |
| `changed_at` | ISO 8601 | — | Timestamp de la operación |

---

### `environment.maintenance.started`

Publicado cuando se registra un período de mantenimiento. El bloqueo de disponibilidad es inmediato desde `start_date`. `scheduling-service` debe re-sincronizar su read model al recibir este evento (ver ADR-002).

```json
{
  "event_id": "a1b2c3d4-0000-0000-0000-000000000003",
  "event_type": "environment.maintenance.started",
  "version": "1.0",
  "timestamp": "2026-06-20T10:00:00Z",
  "source_service": "training-environment-service",
  "correlation_id": "f9e8d7c6-0000-0000-0000-000000000099",
  "payload": {
    "environment_id": "uuid-ambiente",
    "maintenance_id": "uuid-mantenimiento",
    "start_date": "2026-07-05",
    "end_date": "2026-07-07",
    "description": "Reemplazo de equipos de cómputo",
    "created_by": "uuid-usuario",
    "created_at": "2026-06-20T10:00:00Z"
  }
}
```

| Campo | Tipo | Descripción |
|-------|------|-------------|
| `environment_id` | UUID | Ambiente que entra en mantenimiento |
| `maintenance_id` | UUID | ID del registro de mantenimiento |
| `start_date` | ISO 8601 date | Inicio del bloqueo |
| `end_date` | ISO 8601 date | Fin del bloqueo (exclusivo: el ambiente vuelve a estar disponible en `end_date`) |
| `description` | string | Motivo del mantenimiento |
| `created_by` | UUID | Usuario que registró el mantenimiento |
| `created_at` | ISO 8601 | Timestamp de creación del registro |

---

### `environment.reservation.created`

Publicado cuando una reserva queda confirmada. No modifica las reglas de disponibilidad base; representa una ocupación puntual.

```json
{
  "event_id": "a1b2c3d4-0000-0000-0000-000000000004",
  "event_type": "environment.reservation.created",
  "version": "1.0",
  "timestamp": "2026-06-20T11:00:00Z",
  "source_service": "training-environment-service",
  "correlation_id": "f9e8d7c6-0000-0000-0000-000000000099",
  "payload": {
    "reservation_id": "uuid-reserva",
    "environment_id": "uuid-ambiente",
    "reservation_date": "2026-07-10",
    "start_time": "08:00:00",
    "end_time": "10:00:00",
    "status": "CONFIRMED",
    "requester_id": "uuid-usuario",
    "created_at": "2026-06-20T11:00:00Z"
  }
}
```

| Campo | Tipo | Valores | Descripción |
|-------|------|---------|-------------|
| `reservation_id` | UUID | — | ID de la reserva |
| `environment_id` | UUID | — | Ambiente reservado |
| `reservation_date` | ISO 8601 date | — | Fecha de la reserva |
| `start_time` | TIME | HH:MM:SS | Inicio de la ocupación |
| `end_time` | TIME | HH:MM:SS | Fin de la ocupación |
| `status` | enum | `PENDING` \| `CONFIRMED` \| `CANCELLED` | Estado al momento de publicar el evento |
| `requester_id` | UUID | — | Usuario que solicitó la reserva |
| `created_at` | ISO 8601 | — | Timestamp de creación |

---

## Eventos consumidos

> `training-environment-service` no consume eventos de otros servicios. Todas sus dependencias externas son síncronas: validación de JWT contra `iam-service` y lectura de catálogos contra `reference-data-service`.

---

## Formato de envelope

Todos los eventos siguen el envelope estándar del ecosistema:

```json
{
  "event_id": "uuid-v4",
  "event_type": "<servicio>.<entidad>.<accion>",
  "version": "1.0",
  "timestamp": "2026-06-20T00:00:00Z",
  "source_service": "training-environment-service",
  "correlation_id": "uuid-v4",
  "payload": {}
}
```

| Campo | Tipo | Obligatorio | Descripción |
|-------|------|-------------|-------------|
| `event_id` | UUID v4 | Sí | Identificador único del evento; permite deduplicación |
| `event_type` | string | Sí | Nombre canónico del evento (convención: `<service>.<entity>.<action>`) |
| `version` | string | Sí | Versión del schema del payload (`"1.0"`) |
| `timestamp` | ISO 8601 | Sí | Momento en que ocurrió el hecho de negocio |
| `source_service` | string | Sí | Servicio publicador; siempre `"training-environment-service"` para este topic |
| `correlation_id` | UUID v4 | Sí | ID de la operación de origen; permite trazar el flujo en logs distribuidos |
| `payload` | object | Sí | Datos específicos del evento (ver secciones anteriores) |

---

## Política de reintentos

- **Topic**: `environment-events`
- **Dead Letter Queue**: `environment-events.dlq`
- **Reintentos antes de DLQ**: 3 intentos con backoff exponencial (1 s → 2 s → 4 s)
- **Retención en DLQ**: 7 días
- **Acción ante mensaje en DLQ**: alerta a `monitoring-service`; revisión manual obligatoria antes de reencolar

Los consumidores deben implementar procesamiento **idempotente**: el `event_id` del envelope es la clave de deduplicación. Un evento reencolado desde la DLQ con el mismo `event_id` no debe producir efectos secundarios adicionales.

---

## Nota para scheduling-service

`scheduling-service` mantiene read models locales de disponibilidad de ambientes (ver [ADR-002](../../../05-architecture/decisions/records/ADR-002-scheduling-read-models.md)). Debe re-sincronizar su read model al recibir `environment.availability.changed` o `environment.maintenance.started`.

Para el poblado inicial del read model (bootstrap), `training-environment-service` expone un endpoint de snapshot en `training-environment-api`. Consultar el [contrato de la API](./components/training-environment-api/contract.md) para los detalles del endpoint.
