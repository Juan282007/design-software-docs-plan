# Mapa de Dominio — SENA Gestión de Horarios

> Estado: 🟡 Borrador | Última actualización: 2026-08-28
> Fuente: análisis M1-M9 · DDD Strategic Design
> Relacionado: [entities-and-rules.md](./entities-and-rules.md) · [pattern-guide.md](../05-architecture/pattern-guide.md)

## Propósito

Este documento describe el diseño estratégico del dominio: los bounded contexts, sus relaciones (context map) y la clasificación de cada contexto según su valor de negocio. Es la vista de "entendimiento" que conecta el dominio SENA con la arquitectura de microservicios.

---

## Clasificación de subdominios

En DDD, no todos los subdominios tienen el mismo valor estratégico. Esta clasificación guía dónde invertir más esfuerzo de modelado.

| Subdominio | Tipo | Bounded Context | Justificación |
|-----------|------|-----------------|---------------|
| **Asignación de horarios** | CORE | scheduling-service | Es la razón de ser del sistema; la detección de conflictos es la lógica diferenciadora |
| **Seguimiento y alertas** | CORE | monitoring-service | Genera el valor pedagógico: detectar riesgo de deserción y desviaciones |
| Gestión de actores | SUPPORTING | actors-service | Necesario para horarios, pero no diferenciador en sí mismo |
| Gestión de ambientes | SUPPORTING | training-environment-service | Insumo crítico para horarios; reglas de disponibilidad propias del SENA |
| Gestión académica | SUPPORTING | academic-management-service | Estructura curricular; mayormente derivada de SOFIA Plus |
| Identidad y acceso | GENERIC | iam-service | Problema resuelto; patrón estándar RBAC |
| Datos de referencia | GENERIC | reference-data-service | Catálogos y jerarquía; CRUD estándar |
| Documentos | GENERIC | document-service | Generación de PDFs; patrón conocido |
| Auditoría | GENERIC | audit-service | Log append-only; patrón estándar |
| Notificaciones | GENERIC (transversal) | notification-service | Micro transversal extraído de monitoring (ADR-005); consume eventos y entrega notificaciones con idempotencia + DLQ |

**Implicación**: el mayor esfuerzo de diseño y pruebas debe concentrarse en `scheduling-service` y `monitoring-service` (CORE). Los contextos GENERIC pueden usar implementaciones más directas.

---

## Context Map — Relaciones entre bounded contexts

Notación de patrones de relación DDD:
- **U** = Upstream (servicio proveedor); **D** = Downstream (servicio consumidor)
- **CS** = Customer-Supplier; **C** = Conformist; **ACL** = Anti-Corruption Layer
- **OHS** = Open Host Service; **PL** = Published Language

```
                        ┌──────────────────────┐
                        │     iam-service       │
                        │ (OHS - emite JWT RS256)│  U
                        └───────────┬───────────┘
                                    │ JWT RS256 (PL) — verificado local vía JWKS
              ┌─────────────────────┼─────────────────────┐
              │ todos los servicios consumen identidad     │ D
              ▼                                            ▼
   ┌──────────────────────┐                  ┌──────────────────────────┐
   │ reference-data       │  U               │  academic-management     │  U
   │ (OHS - catálogos)    │─────────────────▶│  (CS - fichas, programas)│
   └──────────────────────┘  Conformist      └────────────┬─────────────┘
                                                           │ U (fichas, competencias)
                                                           │
         ┌─────────────────────────────────────────────────┤
         │ CS                          CS                    │ CS
         ▼                              ▼                    ▼
┌─────────────────┐        ┌────────────────────┐   ┌──────────────────┐
│ actors-service  │   U    │ training-          │ U │  scheduling-     │ ← CORE
│                 │───────▶│ environment-service│──▶│  service         │
│  (CS via ACL)   │ events │                    │   │  (read models)   │
└────────┬────────┘        └─────────┬──────────┘   └────────┬─────────┘
         │                           │                       │
         │ U (events)                │ U (events)            │ U (events)
         └───────────────┬───────────┴───────────────────────┘
                         ▼
              ┌──────────────────────┐
              │  monitoring-service  │ ← CORE
              │  (D - consume events)│
              └──────────────────────┘

         ┌──────────────────────┐      ┌──────────────────────┐
         │  document-service    │      │   audit-service      │
         │  (D - consume events)│      │ (D - consume TODO)   │
         └──────────────────────┘      └──────────────────────┘

         ┌────────────────────────────────────────────────────┐
         │           notification-service (TRANSVERSAL)        │
         │  (D - consume monitoring.*/scheduling.*/iam.* con   │
         │   idempotencia por event_id + DLQ; publica          │
         │   notification.notification.sent)                   │
         └────────────────────────────────────────────────────┘
```

