<!-- RESUMEN-EJECUTIVO
capacidad: contrato async de notification-worker
fase: implementación incremental
estado: accepted
dependencias_entrada: event-catalog.md, notification-service/data-model.md
tldr: Consumidor transversal que transforma eventos de dominio en entregas, con idempotencia, retry/DLQ y Outbox.
-->

# Contrato — notification-worker

> Estado: 🟡 En progreso | Última actualización: 2026-08-26

## Eventos consumidos

| Evento | Exchange | Acción |
|--------|----------|--------|
| `monitoring.alert.triggered` | `monitoring-events` | Resolver actor afectado y notificar alerta. |
| `scheduling.schedule.published` | `scheduling-events` | Notificar confirmación de publicación. |

## Evento producido

| Evento | Exchange | Condición |
|--------|----------|-----------|
| `notification.notification.sent` | `notification-events` | Solo entrega exitosa; publicación vía Outbox. |

## Reglas de procesamiento

1. validar envelope y `event_type`;
2. consultar idempotencia por `source_event_id`;
3. resolver destinatario sin llamada síncrona a servicios de dominio (contrato/proyección local);
4. resolver/renderizar template cuando exista;
5. entregar por el adaptador del canal;
6. en error transitorio: persistir `FAILED`, devolver error y reintentar;
7. en éxito: persistir `SENT` + Outbox atómicamente;
8. `ack` únicamente después de terminar el procesamiento.

## Fallos

| Tipo | Comportamiento |
|------|----------------|
| Evento no soportado / payload no recuperable | Permanente → `nack` sin retry → DLQ. |
| SMTP/Broker/DB temporal | Backoff exponencial hasta máximo configurable. |
| Reintentos agotados | DLQ `notification-worker.dlq`. |
| Evento ya `SENT` | No-op idempotente + `ack`. |

## Privacidad

El worker no registra cuerpo de mensaje ni dirección de email completa en logs. Los logs operativos deben usar IDs, estado, canal y `trace_id`.

## Observabilidad

Propagar `traceparent` desde el evento de entrada al Outbox y al evento de salida. Instrumentar consumo AMQP, entrega, persistencia y publicación.
