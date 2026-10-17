# Convenciones de Modelado — Estándar transversal

> Fase: 03-Design | Agente: A10 / A13 | Estado: 🟡 Borrador
> Fecha: 2026-06-17
> Autoritativo: este documento es la **fuente única** de las convenciones transversales.
> El bloque "Convenciones de modelado" al inicio de cada `data-model.md` de servicio debe referenciar este estándar.
> Ratificado en [ADR-004](../05-architecture/decisions/records/ADR-004-status-parametrization-and-audit-standard.md).

Aplica a los 9 microservicios. Cada servicio **instancia** estos patrones en su propia base de datos (DB-por-servicio); no hay tablas compartidas entre servicios.

---

## 1. Tres conceptos de "estado" — no confundir

El sistema maneja tres nociones distintas que históricamente se mezclaban en un solo `is_active` o en ENUMs inline. Se separan explícitamente:

| Concepto | Qué representa | Cómo se modela |
|----------|----------------|----------------|
| **Ciclo de vida del registro** | ¿La fila existe y está habilitada? (técnico) | Columnas de auditoría: `is_active` + `deleted_at` (soft delete) |
| **Estado de negocio** | ¿En qué punto de su máquina de estados está el agregado? (DRAFT→PUBLISHED) | FK `status_id` → catálogo parametrizable `status` |
| **Enum técnico cerrado** | Conjunto fijo e inmutable por naturaleza (CC/CE, EMAIL/IN_APP) | `VARCHAR` + `CHECK (... IN (...))` |

> Regla de oro: **un registro puede estar `is_active = true` (habilitado) y a la vez tener `status = PUBLISHED` (estado de negocio).** Son ejes ortogonales.

---

## 2. Estándar de auditoría (columnas obligatorias)

### Tablas transaccionales (todas)

| Columna | Tipo | Nullable | Descripción |
|---------|------|----------|-------------|
| `created_at` | TIMESTAMPTZ | No | Momento de creación |
| `created_by` | UUID | No | Usuario que creó (ref. lógica → iam-service) |
| `updated_at` | TIMESTAMPTZ | No | Última modificación |
| `updated_by` | UUID | No | Usuario que modificó |
| `deleted_at` | TIMESTAMPTZ | Sí | Soft delete; `NULL` = no eliminado |
| `deleted_by` | UUID | Sí | Usuario que eliminó; `NULL` si vigente |
| `is_active` | BOOLEAN | No | Habilitado/deshabilitado **sin** eliminar (DEFAULT true) |
| `row_version` | INT | No | Bloqueo optimista (DEFAULT 0) |

### Actor de sistema

Las acciones automáticas (workers, jobs, seeds) usan el UUID reservado:

```
SYSTEM_ACTOR_ID = 00000000-0000-0000-0000-000000000000
```

`created_by`/`updated_by`/`deleted_by` nunca quedan en NULL para acciones del sistema: se usa `SYSTEM_ACTOR_ID`.

### Tablas catálogo

Solo requieren `created_at`, `created_by`, `updated_at`, `updated_by`, `is_active`. No llevan soft delete (`deleted_at`): un valor de catálogo se desactiva con `is_active = false`, no se elimina.

### Tablas append-only

`audit_record`, `audit_login`, `activity_log` y similares conservan **únicamente** su timestamp de inserción (`recorded_at` / `received_at` / `attempted_at`). No tienen `updated_*`, `deleted_*` ni `is_active`: son inmutables por definición.

### Regla de consulta

Toda query de lectura sobre tablas transaccionales filtra por defecto `WHERE deleted_at IS NULL`. La recuperación de registros eliminados es una operación administrativa explícita.

### Estados derivados del ciclo de vida del registro

No se almacenan; se derivan de las columnas:

| Estado del registro | Condición |
|---------------------|-----------|
| ACTIVO | `deleted_at IS NULL AND is_active = true` |
| INACTIVO (deshabilitado) | `deleted_at IS NULL AND is_active = false` |
| ELIMINADO (lógico) | `deleted_at IS NOT NULL` |

---

## 3. Estados de negocio parametrizables — patrón genérico

Cada servicio que tiene agregados con máquina de estados implementa este patrón en su propia BD. Es la **entidad padre** que parametriza cualquier estado de negocio del servicio.

### `status_category` — el padre que parametriza tipos de estado

