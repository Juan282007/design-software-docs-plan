# Operaciones

> Estado: 🟡 En progreso | Última actualización: 2026-08-01
> Autor: Jesús Ariel González Bonilla (PM + Arquitecto) | Equipo: Operaciones

## Contexto

Describe cómo operar, monitorear, responder incidentes y recuperar el sistema **Horarios SENA**. El estado de construcción condiciona el alcance operativo: **hoy solo existe la capa de datos** (PostgreSQL 16 + Liquibase en Docker, un repo `*-db` por servicio). Las capas de aplicación (API / worker / workflow) aún no se han construido, por lo que gran parte de la operación actual gira alrededor de la **base de datos y sus migraciones**. Cada documento marca explícitamente qué aplica hoy y qué se activa cuando exista la capa de aplicación, además de los servicios transversales ya modelados: `audit` (event store append-only) y `monitoring` (KPIs y alertas).

## Contenido

Describe cómo operar, monitorear, responder incidentes y recuperar el sistema.

## Archivos

| Archivo | Descripción | Estado |
|---------|-------------|--------|
| [observability.md](./observability.md) | Métricas, logs, trazas, alertas y tableros | 🟡 |
| [incident-management.md](./incident-management.md) | Clasificación, respuesta y comunicación de incidentes | 🟡 |
| [backup-and-recovery.md](./backup-and-recovery.md) | Backups, restauración, RPO/RTO y pruebas de recuperación | 🟡 |

## Plantillas

| Plantilla | Descripción |
|-----------|-------------|
| [_template-runbook.md](./_template-runbook.md) | Runbook operativo: alertas, diagnóstico, procedimientos y escalamiento |
| [_template-observability.md](./_template-observability.md) | Observabilidad: logs, métricas RED/USE, trazas, alertas y healthchecks |
| [_template-sla-slo-sli.md](./_template-sla-slo-sli.md) | SLA/SLO/SLI: objetivos por servicio, error budget y políticas |
| [_template-incident-postmortem.md](./_template-incident-postmortem.md) | Postmortem: línea de tiempo, causa raíz, impacto y acciones correctivas |

## Puntos abiertos

- Selección del stack concreto de observabilidad (opciones a evaluar, no decididas): pendiente hasta desplegar la capa de aplicación.
- SLO/SLA por servicio con números comprometidos: por definir cuando exista tráfico real (usar [_template-sla-slo-sli.md](./_template-sla-slo-sli.md)).
- Roles de guardia (on-call) y canales reales de escalamiento: por completar.
