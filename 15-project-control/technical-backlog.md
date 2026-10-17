# Backlog Técnico

> Estado: 🟡 En progreso | Última actualización: 2026-08-28
> Autor: Jesús Ariel González Bonilla (PM + Arquitecto) | Equipo: Arquitectura

Deuda técnica conocida y pendientes de construcción del proyecto **Horarios SENA**. La plataforma combina repositorios de datos y aplicaciones con madurez desigual; este backlog registra trabajo estructural pendiente sin asumir un único estado para todos los servicios.

Ver contexto en [service-catalog.md](../09-microservices/service-catalog.md), [migration-strategy.md](../06-data/migration-strategy.md) y [dependency-map.md](../09-microservices/dependency-map.md).

## Leyenda

- **Impacto**: Bajo / Medio / Alto — efecto sobre despliegue, integridad o velocidad del equipo.
- **Prioridad**: P0 (bloqueante) · P1 (alta) · P2 (media) · P3 (baja).
- **Estado**: 🔴 Abierto · 🟡 En curso · 🟢 Resuelto.

## Deuda técnica activa

| ID | Descripción | Impacto | Prioridad | Servicio(s) afectado(s) | Estado |
|----|-------------|---------|-----------|-------------------------|--------|
| TD-001 | **Secretos `.env.*` versionados**: los archivos de entorno (`.env.develop/.qa/.staging/.main`) con credenciales de conexión están bajo control de versiones. Requiere purga del historial, rotación de credenciales y mover secretos a un gestor (Secret Manager) referenciado desde `.gitignore`. | Alto | P0 | Todos los `*-db` | 🔴 |
| TD-002 | **Doble anidamiento de carpetas en los repos `-db`**: la estructura DDL quedó anidada un nivel de más (p. ej. `db/db/...`), lo que rompe rutas relativas y complica el `changelog-master`. Aplanar a la estructura de [modeling-conventions §7](../06-data/modeling-conventions.md#7-estructura-ddl-y-orden-de-aplicación-liquibase). | Alto | P0 | Varios `*-db` | 🔴 |
| TD-003 | **Changelogs Liquibase con rutas rotas**: varios `changelog-master.yaml` incluyen rutas que no resuelven (consecuencia de TD-002 y de renombrados), lo que impide un `liquibase update` limpio desde cero. Validar arranque en base vacía por cada repo. | Alto | P0 | Varios `*-db` | 🔴 |
| TD-004 | **Driver JDBC de PostgreSQL obsoleto**: la versión de driver empaquetada con Liquibase está desactualizada frente a PostgreSQL 16. Actualizar a una versión soportada y fijarla explícitamente para reproducibilidad. | Medio | P1 | Todos los `*-db` | 🔴 |
| TD-005 | **Falta de índices en llaves foráneas**: las FK declaradas no tienen su índice de apoyo, en contra de la regla "un índice por cada FK" de [migration-strategy.md](../06-data/migration-strategy.md). Degrada joins y borrados en cascada. | Medio | P1 | `document-service`, `monitoring-service` | 🔴 |
| TD-006 | **`document-service` sin restricciones ni índices**: el esquema de `document` carece de `CHECK` de dominio (estados/tipos), de índices de consulta y de índices de FK. Reforzar antes de exponer la API y alinear con [ADR-003](../05-architecture/decisions/records/ADR-003-object-storage.md) y [ADR-004](../05-architecture/decisions/records/ADR-004-status-parametrization-and-audit-standard.md). | Medio | P1 | `document-service` | 🔴 |
| TD-007 | **Convención de ramas inconsistente entre repos**: no todos los repos `-db` siguen el flujo `develop → qa → staging → main` ni el naming `hu-<n>-dev` de [git-conventions.md](../00-governance/git-conventions.md). Homologar ramas protegidas y plantillas de PR. | Medio | P2 | Todos los `*-db` | 🔴 |
| TD-008 | **Madurez desigual de aplicaciones**: varios servicios siguen como skeleton/diseño parcial mientras otros ya poseen API/worker. Completar casos de uso y gates por servicio. | Alto | P1 | Varios servicios | 🟡 |
| TD-009 | **CHECK de estados aún no parametrizados de forma uniforme**: [ADR-004](../05-architecture/decisions/records/ADR-004-status-parametrization-and-audit-standard.md) define estados parametrizables + auditoría estándar, pero no todos los servicios lo aplican todavía. Cerrar la brecha transversal. | Medio | P2 | `document`, `monitoring`, otros | 🔴 |
| TD-010 | **Validación del broker y contratos asíncronos pendiente por ambiente**: RabbitMQ es requerido por Audit, Monitoring y Notification; completar configuración, DLQ, retry y pruebas de contrato en CI. | Medio | P1 | `audit`, `monitoring`, `notification`, productores de eventos | 🟡 |

| TD-011 | **Completar separación contractual Monitoring → Notification**: retirar del `shared-contracts` canónico cualquier operación `/sent-notifications` o evento `monitoring.notification.sent`; mantener `notification.yaml` y `notification.notification.sent` como ownership de Notification. | Alto | P0 | `shared-contracts`, `monitoring`, `notification` | 🔴 |
| TD-012 | **Bandeja de notificaciones sin contrato de colección**: UX requiere `GET /notifications`, pero el OpenAPI vigente solo define creación y consulta por ID. Diseñar contrato paginado/cursor bajo Notification antes de implementar la campana global. | Medio | P1 | `notification`, `shell-host`, `shared-contracts` | 🔴 |

## Trabajo estructural pendiente (capa de aplicación)

Desglose de TD-008 en fases relativas. No se comprometen fechas hasta cerrar los bloqueantes P0.

> **Progreso: 2/10 microservicios con capa de aplicación completa en `develop`** — `iam-service` (12/12 HU Done·Dev) y `notification-service` (8/8 HU Done·Dev). El broker RabbitMQ ya está provisionado y validado E2E en `develop` (ver R-005 resuelto en [risks.md](./risks.md)). Los 8 servicios restantes siguen sólo en capa `-db`. Estado vivo por HU en el board (GitHub Project #3).

| Fase | Alcance | Depende de | Estado |
|------|---------|------------|--------|
| Fase A — Fundaciones de datos | Cerrar TD-001…TD-006; base de datos reproducible y segura por servicio. | — | 🟡 En curso |
| Fase B — Núcleo de identidad | Construir `iam-service` (API): login, JWT RS256, RBAC por feature+scope (ver [authentication.md](../07-api/authentication.md)). | Fase A | ✅ Done·Dev — en promoción a qa/staging/main |
| Fase C — Catálogos y dominio base | `reference-data-service`, `academic-management-service`, `training-environment-service`, `actors-service`. | Fase B | ⬜ Pendiente (sólo capa `-db`) |
| Fase D — Motor de horarios | `scheduling-service` + read models de [ADR-002](../05-architecture/decisions/records/ADR-002-scheduling-read-models.md). | Fase C | ⬜ Pendiente (sólo capa `-db`) |
| Fase E — Transversales | `document-service`, `audit-service`, `monitoring-service` y `notification-service`; validar separación por eventos según ADR-005. | Fase B–D | 🟡 Parcial — `notification-service` ✅ Done·Dev; `document`/`audit`/`monitoring` sólo capa `-db` |

## Criterio de ingreso al backlog

Un ítem entra aquí cuando: (1) está identificado con servicio y evidencia concreta, (2) tiene impacto y prioridad asignados, y (3) no es una tarea de contenido puramente documental (esas van a las ramas `docs/*`). Al resolverse se marca 🟢 y se mueve al histórico en el cierre de iteración.
