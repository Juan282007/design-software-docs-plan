# Contrato API — <PROJECT_KEY> — [Nombre del servicio]

> **PLANTILLA** — Copiar como `api-contract.md` en la carpeta del servicio o en `07-api/contracts/`.
> Eliminar esta línea antes de hacer commit.

> Estado: 🔴 Pendiente | Última actualización: YYYY-MM-DD
> Autor: Por definir | Equipo: Arquitectura + Backend

## Metadata

| Campo | Valor |
|-------|-------|
| Servicio | [nombre] |
| Version API | v1 |
| Base URL | `https://api.<proyecto>.com/v1` |
| Formato | REST / JSON |
| OpenAPI spec | `07-api/contracts/openapi/<servicio>.yaml` (cuando exista) |

## Autenticación y autorización

| Mecanismo | Descripción |
|-----------|-------------|
| Autenticación | [JWT Bearer / API Key / mTLS] |
| Header requerido | `Authorization: Bearer <token>` |
| Token TTL | acceso: 1h / refresh: 7d |
| Roles | [admin, user, readonly] |

## Convenciones generales

- Fechas: ISO 8601 UTC — `YYYY-MM-DDTHH:MM:SSZ`
- Paginación: `?page=1&limit=20` (límite máximo: 100)
- IDs: UUID v4
- Encoding: UTF-8
- Content-Type: `application/json`

### Formato de errores estándar

```json
{
  "error": {
    "code": "VALIDATION_ERROR",
    "message": "El campo email es obligatorio",
    "details": [{ "field": "email", "message": "required" }],
    "trace_id": "abc123"
  }
}
```

| HTTP | Cuándo usar |
|------|-------------|
| 200 | Operación exitosa |
| 201 | Recurso creado |
| 204 | Exitoso sin cuerpo |
| 400 | Datos inválidos |
| 401 | Sin autenticación válida |
| 403 | Sin permiso |
| 404 | Recurso no existe |
| 409 | Conflicto de negocio |
| 429 | Rate limiting |
| 500 | Error del servidor |

## Endpoints

### POST /[recurso]

**Descripción:** Crear un nuevo [recurso].
**Auth:** Requerida. Rol: [user, admin]

**Request:**
```json
{
  "campo1": "valor",
  "campo2": "valor"
}
```

| Campo | Tipo | Obligatorio | Validaciones |
|-------|------|-------------|-------------|
| campo1 | string | Sí | |

**Response 201:**
```json
{
  "id": "uuid-v4",
  "campo1": "valor",
  "created_at": "2026-01-15T10:30:00Z"
}
```

**Errores:**

| HTTP | Código | Cuándo |
|------|--------|--------|
| 400 | VALIDATION_ERROR | Campo inválido |
| 409 | DUPLICATE | Ya existe |

---

### GET /[recurso]/{id}

**Descripción:** Obtener un [recurso] por ID.
**Auth:** Requerida.

**Response 200:** objeto completo.

**Errores:** 404 NOT_FOUND, 403 FORBIDDEN.

---

### GET /[recurso]

**Descripción:** Listar con paginación.
**Auth:** Requerida.

**Query params:** `page`, `limit`, `sort`, `order`.

**Response 200:** `{ "data": [...], "pagination": { "page": 1, "limit": 20, "total": N } }`

## Rate limiting

| Scope | Límite | Ventana |
|-------|--------|---------|
| Por IP | [5000 req] | 1 hora |
| Por usuario | [1000 req] | 1 minuto |

## Versionado

- Versión en URL: `/api/v1/`
- Deprecación: anunciada con 6 meses de anticipación via header `Deprecation: true`.

## Referencias

- [Architecture](../05-architecture/architecture.md)
- [Data Model](../06-data/data-model.md)
- [Security Threat Model](../05-architecture/security-threat-model.md)
