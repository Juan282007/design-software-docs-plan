# Plan de migración de datos — <PROJECT_KEY>

> **PLANTILLA** — Copiar como `data-migration-plan.md` y completar.
> Eliminar esta línea antes de hacer commit.

> Estado: 🔴 Pendiente | Última actualización: YYYY-MM-DD
> Autor: Por definir | Equipo: Arquitectura de Datos

## Contexto

| Campo | Valor |
|-------|-------|
| Tipo de migración | [legacy-rewrite / nueva versión de schema / integración] |
| Origen | [sistema legacy / BD actual] |
| Destino | [nuevo esquema / nuevo motor] |
| Ventana de migración | [con downtime / sin downtime / migración progresiva] |

## Mapeo de datos

| Entidad/Tabla origen | Entidad/Tabla destino | Transformación | Validación |
|---------------------|----------------------|----------------|------------|
| `tbl_vieja.campo` | `tabla_nueva.columna` | [concatenar / normalizar / calcular] | [regla de validación] |

## Pasos de migración

| Paso | Descripción | Script | Responsable | Rollback |
|------|-------------|--------|-------------|----------|
| 1 | [Crear tablas nuevas en paralelo] | `V001__create_tables.sql` | DBA | DROP TABLE |
| 2 | [ETL batch de datos históricos] | `migration_batch.py` | Data | Datos intactos en origen |
| 3 | [Validar conteos y checksums] | `validate.sql` | QA | — |
| 4 | [Cutover: apuntar tráfico al nuevo esquema] | — | DevOps | Rollback a paso 3 |

## Validaciones post-migración

| Validación | Query / Método | Criterio de aceptación |
|------------|----------------|----------------------|
| Conteo de registros | `SELECT COUNT(*)` | Origen == Destino ± tolerancia |
| Integridad referencial | FK checks | 0 violaciones |
| Datos PII cifrados | Inspección de columnas | Sin valores planos |

## Plan de rollback

| Condición de rollback | Acción | Tiempo estimado |
|-----------------------|--------|-----------------|
| Validación fallida en paso 3 | Restaurar backup pre-migración | [X min] |
| Error en cutover | Reapuntar conexiones a BD original | [X min] |

## Referencias

- [DB Design](./db-design.md)
- [Data Model](./data-model.md)
- [Rollback Plan](../10-devops/rollback-plan.md)
