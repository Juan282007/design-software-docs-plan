# Control de Proyecto

> Estado: 🟡 En progreso | Última actualización: 2026-08-28
> Autor: Jesús Ariel González Bonilla (PM + Arquitecto) | Equipo: Proyecto

## Contenido

Agrupa backlog técnico, riesgos, dependencias y preguntas abiertas del proyecto **Horarios SENA**. Es la vista de gestión: qué está pendiente, qué puede salir mal y de qué depende cada pieza. El detalle técnico vive en las carpetas de arquitectura (`05-architecture`), datos (`06-data`) y microservicios (`09-microservices`).

## Estado del proyecto

La plataforma tiene **10 microservicios backend** con madurez heterogénea: **2 con capa de aplicación completa en `develop`** (`iam-service` 12/12 HU Done·Dev, `notification-service` 8/8 HU Done·Dev) y **8 aún sólo en capa `-db`**. El broker **RabbitMQ está provisionado y validado E2E en `develop`**. El backlog registra brechas de datos, aplicación, contratos e infraestructura por servicio; la separación de `notification-service` se controla en [notification-service-refactor.md](./notification-service-refactor.md). El **estado vivo del avance por HU se lleva en el board (GitHub Project #3 = SSOT)**.

## Archivos

| Archivo | Descripción | Estado |
|---------|-------------|--------|
| [technical-backlog.md](./technical-backlog.md) | Pendientes técnicos y deuda documentada | 🟡 |
| [risks.md](./risks.md) | Riesgos, impacto, probabilidad y mitigación | 🟡 |
| [dependencies.md](./dependencies.md) | Dependencias internas y externas | 🟡 |
| [open-questions.md](./open-questions.md) | Preguntas pendientes de decisión o aclaración (PA-01…PA-04) | 🟡 |

## Plantillas

| Plantilla | Descripción |
|-----------|-------------|
| [_template-risk-register.md](./_template-risk-register.md) | Registro de riesgos: activos, bloqueantes, resueltos y aceptados |
| [_template-sprint-plan.md](./_template-sprint-plan.md) | Plan de sprint: HUs comprometidas, trazabilidad QA y Definition of Done |
