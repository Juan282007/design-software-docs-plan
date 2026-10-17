# Modelo de datos — actors-service

> Estado: 🟢 Estable | Última actualización: 2026-06-20
> Naming: entidades y atributos en inglés (HALT-DB-NAMING)
> Fuente de dominio: análisis M7 (Instructores, Aprendices, Empresas)

## Convenciones de modelado (transversales)

> **Estándar autoritativo:** [06-data/modeling-conventions.md](../../../06-data/modeling-conventions.md) (ratificado en ADR-004). Resumen aplicable:

- **Tres conceptos de estado (no confundir):** (1) *ciclo de vida del registro* (técnico) → `is_active` + soft delete (`deleted_at`); (2) *estado de negocio* (parametrizable) → FK `status_id` → catálogo `status`; (3) *enum técnico cerrado* (CC/CE, EMAIL/IN_APP…) → `VARCHAR` + `CHECK (... IN (...))`. **No se usa el tipo `ENUM` nativo de Postgres** (su `ALTER TYPE` bloquea migraciones independientes por servicio).
- **Estados de negocio parametrizables:** el servicio implementa `status_category` + `status` (+ `status_transition` si aplica) en su propia BD; los agregados con máquina de estados referencian `status_id`. Solo migran estados de **negocio**; los enums técnicos cerrados (`*_type`, `channel`…) permanecen como `VARCHAR + CHECK`.
- **Auditoría (tablas transaccionales):** `created_at`/`created_by`, `updated_at`/`updated_by`, `deleted_at`/`deleted_by` (soft delete), `is_active` y `row_version`. Acciones del sistema usan `SYSTEM_ACTOR_ID = 00000000-0000-0000-0000-000000000000`. Catálogos: `created_*`/`updated_*` + `is_active` (sin soft delete). Append-only: solo el timestamp de inserción.
- **Acciones referenciales:** cada FK declara `ON UPDATE`/`ON DELETE`. Por defecto: catálogo/padre → `RESTRICT`; hijo de agregado (composición) → `CASCADE`; FK opcional → `SET NULL`.
- **Nomenclatura de constraints:** `pk_<tabla>`, `uq_<tabla>_<cols>`, `fk_<tabla>_<ref>`, `ck_<tabla>_<regla>`, `ix_<tabla>_<cols>`.

## Entidades propias

---

### `instructor`

Formador vinculado al SENA. Un instructor puede estar asignado a múltiples fichas y tener múltiples competencias.

| Campo | Tipo | Nullable | PII | Descripción |
|-------|------|----------|-----|-------------|
| `id` | UUID | No | — | PK |
| `user_id` | UUID | No | — | Referencia externa → iam-service |
| `document_type` | VARCHAR(2) | No | ✓ | CC / CE / PA / PE |
| `document_number` | VARCHAR(20) | No | ✓ | UNIQUE |
| `first_name` | VARCHAR(100) | No | ✓ | Nombre(s) del instructor |
| `last_name` | VARCHAR(100) | No | ✓ | Apellido(s) del instructor |
| `email` | VARCHAR(255) | No | ✓ | Correo institucional |
| `phone` | VARCHAR(20) | Sí | ✓ | |
| `default_max_hours_per_week` | DECIMAL(4,1) | No | — | Carga máxima autorizada por defecto |
| `is_active` | BOOLEAN | No | — | false = inactivo o retirado |
| `created_at` | TIMESTAMPTZ | No | — | |
| `updated_at` | TIMESTAMPTZ | No | — | |

**CHECK**: `document_type IN ('CC','CE','PA','PE')`; `default_max_hours_per_week >= 0`.

**Nota — `default_max_hours_per_week`**: Límite por defecto; el límite efectivo lo define el contrato vigente (`instructor_contract.weekly_hour_limit` WHERE is_current). Si el contrato no especifica, se usa este valor.

**Retención PII**: datos de `instructor` → 5 años después de `is_active = false`.

---

### `instructor_area`

Áreas tecnológicas en las que el instructor puede impartir formación. Un instructor puede tener una o varias áreas.

