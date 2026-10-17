# Modelo de datos — document-service

> Estado: 🟢 Estable | Última actualización: 2026-06-20
> Naming: entidades y atributos en inglés (HALT-DB-NAMING)

## Convenciones de modelado (transversales)

> **Estándar autoritativo:** [06-data/modeling-conventions.md](../../../06-data/modeling-conventions.md) (ratificado en ADR-004). Resumen aplicable:

- **Tres conceptos de estado (no confundir):** (1) *ciclo de vida del registro* (técnico) → `is_active` + soft delete (`deleted_at`); (2) *estado de negocio* (parametrizable) → FK `status_id` → catálogo `status`; (3) *enum técnico cerrado* (CC/CE, EMAIL/IN_APP…) → `VARCHAR` + `CHECK (... IN (...))`. **No se usa el tipo `ENUM` nativo de Postgres** (su `ALTER TYPE` bloquea migraciones independientes por servicio).
- **Estados de negocio parametrizables:** el servicio implementa `status_category` + `status` (+ `status_transition` si aplica) en su propia BD; los agregados con máquina de estados referencian `status_id`. Solo migran estados de **negocio**; los enums técnicos cerrados (`*_type`, `channel`…) permanecen como `VARCHAR + CHECK`.
- **Auditoría (tablas transaccionales):** `created_at`/`created_by`, `updated_at`/`updated_by`, `deleted_at`/`deleted_by` (soft delete), `is_active` y `row_version`. Acciones del sistema usan `SYSTEM_ACTOR_ID = 00000000-0000-0000-0000-000000000000`. Catálogos: `created_*`/`updated_*` + `is_active` (sin soft delete). Append-only: solo el timestamp de inserción.
- **Acciones referenciales:** cada FK declara `ON UPDATE`/`ON DELETE`. Por defecto: catálogo/padre → `RESTRICT`; hijo de agregado (composición) → `CASCADE`; FK opcional → `SET NULL`.
- **Nomenclatura de constraints:** `pk_<tabla>`, `uq_<tabla>_<cols>`, `fk_<tabla>_<ref>`, `ck_<tabla>_<regla>`, `ix_<tabla>_<cols>`.

## Entidades propias

### `document_template`

Plantilla para generación de documentos (HTML/Handlebars).

| Campo | Tipo | Nullable | Descripción |
|-------|------|----------|-------------|
| `id` | UUID | No | PK |
| `code` | VARCHAR(50) | No | Único (ej: `SCHEDULE_CERTIFICATE`, `ENROLLMENT_RECORD`) |
| `name` | VARCHAR(200) | No | |
| `template_body` | TEXT | No | Cuerpo de la plantilla (HTML) |
| `output_type` | VARCHAR(10) | No | PDF / EXCEL / WORD |
| `version` | INT | No | Versión de la plantilla |
| `is_active` | BOOLEAN | No | |

**Valores permitidos (`output_type`):** PDF, EXCEL, WORD. Enforced via CHECK constraint: CHECK (output_type IN ('PDF','EXCEL','WORD')).

### `document`

Documento generado en el sistema. Los binarios NO se almacenan aquí.

| Campo | Tipo | Nullable | Descripción |
|-------|------|----------|-------------|
| `id` | UUID | No | PK |
| `template_id` | UUID | Sí | FK → document_template (null si doc libre) |
| `title` | VARCHAR(300) | No | Nombre del archivo |
| `domain` | VARCHAR(50) | No | Contexto: `SCHEDULE`, `FICHA`, `CERTIFICATE`, `ACTOR`, `REPORT` |
| `owner_service` | VARCHAR(100) | No | Servicio solicitante |
| `owner_entity_id` | UUID | No | ID de la entidad de negocio que lo originó |
| `storage_key` | TEXT | Sí | Ruta en object storage (S3/MinIO). Null mientras `status` sea `GENERATING` o `GENERATION_FAILED` |
| `mime_type` | VARCHAR(100) | Sí | `application/pdf`, etc. Null mientras `status` sea `GENERATING` o `GENERATION_FAILED` |
| `size_bytes` | BIGINT | Sí | |
| `status` | VARCHAR(20) | No | GENERATING / AVAILABLE / ARCHIVED / EXPIRED / GENERATION_FAILED |
| `row_version` | INT | No | Bloqueo optimista (DEFAULT 0) |
| `created_by` | UUID | No | ID de usuario (iam-service) |
| `created_at` | TIMESTAMPTZ | No | |
| `updated_at` | TIMESTAMPTZ NOT NULL | No | |
| `training_center_id` | UUID | Sí | Centro de formación del solicitante (claim `training_center_id` del JWT, que `POST /documents/generate` guarda). Define el acceso con scope `TRAINING_CENTER`; null si el token no lo trae (solo lo ve el scope `GLOBAL`). Referencia lógica cross-service, sin FK |

