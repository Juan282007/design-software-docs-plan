# audit-service

> Estado: 🟡 Revisión | Última actualización: 2026-10-07
> Autor: Kevin Andrey Culma (Arquitecto) | Equipo: Arquitectura.

| Campo | Valor |
|---|---|
| Módulo | Transversal |
| App repo | `design-software-audit-service` |
| DB repo | `design-software-audit-db` |
| Stack | Go |
| Deployables | audit-query-api, audit-worker |
| Estado observado | Esqueleto Go con query API de health y worker pendiente. |

## Responsabilidad

Registro append-only de auditoría y consulta de eventos auditables.

## Persistencia

- Esquemas: audit
- Tablas: audit.audit_event_seen, audit.audit_record, audit.audit_record_2026_08, audit.audit_record_default

## API detectada

| Metodo | Ruta | Fuente |
|---|---|---|
| GET | `/health` | `.\design-software-audit-service\cmd\audit-query-api\main.go` |

## Configuración

- `PORT`

## Notas

Debe consumir eventos por AMQP y persistir auditoría idempotente.
