<!-- RESUMEN-EJECUTIVO
agente: Jesús Ariel González Bonilla (PM + Arquitecto)
capacidad: contrato de evento / mensajería async
fase: diseño
estado: accepted
dependencias_entrada: 09-microservices/event-catalog.md, la BD de document-service (document_version)
consumidores_siguientes: document-lifecycle-worker (pendiente reconciliación de eventos producidos, ver halts)
tldr: Worker que consume document-generation-queue, renderiza el PDF, lo sube a object storage, registra la versión y publica document.document.generated y document.version.created.
decisiones_clave: storage_key determinista; idempotencia por correlation_id; reintentos con backoff y DLQ pdf-renderer.dlq.
halts_registrados: ninguno. document.render.requested/completed/failed existen en event-catalog.md como eventos internos del servicio; la implementación de HU-DOC-004 no los usa
-->

# Contrato — pdf-renderer-worker

> Estado: 🟡 En progreso | Última actualización: 2026-08-06
> Autor: Jesús Ariel González Bonilla (PM + Arquitecto) | Equipo: Arquitectura

> **Fuente de verdad:** el inventario de eventos lo gobierna [`event-catalog.md`](../../../../event-catalog.md) y el envelope estándar [`_template/service/events.md`](../../../../_template/service/events.md). Ante discrepancia, esos documentos prevalecen sobre este contrato.

> **Implementado (HU-DOC-004)** en `design-software-document-service`. Contrato de mensajería. Lo que esa HU no usa, los eventos internos `document.render.*`, se indica más abajo.

## Mensaje consumido

Cola `document-generation-queue` (AMQP/RabbitMQ), publicada por `POST /documents/generate` de [document-api](../document-api/contract.md). JSON persistente:

```json
{
  "document_id": "8f3c0000-0000-0000-0000-000000000001",
  "template_code": "SCHEDULE_CERTIFICATE",
  "data_payload": { "learner_name": "Ana" },
  "correlation_id": "corr-5b0f6f0e"
}
```

- `data_payload` es opcional (por defecto `{}`). `correlation_id` identifica la solicitud y es la clave de idempotencia.
- Un mensaje para un documento que ya está `AVAILABLE` se interpreta como regeneración y crea la versión siguiente.

## Eventos producidos

Topic `document-events` (exchange de tipo topic; la routing key es el tipo del evento). Sus payloads están en [events.md](../../events.md).

| Evento | Cuándo | Consumidores |
|--------|--------|--------------|
| `document.document.generated` | Se genera la primera versión del documento | `audit-service` |
| `document.version.created` | Se crea cualquier versión (incluida la primera) | `audit-service` |

> Los eventos internos `document.render.requested/completed/failed` **sí existen** en [`event-catalog.md`](../../../../event-catalog.md) (sección "Eventos internos de servicio"), pero quedan fuera del flujo implementado en HU-DOC-004: el worker consume la cola de generación directamente y publica los dos eventos de integración anteriores.

## Envelope (estándar)

```json
{
  "event_id": "uuid",
  "event_type": "document.version.created",
  "version": "1.0",
  "timestamp": "2026-08-01T14:00:00Z",
  "source_service": "document-service",
  "correlation_id": "corr-5b0f6f0e",
  "payload": { "version_id": "uuid", "document_id": "uuid", "version_number": 2, "storage_key": "SCHEDULE/2026/08/<owner_entity_id>/<document_id>-v2.pdf", "created_by": "pdf-renderer-worker", "created_at": "2026-08-01T14:00:00Z" }
}
```

`event_id` es determinista (derivado del `id` de la versión y del tipo): un evento republicado conserva su identidad y los consumidores deben deduplicar por `event_id`.

## Efectos (por mensaje)

1. Resuelve la plantilla **activa** por `template_code` y renderiza el HTML (Handlebars) con `data_payload`.
2. Convierte el HTML a PDF y sube el binario al bucket con la clave `<domain>/<año>/<mes>/<owner_entity_id>/<document_id>-v<n>.pdf` ([storage-adapters.md](../../storage-adapters.md)).
3. En una sola transacción registra el `document_version` (con su `correlation_id`) y actualiza `document` (`storage_key`, `mime_type`, `size_bytes`, `status = AVAILABLE`, `row_version`), con el actor de sistema `00000000-0000-0000-0000-000000000000`.
4. Publica los eventos de salida.
5. Confirma (`ack`) el mensaje solo cuando quedó resuelto; si no puede resolverlo lo devuelve a la cola.

## Idempotencia

`document_version.correlation_id` es único por documento. Un mensaje repetido no renderiza ni crea otra versión; republica los eventos con el mismo `event_id`.

## Política de reintentos y DLQ

- Errores transitorios (motor de PDF, object storage, broker, base de datos): hasta `RENDER_MAX_RETRIES` reintentos (por defecto 3) con espera exponencial desde `RENDER_BACKOFF_SECONDS` (por defecto 1 s, tope 30 s). La espera mantiene viva la conexión con el broker.
- Fallo definitivo: un documento que esperaba su primer archivo queda `GENERATION_FAILED` (una regeneración fallida conserva la versión vigente), el mensaje original se envía a la **DLQ** `pdf-renderer.dlq` con la cabecera `x-failure-reason` y no se publica `document.document.generated`.
- Sin reintento (DLQ directa): mensaje inválido, documento inexistente o archivado/expirado, plantilla inexistente o inactiva, `output_type` distinto de PDF y plantilla que no produce contenido.

## Seguridad del render

La plantilla no puede cargar nada fuera del propio documento: el motor de PDF aplica una política estricta de recursos (sin acceso a la red para evitar SSRF, sin lectura de archivos locales y con límites de tamaño y de tiempo). Solo se admiten recursos incrustados (`data:`); cualquier otro se omite del PDF y se registra en los logs.

## Observabilidad

Métricas Prometheus en `:METRICS_PORT/metrics`: `document_render_messages_total{outcome}` (`success`, `duplicate`, `failed`), `document_render_duration_seconds`, `document_render_retries_total`, `document_render_dead_lettered_total` y `document_generation_queue_depth` (backlog). Logs JSON con `document_id`, `correlation_id`, `outcome` y `reason`.
