# UML

> Estado: 🟡 En progreso | Última actualización: 2026-08-28
> Autor: Jesús Ariel González Bonilla (PM + Arquitecto) | Equipo: Arquitectura

Repositorio de diagramas UML y arquitectura visual del proyecto **Horarios SENA**. Todo diagrama debe tener **fuente editable** (texto versionable) y **exportación revisable** (SVG).

> ⚠️ El comportamiento **ya no es solo previsto**: dos servicios tienen capa de aplicación construida y probada en `develop` — `iam-service` (Java) y `notification-service` (Go) — así que sus flujos de comportamiento **son diagramables hoy con base en lo construido**. Ya se dibujaron los tres diagramas de secuencia de flujos vivos (login+JWT, outbox→relay→iam-events, consumo/DLQ de notification). Los diagramas de comportamiento de los servicios que aún viven solo en capa de datos siguen **previstos**. Los diagramas ER por servicio pueden derivarse ya del esquema real de cada repo `*-db` (el de `iam` es directamente derivable).

## Convenciones

- **Fuentes** en `diagrams/source/` con extensión `.wsd` o `.puml` (PlantUML). Diagramas simples pueden expresarse como bloques ```mermaid``` embebidos en Markdown (como en [dependencies.md](../15-project-control/dependencies.md)).
- **Exportaciones** en `diagrams/exports/` en formato `.svg` (preferido) o `.png`, generadas desde la fuente. Los SVG compartidos se guardan en `assets/` a nivel raíz cuando se referencian desde varios documentos.
- **Nombre de archivo**: `<dominio>-<tipo>.<ext>` (ej. `scheduling-sequence.wsd`, `iam-component.wsd`).
- Todo diagrama debe registrarse en [diagram-index.md](./diagram-index.md).
- La fuente es la **única verdad**: nunca editar un SVG a mano; regenerarlo desde la fuente.

## Tipos de diagrama

| Tipo | Archivo fuente | Herramienta |
|------|---------------|-------------|
| Contexto (C4 nivel 1) | `*-context.wsd` | PlantUML |
| Componentes (C4 nivel 3) | `*-component.wsd` | PlantUML |
| Casos de uso | `*-use-case.wsd` | PlantUML |
| Clases | `*-class.wsd` | PlantUML |
| Secuencia | `*-sequence.wsd` | PlantUML |
| Actividad | `*-activity.wsd` | PlantUML |
| Estado | `*-state.wsd` | PlantUML |
| Despliegue | `*-deployment.wsd` | PlantUML |
| Entidad-Relación (por servicio) | `*-er.wsd` | PlantUML / Mermaid |

## Índice de diagramas

Estado: 🔴 No creado · 🟡 En curso / derivable ahora · 🟢 Listo.

| Diagrama | Tipo | Alcance | Fuente prevista | Estado |
|----------|------|---------|-----------------|--------|
| Contexto del sistema | Contexto | Horarios SENA y actores externos (SENA, IdP futuro) | `system-context.wsd` | 🔴 |
| Panorama de microservicios | Componentes | Los 9 servicios y sus dependencias (deriva de [dependency-map.md](../09-microservices/dependency-map.md)) | `platform-component.wsd` | 🔴 |
| Autenticación (login + JWT) | Secuencia | Flujo de [authentication.md](../07-api/authentication.md) en `iam-service` (construido) | [`diagrams/source/login-jwt.md`](./diagrams/source/login-jwt.md) | 🟢 |
| Creación de horario | Secuencia | scheduling ↔ actors / training-environment / academic-management | `scheduling-create-sequence.wsd` | 🔴 |
| Propagación de eventos (outbox → relay) | Secuencia | `iam-service`: `OutboxRelayScheduler` publica de `identity.outbox` a `iam-events` (ver [ADR-001](../05-architecture/decisions/records/ADR-001-message-broker.md)) | [`diagrams/source/outbox-relay-events.md`](./diagrams/source/outbox-relay-events.md) | 🟢 |
| Consumo de notificación (retry/DLQ) | Secuencia | `notification-service`: consume evento AMQP → idempotencia → entrega → SENT / retry → DLQ | [`diagrams/source/notification-consume.md`](./diagrams/source/notification-consume.md) | 🟢 |
| Estados parametrizables | Estado | Ciclo de vida de estados según [ADR-004](../05-architecture/decisions/records/ADR-004-status-parametrization-and-audit-standard.md) | `status-lifecycle-state.wsd` | 🔴 |
| ER — iam-service | Entidad-Relación | Esquema `identity`/`rbac`/`session` (derivable ya del esquema real construido) | `iam-er.wsd` | 🟡 derivable ahora |
| ER — reference-data-service | Entidad-Relación | Catálogos y estructura institucional | `reference-data-er.wsd` | 🔴 |
| ER — academic-management-service | Entidad-Relación | Programas y fichas | `academic-management-er.wsd` | 🔴 |
| ER — training-environment-service | Entidad-Relación | Ambientes, inventario, disponibilidad | `training-environment-er.wsd` | 🔴 |
| ER — scheduling-service | Entidad-Relación | Horarios, sesiones, read models | `scheduling-er.wsd` | 🔴 |
| ER — actors-service | Entidad-Relación | Instructores, aprendices, empresas | `actors-er.wsd` | 🔴 |
| ER — document-service | Entidad-Relación | Documentos y `storage_key` (ver [ADR-003](../05-architecture/decisions/records/ADR-003-object-storage.md)) | `document-er.wsd` | 🔴 |
| ER — monitoring-service | Entidad-Relación | KPIs, alertas, seguimiento | `monitoring-er.wsd` | 🔴 |
| ER — audit-service | Entidad-Relación | Log append-only | `audit-er.wsd` | 🔴 |

> Prioridad de dibujo: **ya construidos** los tres diagramas de secuencia de flujos vivos de `iam` y `notification` (login+JWT, outbox→relay, consumo/DLQ). Sigue pendiente: el **ER de `iam`** (derivable ya del esquema real), el resto de **ER por servicio**, el **contexto/componentes**, y los diagramas de secuencia de los servicios que aún no tienen capa de aplicación.

## Archivos

| Archivo | Descripción | Estado |
|---------|-------------|--------|
| [diagram-index.md](./diagram-index.md) | Índice registrado de fuentes y exportaciones | 🟡 |
| [diagrams/source/](./diagrams/source/) | Fuentes editables de diagramas | 🟡 (3 secuencias en Mermaid) |
| [diagrams/exports/](./diagrams/exports/) | Exportaciones SVG o PNG | 🔴 |
