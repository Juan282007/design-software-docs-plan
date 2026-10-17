# training-environment-api

> Estado: 🟡 En progreso | Última actualización: 2026-08-01
> Autor: Jesús Ariel González Bonilla (PM + Arquitecto) | Equipo: Arquitectura

> **Diseño previsto — no implementado.** Agnóstico de lenguaje.

## Tipo de componente
`-api` — REST API sincrónica.

## Responsabilidad
Exponer la gestión de ambientes de formación, sus reglas de disponibilidad, reservas e inventario.

## Tecnologías
| Capa | Tecnología |
|------|-----------|
| Runtime | Agnóstico — cualquier backend (a definir) |
| Framework | A definir |
| Persistencia | PostgreSQL 16 · schema `training_environment` |
| Autenticación | JWT de `iam-service` |

## Variables de entorno (genéricas)
| Variable | Descripción |
|----------|-------------|
| `SERVICE_PORT` | Puerto de escucha |
| `DB_URL` | Conexión a PostgreSQL |
| `JWT_PUBLIC_KEY` | Verificación del token |
| `BROKER_URL` | Publicación de eventos de disponibilidad |

## Escalado, salud y observabilidad
Stateless; `GET /health`; la restricción `EXCLUDE` anti-solape de reservas se garantiza en BD (no en la app); publica eventos vía Outbox.

## Contrato
Ver [contract.md](./contract.md) · [data-model.md](../../data-model.md)
