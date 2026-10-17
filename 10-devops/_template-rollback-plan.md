# Plan de rollback — <PROJECT_KEY>

> **PLANTILLA** — Copiar como `rollback-plan.md` y completar.
> Eliminar esta línea antes de hacer commit.

> Estado: 🔴 Pendiente | Última actualización: YYYY-MM-DD
> Autor: Por definir | Equipo: DevOps

## Criterios de activación del rollback

| Condición | Acción | Quién decide |
|-----------|--------|--------------|
| Error rate > 5% por más de 5 min | Rollback inmediato | DevOps + Tech Lead |
| Migración de BD fallida | Rollback inmediato | DBA |
| SLO violado por más de 15 min | Rollback + postmortem | Tech Lead |
| P0 detectado en producción | Rollback inmediato | Tech Lead |

## Procedimiento por componente

### Servicio / API

```bash
# Ver historial de despliegues
kubectl rollout history deployment/<nombre-servicio> -n <namespace>

# Rollback a la versión anterior
kubectl rollout undo deployment/<nombre-servicio> -n <namespace>

# Verificar
kubectl rollout status deployment/<nombre-servicio> -n <namespace>
```

### Base de datos

| Paso | Acción | Responsable |
|------|--------|-------------|
| 1 | Detener tráfico hacia el servicio afectado | DevOps |
| 2 | Restaurar backup del punto anterior a la migración | DBA |
| 3 | Verificar integridad de datos | DBA + QA |
| 4 | Reactivar tráfico | DevOps |

### Frontend

```bash
# Reactivar versión anterior en CDN/servidor
# [Comandos específicos según plataforma de despliegue]
```

## Verificación post-rollback

- [ ] Healthcheck de todos los servicios OK
- [ ] Error rate < 1% por 10 minutos consecutivos
- [ ] Smoke tests pasan en producción
- [ ] Stakeholders notificados

## Comunicación

| Evento | Canal | Responsable |
|--------|-------|-------------|
| Decisión de rollback | [Slack #incidents] | Tech Lead |
| Rollback en progreso | [Slack #incidents] | DevOps |
| Rollback completado | [Slack #incidents + stakeholders] | Tech Lead |

## Post-rollback

- Registrar incidente en [13-operations/incident-postmortem.md](../13-operations/_template-incident-postmortem.md)
- Identificar causa raíz antes de reintentar el despliegue

## Referencias

- [Deployment Plan](./deployment-plan.md)
- [Runbook](../13-operations/runbook.md)
- [Incident Postmortem](../13-operations/_template-incident-postmortem.md)
