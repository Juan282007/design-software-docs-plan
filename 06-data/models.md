# Modelo de Datos Lógico Global — SENA Gestión de Horarios

> Fase: 03-Design | Agente: A10 | Estado: 🟡 Borrador
> Fecha: 2026-06-17
> Prerequisitos: [overview.md](../05-architecture/overview.md) · [pattern-guide.md](../05-architecture/pattern-guide.md)
>
> **Nota de alcance**: Este documento es el **modelo lógico global** — entidades, atributos, relaciones y restricciones. El diseño físico (esquemas SQL, índices, migraciones) es responsabilidad del agente A13 en `db-design.md` por servicio.
>
> **Regla de naming**: Entidades y atributos en **inglés, singular, ASCII**. Toda descripción funcional puede ir en español. Violación: HALT-DB-NAMING.

---

## Glosario — Lenguaje Ubicuo

| Término (inglés) | Término (español SENA) | Definición |
|-------------------|------------------------|------------|
| `TrainingCenter` | Centro de Formación | Unidad operativa del SENA donde ocurre la formación |
| `EnrollmentFicha` | Ficha de Caracterización | Instancia de un programa para una cohorte específica |
| `TrainingProgram` | Programa de Formación | Diseño curricular aprobado (Técnico, Tecnólogo, etc.) |
| `Competency` | Competencia | Unidad de aprendizaje dentro de un programa |
| `LearningOutcome` | Resultado de Aprendizaje (RAP) | Subunidad de una competencia |
| `Schedule` | Horario | Asignación de sesiones de clase para una ficha |
| `ClassSession` | Sesión de Clase | Instancia de clase: ficha + instructor + ambiente + franja |
| `TimeSlot` | Franja Horaria | Bloque de tiempo recurrente (ej: lunes 07:00–10:00) |
| `Environment` | Ambiente de Formación | Espacio físico donde ocurre la clase |
| `Instructor` | Instructor | Formador vinculado al SENA, planta o contratista |
| `Learner` | Aprendiz | Persona en proceso de formación |
| `KpiTracking` | Seguimiento de KPI | Medición periódica de indicadores por ficha |
| `AuditRecord` | Registro de Auditoría | Log inmutable de toda acción en el sistema |

---

## Estándar transversal — Estado, ciclo de vida y auditoría

> Detalle completo y DDL en [modeling-conventions.md](./modeling-conventions.md) · ratificado en [ADR-004](../05-architecture/decisions/records/ADR-004-status-parametrization-and-audit-standard.md).

Tres conceptos de estado se modelan por separado en **todos** los bounded contexts:

| Concepto | Modelado |
|----------|----------|
| Ciclo de vida del registro (técnico) | `is_active` + soft delete (`deleted_at`/`deleted_by`) |
| Estado de negocio (parametrizable) | FK `status_id` → catálogo `status` |
| Enum técnico cerrado | `VARCHAR` + `CHECK` |

### Entidades transversales (instanciadas por servicio, no compartidas)

#### `Status_Category` — padre que parametriza tipos de estado
| Atributo | Tipo | Restricciones |
|----------|------|---------------|
| `id` | UUID | PK |
| `code` | VARCHAR(50) | UNIQUE — ej: `SCHEDULE`, `FICHA`, `LEARNER_ENROLLMENT`, `KPI`, `RISK` |
| `name` | VARCHAR(120) | NOT NULL |
| `applies_to_entity` | VARCHAR(80) | nullable (documental) |
| `is_active` | BOOLEAN | NOT NULL |

#### `Status` — valores de estado parametrizables
| Atributo | Tipo | Restricciones |
|----------|------|---------------|
| `id` | UUID | PK |
| `status_category_id` | UUID | FK → Status_Category |
| `code` | VARCHAR(50) | ej: `DRAFT`, `PUBLISHED`, `ARCHIVED` |
| `name` | VARCHAR(120) | NOT NULL |
| `is_initial` | BOOLEAN | estado inicial de la máquina |
| `is_terminal` | BOOLEAN | estado final |
| `display_order` | SMALLINT | orden en UI |
| `color_hex` | VARCHAR(7) | nullable |
| `is_active` | BOOLEAN | NOT NULL |

UNIQUE: `(status_category_id, code)`.

#### `Status_Transition` — transiciones gobernadas (opcional por agregado)
| Atributo | Tipo | Restricciones |
|----------|------|---------------|
| `id` | UUID | PK |
| `from_status_id` | UUID | FK → Status |
| `to_status_id` | UUID | FK → Status |
| `required_feature_code` | VARCHAR(60) | nullable — feature de iam que habilita la transición |
| `is_active` | BOOLEAN | NOT NULL |

