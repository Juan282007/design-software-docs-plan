# Secuencia — Outbox → relay → `iam-events` (`iam-service`)

> Estado: 🟢 Listo | Última actualización: 2026-08-28
> Autor: Jesús Ariel González Bonilla (PM + Arquitecto) | Equipo: Arquitectura
> Alcance: relay del patrón Outbox construido y probado en `develop` (Java / Spring). Ver [ADR-001](../../../05-architecture/decisions/records/ADR-001-message-broker.md) y [events.md del iam-service](../../../09-microservices/services/01-iam-service/events.md).

`OutboxRelayScheduler` sondea periódicamente la tabla `identity.outbox`, publica cada evento
pendiente al exchange `iam-events` del broker y marca el registro como publicado. El evento se
persiste con su carga bajo la columna `data` y el relay la **renombra a `payload`** al construir
el envelope que publica. Es el mecanismo que evita el *dual-write* entre la BD y el broker.

```mermaid
sequenceDiagram
    autonumber
    participant Sched as OutboxRelayScheduler
    participant DB as iam_db (identity.outbox)
    participant Broker as RabbitMQ (exchange iam-events)
    participant Cons as Consumidores (notification, audit)

    loop poll periódico
        Sched->>DB: SELECT * FROM identity.outbox WHERE published_at IS NULL ORDER BY created_at
        DB-->>Sched: filas pendientes (event_id, event_type, data)

        alt hay pendientes
            loop por cada evento
                Sched->>Sched: construir envelope (rename columna data -> payload)
                Sched->>Broker: publish(routing_key = event_type, envelope)
                alt publish OK (ack del broker)
                    Broker-->>Sched: ack
                    Sched->>DB: UPDATE outbox SET published_at = now() WHERE event_id
                    Broker-->>Cons: entrega asíncrona del evento
                else publish falla
                    Broker-->>Sched: nack / timeout
                    Note over Sched,DB: no se marca published_at; se reintenta en el próximo poll (at-least-once)
                end
            end
        else sin pendientes
            Sched-->>Sched: nada que publicar
        end
    end
```

## Notas

- Eventos publicados por `iam` en `iam-events`: `user.created`, `user.deactivated`,
  `role.assigned` y `session.started` (este último *staged* durante el login — ver
  [login-jwt.md](./login-jwt.md)).
- El relay es **at-least-once**: si el `ack` no llega, el registro queda sin `published_at` y se
  reintenta; los consumidores deben ser idempotentes por `event_id`.
- Renombrado `data` → `payload`: es una deuda de contrato conocida (envelope plano vs. anidado en
  `shared-contracts`); el relay es el punto donde se normaliza la carga hacia el envelope publicado.
- Cobertura: verificado en el E2E de compose de `iam` (outbox→relay→iam-events), con trazas en Tempo.
