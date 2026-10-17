# Onboarding Técnico

> Estado: 🟡 En progreso | Última actualización: 2026-08-01
> Autor: Jesús Ariel González Bonilla (PM + Arquitecto) | Equipo: Formación

Guía de entrada para nuevos integrantes técnicos del proyecto **SENA — Gestión de Horarios**. Al terminar deberías poder clonar los repos, levantar la base de datos en local y aplicar las migraciones de cada módulo.

> **Estado real del proyecto:** hoy está construida **únicamente la capa de datos** (repositorios `*-db` con Liquibase + PostgreSQL 16 en Docker). **Aún no hay servicios de API ni UI**; por eso este onboarding se centra en datos e infraestructura local. Complementa a [local-setup.md](../10-devops/local-setup.md) y a [migration-strategy.md](../06-data/migration-strategy.md).

## 1. Qué es el sistema (en una página)

Plataforma de **microservicios** para que los coordinadores del SENA **creen, validen y publiquen horarios de formación** con detección automática de conflictos. El diseño contempla **9 servicios** sobre 9 bounded contexts; la lógica diferenciadora vive en `scheduling-service` (motor de horarios) y `monitoring-service` (seguimiento). Ver [overview.md](../05-architecture/overview.md) y [domain-map.md](../02-domain/domain-map.md).

