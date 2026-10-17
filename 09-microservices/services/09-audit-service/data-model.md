# Modelo de datos — audit-service

> Estado: 🟢 Estable | Última actualización: 2026-06-20
> Naming: entidades y atributos en inglés (HALT-DB-NAMING)

## Convenciones de modelado (transversales)

> **Estándar autoritativo:** [06-data/modeling-conventions.md](../../../06-data/modeling-conventions.md) (ratificado en ADR-004). Resumen aplicable:

- **Tres conceptos de estado (no confundir):** (1) *ciclo de vida del registro* (técnico) → `is_active` + soft delete (`deleted_at`); (2) *estado de negocio* (parametrizable) → FK `status_id` → catálogo `status`; (3) *enum técnico cerrado* (CC/CE, EMAIL/IN_APP…) → `VARCHAR` + `CHECK (... IN (...))`. **No se usa el tipo `ENUM` nativo de Postgres** (su `ALTER TYPE` bloquea migraciones independientes por servicio).
- **Estados de negocio parametrizables:** el servicio implementa `status_category` + `status` (+ `status_transition` si aplica) en su propia BD; los agregados con máquina de estados referencian `status_id`. Solo migran estados de **negocio**; los enums técnicos cerrados (`*_type`, `channel`…) permanecen como `VARCHAR + CHECK`. **Nota:** este servicio es append-only (`audit_record`), por lo que no aplica máquina de estados ni soft delete.
- **Auditoría (tablas transaccionales):** `created_at`/`created_by`, `updated_at`/`updated_by`, `deleted_at`/`deleted_by` (soft delete), `is_active` y `row_version`. Acciones del sistema usan `SYSTEM_ACTOR_ID = 00000000-0000-0000-0000-000000000000`. Catálogos: `created_*`/`updated_*` + `is_active` (sin soft delete). Append-only: solo el timestamp de inserción.
- **Acciones referenciales:** cada FK declara `ON UPDATE`/`ON DELETE`. Por defecto: catálogo/padre → `RESTRICT`; hijo de agregado (composición) → `CASCADE`; FK opcional → `SET NULL`.
- **Nomenclatura de constraints:** `pk_<tabla>`, `uq_<tabla>_<cols>`, `fk_<tabla>_<ref>`, `ck_<tabla>_<regla>`, `ix_<tabla>_<cols>`.

## Entidades propias

### `audit_record`

Registro inmutable de cada evento de negocio recibido por el sistema.

> **Tabla append-only.** Por diseño, `audit_record` solo admite INSERT (ver "Regla crítica"). En consecuencia conserva un único timestamp de inserción (`received_at`) y **no lleva `updated_at`**: la convención transversal de `updated_at` y la de **bloqueo optimista (`row_version`) NO aplican** a esta tabla.

| Campo | Tipo | Nullable | Descripción |
|-------|------|----------|-------------|
| `id` | UUID | No | PK |
| `event_id` | UUID | No | UNIQUE — ID del evento original; garantiza idempotencia |
| `event_type` | VARCHAR(100) | No | Ej: `scheduling.class_session.created` |
| `source_service` | VARCHAR(50) | No | Servicio que publicó el evento |
| `actor_id` | UUID | Sí | Usuario que originó la acción (null = acción del sistema) |
| `entity_type` | VARCHAR(50) | Sí | Tipo de entidad afectada (ej: `ClassSession`) |
| `entity_id` | UUID | Sí | ID de la entidad afectada |
| `payload` | JSONB | No | Payload completo del evento |
| `event_occurred_at` | TIMESTAMPTZ | Sí | Timestamp del evento en el servicio origen (extraído del envelope). Distinto de received_at que es cuando llegó al audit-worker. |
| `received_at` | TIMESTAMPTZ | No | Timestamp de recepción en el audit-worker |

## Regla crítica

> **Solo INSERT.** Ninguna operación UPDATE o DELETE está permitida en `audit_db`.
> El log de auditoría es inmutable por definición. Un intento de modificar un `audit_record` es un error de diseño.

La unicidad por `event_id` garantiza idempotencia: si el broker re-entrega un evento (at-least-once), el segundo INSERT falla silenciosamente por constraint UNIQUE.

## Retención

- Mínimo: 7 años (pendiente de confirmación con normativa legal SENA)
- Estrategia de archivado: cold storage después de 2 años activos en tabla principal

## Índices relevantes

| Tabla | Campos | Tipo | Motivo |
|-------|--------|------|--------|
| `audit_record` | `event_id` | UNIQUE | Idempotencia; constraint crítica |
| `audit_record` | `event_type` | B-Tree | Filtrar por tipo de evento |
| `audit_record` | `source_service` | B-Tree | Filtrar por servicio fuente |
| `audit_record` | `(actor_id, received_at)` | B-Tree | Trazabilidad por usuario en período |
| `audit_record` | `entity_id` | B-Tree | Historial de cambios por entidad específica |
| `audit_record` | `event_occurred_at` | B-Tree | Consultas por rango de tiempo del evento original |

## Particionamiento recomendado

Para mantener rendimiento con retención de 7 años, se recomienda particionar `audit_record` por `received_at` usando PostgreSQL declarative partitioning (RANGE mensual). Cada partición mensual puede moverse a tablespace cold después de 2 años activos.

La clave de particionamiento se alinea con la consulta dominante (recuperación por ventana temporal de recepción y archivado mensual a cold storage): se confirma **`received_at` como clave de partición RANGE mensual**.

> **Nota sobre `event_id` UNIQUE:** en tablas particionadas de Postgres, un índice UNIQUE global solo es posible si la clave de partición forma parte del índice; como `event_id` es independiente de `received_at`, la unicidad se implementa o bien como **índice único global** mediante un esquema que incluya la clave de partición / verificación externa, o bien como **índice único por-partición** (uniqueness garantizada dentro de cada partición), según la estrategia adoptada. La idempotencia at-least-once exige la garantía global, por lo que se prefiere la opción que la asegure.

## Remediación de congruencia — R3 (2026-08-06)

Implementación efectiva (changesets nuevos, sin modificar los corridos):
- **`remediation-audit-960`:** `audit_record` ahora es **tabla particionada** (`PARTITION BY
  RANGE (received_at)`, particiones mensuales + `DEFAULT`). La PK pasa a `(id, received_at)` y el
  `UNIQUE(event_id)` a `(event_id, received_at)` (per-partición, requisito de Postgres). Además,
  índices renombrados `idx_` → `ix_` (convención transversal).
- **`remediation-audit-961`:** para preservar la **garantía global** de `event_id` (idempotencia
  at-least-once, la opción preferida arriba) se agrega la tabla NO particionada
  `audit_event_seen (event_id UUID PRIMARY KEY, received_at TIMESTAMPTZ)`: el `audit-worker`
  inserta ahí primero (`ON CONFLICT DO NOTHING`) para deduplicar globalmente antes de escribir en
  `audit_record`.