| Campo | Tipo | Nullable | Descripción |
|-------|------|----------|-------------|
| `id` | UUID | No | PK |
| `instructor_id` | UUID | No | FK → instructor |
| `area_name` | VARCHAR(100) | No | Nombre del área tecnológica |
| `is_primary` | BOOLEAN | No | true = área principal del instructor |
| `is_active` | BOOLEAN | No | DEFAULT true |
| `created_at` | TIMESTAMPTZ | No | |

**Constraint UNIQUE**: `(instructor_id, area_name)`

---

### `instructor_contract`

Vinculación laboral del instructor. Un instructor puede tener múltiples contratos históricos.
Solo puede haber un contrato con `is_current = true` por instructor.

| Campo | Tipo | Nullable | Descripción |
|-------|------|----------|-------------|
| `id` | UUID | No | PK |
| `instructor_id` | UUID | No | FK → instructor |
| `contract_type` | VARCHAR(12) | No | STAFF=planta, CONTRACTOR=OPS, HOURLY=hora-cátedra |
| `start_date` | DATE | No | |
| `end_date` | DATE | Sí | null = vigente |
| `training_center_id` | UUID | No | Referencia externa → reference-data-service |
| `weekly_hour_limit` | DECIMAL(4,1) | Sí | Límite específico del contrato (fuente autoritativa del límite vigente) |
| `is_current` | BOOLEAN | No | true = vinculación activa; UNIQUE parcial |
| `notes` | TEXT | Sí | Observaciones del contrato |
| `created_at` | TIMESTAMPTZ | No | |
| `updated_at` | TIMESTAMPTZ | No | |

**CHECK**: `contract_type IN ('STAFF','CONTRACTOR','HOURLY')`.

---

### `competency_assignment`

Competencias que el instructor está certificado para impartir. Base para filtrar
instructores disponibles durante la creación de horarios.

| Campo | Tipo | Nullable | Descripción |
|-------|------|----------|-------------|
| `id` | UUID | No | PK |
| `instructor_id` | UUID | No | FK → instructor |
| `competency_id` | UUID | No | Referencia externa → academic-management-service |
| `certified_at` | DATE | No | Fecha de certificación o habilitación |
| `certification_entity` | VARCHAR(200) | Sí | Entidad que certificó (SENA, universidad, etc.) |
| `is_active` | BOOLEAN | No | false = habilitación revocada |
| `created_at` | TIMESTAMPTZ | No | |
| `updated_at` | TIMESTAMPTZ | No | |

**Constraint UNIQUE**: `(instructor_id, competency_id)` para competencias activas.

---

### `instructor_availability_exception`

Bloqueos de disponibilidad no-recurrentes del instructor: permisos, licencias, incapacidades,
comisiones, etc. Complementa las reglas de disponibilidad del ambiente en scheduling.

| Campo | Tipo | Nullable | Descripción |
|-------|------|----------|-------------|
| `id` | UUID | No | PK |
| `instructor_id` | UUID | No | FK → instructor |
| `exception_type` | VARCHAR(20) | No | Tipo de ausencia |
| `start_datetime` | TIMESTAMPTZ | No | Inicio del bloqueo |
| `end_datetime` | TIMESTAMPTZ | No | Fin del bloqueo |
| `description` | TEXT | Sí | Detalle adicional |
| `approved_by` | UUID | Sí | Referencia externa → iam-service (coordinador que aprobó) |
| `created_at` | TIMESTAMPTZ | No | |

**CHECK**: `exception_type IN ('SICK_LEAVE','VACATION','COMMISSION','PERSONAL_LEAVE','TRAINING','OTHER')`.

**Uso en scheduling**: el endpoint `GET /instructors/available` descuenta estas excepciones
al calcular disponibilidad en el período solicitado.

---

### `learner`

Aprendiz en proceso de formación. La ficha activa y la etapa actual del aprendiz se registran en learner_ficha_enrollment.