**Invariante**: Los archivos binarios **nunca** se almacenan en `document_db`. Solo `storage_key` referencia el archivo en object storage.

**Valores permitidos (`domain`):** SCHEDULE, FICHA, CERTIFICATE, ACTOR, REPORT. Enforced via CHECK constraint: CHECK (domain IN ('SCHEDULE','FICHA','CERTIFICATE','ACTOR','REPORT')).

**Valores permitidos (`status`):** GENERATING, AVAILABLE, ARCHIVED, EXPIRED, GENERATION_FAILED. Enforced via CHECK constraint: CHECK (status IN ('GENERATING','AVAILABLE','ARCHIVED','EXPIRED','GENERATION_FAILED')). El estado `GENERATION_FAILED` es coherente con el manejo de fallos del pdf-renderer-worker.

**CHECK (`size_bytes`):** CHECK (size_bytes >= 0).

**CHECK (`storage_key` / `mime_type`):** el documento se persiste en `GENERATING` antes de que exista el archivo, por lo que ambos campos son nullable. `ck_document_storage_when_available`: CHECK (status IN ('GENERATING','GENERATION_FAILED') OR (storage_key IS NOT NULL AND mime_type IS NOT NULL)). Changeset `hu-doc-002-document-generating-nullable-storage-972`.

**Relación `storage_key` ↔ `document_version`:** `document.storage_key` apunta siempre a la versión vigente; `document_version` guarda el historial. Es una desnormalización intencional para evitar un JOIN al leer la versión actual (ver decisions.md).

### `document_version`

Historial de versiones de un documento.

| Campo | Tipo | Nullable | Descripción |
|-------|------|----------|-------------|
| `id` | UUID | No | PK |
| `document_id` | UUID | No | FK → document |
| `version_number` | INT | No | Autoincremental por documento |
| `storage_key` | TEXT | No | Clave de esta versión en object storage |
| `created_by` | UUID | No | ID de usuario (iam-service) |
| `created_at` | TIMESTAMPTZ | No | |
| `notes` | TEXT | Sí | Observaciones sobre esta versión |
| `correlation_id` | VARCHAR(100) | Sí | `correlation_id` del mensaje de generación que creó la versión; idempotencia del `pdf-renderer-worker` |

## Notas

- Ver estrategia de almacenamiento en [storage-adapters.md](./storage-adapters.md).
- MinIO en DEV/QA; S3 en producción. Ver ADR-003.

## Índices relevantes

| Tabla | Campos | Tipo | Motivo |
|-------|--------|------|--------|
| `document_template` | `code` | UNIQUE | Lookup por código de plantilla |
| `document` | `(owner_service, owner_entity_id)` | B-Tree | Documentos por entidad de negocio |
| `document` | `(domain, status)` | B-Tree | Filtros por dominio y estado (`ix_document_domain_status`) |
| `document` | `training_center_id` | B-Tree | Acceso por scope `TRAINING_CENTER` (`ix_document_training_center`) |
| `document_version` | `(document_id, version_number)` | UNIQUE | Versión única por documento |
| `document_version` | `(document_id, correlation_id)` WHERE `correlation_id IS NOT NULL` | UNIQUE parcial | Idempotencia del worker (`uq_document_version_correlation`) |

## Cambios por HU-DOC-002, HU-DOC-003 y HU-DOC-004 (2026-10)

- `hu-doc-002-document-generating-nullable-storage-972`: `storage_key` y `mime_type` nullable con el CHECK `ck_document_storage_when_available`.
- `hu-doc-003-document-access-and-filters-973`: columna `document.training_center_id` e índices `ix_document_owner` (`owner_service`, `owner_entity_id`), `ix_document_training_center` y `ix_document_domain_status`.
- `hu-doc-004-version-idempotency-974`: columna `document_version.correlation_id` e índice único parcial `uq_document_version_correlation`.

## Remediación de congruencia — R3 (2026-08-06)

Cierre de brechas contrato↔BD (la BD no tenía **ningún** CHECK). Vía changeset nuevo
`remediation-document-960` (no se modifica ningún changeset corrido; se preserva el trabajo previo):
- **CHECKs** documentados ahora implementados: `output_type IN ('PDF','EXCEL','WORD')`,
  `domain IN ('SCHEDULE','FICHA','CERTIFICATE','ACTOR','REPORT')`,
  `status IN ('GENERATING','AVAILABLE','ARCHIVED','EXPIRED','GENERATION_FAILED')`, `size_bytes >= 0`.
- `document_template.is_active BOOLEAN NOT NULL DEFAULT true` (exigido por el contrato; complementa a `state`).
- `created_by`/`updated_by`: `VARCHAR(100)` → `UUID` (referencia lógica a iam-service) en las 3 tablas.
