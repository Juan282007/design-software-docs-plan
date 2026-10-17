# Estrategia de pruebas

> Estado: 🟡 En progreso | Última actualización: 2026-08-28
> Autor: Jesús Ariel González Bonilla (PM + Arquitecto) | Equipo: Calidad

Estrategia de pruebas para los microservicios del sistema **Horarios SENA**. El stack es **políglota Java + Go** ([ADR-006](../05-architecture/decisions/records/ADR-006-polyglot-stack-strategy.md)), no un único lenguaje: `iam`, `scheduling`, `academic-management` y `actors` son Java/Spring; `notification`, `audit`, `reference-data`, `training-environment` y `monitoring` son Go. La construcción avanza por capas y hoy conviven dos estados:

- **Capa de aplicación ya construida y probada en `develop`:** `iam-service` (Java, 12/12 HU — ~162 tests, 9+ ITs contra Postgres real) y `notification-service` (Go, 8/8 HU — unit + E2E vivo). Aquí los niveles Unitario, Contrato, Integración y E2E están **ACTIVOS**.
- **Solo capa de datos:** el resto de servicios (`*-db`, Liquibase + PostgreSQL 16 en Docker), donde el foco sigue siendo migraciones, DDL, seeds y restauración.

Este documento describe la pirámide completa y marca en cada nivel **qué está activo hoy** (con evidencia en `iam`/`notification`) y **qué se activará cuando el resto de servicios construyan su capa de aplicación**.

## Principios

- **Prueba lo que existe.** Para los servicios que solo tienen capa de datos, el foco es migraciones, DDL, seeds y restauración. Para los que ya tienen capa de aplicación (`iam`, `notification`), aplican los cuatro niveles superiores de la pirámide con evidencia real.
- **Aislamiento por servicio.** Cada `*-db` se prueba de forma independiente, con su propio schema y su propio tracking de Liquibase (ver [migration-strategy.md](../06-data/migration-strategy.md)).
- **Idempotencia y reversibilidad.** Toda migración debe poder aplicarse, revertirse y re-aplicarse en un entorno limpio sin intervención manual.
- **Rápido y determinista.** Las pruebas corren sobre contenedores efímeros de PostgreSQL 16 y no dependen de datos preexistentes ni de orden entre servicios.

## Pirámide de pruebas

De más numerosas/baratas (base) a más costosas (cima). El estado indica la disponibilidad **hoy**.

| Nivel | Qué valida | Estado hoy |
|-------|------------|------------|
| Migraciones y DDL | El esquema se crea, revierte y re-crea correctamente por servicio | ✅ ACTIVO (todos los `*-db`) |
| Datos semilla (seeds) | Catálogos idempotentes se cargan sin duplicar ni fallar | ✅ ACTIVO |
| Unitarias (Java / Go) | Lógica de negocio pura en servicios/handlers/use cases | ✅ ACTIVO en `iam` (Java) y `notification` (Go) · ⏳ resto de servicios |
| Contrato entre servicios | Eventos y respuestas respetan el contrato publicado; **gate Pact operativo** | ✅ ACTIVO (Pact) para pares con productor/consumidor construidos · ⏳ resto |
| Integración | Servicio + su BD + broker reales | ✅ ACTIVO en `iam` (9+ ITs vs Postgres real) y `notification` · ⏳ resto |
| End-to-end | Flujo de negocio completo con broker/BD frescos | ✅ ACTIVO en `iam` (login→JWT→outbox→relay→iam-events) y `notification` (happy/DLQ/idempotencia) · ⏳ flujos multi-servicio posteriores |

## Nivel 1 — Pruebas de migraciones Liquibase (hoy)

Es el nivel con mayor cobertura efectiva en el estado actual del proyecto. Cada repo `*-db` tiene un `changelog-master.yaml` y se despliega con el perfil `tooling` de Docker Compose (ver [local-setup.md](../10-devops/local-setup.md)).

Casos mínimos por servicio:

1. **Forward limpio.** Sobre una BD vacía, `update` aplica todo el changelog sin error y respetando el orden `00_extensions → 01_schemas → 02_types → 03_tables → 04_alter (FKs) → 05_views…08_triggers → 10_indexes → 02_dml (seeds) → 03_dcl → 04_tcl`.
2. **Rollback.** Cada changeset declara su `rollback` espejo en `05_rollbacks/`; se valida `rollback <tag>` hasta dejar la BD en el estado anterior. Los rollbacks se prueban **en local**; en `qa`/`staging`/`main` las migraciones son forward-only.
3. **Re-aplicación (idempotencia de esquema).** Tras un rollback, un nuevo `update` vuelve a dejar el esquema íntegro.
4. **`runOnChange` en lógica almacenada.** Vistas, funciones, procedimientos y triggers (`05_views`/`06_functions`/`07_procedures`/`08_triggers`) marcados con `runOnChange: true` se re-aplican al cambiar su definición.
5. **Integridad referencial.** Las FKs creadas en `04_alter` existen con la acción `ON UPDATE`/`ON DELETE` correcta y hay un índice por cada FK.

