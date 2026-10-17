# ADR-008: Observabilidad — OpenTelemetry + Grafana (LGTM)

**Estado:** ACCEPTED (2026-08-14)
**Fecha:** 2026-08-14
**Autores:** Jesús Ariel González Bonilla (PM + Arquitecto)
**Equipos involucrados:** Arquitectura, Backend, DevOps

---

## Contexto

Sistema polyglot (Go/Java/Python/Node — [ADR-006](./ADR-006-polyglot-stack-strategy.md)). El reto es
**correlacionar señales entre lenguajes** y cumplir la propagación de `trace_id`/`correlation_id` que ya
exige [communication-patterns.md](../../../09-microservices/communication-patterns.md). Sin un estándar
único, cada stack observa distinto y no se ve el flujo E2E.

## Decisión

### 1. Estándar de instrumentación = OpenTelemetry (vendor-neutral, los 4 stacks)

Todos los servicios instrumentan con **OTel** y exportan **OTLP** (gRPC/HTTP) a un pipeline común.

### 2. Pipeline (por ambiente, en `docker-infra`)

```
servicios (Go/Java/Python/Node)  --OTLP-->  OpenTelemetry Collector  -->  Grafana LGTM
                                                                          Tempo  (traces)
                                                                          Prometheus (métricas)
                                                                          Loki   (logs)
                                                                          Grafana (un solo panel)
```
Traefik (edge, [ADR-007](./ADR-007-edge-gateway-and-port-convention.md)) **inicia la traza** (OTel nativo).

### 3. Tres señales

| Señal | Contenido | Backend |
|-------|-----------|---------|
| **Traces** | traza distribuida E2E (`trace_id`) | Tempo |
| **Metrics** | RED (rate/errors/duration) + custom (profundidad de cola, KPIs) | Prometheus |
| **Logs** | JSON estructurado **con `trace_id` embebido** | Loki |

Correlación en Grafana: de una traza saltás a sus logs y métricas (un solo panel).

### 4. Propagación de contexto (síncrono y asíncrono — la pieza clave)

- **HTTP:** header W3C `traceparent`/`tracestate` (OTel lo propaga automáticamente).
- **Eventos (AMQP):** al publicar se **inyecta `traceparent` en los headers del mensaje**; el consumidor lo
  **extrae** → la traza **cruza el salto async** (ej. `scheduling.schedule.published` → consumo en
  `notification` = **misma traza**). El `correlation_id` del envelope viaja al lado.

### 5. La instrumentación vive en la shared platform lib por stack ([ADR-006](./ADR-006-polyglot-stack-strategy.md))

Cada lib inicializa OTel (`service.name`, `deployment.environment`, endpoint del collector) → el servicio
solo la importa y queda observable de forma uniforme.

| Stack | Instrumentación |
|-------|-----------------|
| **Java/Spring** | OTel Java agent (auto, **zero-code**) |
| **Go** | OTel SDK + `otelhttp` + `otelsql` + wrapper AMQP |
| **Python/FastAPI** | `opentelemetry-instrument` (auto: FastAPI/httpx/SQLAlchemy) |
| **Node/TS** | OTel Node auto-instrumentation |

### 6. Por ambiente

Un **Collector + LGTM por ambiente** (develop/qa/staging/main), puertos host por offset (Grafana 3000+off,
Tempo 3200+off, etc.). Atributo `deployment.environment` en todas las señales. (Alternativa: stack
compartido con el atributo como discriminador — se decide al operar.)

## Consecuencias

### Positivas
- Un solo panel cross-lenguaje; el flujo E2E (sync + async) se ve como un sistema.
- Todo servicio queda observable con solo importar su platform lib.
- OTLP es neutral → cambiar de backend (Jaeger↔Tempo) no toca los servicios.

### Negativas / Trade-offs
- Nueva infra a operar (Collector + Tempo/Prometheus/Loki/Grafana) por ambiente.
- Overhead de instrumentación (mínimo con sampling adecuado).

### Riesgos
- Volumen de trazas/logs. Mitigación: sampling (head/tail) en el Collector; retención por ambiente.

## Alternativas consideradas

| Alternativa | Por qué se descartó |
|-------------|---------------------|
| Sin estándar (cada stack lo suyo) | No hay correlación cross-lenguaje; se pierde el E2E. |
| Solo Jaeger (traces) | Válido y más simple, pero deja métricas/logs fuera; LGTM da un panel único. |
| APM propietario | Vendor lock-in; OTel es neutral y formativo. |

## Referencias

- [ADR-006](./ADR-006-polyglot-stack-strategy.md) · [ADR-007](./ADR-007-edge-gateway-and-port-convention.md)
- [communication-patterns.md](../../../09-microservices/communication-patterns.md) — propagación `trace_id`/`correlation_id`
