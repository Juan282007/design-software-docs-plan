<!-- RESUMEN-EJECUTIVO
agente: Jesús Ariel González Bonilla (PM + Arquitecto)
capacidad: contrato de evento / mensajería async
fase: diseño
estado: accepted
dependencias_entrada: 09-microservices/event-catalog.md, la BD de document-service
consumidores_siguientes: audit-service (pendiente reconciliación de eventos producidos, ver halts)
tldr: Worker que gestiona el ciclo de vida del documento (DRAFT → EMITIDO → ARCHIVADO) según renders completados y política de retención.
decisiones_clave: Borrado lógico (soft delete); transición hacia un estado ya alcanzado es no-op; DLQ document-lifecycle.dlq.
halts_registrados: document.render.completed y document.lifecycle.tick existen en event-catalog.md como eventos internos, y document.status.changed y document.archived como eventos publicados; pendiente reconciliar los estados de este contrato (DRAFT, EMITIDO, ARCHIVADO) con los del modelo de datos
-->

# Contrato — document-lifecycle-worker

> Estado: 🟡 En progreso | Última actualización: 2026-08-06
> Autor: Jesús Ariel González Bonilla (PM + Arquitecto) | Equipo: Arquitectura

> **Fuente de verdad:** el inventario de eventos lo gobierna [`event-catalog.md`](../../../../event-catalog.md) y el envelope estándar [`_template/service/events.md`](../../../../_template/service/events.md). Ante discrepancia, esos documentos prevalecen sobre este contrato.

> **Diseño previsto — no implementado.** Contrato de mensajería, agnóstico de lenguaje.

## Eventos consumidos
| Evento | Origen | Acción |
|--------|--------|--------|
| `document.render.completed` | pdf-renderer-worker | Marcar documento como `EMITIDO` |
| `document.lifecycle.tick` | scheduler | Evaluar retención/expiración periódica |

## Eventos producidos
| Evento | Descripción |
|--------|-------------|
| `document.status.changed` | El documento cambió de estado |
| `document.archived` | Documento archivado por política de retención |

## Transiciones de estado (documento)
`DRAFT → EMITIDO → ARCHIVADO`. El borrado es lógico (soft delete); el binario en object storage se marca para expiración según `RETENTION_POLICY`.

## Envelope (estándar)
```json
{
  "event_id": "uuid-v4",
  "event_type": "document.status.changed",
  "version": "1.0",
  "timestamp": "2026-08-01T14:00:00Z",
  "source_service": "document-service",
  "correlation_id": "uuid-v4",
  "payload": { "document_id": "uuid", "from_status": "DRAFT", "to_status": "EMITIDO" }
}
```

## Idempotencia
`event_id` único; una transición hacia un estado ya alcanzado es no-op.

## Política de reintentos
Fallo persistente → reintento con backoff; agotados los reintentos → **DLQ** `document-lifecycle.dlq`.
