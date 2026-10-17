# template-api

> Estado: 🟡 En progreso | Última actualización: 2026-08-01
> Autor: Jesús Ariel González Bonilla (PM + Arquitecto) | Equipo: Arquitectura

> **Implementado** en `design-software-document-service` (HU-DOC-001). Contrato a nivel de protocolo (REST/JSON).

## Tipo de componente

`-api` — REST API sincrónica

## Responsabilidad

Administra las plantillas de documentos (entidad `document_template`): CRUD del cuerpo HTML,
tipo de salida (`output_type`: PDF/EXCEL/WORD) y versión. Las plantillas son consumidas por el
[pdf-renderer-worker](../pdf-renderer-worker/README.md) al generar el documento final. Permite
que usuarios no técnicos editen contenido y estructura sin desplegar (decisión 04 de
[decisions.md](../../decisions.md)).

> El motor de plantillas concreto (sustitución de variables sobre el cuerpo HTML) queda **a
> definir** junto con el backend. El contrato solo fija que el cuerpo es HTML parametrizable.

## Tecnologías

| Capa | Tecnología |
|------|-----------|
| Runtime | Agnóstico — cualquier backend (a definir) |
| Framework | A definir |
| Base de datos | PostgreSQL 16 (`document_db`) |

## Variables de entorno requeridas

| Variable | Descripción | Ejemplo |
|----------|-------------|---------|
| `SERVICE_PORT` | Puerto de escucha (sub-ruta `/templates`) | `8007` |
| `DB_URL` | Conexión a `document_db` (PostgreSQL 16) | `postgresql://user:pass@host:5432/document_db` |
| `IAM_JWKS_URL` | Endpoint de claves públicas de `iam-service` para validar JWT | `https://iam/.well-known/jwks.json` |

## Contrato

Ver [contract.md](./contract.md)
