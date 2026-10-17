# actors-api

> Estado: 🟡 En progreso | Última actualización: 2026-08-01
> Autor: Jesús Ariel González Bonilla (PM + Arquitecto) | Equipo: Arquitectura

> **Diseño previsto — no implementado.** Agnóstico de lenguaje.

## Tipo de componente
`-api` — REST API sincrónica.

## Responsabilidad
Exponer la gestión de instructores, aprendices y empresas, sus contratos, áreas de conocimiento y disponibilidad.

## Tecnologías
| Capa | Tecnología |
|------|-----------|
| Runtime | Agnóstico — cualquier backend (a definir) |
| Framework | A definir |
| Persistencia | PostgreSQL 16 · schema `actors_parameterization` |
| Autenticación | JWT de `iam-service` |

## Variables de entorno (genéricas)
| Variable | Descripción |
|----------|-------------|
| `SERVICE_PORT` | Puerto de escucha |
| `DB_URL` | Conexión a PostgreSQL |
| `JWT_PUBLIC_KEY` | Verificación del token |
| `BROKER_URL` | Publicación de eventos |

## Escalado, salud y observabilidad
Stateless; `GET /health`; la disponibilidad del instructor es insumo del motor de horarios → publica `actors.instructor.availability_changed` vía Outbox.

## Contrato
Ver [contract.md](./contract.md) · [data-model.md](../../data-model.md)
