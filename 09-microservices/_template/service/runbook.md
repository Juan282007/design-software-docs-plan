# Runbook — [service-name]

> **PLANTILLA** — Completar antes de primer despliegue a QA.
> Estado: 🔴 Pendiente | Última actualización: YYYY-MM-DD

## Healthcheck

| Endpoint | Respuesta esperada | SLO |
|----------|--------------------|-----|
| `GET /health` | `200 { "status": "ok" }` | < 200 ms |
| `GET /health/ready` | `200` cuando BD conectada | < 500 ms |

## Alertas críticas

| Alerta | Condición | Severidad | Acción inmediata |
|--------|-----------|-----------|------------------|
| BD no responde | Timeout > 5 s | P0 | Verificar conexión y reiniciar |
| Error rate > 5 % | En ventana de 5 min | P1 | Revisar logs y escalar |

## Reinicio del servicio

```bash
# Pendiente — completar según plataforma de despliegue (Docker Compose / K8s / Railway)
```

## Revisión de logs

```bash
# Pendiente — completar según stack de observabilidad (Datadog / Loki / CloudWatch)
```

## Escalamiento

| Condición | Paso siguiente | Contacto |
|-----------|---------------|----------|
| No se resuelve en 15 min | Escalar a tech lead | @[handle] |
| Incidente de datos | Activar [rollback-plan.md](../../../10-devops/_template-rollback-plan.md) | @[handle] |

## Documentos relacionados

- Postmortem template: [_template-incident-postmortem.md](../../../13-operations/_template-incident-postmortem.md)
- SLA/SLO: [_template-sla-slo-sli.md](../../../13-operations/_template-sla-slo-sli.md)
