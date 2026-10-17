# Refactor arquitectónico — Monitoring y Notification

> Estado: 🟢 En cierre (casi completo) | Última actualización: 2026-08-28
> Owner: Arquitectura / Backend / Datos / DevOps
> Decisión: [ADR-005](../05-architecture/decisions/records/ADR-005-microservices-topology-and-transversal-capabilities.md)

## Objetivo

Separar definitivamente la gestión de notificaciones de `monitoring-service` y consolidarla en `notification-service` como microservicio transversal, desplegable y operable de forma independiente.

## Estado objetivo

```text
monitoring-service
  ├─ tracking
  ├─ KPIs
  ├─ generated_alert
  └─ publica monitoring.alert.triggered
          │
          ▼
       RabbitMQ
          │
          ▼
notification-service
  ├─ notification-api
  ├─ notification-worker
  ├─ sent_notification
  ├─ notification_template
  ├─ outbox
  └─ publica notification.notification.sent
```

## Principios de desacoplamiento

1. Monitoring no entrega email, IN_APP ni otros canales.
2. Monitoring no persiste `sent_notification` ni templates.
3. Notification no accede a tablas, clases o workers internos de Monitoring.
4. La integración entre ambos se hace exclusivamente por contratos públicos/eventos.
5. El ciclo de despliegue, escalado, rollback y observabilidad es independiente por servicio.
6. Reintentos, SMTP, DLQ y fallos de entrega son responsabilidad de Notification.
7. Una indisponibilidad de Notification no debe impedir que Monitoring persista su alerta y continúe su procesamiento; el evento debe poder reintentarse/recuperarse mediante mensajería confiable.

## Cambios documentales aplicados

### Monitoring

- retirado `components/notification-worker/`;
- retirado `sent_notification` del modelo de datos;
- retirados endpoints de notificación del contrato de Monitoring;
- retirado `monitoring.notification.sent` como evento vigente;
- `monitoring.alert.triggered` queda como contrato canónico de salida para alertas notificables.

### Notification

Creada la documentación en:

```text
09-microservices/services/10-notification-service/
├── README.md
├── data-model.md
├── events.md
├── decisions.md
├── runbook.md
└── components/
    ├── notification-api/
    │   ├── README.md
    │   └── contract.md
    └── notification-worker/
        ├── README.md
        └── contract.md
```

Además se incorpora `07-api/contracts/openapi/notification.yaml` como espejo documental del contrato canónico.

## Dependencias externas al repositorio Docs

La documentación solo se considera completamente materializada cuando los repositorios relacionados cumplen el mismo ownership.

| Repositorio | Cambio requerido | Criterio |
|-------------|------------------|----------|
| `design-software-shared-contracts` | Monitoring sin recursos/eventos de entrega; Notification con OpenAPI/eventos propios | contrato único sin ownership duplicado |
| `design-software-monitoring-service` | sin SMTP, delivery adapters ni persistencia Notification | publica alertas; no entrega |
| `design-software-monitoring-db` | sin tablas de Notification | solo datos de M9 |
| `design-software-notification-service` | API + worker + idempotencia + Outbox + canales | desplegable independiente |
| `design-software-notification-db` | `sent_notification`, templates, Outbox | ownership exclusivo |
| `design-software-iam-*` | features `NOTIF_*` / módulo transversal | autorización sin `MON_NOTIFICATION_*` |
| `design-software-docker-infra` | contenedores, secretos, RabbitMQ, SMTP de desarrollo, OTel | despliegue independiente |
| shell/frontend | bandeja/campana bajo Notification | no consulta Monitoring |

## Gap conocido: bandeja global

El OpenAPI vigente de Notification soporta `POST /notifications` y `GET /notifications/{id}`. La UX de campana/bandeja requiere formalizar un endpoint de colección, recomendado:

```http
GET /notifications?recipient_id=<uuid>&cursor=<opaque>&limit=20
```

Este endpoint **no debe implementarse bajo Monitoring**. Debe aprobarse primero en `design-software-shared-contracts` y luego implementarse en Notification.

## Criterios de cierre

`notification-service` está **construido y operando en `develop`** (8/8 HU Done·Dev): API + worker + BD + Outbox + retry/DLQ, consumiendo del broker RabbitMQ (validado E2E) y publicando `notification.notification.sent`. La mayoría de los criterios ya se cumplen; quedan abiertos sólo los ítems de saneamiento contractual (**TD-011**) y la bandeja de colección (**TD-012**).

- [x] No existe `notification-worker` bajo Monitoring.
- [ ] Monitoring no expone `/sent-notifications` — **pendiente en el `shared-contracts` canónico (TD-011)**; ya retirado en la documentación de Monitoring.
- [x] Monitoring no persiste `sent_notification`.
- [x] `monitoring.alert.triggered` es consumible por Notification mediante contrato versionado.
- [x] Notification posee API, worker, BD, retry/DLQ y Outbox propios. *(Done·Dev)*
- [x] `notification.notification.sent` es publicado por Notification. *(validado E2E en dev)*
- [x] IAM usa features `NOTIF_*`, no `MON_NOTIFICATION_*`.
- [x] C4/dependency map muestran ambos servicios por separado. *(ver [dependencies.md](./dependencies.md))*
- [x] Runbooks y observabilidad asignan fallos de entrega a Notification. *(OTel/Tempo operativo)*
- [ ] Contract tests bloquean una reintroducción del ownership de Notification en Monitoring — **pendiente (TD-011)**.

### Pendientes de cierre

- **TD-011** — completar la separación contractual en `shared-contracts`: retirar toda operación `/sent-notifications` y el evento `monitoring.notification.sent` del contrato canónico; dejar `notification.yaml` y `notification.notification.sent` como ownership exclusivo de Notification, y añadir contract tests que impidan la regresión.
- **TD-012** — formalizar la **bandeja global** `GET /notifications` (paginación por cursor) bajo Notification, aprobada primero en `shared-contracts` (ver sección *Gap conocido* arriba).
