<!-- RESUMEN-EJECUTIVO
agente: Jesús Ariel González Bonilla (PM + Arquitecto)
capacidad: contrato de API (OpenAPI 3.1)
fase: diseño (api-first)
estado: accepted
dependencias_entrada: 09-microservices/services/07-document-service/data-model.md, 07-api/guidelines.md, 07-api/contracts/openapi/_shared.yaml
consumidores_siguientes: backend document-service, pdf-renderer-worker, servicios propietarios (scheduling/ficha/certificate/actors), frontend
tldr: CRUD de documentos + versiones (multipart upload), generación asíncrona desde plantilla, descarga por URL firmada y reporte de inventario por estado. La fuente de verdad es document.yaml.
decisiones_clave: OpenAPI publicable en 07-api/contracts/openapi/document.yaml; storage_key nunca se expone al cliente (solo writeOnly en creación); versiones inmutables (solo GET/POST); reporte con paginación por offset (inventario acotado, no log continuo)
halts_registrados: ninguno
-->

# Contrato — document-api

> Estado: 🟢 Aceptado | Última actualización: 2026-08-06
> Autor: Jesús Ariel González Bonilla (PM + Arquitecto) | Equipo: Arquitectura

> **Fuente de verdad (normativa):** el spec OpenAPI 3.1 en
> [`07-api/contracts/openapi/document.yaml`](../../../../../07-api/contracts/openapi/document.yaml).
> Este documento es la **narrativa** que lo explica; ante cualquier diferencia, **manda el
> `document.yaml`**. Convenciones transversales en
> [07-api/guidelines.md](../../../../../07-api/guidelines.md).

> **Parcialmente implementado** en `design-software-document-service` (HU-DOC-002 y HU-DOC-003): `POST /documents/generate`, `GET /documents`, `GET /documents/{id}`, versiones y `download-url`. Siguen como diseño previsto `POST /documents`, `PATCH` y `DELETE` de documentos, la carga de versiones y `/reports/document-status`. Contrato a nivel de protocolo (REST/JSON). Sin
> código de ningún lenguaje. Las entidades y campos son los reales de [data-model.md](../../data-model.md).

## Autenticación

Bearer JWT emitido por `iam-service`. Todos los endpoints requieren el header:

```
Authorization: Bearer <token>
```

La autorización se resuelve por scope/rol del token. Roles referenciados: `coordinador`,
`admin`, `sistema` (servicio a servicio). `*` = cualquier usuario autenticado con acceso a la
entidad propietaria.

El acceso a los datos se resuelve por el scope de los pares `FEATURE:SCOPE` del token: `GLOBAL` ve todos
los documentos y `TRAINING_CENTER` solo los de su centro de formación (claim `training_center_id`, que
`POST /documents/generate` guarda en `document.training_center_id`). Un documento fuera del scope responde
`404`, igual que uno inexistente; un feature o scope que no concede acceso a documentos responde `403`.
Features: `DOC_VIEW` (consulta), `DOC_CREATE` (generar/registrar) y `DOC_DOWNLOAD` (URL de descarga).

## Base URL

`/api/v1/documents`

## Endpoints

| Método | Path | Descripción | Roles |
|--------|------|-------------|-------|
| `GET` | `/documents` | Listar documentos (paginado, offset), filtrable por `owner_service`, `owner_entity_id`, `domain`, `status` | `*` |
| `POST` | `/documents` | Registra un documento cuyo binario ya fue subido al object storage (metadatos + `storage_key`) | `coordinador`, `admin` |
| `POST` | `/documents/generate` | Solicita la generación asíncrona de un documento a partir de una plantilla | `coordinador`, `sistema` |
| `GET` | `/documents/{id}` | Metadatos de un documento | `*` |
| `PATCH` | `/documents/{id}` | Actualiza `title`/`domain`; requiere `row_version` (bloqueo optimista) | `coordinador`, `admin` |
| `DELETE` | `/documents/{id}` | Archiva el documento (`status = ARCHIVED`, soft) | `admin` |
| `GET` | `/documents/{id}/download-url` | Genera y retorna una URL firmada de la versión vigente | `*` |
| `GET` | `/documents/{id}/versions` | Historial de versiones (`document_version`) | `*` |
| `POST` | `/documents/{id}/versions` | Sube el binario de una nueva versión (`multipart/form-data`) | `coordinador`, `admin`, `sistema` |
| `GET` | `/documents/{id}/versions/{version_id}` | Detalle de una versión puntual | `*` |
| `GET` | `/reports/document-status` | Inventario de documentos por entidad/dominio/estado (paginado, offset) | `coordinador`, `admin` |

Las versiones son **inmutables/append-only** (ver convenciones de modelado en data-model.md):
no existe `PUT`/`PATCH`/`DELETE` sobre `document_version`.

## Entidades del contrato

Campos reales de [data-model.md](../../data-model.md):

- `document`: `id`, `template_id?`, `title`, `domain`, `owner_service`, `owner_entity_id`,
  `storage_key`, `mime_type`, `size_bytes?`, `status`, `row_version`, `created_by`, `created_at`,
  `updated_at`.
- `document_version`: `id`, `document_id`, `version_number`, `storage_key`, `created_by`,
  `created_at`, `notes?`.

`storage_key` es de **solo escritura** en el contrato: se acepta al registrar un documento
(`POST /documents`) pero **nunca se retorna** en ninguna respuesta; la descarga se resuelve
siempre vía `GET /documents/{id}/download-url`.

`owner_entity_id` y `created_by` son referencias lógicas cross-service (a `owner_service` y a
`iam-service` respectivamente): `UUID` sin FK física ni endpoint de join.

**Valores permitidos:**

