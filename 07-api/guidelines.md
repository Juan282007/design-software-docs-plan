# Convenciones de Diseño de API REST

> Estado: 🟡 En progreso | Última actualización: 2026-08-06
> Autor: Jesús Ariel González Bonilla (PM + Arquitecto) | Equipo: Arquitectura

Convenciones REST que **regirán** las APIs de los servicios de **Horarios SENA**.

> ⚠️ **Las APIs aún no están implementadas.** Hoy sólo existe la capa de datos (repos `*-db`). Este documento es el estándar de diseño que deberá cumplir cada servicio al construir su capa de aplicación (ver fases en [technical-backlog.md](../15-project-control/technical-backlog.md)). Todo ejemplo es ilustrativo, no un endpoint existente.

## 1. Estilo y formato

- **REST sobre HTTP/JSON.** `Content-Type: application/json; charset=utf-8`.
- Cuerpos de request y response siempre en JSON; nada de form-encoding salvo cargas de archivo en `document-service` (multipart).
- Nombres de campos en `snake_case` (coherente con el modelo de datos y el JWT descrito en [authentication.md](./authentication.md)).
- Fechas y horas en **ISO 8601 UTC** (`2026-08-01T14:30:00Z`).
- Identificadores públicos: **UUID** (nunca IDs autoincrementales expuestos).

## 2. Versionado

- Versión mayor en la **ruta base**: `/api/v1/...`. Un salto de versión (`v2`) sólo ante cambios incompatibles.
- Cambios retrocompatibles (nuevos campos opcionales, nuevos endpoints) **no** cambian la versión.
- Se mantiene al menos una versión mayor anterior en paralelo durante la transición.

## 3. Nombres de recursos

- Recursos en **plural, sustantivo, kebab-case**: `/training-centers`, `/schedules`, `/instructors`.
- Jerarquía por anidamiento sólo cuando el hijo no tiene sentido fuera del padre: `/fichas/{ficha_id}/sessions`.
- Nada de verbos en la ruta. Las acciones que no son CRUD se modelan como sub-recurso o como transición de estado: `POST /schedules/{id}/publish` (no `/publishSchedule`).
- Cada servicio expone sólo los recursos de su bounded context (ver [service-catalog.md](../09-microservices/service-catalog.md)).

## 4. Métodos HTTP y semántica

| Método | Uso | Idempotente |
|--------|-----|-------------|
| `GET` | Leer recurso o colección | Sí |
| `POST` | Crear recurso / disparar acción | No |
| `PUT` | Reemplazo completo del recurso | Sí |
| `PATCH` | Actualización parcial | No (por convención) |
| `DELETE` | Eliminación (lógica por defecto; ver ADR-004) | Sí |

## 5. Códigos de estado HTTP

| Código | Cuándo |
|--------|--------|
| `200 OK` | GET/PUT/PATCH exitoso con cuerpo |
| `201 Created` | POST que crea recurso; incluye `Location` y el recurso creado |
| `202 Accepted` | Trabajo aceptado para proceso asíncrono (p. ej. generación de PDF) |
| `204 No Content` | DELETE o acción sin cuerpo de respuesta |
| `400 Bad Request` | Payload malformado o validación de dominio fallida |
| `401 Unauthorized` | Token ausente, inválido o expirado (ver [authentication.md](./authentication.md)) |
| `403 Forbidden` | Feature o scope insuficiente (RBAC) |
| `404 Not Found` | Recurso inexistente o fuera del scope del usuario |
| `409 Conflict` | Conflicto de estado (p. ej. choque de horario, versión concurrente) |
| `422 Unprocessable Entity` | Reglas de negocio no satisfechas con payload sintácticamente válido |
| `429 Too Many Requests` | Límite de tasa superado |
| `500 Internal Server Error` | Error no controlado (nunca filtrar stack traces) |

## 6. Paginación, filtrado y ordenamiento

- Paginación **basada en offset** por defecto: `?page=1&page_size=20` (`page_size` máximo 100).
- Para colecciones grandes o de crecimiento continuo (auditoría, sesiones), paginación **por cursor**: `?cursor=<opaco>&limit=50`.
- Respuesta de colección envuelta con metadatos:

