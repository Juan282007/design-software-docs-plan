# Observabilidad

> Estado: 🟡 En progreso | Última actualización: 2026-08-01
> Autor: Jesús Ariel González Bonilla (PM + Arquitecto) | Equipo: Operaciones

Cómo se observa el sistema **Horarios SENA** a través de los tres pilares —logs, métricas y trazas— más healthchecks y alertas. El alcance depende del estado de construcción: **hoy la única capa desplegable es la de datos** (PostgreSQL 16 en Docker), por lo que la observabilidad efectiva se concentra en la base de datos y en la ejecución de migraciones. Los pilares de aplicación se activan cuando existan los componentes Go (API / worker / workflow). El sistema, además, incluye dos servicios propios que son **fuentes de observabilidad del dominio**: `monitoring` (KPIs y alertas de negocio) y `audit` (event store append-only).

## Fuentes de observabilidad

| Fuente | Naturaleza | Estado hoy |
|--------|------------|------------|
| PostgreSQL 16 (logs, `pg_stat_*`, `pg_isready`) | Infra de datos | ✅ Disponible hoy |
| Liquibase (`databasechangelog`, salida de `update`/`status`) | Trazabilidad de migraciones | ✅ Disponible hoy |
| `audit-service` (`audit_record`) | Registro inmutable de eventos de negocio | 🟡 Modelado; activo con la capa de aplicación |
| `monitoring-service` (KPIs, alertas) | Analítica y alertas de dominio | 🟡 Modelado; activo con la capa de aplicación |
| Logs / métricas / trazas de servicios Go | Aplicación | ⏳ Cuando exista la capa de aplicación |

> **Distinción importante:** `monitoring` y `audit` son observabilidad **del negocio** (KPIs pedagógicos, historial de eventos), no reemplazan la observabilidad **técnica** (latencia, errores 5xx, saturación) de la plataforma. Ambas coexisten.

## Pilar 1 — Logs

### Hoy (capa de datos)

- **Logs de PostgreSQL**: errores, `log_min_duration_statement` para consultas lentas, conexiones y checkpoints.
- **Salida de Liquibase**: cada `update`/`rollback`/`status` deja traza de qué changesets se aplicaron; la tabla `databasechangelog` (aislada por módulo) es el registro auditable de migraciones.

### Futuro (capa de aplicación)

Formato de log **estructurado en JSON** con campos mínimos por evento: `timestamp`, `level` (INFO/WARN/ERROR), `service` (ej. `scheduling-api`), `requestId`, `userId` si aplica, `message` y `context`. Detalle y política de retención por ambiente: ver [_template-observability.md](./_template-observability.md). La herramienta de agregación es un **punto abierto** (opciones a evaluar).

## Pilar 2 — Métricas

### Hoy (base de datos)

Métricas útiles disponibles vía PostgreSQL sin capa de aplicación:

- Conexiones activas vs. límite del pool.
- Consultas lentas y locks (`pg_stat_activity`, `pg_locks`).
- Tamaño de BD y uso de disco (relevante para alertas de espacio).
- Éxito/fallo de la última ejecución de migraciones por módulo.

### Futuro (aplicación)

Métricas **RED** por endpoint (Rate, Errors, Duration p95) y **USE** por recurso (Utilización, Saturación, Errores de CPU, memoria y pool de conexiones). Además, `monitoring-service` expone KPIs de dominio (asistencia, avance curricular, riesgo de deserción, avance de etapa productiva) que se calculan sobre read models (ADR-002) y **no** son métricas de plataforma.

## Pilar 3 — Trazas distribuidas

- **Hoy:** no aplica (no hay llamadas entre servicios en ejecución).
- **Futuro:** al existir comunicación por broker (ADR-001) y llamadas síncronas, se instrumentarán trazas con propagación de contexto (W3C TraceContext) y correlación por `event_id`/`requestId`. Como el bus es orientado a eventos, la correlación asíncrona (evento → consumidores) es tan importante como la traza síncrona. Herramienta APM: **punto abierto**.

## Healthchecks

### Base de datos (hoy)

```bash
# Disponibilidad de PostgreSQL en el contenedor
docker compose --env-file .env.develop exec postgres pg_isready -U <user> -d <db>
# Respuesta esperada: "accepting connections"

# Estado de migraciones de un módulo
docker compose --env-file .env.develop --profile tooling run --rm liquibase-<modulo> status --verbose
```

`pg_isready` es la sonda mínima de vida de la BD y es la base del healthcheck del contenedor en Docker.

### Aplicación (futuro)

Cada servicio Go expondrá `GET /health` (proceso), `GET /health/live` (vivo, sin dependencias) y `GET /health/ready` (listo, verifica BD y broker). Ver contrato en [_template-observability.md](./_template-observability.md).

## Alertas

Marco de alertas (los umbrales son de referencia hasta fijar SLOs reales):

| Alerta | Condición (referencia) | Severidad | Aplica hoy |
|--------|------------------------|-----------|------------|
| BD sin espacio | Disco > 85% | P1 | ✅ |
| PostgreSQL no responde | `pg_isready` falla | P0 | ✅ |
| Migración fallida | `update` de un módulo termina en error | P1 | ✅ |
| Consultas lentas sostenidas | queries > umbral por N min | P2 | ✅ |
| Alta tasa de errores 5xx | Error rate > umbral | P1 | ⏳ Con aplicación |
| Latencia alta | p95 > SLO | P2 | ⏳ Con aplicación |

El enrutamiento de alertas a un canal/guardia y la herramienta de alerting son un **punto abierto** (ver [incident-management.md](./incident-management.md)).

## Puntos abiertos

- Stack concreto de logs/métricas/trazas: opciones a evaluar, sin decisión tomada.
- Umbrales definitivos: dependen de los SLO por servicio (usar [_template-sla-slo-sli.md](./_template-sla-slo-sli.md)).
- Dashboards y su URL: por crear cuando exista el stack.

## Referencias

- [_template-observability.md](./_template-observability.md)
- [incident-management.md](./incident-management.md)
- [backup-and-recovery.md](./backup-and-recovery.md)
- [ADR-001 — Message broker](../05-architecture/decisions/records/ADR-001-message-broker.md)
- [ADR-002 — Read models de scheduling](../05-architecture/decisions/records/ADR-002-scheduling-read-models.md)