| Campo | Tipo | Nullable | PII | Descripción |
|-------|------|----------|-----|-------------|
| `id` | UUID | No | — | PK |
| `user_id` | UUID | No | — | Referencia externa → iam-service |
| `document_type` | VARCHAR(2) | No | ✓ | CC / CE / RC / TI |
| `document_number` | VARCHAR(20) | No | ✓ | UNIQUE |
| `first_name` | VARCHAR(100) | No | ✓ | Nombre(s) del aprendiz |
| `last_name` | VARCHAR(100) | No | ✓ | Apellido(s) del aprendiz |
| `email` | VARCHAR(255) | No | ✓ | |
| `phone` | VARCHAR(20) | Sí | ✓ | |
| `birth_date` | DATE | Sí | ✓ | |
| `created_at` | TIMESTAMPTZ | No | — | |

**CHECK**: `document_type IN ('CC','CE','RC','TI')`.

**Nota**: el estado de vinculación (`enrollment_status_id`) y la fecha de vinculación
(`enrollment_date`) NO viven en `learner` — pertenecen a la inscripción de ficha vigente;
ver `learner_ficha_enrollment`.

**Retención PII**: datos de `learner` → 5 años después de graduación o desvinculación.

---

### `learner_ficha_enrollment`

Historial de inscripciones de un aprendiz en fichas de caracterización. Permite registrar traslados entre fichas manteniendo el historial completo. Un aprendiz tiene exactamente un registro con is_current=true en todo momento.

| Campo | Tipo | Nullable | Descripción |
|-------|------|----------|-------------|
| `id` | UUID | No | PK |
| `learner_id` | UUID | No | FK → learner |
| `ficha_id` | UUID | No | Referencia externa → academic-management-service |
| `enrollment_status_id` | UUID | No | FK → `actors_parameterization.status` (estado de negocio, parametrizable) |
| `stage` | VARCHAR(12) | No | Etapa actual del aprendiz en esta ficha |
| `enrollment_date` | DATE | No | Fecha de inscripción en esta ficha |
| `completion_date` | DATE | Sí | null = inscripción vigente |
| `is_current` | BOOLEAN | No | true = inscripción activa; solo una por aprendiz |
| `transfer_reason` | TEXT | Sí | Motivo de traslado si aplica |
| `created_at` | TIMESTAMPTZ | No | |

**CHECK**: `stage IN ('INDUCTION','LECTURE','PRODUCTIVE')`.

**Constraints**:
- UNIQUE `(learner_id, ficha_id)`
- Partial UNIQUE `(learner_id)` WHERE `is_current = true` — un solo registro activo por aprendiz

---

### `company`

Empresa que participa en la etapa productiva de aprendices SENA.

| Campo | Tipo | Nullable | PII | Descripción |
|-------|------|----------|-----|-------------|
| `id` | UUID | No | — | PK |
| `nit` | VARCHAR(20) | No | — | UNIQUE; número de identificación tributaria |
| `business_name` | VARCHAR(200) | No | — | Razón social |
| `trade_name` | VARCHAR(200) | Sí | — | Nombre comercial |
| `economic_activity` | VARCHAR(100) | Sí | — | Actividad económica principal (CIIU) |
| `address` | TEXT | Sí | — | |
| `city_id` | UUID | Sí | — | Referencia externa → reference-data-service (Municipality) |
| `contact_name` | VARCHAR(200) | Sí | ✓ | Nombre del responsable de etapa productiva |
| `contact_email` | VARCHAR(255) | Sí | ✓ | |
| `contact_phone` | VARCHAR(20) | Sí | ✓ | |
| `is_active` | BOOLEAN | No | — | |
| `created_at` | TIMESTAMPTZ | No | — | |
| `updated_at` | TIMESTAMPTZ | No | — | |

---

### `productive_stage`

Etapa productiva de un aprendiz en una empresa. El aprendiz pasa de etapa lectiva a productiva
cuando cumple los requisitos académicos del programa.

