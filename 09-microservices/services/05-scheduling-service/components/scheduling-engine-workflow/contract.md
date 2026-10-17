<!-- RESUMEN-EJECUTIVO
agente: Jesús Ariel González Bonilla (PM + Arquitecto)
capacidad: contrato de evento / mensajería async
fase: diseño
estado: accepted
dependencias_entrada: 09-microservices/event-catalog.md, la BD de scheduling-service (read models CQRS-lite)
consumidores_siguientes: conflict-validator-worker, audit-service, monitoring-service, actors-service, document-service
tldr: Saga que orquesta la generación de horario por ficha (insumos → asignación → persistencia → validación de conflictos → publicación) con compensación por paso.
decisiones_clave: Read models locales sincronizados por eventos (ADR-002); idempotencia por (ficha_id, period) vía índice único parcial; DLQ scheduling-engine.dlq.
halts_registrados: academic.enrollment_ficha.created, actors.instructor.availability_changed y scheduling.class_session.updated no están en event-catalog.md (pendiente reconciliación)
-->

# Contrato — scheduling-engine-workflow

> Estado: 🟡 En progreso | Última actualización: 2026-08-06
> Autor: Jesús Ariel González Bonilla (PM + Arquitecto) | Equipo: Arquitectura

> **Fuente de verdad:** el inventario de eventos lo gobierna [`event-catalog.md`](../../../../event-catalog.md) y el envelope estándar [`_template/service/events.md`](../../../../_template/service/events.md). Ante discrepancia, esos documentos prevalecen sobre este contrato.

> **Diseño previsto — no implementado.** Orquestación (saga) a nivel de protocolo, agnóstica de lenguaje.

## Propósito
Orquestar la generación de un horario para una ficha: recolectar insumos, asignar sesiones a franjas/ambientes/instructores sin solape, y publicar el resultado. Mantiene read models locales (CQRS-lite, [ADR-002](../../../../../05-architecture/decisions/records/ADR-002-scheduling-read-models.md)).

## Disparador
Comando `POST /api/v1/schedules/{ficha_id}/generate` (schedules-api) o evento `academic.enrollment_ficha.created`.

## Pasos de la saga (con compensación)
| # | Paso | Compensación |
|---|------|--------------|
| 1 | Cargar insumos (programa/competencias, disponibilidad de instructores y ambientes desde read models) | — (solo lectura) |
| 2 | Calcular asignación de `class_session` sin solape | — |
| 3 | Persistir `schedule` + `class_session` (estado `DRAFT`) | `DELETE`/marcar `CANCELLED` |
| 4 | Solicitar validación de conflictos (evento a `conflict-validator-worker`) | — |
| 5 | Si no hay conflictos, publicar `scheduling.schedule.published`; si hay, dejar en `DRAFT` con conflictos | revertir a `DRAFT` |

## Read models (ADR-002)
Se sincronizan al recibir `environment.availability.changed`, `environment.maintenance.started`, `actors.instructor.availability_changed`. Reducen las dependencias síncronas a ≤ 2.

## Eventos consumidos
| Evento | Origen | Acción |
|--------|--------|--------|
| `academic.enrollment_ficha.created` | academic-management-service | Dispara la saga (alternativa al comando síncrono) |
| `environment.availability.changed` | training-environment-service | Sincroniza read model de disponibilidad de ambientes |
| `environment.maintenance.started` | training-environment-service | Sincroniza read model de disponibilidad de ambientes |
| `actors.instructor.availability_changed` | actors-service | Sincroniza read model de disponibilidad de instructores |

## Eventos producidos
| Evento | Descripción |
|--------|-------------|
| `scheduling.schedule.published` | Horario sin conflictos publicado a los actores |
| `scheduling.class_session.created` | Sesión de clase asignada durante la saga |
| `scheduling.class_session.updated` | Sesión de clase reasignada tras revalidación |

## Idempotencia
La generación es idempotente por `(ficha_id, period)`; una re-ejecución no duplica el horario publicado (índice único parcial `uq_schedule_ficha_id_period_published`).

## Política de reintentos
Fallo en un paso de la saga → reintento con backoff antes de compensar; si el paso sigue fallando tras agotar reintentos, se ejecuta la compensación de la tabla anterior y el evento que originó el paso se envía a **DLQ** `scheduling-engine.dlq`.