UNIQUE: `(from_status_id, to_status_id)`.

### Columnas de auditoría (estándar en tablas transaccionales)

`created_at`/`created_by`, `updated_at`/`updated_by`, `deleted_at`/`deleted_by`, `is_active`, `row_version`. Acciones del sistema: `SYSTEM_ACTOR_ID = 00000000-0000-0000-0000-000000000000`. Append-only conservan solo su timestamp de inserción.

> En las definiciones de entidades por bounded context que siguen, las columnas de auditoría se omiten por brevedad: **se asumen presentes** en toda tabla transaccional según este estándar.

---

## Bounded Context: `iam-service`

> Diseño RBAC completo en [01-iam-service/rbac-design.md](../09-microservices/services/01-iam-service/rbac-design.md)

### Entidades

#### `User`
Cuenta de acceso. Cualquier persona que interactúa con el sistema.

| Atributo | Tipo | Restricciones | PII |
|----------|------|---------------|-----|
| `id` | UUID | PK, NOT NULL | — |
| `email` | VARCHAR(255) | UNIQUE, NOT NULL | ✓ |
| `password_hash` | TEXT | NOT NULL | — |
| `full_name` | VARCHAR(200) | NOT NULL | ✓ |
| `actor_type` | ENUM(`USER`, `INSTRUCTOR`, `LEARNER`) | NOT NULL | — |
| `actor_id` | UUID | NULLABLE | — |
| `is_active` | BOOLEAN | NOT NULL, DEFAULT true | — |
| `failed_attempts` | SMALLINT | NOT NULL, DEFAULT 0 | — |
| `locked_until` | TIMESTAMPTZ | NULLABLE | — |
| `created_at` | TIMESTAMPTZ | NOT NULL | — |
| `updated_at` | TIMESTAMPTZ | NOT NULL | — |

#### `Module`
Grupo funcional de alto nivel (sección de UI). Dato de seed, no modificable en runtime.

| Atributo | Tipo | Restricciones |
|----------|------|---------------|
| `id` | UUID | PK |
| `code` | VARCHAR(30) | UNIQUE, NOT NULL — ej: `MOD_SCHEDULING` |
| `name` | VARCHAR(100) | NOT NULL |
| `display_order` | SMALLINT | NOT NULL |
| `icon_key` | VARCHAR(50) | NULLABLE |
| `is_active` | BOOLEAN | NOT NULL |

#### `Feature`
Capacidad específica dentro de un módulo. **Unidad de permiso del sistema.**
Reemplaza la entidad `Permission` del modelo plano anterior.

| Atributo | Tipo | Restricciones |
|----------|------|---------------|
| `id` | UUID | PK |
| `module_id` | UUID | FK → Module, NOT NULL |
| `code` | VARCHAR(60) | UNIQUE, NOT NULL — ej: `SCH_CREATE`, `MON_ALERT_RESOLVE` |
| `name` | VARCHAR(120) | NOT NULL |
| `action_level` | ENUM(`READ`, `WRITE`, `DELETE`, `PUBLISH`, `APPROVE`) | NOT NULL |
| `is_active` | BOOLEAN | NOT NULL |

~50 features pre-cargados en seed. Ver catálogo en rbac-design.md.

#### `Role`
Conjunto de features y scopes asignables a usuarios.

| Atributo | Tipo | Restricciones |
|----------|------|---------------|
| `id` | UUID | PK |
| `name` | VARCHAR(50) | UNIQUE, NOT NULL |
| `display_name` | VARCHAR(100) | NOT NULL |
| `is_system_role` | BOOLEAN | NOT NULL |

7 roles: `SYSTEM_ADMIN`, `CENTER_DIRECTOR`, `COORDINATOR`, `AREA_LEADER`, `INSTRUCTOR`, `LEARNER`, `ADMIN_STAFF`

#### `RoleFeature`
Asignación de un feature a un rol con su scope de acceso.
Reemplaza las tablas `RolePermission` y `Permission` del modelo plano.

| Atributo | Tipo | Restricciones |
|----------|------|---------------|
| `id` | UUID | PK |
| `role_id` | UUID | FK → Role |
| `feature_id` | UUID | FK → Feature |
| `scope_type` | ENUM(`GLOBAL`, `TRAINING_CENTER`, `AREA`, `OWN_FICHAS`, `OWN_SCHEDULE`, `OWN_PROFILE`, `OWN_FICHA_AS_LEARNER`) | NOT NULL |