| Campo | Tipo | Nullable | Descripción |
|-------|------|----------|-------------|
| `id` | UUID | No | PK |
| `learner_id` | UUID | No | FK → learner |
| `company_id` | UUID | No | FK → company |
| `supervisor_instructor_id` | UUID | No | FK → instructor (instructor de seguimiento) |
| `start_date` | DATE | No | |
| `planned_end_date` | DATE | No | Fecha fin esperada según programa |
| `actual_end_date` | DATE | Sí | Fecha fin real (null si en curso) |
| `total_hours_required` | INT | No | Horas requeridas según programa |
| `total_hours_completed` | INT | No | DEFAULT 0 |
| `status_id` | UUID | No | FK → `actors_parameterization.status` (estado de negocio, parametrizable) |
| `interruption_reason` | TEXT | Sí | Solo si el status vigente corresponde a INTERRUPTED |
| `created_at` | TIMESTAMPTZ | No | |
| `updated_at` | TIMESTAMPTZ | No | |

**CHECK**: `total_hours_required >= 0`; `total_hours_completed >= 0`.

---

### `company_visit`

Visita de seguimiento del instructor supervisor a la empresa donde está el aprendiz.
Normativa SENA: mínimo 2 visitas durante la etapa productiva (Acuerdo 00003/2012, Art. 65).

| Campo | Tipo | Nullable | Descripción |
|-------|------|----------|-------------|
| `id` | UUID | No | PK |
| `productive_stage_id` | UUID | No | FK → productive_stage |
| `instructor_id` | UUID | No | FK → instructor |
| `visit_number` | SMALLINT | No | Número de visita (1, 2, 3...) |
| `visit_date` | DATE | No | |
| `visit_type` | VARCHAR(10) | No | Modalidad de visita |
| `learner_performance` | VARCHAR(12) | No | Evaluación del desempeño |
| `company_satisfaction` | VARCHAR(6) | Sí | Satisfacción de la empresa |
| `observations` | TEXT | Sí | Observaciones del instructor |
| `next_visit_date` | DATE | Sí | Fecha sugerida para próxima visita |
| `has_improvement_plan` | BOOLEAN | No | DEFAULT false |
| `created_at` | TIMESTAMPTZ | No | |
| `updated_at` | TIMESTAMPTZ | No | |

**CHECK**: `visit_type IN ('IN_PERSON','VIRTUAL','PHONE')`; `learner_performance IN ('EXCELLENT','GOOD','ACCEPTABLE','DEFICIENT')`; `company_satisfaction IN ('HIGH','MEDIUM','LOW')`.

---

### `actor_improvement_plan`

Plan de mejoramiento individual de un actor (instructor o aprendiz), generado a partir de
evaluaciones de desempeño o visitas empresariales. Separado del `ImprovementPlan`
de monitoring-service, que es a nivel de ficha.

| Campo | Tipo | Nullable | Descripción |
|-------|------|----------|-------------|
| `id` | UUID | No | PK |
| `instructor_id` | UUID | Sí | FK → instructor (arco exclusivo) |
| `learner_id` | UUID | Sí | FK → learner (arco exclusivo) |
| `origin` | VARCHAR(20) | No | Origen del plan |
| `origin_reference_id` | UUID | Sí | ID del company_visit o sesión de origen |
| `description` | TEXT | No | Descripción del plan de mejoramiento |
| `specific_goals` | TEXT | No | Metas específicas y verificables |
| `due_date` | DATE | No | |
| `status_id` | UUID | No | FK → `actors_parameterization.status` (estado de negocio, parametrizable) |
| `created_by` | UUID | No | Referencia externa → iam-service |
| `created_at` | TIMESTAMPTZ | No | |
| `completed_at` | TIMESTAMPTZ | Sí | |

**CHECK**: `origin IN ('COMPANY_VISIT','PERFORMANCE_REVIEW','TRACKING_SESSION','DISCIPLINARY')`.

**Arco exclusivo (FK polimórfica resuelta)**: el actor se referencia mediante dos FK nullable
(`instructor_id`, `learner_id`) en vez de un par `actor_type` + `actor_id` sin integridad
referencial. Un `CHECK` garantiza que exactamente una esté poblada:
`CHECK ((instructor_id IS NOT NULL) <> (learner_id IS NOT NULL))`. Esto permite declarar FK
reales hacia `instructor` y `learner`, recuperando la integridad que una FK polimórfica no ofrece.

