# notification-worker

> Estado: 🟡 En progreso | Última actualización: 2026-08-26
> Stack: Go

## Tipo de componente

`-worker` — consumidor AMQP + relay Outbox.

## Responsabilidad

- consumir eventos soportados;
- aplicar idempotencia por `event_id`;
- resolver destinatario;
- renderizar plantilla;
- entregar `EMAIL` / `IN_APP`;
- persistir `SENT` / `FAILED`;
- reintentar fallos transitorios;
- enrutar fallos agotados a DLQ;
- publicar `notification.notification.sent` desde Outbox.

## Entradas actuales

- `monitoring.alert.triggered`
- `scheduling.schedule.published`

## Salida

- `notification.notification.sent`

## Health

El worker expone un servidor mínimo independiente:

- `GET /health`
- `GET /ready` — DB + RabbitMQ

## Contrato

Ver [contract.md](./contract.md) y [events.md](../../events.md).
