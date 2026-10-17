# Observabilidad — <PROJECT_KEY>

> **PLANTILLA** — Copiar como `observability.md` y completar.
> Eliminar esta línea antes de hacer commit.

> Estado: 🔴 Pendiente | Última actualización: YYYY-MM-DD
> Autor: Por definir | Equipo: DevOps / Arquitectura

## Stack de observabilidad

| Pilar | Herramienta | Destino | Estado |
|-------|-------------|---------|--------|
| Logs | [ELK / Loki / CloudWatch] | [URL dashboard] | Pendiente |
| Métricas | [Prometheus / Datadog / CloudWatch] | [URL dashboard] | Pendiente |
| Trazas | [Jaeger / Zipkin / X-Ray / Datadog APM] | [URL dashboard] | Pendiente |
| Alertas | [PagerDuty / Alertmanager / Opsgenie] | [canal de notificaciones] | Pendiente |

## Logs

### Formato estructurado obligatorio

```json
{
  "timestamp": "2026-01-15T10:30:00.000Z",
  "level": "INFO | WARN | ERROR",
  "service": "<proyecto>-<contexto>-api",
  "requestId": "uuid-v4",
  "userId": "uuid-v4 (si aplica)",
  "message": "descripción del evento",
  "context": {}
}
```

### Campos obligatorios por nivel

| Level | Cuándo usar | Campos adicionales obligatorios |
|-------|-------------|--------------------------------|
| INFO | Eventos de negocio normales | requestId |
| WARN | Situaciones degradadas pero recuperables | requestId, motivo |
| ERROR | Errores que afectan al usuario | requestId, error.code, stack |

### Política de retención

| Ambiente | Retención |
|----------|-----------|
| Producción | [90 días] |
| QA | [30 días] |
| Dev | [7 días] |

## Métricas

### Métricas RED (por endpoint)

| Métrica | Descripción | Umbral de alerta |
|---------|-------------|-----------------|
| Rate | Requests por segundo | — (informativo) |
| Errors | % de respuestas 5xx | > 1% por 5 min |
| Duration p95 | Latencia percentil 95 | > [200ms] por 5 min |

### Métricas USE (por recurso)

| Recurso | Utilización | Saturación | Errores |
|---------|-------------|------------|---------|
| CPU | > 80% por 10 min | Cola de procesos | — |
| Memoria | > 85% por 5 min | OOM events | — |
| Conexiones BD | > 80% del pool | Timeouts | Connection errors |

## Trazas distribuidas

| Configuración | Valor |
|--------------|-------|
| Sampling | 100% en errores; [10%] en requests normales |
| Propagación de headers | W3C TraceContext (`traceparent`, `tracestate`) |
| Atributos obligatorios | `service.name`, `http.method`, `http.route`, `user.id` |

## Alertas

| ID | Alerta | Condición | Severidad | Canal | Runbook |
|----|--------|-----------|-----------|-------|---------|
| ALT-001 | Alta tasa de errores | Error rate > 1% por 5 min | P1 | [Slack #alerts] | [runbook.md#alta-tasa-errores] |
| ALT-002 | Latencia alta | p95 > [200ms] por 10 min | P2 | [Slack #alerts] | [runbook.md#latencia-alta] |
| ALT-003 | Servicio caído | Healthcheck fallando | P0 | [PagerDuty] | [runbook.md] |
| ALT-004 | BD sin espacio | Disco > 85% | P1 | [Slack #alerts] | [runbook.md] |

## Healthchecks

| Endpoint | Qué verifica | Respuesta esperada |
|----------|--------------|--------------------|
| `GET /health` | Disponibilidad básica del proceso | `200 { "status": "ok" }` |
| `GET /health/ready` | Disponibilidad + dependencias (BD, cache) | `200 { "status": "ready" }` |
| `GET /health/live` | Proceso vivo (sin dependencias) | `200 { "status": "alive" }` |

## Dashboard principal

| Sección | Paneles |
|---------|---------|
| Resumen ejecutivo | Error rate, latencia p95, uptime |
| Por endpoint | Rate, errors, duration (RED) |
| Infraestructura | CPU, memoria, disco, conexiones (USE) |
| Trazas lentas | Top 10 requests más lentos últimas 24h |

> URL dashboard: [agregar cuando esté configurado]

## Referencias

- [SLA/SLO/SLI](./sla-slo-sli.md)
- [Runbook](./runbook.md)
- [NFR](../04-requirements/non-functional.md)
- [Architecture](../05-architecture/architecture.md)
