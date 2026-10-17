# DB Design — <PROJECT_KEY>

> **PLANTILLA** — Copiar como `db-design.md` y completar.
> Eliminar esta línea antes de hacer commit.

> Estado: 🔴 Pendiente | Última actualización: YYYY-MM-DD
> Autor: Por definir | Equipo: Arquitectura de Datos + DBA

## Motor de base de datos

| Campo | Valor |
|-------|-------|
| Motor | [PostgreSQL / MySQL / MongoDB / otro] |
| Versión | |
| Modo | [Standalone / Replica Set / Cluster] |
| Justificación | Ver [ADR-NNN](../05-architecture/decisions/records/) |

## Esquema físico

### Tabla: `[nombre_tabla]`

| Columna | Tipo físico | Nullable | Default | Constraint | PII | Notas |
|---------|-------------|----------|---------|------------|-----|-------|
| id | UUID / BIGSERIAL | No | gen_random_uuid() | PK | No | |
| created_at | TIMESTAMPTZ | No | NOW() | NOT NULL | No | UTC |
| updated_at | TIMESTAMPTZ | No | NOW() | NOT NULL | No | auto-update trigger |

### Índices

| Nombre | Tabla | Columnas | Tipo | Propósito |
|--------|-------|----------|------|-----------|
| idx_[tabla]_[col] | [tabla] | [col] | BTREE | Búsqueda por [campo] |

### Foreign keys

| FK | Tabla origen | Columna | Tabla destino | ON DELETE | ON UPDATE |
|----|-------------|---------|---------------|-----------|-----------|
| fk_[nombre] | | | | RESTRICT | CASCADE |

## Estrategia de migraciones

| Campo | Decisión |
|-------|----------|
| Herramienta | [Liquibase / Flyway / Alembic / otro] |
| Convención de naming | `V{timestamp}__{descripcion}.sql` |
| Rollback | [automático / manual — ver rollback-plan] |
| Ambiente de validación | [staging antes de prod] |

## RTO / RPO

| Métrica | Objetivo | Mecanismo |
|---------|----------|-----------|
| RTO | [30 min] | Failover a réplica |
| RPO | [5 min] | Backups continuos |
| Backups | [cada N horas] | [herramienta / S3 / otro] |

## Cifrado y seguridad

| Dato | Mecanismo | Columnas afectadas |
|------|-----------|-------------------|
| PII en reposo | AES-256 / cifrado de columna | [email, documento] |
| Conexión | TLS obligatorio | Todas |
| Secretos de conexión | [Vault / SSM / env cifrado] | — |

## Referencias

- [Data Model](./data-model.md)
- [Architecture](../05-architecture/architecture.md)
- [Migration Plan](./data-migration-plan.md)
