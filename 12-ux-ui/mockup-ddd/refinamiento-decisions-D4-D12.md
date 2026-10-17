<!-- RESUMEN-EJECUTIVO
agente: Jesús Ariel González Bonilla (PM + Arquitecto)
capacidad: registro oficial de las decisiones de negocio D-4..D-12 y su implementación
fase: diseño/implementación — refinamiento
estado: aprobado (base: documento de resolución D-4..D-12)
dependencias_entrada: refinamiento-alignment.md; matriz de triage; changesets refinamiento-*-980/981/970
consumidores_siguientes: backend/frontend cuando se construyan; ADRs
tldr: Resoluciones de las 9 decisiones abiertas D-4..D-12 y qué se implementó (BD aditiva en los 4 ambientes) vs qué queda V2/postergado.
decisiones_clave: MVP sin sobre-ingeniería; separar autorización/auditoría/seguimiento/soporte; evolución progresiva
halts_registrados: ninguno
-->

# Registro de decisiones D-4 … D-12 (aprobadas)

Base: documento de resolución técnica D-4..D-12 (Product/Arquitectura). Aquí queda el
resultado **y su estado de implementación** en el proyecto. Cambios de BD = **aditivos**,
validados con rebuild limpio 9/9 y desplegados en los 4 ambientes (develop/qa/staging/main).

| # | Decisión | Estado | Implementado (aditivo) | V2 / Pendiente |
|---|---|---|---|---|
| **D-4** | Sesión ejecutada ≠ asistencia individual | Aprobada | `scheduling.class_session.execution_status` (PENDING/EXECUTED/NOT_EXECUTED/CANCELLED) — `refinamiento-scheduling-d4-execution-980` | Tabla `learner_attendance` (asistencia por aprendiz) |
| **D-5** | Director ≠ Administrador | Aprobada | Ya diferenciado en rbac (CENTER_DIRECTOR vs ADMIN_STAFF/SYSTEM_ADMIN); scopes GLOBAL/TRAINING_CENTER | Jerarquía Regional→Centro→Sede como autorización |
| **D-6** | Horario PUBLISHED inmutable | Aprobada | Regla sobre estados existentes (`schedule.status` DRAFT/UNDER_REVIEW/PUBLISHED/ARCHIVED) — **sin cambio de BD** | Tabla de versiones explícita si auditoría normativa lo exige |
| **D-7** | Módulo de soporte / tickets | **Postergada** | — (no crear entidad ni servicio) | `support-service` tras definir flujo/SLA/ownership |
| **D-8** | `status_transition` transversal | Aprobada | Incremental donde exista máquina de estados; **no refactor masivo** — auditoría conserva la evidencia | Extensión a nuevos agregados al intervenirlos |
| **D-9** | Versionado de plantillas | Aprobada MVP | `document.document_template_version` (HTML, DRAFT/UNDER_REVIEW/PUBLISHED/OBSOLETE), `document_template_variable`, `document.template_version_id`, índice "una sola PUBLISHED" — `refinamiento-document-d9-*-970` | Gobierno documental avanzado |
| **D-10** | Estados de matrícula | **Sugerencia futura** | — (sin cambios; requiere validación institucional) | Catálogo canónico (separar ficha/matrícula/disciplinario) |
| **D-11** | Traslado entre sedes | Aprobada MVP | `conflict_type` += `INSTRUCTOR_TRAVEL_CONFLICT` (6º); tabla `scheduling.site_travel_time` — `refinamiento-scheduling-d11-travel-981` | Reglas geográficas más finas |
| **D-12** | Idempotencia / dedup de eventos | Aprobada MVP | Índice único parcial `uq_sent_notification_dedup` (`source_event_id, recipient_id, channel, category`) — `refinamiento-monitoring-d12-dedup-980` | Tabla transversal `processed_event` (patrón por consumidor); acuse formal |

## Reconciliaciones aplicadas contra la realidad de los repos

- **D-12**: se usaron los campos reales `source_event_id`/`category` (G2), no `event_id`/`notification_type` del modelo conceptual. `processed_event` queda como **patrón** (no tabla especulativa) para evitar sobre-ingeniería.
- **D-11**: `INSTRUCTOR_TRAVEL_CONFLICT` es el **6º** `conflict_type` (los 5 previos venían de D-1); `site_travel_time` usa refs lógicas a sedes de `reference-data` (sin FK física cross-servicio). Mismo-sitio permitido (`travel_minutes` puede ser 0).
- **D-9**: se **conserva** `document_template.template_body` (no-daño); el versionado HTML vive en la nueva `document_template_version`. FKs internas al schema `document` (permitidas).
- **D-10**: se respetó la reversión respecto a la recomendación previa (no tocar hasta validación institucional).

## Consistencia contrato ↔ BD

- `07-api/contracts/openapi/scheduling.yaml`: enum de `conflict_type` actualizado a los **6** valores.
- `09-microservices/services/05-scheduling-service/data-model.md`: conflicto (6 tipos + severidad/resolución/bloqueo), `execution_status`, `site_travel_time`.

Los cambios de BD siguieron el flujo endurecido: rama hija por ambiente + PR a su padre, sin merge entre padres ni `-X theirs`.
