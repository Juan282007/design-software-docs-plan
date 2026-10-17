# Decisiones internas — notification-service

> Estado: 🟢 Vigente | Última actualización: 2026-08-26

Las decisiones de esta página especializan ADRs globales sin reemplazarlas.

| ID | Decisión | Alternativas | Justificación | Estado |
|----|----------|--------------|---------------|--------|
| N01 | `notification-service` es una capacidad transversal independiente | Mantener `notification-worker` dentro de Monitoring | ADR-005 elimina el acoplamiento de dominios que necesitan notificar con Monitoring. | ACCEPTED |
| N02 | Componentes: `notification-api` + `notification-worker` dentro del mismo repo | Repo por worker/API | ADR-005: repo = bounded context; contenedor = proceso. | ACCEPTED |
| N03 | Go como runtime actual | Java/Python/Node | ADR-006 ubica Notification en el clúster Go por throughput y simplicidad operacional. | ACCEPTED |
| N04 | `EMAIL` e `IN_APP` son los únicos canales v1 | SMS, WhatsApp, push móvil | Mantener alcance MVP; nuevos canales son cambios aditivos de contrato. | ACCEPTED |
| N05 | Entrega por eventos + API pública | Invocar directamente un worker | Los workers son internos; los consumidores usan contratos públicos. | ACCEPTED |
| N06 | Idempotencia por `source_event_id` | Deduplicación solo en memoria | RabbitMQ es at-least-once; la protección debe sobrevivir reinicios/concurrencia. | ACCEPTED |
| N07 | Patrón Outbox para `notification.notification.sent` | Persistir y publicar en dos pasos independientes | Evita dual-write y pérdida de eventos de salida. | ACCEPTED |
| N08 | Reintentos con backoff + DLQ | Retry infinito | Evita bloquear la cola y separa fallos permanentes. | ACCEPTED |
| N09 | Plantillas propias en `notification_db` | Plantillas dentro de cada servicio productor | Centraliza la presentación de la notificación sin trasladar reglas de negocio. | ACCEPTED |
| N10 | La bandeja UI pertenece a `shell-host` | `notification-mfe` o `monitoring-mfe` | ADR-005 define notificaciones como chrome global; no se crea MFE #10. | ACCEPTED |
| N11 | `GET /notifications` queda como gap explícito hasta formalizar contrato | Reutilizar `/sent-notifications` de Monitoring | Monitoring ya no es dueño de los datos. El listado debe nacer en Notification o en un BFF/read model. | ACCEPTED |
| N12 | Sin dependencia síncrona a otros servicios durante la entrega async | Consultar Actors/Monitoring por HTTP al consumir cada evento | Mantiene aislamiento y resiliencia. El destino debe venir en contrato autorizado o en proyección local propia; el stub actual solo aplica a desarrollo. | ACCEPTED |

## Decisión supersedida en Monitoring

La decisión histórica `monitoring-service/decisions.md#10` que mantenía `notification-worker` dentro de Monitoring queda **SUPERSEDED por ADR-005**. Se conserva una nota histórica en Monitoring solo para trazabilidad; no es una opción vigente.

## Referencias

- [ADR-005](../../../05-architecture/decisions/records/ADR-005-microservices-topology-and-transversal-capabilities.md)
- [ADR-006](../../../05-architecture/decisions/records/ADR-006-polyglot-stack-strategy.md)
- [events.md](./events.md)
- [data-model.md](./data-model.md)