UNIQUE: `(role_id, feature_id)`. ~120 registros de seed según la matriz de permisos.

#### `UserRole`
Asignación de rol a usuario, opcionalmente restringida a un centro.

| Atributo | Tipo | Restricciones |
|----------|------|---------------|
| `id` | UUID | PK |
| `user_id` | UUID | FK → User |
| `role_id` | UUID | FK → Role |
| `training_center_id` | UUID | NULLABLE — null = alcance global |
| `assigned_by` | UUID | FK → User |
| `expires_at` | TIMESTAMPTZ | NULLABLE |

#### `UserScopeOverride`
Override individual de acceso (aditivo o restrictivo) para casos excepcionales.

| Atributo | Tipo | Restricciones |
|----------|------|---------------|
| `id` | UUID | PK |
| `user_id` | UUID | FK → User |
| `feature_id` | UUID | FK → Feature |
| `scope_type` | ENUM (mismo que RoleFeature) | NOT NULL |
| `is_allowed` | BOOLEAN | NOT NULL |
| `reason` | TEXT | NOT NULL |
| `expires_at` | TIMESTAMPTZ | NULLABLE |

#### `RefreshToken`
Token de renovación de sesión. Un usuario puede tener múltiples sesiones activas.

| Atributo | Tipo | Restricciones | PII |
|----------|------|---------------|-----|
| `id` | UUID | PK | — |
| `user_id` | UUID | FK → User | — |
| `token_hash` | TEXT | UNIQUE, NOT NULL | — |
| `ip_address` | VARCHAR(45) | NULLABLE | ✓ |
| `expires_at` | TIMESTAMPTZ | NOT NULL | — |
| `is_revoked` | BOOLEAN | NOT NULL | — |

**Retención**: eliminar registros con `expires_at < NOW() - 30 días`.

#### `AuditLogin`
Log inmutable de intentos de autenticación. Solo INSERT en `iam_db`.

| Atributo | Tipo | Restricciones | PII |
|----------|------|---------------|-----|
| `id` | UUID | PK | — |
| `user_id` | UUID | NULLABLE | — |
| `email_attempted` | VARCHAR(255) | NOT NULL | ✓ |
| `outcome` | ENUM(`SUCCESS`, `INVALID_PASSWORD`, `USER_NOT_FOUND`, `ACCOUNT_LOCKED`, `TOKEN_EXPIRED`) | NOT NULL | — |
| `ip_address` | VARCHAR(45) | NULLABLE | ✓ |
| `attempted_at` | TIMESTAMPTZ | NOT NULL | — |

**Retención**: 90 días.

**Relaciones**:
- `User` 1──* `UserRole` *──1 `Role`
- `Role` 1──* `RoleFeature` *──1 `Feature` *──1 `Module`
- `User` 1──* `UserScopeOverride` *──1 `Feature`
- `User` 1──* `RefreshToken`

---

## Bounded Context: `reference-data-service`

### Entidades M2 — Jerarquía Institucional

#### `Macroregion`
| Atributo | Tipo | Restricciones |
|----------|------|---------------|
| `id` | UUID | PK |
| `code` | VARCHAR(10) | UNIQUE |
| `name` | VARCHAR(100) | NOT NULL |

#### `Microregion`
| Atributo | Tipo |
|----------|------|
| `id` | UUID PK |
| `macroregion_id` | FK → Macroregion |
| `code` | VARCHAR(10) UNIQUE |
| `name` | VARCHAR(100) |

#### `Department`
| Atributo | Tipo |
|----------|------|
| `id` | UUID PK |
| `microregion_id` | FK → Microregion |
| `name` | VARCHAR(100) |
| `dane_code` | VARCHAR(5) UNIQUE |

#### `Municipality`
| Atributo | Tipo |
|----------|------|
| `id` | UUID PK |
| `department_id` | FK → Department |
| `name` | VARCHAR(100) |
| `dane_code` | VARCHAR(8) UNIQUE |

#### `TrainingCenter`
| Atributo | Tipo | PII |
|----------|------|-----|
| `id` | UUID PK | — |
| `municipality_id` | FK → Municipality | — |
| `center_code` | VARCHAR(10) UNIQUE | — |
| `name` | VARCHAR(200) | — |
| `address` | TEXT | — |
| `phone` | VARCHAR(20) | ✓ |
| `is_active` | BOOLEAN | — |

#### `InstitutionalUnit`
| Atributo | Tipo |
|----------|------|
| `id` | UUID PK |
| `training_center_id` | FK → TrainingCenter |
| `name` | VARCHAR(200) |
| `unit_type` | VARCHAR(50) |

