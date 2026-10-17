<!-- RESUMEN-EJECUTIVO
agente: Jesús Ariel González Bonilla (PM + Arquitecto)
capacidad: contrato de API (OpenAPI 3.1)
fase: diseño (api-first)
estado: accepted
dependencias_entrada: 09-microservices/services/03-academic-management-service/data-model.md, 07-api/guidelines.md, 07-api/contracts/openapi/_shared.yaml
consumidores_siguientes: backend academic-management-service, scheduling-service (fichas), frontend, pruebas de contrato
tldr: Jerarquía curricular (tech_line→tech_network→knowledge_network→training_program), competencias/resultados de aprendizaje anidados y fichas de matrícula con máquina de estados, más 2 reportes propios. La fuente de verdad es academic-management.yaml.
decisiones_clave: OpenAPI publicable en 07-api/contracts/openapi/academic-management.yaml; competencias y resultados de aprendizaje anidados bajo su padre (sin sentido fuera de él); program_version se congela server-side al abrir la ficha; cambios de status de ficha solo vía PATCH /fichas/{id}/status
halts_registrados: ninguno
-->

# Contrato — academic-management-api

> Estado: 🟢 Aceptado | Última actualización: 2026-08-06

> **Fuente de verdad (normativa):** el spec OpenAPI 3.1 en
> [`07-api/contracts/openapi/academic-management.yaml`](../../../../../07-api/contracts/openapi/academic-management.yaml).
> Este documento es la **narrativa** que lo explica; ante cualquier diferencia, **manda el
> `academic-management.yaml`**. Convenciones transversales en
> [07-api/guidelines.md](../../../../../07-api/guidelines.md).

## Autenticación
`Authorization: Bearer <JWT>` de `iam-service`; autorización por feature + scope (un coordinador solo opera en su centro de formación → `scope = TRAINING_CENTER`).

## Base URL
`/api/v1`

## Endpoints (entidades reales)
| Método | Path | Descripción | Feature |
|--------|------|-------------|---------|
| `GET` `POST` | `/tech-lines` | Listar / crear líneas tecnológicas | `ACAD_READ` / `ACAD_MANAGE` |
| `GET` `PUT` `DELETE` | `/tech-lines/{id}` | Detalle / reemplazo / baja lógica | `ACAD_READ` / `ACAD_MANAGE` / `ACAD_ADMIN` |
| `GET` `POST` | `/tech-networks` | Listar (filtro `tech_line_id`) / crear redes tecnológicas | `ACAD_READ` / `ACAD_MANAGE` |
| `GET` `PUT` `DELETE` | `/tech-networks/{id}` | Detalle / reemplazo / baja lógica | `ACAD_READ` / `ACAD_MANAGE` / `ACAD_ADMIN` |
| `GET` `POST` | `/knowledge-networks` | Listar (filtro `tech_network_id`) / crear redes de conocimiento | `ACAD_READ` / `ACAD_MANAGE` |
| `GET` `PUT` `DELETE` | `/knowledge-networks/{id}` | Detalle / reemplazo / baja lógica | `ACAD_READ` / `ACAD_MANAGE` / `ACAD_ADMIN` |
| `GET` `POST` | `/training-programs` | Listar (paginado) / crear programa | `ACAD_READ` / `ACAD_MANAGE` |
| `GET` `PUT` `DELETE` | `/training-programs/{id}` | Detalle + versión / reemplazo / baja lógica | `ACAD_READ` / `ACAD_MANAGE` / `ACAD_ADMIN` |
| `GET` `POST` | `/training-programs/{id}/competencies` | Competencias del programa / crear competencia | `ACAD_READ` / `ACAD_MANAGE` |
| `GET` `PUT` `DELETE` | `/competencies/{id}` | Detalle / reemplazo / baja lógica | `ACAD_READ` / `ACAD_MANAGE` / `ACAD_ADMIN` |
| `GET` `POST` | `/competencies/{id}/learning-outcomes` | Resultados de aprendizaje / crear | `ACAD_READ` / `ACAD_MANAGE` |
| `GET` `PUT` `DELETE` | `/learning-outcomes/{id}` | Detalle / reemplazo / baja lógica | `ACAD_READ` / `ACAD_MANAGE` / `ACAD_ADMIN` |
| `GET` `POST` | `/fichas` | Listar fichas (filtros: `program_id`, `status`, `ficha_number`, `training_center_id`, `from`/`to`) / crear ficha | `ACAD_READ` / `ACAD_MANAGE` |
| `GET` `PUT` `DELETE` | `/fichas/{id}` | Detalle / actualizar datos operativos / baja lógica (→ `CANCELLED`) | `ACAD_READ` / `ACAD_MANAGE` / `ACAD_ADMIN` |
| `PATCH` | `/fichas/{id}/status` | Cambiar estado de la ficha (máquina de estados) | `ACAD_MANAGE` |

