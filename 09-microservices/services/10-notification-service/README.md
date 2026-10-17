# notification-service

> Estado: 🟡 Revisión | Última actualización: 2026-10-07
> Autor: Kevin Andrey Culma (Arquitecto) | Equipo: Arquitectura.

| Campo | Valor |
|---|---|
| Módulo | Transversal |
| App repo | `design-software-notification-service` |
| DB repo | `design-software-notification-db` |
| Stack | Go |
| Deployables | notification-api, notification-worker |
| Estado observado | Implementado con API HTTP, worker AMQP, outbox, templates y SMTP/MailHog para dev. |

## Responsabilidad

Envío, trazabilidad e idempotencia de notificaciones EMAIL e IN_APP.

## Persistencia

- Esquemas: notification
- Tablas: notification.notification_template, notification.outbox, notification.sent_notification

## API detectada

| Metodo | Ruta | Fuente |
|---|---|---|
| GET | `/health` | `.\design-software-notification-service\internal\adapter\in\http\handler.go` |
| GET | `/health` | `.\design-software-notification-service\cmd\notification-worker\main.go` |
| POST | `/notifications` | `.\design-software-notification-service\internal\adapter\in\http\handler.go` |
| GET | `/notifications/{id}` | `.\design-software-notification-service\internal\adapter\in\http\handler.go` |
| GET | `/ready` | `.\design-software-notification-service\cmd\notification-worker\main.go` |
| GET | `/ready` | `.\design-software-notification-service\internal\adapter\in\http\handler.go` |

## Configuración

- `ID`
- `NOTIFICATION_AMQP_URL`
- `NOTIFICATION_DB_DSN`
- `NOTIFICATION_DEPLOYMENT_ENVIRONMENT`
- `NOTIFICATION_RECIPIENT_STUB_EMAIL`
- `NOTIFICATION_SMTP_ADDR`
- `NOTIFICATION_SMTP_FROM`
- `OTEL_EXPORTER_OTLP_ENDPOINT`
- `OTEL_EXPORTER_OTLP_INSECURE`
- `PORT`
- `WORKER_HEALTH_PORT`

## Notas

Propietario único de sent_notification y notification_template.