### Entidades M4 — Catálogos

#### `Catalog`
Agrupador de valores de catálogo.

| Atributo | Tipo |
|----------|------|
| `id` | UUID PK |
| `code` | VARCHAR(50) UNIQUE |
| `name` | VARCHAR(100) |
| `description` | TEXT |
| `is_active` | BOOLEAN |

#### `CatalogDetail`
Valores individuales dentro de un catálogo.

| Atributo | Tipo |
|----------|------|
| `id` | UUID PK |
| `catalog_id` | FK → Catalog |
| `code` | VARCHAR(50) |
| `label` | VARCHAR(255) |
| `display_order` | INTEGER |
| `is_active` | BOOLEAN |

#### `Parameter`
Parámetro de sistema clave-valor (EAV). Tabla independiente, sin relaciones.

| Atributo | Tipo | Restricciones |
|----------|------|---------------|
| `id` | UUID | PK |
| `key` | VARCHAR(100) | UNIQUE, NOT NULL — ej: `MAX_HOURS_PER_WEEK` |
| `value` | TEXT | NOT NULL — almacenado como string; validado según `value_type` en la capa de aplicación |
| `value_type` | ENUM(`integer`, `string`, `boolean`, `json`) | NOT NULL |
| `description` | TEXT | nullable |

**Relaciones**:
- `Macroregion` 1──* `Microregion` ──* `Department` ──* `Municipality` ──* `TrainingCenter` ──* `InstitutionalUnit`
- `Catalog` 1──* `CatalogDetail`
- `Parameter` — sin relaciones (EAV independiente)

---

## Bounded Context: `academic-management-service`

### Entidades M5 — Estructura Curricular

#### `TechLine`
| Atributo | Tipo |
|----------|------|
| `id` | UUID PK |
| `name` | VARCHAR(100) UNIQUE |
| `code` | VARCHAR(10) |

#### `TechNetwork`
| Atributo | Tipo |
|----------|------|
| `id` | UUID PK |
| `tech_line_id` | FK → TechLine |
| `name` | VARCHAR(100) |

#### `KnowledgeNetwork`
| Atributo | Tipo |
|----------|------|
| `id` | UUID PK |
| `tech_network_id` | FK → TechNetwork |
| `name` | VARCHAR(100) |

#### `TrainingProgram`
| Atributo | Tipo | Restricciones |
|----------|------|---------------|
| `id` | UUID | PK |
| `knowledge_network_id` | UUID | FK → KnowledgeNetwork |
| `program_code` | VARCHAR(20) | UNIQUE |
| `name` | VARCHAR(200) | NOT NULL |
| `training_level` | ENUM(`AUXILIARY`, `OPERATOR`, `TECHNICIAN`, `TECHNOLOGIST`) | NOT NULL |
| `total_hours` | INTEGER | NOT NULL |
| `version` | INTEGER | NOT NULL |
| `is_active` | BOOLEAN | DEFAULT true |

#### `Competency`
| Atributo | Tipo |
|----------|------|
| `id` | UUID PK |
| `program_id` | FK → TrainingProgram |
| `sena_code` | VARCHAR(20) UNIQUE |
| `name` | VARCHAR(300) |
| `total_hours` | INTEGER |

#### `LearningOutcome`
| Atributo | Tipo |
|----------|------|
| `id` | UUID PK |
| `competency_id` | FK → Competency |
| `code` | VARCHAR(20) |
| `description` | TEXT |

### Entidades M6 — Fichas y Oferta

#### `EnrollmentFicha`
Instancia operativa de un programa en un centro para una cohorte.

| Atributo | Tipo | Restricciones |
|----------|------|---------------|
| `id` | UUID | PK |
| `program_id` | UUID | FK → TrainingProgram |
| `training_center_id` | UUID | Referencia externa → reference-data-service |
| `ficha_number` | VARCHAR(20) | UNIQUE, NOT NULL |
| `start_date` | DATE | NOT NULL |
| `expected_end_date` | DATE | NOT NULL |
| `learner_count` | INTEGER | NOT NULL |
| `status` | ENUM(`INDUCTION`, `EXECUTION`, `PRODUCTIVE_STAGE`, `COMPLETED`, `CANCELLED`) | NOT NULL |
| `created_at` | TIMESTAMPTZ | — |

**Retención**: `EnrollmentFicha` → 10 años desde `status = COMPLETED` (normativa SENA)

