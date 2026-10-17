# Registro de Riesgos

> Estado: 🟡 En progreso | Última actualización: 2026-08-28
> Autor: Jesús Ariel González Bonilla (PM + Arquitecto) | Equipo: Proyecto

Riesgos del proyecto **Horarios SENA**. Consolida riesgos técnicos, de alcance, de equipo, externos y de seguridad. Sigue el formato de [_template-risk-register.md](./_template-risk-register.md).

## Cómo se calcula la exposición

**Exposición = Probabilidad × Impacto** (escala Baja=1 / Media=2 / Alta=3).

- Baja: producto < 3
- Media: producto 3–6
- Alta: producto > 6

## Riesgos activos

| ID | Categoría | Riesgo | Prob. | Impacto | Exposición | Mitigación | Owner | Estado |
|----|-----------|--------|-------|---------|------------|------------|-------|--------|
| R-001 | Seguridad | Secretos `.env.*` versionados en los repos `-db` pueden filtrar credenciales de base de datos. | Alta | Alto | Alta | Purgar del historial, rotar credenciales, mover a Secret Manager y bloquear vía `.gitignore`. Ver TD-001 en [technical-backlog.md](./technical-backlog.md). | PM/Arquitecto | Abierto |
| R-002 | Técnico | Migraciones Liquibase no reproducibles (doble anidamiento + rutas rotas) impiden levantar el esquema desde cero. | Alta | Alto | Alta | Aplanar estructura, validar `liquibase update` sobre base vacía por repo en CI. Ver TD-002/TD-003. | Equipo Datos | Abierto |
| R-003 | Alcance | La capa de aplicación (API/worker/workflow) está construida en **2/10 servicios** (`iam-service`, `notification-service`) en `develop`; los 8 restantes siguen sólo en capa `-db`. Subestimar el esfuerzo pendiente aún puede tensar el cronograma. | Media | Alto | Media | Plan por fases A–E (ver backlog), comprometer alcance por iteración con DoR/DoD, no comprometer fechas hasta cerrar P0. Los dos servicios ya construidos validan el patrón y bajan la incertidumbre. | PM/Arquitecto | Abierto |
| R-004 | Técnico | `scheduling-service` supera el límite de 2 dependencias síncronas (depende de actors, training-environment y academic-management). | Media | Alto | Media | Implementar read models según [ADR-002](../05-architecture/decisions/records/ADR-002-scheduling-read-models.md); aprobar la ADR antes de construir el motor. | Arquitectura | Abierto |
| R-006 | Seguridad | Integridad de datos débil en `document-service` y `monitoring-service` (faltan CHECK e índices de FK) puede permitir estados inválidos y consultas costosas. | Media | Medio | Media | Reforzar esquema (TD-005/TD-006) y aplicar estados parametrizables de [ADR-004](../05-architecture/decisions/records/ADR-004-status-parametrization-and-audit-standard.md) antes de exponer API. | Equipo Datos | Abierto |
| R-007 | Equipo | Concentración de conocimiento: el ownership y conocimiento de los **10 microservicios backend** puede concentrarse en pocas personas; ausencia o rotación frena todo el proyecto (factor bus = 1). | Media | Alto | Media | Documentar decisiones (ADRs, runbooks), mantener este repo de docs como fuente de verdad, incorporar y formar equipo por dominio. | PM | Abierto |
| R-008 | Técnico | Convención de ramas inconsistente entre repos rompe la trazabilidad `develop → qa → staging → main` y la promoción de cambios. | Media | Medio | Media | Homologar ramas protegidas y naming de [git-conventions.md](../00-governance/git-conventions.md); plantillas de PR comunes. Ver TD-007. | PM/Arquitecto | Abierto |
| R-009 | Externo | Driver JDBC obsoleto frente a PostgreSQL 16 puede causar incompatibilidades en despliegue de migraciones. | Baja | Medio | Baja | Actualizar y fijar versión de driver soportada (TD-004); validar en CI. | Equipo Datos | Abierto |
| R-010 | Alcance | Contratos de API aún no implementados: el diseño podría divergir del esquema de datos real al construir la capa app. | Media | Medio | Media | Mantener el contrato de implementación junto al servicio y validar contra el modelo de datos; publicar OpenAPI sólo al estabilizar (ver [07-api/README.md](../07-api/README.md)). | Arquitectura | Abierto |

## Bloqueantes activos

| ID | Bloqueante | Condición de desbloqueo | Responsable |
|----|-----------|-------------------------|-------------|
| B-001 | No se puede construir ningún servicio de aplicación con seguridad hasta sanear secretos. | Cerrar R-001 (rotación + Secret Manager). | PM/Arquitecto |
| B-002 | No se puede validar despliegue de esquema en CI mientras las migraciones no arranquen desde cero. | Cerrar R-002 (TD-002/TD-003). | Equipo Datos |
| B-003 | `scheduling-service` no puede diseñarse en detalle sin decisión de read models. | Aprobar ADR-002. | Arquitectura |

## Riesgos aceptados

| ID | Riesgo | Justificación |
|----|--------|---------------|
| RA-001 | En el MVP, `scheduling-service` propaga el JWT de usuario a servicios downstream en lugar de usar tokens M2M. | Simplicidad de MVP; el scope del usuario aplica en toda la cadena. Migración a token M2M prevista en V2 (ver [authentication.md](../07-api/authentication.md)). |

## Riesgos resueltos

| ID | Riesgo | Resolución | Estado |
|----|--------|------------|--------|
| R-005 | El broker de mensajes de [ADR-001](../05-architecture/decisions/records/ADR-001-message-broker.md) seguía en `PROPOSED`; sin él, auditoría y monitoreo asíncronos no avanzaban. | **RabbitMQ provisionado y validado E2E en `develop`**: `iam-service` publica en `iam-events` y `notification-service` consume con DLQ real; observabilidad OTel/Tempo operativa. Queda como pendiente **de promoción** (no de riesgo) la configuración de broker/DLQ/contratos en `qa/staging/main` — se sigue en TD-010 de [technical-backlog.md](./technical-backlog.md). | ✅ Resuelto en dev |
