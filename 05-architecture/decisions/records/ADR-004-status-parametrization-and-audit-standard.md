# ADR-004: Parametrización de estados y estándar de auditoría

**Estado:** PROPOSED
**Fecha:** 2026-06-17
**Autores:** Jesús Ariel González Bonilla (Arquitecto) + Equipo de Datos
**Equipos involucrados:** Arquitectura, Datos, Backend (todos los servicios)

---

## Contexto

La revisión de los modelos de datos detectó tres brechas transversales:

1. **No existe una entidad padre que parametrice estados.** Los estados de negocio estaban dispersos como `VARCHAR + CHECK` inline (`schedule.status`, `enrollment_ficha.status`, `learner.enrollment_status`) o como catálogos puntuales sin un patrón común (`kpi_status`, `risk_level`). No había forma uniforme de parametrizar "cualquier estado que surja en el sistema".

2. **La auditoría era incompleta.** La convención previa solo incluía `created_at` y `updated_at`. Faltaba el **quién** (`created_by`, `updated_by`) y el **borrado lógico** (`deleted_at`, `deleted_by`).

3. **Se confundía el ciclo de vida del registro con el estado de negocio.** El `is_active BOOLEAN` (registro habilitado/deshabilitado) se mezclaba conceptualmente con el estado de negocio del agregado (DRAFT/PUBLISHED), siendo ejes ortogonales.

Estas brechas afectan a los 9 servicios, por lo que requieren una decisión transversal.

## Decisión

Se decide adoptar un **estándar transversal de modelado** documentado en [modeling-conventions.md](../../../06-data/modeling-conventions.md), con tres componentes:

1. **Patrón genérico de estados parametrizables por servicio**: cada bounded context implementa, en su propia BD, las entidades `status_category` (padre que parametriza tipos de estado), `status` (valores) y opcionalmente `status_transition` (transiciones gobernadas). Los agregados con estado de negocio referencian `status_id`.

2. **Estándar de auditoría completo con soft delete**: toda tabla transaccional lleva `created_at/created_by`, `updated_at/updated_by`, `deleted_at/deleted_by`, `is_active` y `row_version`. Las acciones del sistema usan un UUID de actor reservado. Las tablas append-only conservan solo su timestamp de inserción.

3. **Separación explícita de tres conceptos de estado**: ciclo de vida del registro (técnico, vía `is_active` + `deleted_at`), estado de negocio (parametrizable, vía `status_id`) y enum técnico cerrado (`VARCHAR + CHECK`).

**Alcance de conversión**: solo los **estados de negocio** migran al catálogo `status`. Los enums técnicos cerrados (ej. `document_type`, `contract_type`, `channel`) permanecen como `VARCHAR + CHECK`.

**Ubicación**: patrón **replicado por servicio**, no una tabla global compartida — para no violar DB-por-servicio ni la propiedad de datos por bounded context.

## Consecuencias

### Positivas

- Una única forma de parametrizar cualquier estado de negocio, extensible en runtime sin deploy
- Trazabilidad completa: se sabe quién creó, modificó y eliminó cada registro, y cuándo
- El soft delete preserva historia y permite recuperación; cumple requisitos de auditoría (Ley 1581)
- Separación limpia: habilitar/deshabilitar un registro es independiente de su estado de negocio
- Las transiciones de estado se parametrizan (`status_transition`) en vez de hardcodearse, y se ligan a features de RBAC
- Preserva la independencia de los bounded contexts (cada servicio dueño de sus estados)

### Negativas / Trade-offs

- Cada agregado con estado de negocio pasa de un `VARCHAR + CHECK` simple a una FK (`status_id`) + join para resolver el código
- Cada servicio replica el patrón `status_category`/`status` (duplicación estructural intencional, no de datos)
- Las columnas de auditoría agregan 8 campos a cada tabla transaccional
- Validar transiciones contra `status_transition` agrega lógica al servicio (antes era un CHECK)

### Riesgos

- **Inconsistencia de seeds entre servicios** si los códigos de estado divergen. Mitigación: el estándar documenta los códigos canónicos por categoría; revisión en PR.
- **Migración de los 9 modelos estables**: ya están en estado 🟢 Estable. Mitigación: aplicar el estándar de forma incremental por servicio, empezando por los que tienen máquinas de estado de negocio reales (scheduling, academic, actors, monitoring).

## Alternativas consideradas

| Alternativa | Por qué se descartó |
|-------------|---------------------|
| Tabla `status` global en reference-data-service | Crea dependencia de lectura runtime y acopla las máquinas de estado a un servicio compartido; viola DB-por-servicio para escritura |
| Dejar todo como `VARCHAR + CHECK` | No permite parametrización en runtime ni metadata (labels, colores, orden, transiciones); el negocio no puede extender estados sin deploy |
| Convertir TODO enum (incluidos técnicos) al catálogo | Pierde la integridad simple y barata del CHECK para conjuntos cerrados inmutables; agrega joins innecesarios |
| Solo timestamps sin `_by` | No cumple el requisito de trazabilidad de autoría que motivó la decisión |
| Borrado físico en vez de soft delete | Pierde trazabilidad de lo eliminado; incompatible con auditoría y recuperación |

## Referencias

- [modeling-conventions.md](../../../06-data/modeling-conventions.md) — estándar detallado
- [06-data/models.md](../../../06-data/models.md) — modelo lógico global
- [entities-and-rules.md](../../../02-domain/entities-and-rules.md) — RN-ACAD-05, RN-SCH-01 (transiciones)
- [rbac-design.md](../../../09-microservices/services/01-iam-service/rbac-design.md) — features que gobiernan transiciones