**Relaciones**:
- `TechLine` 1──* `TechNetwork` ──* `KnowledgeNetwork` ──* `TrainingProgram`
- `TrainingProgram` 1──* `Competency` ──* `LearningOutcome`
- `TrainingProgram` 1──* `EnrollmentFicha`

---

## Bounded Context: `training-environment-service`

#### `EnvironmentType`
| Atributo | Tipo |
|----------|------|
| `id` | UUID PK |
| `code` | VARCHAR(20) UNIQUE |
| `name` | VARCHAR(100) |
| `description` | TEXT |

#### `Environment`
Espacio físico donde ocurren las clases.

| Atributo | Tipo | Restricciones |
|----------|------|---------------|
| `id` | UUID | PK |
| `environment_type_id` | UUID | FK → EnvironmentType |
| `training_center_id` | UUID | Referencia externa |
| `name` | VARCHAR(100) | NOT NULL |
| `capacity` | INTEGER | NOT NULL |
| `location` | VARCHAR(200) | — |
| `is_active` | BOOLEAN | DEFAULT true |

#### `AvailabilityRule`
Regla recurrente de disponibilidad del ambiente (hasta 24 reglas por ambiente).

| Atributo | Tipo | Restricciones |
|----------|------|---------------|
| `id` | UUID | PK |
| `environment_id` | UUID | FK → Environment |
| `day_of_week` | SMALLINT | 1=lunes…7=domingo |
| `start_time` | TIME | NOT NULL |
| `end_time` | TIME | NOT NULL |
| `effective_from` | DATE | NOT NULL |
| `effective_until` | DATE | — (null = indefinido) |

#### `Maintenance`
Período de mantenimiento programado que bloquea el ambiente.

| Atributo | Tipo |
|----------|------|
| `id` | UUID PK |
| `environment_id` | FK → Environment |
| `start_datetime` | TIMESTAMPTZ |
| `end_datetime` | TIMESTAMPTZ |
| `reason` | TEXT |
| `created_by` | UUID |

#### `InventoryItem`
Elemento de equipamiento del ambiente.

| Atributo | Tipo |
|----------|------|
| `id` | UUID PK |
| `environment_id` | FK → Environment |
| `name` | VARCHAR(200) |
| `quantity` | INTEGER |
| `condition` | ENUM(`GOOD`, `FAIR`, `POOR`, `OUT_OF_SERVICE`) |

**Relaciones**:
- `Environment` *──1 `EnvironmentType`
- `Environment` 1──* `AvailabilityRule`
- `Environment` 1──* `Maintenance`
- `Environment` 1──* `InventoryItem`

---

## Bounded Context: `scheduling-service`

#### `Schedule`
Agregado raíz del bounded context. Contiene el conjunto de sesiones de clase para una ficha.

| Atributo | Tipo | Restricciones |
|----------|------|---------------|
| `id` | UUID | PK |
| `ficha_id` | UUID | Referencia externa → academic-management-service |
| `name` | VARCHAR(200) | NOT NULL |
| `status` | ENUM(`DRAFT`, `UNDER_REVIEW`, `PUBLISHED`, `ARCHIVED`) | NOT NULL, DEFAULT `DRAFT` |
| `published_at` | TIMESTAMPTZ | — (null hasta publicación) |
| `published_by` | UUID | — |
| `created_by` | UUID | NOT NULL |
| `created_at` | TIMESTAMPTZ | NOT NULL |

**Invariante de dominio**: Una vez `status = PUBLISHED`, ningún atributo ni sesión puede modificarse. Los cambios generan un nuevo `Schedule` con `status = DRAFT`.

#### `TimeSlot`
Franja horaria recurrente (lunes de 07:00 a 10:00).

| Atributo | Tipo |
|----------|------|
| `id` | UUID PK |
| `day_of_week` | SMALLINT (1=lunes…7=domingo) |
| `start_time` | TIME |
| `end_time` | TIME |
| `name` | VARCHAR(50) |

#### `ClassSession`
Instancia de una clase: ficha + instructor + ambiente + franja.

| Atributo | Tipo | Restricciones |
|----------|------|---------------|
| `id` | UUID | PK |
| `schedule_id` | UUID | FK → Schedule |
| `competency_id` | UUID | Referencia externa |
| `instructor_id` | UUID | Referencia externa → actors-service |
| `environment_id` | UUID | Referencia externa → training-environment-service |
| `time_slot_id` | UUID | FK → TimeSlot |
| `session_date` | DATE | NOT NULL |
| `notes` | TEXT | — |

#### `SchedulingConflict`
Conflicto detectado durante la validación. Se elimina al corregir o al descartar el borrador.

