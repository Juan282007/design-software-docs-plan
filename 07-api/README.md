# API

> Estado: 🟡 Revisión | Última actualización: 2026-10-07
> Autor: Kevin Andrey Culma (Arquitecto) | Equipo: Arquitectura.

## Contenido

Define lineamientos de API, autenticación y ubicación de contratos OpenAPI.

> ⚠️ Las APIs de los servicios **aún no están implementadas**: hoy sólo existe la capa de datos (repos `*-db`). Esta carpeta contiene el estándar de diseño que cada servicio deberá cumplir al construir su capa de aplicación. Ver plan por fases en [technical-backlog.md](../15-project-control/technical-backlog.md).

## Archivos

| Archivo | Descripción | Estado |
|---------|-------------|--------|
| [guidelines.md](./guidelines.md) | Convenciones de diseño de APIs | 🟡 |
| [authentication.md](./authentication.md) | Estrategia de autenticación y autorización | 🟡 |
| [api-surface-map.md](./api-surface-map.md) | Inventario de endpoints detectados por servicio | 🟡 |
| [openapi-contract-review.md](./openapi-contract-review.md) | Revisión de cobertura entre código y contratos OpenAPI | 🟡 |
| [contracts/openapi/](./contracts/openapi/) | Contratos OpenAPI validados y publicables | 🔴 |

## Dónde va cada contrato

| Tipo de contrato | Dónde vive | Cuándo se usa |
|-----------------|------------|---------------|
| Contrato de implementación (borrador, en evolución) | `09-microservices/services/<svc>/api-contract.md` | Durante el desarrollo del servicio |
| Contrato OpenAPI publicable (estable, revisado) | `07-api/contracts/openapi/<svc>.yaml` | Cuando el contrato es estable y puede compartirse con consumidores externos |

**Regla:** el contrato en `07-api/contracts/openapi/` es la versión aprobada por arquitectura. Si hay diferencia entre ambos, el de `07-api/` tiene precedencia para consumidores externos.

## Plantillas

| Plantilla | Descripción |
|-----------|-------------|
| [_template-api-contract.md](./_template-api-contract.md) | Contrato REST: autenticación, endpoints CRUD, errores, rate limiting y versionado |
