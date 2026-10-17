# Visión de Producto — SENA Gestión de Horarios

> Estado: 🟡 En progreso | Última actualización: 2026-08-28
> Autor: Jesús Ariel González Bonilla (PM + Arquitecto) | Equipo: Producto

> Fuente: [discovery-brief.md](./discovery-brief.md) · [problem-framing.md](./problem-framing.md) · [01-context/overview.md](../01-context/overview.md)

## 1. El problema

Los centros de formación del SENA operan decenas de fichas de caracterización activas de forma simultánea. Cada semana, el coordinador académico debe asignar a cada ficha en ejecución: instructores con la competencia certificada, ambientes físicos con capacidad y equipamiento adecuado, y franjas horarias compatibles con la disponibilidad de todos.

Hoy ese proceso es **manual o semi-manual**: no existe una herramienta que valide conflictos de recursos antes de publicar, ni que comunique el horario resultante a instructores y aprendices. La consecuencia son conflictos detectados tarde (un instructor citado en dos ambientes a la misma hora, un ambiente sobreprogramado), 4–8 horas semanales de trabajo de reconciliación, y falta de visibilidad para quienes deben cumplir el horario.

La causa raíz (ver [problem-framing.md](./problem-framing.md), 5 Whys) es la **ausencia de un sistema operativo de centro** que integre la visibilidad de recursos con el proceso de construcción y publicación de horarios. El sistema nacional SOFIA Plus fue diseñado para el registro académico, no para la coordinación operativa diaria de un centro.

## 2. Objetivo del producto

> Permitir que un coordinador académico **construya, valide y publique** un horario de formación libre de conflictos en **menos de una hora**, y que instructores y aprendices consulten su horario vigente en tiempo real; habilitando además el **seguimiento pedagógico** de cada ficha con alertas tempranas de riesgo.

Objetivos medibles (alineados con [overview.md](../01-context/overview.md) y [non-functional.md](../04-requirements/non-functional.md)):

1. Crear y publicar un horario válido en **< 1 h** (vs. 4–8 h manuales) — NFR-USA-01.
2. **Detectar automáticamente** conflictos de instructor, ambiente y franja antes de publicar — CF-02.
3. Dar **visibilidad en tiempo real** del horario a instructores y aprendices — CF-04, CF-05.
4. Habilitar el **seguimiento** de asistencia, avance y riesgo de deserción por ficha.
5. Garantizar **trazabilidad** completa de las acciones del sistema (auditoría append-only).

## 3. Usuarios y sus necesidades

| Rol | Necesidad principal | Valor que recibe |
|-----|---------------------|------------------|
| **Coordinador académico** | Construir horarios sin conflictos y publicarlos rápido | Motor que valida disponibilidad y bloquea publicación con conflictos |
| **Instructor** | Conocer su carga horaria con anticipación y gestionar su disponibilidad | Consulta de su horario por semana; registro de excepciones de disponibilidad |
| **Aprendiz** | Saber cuándo y dónde tiene clase, y enterarse de cambios | Consulta del horario de su ficha, siempre actualizado |
| **Administrador / Director de centro** | Métricas de utilización de ambientes y carga de instructores; alertas de riesgo | KPIs y alertas del servicio de seguimiento |

El detalle del dolor de cada rol está en [discovery-brief.md](./discovery-brief.md) y [problem-framing.md](./problem-framing.md) (mapa de impacto).

## 4. Propuesta de valor

Una **plataforma de operación de centro** que:

- **Centraliza** la disponibilidad de instructores, ambientes y franjas en un solo lugar.
- **Valida antes de publicar**: el motor de horarios (`scheduling-service`, contexto CORE) detecta `INSTRUCTOR_DOUBLE_BOOKED`, `ENVIRONMENT_DOUBLE_BOOKED`, `SESSIONS_OVERLAP`, `ENVIRONMENT_MAINTENANCE` e `INSTRUCTOR_UNAVAILABLE`, y **bloquea la publicación** mientras haya conflictos activos (RN-SCH-03/04).
- **Publica horarios inmutables**: un horario `PUBLISHED` no se edita; cualquier cambio genera una nueva versión y archiva la anterior (RN-SCH-01), preservando trazabilidad.
- **Comunica en tiempo real**: al publicar, el evento `scheduling.schedule.published` propaga la novedad a seguimiento, notificación de actores y generación de documentos.
- **Aporta valor pedagógico**: `monitoring-service` (contexto CORE) calcula KPIs por ficha y dispara alertas de riesgo de deserción según umbrales reglamentarios (RN-MON-01 a RN-MON-06).

### Diferenciador

