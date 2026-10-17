# Requisitos

> Estado: 🟡 En progreso | Última actualización: 2026-08-28
> Autor: Jesús Ariel González Bonilla (PM + Arquitecto) | Equipo: Producto

## Contenido

Centraliza los requisitos del sistema **SENA Gestión de Horarios**: funcionales, no funcionales, historias de usuario y su trazabilidad. Traduce la visión de producto ([03-product/vision.md](../03-product/vision.md)) y las reglas de dominio ([02-domain/entities-and-rules.md](../02-domain/entities-and-rules.md)) en requisitos verificables que guían la construcción de los **10 servicios** (los 9 originales + `notification-service`, extraído de `monitoring` por ADR-005).

> **Estado de construcción.** A 2026-08-28 hay dos servicios **construidos y verificados en `dev`**: `iam-service` (EP-IAM, 12/12 HU Done·Dev) y `notification-service` (EP-NOTIF, 8/8 HU Done·Dev). Aún no están promovidos a `qa`/`staging`/`main` (solo la capa de datos está avanzada). Estos `.md` son la **SSOT de trazabilidad HU ↔ requisito**, no del estado de ejecución: el **estado vivo** por historia se lleva en **GitHub Project #3**.

## Archivos

| Archivo | Descripción | Estado |
|---------|-------------|--------|
| [functional.md](./functional.md) | Requisitos funcionales por módulo (RF-*), derivados de las entidades reales | 🟡 |
| [non-functional.md](./non-functional.md) | Requisitos de calidad, seguridad, rendimiento y operación (NFR-*) | 🟡 |
| [user-stories.md](./user-stories.md) | Historias de usuario por rol (HU-*) con criterios de aceptación | 🟡 |
| [traceability-matrix.md](./traceability-matrix.md) | Relación RF ↔ HU ↔ servicio ↔ RN ↔ ADR | 🟡 |
| [_template-hu.md](./_template-hu.md) | Plantilla para documentar una historia de usuario | — |
| [_template-nfr.md](./_template-nfr.md) | Plantilla para requisitos no funcionales | — |

## Convención de identificadores

| Prefijo | Significado | Ejemplo |
|---------|-------------|---------|
| `RF-<MÓDULO>-##` | Requisito funcional por módulo | `RF-SCH-03` |
| `NFR-<CATEGORÍA>-##` | Requisito no funcional | `NFR-PERF-01` |
| `HU-##` | Historia de usuario (planeación / backlog) | `HU-09` |
| `HU-<ÉPICA>-###` | Historia de **ejecución** por épica construida | `HU-IAM-007` |
| `EP-<ÉPICA>` | Épica de construcción (issue en repo docs) | `EP-IAM` (docs#79) |
| `docs#NNN` | Issue de trazabilidad en el repo docs | `docs#80` |
| `RN-<DOMINIO>-##` | Regla de negocio (en [02-domain](../02-domain/entities-and-rules.md)) | `RN-SCH-04` |

Módulos de RF: `IAM`, `REF`, `ACAD`, `ENV`, `SCH` (**CORE**), `ACT`, `DOC`, `MON` (**CORE**), `NOTIF`, `AUD`.

**Convención de IDs de ejecución.** Cada épica construida usa IDs propios: la épica es `EP-<ÉPICA>` (issue en el repo docs) y sus historias `HU-<ÉPICA>-###`, cada una atada a un issue `docs#NNN`. Épicas construidas: **EP-IAM** (`HU-IAM-001..012`, issues docs#80–#91) y **EP-NOTIF** (`HU-NOTIF-001..008`). Estos IDs conviven con los de planeación `HU-01..HU-23`; el mapeo entre ambos está en [traceability-matrix.md](./traceability-matrix.md) y [user-stories.md](./user-stories.md).

## Flujo de trazabilidad

```
02-domain (RN-*) ──▶ functional (RF-*) ──▶ user-stories (HU-*)
        │                   │                     │
        └───────────────────┴─────────────────────┴──▶ traceability-matrix ──▶ ADRs
```

Ver convenciones de naming, estatus e integración con tracker en [00-governance/agile-conventions.md](../00-governance/agile-conventions.md).
