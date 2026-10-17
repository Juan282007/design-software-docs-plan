# Modelo de datos — [service-name]

> **PLANTILLA** — Completar con las entidades propias del bounded context.
> Estado: 🔴 Pendiente | Última actualización: YYYY-MM-DD

## Entidades propias

### [NombreEntidad]

| Campo | Tipo | Nullable | Descripción |
|-------|------|----------|-------------|
| `id` | UUID | No | PK |
| `created_at` | TIMESTAMP | No | Fecha de creación |
| `updated_at` | TIMESTAMP | No | Fecha de última modificación |

## Referencias externas

<!-- Solo IDs foráneos. No FKs cruzadas entre bases de datos. -->

| Campo | Referencia lógica | Servicio propietario |
|-------|-------------------|----------------------|
| `[id_externo]` | `[Entidad].[id]` | `[servicio]-service` |

## Índices relevantes

| Tabla | Campos indexados | Tipo | Motivo |
|-------|-----------------|------|--------|

## Notas de integridad

<!-- Reglas de negocio que no se expresan en constraints de BD -->
