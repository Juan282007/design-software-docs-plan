# SLA / SLO / SLI — <PROJECT_KEY>

> **PLANTILLA** — Copiar como `sla-slo-sli.md` y completar.
> Eliminar esta línea antes de hacer commit.

> Estado: 🔴 Pendiente | Última actualización: YYYY-MM-DD
> Autor: Por definir | Equipo: DevOps / Arquitectura

## Definiciones

| Concepto | Definición |
|----------|------------|
| SLA | Acuerdo de nivel de servicio con el cliente/usuario — consecuencias si se incumple |
| SLO | Objetivo interno de nivel de servicio — meta técnica que el equipo se compromete a mantener |
| SLI | Indicador de nivel de servicio — métrica real medida que evalúa el SLO |

## SLOs por servicio

### Servicio: [nombre-del-servicio]

| SLI | SLO | Ventana de medición | SLA asociado |
|-----|-----|---------------------|--------------|
| Disponibilidad (% uptime) | ≥ 99.9% | Rolling 30 días | Si cae < 99.5%: [compensación] |
| Latencia p95 | < [200ms] | Rolling 7 días | — |
| Latencia p99 | < [500ms] | Rolling 7 días | — |
| Tasa de errores | < 0.5% | Rolling 24 horas | — |
| Tiempo de recuperación (MTTR) | < [30 min] | Por incidente | — |

## Error budget

| Servicio | SLO disponibilidad | Error budget mensual | Consumido (mes actual) |
|----------|-------------------|---------------------|----------------------|
| [nombre] | 99.9% | 43.2 min/mes | [X min] |

> Si el error budget se agota, se congela el desarrollo de nuevas features hasta que se recupere.

## Políticas

| Condición | Acción |
|-----------|--------|
| Error budget < 50% restante | Alerta al Tech Lead y congelamiento de deploys no críticos |
| Error budget agotado | Feature freeze + postmortem obligatorio |
| SLO incumplido 2 meses consecutivos | Revisión de arquitectura |

## Métricas y herramientas

| Métrica | Herramienta | Dashboard |
|---------|-------------|-----------|
| Uptime | [UptimeRobot / Datadog / Grafana] | [URL] |
| Latencia | [Prometheus / Datadog] | [URL] |
| Error rate | [ELK / Datadog / Grafana] | [URL] |

## Referencias

- [Runbook](./runbook.md)
- [NFR](../04-requirements/non-functional.md)
- [Architecture](../05-architecture/architecture.md)