## Ejemplo de respuesta (ficha)
```json
{
  "id": "uuid",
  "ficha_number": "2874521",
  "program_id": "uuid",
  "program_version": 3,
  "status": "INDUCTION",
  "training_center_id": "uuid",
  "created_at": "2026-08-01T14:00:00Z"
}
```

## Reportes (dominio propio)

Reportes **descentralizados** (guidelines §11): este servicio expone y responde por sus propios
reportes sobre `enrollment_ficha`. Solo lectura, paginación por offset (`page`/`page_size`) como el
resto de colecciones del servicio.

### `GET /reports/enrollment-by-ficha`

Capacidad y estado de oferta por ficha (proyección de `enrollment_ficha` con `program_code` /
`program_name` de `training_program`).

- **Feature requerido:** `ACAD_READ`.
- **Filtros:** `training_center_id`, `program_id`, `status`, `from`/`to` (rango de `start_date`).
- **Inventario:** Usuarios: coordinadores académicos y dashboards de centro · Frecuencia:
  on-demand/diaria · Formato: JSON · Fuente: `enrollment_ficha` + `training_program`.

### `GET /reports/ficha-progress`

Avance temporal de fichas: días transcurridos desde `start_date`, días restantes hasta
`expected_end_date` y bandera `is_delayed` cuando se supera la fecha esperada sin llegar a un
estado terminal (`COMPLETED`/`CANCELLED`).

- **Feature requerido:** `ACAD_READ`.
- **Filtros:** `training_center_id`, `program_id`, `status`, `from`/`to` (rango de `start_date`).
- **Inventario:** Usuarios: coordinadores académicos y subdirectores de centro · Frecuencia:
  on-demand/semanal · Formato: JSON · Fuente: `enrollment_ficha`.

## Formato de error estándar

Aplica el **envelope estándar de la plataforma** ([guidelines §7](../../../../../07-api/guidelines.md) /
`_shared.yaml#/components/schemas/Error`):

```json
{
  "error": {
    "code": "FICHA_NUMBER_ALREADY_EXISTS",
    "message": "El número de ficha ya existe",
    "details": [ { "field": "ficha_number", "issue": "UNIQUE_VIOLATION" } ],
    "trace_id": "uuid-v4"
  }
}
```

## Códigos de error propios

| Código | HTTP | Descripción |
|--------|------|-------------|
| `FICHA_NUMBER_ALREADY_EXISTS` | 409 | El `ficha_number` ya existe |
| `PROGRAM_CODE_ALREADY_EXISTS` | 409 | El `program_code` ya existe |
| `INVALID_STATUS_TRANSITION` | 409 | Transición de `status` no permitida por la máquina de estados de la ficha |
| `TERMINAL_STATUS` | 409 | La ficha ya está en un estado terminal (`COMPLETED`/`CANCELLED`) |

## Eventos publicados
`academic.enrollment_ficha.created`, `academic.enrollment_ficha.status_changed`, `academic.training_program.published`. Ver [event-catalog.md](../../../../event-catalog.md).
