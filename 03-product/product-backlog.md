# Product Backlog — SENA Gestión de Horarios

> Estado: 🟡 En progreso | Última actualización: 2026-08-28
> Autor: Jesús Ariel González Bonilla (PM + Arquitecto) | Equipo: Producto

> Fuente: [vision.md](./vision.md) · [roadmap.md](./roadmap.md) · [04-requirements/user-stories.md](../04-requirements/user-stories.md)

## Propósito

Este documento es el **mapa estable de épicas (EPC-NN)** del producto: agrupa las capacidades por fase del [roadmap.md](./roadmap.md) y prioridad **MoSCoW** (Must / Should / Could / Won't-now). **No es el board.**

> **El estado vivo del avance se lleva por historia de usuario en el board — GitHub Project #3, que es la única fuente de verdad (SSOT) del progreso.** Allí viven las HU con su identificador real (`HU-IAM-001..012`, `HU-NOTIF-001..008`, etc.), su estado (Todo / In Progress / Done·Dev / …) y su ambiente. Este `.md` deliberadamente **no** replica el desglose feature-por-HU para no divergir del board; sólo mantiene el marco de épicas y su anclaje a fase, servicio y requisitos ([functional.md](../04-requirements/functional.md), RF-*).

## Épicas

Columna **Estado** = estado agregado de la épica a nivel de fase (resumen; el detalle por HU está en el board).

| Épica | Nombre | Fase | Servicio(s) | Prioridad | Estado |
|-------|--------|------|-------------|-----------|--------|
| **EPC-01** | Identidad y control de acceso | 0 | `iam-service` | Must | ✅ Done·Dev (12/12 HU) |
| **EPC-02** | Plataforma de eventos y auditoría | 0 | broker, `audit-service` | Must | 🟡 Broker ✅ dev; `audit` sólo `-db` |
| **EPC-03** | Datos maestros institucionales | 1 | `reference-data-service` | Must | ⬜ Sólo capa `-db` |
| **EPC-04** | Gestión académica (programas y fichas) | 1 | `academic-management-service` | Must | ⬜ Sólo capa `-db` |
| **EPC-05** | Gestión de ambientes y disponibilidad | 1 | `training-environment-service` | Must | ⬜ Sólo capa `-db` |
| **EPC-06** | Gestión de actores (instructores/aprendices) | 1 | `actors-service` | Must | ⬜ Sólo capa `-db` |
| **EPC-07** | Motor de horarios: construcción y conflictos | 2 | `scheduling-service` | Must | ⬜ Sólo capa `-db` |
| **EPC-08** | Publicación y consulta de horarios | 2 | `scheduling-service` | Must | ⬜ Sólo capa `-db` |
| **EPC-09** | Seguimiento, KPIs y alertas | 3 | `monitoring-service` | Should | ⬜ Sólo capa `-db` |
| **EPC-09b** | Notificaciones transversales (entrega email/IN_APP) | 3 (transversal) | `notification-service` | Should | 🟡 En progreso (0/8 HU cerradas — board real; código mergeado por el PM, sin revisión ni cierre de HU) |
| **EPC-10** | Documentos (PDFs) | 3 | `document-service` | Should | ⬜ Sólo capa `-db` |
| **EPC-11** | Planes de mejoramiento | 3 | `monitoring`/`actors` | Should | ⬜ Sólo capa `-db` |
| **EPC-12** | Extensiones (SOFIA Plus, IA, offline, 2FA) | 4 | varios | Won't-now | ⬜ Post-MVP |

> **EPC-09b (Notificaciones):** la capacidad de entrega de notificaciones se **extrajo de EPC-09/`monitoring-service` por [ADR-005](../05-architecture/decisions/records/ADR-005-microservices-topology-and-transversal-capabilities.md)** y hoy es un servicio transversal propio (`notification-service`) con API + worker + Outbox + retry/DLQ, ya construido en `develop`. `monitoring` sólo publica `monitoring.alert.triggered`; la entrega es responsabilidad de Notification. Ver [15-project-control/notification-service-refactor.md](../15-project-control/notification-service-refactor.md).

---

## Detalle por historia de usuario → board

El desglose feature-por-HU (identificadores, criterios de aceptación, estado y ambiente) vive en el **board (GitHub Project #3)**, que es la SSOT viva del avance. Las HU allí usan los identificadores reales por servicio — p. ej. `HU-IAM-001..012` (identidad, épica cerrada, 12/12 Done) y `HU-NOTIF-001..008` (notificaciones, **en progreso** — mergeado por el PM, 0/8 HU cerradas realmente en el board). Para la definición estable de requisitos ver [04-requirements/user-stories.md](../04-requirements/user-stories.md) y [functional.md](../04-requirements/functional.md); para el estado, consultar siempre el board directamente (no asumir por lo dicho aquí).

> Nota: versiones previas de este documento incluían un desglose HU propio con numeración `HU-NN` que **no coincidía** con los identificadores del board; se retiró para evitar divergencia. El board manda.

---

## Resumen por release

| Release | Épicas | Resultado |
|---------|--------|-----------|
| **R0 — Fundaciones** | EPC-01, EPC-02 | Identidad, eventos y auditoría operativos |
| **R1 — Datos maestros** | EPC-03..06 | Insumos del motor cargados; disponibilidad consultable |
| **R2 — MVP horarios** (hito) | EPC-07, EPC-08 | Construir → validar → publicar → consultar horarios |
| **R3 — Valor pedagógico** | EPC-09, EPC-09b, EPC-10, EPC-11 | Seguimiento, KPIs, alertas, notificaciones, documentos y planes |
| **R4 — Extensiones** | EPC-12 | Post-MVP |

## Mapa de dependencias (resumen)

```
EPC-01/02 ──▶ EPC-03/04/05/06 ──▶ EPC-07 ──▶ EPC-08 ──▶ EPC-09/10/11 ──▶ EPC-12
(identidad)   (datos maestros)     (motor)   (publicar)  (seguimiento)    (extensiones)
```

- EPC-07 depende de disponibilidad de instructores/ambientes (EPC-05/06) y de fichas en `EXECUTION` (EPC-04).
- EPC-09/10/11 dependen del evento `scheduling.schedule.published` (EPC-08).

## Puntos abiertos

- Priorización interna de Fase 3 (documentos vs. planes) por confirmar con Producto.
- **PA-03**: si SOFIA Plus (EPC-12) se adelanta al MVP, EPC-04 debe incorporar el ACL correspondiente (ver [domain-map.md](../02-domain/domain-map.md)).

## Referencias

- [vision.md](./vision.md) · [roadmap.md](./roadmap.md)
- [04-requirements/functional.md](../04-requirements/functional.md) · [04-requirements/user-stories.md](../04-requirements/user-stories.md) · [04-requirements/traceability-matrix.md](../04-requirements/traceability-matrix.md)
