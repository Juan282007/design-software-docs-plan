# Runbook — monitoring-service

> Estado: 🟡 En progreso | Última actualización: 2026-08-26

## Componentes operativos

| Componente | Verificación | Objetivo |
|------------|--------------|----------|
| `monitoring-api` | `GET /health` | Liveness HTTP. |
| `monitoring-api` | `GET /ready` | Readiness de BD y broker antes de recibir tráfico. |
| `alert-worker` | proceso + lag AMQP + métricas | Procesar eventos/KPIs sin backlog sostenido. |

No existe un `notification-worker` dentro de este servicio. Incidentes SMTP, `send_status`, DLQ de entrega y Outbox de notificación se atienden con el [runbook de notification-service](../10-notification-service/runbook.md).

## Configuración

La implementación actual es Python/FastAPI y está en evolución. Variables definitivas deben consolidarse en infraestructura/golden template; como mínimo se requieren las conexiones del servicio a:

- `monitoring_db`;
- broker AMQP para el worker cuando se conecte;
- configuración de JWT/JWKS para API;
- OpenTelemetry Collector.

**No configurar credenciales SMTP en Monitoring.**

Variables operativas esperadas:

| Variable | Uso |
| --- | --- |
| `DATABASE_URL` | Conexión a `monitoring_db`. |
| `AMQP_URL` | Conexión RabbitMQ/AMQP para worker, relay y readiness. |
| `JWKS_URL` | JWKS de IAM para validación JWT. |
| `API_HOST` / `API_PORT` | Bind HTTP local o contenedor. |
| `REQUEST_TIMEOUT_SECONDS` | Presupuesto de timeout para operaciones salientes. |
| `READINESS_TIMEOUT_SECONDS` | Timeout de chequeos `/ready`. |
| `OTEL_ENABLED` | Habilita instrumentación OpenTelemetry. |
| `OTEL_SERVICE_NAME` | Nombre de servicio en trazas y métricas. |
| `OTEL_EXPORTER_OTLP_ENDPOINT` | Endpoint OTLP HTTP del Collector. |
| `DEPLOYMENT_ENVIRONMENT` | Atributo OTel del ambiente. |
| `LOG_LEVEL` | Nivel de logs JSON. |
| `ALERT_WORKER_MODE` | `consumer` o `relay`. |

## Arranque local

```powershell
python -m pip install -e ".[dev]"
$env:DATABASE_URL="postgresql+psycopg://monitoring:monitoring@localhost:5432/monitoring_db"
$env:AMQP_URL="amqp://guest:guest@localhost:5672/"
python -m uvicorn apps.api.main:app --host 0.0.0.0 --port 8000
```

Worker:

```powershell
$env:ALERT_WORKER_MODE="consumer"
python -m apps.alert_worker.main
```

Relay Outbox:

```powershell
$env:ALERT_WORKER_MODE="relay"
python -m apps.alert_worker.main
```

## Migraciones

Aplicar los changelogs de `design-software-monitoring-db` antes de levantar API/worker. Validar que existan:

```sql
SELECT to_regclass('monitoring.kpi_tracking');
SELECT to_regclass('monitoring.generated_alert');
SELECT to_regclass('monitoring.outbox');
```

## Verificación de readiness

```powershell
Invoke-RestMethod http://localhost:8000/api/v1/health
Invoke-RestMethod http://localhost:8000/api/v1/ready
```

`/ready` debe responder `ready` solo si BD y broker están disponibles.

## P0 — `monitoring_db` no responde

Impacto: no se registran/consultan KPIs ni alertas.

1. validar conectividad PostgreSQL;
2. revisar pool, locks y capacidad;
3. evitar consumir/acknowledge eventos si no puede persistirse el resultado;
4. recuperar BD y verificar backlog del worker.

## P1 — Lag del `alert-worker`

Impacto: KPIs/alertas se calculan tarde. Notification puede estar sano pero no recibirá nuevos `monitoring.alert.triggered` hasta que Monitoring procese el backlog.

1. revisar estado del proceso;
2. revisar errores de deserialización/contrato;
3. revisar latencia de BD;
4. revisar profundidad de cola y DLQ de Monitoring;
5. escalar consumidores solo si la semántica de orden/idempotencia lo permite.

## P1 — Mensajes en DLQ de Monitoring

1. identificar routing key `monitoring.alert-worker.dlq`;
2. capturar `event_id`, `event_type`, `correlation_id` y motivo del fallo;
3. corregir causa raíz (contrato, catálogo faltante, BD, regla KPI);
4. reprocesar manualmente solo cuando la idempotencia por `source_event_id` esté verificada;
5. no mover mensajes a colas de Notification.

## Replay de Outbox

Consultar pendientes:

```sql
SELECT id, event_id, event_type, created_at
FROM monitoring.outbox
WHERE published_at IS NULL
ORDER BY created_at;
```

Ejecutar relay:

```powershell
$env:ALERT_WORKER_MODE="relay"
python -m apps.alert_worker.main
```

El relay marca `published_at` únicamente tras ack del broker.

## P1 — Alertas duplicadas

```sql
SELECT source_event_id, count(*)
FROM monitoring.generated_alert
WHERE source_event_id IS NOT NULL
GROUP BY source_event_id
HAVING count(*) > 1;
```

Corregir deduplicación antes de reprocesar mensajes. No “compensar” eliminando notificaciones: ese dato pertenece a otro servicio.

## Resolver una alerta

Usar el endpoint/caso de uso de `monitoring-api` correspondiente. La resolución cambia el estado de `generated_alert`; no modifica registros de Notification.

## Recalcular KPI

El procedimiento de recalculo debe publicar solo los eventos de monitoreo que correspondan y mantener idempotencia. Una recalculación no debe invocar directamente a Notification.

## Incidente de notificación reportado desde una alerta

Si la alerta existe en Monitoring pero el usuario no recibió aviso:

1. comprobar que `monitoring.alert.triggered` fue publicado con `event_id` válido;
2. capturar `event_id`/`correlation_id`;
3. transferir investigación al runbook de `notification-service`;
4. **no** consultar ni modificar tablas de `notification_db` desde Monitoring.

## Escalamiento

| Nivel | Responsable | Caso |
|-------|-------------|------|
| L1 | Equipo Monitoring | health, BD, lag, errores de API/worker |
| L2 | Líder técnico Monitoring | reglas KPI, duplicidad, contratos de evento |
| L3 | Arquitectura/Infra | broker, base de datos, cambios de frontera |
| Transferencia | Equipo Notification | entrega EMAIL/IN_APP, retries, DLQ, templates |
