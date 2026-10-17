# Contrato — [component-name]

> **PLANTILLA** — Completar la sección que corresponda según el tipo del componente.
> Eliminar las secciones que no apliquen.
>
> Estado: 🔴 Pendiente | Última actualización: YYYY-MM-DD

---

## REST (para `-api`)

### Autenticación

Bearer JWT emitido por `iam-service`. Todos los endpoints requieren header:
```
Authorization: Bearer <token>
```

### Base URL

`/api/v1/[recurso]`

### Endpoints

| Método | Path | Descripción | Roles permitidos |
|--------|------|-------------|-----------------|
| `GET` | `/[recurso]` | Listar (paginado) | `*` |
| `GET` | `/[recurso]/{id}` | Obtener uno | `*` |
| `POST` | `/[recurso]` | Crear | `admin`, `coordinador` |
| `PUT` | `/[recurso]/{id}` | Actualizar | `admin`, `coordinador` |
| `DELETE` | `/[recurso]/{id}` | Eliminar (soft delete) | `admin` |

### Formato de error estándar

```json
{
  "error_code": "RESOURCE_NOT_FOUND",
  "message": "El recurso solicitado no existe",
  "trace_id": "uuid-v4"
}
```

---

## Worker (para `-worker`)

### Topic / Queue consumido

`[nombre-del-topic]`

### Payload de entrada

```json
{
  "event_type": "[servicio].[entidad].[accion]",
  "payload": {}
}
```

### Acción que ejecuta

<!-- Qué hace el worker al procesar cada mensaje -->

### Salida / side-effects

<!-- Eventos que publica, registros que escribe -->

### Dead letter queue

`[nombre-del-topic].dlq` — mensajes fallidos después de 3 reintentos.

---

## Workflow (para `-workflow`)

### Steps

| # | Step | Tipo | Compensación si falla |
|---|------|------|----------------------|
| 1 | [nombre] | [sync/async] | [acción de compensación] |

### Input

```json
{}
```

### Output

```json
{}
```

### Timeouts

| Step | Timeout | Política |
|------|---------|---------|
