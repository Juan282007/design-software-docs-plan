# notification-api

> Estado: 🟡 En progreso | Última actualización: 2026-08-26
> Stack: Go

## Tipo de componente

`-api` — API HTTP síncrona del `notification-service`.

## Responsabilidad

- recibir solicitudes manuales/síncronas de notificación;
- validar DTOs generados desde `shared-contracts`;
- consultar una notificación por ID;
- exponer liveness/readiness;
- delegar los casos de uso a `application`, sin lógica de infraestructura en dominio.

No calcula alertas ni accede a `monitoring_db`.

## Endpoints

| Método | Ruta | Descripción |
|--------|------|-------------|
| `POST` | `/notifications` | Solicita una notificación. |
| `GET` | `/notifications/{id}` | Consulta una notificación. |
| `GET` | `/health` | Liveness. |
| `GET` | `/ready` | Readiness (DB). |

## Arquitectura

```text
HTTP handler
   ↓
application/usecase
   ↓
ports
   ↓
persistence / notifier / template repository
```

## Contrato

- [contract.md](./contract.md)
- [`07-api/contracts/openapi/notification.yaml`](../../../../../07-api/contracts/openapi/notification.yaml)