> **Enforcement del invariante (ADR-005).** La exclusión mutua (mismo instructor o ambiente en franjas
> solapadas) se garantiza con un **`EXCLUDE` constraint** transaccional en `scheduling_db` — scheduling
> es dueño de todas las `ClassSession`. El `conflict-validator-worker` es **UX** (mostrar conflictos al
> armar el borrador), no el mecanismo de enforcement; así se evita la ventana de carrera de validar async.

| Atributo | Tipo |
|----------|------|
| `id` | UUID PK |
| `schedule_id` | FK → Schedule |
| `conflict_type` | ENUM(`INSTRUCTOR_DOUBLE_BOOKED`, `ENVIRONMENT_DOUBLE_BOOKED`, `SESSIONS_OVERLAP`) |
| `description` | TEXT |
| `session_a_id` | FK → ClassSession |
| `session_b_id` | FK → ClassSession (nullable) |
| `detected_at` | TIMESTAMPTZ |

**Relaciones**:
- `Schedule` 1──* `ClassSession`
- `Schedule` 1──* `SchedulingConflict`
- `ClassSession` *──1 `TimeSlot`

---

## Bounded Context: `actors-service`

#### `Instructor`
Formador vinculado al SENA.

| Atributo | Tipo | PII |
|----------|------|-----|
| `id` | UUID PK | — |
| `user_id` | UUID (referencia externa → iam-service) | — |
| `document_type` | VARCHAR(10) | ✓ |
| `document_number` | VARCHAR(20) UNIQUE | ✓ |
| `full_name` | VARCHAR(200) | ✓ |
| `email` | VARCHAR(255) | ✓ |
| `phone` | VARCHAR(20) | ✓ |
| `max_hours_per_week` | DECIMAL(4,1) | — |
| `is_active` | BOOLEAN | — |

#### `InstructorContract`
Vinculación laboral del instructor (puede haber varias en el tiempo).

| Atributo | Tipo |
|----------|------|
| `id` | UUID PK |
| `instructor_id` | FK → Instructor |
| `contract_type` | ENUM(`STAFF`, `CONTRACTOR`) |
| `start_date` | DATE |
| `end_date` | DATE (nullable) |
| `training_center_id` | UUID (referencia externa) |

#### `CompetencyAssignment`
Competencias que el instructor está certificado para impartir.

| Atributo | Tipo |
|----------|------|
| `id` | UUID PK |
| `instructor_id` | FK → Instructor |
| `competency_id` | UUID (referencia externa → academic-management-service) |
| `certified_at` | DATE |
| `is_active` | BOOLEAN |

#### `Learner`
Aprendiz en proceso de formación.

| Atributo | Tipo | PII |
|----------|------|-----|
| `id` | UUID PK | — |
| `user_id` | UUID (referencia externa) | — |
| `document_type` | VARCHAR(10) | ✓ |
| `document_number` | VARCHAR(20) UNIQUE | ✓ |
| `full_name` | VARCHAR(200) | ✓ |
| `email` | VARCHAR(255) | ✓ |
| `ficha_id` | UUID (referencia externa) | — |
| `enrollment_status` | ENUM(`ACTIVE`, `DROPOUT`, `TRANSFERRED`, `COMPLETED`) | — |
| `current_stage` | ENUM(`LECTURE`, `PRODUCTIVE`) | — |

#### `Company`
Empresa que participa en etapa productiva.

| Atributo | Tipo | PII |
|----------|------|-----|
| `id` | UUID PK | — |
| `nit` | VARCHAR(20) UNIQUE | — |
| `business_name` | VARCHAR(200) | — |
| `contact_name` | VARCHAR(200) | ✓ |
| `contact_email` | VARCHAR(255) | ✓ |
| `contact_phone` | VARCHAR(20) | ✓ |

#### `ProductiveStage`
Etapa productiva de un aprendiz en una empresa.

| Atributo | Tipo |
|----------|------|
| `id` | UUID PK |
| `learner_id` | FK → Learner |
| `company_id` | FK → Company |
| `start_date` | DATE |
| `end_date` | DATE |
| `supervisor_instructor_id` | FK → Instructor |
| `status_id` | UUID FK → Status (catálogo `status`) |

#### `CompanyVisit`
Visita de seguimiento a empresa por el instructor supervisor.

| Atributo | Tipo |
|----------|------|
| `id` | UUID PK |
| `productive_stage_id` | FK → ProductiveStage |
| `instructor_id` | FK → Instructor |
| `visit_date` | DATE |
| `observations` | TEXT |
| `next_visit_date` | DATE |

