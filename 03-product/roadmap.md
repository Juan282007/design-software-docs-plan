# Roadmap de Producto — SENA Gestión de Horarios

> Estado: 🟡 En progreso | Última actualización: 2026-08-28
> Autor: Jesús Ariel González Bonilla (PM + Arquitecto) | Equipo: Producto

> Fuente: [vision.md](./vision.md) · [product-backlog.md](./product-backlog.md) · [01-context/scope.md](../01-context/scope.md)

## Cómo leer este roadmap

El proyecto es **greenfield** y de carácter formativo (programa ADSO). Por eso el roadmap se expresa en **fases relativas** (Fase 0…4), no en fechas de calendario: los compromisos temporales se acuerdan por sprint en el tracker y no se fijan aquí. Cada fase entrega un incremento funcional verificable.

**Estado actual de la construcción:** la plataforma tiene **10 microservicios backend** (tras la extracción de `notification-service` desde `monitoring-service`, [ADR-005](../05-architecture/decisions/records/ADR-005-microservices-topology-and-transversal-capabilities.md)). De ellos, **2 tienen su capa de aplicación completa en `develop`** — `iam-service` (12/12 HU Done·Dev) y `notification-service` (8/8 HU Done·Dev) — y los **8 restantes existen aún sólo como su repositorio de capa de datos** (`*-db`, Liquibase/PostgreSQL), a la espera de que se construyan API, worker y/o workflow. Además, el **broker RabbitMQ ya está provisionado y validado E2E en `develop`** (`iam` publica en `iam-events`; `notification` consume con DLQ real) y la observabilidad OTel/Tempo está operativa. Ver [09-microservices/service-catalog.md](../09-microservices/service-catalog.md). El **estado vivo del avance por HU se lleva en el board (GitHub Project #3)**; este roadmap es el marco estable de fases.

## Vista general

```
Fase 0 ──▶ Fase 1 ──────▶ Fase 2 ─────────▶ Fase 3 ──────────▶ Fase 4
Fundaciones  Datos maestros  MOTOR DE HORARIOS  Seguimiento y      Escala y
de plataforma + ambientes    (CORE) + consulta  documentos (CORE)  extensiones
```

---

## Fase 0 — Fundaciones de plataforma

**Meta:** que exista identidad, y que cualquier servicio pueda autenticar, autorizar, emitir eventos y ser auditado.

| Entregable | Servicio | Referencia | Estado |
|------------|----------|------------|--------|
| Login JWT + refresh, RBAC por feature + scope | `iam-service` | RN-IAM-01..05 | ✅ Done·Dev (12/12 HU) |
| Broker de eventos operativo (RabbitMQ) + DLQ | Transversal | ADR-001 | ✅ Provisionado y validado E2E en `develop` |
| Auditoría append-only consumiendo todos los topics | `audit-service` | RN-CROSS-01 | ⬜ Sólo capa `-db` |
| Estándar transversal de estados + auditoría (status/soft delete) | Todos | ADR-004 | 🟡 En curso |

**Criterio de salida:** un usuario puede autenticarse, recibir un JWT con sus features, y toda acción queda registrada en `audit-service`.

> **Avance de Fase 0:** identidad y broker ya operan en `develop`. `iam-service` publica en `iam-events` y hay un consumidor real con DLQ (hoy `notification-service`). Falta la capa de aplicación de `audit-service` para cerrar la fase por completo.

---

## Fase 1 — Datos maestros y ambientes

**Meta:** tener cargados los insumos que el motor de horarios necesitará: jerarquía institucional, programas/competencias/fichas, y ambientes con su disponibilidad.

| Entregable | Servicio |
|------------|----------|
| Jerarquía institucional (centro → municipio → …) y catálogos del sistema | `reference-data-service` |
| Programas, competencias, RAPs; fichas y su ciclo de estados | `academic-management-service` |
| Ambientes, tipos, reglas de disponibilidad, mantenimientos | `training-environment-service` |
| Instructores (competencias, disponibilidad) y aprendices | `actors-service` |
| Endpoints de disponibilidad `GET /environments/available`, `GET /instructors/available` (< 300 ms) | environment / actors |

**Criterio de salida:** existe al menos una ficha en estado `EXECUTION` con instructores y ambientes disponibles consultables — precondición del motor de horarios.

---

## Fase 2 — Motor de horarios y consulta (CORE)

**Meta:** el corazón del producto. Construir, validar, publicar y consultar horarios.

| Entregable | Servicio | Reglas |
|------------|----------|--------|
| Crear borrador de horario para una ficha en `EXECUTION` | `scheduling-service` | RN-ACAD-04, RN-SCH-02 |
| Agregar/editar/cancelar sesiones de clase (ficha+instructor+ambiente+franja) | `scheduling-service` | RN-SCH-05/06 |
| **Detección de conflictos** (5 tipos) con validación inmediata y completa | `scheduling-service` | RN-SCH-03/04 |
| **Publicación inmutable** con versionado (archivar anterior) | `scheduling-service` | RN-SCH-01 |
| **Read models** de instructores y ambientes por eventos | `scheduling-service` | ADR-002 |
| Consulta de horario por instructor y por aprendiz | `scheduling-service` (lectura) | CF-04/05 |

**Criterio de salida (hito de MVP):** un coordinador construye y publica un horario sin conflictos en < 1 h, y un instructor/aprendiz consulta su horario vigente. Aquí se cumplen los criterios funcionales CF-01…CF-06 de [problem-framing.md](./problem-framing.md).

---

## Fase 3 — Seguimiento, documentos y planes (CORE)

**Meta:** cerrar el valor pedagógico y documental sobre los horarios ya publicados.

| Entregable | Servicio |
|------------|----------|
| Inicio de seguimiento al publicarse un horario (`scheduling.schedule.published`) | `monitoring-service` |
| KPIs por ficha (asistencia, avance) y cálculo de riesgo de deserción | `monitoring-service` |
| Alertas por umbral (INFO→CRITICAL) y su resolución trazable | `monitoring-service` |
| Entrega de notificaciones (email/IN_APP) al consumir `monitoring.alert.triggered` | `notification-service` (transversal) |
| Planes de mejoramiento (instructor/aprendiz) | `monitoring-service` / `actors-service` |
| Generación de PDFs (horario, constancias) en object storage S3 | `document-service` |

**Criterio de salida:** una ficha publicada genera seguimiento, KPIs y alertas; el coordinador puede descargar el horario en PDF y gestionar planes de mejoramiento.

> **Nota — `notification-service` (transversal, ya construido en `develop`):** se extrajo de `monitoring-service` por [ADR-005](../05-architecture/decisions/records/ADR-005-microservices-topology-and-transversal-capabilities.md) y ya tiene su capa de aplicación completa (API + worker + Outbox + retry/DLQ, 8/8 HU Done·Dev). Es un servicio transversal: aunque su consumidor de negocio principal (alertas de `monitoring`) pertenece a la Fase 3, su plataforma de entrega quedó operativa desde ahora y consume eventos vía el broker de la Fase 0.

---

## Fase 4 — Escala y extensiones (post-MVP)

**Meta:** ampliar cobertura y robustez una vez validado el flujo core. Estos ítems están **fuera del MVP** (ver [scope.md](../01-context/scope.md)).

- Sugerencia automática de asignación (scheduling asistido) y resolución de conflictos.
- Integración con **SOFIA Plus** (reemplaza el ingreso manual de fichas/programas).
- Etapa productiva completa (visitas a empresa, convenios) — RN-ACT-04/05.
- Notificaciones push, exportación a calendarios externos.
- 2FA, soporte offline (PWA), app móvil nativa.
- Crecimiento multi-centro a escala nacional (la arquitectura ya lo soporta — NFR-SCALE-04).

---

## Dependencias entre fases

```
Fase 0 (identidad, eventos, auditoría)
   └─▶ Fase 1 (datos maestros + ambientes + actores)
          └─▶ Fase 2 (MOTOR de horarios) ◀── depende de disponibilidad de F1
                 └─▶ Fase 3 (seguimiento/documentos) ◀── depende de horarios publicados
                        └─▶ Fase 4 (extensiones)
```

- La Fase 2 **no puede empezar** sin fichas en `EXECUTION` y disponibilidad de instructores/ambientes (Fase 1).
- La Fase 3 se dispara con el evento `scheduling.schedule.published` (Fase 2).

## Riesgos que condicionan el roadmap

Heredados de [problem-framing.md](./problem-framing.md):

| Riesgo | Impacto en el roadmap |
|--------|-----------------------|
| Integración con SOFIA Plus exigida antes de lo previsto | Adelantaría trabajo de Fase 4 al MVP; **punto abierto PA-03** |
| Reglas de disponibilidad de ambientes más complejas de lo modelado | Puede extender la Fase 1 |
| Resistencia del coordinador al cambio | Mitigar con validación de UX temprana durante Fase 2 |

## Puntos abiertos

- **PA-03** (roadmap): confirmar con el sponsor si SOFIA Plus entra al MVP o queda en Fase 4.
- Prioridad relativa dentro de Fase 3 entre *documentos* y *planes de mejoramiento* (por definir con Producto).

## Referencias

- [vision.md](./vision.md) · [product-backlog.md](./product-backlog.md)
- [01-context/scope.md](../01-context/scope.md)
- [09-microservices/service-catalog.md](../09-microservices/service-catalog.md)
- [05-architecture/decisions/records/ADR-001-message-broker.md](../05-architecture/decisions/records/ADR-001-message-broker.md) · [ADR-002](../05-architecture/decisions/records/ADR-002-scheduling-read-models.md) · [ADR-004](../05-architecture/decisions/records/ADR-004-status-parametrization-and-audit-standard.md)
