# audit-worker

> Estado: 🟡 En progreso | Última actualización: 2026-08-01
> Autor: Jesús Ariel González Bonilla (PM + Arquitecto) | Equipo: Arquitectura

> **Diseño previsto — no implementado.** Agnóstico de lenguaje.

## Tipo de componente
`-worker` — consumidor **universal** de eventos.

## Responsabilidad
Suscribirse a **todos los topics** (fan-out, [ADR-001](../../../../../05-architecture/decisions/records/ADR-001-message-broker.md)) y persistir cada evento en el log **append-only** `audit_record` (solo INSERT). No expone escritura externa; garantiza trazabilidad ([ADR-004](../../../../../05-architecture/decisions/records/ADR-004-status-parametrization-and-audit-standard.md)).

## Tecnologías
| Capa | Tecnología |
|------|-----------|
| Runtime | Agnóstico — cualquier backend (a definir) |
| Framework | A definir |
| Persistencia | PostgreSQL 16 · schema `audit` (event store append-only) |
| Transporte | Broker AMQP (RabbitMQ, ADR-001), suscripción fan-out a todos los topics |

## Variables de entorno (genéricas)
| Variable | Descripción |
|----------|-------------|
| `DB_URL` | Conexión a PostgreSQL |
| `BROKER_URL` | Conexión al broker |
| `CONSUMER_GROUP` | Grupo del consumidor de auditoría |

## Idempotencia
`event_id` es **UNIQUE** en `audit_record` → un evento repetido no crea un segundo registro. Entrega at-least-once tolerada por diseño.

## Contrato
Ver [contract.md](./contract.md) · [events.md](../../events.md)
