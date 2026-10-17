# Microservicios

> Estado: 🟡 En progreso | Última actualización: 2026-08-26
> Autor: Equipo de Arquitectura

## Contenido

Centraliza el catálogo, reglas de frontera, contratos de integración, ownership y documentación por servicio.

## Topología vigente

- **9 módulos funcionales:** M1–M9.
- **10 microservicios backend desplegables.**
- `notification-service` es una capacidad transversal extraída de Monitoring por ADR-005.
- La existencia de un servicio transversal no implica un módulo funcional o MFE adicional.

## Estado de implementación

La madurez ya no es homogénea: existen repositorios de aplicación en varios servicios y algunos contienen skeletons/implementaciones parciales. Por ello el estado debe evaluarse **por servicio**, no con la afirmación global “solo existe capa de datos”.

## Archivos raíz

| Archivo | Descripción |
|---------|-------------|
| [service-catalog.md](./service-catalog.md) | Inventario canónico de los 10 microservicios. |
| [communication-patterns.md](./communication-patterns.md) | REST/eventos y resiliencia. |
| [dependency-map.md](./dependency-map.md) | Dependencias entre servicios. |
| [data-ownership-matrix.md](./data-ownership-matrix.md) | Propietario único de cada entidad. |
| [event-catalog.md](./event-catalog.md) | Eventos cross-service. |
| [service-boundary-rules.md](./service-boundary-rules.md) | Reglas de desacople. |
| [service-readiness-checklist.md](./service-readiness-checklist.md) | Madurez por servicio. |

## Servicios

| # | Servicio | Módulo(s) | Componentes principales |
|---|----------|-----------|-------------------------|
| 01 | [iam-service](./services/01-iam-service/) | M1 | `iam-api` |
| 02 | [reference-data-service](./services/02-reference-data-service/) | M2 + M4 | `reference-data-api` |
| 03 | [academic-management-service](./services/03-academic-management-service/) | M5 + M6 | `academic-management-api` |
| 04 | [training-environment-service](./services/04-training-environment-service/) | M3 | `training-environment-api` |
| 05 | [scheduling-service](./services/05-scheduling-service/) | M8 | `schedules-api`, workflow/worker según evolución |
| 06 | [actors-service](./services/06-actors-service/) | M7 | `actors-api` |
| 07 | [document-service](./services/07-document-service/) | transversal | APIs + workers documentales |
| 08 | [monitoring-service](./services/08-monitoring-service/) | M9 | `monitoring-api`, `alert-worker` |
| 09 | [audit-service](./services/09-audit-service/) | transversal | `audit-worker`, `audit-query-api` |
| 10 | [notification-service](./services/10-notification-service/) | transversal | `notification-api`, `notification-worker` |

> El antiguo `08-monitoring-service/components/notification-worker/` se retira. El worker vigente está en `10-notification-service`.
