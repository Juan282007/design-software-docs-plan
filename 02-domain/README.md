# Dominio

> Estado: 🟡 En progreso | Última actualización: 2026-08-28
> Autor: Jesús Ariel González Bonilla (PM + Arquitecto) | Equipo: Arquitectura

Índice de la sección de dominio del sistema de Gestión de Horarios SENA. Describe el modelo del negocio: los bounded contexts, las entidades y sus invariantes, las reglas de negocio y los eventos de dominio.

## Propósito

Capturar el **diseño de dominio** (DDD) que sirve de puente entre el [contexto](../01-context/) de negocio y la [arquitectura](../05-architecture/) técnica: qué contextos existen, cómo se relacionan, qué reglas rigen y qué eventos comunican los cambios. Es la referencia para validaciones en servicios, contratos de API y decisiones de modelado.

> **Diferencia con `06-data`:** esta sección describe el dominio en términos de **negocio** (entidades, invariantes, reglas, lenguaje ubicuo). No describe tablas, columnas ni esquemas de base de datos; esos detalles de implementación viven en [`06-data/`](../06-data/).

## Archivos

| Archivo | Descripción | Estado |
|---------|-------------|--------|
| [domain-map.md](./domain-map.md) | Diseño estratégico: bounded contexts, context map (DDD), clasificación CORE/SUPPORTING/GENERIC y lenguaje ubicuo | 🟡 |
| [entities-and-rules.md](./entities-and-rules.md) | Entidades, invariantes y reglas de negocio no derivables del código (RN-*) | 🟡 |
| [domain-events.md](./domain-events.md) | Eventos de dominio y su significado funcional (vista de negocio del event-catalog) | 🟡 |

## Cómo se relaciona

- Los **contextos CORE** (`scheduling-service`, `monitoring-service`) concentran el mayor esfuerzo de modelado; los GENERIC usan implementaciones directas (ver [domain-map.md](./domain-map.md)).
- El **context map** se materializa en la [arquitectura](../05-architecture/pattern-guide.md) y en las reglas de frontera de [09-microservices](../09-microservices/service-boundary-rules.md).
- Los **eventos de dominio** ([domain-events.md](./domain-events.md)) son la vista de negocio del [event-catalog.md](../09-microservices/event-catalog.md) (vista técnica); su transporte se decide en [ADR-001](../05-architecture/decisions/records/ADR-001-message-broker.md).
- Las **reglas de estado y transición** (RN-*) se implementan con el patrón parametrizable de [ADR-004](../05-architecture/decisions/records/ADR-004-status-parametrization-and-audit-standard.md).
