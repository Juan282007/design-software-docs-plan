<!-- RESUMEN-EJECUTIVO
agente: Jesús Ariel González Bonilla (PM + Arquitecto)
capacidad: contrato de API (OpenAPI 3.1)
fase: diseño (api-first)
estado: accepted
dependencias_entrada: 09-microservices/services/07-document-service/data-model.md, 07-api/guidelines.md, 07-api/contracts/openapi/_shared.yaml
consumidores_siguientes: backend document-service, pdf-renderer-worker (resolución por code), frontend (gestión de plantillas), pruebas de contrato
tldr: CRUD de document_template (HTML/Handlebars) con paginación offset, resolución id-o-code para workers, acción de previsualización de renderizado y reporte de uso de plantillas. La fuente de verdad es template.yaml.
decisiones_clave: OpenAPI publicable en 07-api/contracts/openapi/template.yaml; GET /document-templates/{id} acepta UUID o code (evita ruta by-code ambigua con /preview); preview no persiste document ni document_version (dominio de document-api); reporte agrega document_template + document dentro del propio servicio
halts_registrados: ninguno
-->

# Contrato — template-api

> Estado: 🟢 Aceptado | Última actualización: 2026-08-06
> Autor: Jesús Ariel González Bonilla (PM + Arquitecto) | Equipo: Arquitectura

> **Fuente de verdad (normativa):** el spec OpenAPI 3.1 en
> [`07-api/contracts/openapi/template.yaml`](../../../../../07-api/contracts/openapi/template.yaml).
> Este documento es la **narrativa** que lo explica; ante cualquier diferencia, **manda el
> `template.yaml`**. Convenciones transversales en
> [07-api/guidelines.md](../../../../../07-api/guidelines.md).

> **Alcance:** este contrato cubre únicamente `document_template`. El agregado `document` /
> `document_version` (generación y almacenamiento de documentos) es responsabilidad del
> componente hermano `document-api` (`07-api/contracts/openapi/document.yaml`); no se toca aquí.

## Base URL

`/api/v1`

## Autenticación

Bearer JWT emitido por `iam-service`. Todos los endpoints requieren el header:

```
Authorization: Bearer <token>
```

Autorización RBAC por `feature` (módulo `MOD_DOCUMENTS`, ver `rbac-design.md`), alineada al
esquema granular del módulo (como `document.yaml`): lectura de plantillas (listar/detalle) con
`DOC_VIEW`, escritura y previsualización con `DOC_TEMPLATE_MANAGE`, y el reporte de uso
con `DOC_REPORT_VIEW`.

## Entidad del contrato

Campos reales de `document_template` (ver [data-model.md](../../data-model.md)):

`id`, `code`, `name`, `template_body` (HTML/Handlebars), `output_type`, `version`, `is_active`.

- `code` es único (ej: `SCHEDULE_CERTIFICATE`, `ENROLLMENT_RECORD`).
- **Valores permitidos (`output_type`):** `PDF`, `EXCEL`, `WORD`.
- La representación de **lista** (`DocumentTemplateListItem`) omite `template_body` (TEXT, puede
  ser extenso); la representación de **detalle** (`DocumentTemplate`) sí la incluye.

## Endpoints

| Método | Path | Descripción |
|--------|------|-------------|
| `GET` | `/document-templates` | Lista paginada, filtrable por `output_type`, `code`, `is_active` |
| `POST` | `/document-templates` | Crea una plantilla (`version = 1`, `is_active = true`) |
| `GET` | `/document-templates/{id}` | Detalle; `id` acepta UUID **o** `code` (resolución usada por workers) |
| `PUT` | `/document-templates/{id}` | Actualiza; incrementa `document_template.version` |
| `DELETE` | `/document-templates/{id}` | Desactiva (soft delete vía `is_active = false`) |
| `POST` | `/document-templates/{id}/preview` | Renderiza `template_body` con datos de muestra, sin persistir nada |
| `GET` | `/reports/template-usage` | Reporte propio: uso de plantillas por volumen de `document` generados |

### Resolución por `id` o `code` — `GET /document-templates/{id}`

El contrato original preveía una ruta separada `GET /templates/by-code/{code}` para que los
workers (p. ej. `pdf-renderer-worker`) resolvieran la plantilla por su `code`. Esa ruta resultó
**estructuralmente ambigua** para el router frente a `POST /document-templates/{id}/preview`
(ambas de 3 segmentos, una con literal en la posición 2 y otra con literal en la posición 3;
Redocly `no-ambiguous-paths` lo marca como error de diseño). En vez de forzar un segmento
adicional artificial, se decidió que el mismo path param `id` acepte **UUID o `code`**: un único
endpoint, sin ambigüedad de ruteo, y el mismo resultado funcional para los workers.

### Previsualización — `POST /document-templates/{id}/preview`

Acción de sub-recurso (no CRUD, guidelines §3). Resuelve el Handlebars de `template_body` contra
un `sample_data` arbitrario y devuelve el HTML resultante (`rendered_html`) más los placeholders
sin resolver (`missing_placeholders`), si los hay. **No** crea filas en `document` ni
`document_version` — la generación real y la conversión a PDF/EXCEL/WORD las realiza
`document-api` + `pdf-renderer-worker`.

## Crear plantilla — `POST /document-templates`

**Request:**

```json
{
  "code": "ENROLLMENT_RECORD",
  "name": "Comprobante de matrícula",
  "template_body": "<html>...{{ variables }}...</html>",
  "output_type": "PDF"
}
```

**Response — `201 Created`:** el recurso creado con `version = 1` e `is_active = true`.

## Versionado

`PUT /document-templates/{id}` incrementa `document_template.version`. El versionado de plantilla
es independiente del versionado de documentos generados (`document_version`, propiedad de
`document-api`). Los documentos ya generados conservan su binario; solo las generaciones
posteriores usan la nueva versión de la plantilla. `code` es inmutable tras la creación (no forma
parte del payload de `PUT`), porque los workers lo usan como clave de resolución estable.

## Paginación, filtrado y errores

Sigue las convenciones transversales de
[07-api/guidelines.md](../../../../../07-api/guidelines.md) §6 y §7: paginación por offset
(`page`/`page_size`, envelope `{ data, pagination }`), filtros explícitos por query param, y el
envelope de error estándar (`{ error: { code, message, details, trace_id } }`) vía
`_shared.yaml#/components/schemas/Error`. Cada operación documenta en `template.yaml` sus
respuestas de error concretas (`400/401/403/404/409/422` según aplique).

## Reportes (dominio propio)

Conforme a guidelines §11 (reportes descentralizados, sin hub central), `template-api` expone y
responde por su propio reporte:

| Reporte | Endpoint | Fuente | Frecuencia | Formato | Consumidores |
|---------|----------|--------|------------|---------|--------------|
| Uso de plantillas | `GET /reports/template-usage` | `document_template` + `document` (mismo servicio; agregación por `template_id`, sin cruzar límites de servicio) | On-demand | JSON (paginado, offset — acotado por el número de plantillas) | Coordinadores/admin (panel de documentos); `monitoring-service` si publica un KPI propio derivado |

Devuelve por plantilla: `template_id`, `template_code`, `template_name`, `output_type`,
`total_documents`, `last_generated_at` y `status_breakdown` (conteo por `document.status`).
Filtrable por `template_id`, `output_type` y rango `from`/`to` sobre `document.created_at`.
Solo lectura; nunca muta estado.