| Campo | Tipo | Nullable | Descripción |
|-------|------|----------|-------------|
| `id` | UUID | No | PK |
| `code` | VARCHAR(50) | No | UNIQUE; ej: `SCHEDULE`, `FICHA`, `LEARNER_ENROLLMENT`, `PRODUCTIVE_STAGE`, `KPI`, `RISK` |
| `name` | VARCHAR(120) | No | Nombre para visualización |
| `description` | TEXT | Sí | Para qué agregado/proceso aplica |
| `applies_to_entity` | VARCHAR(80) | Sí | Entidad a la que aplica (documental, ej: `Schedule`) |
| `is_active` | BOOLEAN | No | DEFAULT true |
| `created_at`, `created_by`, `updated_at`, `updated_by` | — | — | Auditoría de catálogo |

### `status` — los valores de estado parametrizables

| Campo | Tipo | Nullable | Descripción |
|-------|------|----------|-------------|
| `id` | UUID | No | PK |
| `status_category_id` | UUID | No | FK → status_category (ON DELETE RESTRICT) |
| `code` | VARCHAR(50) | No | ej: `DRAFT`, `PUBLISHED`, `ARCHIVED` |
| `name` | VARCHAR(120) | No | Etiqueta para UI |
| `description` | TEXT | Sí | |
| `is_initial` | BOOLEAN | No | true = estado inicial de la máquina |
| `is_terminal` | BOOLEAN | No | true = estado final (sin transiciones salientes) |
| `display_order` | SMALLINT | No | Orden de presentación |
| `color_hex` | VARCHAR(7) | Sí | Color opcional para UI |
| `is_active` | BOOLEAN | No | DEFAULT true |
| `created_at`, `created_by`, `updated_at`, `updated_by` | — | — | Auditoría de catálogo |

**Constraint UNIQUE**: `(status_category_id, code)`.

### `status_transition` — transiciones gobernadas (opcional por agregado)

Parametriza qué transiciones de estado son válidas y quién puede ejecutarlas. Reemplaza las reglas de transición hardcodeadas (ej. RN-ACAD-05, RN-SCH-01).

| Campo | Tipo | Nullable | Descripción |
|-------|------|----------|-------------|
| `id` | UUID | No | PK |
| `status_category_id` | UUID | No | FK → status_category |
| `from_status_id` | UUID | No | FK → status |
| `to_status_id` | UUID | No | FK → status |
| `required_feature_code` | VARCHAR(60) | Sí | Feature de iam que habilita la transición (ej: `SCH_PUBLISH`) |
| `is_active` | BOOLEAN | No | DEFAULT true |
| `created_at`, `created_by`, `updated_at`, `updated_by` | — | — | Auditoría de catálogo |

**Constraint UNIQUE**: `(from_status_id, to_status_id)`.

### Uso en los agregados

Cualquier entidad con estado de negocio reemplaza su `VARCHAR + CHECK` de estado por:

```
status_id  UUID  NOT NULL  FK → status (ON DELETE RESTRICT)
```

El servicio valida cada cambio de estado contra `status_transition` antes de persistir.

---

## 4. Mapeo — qué pasa al catálogo y qué queda como CHECK

Decisión (ADR-004): **solo estados de negocio** se parametrizan; los enums técnicos cerrados siguen como `VARCHAR + CHECK`.

| Servicio | Estado de negocio → `status` (categoría) | Enum técnico → VARCHAR+CHECK |
|----------|------------------------------------------|------------------------------|
| iam | — (no aplica máquina de estados de negocio) | `audit_login.outcome` |
| reference-data | — | — |
| academic-management | `enrollment_ficha.status` (cat. `FICHA`) | `training_level`, `training_shift`, `training_modality` (o catálogo de ref-data) |
| training-environment | `reservation.status` (cat. `RESERVATION`) | — |
| scheduling | `schedule.status` (cat. `SCHEDULE`) | `class_session.status`, `conflict_type` |
| actors | `learner_ficha_enrollment.enrollment_status` (cat. `LEARNER_ENROLLMENT`), `productive_stage.status` (cat. `PRODUCTIVE_STAGE`), `actor_improvement_plan.status` (cat. `IMPROVEMENT_PLAN`) | `document_type`, `contract_type`, `exception_type`, `visit_type` |
| document | `document.status` (cat. `DOCUMENT`) | `output_type`, `mime_type` |
| monitoring | `ficha_tracking.overall_status`, `kpi_status`, `risk_level` (cat. `KPI_STATUS`, `RISK_LEVEL`), `improvement_plan.status` | `kpi_type.unit`, `sent_notification.channel`, `tracking_session.session_type` |
| audit | — (append-only) | `event_type` (texto libre del evento) |

