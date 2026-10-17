# Runbook — <PROJECT_KEY> — [Nombre del servicio]

> **PLANTILLA** — Copiar como `runbook-<servicio>.md` y completar.
> Eliminar esta línea antes de hacer commit.

> Estado: 🔴 Pendiente | Última actualización: YYYY-MM-DD
> Autor: Por definir | Equipo: DevOps / Operaciones

## Información rápida del servicio

| Campo | Valor |
|-------|-------|
| Repositorio | [url] |
| Ambiente producción | [url / namespace] |
| Dashboard principal | [url Grafana / Datadog] |
| Canal de alertas | [Slack #canal / PagerDuty] |
| Contacto de escalamiento | [nombre / handle] |
| RTO objetivo | [30 min] |
| RPO objetivo | [5 min] |

## Arquitectura operativa (solo lo necesario para operar)

```text
[Usuario] → [Load Balancer] → [API Service] → [PostgreSQL]
                                            → [Cache]
                                            → [API externa X]
```

**Dependencias críticas:**
- `[BD]`: si cae, el servicio no puede procesar [operación crítica]
- `[Cache]`: si cae, mayor latencia pero el servicio continúa
- `[API externa X]`: si cae, [flujo Y] falla con error 503

## Alertas y diagnóstico rápido

### Alta tasa de errores 5xx

**Síntoma:** Error rate > 2% durante 5 minutos.

```bash
# Ver logs recientes
kubectl logs -n <namespace> -l app=<service-name> --tail=100

# Verificar estado de pods
kubectl get pods -n <namespace> -l app=<service-name>
```

**Acciones:**
1. Si hay crash loops: ver `kubectl describe pod <pod> -n <namespace>`
2. Si es error de BD: ver sección 4.1
3. Si es error de dependencia externa: ver sección 4.3

### Latencia alta (p95 > SLO)

**Síntoma:** p95 > [200ms] durante 10 minutos.

```bash
# Verificar uso de recursos
kubectl top pods -n <namespace>
```

**Acciones:**
1. Si CPU saturada: escalar horizontalmente (ver sección 5.1)
2. Si query lento: ver sección 4.1 (BD)

## Procedimientos por componente

### 4.1 Base de datos

```bash
# Verificar salud
kubectl exec -n <namespace> <postgres-pod> -- pg_isready -U <user> -d <db>
```

### 4.2 Cache

```bash
kubectl exec -n <namespace> <redis-pod> -- redis-cli ping
# Respuesta esperada: PONG
```

## Operaciones de escala y mantenimiento

### Escalar horizontalmente

```bash
kubectl scale deployment/<nombre> -n <namespace> --replicas=<N>
kubectl get pods -n <namespace> -l app=<service-name>
```

### Rollback de emergencia

```bash
kubectl rollout undo deployment/<nombre> -n <namespace>
kubectl rollout status deployment/<nombre> -n <namespace>
```

## Comunicación durante incidente

| Evento | Canal | Mensaje tipo |
|--------|-------|-------------|
| Detección P0 | [Slack #incidents] | `[P0 INICIO] [servicio] degradado. Investigando.` |
| Cada 15 min | [Slack #incidents] | `[P0 UPDATE] Causa: [...]. Acción: [...]. ETA: [...]` |
| Resolución | [Slack #incidents] | `[P0 RESUELTO] Duración: [X min]. Causa raíz: [...]` |

## Checklist post-incidente

- [ ] Servicio estable con métricas en objetivo SLO
- [ ] Incidente registrado en [incident-postmortem.md](./_template-incident-postmortem.md)
- [ ] Stakeholders notificados de resolución
- [ ] Ticket de mejora creado

## Referencias

- [SLA/SLO/SLI](./sla-slo-sli.md)
- [Architecture](../05-architecture/architecture.md)
- [Rollback Plan](../10-devops/rollback-plan.md)
