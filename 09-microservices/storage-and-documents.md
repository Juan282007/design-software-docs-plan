# Almacenamiento y documentos

> Estado: 🟡 En progreso | Última actualización: 2026-08-01
> Autor: Jesús Ariel González Bonilla (PM + Arquitecto) | Equipo: Arquitectura

Estrategia de almacenamiento del sistema. Agnóstica de lenguaje.

## Tres tipos de almacenamiento

| Tipo | Tecnología | Qué guarda | Regla |
|------|-----------|------------|-------|
| **Relacional** | PostgreSQL 16 (un schema por servicio) | Datos estructurados y metadatos | Fuente de verdad transaccional; una BD por servicio |
| **Object storage** | S3-compatible (MinIO DEV/QA, S3 PROD) | Binarios: PDFs generados por `document-service` | **Binarios nunca en la BD**; solo metadatos + `storage_key` ([ADR-003](../05-architecture/decisions/records/ADR-003-object-storage.md)) |
| **Caché** (previsto) | Redis | Consultas de alta frecuencia (catálogos, disponibilidad) | Por consumidor; invalidación por evento |

## Documentos (document-service)
- La BD guarda `document`, `document_template`, `document_version` (metadatos, estado, versión, ruta).
- El binario PDF vive en object storage bajo una `storage_key` determinista; se sirve por **URL firmada** temporal, no por proxy del servicio.
- Ver adaptador en [services/07-document-service/storage-adapters.md](./services/07-document-service/storage-adapters.md).

## Principios
- **Portabilidad:** el acceso a object storage se hace por un adaptador S3-compatible único, de modo que el código sea idéntico entre MinIO (DEV/QA) y S3 (PROD).
- **Retención:** los binarios siguen la política de retención del `document-lifecycle-worker`.
- **Auditoría:** los eventos de creación/cambio de documento se auditan vía `audit-service` (append-only).
