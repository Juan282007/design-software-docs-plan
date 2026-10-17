# Decisiones internas — document-service

estado 🟢 Estable — 2026-06-20

> Para decisiones que afectan más de un servicio → crear ADR en `05-architecture/decisions/records/`
> Ver ADR-003 (estrategia de almacenamiento de documentos)

| # | Decisión | Alternativas descartadas | Motivo | Fecha |
|---|----------|--------------------------|--------|-------|
| 01 | Binarios en object storage únicamente; `document_db` almacena solo metadatos | Guardar archivos en BD como `bytea` o `BLOB` | Object storage es más barato, más eficiente para archivos grandes y habilita pre-signed URLs y distribución CDN; los binarios en BD inflan los backups, degradan el rendimiento de las queries sobre metadatos y no permiten streaming parcial | 2026-06-17 |
| 02 | Generación de PDF es asíncrona mediante `pdf-renderer-worker` + cola | Generación síncrona en el endpoint REST | El renderizado de PDF (HTML→PDF via headless browser o librería) puede superar 2 segundos; una respuesta síncrona bloquearía la conexión del cliente y excedería timeouts de API gateway; el patrón worker + cola permite reintentos, backpressure y paralelismo controlado | 2026-06-17 |
| 03 | `document_version` para historial; la versión actual es `document.storage_key` (sin `current_version_id` separado) | Mantener un campo `current_version_id` con FK a `document_version` | Un `current_version_id` requeriría un JOIN adicional en cada lectura de documento; `document.storage_key` apunta siempre a la versión activa directamente; el historial completo sigue disponible en `document_version` para quien lo necesite | 2026-06-20 |
| 04 | Sistema de plantillas HTML/Handlebars; no PDFs hardcodeados | Plantillas hardcodeadas como archivos PDF estáticos o código Java/Node que genera el layout | Las plantillas HTML/Handlebars permiten que usuarios no técnicos editen el contenido y la estructura desde la interfaz de administración; los PDFs estáticos no son parametrizables; el código de layout mezclado con la lógica de negocio hace que cada cambio de formato requiera un deploy | 2026-06-20 |
| 05 | Dos proveedores de storage (MinIO en desarrollo, S3 en producción) mediante patrón adapter (ADR-003) | Un único proveedor en todos los entornos | MinIO permite desarrollo y pruebas sin costo ni acceso a AWS; el adapter pattern garantiza que el código de negocio no dependa del proveedor concreto; cambiar de proveedor o agregar uno nuevo (Azure Blob, GCS) requiere solo implementar la interfaz, no modificar la lógica de servicio | 2026-06-20 |