Comandos de referencia (un módulo a la vez; módulos: `liquibase-academic`, `liquibase-actors`, `liquibase-audit`, `liquibase-document`, `liquibase-iam`, `liquibase-monitoring`, `liquibase-reference`, `liquibase-scheduling`, `liquibase-training`):

```bash
# Levantar Postgres del ambiente de pruebas
docker compose --env-file .env.develop up postgres -d

# Aplicar y verificar el estado de un módulo
docker compose --env-file .env.develop --profile tooling run --rm liquibase-<modulo> update
docker compose --env-file .env.develop --profile tooling run --rm liquibase-<modulo> status --verbose

# Validar reversibilidad en local
docker compose --env-file .env.develop --profile tooling run --rm liquibase-<modulo> rollback <tag>
```

> Buena práctica: usar `liquibase validate` para detectar changesets con `id`/`author` duplicados o `rollback` faltante antes de abrir el PR.

## Nivel 2 — Datos de prueba y seeds (hoy)

- Los **seeds de catálogo** viven en `02_dml/` y son **idempotentes** (`INSERT ... ON CONFLICT DO NOTHING` o `MERGE`), de modo que re-ejecutarlos no falla ni duplica. La prueba consiste en aplicar el seed dos veces y verificar conteos estables.
- Los **datos de prueba** (fixtures que no deben llegar a `staging`/`main`) se aíslan con `context`/`labels` de Liquibase, de modo que un `update` en ambientes compartidos no los cargue.
- Para escenarios que necesiten datos ficticios, usar valores seguros (ver [security-rules.md](../00-governance/security-rules.md)); nunca datos personales reales de aprendices o instructores.

## Nivel 3 — Unitarias de la capa de aplicación (ACTIVO en `iam` y `notification`)

Cubren lógica de negocio pura con dependencias externas simuladas. Ya activo:

- **`iam-service` (Java/Spring):** parte de las ~162 pruebas del servicio son unitarias sobre reglas de autenticación, RBAC y emisión de token.
- **`notification-service` (Go):** unitarias sobre resolución de destinatario, render de plantilla y máquina de reintentos.
- Para el resto de servicios (aún en capa de datos), cuando construyan su capa de aplicación: cubrir validaciones, máquinas de estado parametrizables de ADR-004 y cálculo de KPIs en `monitoring`. `scheduling-engine-workflow` y `conflict-validator-worker` (validación de choques de horario) son candidatos prioritarios por su densidad de reglas.
- Umbral de cobertura objetivo: **punto abierto**, a fijar en el gate de [ci-cd.md](../10-devops/ci-cd.md).

> ⚠️ **Las unitarias no bastan por sí solas.** Dos incidentes reales confirmaron que suites unitarias verdes dejaron pasar defectos de arranque y de integración; ver las reglas de **contexto completo (Java)** y **E2E con broker/BD fresco (Go)** más abajo.

### Regla de contexto completo (Java) — `@SpringBootTest` obligatorio

Todo servicio Java debe tener **al menos un `@SpringBootTest` de contexto completo** (*context-loads*) que arranque el `web` environment real, **además** de los `@WebMvcTest`/unitarios por controlador.

- **Incidente que lo motiva (ambiguous-mapping):** dos `@RestController` implementaban la misma interfaz OpenAPI generada (`UsersApi`) → Spring falló al arrancar por *ambiguous mapping*. Los `@WebMvcTest` y las unitarias **no lo cazaron**, porque cargan un solo controlador aislado y nunca levantan el contexto web completo.
- **Remediación adoptada:** se agregó `IamApiApplicationContextTest`, un test *full-boot* que arranca el contexto entero; solo esa clase detecta el choque de mapeos.
- **Regla derivada:** una interfaz OpenAPI generada = **un único** `@RestController` que la implementa, y **un** test de contexto completo por servicio Java que garantice que el ApplicationContext levanta.

### Regla Go / eventos — E2E con broker y BD frescos

