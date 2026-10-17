# _template — Plantillas de microservicios

> Ver instrucciones completas en [microservices-documentation.md](../../00-governance/microservices-documentation.md)

Esta carpeta contiene dos tipos de plantillas:

| Carpeta | Usar para |
|---------|-----------|
| [`service/`](./service/) | Documentar un microservicio completo (bounded context, entidades, eventos, runbook, decisiones) |
| [`component/`](./component/) | Documentar un componente desplegable individual dentro de un servicio (-api, -worker, -workflow) |

## Cómo usar

1. Copiar `service/` → `services/<nn>-<nombre>-service/`
2. Para cada componente desplegable: copiar `component/` → `services/<nn>-<nombre>-service/components/<nombre>-<sufijo>/`
3. Completar cada archivo con contenido real; eliminar comentarios de ayuda
4. Registrar el servicio en [`service-catalog.md`](../service-catalog.md)

## Archivos deprecados (no usar)

Los siguientes archivos son reemplazados por la estructura `service/` + `component/`:

| Archivo | Estado |
|---------|--------|
| `api-contract.md` | ⚫ Deprecado |
| `data-model.md` | ⚫ Deprecado |
| `events.md` | ⚫ Deprecado |
| `runbook.md` | ⚫ Deprecado |
