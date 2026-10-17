# Adaptadores de almacenamiento — document-service

> Estado: 🟡 En progreso | Última actualización: 2026-08-01
> Autor: Jesús Ariel González Bonilla (PM + Arquitecto) | Equipo: Arquitectura

> **Parcialmente implementado** en `design-software-document-service`: las operaciones `put` (pdf-renderer-worker) y `signed_url` (document-api) ya existen; `get` y `delete` siguen
> como diseño previsto. El adaptador se describe como **contrato de operaciones** (interfaz), no como código de ningún lenguaje. Es válido para cualquier
> backend que hable el protocolo S3.

## Estrategia

`document-service` abstrae el almacenamiento físico de binarios (PDF, Excel, Word, plantillas,
evidencias) detrás de un **adaptador de storage** con interfaz S3. El código de negocio no
depende del proveedor concreto: solo cambia la configuración por ambiente.

- **DEV / QA:** MinIO (self-hosted en contenedor, sin costo ni dependencia de nube).
- **PROD:** object storage compatible S3 (AWS S3, GCS en modo S3, DigitalOcean Spaces, etc.).

Ambos ambientes hablan el mismo subconjunto del protocolo S3, por lo que el diseño es idéntico;
solo varían endpoint y credenciales. Ver [ADR-003](../../../05-architecture/decisions/records/ADR-003-object-storage.md) y la decisión 05 de [decisions.md](./decisions.md).

## Invariante fundamental

Los binarios **nunca** se almacenan en `document_db`. La base de datos guarda únicamente
metadatos y la `storage_key` que apunta al objeto en el storage. Ver [data-model.md](./data-model.md)
(entidad `document`, campo `storage_key`) y la decisión 01 de [decisions.md](./decisions.md).

## Interfaz del adaptador (agnóstica)

El adaptador expone cuatro operaciones. Los nombres son ilustrativos del **contrato**, no de
una API de lenguaje concreta.

| Operación | Entradas | Salida | Descripción |
|-----------|----------|--------|-------------|
| `put` | `key`, `content` (stream/bytes), `content_type` | `storage_key` | Sube el binario y retorna la clave definitiva. |
| `get` | `key` | stream de contenido | Descarga el binario (uso interno del servicio). |
| `signed_url` | `key`, `expires_in` | URL temporal | Genera una URL firmada de descarga con expiración. |
| `delete` | `key` | — | Elimina el objeto (solo archivado/limpieza por el `document-lifecycle-worker`). |

> El acceso del cliente final al binario **siempre** es vía `signed_url`; el servicio nunca
> transmite el binario a través de la API REST ni expone la `storage_key`.

## Implementaciones previstas

| Adaptador | Ambiente | Configuración (variables de entorno) |
|-----------|----------|--------------------------------------|
| MinIO (S3-compatible) | Local / DEV / QA | `STORAGE_ENDPOINT`, `STORAGE_ACCESS_KEY`, `STORAGE_SECRET_KEY`, `STORAGE_BUCKET` |
| S3 | PROD | `AWS_REGION`, `S3_BUCKET`, `AWS_ACCESS_KEY_ID`, `AWS_SECRET_ACCESS_KEY` |

> Las credenciales viven en el Secret Manager de la plataforma; nunca en el repositorio.

## Convención de claves (`storage_key`)

```
<domain>/<año>/<mes>/<owner_entity_id>/<uuid>.<ext>

Ejemplo: SCHEDULE/2026/06/ficha-abc123/8f3c...-v1.pdf
```

- `domain` es uno de los valores permitidos de `document.domain`: `SCHEDULE`, `FICHA`,
  `CERTIFICATE`, `ACTOR`, `REPORT` (ver [data-model.md](./data-model.md)).
- La versión vigente se referencia desde `document.storage_key`; el historial completo vive
  en `document_version.storage_key` (decisión 03 de [decisions.md](./decisions.md)).

## Notas de seguridad

- Los buckets son **privados**; nunca de acceso público.
- El acceso a archivos es siempre vía URL firmada con expiración corta. [ADR-003](../../../05-architecture/decisions/records/ADR-003-object-storage.md)
  fija **5 minutos** para descargas (mitiga la fuga de enlaces, amenaza I-04 del threat model).
- La URL firmada se genera **solo después** de verificar el scope/rol del usuario contra `iam-service`.
- Nunca se expone `storage_key` al frontend: el cliente usa el `document_id` y el backend
  produce la URL.
