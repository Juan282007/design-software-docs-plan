# alert-worker

> Estado: 🟡 En progreso | Última actualización: 2026-08-26
> Stack: Python

## Tipo de componente

`-worker` — consumidor/evaluador asíncrono del bounded context M9.

## Responsabilidad

Evaluar KPIs/umbrales, persistir `kpi_tracking`/`generated_alert` y registrar
`monitoring.alert.triggered` en el outbox transaccional cuando corresponde. El relay del
componente publica después el evento al broker.

**No entrega notificaciones.** `notification-service` es un consumidor externo del evento publicado.

## Estado de implementación

`apps/alert_worker/main.py` existe como skeleton; AMQP, outbox/relay, deduplicación y OTel están pendientes de completar.

## Persistencia

PostgreSQL 16 · `monitoring_db`.

## Contrato

- [contract.md](./contract.md)
- [events.md](../../events.md)
