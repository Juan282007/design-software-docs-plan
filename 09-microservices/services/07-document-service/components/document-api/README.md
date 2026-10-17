# document-api

> Estado: 🟡 En progreso | Última actualización: 2026-08-01
> Autor: Jesús Ariel González Bonilla (PM + Arquitecto) | Equipo: Arquitectura

> **Parcialmente implementado** en `design-software-document-service` (HU-DOC-002 y HU-DOC-003): `POST /documents/generate`, `GET /documents`, `GET /documents/{id}`, versiones y `download-url`. Siguen como diseño previsto `POST /documents`, `PATCH` y `DELETE` de documentos, la carga de versiones y `/reports/document-status`. Contrato a nivel de protocolo (REST/JSON).

## Tipo de componente

`-api` — REST API sincrónica

## Responsabilidad

Expone la API REST sobre las entidades `document` y `document_version`: consulta de metadatos,
listado por entidad de negocio, historial de versiones, solicitud de **generación asíncrona**
y entrega de la **URL firmada** de descarga. No renderiza documentos: la generación la ejecuta
el [pdf-renderer-worker](../pdf-renderer-worker/README.md) tras encolar la solicitud. Los
binarios se sirven exclusivamente vía URL firmada, nunca a través de este servicio.

## Tecnologías

| Capa | Tecnología |
|------|-----------|
| Runtime | Agnóstico — cualquier backend (a definir) |
| Framework | A definir |
| Base de datos | PostgreSQL 16 (`document_db`) |
| Object storage | S3-compatible / MinIO ([ADR-003](../../../../../05-architecture/decisions/records/ADR-003-object-storage.md)) |
| Broker | AMQP — RabbitMQ, para encolar en `document-generation-queue` ([ADR-001](../../../../../05-architecture/decisions/records/ADR-001-message-broker.md)) |

## Variables de entorno requeridas

| Variable | Descripción | Ejemplo |
|----------|-------------|---------|
| `SERVICE_PORT` | Puerto de escucha | `8007` |
| `DB_URL` | Conexión a `document_db` (PostgreSQL 16) | `postgresql://user:pass@host:5432/document_db` |
| `IAM_JWKS_URL` | Endpoint de claves públicas de `iam-service` para validar JWT | `https://iam/.well-known/jwks.json` |
| `AMQP_URL` | Conexión al broker RabbitMQ | `amqp://user:pass@host:5672/` |
| `GENERATION_QUEUE` | Nombre de la cola de render | `document-generation-queue` |
| `STORAGE_ENDPOINT` | Endpoint del object storage (MinIO/S3) | `https://minio:9000` |
| `STORAGE_BUCKET` | Bucket de binarios | `documents` |
| `SIGNED_URL_TTL_SECONDS` | Expiración de la URL firmada de descarga | `300` |

## Contrato

Ver [contract.md](./contract.md)