---

## Detalle de relaciones

| Upstream (U) | Downstream (D) | Patrón | Mecanismo | Descripción |
|--------------|----------------|--------|-----------|-------------|
| iam-service | Todos | OHS + Published Language | JWT RS256 verificado vía JWKS | iam publica identidad y features como JWT firmado con RS256; edge y servicios **verifican localmente** con las claves públicas de JWKS (`/.well-known/jwks.json`), sin llamada síncrona a iam. La rotación de clave se gestiona por JWKS (DEC-008) |
| monitoring/scheduling/iam (emisores) | notification-service | Customer-Supplier (consumer transversal) | Eventos → notificación | notification-service consume eventos de dominio (alertas, publicaciones, identidad) y entrega notificaciones con idempotencia por `event_id` + DLQ; publica `notification.notification.sent` |
| reference-data-service | academic, actors, environment | Conformist | API REST + caché | Los consumidores aceptan el modelo de catálogos tal cual; no lo traducen |
| academic-management-service | scheduling-service | Customer-Supplier | API REST (sync puntual) | scheduling valida fichas/competencias contra academic al crear borrador |
| academic-management-service | actors-service | Customer-Supplier | Evento `academic.competency.updated` | actors actualiza sus referencias de competencia |
| actors-service | scheduling-service | Customer-Supplier vía read model | Eventos → read model local | scheduling proyecta instructores y competencias localmente (ADR-002) |
| training-environment-service | scheduling-service | Customer-Supplier vía read model | Eventos → read model local | scheduling proyecta ambientes y disponibilidad localmente (ADR-002) |
| scheduling-service | monitoring-service | Customer-Supplier | Evento `scheduling.schedule.published` | monitoring inicia seguimiento al publicarse un horario |
| Todos | audit-service | Conformist (consumer universal) | Eventos (wildcard) | audit consume todos los eventos sin imponer modelo |
| Todos | document-service | Customer-Supplier | Eventos + API | document genera PDFs bajo demanda de otros servicios |

---

## Anti-Corruption Layers (ACL)

Un ACL traduce el modelo de un contexto externo al modelo interno, evitando que conceptos ajenos contaminen el dominio.

| Contexto | ACL hacia | Por qué |
|----------|-----------|---------|
| academic-management-service | SOFIA Plus (sistema nacional SENA) | Los números de ficha y códigos de programa vienen de SOFIA Plus; un ACL traduce su formato al modelo interno sin acoplar el dominio al sistema legado |
| actors-service | iam-service | El `user_id` externo se traduce al `Instructor`/`Learner` interno; el evento `iam.user.created` dispara la creación del perfil de actor |

---

## Lenguaje Ubicuo — Mapeo dominio ↔ técnico

El equipo habla en español (dominio SENA); el código y los contratos van en inglés (HALT-DB-NAMING). Este es el puente:

| Concepto SENA (es) | Bounded Context | Entidad técnica (en) |
|--------------------|-----------------|----------------------|
| Ficha de caracterización | academic-management | `EnrollmentFicha` |
| Programa de formación | academic-management | `TrainingProgram` |
| Competencia | academic-management | `Competency` |
| Resultado de Aprendizaje (RAP) | academic-management | `LearningOutcome` |
| Línea / Red tecnológica | academic-management | `TechLine` / `TechNetwork` |
| Ambiente de formación | training-environment | `Environment` |
| Instructor | actors | `Instructor` |
| Aprendiz | actors | `Learner` |
| Etapa productiva | actors | `ProductiveStage` |
| Horario | scheduling | `Schedule` |
| Sesión de clase | scheduling | `ClassSession` |
| Franja horaria | scheduling | `TimeSlot` |
| Conflicto | scheduling | `SchedulingConflict` |
| Seguimiento de ficha | monitoring | `FichaTracking` |
| Alerta | monitoring | `GeneratedAlert` |
| Plan de mejoramiento | monitoring / actors | `ImprovementPlan` / `ActorImprovementPlan` |

---

## Reglas de evolución de los bounded contexts

1. **Una entidad pertenece a un solo contexto** (ver [data-ownership-matrix.md](../09-microservices/data-ownership-matrix.md))
2. **Cruzar una frontera requiere ADR** — toda nueva dependencia entre contextos se documenta
3. **Los contextos CORE no se fusionan** con SUPPORTING/GENERIC para mantener su foco
4. **Cambios en el Published Language (JWT, eventos) son breaking** y requieren versionado
