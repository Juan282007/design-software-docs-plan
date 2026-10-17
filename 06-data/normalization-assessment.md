# Evaluación de normalización y calidad del modelo de datos

> Estado: 🟢 Estable | Última actualización: 2026-08-01
> Autor: Jesús Ariel González Bonilla (PM + Arquitecto) | Equipo: Datos

Evaluación del modelo de datos **físico real** de los 9 servicios (esquemas PostgreSQL en los repos `*-db`). Se analiza el nivel de forma normal alcanzado y la calidad de integridad. Complementa a [modeling-conventions.md](./modeling-conventions.md) y al [data-dictionary.md](./data-dictionary.md).

> **Contexto — sistema de microservicios.** Cada servicio es dueño de su esquema. Las referencias a entidades de **otros** servicios se guardan como `UUID` **sin FK física** (integridad garantizada por contrato/evento, no por la BD). Esto **no** es una violación de diseño: es el patrón correcto de desacople entre microservicios. Por tanto, la ausencia de FK cross-servicio no baja la calificación.

## Tabla de estado (por servicio)

| # | Servicio | Tablas | Nivel NF | Calidad | Nota clave |
|---|----------|:-----:|----------|---------|-----------|
| 1 | reference-data | 9 | **BCNF** | 🟢 Alta | Jerarquía geográfica textbook + catálogos parametrizables (`catalog`/`catalog_detail`/`parameter`). Modelo de referencia. |
| 2 | training-environment | 7 | **BCNF** | 🟢 Alta | Sin violaciones; `EXCLUDE` anti-solape. Pulir: `TIMESTAMP`→`TIMESTAMPTZ`. |
| 3 | iam | 10 | **3NF/BCNF** | 🟢 Alta | RBAC bien normalizado. Ajustar: UNIQUE con `training_center_id` NULL permite duplicados lógicos. |
| 4 | actors | 15 | **3NF** | 🟢 Alta | Quitar transitiva `status_transition.status_category_id`. Resto BCNF. |
| 5 | academic-management | 7 | **3NF/BCNF** | 🟢 Alta | Añadir CHECK a `enrollment_ficha.status`; cualificar schema (sacar de `public`); unificar semántica de versión. |
| 6 | monitoring | 10 | **3NF** | 🟢 Alta | Resolver agregado `ficha_tracking.active_alert_count` (vista/derivado o trigger auditado); añadir índices de FK. |
| 7 | scheduling | 4 | **2NF** (excepción justificada) | 🟢 Alta | Redundancia temporal exigida por el `EXCLUDE`. Ver [Excepciones](#excepciones-de-normalización-intencionales). |
| 8 | audit | 1 | **1NF** (event store) | 🟢 Alta | Desnormalización intencional (`JSONB` + columnas extraídas), append-only. Ver [Excepciones](#excepciones-de-normalización-intencionales). |
| 9 | document | 3 | **3NF/BCNF** | 🟢 Alta | Reforzar integridad: CHECK en estados, índices de FK, UNIQUE `(document_id, version_number)`, `TIMESTAMPTZ`, `created_by` UUID. |

**Leyenda NF:** 1NF (atómico) → 2NF (sin dependencia parcial de PK compuesta) → 3NF (sin dependencia transitiva) → BCNF (todo determinante es clave candidata).

> La columna **Calidad** refleja el **objetivo (target) del modelo** una vez aplicadas las mejoras de la sección siguiente. Los servicios ya en Alta sin salvedades: reference-data, training-environment, iam, actors.

## Excepciones de normalización intencionales

Dos servicios están **por debajo de 3NF a propósito**, por patrones válidos en un sistema de microservicios. **No deben "corregirse" a 3NF estricto.**

### `audit` — Event store (1NF)
`audit_record` es un almacén de eventos append-only e inmutable. El `payload JSONB` (schema-on-read) y las columnas extraídas (`event_type`, `actor_id`, `entity_type`, `entity_id`) que lo duplican para indexar/filtrar son deliberadas. `UNIQUE(event_id)` garantiza idempotencia. Sin FKs por diseño (sink desacoplado). **Correcto.**

### `scheduling` — Redundancia por `EXCLUDE` (2NF)
`class_session` almacena `session_date`, `start_time`, `end_time`, `day_of_week` que dependen transitivamente de `time_slot_id`. Es **requerido**: las restricciones `EXCLUDE USING gist` anti-solapamiento (instructor/ambiente) no pueden hacer join contra `time_slot`. Trade-off consciente.
- **Salvaguarda obligatoria:** un **trigger/CHECK** que garantice que los valores locales de `class_session` coinciden con los de su `time_slot` (evita inconsistencia). Documentar la excepción en el `data-model.md` del servicio.

## Brechas de calidad a cerrar (para Alta uniforme)

Violaciones **accidentales** de 3NF a corregir (solo 2):
1. `actors.status_transition.status_category_id` — transitiva (categoría derivable de `from_status_id`). Eliminar la columna.
2. `monitoring.ficha_tracking.active_alert_count` — agregado almacenado no-`GENERATED`. Convertir en vista/derivado o mantener por trigger auditado.

Refuerzos de integridad (no de forma normal):
- **document (prioridad):** CHECK en `status`/`output_type`/`state`; índices en FK; UNIQUE `(document_id, version_number)`; `TIMESTAMPTZ`; `created_by/updated_by` como `UUID`.
- **iam:** rehacer el UNIQUE de `user_role` para tratar `training_center_id` NULL sin duplicados (índice único con `COALESCE` o parcial).
- **academic:** CHECK en `enrollment_ficha.status`; cualificar el schema del módulo.
- **monitoring:** índices en FK (`improvement_plan.*`, `sent_notification.*`, `kpi_tracking.kpi_status_id`); CHECK de dominio (porcentajes 0-100, `attendance_count ≤ total_learners`); CHECK de formato en `color_hex`.
- **training / reference:** `TIMESTAMPTZ` y columnas de auditoría faltantes en tablas geográficas.
- **Transversal:** mover textos libres a catálogos parametrizables (`instructor_area.area_name`, `company.economic_activity`, `event_type`).

## Veredicto

Modelo **bien normalizado**: 7 de 9 servicios en 3NF/BCNF; los 2 por debajo lo están por **diseño intencional y correcto** para microservicios (event store y restricción de exclusión temporal). El foco de mejora **no es la normalización sino el enforcement de integridad** (CHECK/UNIQUE/índices), concentrado en `document` y en ajustes puntuales de iam/monitoring/academic/actors.