---

### `activity_log`

Bitácora de actividad relevante de cada actor en el sistema. Registra cambios de estado,
asignaciones y eventos importantes del ciclo de vida del actor. No reemplaza al audit-service
(que registra eventos técnicos); este registro es funcional/pedagógico.

| Campo | Tipo | Nullable | Descripción |
|-------|------|----------|-------------|
| `id` | UUID | No | PK |
| `instructor_id` | UUID | Sí | FK → instructor (arco exclusivo) |
| `learner_id` | UUID | Sí | FK → learner (arco exclusivo) |
| `event_type` | VARCHAR(60) | No | Ej: `STATUS_CHANGED`, `COMPETENCY_ADDED`, `STAGE_CHANGED` |
| `description` | TEXT | No | Descripción legible del evento |
| `previous_value` | TEXT | Sí | Valor anterior (JSON string si aplica) |
| `new_value` | TEXT | Sí | Valor nuevo |
| `recorded_by` | UUID | Sí | null = evento del sistema |
| `recorded_at` | TIMESTAMPTZ | No | |

**Arco exclusivo (FK polimórfica resuelta)**: el actor se referencia mediante dos FK nullable
(`instructor_id`, `learner_id`) en vez de un par `actor_type` + `actor_id` sin integridad
referencial. Un `CHECK` garantiza que exactamente una esté poblada:
`CHECK ((instructor_id IS NOT NULL) <> (learner_id IS NOT NULL))`, recuperando la integridad
referencial que una FK polimórfica no ofrece.

**Solo INSERT** — la bitácora es append-only. No se modifica ni elimina.

---

## Referencias externas

| Campo | Referencia lógica | Servicio propietario |
|-------|-------------------|----------------------|
| `user_id` (instructor) | `user.id` | `iam-service` |
| `user_id` (learner) | `user.id` | `iam-service` |
| `competency_id` | `competency.id` | `academic-management-service` |
| `ficha_id` (learner_ficha_enrollment) | `enrollment_ficha.id` | `academic-management-service` |
| `training_center_id` | `training_center.id` | `reference-data-service` |
| `city_id` | `municipality.id` | `reference-data-service` |
| `approved_by` (availability exception) | `user.id` | `iam-service` |

---

## Índices relevantes

| Tabla | Campos indexados | Tipo | Motivo |
|-------|-----------------|------|--------|
| `instructor` | `document_number` | UNIQUE | Identificación única |
| `instructor` | `user_id` | UNIQUE | Vínculo 1:1 con User |
| `instructor` | `is_active` | Partial | Filtrar solo instructores activos |
| `instructor` | `(last_name, first_name)` | B-Tree | Búsqueda por nombre |
| `instructor_area` | `instructor_id` | B-Tree | Áreas de un instructor |
| `competency_assignment` | `(instructor_id, competency_id)` | UNIQUE | Evita duplicados activos |
| `competency_assignment` | `competency_id` WHERE `is_active = true` | Partial | `GET /instructors/available?competency_id=` (< 300 ms) |
| `instructor_availability_exception` | `(instructor_id, start_datetime, end_datetime)` | B-Tree | Overlap de disponibilidad |
| `learner` | `document_number` | UNIQUE | Identificación única |
| `learner` | `(last_name, first_name)` | B-Tree | Búsqueda por nombre |
| `learner_ficha_enrollment` | `(learner_id, is_current)` | Partial | Inscripción activa del aprendiz |
| `learner_ficha_enrollment` | `ficha_id` | B-Tree | Aprendices por ficha |
| `learner_ficha_enrollment` | `(learner_id, enrollment_date)` | B-Tree | Historial cronológico |
| `activity_log` | `(instructor_id, recorded_at)` WHERE `instructor_id IS NOT NULL` | Partial | Bitácora cronológica por instructor |
| `activity_log` | `(learner_id, recorded_at)` WHERE `learner_id IS NOT NULL` | Partial | Bitácora cronológica por aprendiz |
| `company` | `nit` | UNIQUE | Identificación tributaria única |
