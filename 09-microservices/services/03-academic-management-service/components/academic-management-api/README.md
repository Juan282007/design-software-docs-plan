# academic-management-api

> Estado: 🟡 En progreso | Última actualización: 2026-08-01
> Autor: Jesús Ariel González Bonilla (PM + Arquitecto) | Equipo: Arquitectura

> **Diseño previsto — no implementado.** Agnóstico de lenguaje: sirve para cualquier backend.

## Tipo de componente
`-api` — REST API sincrónica.

## Responsabilidad
Exponer la gestión de programas de formación, competencias, resultados de aprendizaje y fichas de matrícula.

## Tecnologías
| Capa | Tecnología |
|------|-----------|
| Runtime | Agnóstico — cualquier backend (a definir) |
| Framework | A definir |
| Persistencia | PostgreSQL 16 · schema `academic_management` |
| Autenticación | JWT de `iam-service` (RBAC feature + scope) |

## Variables de entorno (genéricas)
| Variable | Descripción |
|----------|-------------|
| `SERVICE_PORT` | Puerto de escucha |
| `DB_URL` | Conexión a PostgreSQL |
| `JWT_PUBLIC_KEY` | Verificación del token |
| `BROKER_URL` | Publicación de eventos de dominio |

## Escalado, salud y observabilidad
Stateless (escala por réplicas); `GET /health`; logs con `request_id`; publica eventos de dominio de forma transaccional (patrón Outbox, [communication-patterns.md](../../../../communication-patterns.md)).

## Contrato
Ver [contract.md](./contract.md) · [data-model.md](../../data-model.md)
