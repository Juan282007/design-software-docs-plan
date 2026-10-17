# monitoring-service

> Estado: 🟡 Revisión | Última actualización: 2026-10-07
> Autor: Kevin Andrey Culma (Arquitecto) | Equipo: Arquitectura.

| Campo | Valor |
|---|---|
| Módulo | M9 |
| App repo | `design-software-monitoring-service` |
| DB repo | `design-software-monitoring-db` |
| Stack | Python / FastAPI |
| Deployables | api, alert_worker |
| Estado observado | Implementado parcialmente con API FastAPI, caso de uso RecordKpi y worker de alertas pendiente. |

## Responsabilidad

KPIs, seguimiento de fichas, sesiones de seguimiento, alertas generadas y planes de mejora.

## Persistencia

- Esquemas: monitoring
- Tablas: monitoring.alert_type, monitoring.ficha_tracking, monitoring.generated_alert, monitoring.improvement_plan, monitoring.kpi_status, monitoring.kpi_tracking, monitoring.kpi_tracking_2026_08, monitoring.kpi_tracking_default, monitoring.kpi_type, monitoring.outbox, monitoring.risk_level, monitoring.sent_notification, monitoring.tracking_session

## API detectada

| Metodo | Ruta | Fuente |
|---|---|---|
| GET | `/health` | `.\design-software-monitoring-service\src\monitoring\adapter\inbound\http\router.py` |
| POST | `/kpis` | `.\design-software-monitoring-service\src\monitoring\adapter\inbound\http\router.py` |

## Configuración

- Sin variables detectadas automáticamente en archivos de configuración/código.

## Notas

Debe detectar/publicar alertas; la entrega pertenece a notification-service.
