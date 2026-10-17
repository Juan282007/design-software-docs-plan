<!-- RESUMEN-EJECUTIVO
agente: Jesús Ariel González Bonilla (PM + Arquitecto)
capacidad: contrato de API (OpenAPI 3.1)
fase: diseño (api-first)
estado: accepted
dependencias_entrada: 09-microservices/services/04-training-environment-service/data-model.md, 07-api/guidelines.md, 07-api/contracts/openapi/_shared.yaml
consumidores_siguientes: backend training-environment-service, scheduling-service (disponibilidad), frontend, pruebas de contrato
tldr: CRUD de ambientes de formación, catálogo de tipos, reglas de disponibilidad, mantenimiento (append-only) y reservas con anti-solapamiento (409 RESERVATION_OVERLAP), más reporte de utilización. La fuente de verdad es training-environment.yaml.
decisiones_clave: OpenAPI publicable en 07-api/contracts/openapi/training-environment.yaml; "recursos" del dominio se resuelven en environment-types + availability-rules + maintenance + reservations (data-model.md no define una entidad `resource`; `inventory_item` mencionado en README.md está sin modelar y queda fuera hasta que exista su tabla); maintenance y environment_type son append-only/inmutables (sin PUT/PATCH/DELETE) porque sus tablas no tienen columnas de actualización o borrado; cancelar reserva = status→CANCELLED (no hay deleted_at)
halts_registrados: ninguno
-->

# Contrato — training-environment-api

> Estado: 🟢 Aceptado | Última actualización: 2026-08-06
> Autor: Jesús Ariel González Bonilla (PM + Arquitecto) | Equipo: Arquitectura

> **Fuente de verdad (normativa):** el spec OpenAPI 3.1 en
> [`07-api/contracts/openapi/training-environment.yaml`](../../../../../07-api/contracts/openapi/training-environment.yaml).
> Este documento es la **narrativa** que lo explica; ante cualquier diferencia, **manda el
> `training-environment.yaml`**. Convenciones transversales en
> [07-api/guidelines.md](../../../../../07-api/guidelines.md).

> **Diseño previsto — no implementado.** Contrato REST/JSON, agnóstico de lenguaje.

## Autenticación
`Authorization: Bearer <JWT>` de `iam-service`; feature + scope.

## Base URL
`/api/v1`

## Endpoints (entidades reales)
| Método | Path | Descripción | Feature |
|--------|------|-------------|---------|
| `GET` | `/environment-types` | Catálogo de tipos de ambiente | `ENVIRONMENT_VIEW` |
| `POST` | `/environment-types` | Crear tipo de ambiente | `ENVIRONMENT_TYPE_MANAGE` |
| `GET` | `/environment-types/{id}` | Detalle de tipo | `ENVIRONMENT_VIEW` |
| `GET` | `/training-environments` | Listar ambientes (filtros: tipo, centro, activo, ventana de disponibilidad) | `ENVIRONMENT_VIEW` |
| `POST` | `/training-environments` | Crear ambiente | `ENVIRONMENT_MANAGE` |
| `GET` | `/training-environments/{id}` | Detalle | `ENVIRONMENT_VIEW` |
| `PUT`/`PATCH` | `/training-environments/{id}` | Actualizar ambiente | `ENVIRONMENT_MANAGE` |
| `DELETE` | `/training-environments/{id}` | Baja lógica (`is_active=false`) | `ENVIRONMENT_MANAGE` |
| `GET`/`POST` | `/training-environments/{id}/availability-rules` | Reglas de disponibilidad recurrente | `ENVIRONMENT_VIEW` / `ENVIRONMENT_AVAILABILITY_MANAGE` |
| `GET`/`PUT`/`DELETE` | `/training-environments/{id}/availability-rules/{rule_id}` | Detalle/editar/eliminar regla | `ENVIRONMENT_VIEW` / `ENVIRONMENT_AVAILABILITY_MANAGE` |
| `GET`/`POST` | `/training-environments/{id}/maintenance` | Períodos de mantenimiento (filtro `from`/`to`) | `ENVIRONMENT_VIEW` / `ENVIRONMENT_MAINTENANCE_MANAGE` |
| `GET` | `/training-environments/{id}/maintenance/{maintenance_id}` | Detalle (append-only) | `ENVIRONMENT_VIEW` |
| `GET`/`POST` | `/reservations` | Listar/crear reservas (rechaza solapamiento) | `ENVIRONMENT_RESERVATION_VIEW` / `ENVIRONMENT_RESERVATION_MANAGE` |
| `GET`/`PATCH`/`DELETE` | `/reservations/{id}` | Detalle/actualizar/cancelar reserva | `ENVIRONMENT_RESERVATION_VIEW` / `ENVIRONMENT_RESERVATION_MANAGE` |
| `GET` | `/reports/environment-utilization` | Reporte de utilización (dominio propio) | `ENVIRONMENT_REPORT_VIEW` |

## Regla de negocio clave
`POST /reservations` (y `PATCH /reservations/{id}` al reprogramar) puede fallar con `409 Conflict` (`code: RESERVATION_OVERLAP`) porque la BD impone no-solapamiento por ambiente mediante una constraint `EXCLUDE` GiST sobre reservas no canceladas. El cliente debe manejar ese caso. Cancelar una reserva no la borra: transiciona `status` a `CANCELLED` (`DELETE /reservations/{id}` → `204`).

## Formato de error estándar
Envelope uniforme de la plataforma (guidelines §7), definido en `_shared.yaml#/components/schemas/Error`:
```json
{
  "error": {
    "code": "RESERVATION_OVERLAP",
    "message": "El ambiente ya está reservado en ese rango.",
    "details": [{ "field": "reservation_date", "issue": "OVERLAPS_EXISTING_RESERVATION" }],
    "trace_id": "b3f1c2a4-..."
  }
}
```

## Reportes (dominio propio)

Cada servicio expone y responde por sus propios reportes (guidelines §11); no hay hub central.

| Reporte | Endpoint | Usuarios | Frecuencia | Formato | Fuente |
|---------|----------|----------|------------|---------|--------|
| Utilización de ambientes | `GET /reports/environment-utilization` | Coordinadores, `scheduling-service`, `monitoring-service` | On-demand | JSON (paginado, offset) | Tablas `environment`, `availability_rule`, `maintenance`, `reservation` |

Calcula, por ambiente y rango `[from, to)`, horas disponibles (según `availability_rule`), horas reservadas (`reservation` no cancelada) y horas en mantenimiento (`maintenance`), y deriva `utilization_percentage`. Solo lectura; no muta estado.

## Eventos publicados
`environment.environment.created`, `environment.availability.changed`, `environment.maintenance.started`, `environment.reservation.created` (ver [events.md](../../events.md)).