```json
{
  "data": [ /* ... */ ],
  "pagination": {
    "page": 1,
    "page_size": 20,
    "total_items": 137,
    "total_pages": 7
  }
}
```

- Filtrado por query params explícitos: `?training_center_id=<uuid>&status=PUBLISHED`.
- Ordenamiento: `?sort=created_at:desc` (campo permitido por endpoint, no arbitrario).

## 7. Formato de errores

Cuerpo de error uniforme en toda la plataforma (alineado con los `error_code` de [authentication.md](./authentication.md)):

```json
{
  "error": {
    "code": "SCHEDULE_CONFLICT",
    "message": "El instructor ya tiene una sesión en ese bloque horario.",
    "details": [
      { "field": "start_time", "issue": "OVERLAPS_EXISTING_SESSION" }
    ],
    "trace_id": "b3f1c2a4-..."
  }
}
```

- `code`: identificador estable en `SCREAMING_SNAKE_CASE`, independiente del idioma.
- `message`: legible para humanos, en español; no expone internals.
- `details`: opcional; errores de validación por campo.
- `trace_id`: correlación con logs y con `audit-service`.

## 8. Autenticación y autorización

- Autenticación vía **`iam-service`**: JWT Bearer RS256, verificado localmente por cada servicio con la clave pública (JWKS). Detalle en [authentication.md](./authentication.md).
- Autorización **RBAC por `feature` + `scope`**: el token trae los features pre-calculados; cada endpoint declara el feature que exige y aplica el filtro de scope en sus queries. Diseño en [rbac-design.md](../09-microservices/services/01-iam-service/rbac-design.md).
- Ningún servicio consulta a IAM por request (verificación O(1)).

## 9. Contratos

- **Contrato de implementación** (borrador, evoluciona con el servicio): junto al servicio, en `09-microservices/services/<svc>/components/<svc>-api/contract.md`.
- **Contrato OpenAPI publicable** (estable, aprobado por arquitectura): `07-api/contracts/openapi/<svc>.yaml`. Ver reglas de precedencia en [07-api/README.md](./README.md).
- El contrato se diseña **contra el modelo de datos real** del repo `*-db` correspondiente, para evitar divergencia (riesgo R-010 en [risks.md](../15-project-control/risks.md)).
- Toda API nueva se documenta con la plantilla [_template-api-contract.md](./_template-api-contract.md).

## 10. Convenciones transversales

- **Idempotencia** en escritura sensible: header `Idempotency-Key` en `POST` que crean recursos con efecto de negocio.
- **Correlación**: propagar `trace_id` entre servicios y hacia eventos/auditoría.
- **Rate limiting**: respuesta `429` con `Retry-After`; límites por identidad, no por IP.
- **Compatibilidad**: nunca romper un contrato publicado sin subir versión mayor.

## 11. Reportes

- **Descentralizados:** cada servicio **expone y responde por sus propios reportes**. **No hay hub central**; `monitoring-service` sólo publica los KPIs/indicadores de **su** dominio, no los de otros servicios.
- Endpoint de lectura dentro del propio servicio: `GET /api/v1/reports/<nombre-kebab>` (p. ej. `GET /api/v1/reports/schedule-conflicts`). Nunca mutan estado.
- Filtros por query params (rango de fechas `from`/`to`, ids de entidad, `status`), con las mismas reglas de la sección 6.
- Cada reporte declara su **inventario** en el `contract.md` del servicio: quién lo usa, frecuencia, formato de salida (JSON por defecto; `202 Accepted` + `document-service` para PDF/exports pesados) y su **fuente de datos** (tablas del repo `*-db`).

## 12. Resumen ejecutivo en contratos

- Todo `contract.md` inicia con un bloque `<!-- RESUMEN-EJECUTIVO … -->` de **≤15 líneas**: `agente`, `capacidad`, `fase`, `estado` (`draft|review|accepted`), `dependencias_entrada`, `consumidores_siguientes`, `tldr`, `decisiones_clave`, `halts_registrados`. Permite que un consumidor (o agente) lea ~300 tokens en vez del archivo completo.
