# Respaldo y recuperación

> Estado: 🟡 En progreso | Última actualización: 2026-08-01
> Autor: Jesús Ariel González Bonilla (PM + Arquitecto) | Equipo: Operaciones

Estrategia de respaldo y recuperación de la **base de datos** del sistema **Horarios SENA**. Es el documento operativo más relevante en el estado actual, porque **la capa de datos es la única construida**: todos los servicios comparten una única instancia de PostgreSQL 16, con un schema por módulo y el tracking de Liquibase aislado por módulo (ver [migration-strategy.md](../06-data/migration-strategy.md)). Incluye un caso especial: el `audit-service` (`audit_record`) es **append-only e inmutable**, con requisitos de retención propios.

## Qué se respalda

| Activo | Contenido | Criticidad |
|--------|-----------|------------|
| BD PostgreSQL (todos los schemas) | Datos de negocio de los 9 módulos | Alta |
| `audit_record` (append-only) | Historial inmutable de eventos; retención mínima prevista de 7 años (pendiente de confirmar con normativa SENA) | Alta / legal |
| Changelogs Liquibase (`*-db` en git) | Definición versionada del esquema | Media (recuperable desde git) |
| Archivos `.env.*` | Configuración por ambiente — **sin secretos versionados** | Fuera de git (ver [security-rules.md](../00-governance/security-rules.md)) |

> El esquema no requiere backup de datos porque vive en git; lo que se respalda son los **datos**. La recuperación combina *restaurar datos* + *reaplicar/verificar migraciones al tag correcto*.

## Objetivos: RTO y RPO

| Objetivo | Definición | Valor (referencia, **por confirmar**) |
|----------|------------|----------------------------------------|
| **RPO** (Recovery Point Objective) | Máxima pérdida de datos tolerable | Objetivo bajo para producción; requiere WAL archiving / PITR |
| **RTO** (Recovery Time Objective) | Máximo tiempo para restaurar el servicio | A comprometer con el SLO por servicio |

> Los valores definitivos de RTO/RPO son un **punto abierto**: se fijarán al desplegar producción y se registrarán en [_template-sla-slo-sli.md](./_template-sla-slo-sli.md). El punto clave hoy: `pg_dump` por sí solo acota el RPO al intervalo entre dumps; para un RPO fino se necesita **PITR** con archivado de WAL.

## Métodos de respaldo

### 1. Dump lógico (`pg_dump` / `pg_dumpall`)

Sencillo y portable; adecuado para respaldos periódicos y para mover datos entre ambientes.

```bash
# Dump de toda la base (formato custom, comprimido)
docker compose --env-file .env.develop exec -T postgres \
  pg_dump -U <user> -d <db> -F c -f /backups/horarios_$(date +%Y%m%d_%H%M).dump

# Dump de un solo schema/módulo (respaldo granular por servicio)
docker compose --env-file .env.develop exec -T postgres \
  pg_dump -U <user> -d <db> -n <schema_modulo> -F c -f /backups/<modulo>_$(date +%Y%m%d).dump
```

- Poder respaldar **por schema** encaja con el aislamiento por módulo: se puede recuperar un solo servicio sin tocar los demás.
- El RPO de este método es el **intervalo entre dumps**.

### 2. PITR (Point-In-Time Recovery) — para producción

Base física (`pg_basebackup`) + **archivado continuo de WAL**, que permite restaurar a un instante concreto y minimiza el RPO. Es el método recomendado cuando exista un ambiente productivo real. Su configuración concreta (destino de WAL, frecuencia de base) es un **punto abierto**.

## Restauración

```bash
# Restaurar un dump custom completo sobre una BD limpia
docker compose --env-file .env.develop exec -T postgres \
  pg_restore -U <user> -d <db> --clean --if-exists /backups/horarios_YYYYMMDD_HHMM.dump

# Tras restaurar datos, verificar que el esquema está en el tag esperado
docker compose --env-file .env.develop --profile tooling run --rm liquibase-<modulo> status --verbose
```

Orden recomendado de recuperación total:
1. Levantar PostgreSQL limpio del ambiente.
2. Restaurar datos (dump o PITR).
3. Verificar con Liquibase que cada módulo está en el **tag de release** correcto (`04_tcl`); si falta esquema, aplicar `update`.
4. Verificar integridad: `pg_isready`, conteos de catálogos clave y unicidad de `audit_record` por `event_id`.

## Migraciones y rollback como recuperación

La recuperación no siempre es restaurar un backup; a veces es **revertir una migración**:

- Todo changeset forward tiene su **rollback espejo** en `05_rollbacks/`. En **local** se usa `rollback <tag>` para volver al estado previo.
- En `qa`/`staging`/`main` las migraciones son **forward-only**: ante una migración defectuosa en un ambiente compartido no se hace rollback en caliente, se prepara un **forward-fix** (nuevo changeset correctivo) y, si hubo daño de datos, se recurre al backup.
- Los **tags de release** (`04_tcl`) son los puntos de recuperación de esquema conocidos; por eso su disciplina es parte de esta estrategia.

## Prueba de restauración (restore drill)

Un backup no probado no es un backup. Marco de verificación periódica:

1. Tomar el último dump y restaurarlo sobre una BD **efímera** limpia.
2. Ejecutar `liquibase status --verbose` por módulo: no debe faltar ni sobrar esquema.
3. Validar conteos de catálogos y una muestra de integridad referencial.
4. Registrar resultado, duración (insumo para el RTO) y cualquier incidencia.

> **Punto abierto:** frecuencia formal del drill (p. ej. por release) y su automatización en CI/CD; se cerrará junto con [ci-cd.md](../10-devops/ci-cd.md).

## Consideraciones de seguridad

- Los dumps pueden contener datos personales de aprendices/instructores: tratarlos como sensibles, cifrarlos en reposo y **no** versionarlos en git (ver [security-rules.md](../00-governance/security-rules.md)).
- El `audit_record` es inmutable y con retención legal prevista (mín. 7 años, por confirmar): su respaldo y archivado (cold storage tras 2 años activos) debe preservar esa inmutabilidad.

## Puntos abiertos

- Valores definitivos de RTO/RPO por servicio.
- Configuración de PITR (archivado de WAL) para producción.
- Frecuencia y automatización del restore drill.
- Destino y cifrado de los backups en cada ambiente.

## Referencias

- [migration-strategy.md](../06-data/migration-strategy.md)
- [_template-sla-slo-sli.md](./_template-sla-slo-sli.md)
- [incident-management.md](./incident-management.md)
- [local-setup.md](../10-devops/local-setup.md)
- [security-rules.md](../00-governance/security-rules.md)
- [ADR-004 — Parametrización de estados y estándar de auditoría](../05-architecture/decisions/records/ADR-004-status-parametrization-and-audit-standard.md)