En los servicios Go orientados a eventos, las **unitarias mockean el use case**, así que no ejercitan el camino real de consumo/entrega. La validación de aceptación exige un **E2E con broker (RabbitMQ) y BD frescos**.

- **Incidente que lo motiva (notification HU-004):** el use case devolvía `nil`; los unit tests (que mockean el use case) pasaban en verde. **Solo el E2E** con broker/BD limpios detectó el defecto.
- **Regla derivada:** ninguna HU de consumo/entrega de eventos se da por *done* con solo unitarias; requiere E2E real que cubra happy path, DLQ e idempotencia por `event_id`.

## Nivel 4 — Pruebas de contrato entre servicios (ACTIVO — gate Pact)

La arquitectura es orientada a eventos (broker, ADR-001) con read models (ADR-002). El **gate de contrato con Pact ya está operativo** como nivel de prueba real: es un check que valida que productor y consumidor comparten el mismo contrato antes de integrar.

- **Ya activo:** `iam-service` publica `user.created`, `user.deactivated`, `role.assigned` y `session.started` en el exchange `iam-events`; `notification-service` consume vía AMQP. El par productor/consumidor construido se cubre con Pact.
- Validar el **envelope de eventos** y el `event_type` (ej. `scheduling.class_session.created`) contra el [event-catalog.md](../09-microservices/event-catalog.md).
- Verificar **idempotencia de consumo**: `notification`, `audit` y otros consumidores deben tolerar re-entrega at-least-once (unicidad por `event_id` / `source_event_id`).
- Enfoque consumer-driven contract entre cada par publicador/consumidor, para desacoplar despliegues.

> **Deuda de contrato conocida:** en `shared-contracts` conviven un *envelope plano* y uno *anidado* (`error-codes.yaml` vs `_shared.yaml`); además el Outbox persiste la carga bajo `data` y el relay la renombra a `payload` al publicar. Las pruebas de contrato deben fijar cuál es la forma canónica; ver [shared-contracts](../07-api/) y la skill de contratos.

## Nivel 5 — Integración y E2E (ACTIVO en `iam` y `notification`)

- **Integración (activo):** `iam-service` corre **9+ ITs contra PostgreSQL real** (login, refresh, RBAC, `reports/login-audit` con paginación por cursor). `notification-service` ejercita su worker contra broker + BD reales.
- **E2E ejecutado (activo):**
  - `notification`: flujo de consumo con broker/BD frescos cubriendo *happy path*, DLQ e idempotencia por `event_id`.
  - `iam` (compose completo): `login → JWT → endpoints protegidos → escritura en outbox → relay → publicación en iam-events`, con trazas visibles en Tempo.
- **E2E multi-servicio (futuro):** flujo de negocio completo (ej. crear ambiente → programar sesión → validar conflicto → emitir documento) atravesando varios servicios; se planificará a medida que esos servicios construyan su capa de aplicación.

## Deuda de CI (nombrada)

Hoy **no existe pipeline de CI**. Consecuencias que se aceptan como deuda explícita:

- Las suites (~162 tests de `iam` + unit/E2E de `notification`) se ejecutan **a mano**, no en cada PR.
- Hay **3 ITs *flaky*** en `develop` sin pipeline que los aísle o estabilice.
- El **gate Pact** corre pero no está automatizado como check bloqueante de PR.

Deuda registrada: **HU-NOTIF-009 (docs#78)** para montar CI + Pact automatizado, enlazada con [TD-010](../15-project-control/technical-backlog.md) y [ci-cd.md](../10-devops/ci-cd.md).

## Trazabilidad y evidencia

- Cada HU probada se registra con la plantilla [_template-test-evidence.md](./_template-test-evidence.md) (casos `TC-NNN`, defectos, trazabilidad `AC → HU → TC`).
- Cada sprint/release consolida resultados con [_template-qa-report.md](./_template-qa-report.md), incluyendo el gate de calidad.
- La calidad forma parte de la [Definition of Done](../00-governance/definition-of-done.md): un cambio no está *done* si sus pruebas aplicables no pasan.

## Puntos abiertos

- Estandarizar utilería de contenedores efímeros por stack (Java y Go, ADR-006) para los servicios que aún no tienen capa de aplicación; `iam` y `notification` ya la resuelven a su manera.
- Umbral de cobertura y su automatización en CI: por cerrar con [ci-cd.md](../10-devops/ci-cd.md); ver **Deuda de CI** arriba (HU-NOTIF-009 / docs#78).
- Estabilizar los 3 ITs *flaky* de `develop` y convertir el gate Pact en check bloqueante una vez exista pipeline.
