# Modelo de datos — reference-data-service

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

### Jerarquía institucional (M2)

`macroregion` → `microregion` → `department` → `municipality` → `training_center` → `institutional_unit`

| Entidad | PK | FK padre | Campos clave |
|---------|----|-----------|----|
| `macroregion` | UUID | — | `name`, `code` |
| `microregion` | UUID | `macroregion_id` | `name`, `code` |
| `department` | UUID | `microregion_id` | `name`, `dane_code` |
| `municipality` | UUID | `department_id` | `name`, `dane_code` |
| `training_center` | UUID | `municipality_id` | `name`, `center_code`, `is_active` |
| `institutional_unit` | UUID | `training_center_id` | `name`, `unit_type`, `is_active` |

### `macroregion` — detalle

| Campo | Tipo | Nullable | Descripción |
|-------|------|----------|-------------|
| `id` | UUID | No | PK |
| `name` | VARCHAR(100) | No | Nombre de la macrorregión |
| `code` | VARCHAR(10) | No | UNIQUE; código SENA |
| `is_active` | BOOLEAN | No | DEFAULT true |

### `microregion` — detalle

| Campo | Tipo | Nullable | Descripción |
|-------|------|----------|-------------|
| `id` | UUID | No | PK |
| `macroregion_id` | UUID | No | FK → macroregion |
| `name` | VARCHAR(100) | No | |
| `code` | VARCHAR(10) | No | UNIQUE NOT NULL |
| `is_active` | BOOLEAN | No | DEFAULT true |

### `department` — detalle

| Campo | Tipo | Nullable | Descripción |
|-------|------|----------|-------------|
| `id` | UUID | No | PK |
| `microregion_id` | UUID | No | FK → microregion |
| `name` | VARCHAR(100) | No | |
| `dane_code` | VARCHAR(5) | No | UNIQUE; código DANE del departamento |
| `is_active` | BOOLEAN | No | DEFAULT true |

### `municipality` — detalle

| Campo | Tipo | Nullable | Descripción |
|-------|------|----------|-------------|
| `id` | UUID | No | PK |
| `department_id` | UUID | No | FK → department |
| `name` | VARCHAR(150) | No | |
| `dane_code` | VARCHAR(8) | No | UNIQUE; código DANE del municipio |
| `is_active` | BOOLEAN | No | DEFAULT true |

### `training_center` — detalle

| Campo | Tipo | Nullable | Descripción |
|-------|------|----------|-------------|
| `id` | UUID | No | PK |
| `municipality_id` | UUID | No | FK → municipality |
| `center_code` | VARCHAR(10) | No | Código SENA del centro |
| `name` | VARCHAR(200) | No | |
| `address` | TEXT | Sí | |
| `phone` | VARCHAR(20) | Sí | PII |
| `is_active` | BOOLEAN | No | |
| `created_at` | TIMESTAMPTZ | No | Fecha de creación |
| `updated_at` | TIMESTAMPTZ | No | Fecha de última actualización (editable) |

### `institutional_unit` — detalle

| Campo | Tipo | Nullable | Descripción |
|-------|------|----------|-------------|
| `id` | UUID | No | PK |
| `training_center_id` | UUID | No | FK → training_center |
| `name` | VARCHAR(200) | No | |
| `unit_type` | VARCHAR(30) | No | CHECK (`unit_type IN ('ACADEMIC','ADMINISTRATIVE','MIXED')`) |
| `is_active` | BOOLEAN | No | DEFAULT true |
| `created_at` | TIMESTAMPTZ | No | Fecha de creación |

### Catálogos (M4)

### `catalog`

| Campo | Tipo | Nullable | Descripción |
|-------|------|----------|-------------|
| `id` | UUID | No | PK |
| `code` | VARCHAR(50) | No | Clave del catálogo (ej: `MODALITY`, `SHIFT`) |
| `name` | VARCHAR(100) | No | Nombre para visualización |
| `description` | TEXT | Sí | |
| `is_active` | BOOLEAN | No | |
| `created_at` | TIMESTAMPTZ | No | Fecha de creación |

### `catalog_detail`

| Campo | Tipo | Nullable | Descripción |
|-------|------|----------|-------------|
| `id` | UUID | No | PK |
| `catalog_id` | UUID | No | FK → catalog |
| `code` | VARCHAR(50) | No | Clave del valor (ej: `IN_PERSON`) |
| `label` | VARCHAR(255) | No | Etiqueta de presentación |
| `display_order` | INT | Sí | Orden en UI |
| `is_active` | BOOLEAN | No | |
| `created_at` | TIMESTAMPTZ | No | Fecha de creación |

### `parameter`

Patrón EAV de configuración (entidad-atributo-valor): cada fila es un parámetro identificado por `key`, cuyo `value` se almacena como string y se valida según `value_type` en la capa de aplicación.

| Campo | Tipo | Nullable | Descripción |
|-------|------|----------|-------------|
| `id` | UUID | No | PK |
| `key` | VARCHAR(100) | No | Único (ej: `MAX_HOURS_PER_WEEK`) |
| `value` | TEXT | No | Valor como string; validado según `value_type` en la capa de aplicación |
| `value_type` | VARCHAR(20) | No | CHECK (`value_type IN ('integer','string','boolean','json')`) |
| `description` | TEXT | Sí | |
| `created_at` | TIMESTAMPTZ | No | Fecha de creación |

## Índices relevantes

| Tabla | Campos | Tipo | Motivo |
|-------|--------|------|--------|
| `macroregion` | `code` | UNIQUE | Referencia desde otros servicios |
| `microregion` | `(macroregion_id, code)` | UNIQUE | Código único dentro de macrorregión |
| `department` | `dane_code` | UNIQUE | Código DANE único nacional |
| `department` | `(microregion_id, name)` | UNIQUE | Nombre único dentro de la microrregión padre |
| `municipality` | `dane_code` | UNIQUE | Código DANE único nacional |
| `municipality` | `(department_id, name)` | UNIQUE | Nombre único dentro del departamento padre |
| `training_center` | `center_code` | UNIQUE | Referencia desde otros servicios |
| `catalog` | `code` | UNIQUE | Consulta por código de catálogo |
| `catalog_detail` | `(catalog_id, code)` | UNIQUE | Valor único por catálogo |
| `parameter` | `key` | UNIQUE | Consulta por clave de parámetro |

## Remediación de congruencia — R3 (2026-08-06)

`institutional_unit.unit_type` y `parameter.value_type` estaban implementados como **ENUM
nativo de Postgres**, lo que **viola ADR-004** (el `ALTER TYPE` bloquea migraciones
independientes por servicio). Se convierten a **`VARCHAR + CHECK`** vía changeset nuevo
`remediation-reference-960` (no se modifica ningún changeset corrido), conservando el mismo
conjunto de valores: `unit_type IN ('ACADEMIC','ADMINISTRATIVE','MIXED')`,
`value_type IN ('integer','string','boolean','json')`.
