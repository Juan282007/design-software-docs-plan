<!-- RESUMEN-EJECUTIVO
agente: Jesús Ariel González Bonilla (PM + Arquitecto)
capacidad: contrato de evento / mensajería async
fase: diseño
estado: accepted
dependencias_entrada: 09-microservices/event-catalog.md, la BD de scheduling-service
consumidores_siguientes: scheduling-engine-workflow, audit-service, monitoring-service
tldr: Worker que revalida solapes de horario ante cambios de sesión o disponibilidad y publica/resuelve conflictos de instructor, ambiente o ficha.
decisiones_clave: DLQ dedicado conflict-validator.dlq con retry+backoff; idempotencia por event_id; envelope alineado al estándar del ecosistema.
halts_registrados: scheduling.class_session.updated, actors.instructor.availability_changed y scheduling.conflict.resolved no están en event-catalog.md (pendiente reconciliación)
-->

# Contrato — conflict-validator-worker

> Estado: 🟡 En progreso | Última actualización: 2026-08-06
> Autor: Jesús Ariel González Bonilla (PM + Arquitecto) | Equipo: Arquitectura

> **Fuente de verdad:** el inventario de eventos lo gobierna [`event-catalog.md`](../../../../event-catalog.md) y el envelope estándar [`_template/service/events.md`](../../../../_template/service/events.md). Ante discrepancia, esos documentos prevalecen sobre este contrato.

> **Diseño previsto — no implementado.** Contrato de mensajería, agnóstico de lenguaje.

## Eventos consumidos
| Evento | Origen | Acción |
|--------|--------|--------|
| `scheduling.class_session.created` | scheduling-engine | Revalidar solapes de la sesión |
| `scheduling.class_session.updated` | scheduling-engine | Revalidar |
| `environment.availability.changed` | training-environment | Recalcular conflictos de ambiente |
| `actors.instructor.availability_changed` | actors | Recalcular conflictos de instructor |

## Eventos producidos
| Evento | Descripción |
|--------|-------------|
| `scheduling.conflict.detected` | Se registró un `scheduling_conflict` (tipo: instructor/ambiente/ficha) |
| `scheduling.conflict.resolved` | Un conflicto previamente detectado ya no aplica |

## Envelope (estándar)
```json
{
  "event_id": "uuid-v4",
  "event_type": "scheduling.conflict.detected",
  "version": "1.0",
  "timestamp": "2026-08-01T14:00:00Z",
  "source_service": "scheduling-service",
  "correlation_id": "uuid-v4",
  "payload": { "conflict_id": "uuid", "session_a_id": "uuid", "session_b_id": "uuid", "type": "INSTRUCTOR" }
}
```

## Idempotencia
`event_id` único por evento; un evento reprocesado no crea un `scheduling_conflict` duplicado.

## Política de reintentos
Fallo de procesamiento → reintento con backoff; agotados los reintentos → **DLQ** `conflict-validator.dlq`.