Vocabulario mínimo: **ficha** (cohorte de aprendices de un programa), **programa** y **competencia** (estructura curricular), **ambiente** (espacio físico), **horario** y **sesión de clase**. Diccionario completo en el [glosario](../01-context/glossary.md) y el mapeo es↔en en el [lenguaje ubicuo](../02-domain/domain-map.md#lenguaje-ubicuo--mapeo-dominio--técnico).

## 2. Prerrequisitos

- **Docker** + **Docker Compose**.
- **Git** y acceso a los repositorios del proyecto.
- Un cliente PostgreSQL (psql, DBeaver, etc.) para inspeccionar la BD (opcional pero recomendado).

## 3. Estructura del monorepo (capa de datos)

Cada módulo tiene su propio repositorio `*-db`, y una carpeta de infraestructura orquesta la BD y los servicios Liquibase. Los repos `*-db` se clonan **junto a** la carpeta de infraestructura:

```
workspace/
├── infra/                     # docker-compose + archivos de entorno (.env.develop/.qa/.staging/.main)
├── iam-db/                    # módulo M1  → schemas identity/rbac/session
├── reference-data-db/         # módulos M2+M4 → institutional_structure, catálogos
├── academic-management-db/    # módulos M5+M6 → academic_management
├── training-environment-db/   # módulo M3  → training_environment
├── scheduling-db/             # módulo M8  → scheduling
├── actors-db/                 # módulo M7  → actors
├── document-db/               # transversal → document
├── monitoring-db/             # módulo M9  → monitoring
└── audit-db/                  # transversal → audit (append-only)
```

Dentro de cada repo `*-db`, el DDL se organiza por carpetas numeradas y se orquesta desde un `changelog-master.yaml`. Orden canónico (ver [modeling-conventions §7](../06-data/modeling-conventions.md) y [migration-strategy.md](../06-data/migration-strategy.md)):

```
00_extensions → 01_schemas → 02_types → 03_tables (sin FKs)
→ 04_alter (FKs vía ALTER TABLE) → 05_views … 08_triggers → 10_indexes
→ 02_dml (seeds idempotentes) → 03_dcl (roles/grants) → 04_tcl (tags de release)
05_rollbacks/  # espejo de cada changeset forward, misma ruta relativa
```

## 4. Levantar el entorno local

Todos los módulos conviven en **una sola base de datos PostgreSQL 16**; cada módulo usa **su propio schema** y su propio tracking de Liquibase. Docker Compose lee `.env` por defecto, así que hay que pasar el archivo de entorno explícitamente.

```bash
# 1) Levantar Postgres para el ambiente de desarrollo
docker compose --env-file .env.develop up postgres -d

# 2) Aplicar los changelogs de un módulo (perfil "tooling")
docker compose --env-file .env.develop --profile tooling run --rm liquibase-<modulo> update

# 3) Verificar el estado de las migraciones de un módulo
docker compose --env-file .env.develop --profile tooling run --rm liquibase-<modulo> status --verbose
```

Módulos disponibles (`liquibase-<modulo>`): `academic`, `actors`, `audit`, `document`, `iam`, `monitoring`, `reference`, `scheduling`, `training`.

**Orden recomendado de aplicación:** primero `reference` (catálogos e institucional) e `iam` (identidad); luego el resto de módulos que los referencian.

### Rollback y limpieza (solo local)

```bash
# Revertir el último changeset de un módulo
docker compose --env-file .env.develop --profile tooling run --rm liquibase-<modulo> rollbackCount 1

# Volver a un release etiquetado (tags en 04_tcl/)
docker compose --env-file .env.develop --profile tooling run --rm liquibase-<modulo> rollback <tag>

# Apagar / reinicio limpio (borra el volumen de datos)
docker compose --env-file .env.develop down
docker compose --env-file .env.develop down -v
```

> En `qa`/`staging`/`main` las migraciones son **forward-only**; los rollbacks son para desarrollo local y como plan de contingencia documentado, no como flujo normal.

## 5. Convenciones que debes conocer antes de tu primer cambio

- **Naming en inglés** para BD y contratos; dominio se habla en español (HALT-DB-NAMING; ver [lenguaje ubicuo](../02-domain/domain-map.md)).
- **DB por servicio / schema por módulo:** ningún módulo escribe en `public` ni en el schema de otro.
- **Columnas de auditoría obligatorias** en tablas transaccionales (`created_at/by`, `updated_at/by`, `deleted_at/by`, `is_active`, `row_version`) y actor de sistema `00000000-0000-0000-0000-000000000000` ([modeling-conventions §2](../06-data/modeling-conventions.md)).
- **Tres conceptos de "estado"** no se mezclan: ciclo de vida técnico (`is_active`/`deleted_at`), estado de negocio (FK a catálogo `status`), y enum técnico cerrado (`CHECK IN (...)`). Ver [modeling-conventions §1](../06-data/modeling-conventions.md).
- **Cada changeset** declara `id` + `author` únicos y su `rollback` espejo en `05_rollbacks/`.
- **Seeds idempotentes** (`INSERT ... ON CONFLICT DO NOTHING`); separar catálogo de datos de prueba con `context`/`labels`.
- **Lógica almacenada** (vistas/funciones/triggers) con `runOnChange: true`.
- **Secretos fuera de git:** los `.env.*` no versionan contraseñas reales; usar `.env.example` ([security-rules.md](../00-governance/security-rules.md)).
- **Git y ramas:** promoción `develop → qa → staging → main` ([git-conventions.md](../00-governance/git-conventions.md)).

## 6. Gobernanza y decisiones

- Toda dependencia nueva entre bounded contexts requiere un **ADR** ([05-architecture/decisions](../05-architecture/decisions/README.md)).
- Estándares transversales de datos ratificados en [ADR-004](../05-architecture/decisions/records/ADR-004-status-parametrization-and-audit-standard.md).
- Antes de contribuir, lee [CONTRIBUTING.md](../CONTRIBUTING.md) y las [reglas de gobernanza](../00-governance/).

## 7. Checklist de primer día

- [ ] Docker y Docker Compose funcionando.
- [ ] Repos `*-db` e `infra/` clonados con la estructura de la §3.
- [ ] Postgres levantado con `.env.develop`.
- [ ] `reference` e `iam` aplicados y verificados con `status --verbose`.
- [ ] Resto de módulos aplicados en orden.
- [ ] Leídos: [overview.md](../05-architecture/overview.md), [domain-map.md](../02-domain/domain-map.md), [modeling-conventions.md](../06-data/modeling-conventions.md), [migration-strategy.md](../06-data/migration-strategy.md).

## Referencias

- [local-setup.md](../10-devops/local-setup.md) · [migration-strategy.md](../06-data/migration-strategy.md) · [modeling-conventions.md](../06-data/modeling-conventions.md)
- [overview.md (arquitectura)](../05-architecture/overview.md) · [domain-map.md](../02-domain/domain-map.md) · [09-microservices](../09-microservices/README.md)
- [CONTRIBUTING.md](../CONTRIBUTING.md) · [00-governance](../00-governance/)
