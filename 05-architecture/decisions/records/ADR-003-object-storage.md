# ADR-003: Estrategia de almacenamiento de objetos para document-service

**Estado:** PROPOSED
**Fecha:** 2026-06-17
**Autores:** Jesús Ariel González Bonilla (Arquitecto)
**Equipos involucrados:** Arquitectura, DevOps, Backend (document-service)

---

## Contexto

`document-service` genera y almacena archivos binarios (PDFs de constancias, horarios exportados, plantillas). El [data-model.md](../../../09-microservices/services/07-document-service/data-model.md) establece la invariante: **los binarios nunca se almacenan en la base de datos**; solo se guarda la `storage_key` que apunta al archivo en un object storage.

Requisitos:
- Almacenamiento de objetos compatible con API S3 (estándar de facto)
- URLs firmadas con expiración para descarga segura (amenaza I-04 del threat model)
- Funcionar en local (DEV) sin depender de la nube
- Coherencia entre ambientes DEV → QA → PROD
- Costo controlado (proyecto académico/institucional)

Hay que decidir qué tecnología de object storage usar y cómo mantener coherencia entre el desarrollo local y producción.

## Decisión

Se decide usar **MinIO en DEV/QA** y **almacenamiento compatible S3 en PROD**, accedidos a través de un **adaptador único con interfaz S3**.

El `document-service` usa un puerto `StorageAdapter` (capa hexagonal) con dos implementaciones:
- `MinioStorageAdapter` para DEV y QA (MinIO self-hosted en contenedor)
- `S3StorageAdapter` para PROD (AWS S3 o el object storage compatible S3 del proveedor cloud elegido)

Ambas hablan el mismo protocolo S3, por lo que el código de la aplicación es idéntico; solo cambia la configuración (endpoint, credenciales) por ambiente. Ver [storage-adapters.md](../../../09-microservices/services/07-document-service/storage-adapters.md).

Las descargas se sirven mediante **URLs firmadas con expiración de 5 minutos**, generadas solo después de verificar el scope del usuario.

## Consecuencias

### Positivas

- Código idéntico en todos los ambientes (mismo protocolo S3) — sin ramas por ambiente
- DEV funciona 100% local con MinIO en contenedor, sin costo ni dependencia de red
- Portabilidad de nube: el `S3StorageAdapter` funciona con AWS S3, Google Cloud Storage (modo compatibilidad S3), DigitalOcean Spaces, etc.
- URLs firmadas de corta expiración mitigan la fuga de enlaces (amenaza I-04)
- La invariante "binarios fuera de la BD" mantiene `document_db` ligera y los backups rápidos

### Negativas / Trade-offs

- Hay que operar MinIO en DEV/QA (un componente más en el docker-compose local)
- Las URLs firmadas requieren que el cliente maneje la expiración (re-solicitar si caduca)
- La paridad DEV/PROD no es perfecta: MinIO no replica todas las features avanzadas de S3 (lifecycle policies, replicación cross-region), pero ninguna se usa en el MVP

### Riesgos

- **Divergencia de comportamiento MinIO vs S3** en casos borde. Mitigación: usar solo el subconjunto común del API S3 (PUT, GET, DELETE, presigned URLs); pruebas de integración contra MinIO en CI
- **Gestión de credenciales del storage**. Mitigación: credenciales en Secret Manager (consistente con el resto del sistema, ver threat model)

## Alternativas consideradas

| Alternativa | Por qué se descartó |
|-------------|---------------------|
| Solo AWS S3 (también en DEV) | Obliga a tener credenciales AWS y conectividad en desarrollo local; costo y fricción para el equipo |
| Almacenar binarios en PostgreSQL (BYTEA) | Viola la invariante del data-model; infla la BD, ralentiza backups, degrada el rendimiento |
| Sistema de archivos local compartido (NFS) | No escala horizontalmente; sin URLs firmadas nativas; complica el despliegue multi-instancia |
| Almacenamiento en el filesystem del contenedor | Se pierde al reiniciar el contenedor; no apto para producción ni para múltiples réplicas |

## Referencias

- [07-document-service/data-model.md](../../../09-microservices/services/07-document-service/data-model.md)
- [07-document-service/storage-adapters.md](../../../09-microservices/services/07-document-service/storage-adapters.md)
- [storage-and-documents.md](../../../09-microservices/storage-and-documents.md)
- [security-threat-model.md](../../security-threat-model.md) — amenaza I-04