El valor no está en digitalizar un formulario, sino en la **lógica de detección de conflictos y de riesgo**. Por eso `scheduling-service` y `monitoring-service` son los subdominios CORE donde se concentra el esfuerzo de diseño y pruebas (ver [domain-map.md](../02-domain/domain-map.md)).

## 5. Alcance

### 5.1 En alcance — MVP

Cubre el flujo completo "construir → validar → publicar → consultar" más el seguimiento base. Detalle en [scope.md](../01-context/scope.md).

| Capacidad | Servicio(s) |
|-----------|-------------|
| Identidad y acceso (JWT, RBAC por feature + scope) | `iam-service` |
| Jerarquía institucional y catálogos del sistema | `reference-data-service` |
| Programas, competencias, RAPs y fichas | `academic-management-service` |
| Ambientes, tipos, disponibilidad, mantenimientos | `training-environment-service` |
| Motor de horarios: borrador, sesiones, conflictos, publicación | `scheduling-service` (CORE) |
| Instructores (competencias, disponibilidad), aprendices, empresas | `actors-service` |
| Generación de PDFs (constancias, horarios) en object storage | `document-service` |
| KPIs por ficha, alertas configurables, planes de mejoramiento | `monitoring-service` (CORE) |
| Entrega de notificaciones (email/IN_APP) por eventos, con Outbox y retry/DLQ | `notification-service` (transversal) |
| Log de auditoría append-only | `audit-service` |
| Consulta de horario por instructor y por aprendiz | `scheduling-service` (lectura) |

### 5.2 Fuera de alcance — MVP (futuro)

- **Sugerencia automática de asignación** de horario (scheduling con IA) y **resolución automática** de conflictos.
- **Integración con SOFIA Plus** (hoy: ingreso manual de número de ficha y código de programa).
- **Notificaciones push** móviles y **exportación** a calendarios externos (Google/Outlook).
- **Soporte offline** completo (PWA) y **app móvil nativa**.
- **Doble factor (2FA)** — planificado para V2.

## 6. Principios de producto

1. **Validar antes de publicar** — ningún horario con conflictos activos llega a los actores.
2. **Inmutabilidad y trazabilidad** — los registros oficiales no se editan; se versionan y se auditan.
3. **Foco en el CORE** — el esfuerzo se concentra en el motor de horarios y el seguimiento; el resto usa patrones estándar.
4. **Desacoplamiento** — el motor sigue operando aunque servicios de soporte estén caídos, gracias a read models por eventos (ADR-002).
5. **Desktop-first para operar, responsive para consultar** — el coordinador trabaja en escritorio; instructor y aprendiz consultan desde móvil.
6. **Cumplimiento normativo** — reglas del Acuerdo 00003/2012, Circular 1/2014 y Ley 1581/2012 (Habeas Data).

## 7. Métricas de éxito (línea cero → meta MVP)

| KPI | Situación actual | Meta MVP |
|-----|------------------|----------|
| Tiempo de creación de horario semanal | ~4–8 h (estimado) | < 1 h |
| Horarios publicados sin conflictos post-publicación | No medido | > 95 % |
| Instructores que conocen su horario con ≥ 48 h de anticipación | No medido | > 90 % |
| Utilización de ambientes en horas programadas | No medido | Medible desde el primer horario publicado |

## 8. Puntos abiertos

Heredados de discovery (ver [discovery-brief.md](./discovery-brief.md), preguntas abiertas), a resolver antes o durante Build:

- **PA-01** ¿El coordinador puede modificar un horario `PUBLISHED` directamente, o siempre debe crear una nueva versión? *(supuesto actual: siempre nueva versión — RN-SCH-01).*
- **PA-02** ¿Los instructores contratistas tienen límite de horas distinto al de planta? *(supuesto: sí — RN-ACT-01).*
- **PA-03** ¿Se requiere integración con SOFIA Plus en el MVP o en fases posteriores? *(supuesto: fases posteriores).*
- **PA-04** ¿Un aprendiz puede estar matriculado en más de una ficha activa a la vez?

## Referencias

- [discovery-brief.md](./discovery-brief.md) · [problem-framing.md](./problem-framing.md)
- [roadmap.md](./roadmap.md) · [product-backlog.md](./product-backlog.md)
- [01-context/overview.md](../01-context/overview.md) · [01-context/scope.md](../01-context/scope.md)
- [02-domain/domain-map.md](../02-domain/domain-map.md) · [02-domain/entities-and-rules.md](../02-domain/entities-and-rules.md)
- [04-requirements/functional.md](../04-requirements/functional.md) · [04-requirements/non-functional.md](../04-requirements/non-functional.md)
