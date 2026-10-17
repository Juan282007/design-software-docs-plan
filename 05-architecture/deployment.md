# Topología de despliegue

> Estado: 🟡 En progreso | Última actualización: 2026-08-01
> Autor: Jesús Ariel González Bonilla (PM + Arquitecto) | Equipo: Arquitectura

Cómo se despliega el sistema hoy y cómo se prevé que se despliegue cuando exista la capa de aplicación. Complementa a [overview.md](./overview.md) §Topología, a [10-devops/local-setup.md](../10-devops/local-setup.md) y a [10-devops/environments.md](../10-devops/environments.md).

> **Honestidad sobre el estado:** además de la capa de datos (Postgres + runners de Liquibase por módulo),
> ya se despliegan por perfil de compose dos servicios de dominio reales: `notification` (HU-NOTIF-008,
> api+worker) e `iam` (HU-IAM-012, api). Ambos corren detrás del edge gestionado (Traefik, ADR-007) y
> exportan OTel al Collector (ADR-008) cuando el perfil `observability` está activo. El resto de los 9
> servicios (RF-*) sigue sin desplegarse — el resto de esta página describe el **estado objetivo** para esos,
> marcado como tal.

---

## Estado actual — Lo que hoy se despliega

### Componentes reales

| Componente | Qué es | Dónde vive |
|------------|--------|-----------|
| **PostgreSQL 16** | Una única instancia en contenedor Docker; una BD con **un schema por módulo** (`identity`/`rbac`/`session`, `academic_management`, `training_environment`, `monitoring`, `institutional_structure`, `document`, `audit`, `scheduling`, …) | Docker Compose (infra) |
| **Runners Liquibase** (uno por módulo) | Servicios efímeros del perfil `tooling` que aplican el `changelog-master.yaml` de cada repo `*-db` | Docker Compose, perfil `tooling` |
| **Repos `*-db`** | Fuente de las migraciones: `academic-db`, `actors-db`, `audit-db`, `document-db`, `iam-db`, `monitoring-db`, `reference-db`, `scheduling-db`, `training-db` | Repositorios independientes |
| **Archivos de entorno** | `.env.develop`, `.env.qa`, `.env.staging`, `.env.main` — parametrizan Postgres y credenciales por ambiente | Repo de infraestructura |

### Topología local (existe)

```
┌──────────────────────────── Docker Compose ────────────────────────────┐
│                                                                         │
│   ┌───────────────────────────────────────────────────────────────┐    │
│   │                     PostgreSQL 16 (un contenedor)              │    │
│   │   Una BD · un schema por módulo · databasechangelog por módulo │    │
│   └───────────────────────────────────────────────────────────────┘    │
│            ▲          ▲          ▲          ▲          ▲                 │
│            │ update   │ update   │ update   │ update   │ update          │
│   ┌────────┴───┐ ┌────┴─────┐ ┌──┴──────┐ ┌─┴───────┐ ┌┴──────────┐      │
│   │liquibase-  │ │liquibase-│ │liquibase│ │liquibase│ │liquibase- │ ...  │
│   │iam         │ │reference │ │-academic│ │-actors  │ │scheduling │      │
│   └────────────┘ └──────────┘ └─────────┘ └─────────┘ └───────────┘      │
│        (perfil `tooling` — contenedores efímeros que corren y salen)     │
└─────────────────────────────────────────────────────────────────────────┘
      Configuración por ambiente: --env-file .env.<ambiente>
```

### Aislamiento por módulo

- Cada módulo **crea y usa su propio schema**; ningún módulo escribe en `public`.
- Se aísla el **tracking de Liquibase por módulo** (`--liquibase-schema-name=<modulo>`), de modo que cada servicio tenga su propia tabla `databasechangelog` y su historial sea auditable y reversible de forma independiente. Ver [06-data/migration-strategy.md](../06-data/migration-strategy.md).
- **Orden de aplicación recomendado:** primero `reference` (catálogos e institucional) e `iam` (identidad), luego los módulos que los referencian.

---

## Promoción por ambientes

El despliegue de esquema se promueve por las cuatro ramas/ambientes protegidos: **`develop → qa → staging → main`** (ver [00-governance/git-conventions.md](../00-governance/git-conventions.md) y [10-devops/environments.md](../10-devops/environments.md)).