**Retención de PII**: Datos de `Instructor` y `Learner` → 5 años post-desvinculación (normativa SENA)

---

## Bounded Context: `document-service`

#### `DocumentTemplate`
Plantilla para generación de documentos.

| Atributo | Tipo |
|----------|------|
| `id` | UUID PK |
| `code` | VARCHAR(50) UNIQUE |
| `name` | VARCHAR(200) |
| `template_body` | TEXT (HTML/Handlebars) |
| `version` | INTEGER |
| `is_active` | BOOLEAN |

#### `Document`
Documento generado en el sistema.

| Atributo | Tipo | Restricciones |
|----------|------|---------------|
| `id` | UUID | PK |
| `template_id` | UUID | FK → DocumentTemplate (nullable si es libre) |
| `title` | VARCHAR(300) | NOT NULL |
| `owner_service` | VARCHAR(50) | Servicio propietario. Ej: `scheduling`, `academic-management` |
| `owner_entity_id` | UUID | ID del objeto de negocio propietario en `owner_service` |
| `storage_key` | VARCHAR(500) | Ruta en object storage |
| `mime_type` | VARCHAR(100) | |
| `size_bytes` | BIGINT | |
| `created_by` | UUID | |
| `created_at` | TIMESTAMPTZ | |

**Invariante**: Los binarios NUNCA se almacenan en la BD. Solo `storage_key` apunta al object storage.

#### `DocumentVersion`
Versiones de un documento.

| Atributo | Tipo |
|----------|------|
| `id` | UUID PK |
| `document_id` | FK → Document |
| `version_number` | INTEGER |
| `storage_key` | VARCHAR(500) |
| `created_by` | UUID |
| `created_at` | TIMESTAMPTZ |

**Retención**: `Document` → política variable según tipo; mínimo 5 años para documentos académicos

---

## Bounded Context: `monitoring-service`

#### `FichaTracking`
Estado de seguimiento de una ficha activa.

| Atributo | Tipo |
|----------|------|
| `id` | UUID PK |
| `ficha_id` | UUID (referencia externa) |
| `assigned_instructor_id` | UUID (referencia externa) |
| `status` | ENUM(`ON_TRACK`, `AT_RISK`, `CRITICAL`) |
| `last_updated_at` | TIMESTAMPTZ |

#### `TrackingSession`
Sesión de seguimiento periódico a una ficha.

| Atributo | Tipo |
|----------|------|
| `id` | UUID PK |
| `ficha_tracking_id` | FK → FichaTracking |
| `session_date` | DATE |
| `instructor_id` | UUID |
| `attendance_percentage` | DECIMAL(5,2) |
| `progress_percentage` | DECIMAL(5,2) |
| `notes` | TEXT |

#### `KpiTracking`
Medición puntual de un KPI para una ficha.

| Atributo | Tipo |
|----------|------|
| `id` | UUID PK |
| `ficha_tracking_id` | FK → FichaTracking |
| `kpi_code` | VARCHAR(50) |
| `value` | DECIMAL(10,4) |
| `measured_at` | TIMESTAMPTZ |

#### `GeneratedAlert`
Alerta generada cuando un KPI supera un umbral.

| Atributo | Tipo |
|----------|------|
| `id` | UUID PK |
| `ficha_tracking_id` | FK → FichaTracking |
| `alert_type` | VARCHAR(50) |
| `severity` | ENUM(`LOW`, `MEDIUM`, `HIGH`, `CRITICAL`) |
| `message` | TEXT |
| `is_resolved` | BOOLEAN DEFAULT false |
| `generated_at` | TIMESTAMPTZ |
| `resolved_at` | TIMESTAMPTZ (nullable) |

#### `ImprovementPlan`
Plan de mejoramiento generado a partir de una alerta o seguimiento.

| Atributo | Tipo |
|----------|------|
| `id` | UUID PK |
| `ficha_tracking_id` | FK → FichaTracking |
| `generated_alert_id` | FK → GeneratedAlert (nullable) |
| `description` | TEXT |
| `due_date` | DATE |
| `status` | ENUM(`PENDING`, `IN_PROGRESS`, `COMPLETED`, `CANCELLED`) |
| `created_by` | UUID |

#### `SentNotification`
Registro de notificaciones enviadas a actores del sistema.

| Atributo | Tipo | PII |
|----------|------|-----|
| `id` | UUID PK | — |
| `recipient_id` | UUID | — |
| `recipient_email` | VARCHAR(255) | ✓ |
| `channel` | ENUM(`EMAIL`, `IN_APP`) | — |
| `subject` | VARCHAR(300) | — |
| `sent_at` | TIMESTAMPTZ | — |
| `status` | ENUM(`SENT`, `FAILED`, `PENDING`) | — |

