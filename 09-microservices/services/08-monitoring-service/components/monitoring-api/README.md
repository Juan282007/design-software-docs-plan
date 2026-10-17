# monitoring-api

> Estado: 🟡 En progreso | Última actualización: 2026-08-26
> Stack: Python 3.11+ / FastAPI

## Tipo de componente

`-api` — API REST del bounded context M9.

## Responsabilidad

Expone KPIs, alertas, seguimiento, sesiones y planes de mejoramiento. **No expone endpoints de notificaciones.**

## Estado de implementación

El repo actual incluye `GET /health` y un caso de uso inicial `POST /kpis`. El OpenAPI completo representa el contrato objetivo a implementar incrementalmente.

## Tecnologías

| Capa | Tecnología |
|------|-----------|
| Runtime | Python 3.11+ |
| Framework | FastAPI |
| Validación | Pydantic |
| Persistencia | SQLAlchemy / PostgreSQL 16 |
| Contratos | OpenAPI / `shared-contracts` |

## Contrato

- [contract.md](./contract.md)
- [`monitoring.yaml`](../../../../../07-api/contracts/openapi/monitoring.yaml)