- `domain`: `SCHEDULE`, `FICHA`, `CERTIFICATE`, `ACTOR`, `REPORT`.
- `status`: `GENERATING`, `AVAILABLE`, `ARCHIVED`, `EXPIRED`, `GENERATION_FAILED`.

`mime_type` y el archivo asociado no existen mientras `status` sea `GENERATING` o `GENERATION_FAILED`: `mime_type`
es `null` en esos estados. El listado `GET /documents` admite `sort=<campo>:<asc|desc>` con `created_at`,
`updated_at`, `title`, `status`, `domain` y `owner_service` (por defecto `created_at:desc`; otro valor responde
`400 INVALID_SORT`). `GET /documents/{id}/versions` retorna un arreglo ordenado por `version_number` descendente.

## Carga de nueva versión — `POST /documents/{id}/versions`

`Content-Type: multipart/form-data` (guidelines §1, excepción explícita para document-service).
Partes:

| Parte | Tipo | Requerido | Descripción |
|-------|------|-----------|-------------|
| `file` | binario | Sí | Contenido de la nueva versión. |
| `notes` | texto | No | Observaciones sobre esta versión. |

El servicio escribe el binario en object storage, crea el `document_version`
(`version_number` autoincremental) y actualiza `document.storage_key`/`status = AVAILABLE`
para apuntar a la nueva versión (desnormalización intencional, ver data-model.md).

## Solicitud de generación — `POST /documents/generate`

**Request:**

```json
{
  "template_code": "SCHEDULE_CERTIFICATE",
  "domain": "SCHEDULE",
  "owner_service": "scheduling-service",
  "owner_entity_id": "5c1a...-uuid",
  "title": "Horario grupo 2026-1.pdf",
  "data": {}
}
```

- `template_code` referencia `document_template.code`. `data` es el contexto que el
  [pdf-renderer-worker](../pdf-renderer-worker/README.md) combina con la plantilla.

**Response — `202 Accepted`:**

```json
{
  "document_id": "8f3c...-uuid",
  "status": "GENERATING"
}
```

El endpoint crea el registro `document` con `status = GENERATING`, encola la solicitud en
`document-generation-queue` y retorna de inmediato. El cliente consulta `GET /documents/{id}`
hasta que `status = AVAILABLE` (o `GENERATION_FAILED`). La generación es **siempre asíncrona**
(decisión 02 de [decisions.md](../../decisions.md)).

- `data` es opcional (por defecto `{}`).
- Si no existe una plantilla **activa** para `template_code` responde `404` con code `TEMPLATE_NOT_FOUND` y no
  se encola ningún mensaje.
- Header `X-Correlation-Id` (opcional): se propaga como `correlation_id` al mensaje de `document-generation-queue`
  (`{ document_id, template_code, data_payload, correlation_id }`) y se devuelve en la respuesta. Si no llega,
  el servicio genera uno.
- Si el broker no acepta el mensaje responde `503` con code `GENERATION_QUEUE_UNAVAILABLE` y el documento queda
  en `GENERATION_FAILED`.

## Descarga — `GET /documents/{id}/download-url`

**Response — `200 OK`:**

```json
{
  "document_id": "8f3c...-uuid",
  "url": "https://<storage>/documents/SCHEDULE/2026/06/.../8f3c.pdf?X-Amz-...",
  "expires_in": 300
}
```

La URL firmada se genera solo tras validar el scope del usuario. TTL por defecto **300 s**
(5 min, [ADR-003](../../../../../05-architecture/decisions/records/ADR-003-object-storage.md)).
Nunca se expone `storage_key` al cliente. Requiere `status = AVAILABLE`; en otro caso se
responde `409 Conflict` con code `DOCUMENT_NOT_AVAILABLE`. Si el object storage no permite firmar la URL
responde `503` con code `STORAGE_UNAVAILABLE`.

## Reportes (dominio propio)

`GET /reports/document-status` — inventario de documentos por entidad propietaria, dominio y
estado (guidelines §11: reportes descentralizados, sin hub central; cada servicio expone y
responde por los suyos).

| Campo | Valor |
|-------|-------|
| Usuarios | Servicios propietarios (SCHEDULE, FICHA, CERTIFICATE, ACTOR, REPORT) vía integración; coordinadores/administradores vía UI |
| Frecuencia | On-demand |
| Formato | JSON |
| Paginación | Offset (`page`/`page_size`); es un inventario acotado, no un log continuo (a diferencia de `login-audit` en iam-service, que sí usa cursor) |
| Filtros | `domain`, `status`, `owner_service`, `owner_entity_id`, `from`/`to` (rango de `created_at`) |
| Fuente de datos | `document` (`document_db`) |
| Mutación | Ninguna — solo lectura |

## Formato de error estándar

Envelope uniforme de la plataforma (guidelines §7; ver `Error` en
[`_shared.yaml`](../../../../../07-api/contracts/openapi/_shared.yaml)):

```json
{
  "error": {
    "code": "DOCUMENT_NOT_FOUND",
    "message": "El documento solicitado no existe",
    "details": [],
    "trace_id": "b3f1c2a4-...-uuid"
  }
}
```

| `error.code` | HTTP | Caso |
|--------------|------|------|
| `DOCUMENT_NOT_FOUND` | 404 | `id` (o `version_id`) inexistente |
| `DOCUMENT_NOT_AVAILABLE` | 409 | Descarga solicitada con `status` ≠ `AVAILABLE` |
| `ROW_VERSION_MISMATCH` | 409 | `row_version` de `PATCH /documents/{id}` no coincide con el actual |
| `TEMPLATE_NOT_FOUND` | 422 | `template_code` inválido en la solicitud de generación |
| `FORBIDDEN` | 403 | El token no tiene scope sobre la entidad propietaria |
| `UNAUTHORIZED` | 401 | JWT ausente o inválido |