**Retención**: `SentNotification` → 1 año

> **Nota (ADR-005):** la **entrega** de notificaciones se extrae a `notification-service`; monitoring
> solo detecta y publica eventos (`alert.triggered`, `kpi.threshold_breached`). `SentNotification` pasa
> al bounded context `notification-service` (abajo), en `notification_db`.

---

## Bounded Context: `notification-service`

Capacidad transversal (ADR-005): entrega notificaciones a los actores por email / in-app. Consume
eventos de dominio (`scheduling.schedule.published`, `monitoring.alert.triggered`, …) o solicitudes
síncronas (`POST /notifications`). Ningún dominio invoca su worker directamente — solo su contrato.
`SentNotification` (arriba) es su entidad; opcionalmente `NotificationTemplate` y `NotificationPreference`.

---

## Bounded Context: `audit-service`

#### `AuditRecord`
Registro inmutable de cada evento de negocio.

| Atributo | Tipo | Restricciones |
|----------|------|---------------|
| `id` | UUID | PK |
| `event_id` | UUID | UNIQUE (idempotencia) |
| `event_type` | VARCHAR(100) | NOT NULL |
| `source_service` | VARCHAR(50) | NOT NULL |
| `actor_id` | UUID | nullable (acciones del sistema) |
| `entity_type` | VARCHAR(50) | — |
| `entity_id` | UUID | — |
| `payload` | JSONB | NOT NULL |
| `recorded_at` | TIMESTAMPTZ | NOT NULL, DEFAULT NOW() |

**Invariante absoluta**: SOLO INSERT. Ninguna operación UPDATE o DELETE está permitida en `AuditRecord`.
**Retención**: 7 años (normativa de auditoría)

---

## Patrones de acceso — Queries críticos

| Query | Servicio | Frecuencia | Latencia objetivo |
|-------|----------|-----------|-------------------|
| Ambientes disponibles para franja horaria | `training-environment-service` | Alta (durante creación de horario) | < 300 ms |
| Instructores disponibles con competencia | `actors-service` | Alta | < 300 ms |
| Sesiones de clase por instructor y semana | `scheduling-service` | Alta (consulta de horario) | < 500 ms |
| Fichas activas de un centro | `academic-management-service` | Media | < 500 ms |
| KPIs de una ficha | `monitoring-service` | Media | < 1 s |
| Catálogos por código | `reference-data-service` | Muy alta (caché) | < 50 ms (Redis) |
| Historial de auditoría por entidad | `audit-service` | Baja | < 2 s |

---

## Referencias cruzadas — Claves externas lógicas

Las entidades que referencian datos de otro bounded context NO usan foreign keys de BD. La consistencia se garantiza por:
1. **Evento de dominio**: al crear/archivar la entidad fuente, el consumidor actualiza su referencia local
2. **Validación en API**: el servicio valida la existencia consultando el servicio propietario al recibir el dato

| Referencia | Servicio consumidor | Servicio propietario |
|-----------|--------------------|--------------------|
| `training_center_id` en `EnrollmentFicha` | `academic-management-service` | `reference-data-service` |
| `instructor_id` en `ClassSession` | `scheduling-service` | `actors-service` |
| `environment_id` en `ClassSession` | `scheduling-service` | `training-environment-service` |
| `competency_id` en `CompetencyAssignment` | `actors-service` | `academic-management-service` |
| `ficha_id` en `FichaTracking` | `monitoring-service` | `academic-management-service` |

---

## Checklist A10 — Cumplimiento

- [x] Nombres de entidades en inglés, singular, ASCII
- [x] Nombres de atributos en inglés, snake_case
- [x] Tipos de datos definidos (UUID, VARCHAR, TIMESTAMPTZ, ENUM)
- [x] Restricciones principales (PK, UNIQUE, NOT NULL) declaradas
- [x] Atributos PII marcados
- [x] Políticas de retención definidas para entidades con datos sensibles o regulados
- [x] Invariantes de dominio documentadas (`Schedule` inmutable en PUBLISHED; `AuditRecord` solo INSERT)
- [x] Patrones de acceso críticos identificados con latencia objetivo
- [x] Referencias cruzadas entre bounded contexts documentadas
- [ ] Revisar queries críticos en `training-environment-service` con equipo de BD para confirmar índices necesarios
- [ ] Confirmar retención exacta con área jurídica SENA para documentos académicos