| Ambiente | Base de datos | Migraciones | Datos de prueba |
|----------|---------------|-------------|-----------------|
| `develop` | BD de desarrollo | Forward + rollback local permitido | Sí (seeds de prueba con `context`/`labels`) |
| `qa` | BD de QA | **Forward-only** | No (solo catálogos de negocio) |
| `staging` | BD de preproducción | **Forward-only** | No |
| `main` | BD de producción | **Forward-only** | No |

- **Forward-only en ambientes compartidos:** en `qa`/`staging`/`main` las migraciones no se revierten en el flujo normal; los rollbacks son plan de contingencia documentado, no operación rutinaria (ver [migration-strategy.md](../06-data/migration-strategy.md)).
- **Tags de release** (`04_tcl/`) al cierre de cada iteración permiten `rollback <tag>` como contingencia.
- **Seeds idempotentes** (`INSERT ... ON CONFLICT DO NOTHING`) para poder re-ejecutar sin fallar; los datos de prueba se separan de los de catálogo con `context`/`labels` para que **no** lleguen a `staging`/`main`.

---

## Estado objetivo — Cuando exista la capa de aplicación

> Todo lo siguiente es **diseño previsto**, coherente con [overview.md](./overview.md) y los ADR. Nada de esto está desplegado hoy.

Por cada uno de los 9 servicios se prevé desplegar, además de su BD:

- **API del servicio** (una o varias réplicas tras el gateway).
- **Workers** de consumo de eventos donde aplique (p. ej. proyección de read models en `scheduling-service`, ADR-002).

Componentes compartidos previstos:

| Componente | Rol | Decisión |
|------------|-----|----------|
| **API Gateway** | Routing, SSL termination, rate limiting, CORS | [overview.md](./overview.md) |
| **Broker de mensajes (RabbitMQ)** | Transporte de eventos async; fan-out a `audit-service` | [ADR-001](./decisions/records/ADR-001-message-broker.md) |
| **Object storage (MinIO/S3)** | Binarios de `document-service` (PDFs); binarios **nunca** en BD | [ADR-003](./decisions/records/ADR-003-object-storage.md) |
| **Caché (Redis)** | Consultas de alta frecuencia por consumidor | [overview.md](./overview.md) |
| **Secret Manager** | Credenciales y clave privada RSA fuera de git | [security-threat-model.md](./security-threat-model.md) |

### Regla que se mantiene al crecer

- **Una base de datos por servicio.** Hoy conviven como schemas en una sola instancia; el estado objetivo separa a instancia/credencial por servicio (sin usuario de BD compartido, ver threat model I-06). El código de aplicación no cambia: el aislamiento ya es por schema.
- El paso de object storage se hace por adaptador S3 único (MinIO en DEV/QA, compatible S3 en PROD), de modo que el código sea idéntico entre ambientes (ADR-003).

---

## Riesgos de despliegue

| Riesgo | Estado | Mitigación |
|--------|--------|------------|
| Secretos: `.env.*` con posibilidad de contener credenciales versionadas | **Real hoy** | Usar `.env.example` como plantilla; valores reales fuera de git; mover a Secret Manager. Ver [security-threat-model.md](./security-threat-model.md) §Secretos |
| Orden de migración entre módulos con referencias cruzadas | Real hoy | FKs vía `ALTER TABLE` tras crear toda la estructura; aplicar `reference`/`iam` primero |
| Complejidad operativa de 9 BDs al separar instancias | Previsto | Automigración por servicio con Liquibase; sin DBA centralizado |
| Split-brain si el broker cae durante una publicación | Previsto | Patrón Outbox en el productor (ADR-001) para entrega at-least-once |

## Referencias

- [10-devops/local-setup.md](../10-devops/local-setup.md) — comandos de despliegue local
- [10-devops/environments.md](../10-devops/environments.md) — los 4 ambientes y sus `.env`
- [10-devops/ci-cd.md](../10-devops/ci-cd.md) — validación y promoción automatizadas (previsto)
- [06-data/migration-strategy.md](../06-data/migration-strategy.md) — orden, seeds, rollbacks, tags
- [ADR-001](./decisions/records/ADR-001-message-broker.md) · [ADR-003](./decisions/records/ADR-003-object-storage.md)
