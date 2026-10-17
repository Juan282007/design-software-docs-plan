# document-lifecycle-worker

> Estado: 🟡 En progreso | Última actualización: 2026-08-01
> Autor: Jesús Ariel González Bonilla (PM + Arquitecto) | Equipo: Arquitectura

> **Diseño previsto — no implementado.** Agnóstico de lenguaje.

## Tipo de componente
`-worker` — consumidor de eventos / tareas de ciclo de vida.

## Responsabilidad
Gestionar las transiciones de estado del documento (borrador → emitido → archivado) y aplicar políticas de **retención** y expiración sobre documentos y sus binarios.

## Tecnologías
| Capa | Tecnología |
|------|-----------|
| Runtime | Agnóstico — cualquier backend (a definir) |
| Framework | A definir |
| Persistencia | PostgreSQL 16 · schema `document` |
| Binarios | Object storage S3-compatible (ADR-003) |
| Transporte | Broker AMQP (RabbitMQ, ADR-001) |

## Variables de entorno (genéricas)
| Variable | Descripción |
|----------|-------------|
| `DB_URL` | Conexión a PostgreSQL |
| `BROKER_URL` | Conexión al broker |
| `OBJECT_STORAGE_ENDPOINT` | Endpoint S3-compatible |
| `RETENTION_POLICY` | Parámetro de política de retención |

## Idempotencia
Transiciones idempotentes por estado destino; deduplica por `event_id`.

## Contrato
Ver [contract.md](./contract.md)
