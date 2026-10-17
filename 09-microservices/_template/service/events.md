# Eventos — [service-name]

> **PLANTILLA** — Registrar eventos que este servicio publica y consume.
> Estado: 🔴 Pendiente | Última actualización: YYYY-MM-DD

## Eventos publicados

| Evento | Topic / Queue | Payload principal | Consumidores conocidos |
|--------|---------------|-------------------|------------------------|
| `[servicio].[entidad].[acción]` | `[topic]` | `{ id, ... }` | `[servicio-consumidor]` |

## Eventos consumidos

| Evento | Servicio fuente | Acción que dispara |
|--------|----------------|--------------------|
| `[servicio].[entidad].[acción]` | `[servicio]` | [qué hace este servicio al recibirlo] |

## Formato de envelope

Todos los eventos siguen el envelope estándar del ecosistema:

```json
{
  "event_id": "uuid-v4",
  "event_type": "<servicio>.<entidad>.<accion>",
  "version": "1.0",
  "timestamp": "2026-01-01T00:00:00Z",
  "source_service": "<nombre-servicio>",
  "correlation_id": "uuid-v4",
  "payload": {}
}
```

## Política de reintentos

<!-- Circuit breaker, dead letter queue, retries — completar antes de llevar a producción -->
