<!-- RESUMEN-EJECUTIVO
agente: Jesús Ariel González Bonilla (PM + Arquitecto)
capacidad: alineación retros(aprendices) ↔ estado real de repos ↔ mockup ↔ cambio de modelo
fase: diseño (UX/UI) — refinamiento preliminar
estado: draft
dependencias_entrada: retros de refinamiento (22 docs); repos design-software-*-db (estado real verificado); mockup-ddd/flows/*; discovery-findings.md
consumidores_siguientes: changesets aditivos por repo -db (cuando se autorice), refinamiento de contratos/HU/RBAC
tldr: Cierra el ciclo "mejorar mockup → detectar fugas → validar aporte al core sin dañar la BD". Tría cada propuesta de las retros contra el estado REAL de los repos (que las retros no vieron), materializa en el mockup lo que sobrevive, y especifica el cambio de modelo aditivo. Nada de la BD congelada se recrea.
decisiones_clave: retros ciegas a repos → triar contra estado real; solo cambios aditivos; mockup-first; D-1=5 tipos de conflicto, D-2=evento de dominio, D-3=RAP opcional en MVP
halts_registrados: ninguno (B1 resuelto con audit.yaml — API de solo lectura de auditoría)
-->

# Refinamiento — Alineación retros ↔ modelo ↔ BD ↔ mockup

> **ESTADO: PRELIMINAR.** Este documento cierra el ciclo pedido: *refinar → mejorar el mockup →
> dejar que revele fugas/incoherencias → validar con rigor que el cambio aporte al core del negocio
> sin dañar lo que ya está construido y verificado.*

## 1. Contexto y método

Las **22 retrospectivas de aprendices** (`/Refinamiento requerimientos`) auditaron el **mockup público
estático**, **sin acceso** a los repos `-db`, contratos ni data-model reales. Por eso una parte grande de
sus "hallazgos" son artefactos del prototipo o cosas **que ya existen** en la BD. El método aquí:

1. **Extracción** a nivel de ítem de las 22 retros (propuestas concretas de FUNC/BD/API).
2. **Triage contra el estado REAL** verificado por inspección de los repos `-db`.
3. **Materializar en el mockup** lo que aporta al core (instrumento barato de validación).
4. **Especificar el cambio de modelo** aditivo (solo `ADD`), sin tocar tablas congeladas.

Regla de no-daño: solo `ADD COLUMN` / tabla nueva / valor de enum; changeset id único por repo; FKs en
`04_alter`; rollback espejo; validar con `down -v` + rebuild limpio. Ver [discovery-findings.md](discovery-findings.md).

## 2. Decisiones fijadas (defaults razonados)

| # | Decisión | Default adoptado | Razón |
|---|---|---|---|
| D-1 | Catálogo `conflict_type` | **5 canónicos** (functional.md/HU-09): INSTRUCTOR_DOUBLE_BOOKED, ENVIRONMENT_DOUBLE_BOOKED, SESSIONS_OVERLAP, ENVIRONMENT_MAINTENANCE, INSTRUCTOR_UNAVAILABLE | `CAPACITY_EXCEEDED`/`WORKLOAD_EXCEEDED` son validaciones de límite, no cruces |
| D-2 | Fuente de notificación al publicar | **Evento de dominio** `scheduling.schedule.published`/`.updated` → monitoring | Ya hay event-catalog y patrón sin FK cross-servicio |
| D-3 | RAP en la sesión | **`learning_outcome_id` NULLABLE** en MVP; obligatoriedad y % de avance después | Forzarlo bloquearía crear horarios; la fórmula de avance es 💬 |
| — | 2FA para roles admin | **V2** (mantener JWT+RBAC) | Divergencia entre revisores; mayoría lo manda a V2 |

## 3. Matriz de alineación (consolidado 22 retros × estado real)

### 🟢 GAP ADITIVO — aporta al core, no daña
| # | Tema | Estado real | Cambio aditivo |
|---|---|---|---|
| G1 | Ciclo de vida del conflicto | `scheduling_conflict` con `is_resolved`,`detected_at`,`conflict_type`(3) | +severity, resolved_by/at, resolution_reason, is_blocking_publication; +2 enum |
| G2 | Notificación accionable | `sent_notification` con `send_status`,`channel` | +is_read/read_at, category, priority, deep_link, source_event_id/type |
| G3 | Workflow excepción instructor | `instructor_availability_exception` con `exception_type`,`approved_by` | +status, support_document_id, reviewed_by/at, decision_reason |
| G4 | RAP en la sesión | `class_session.competency_id`; `academic.learning_outcome` existe | +learning_outcome_id UUID NULL (ref lógica) |
| G5 | Aprendiz ve su horario (B2) | RBAC exige `SCH_VIEW_ALL` | feature/scope `SCH_VIEW_OWN` (seed+contrato) |
| G6 | Aptitud del ambiente | `environment`,`maintenance`,`inventory_item` ya existen | +inspection_status, certification_type_id, required_equipment |
| G7 | Historial de rol (motivo) | `user_role.assigned_by/at,expires_at` | +assignment_reason, revoked_by/at |
| G8 | Correlación en auditoría | `audit_record` normalizada+indexada, `payload JSONB` | +correlation_id |
| G9 | Fixes de semilla | prog. 233104 (ADSO real 228106); `NIGTH`→`NIGHT`; rol `AREA_LEADER` sin features | corregir seeds/CHECK |
| G10 | Tipo de vinculación instructor | `instructor` sin `contract_type` | +contract_type (planta/contratista) |

### ⚪ YA EXISTE (falso hallazgo del prototipo — no reabrir)
Publicación controlada + estados + `row_version` (schedule) · aforo/mantenimiento/inventario (training_environment) ·
`instructor.default_max_hours_per_week` · auditoría normalizada e indexada · `learner_ficha_enrollment` (múltiples fichas) ·
`class_session.competency_id` · paginación/filtros (limitación del mockup, no de la BD).

### 🟣 DECISIÓN PM (💬) — RESUELTAS
D-4…D-12 fueron resueltas y registradas en **[refinamiento-decisions-D4-D12.md](refinamiento-decisions-D4-D12.md)**.
Implementadas en BD (aditivo, 4 ambientes): D-4 (`execution_status`), D-9 (versionado de plantillas), D-11
(`INSTRUCTOR_TRAVEL_CONFLICT` + `site_travel_time`), D-12 (índice de dedup). Reglas sin cambio de BD: D-5, D-6, D-8.
Postergadas: D-7 (soporte), D-10 (estados de matrícula — validación institucional).

### 🔴 FUERA DE ALCANCE (❌ — consenso)
IA generadora · MFA/SSO/2FA/SMS · SOFIA Plus/ZAJUNA · PWA offline/nativa · control por voz · multi-idioma · modo oscuro ·
export .ics / sync Google-Outlook · firma digital/TRD. *(Divergencias: 2FA-admin y .ics tienen proponentes minoritarios → V2.)*

### Bloqueantes reales que NO son de BD (contrato/RBAC)
- **B1** · ~~audit-service sin API REST~~ **RESUELTO** → `audit.yaml` (API solo lectura: `/audit-records` cursor+filtros, `/{id}`, `/export`) + componente `audit-query-api`. Pantalla de auditoría ya construible.
- **B2** · `SCH_VIEW_OWN` ausente → aprendiz no ve su horario (resuelto en diseño vía G5).
- **B3** · "mis fichas" del instructor exige `MON_DASHBOARD_FULL` (falta `MON_TRACK_OWN`).

## 4. Materializado en el mockup (mockup-first, verificado en DOM)

| Gap | Ruta del mockup | Qué muestra |
|---|---|---|
| G1 | `#/horarios/sch-02/conflictos` (`?modal=resolve`) | 5 tipos + severidad + Bloquea/No bloquea + resolución auditada + justificación obligatoria |
| G2 | topbar `?overlay=notifications` · `#/notificaciones` | Nueva(leído/no-leído) + prioridad + categoría, separado del estado de entrega; badge dinámico |
| G3 | `#/instructor/mi-disponibilidad` (`?modal=exception`) | Estado del flujo + soporte documental + revisor; regla Anular(pendiente)/Eliminar; upload en modal |
| G5 | `#/mi-horario` | Aviso de scope propio `SCH_VIEW_OWN` (solo tu ficha) |
| G6 | `#/disponibilidad/ambientes/env-01` | Panel "Aptitud del ambiente": aforo, inspección, certificación, mantenimiento |
| G7 | `#/admin/usuarios/usr-02?modal=role` | Campo "Motivo de la asignación" (historial de rol auditado) |
| G8 | `#/auditoria` | Filtro y campo `correlation_id` en el payload |
| G9b | `#/admin/parametrizacion/jornadas` | Typo `NIGTH`→`NIGHT` corregido |

Archivos del mockup editados (working-tree, **sin commit**): `data/mock-data.js`, `assets/components.js`,
`shell/shell.js`, `monitoring/screens.js`, `actors/screens.js`, `scheduling/screens.js`, `iam/screens.js`,
`audit/screens.js`, `environment/screens.js`. Validación: `node tools/validate-routes.js` → **53/53**.

## 5. Cambio de modelo propuesto (spec aditivo — NO aplicado)

Solo se aplica lo que sobrevivió el triage y se validó en el mockup. Cada uno es `ADD`, con changeset id
único por repo y rollback espejo. Ver el detalle DDL en el spec de trabajo.

| Repo | Objeto | Cambio | changeset |
|---|---|---|---|
| scheduling-db | `scheduling_conflict` | +5 columnas; ampliar CHECK `conflict_type` a 5 | `refinamiento-scheduling-conflict-001` |
| monitoring-db | `sent_notification` | +7 columnas (lectura/prioridad/categoría/deep-link/evento) + índice parcial | `refinamiento-monitoring-notification-001` |
| actors-db | `instructor_availability_exception` | +5 columnas (workflow + soporte + revisor) | `refinamiento-actors-exception-001` |
| ~~iam-db seed rbac~~ | **NO requerido** | el rbac de BD **ya tenía** `SCH_VIEW_OWN` y `MON_DASHBOARD_OWN`; el hueco era el OpenAPI | — |
| docs `07-api` | contratos openapi | `x-required-feature-any: [ALL, OWN]` en 5 GET de scheduling + `/ficha-trackings` (cierra B2/B3) | rama `feat/refinamiento-rbac-own-*` |

Decisiones de no-daño: "sesiones afectadas" por una excepción se **computan por query** (no tabla puente
cross-servicio); defaults de columnas elegidos para no invalidar filas existentes; coherencia `resolved_*`
validada en app, no con CHECK.

## 6. Pendientes / próximos pasos

1. **Cerrar decisiones 💬** (al menos las del core: D-1/D-2/D-3 ya con default; faltan D-4…D-12).
2. **Redactar changesets** de la §5 en rama hija de `develop` — requiere autorización expresa para escribir
   SQL y, aparte, para commitear/pushear (ver skill design-software-git-flow).
3. **B1/B2/B3 cerrados**: B2/B3 con `x-required-feature-any` (scheduling/monitoring); B1 con `audit.yaml` (API solo lectura). **Sin bloqueantes abiertos** — quedan solo las decisiones D-4…D-12.
4. Regenerar capturas de referencia de las pantallas tocadas cuando el diseño se estabilice.
