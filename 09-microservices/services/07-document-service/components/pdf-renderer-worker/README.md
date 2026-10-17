# pdf-renderer-worker

> Estado: 🟡 En progreso | Última actualización: 2026-08-01
> Autor: Jesús Ariel González Bonilla (PM + Arquitecto) | Equipo: Arquitectura

> **Implementado** en `design-software-document-service` (`apps/pdf_renderer_worker`, HU-DOC-004). Este documento describe el worker tal como está implementado.

## Tipo de componente
`-worker` — consumidor de eventos.

## Responsabilidad
Renderizar documentos a PDF a partir de una plantilla y sus datos, almacenar el binario en object storage y emitir el evento de finalización. **Los binarios nunca se guardan en la BD** (solo metadatos/ruta), ver [ADR-003](../../../../../05-architecture/decisions/records/ADR-003-object-storage.md).

## Tecnologías
| Capa | Tecnología |
|------|-----------|
| Runtime | Agnóstico — cualquier backend (a definir) |
| Framework | A definir |
| Persistencia | PostgreSQL 16 · schema `document` (metadatos) |
| Binarios | Object storage S3-compatible (MinIO DEV/QA, S3 PROD) |
| Transporte | Broker AMQP (RabbitMQ, ADR-001) |
| Motor de PDF | HTML a PDF con `xhtml2pdf` (Python puro, sin dependencias del sistema); plantillas Handlebars |

## Variables de entorno

| Variable | Descripción |
|----------|-------------|
| `DB_URL` | Conexión a PostgreSQL |
| `AMQP_URL` | Conexión al broker AMQP |
| `GENERATION_QUEUE` | Cola de generación (`document-generation-queue`) |
| `DLQ_QUEUE` | Cola de mensajes fallidos (`pdf-renderer.dlq`) |
| `EVENTS_EXCHANGE` | Exchange de eventos de integración (`document-events`) |
| `STORAGE_ENDPOINT` / `STORAGE_BUCKET` / `STORAGE_ACCESS_KEY` / `STORAGE_SECRET_KEY` | Object storage S3-compatible (el bucket debe existir) |
| `RENDER_MAX_RETRIES` | Reintentos ante errores transitorios (3) |
| `RENDER_BACKOFF_SECONDS` | Espera inicial del backoff exponencial (1) |
| `METRICS_PORT` | Puerto de las métricas Prometheus (9108) |
| `LOG_LEVEL` | Nivel de los logs JSON (`INFO`) |

## Idempotencia

Deduplica por `correlation_id` (`document_version.correlation_id`, único por documento): un mensaje repetido no renderiza ni versiona de nuevo. DLQ `pdf-renderer.dlq` ante fallo persistente. Ver el [contrato](./contract.md).

## Contrato
Ver [contract.md](./contract.md) · [storage-adapters.md](../../storage-adapters.md)
