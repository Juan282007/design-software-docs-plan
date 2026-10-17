# Eventos — monitoring-service

> Estado: 🟡 En progreso | Última actualización: 2026-08-26
> Transporte: RabbitMQ / AMQP

> Fuente global: [event-catalog.md](../../event-catalog.md). Monitoring publica **hechos de seguimiento/alerta**, nunca hechos de entrega de notificación.

## Eventos publicados

### `monitoring.alert.triggered`

Se publica cuando el `alert-worker` persiste una nueva `generated_alert` que requiere exposición a consumidores externos.

```json
{
  "event_id": "f3a1bc20-0d4e-4c2a-8e91-2b5d7f0a3e88",
  "event_type": "monitoring.alert.triggered",
  "version": "1.0",
  "timestamp": "2026-08-26T20:00:00Z",
  "source_service": "monitoring-service",
  "correlation_id": "a0e2d91c-11fc-4b78-b563-4f8c2e7d5a01",
  "payload": {
    "alert_id": "9c4e2b1a-3f7d-4a0e-b812-6d5c9f2e0a77",
    "ficha_tracking_id": "b2d3e4f5-6a7b-8c9d-0e1f-2a3b4c5d6e7f",
    "ficha_id": "1a2b3c4d-5e6f-7a8b-9c0d-e1f2a3b4c5d6",
    "alert_type_code": "LOW_ATTENDANCE",
    "risk_level_code": "HIGH",
    "affected_entity_type": "Learner",
    "affected_entity_id": "d4e5f6a7-b8c9-0d1e-2f3a-4b5c6d7e8f9a",
    "generated_at": "2026-08-26T19:59:59Z"
  }
}
```

Consumidores principales: `notification-service`, `audit-service`.

### `monitoring.kpi.threshold_breached`

Se publica cuando una medición de KPI cruza el umbral configurado y otros consumidores necesitan reaccionar al hecho.

```json
{
  "event_id": "e7b2d4a1-9c3f-4e0b-a215-8f6d1c9e3b47",
  "event_type": "monitoring.kpi.threshold_breached",
  "version": "1.0",
  "timestamp": "2026-08-26T20:00:00Z",
  "source_service": "monitoring-service",
  "correlation_id": "a0e2d91c-11fc-4b78-b563-4f8c2e7d5a01",
  "payload": {
    "kpi_tracking_id": "c3d4e5f6-7a8b-9c0d-1e2f-3a4b5c6d7e8f",
    "ficha_tracking_id": "b2d3e4f5-6a7b-8c9d-0e1f-2a3b4c5d6e7f",
    "kpi_type_code": "ATTENDANCE",
    "current_value": 0.72,
    "threshold_value": 0.80,
    "measured_at": "2026-08-26T19:59:58Z"
  }
}
```

## Eventos consumidos

| Evento | Productor | Acción en Monitoring |
|--------|-----------|----------------------|
| `academic.ficha.opened` | `academic-management-service` | Inicializa seguimiento de ficha. |
| `academic.ficha.closed` | `academic-management-service` | Cierra seguimiento activo. |
| `scheduling.schedule.published` | `scheduling-service` | Actualiza contexto/fecha de seguimiento y dispara evaluación pertinente. |
| `scheduling.class_session.cancelled` | `scheduling-service` | Reevalúa KPI afectados. |
| `actors.company_visit.completed` | `actors-service` | Actualiza KPI de etapa productiva. |

Los eventos `academic.ficha.opened` y `academic.ficha.closed` se consumen con el envelope común (`event_id`, `event_type`, `version`, `timestamp`, `source_service`, `payload` y `correlation_id` opcional); sus datos de ficha van dentro de `payload`.

## Evento retirado

`monitoring.notification.sent` **no es un contrato vigente**. Tras ADR-005:

```text
monitoring.alert.triggered
        │
        ▼
notification-service
        │
        ▼
notification.notification.sent
```

Monitoring no conoce `send_status`, SMTP, canal ni `notification_id`.

## Idempotencia y reintentos

- El `alert-worker` deduplica los hechos externos por `event_id`/`source_event_id` conforme a la implementación de persistencia.
- Reintentos de procesamiento de eventos de Monitoring pertenecen al consumer de Monitoring.
- Reintentos de **entrega de notificaciones** pertenecen exclusivamente a `notification-service`.
- DLQ de Monitoring y DLQ de Notification son independientes.

## Flujo de alerta desacoplado

```text
evento de dominio
      │
      ▼
alert-worker / caso de uso M9
      │
      ├─ persiste kpi_tracking / generated_alert
      │
      └─ publica monitoring.alert.triggered
                     │
          ┌──────────┴──────────┐
          ▼                     ▼
 notification-service      audit-service
    (entrega)              (auditoría)
```

## Referencias

- [alert-worker contract](./components/alert-worker/contract.md)
- [notification-service events](../10-notification-service/events.md)
- [ADR-005](../../../05-architecture/decisions/records/ADR-005-microservices-topology-and-transversal-capabilities.md)
