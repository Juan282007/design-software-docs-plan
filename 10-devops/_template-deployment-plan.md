# Plan de despliegue — <PROJECT_KEY>

> **PLANTILLA** — Copiar como `deployment-plan.md` y completar.
> Eliminar esta línea antes de hacer commit.

> Estado: 🔴 Pendiente | Última actualización: YYYY-MM-DD
> Autor: Por definir | Equipo: DevOps

## Metadata del despliegue

| Campo | Valor |
|-------|-------|
| Versión / Release | |
| Ambiente destino | dev / qa / producción |
| Fecha planificada | YYYY-MM-DD HH:MM |
| Responsable | |
| Ventana de despliegue | [duración estimada] |
| Requiere downtime | Sí / No |

## Pre-requisitos

- [ ] Gate de QA aprobado
- [ ] Gate de seguridad aprobado
- [ ] Backups verificados
- [ ] Variables de entorno configuradas en ambiente destino
- [ ] Plan de rollback preparado

## Componentes a desplegar

| Componente | Versión actual | Versión nueva | Cambios |
|------------|----------------|---------------|---------|
| `<servicio>-api` | v1.0.0 | v1.1.0 | |
| `<servicio>-db` | — | migration V005 | |

## Pasos de despliegue

| Orden | Paso | Comando / Acción | Responsable | Verificación |
|-------|------|-----------------|-------------|--------------|
| 1 | Backup de BD | | DBA | Backup confirmar |
| 2 | Aplicar migraciones | | DevOps | `db-status` OK |
| 3 | Desplegar servicio | | DevOps | Healthcheck OK |
| 4 | Smoke test | | QA | Tests pasan |
| 5 | Activar tráfico | | DevOps | Métricas estables |

## Verificación post-despliegue

- [ ] Healthcheck de todos los servicios respondiendo
- [ ] Métricas de error < 1% en los primeros 15 minutos
- [ ] Pruebas de smoke en ambiente destino
- [ ] Dashboard de observabilidad sin anomalías

## Plan de rollback

| Condición | Acción | Tiempo estimado |
|-----------|--------|-----------------|
| Error en migración de BD | Restaurar backup | [X min] |
| Error en despliegue de servicio | Revertir imagen anterior | [X min] |
| Smoke tests fallando | Rollback completo | [X min] |

> Procedimiento detallado en [rollback-plan.md](./rollback-plan.md).

## Comunicación

| Evento | Canal | Mensaje |
|--------|-------|---------|
| Inicio del despliegue | [Slack #deploys] | `[DEPLOY INICIO] v1.1.0 en producción` |
| Despliegue exitoso | [Slack #deploys] | `[DEPLOY OK] v1.1.0 estable` |
| Rollback | [Slack #incidents] | `[ROLLBACK] Motivo: [...]. ETA: [...]` |

## Referencias

- [Release Checklist](./release-checklist.md)
- [Rollback Plan](./rollback-plan.md)
- [Runbook](../13-operations/runbook.md)
