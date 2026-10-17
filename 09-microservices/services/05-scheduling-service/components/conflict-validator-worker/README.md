# conflict-validator-worker

> Estado: 🟡 En progreso | Última actualización: 2026-08-01
> Autor: Jesús Ariel González Bonilla (PM + Arquitecto) | Equipo: Arquitectura

> **Diseño previsto — no implementado.** Agnóstico de lenguaje.

## Tipo de componente
`-worker` — consumidor de eventos / cola.

## Responsabilidad
Detectar conflictos de horario (solape de instructor, ambiente o ficha) de forma asíncrona ante cambios de disponibilidad o de sesiones, y registrar/publicar los conflictos hallados.

## Tecnologías
| Capa | Tecnología |
|------|-----------|
| Runtime | Agnóstico — cualquier backend (a definir) |
| Framework | A definir |
| Persistencia | PostgreSQL 16 · schema `scheduling` |
| Transporte | Broker AMQP (RabbitMQ, [ADR-001](../../../../../05-architecture/decisions/records/ADR-001-message-broker.md)) |

## Variables de entorno (genéricas)
| Variable | Descripción |
|----------|-------------|
| `DB_URL` | Conexión a PostgreSQL |
| `BROKER_URL` | Conexión al broker |
| `CONSUMER_GROUP` | Identificador del consumidor |
| `MAX_RETRIES` | Reintentos antes de DLQ |

## Idempotencia y resiliencia
Deduplica por `event_id`; entrega **at-least-once**; los fallos van a una **DLQ** por consumer. Ver [communication-patterns.md](../../../../communication-patterns.md).

## Contrato
Ver [contract.md](./contract.md) · [data-model.md](../../data-model.md)