**Nota sobre catálogos ricos**: catálogos con atributos de dominio más allá del estado (ej. `alert_type` con umbrales, `kpi_type` con unidad) **no** son estados — permanecen como catálogos especializados propios. Solo los conjuntos de valores que representan *posición en una máquina de estados* migran a `status`.

---

## 5. Seeds por servicio

Cada servicio siembra sus `status_category` + `status` (+ `status_transition` si aplica) en el deploy inicial. Ejemplo para scheduling:

```
status_category: SCHEDULE
status (cat=SCHEDULE):
  DRAFT        (is_initial=true)
  UNDER_REVIEW
  PUBLISHED
  ARCHIVED     (is_terminal=true)
status_transition (cat=SCHEDULE):
  DRAFT        → UNDER_REVIEW   (required_feature=SCH_EDIT)
  UNDER_REVIEW → PUBLISHED      (required_feature=SCH_PUBLISH)
  PUBLISHED    → ARCHIVED       (required_feature=SCH_ARCHIVE)
```

---

## 6. Otras convenciones (vigentes)

- **Sin `ENUM` nativo de Postgres** (su `ALTER TYPE` bloquea y dificulta migraciones independientes por servicio).
- **Acciones referenciales**: cada FK declara `ON UPDATE`/`ON DELETE`. Por defecto: catálogo/padre → `RESTRICT`; hijo de agregado (composición) → `CASCADE`; FK opcional → `SET NULL`.
- **Nomenclatura de constraints**: `pk_<tabla>`, `uq_<tabla>_<cols>`, `fk_<tabla>_<ref>`, `ck_<tabla>_<regla>`, `ix_<tabla>_<cols>`.
- **PK**: UUID v4 en todas las tablas.
- **Timestamps**: siempre `TIMESTAMPTZ` (UTC); la conversión a hora Colombia es de la capa de presentación.

## 7. Estructura DDL y orden de aplicación (Liquibase)

Cada repositorio `*-db` organiza el DDL en carpetas numeradas que **definen el orden de ejecución** de los changelogs. El orden no es cosmético: garantiza que cada objeto se cree cuando sus dependencias ya existen.

```
01_ddl/
  00_extensions/   -- CREATE EXTENSION (pgcrypto / gen_random_uuid)
  01_schemas/      -- CREATE SCHEMA del módulo
  02_types/        -- DOMAIN / tipos (si aplica)
  03_tables/       -- CREATE TABLE  (SIN llaves foráneas)
  04_alter/        -- ALTER TABLE ... ADD CONSTRAINT  (llaves foráneas)  ← ver regla abajo
  05_views/        -- vistas
  06_functions/    -- funciones
  07_procedures/   -- procedimientos
  08_triggers/     -- triggers
  10_indexes/      -- índices (incluye un índice por cada FK)
02_dml/            -- datos semilla (seeds), con control de duplicados
03_dcl/            -- roles y GRANT/REVOKE (least-privilege)
04_tcl/            -- tags de versión / release
05_rollbacks/      -- rollbacks espejo de cada changeset
```

### Regla: las llaves foráneas van en `04_alter`, no en `03_tables`

**Las tablas se crean primero, sin FKs, en `03_tables`. Las llaves foráneas se agregan después, vía `ALTER TABLE ... ADD CONSTRAINT`, en `04_alter`.**

Motivo: una FK requiere que la tabla referenciada **ya exista**. Si las FKs se declaran inline en `CREATE TABLE`, el orden de creación pasa a importar y aparecen rupturas por dependencias circulares o referencias hacia adelante (`relation "..." does not exist`). Separando la creación de la estructura (`03_tables`) del cableado referencial (`04_alter`) el despliegue es **determinista e independiente del orden entre tablas**.

- `03_tables`: `CREATE TABLE` con PK, columnas, `NOT NULL`, `UNIQUE` y `CHECK` locales. **Sin `REFERENCES`.**
- `04_alter`: un changeset por grupo de FKs — `ALTER TABLE <hija> ADD CONSTRAINT fk_<tabla>_<ref> FOREIGN KEY (...) REFERENCES <padre> (...) ON UPDATE ... ON DELETE ...`.
- `10_indexes`: crear el índice de cada columna FK (Postgres **no** lo crea automáticamente).
- `05_rollbacks`: el rollback de `04_alter` hace `DROP CONSTRAINT`; el de `03_tables`, `DROP TABLE`. Deben ser espejo exacto de sus rutas.
