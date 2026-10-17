# Modelo de amenazas de seguridad

> Estado: 🟡 En progreso | Última actualización: 2026-08-01
> Autor: Jesús Ariel González Bonilla (PM + Arquitecto) | Equipo: Arquitectura

Modelo de amenazas **STRIDE ligero** para el sistema de Gestión de Horarios SENA. Clasifica activos, identifica amenazas por categoría STRIDE y define controles con su estado. Complementa a [overview.md](./overview.md), [cross-cutting.md](./cross-cutting.md) y a [00-governance/security-rules.md](../00-governance/security-rules.md).

> **Alcance real:** hoy solo existe la **capa de datos** (PostgreSQL 16 en Docker, migraciones Liquibase, `.env` por ambiente). Por eso la superficie **activa** actual es pequeña (base de datos + secretos de infraestructura). Las amenazas que involucran gateway, JWT, broker o object storage corresponden a la **capa de aplicación aún no construida** y se documentan como diseño previsto. El estado de cada control lo indica.

---

## Activos a proteger

| Activo | Clasificación | Dueño | Impacto si se compromete |
|--------|--------------|-------|--------------------------|
| Credenciales de BD (usuario/clave por módulo) | CRÍTICO | Infra / `.env` | Acceso total a los datos del sistema |
| PII de instructores y aprendices (documento, email, teléfono) | ALTO | actors-service | Violación Ley 1581/2012 (Habeas Data) |
| Log de auditoría (`audit_record`) | ALTO | audit-service | Pérdida de trazabilidad / encubrimiento |
| Credenciales de usuario (`password_hash`) | CRÍTICO | iam-service | Suplantación de usuarios (previsto) |
| Tokens JWT y clave privada RSA | CRÍTICO | iam-service / Secret Manager | Falsificación de sesión o de cualquier token (previsto) |
| Horarios publicados | MEDIO | scheduling-service | Manipulación de la operación del centro |
| Catálogos y parámetros del sistema | BAJO | reference-data-service | Alteración de reglas operativas |

---

## Superficie de ataque

| Componente | Exposición | Estado | Vector principal |
|------------|-----------|--------|------------------|
| **PostgreSQL 16** (una BD, schema por módulo) | Interna | **Existe** | Acceso con credencial filtrada; acceso lateral entre schemas; SQL injection (cuando exista la capa app) |
| **Archivos `.env.*`** | Repositorio / infra | **Existe** | Secretos versionados en git |
| **Runners Liquibase** | Interna (efímeros) | Existe | Migración maliciosa; changeset sin revisar |
| API Gateway | Pública | Previsto | Inyección, fuerza bruta, DoS |
| iam-api / endpoints autenticados | Pública | Previsto | Credential stuffing, abuso de scope, scraping de PII |
| Broker de mensajes | Interna | Previsto | Inyección de eventos falsos, replay |
| Object Storage (S3/MinIO) | Interna + URLs firmadas | Previsto | URLs firmadas filtradas, enumeración |

---

## Amenazas STRIDE

Prob. = probabilidad; los IDs se referencian desde otros documentos. "(previsto)" marca amenazas que solo aplican cuando exista la capa de aplicación.

### S — Spoofing (Suplantación)

| ID | Amenaza | Prob. | Impacto | Control |
|----|---------|-------|---------|---------|
| S-01 | Uso de credenciales de BD filtradas desde un `.env` versionado | **Alta hoy** | Crítico | Sacar secretos de git (ver §Secretos); rotar credenciales; Secret Manager |
| S-02 | Falsificación de JWT con clave comprometida | Baja | Crítico | RSA asimétrico; clave privada solo en Secret Manager; rotación periódica (previsto) |
| S-03 | Credential stuffing en login | Alta | Alto | Bloqueo por intentos (RN-IAM-01); rate limiting; 2FA en V2 (previsto) |
| S-04 | Inyección de evento falso en el broker | Baja | Alto | Red privada; TLS mutuo servicio↔broker; `source_service` verificado (previsto) |

### T — Tampering (Manipulación)

| ID | Amenaza | Prob. | Impacto | Control |
|----|---------|-------|---------|---------|
| T-01 | Changeset Liquibase malicioso o erróneo aplicado a un ambiente compartido | Media | Alto | Revisión en PR; forward-only en qa/staging/main; tags de release para contingencia |
| T-02 | Alteración del log de auditoría | Baja | Alto | `audit_record` solo INSERT (append-only, ADR-004); sin UPDATE/DELETE; permisos de BD restringidos |
| T-03 | SQL Injection | Media | Crítico | Queries parametrizadas; nunca concatenación de SQL; validación de entrada (capa app, previsto) |
| T-04 | Replay de eventos del broker | Media | Medio | Idempotencia por `event_id` UNIQUE (ver [cross-cutting.md](./cross-cutting.md) §6) (previsto) |
| T-05 | Manipulación de horario publicado | Baja | Medio | Invariante de dominio: `PUBLISHED` inmutable; cambios crean nueva versión |

### R — Repudiation (Repudio)

| ID | Amenaza | Prob. | Impacto | Control |
|----|---------|-------|---------|---------|
| R-01 | Falta de trazabilidad de cambios sensibles | Media | Alto | Estándar de auditoría ADR-004 (`created_by`/`updated_by`/`deleted_by`); `audit-service` consume todos los topics (previsto) |
| R-02 | Acción del sistema indistinguible de acción humana | Baja | Medio | UUID de actor reservado para acciones automáticas (ADR-004) |

### I — Information Disclosure (Divulgación)

| ID | Amenaza | Prob. | Impacto | Control |
|----|---------|-------|---------|---------|
| I-01 | Exposición de PII en logs | Media | Alto | PII nunca en logs, solo IDs (ver [cross-cutting.md](./cross-cutting.md) §3) (previsto) |
| I-02 | Fuga de PII por scope mal aplicado | Media | Alto | RBAC scope obligatorio (`TRAINING_CENTER`/`OWN_*`); filtro por centro en cada query (previsto) |
| I-03 | Acceso lateral entre schemas/BDs | Media | Alto | Credenciales propias por módulo; sin usuario compartido; al separar instancias, red segmentada |
| I-04 | URLs firmadas de object storage filtradas | Media | Medio | URLs firmadas con expiración corta (5 min); scope verificado antes de firmar (ADR-003) (previsto) |
| I-05 | Mensajes de error con detalles internos | Media | Medio | Formato de error estándar sin stack trace; `trace_id` interno (previsto) |

### D — Denial of Service

| ID | Amenaza | Prob. | Impacto | Control |
|----|---------|-------|---------|---------|
| D-01 | Saturación de endpoints públicos | Alta | Medio | Rate limiting por IP y usuario; circuit breaker (previsto) |
| D-02 | Agotamiento del pool de conexiones a BD | Media | Alto | Connection pooling con límite; health checks (previsto) |
| D-03 | Query costosa sin límite | Media | Medio | Índices (incluido uno por FK); paginación obligatoria; timeout 5 s |

### E — Elevation of Privilege

| ID | Amenaza | Prob. | Impacto | Control |
|----|---------|-------|---------|---------|
| E-01 | Usuario accede a features no asignados | Media | Alto | Verificación de `feature` del JWT en cada endpoint (previsto) |
| E-02 | Escalamiento horizontal (coordinador ve otro centro) | Media | Alto | Filtro `WHERE training_center_id = jwt.training_center_id` obligatorio (previsto) |
| E-03 | Manipulación del JWT para agregar features | Baja | Crítico | Firma RSA verificada en cada servicio (previsto) |

---

## Secretos — Riesgo real y prioritario hoy

> **Hallazgo actual:** los archivos de entorno `.env.<ambiente>` pueden contener **credenciales reales versionadas en git**. Esto es la mayor exposición de seguridad **hoy**, porque la capa de datos ya existe y sus credenciales viven en esos archivos.

- **Impacto:** cualquiera con acceso al repositorio (o a su historial) obtiene credenciales de base de datos → acceso directo a los datos, incluida PII sujeta a Ley 1581.
- **Regla:** los `.env.*` **no deben contener contraseñas reales**. Usar `.env.example` como plantilla y mantener los valores reales fuera de git (ver [10-devops/local-setup.md](../10-devops/local-setup.md) §Nota de secretos y [10-devops/environments.md](../10-devops/environments.md)).
- **Remediación (orden):**
  1. Rotar de inmediato cualquier credencial que haya estado versionada (**rotar primero, avisar después**, [security-rules.md](../00-governance/security-rules.md)).
  2. Sacar los `.env.*` del control de versiones; versionar solo `.env.example` con valores ficticios.
  3. Evaluar limpieza del historial de git si el secreto quedó registrado.
  4. Estado objetivo: mover credenciales a un **Secret Manager** cuando exista la capa de aplicación.

---

## Controles — Resumen y estado

| Control | Implementación | Estado |
|---------|----------------|--------|
| Auditoría append-only + soft delete | Estándar ADR-004 en el modelo | ✅ En el modelo |
| Estados y transiciones parametrizables | `status`/`status_transition` por servicio | ✅ En el modelo |
| Aislamiento por schema/módulo | Un schema por módulo; tracking Liquibase por módulo | ✅ Existe |
| Índice por cada FK (rendimiento y anti-DoS) | `10_indexes` en cada repo `*-db` | ✅ Existe |
| Secretos fuera de git | `.env.example` + Secret Manager | 🔴 **Pendiente (riesgo real)** |
| Autenticación JWT RS256 | iam-service | 🟡 Diseñado (capa app) |
| Autorización RBAC (Feature + Scope) | iam-service + cada servicio | 🟡 Diseñado (capa app) |
| Cifrado en tránsito (TLS 1.2+) | Todos los canales | 🟡 Por implementar |
| Rate limiting | API Gateway | 🟡 Por implementar |
| 2FA (TOTP) | Roles de coordinación | 🔴 V2 |

---

## Cumplimiento normativo

| Norma | Aplicación | Cómo se cumple |
|-------|-----------|----------------|
| Ley 1581 de 2012 (Habeas Data) | PII de aprendices, instructores, empresas | Auditoría de acceso, soft delete con trazabilidad, acceso por scope, secretos protegidos |
| Ley 1273 de 2009 (Delitos informáticos) | Acceso no autorizado | RBAC, auditoría append-only, autenticación fuerte |
| OWASP Top 10 (2021) | Toda la aplicación | Access control (E-01/E-02/I-02), inyección (T-03), fallos criptográficos y de configuración (S-01 secretos) |

---

## Acciones pendientes (priorizadas)

| # | Acción | Responsable | Prioridad |
|---|--------|-------------|-----------|
| 1 | Sacar `.env.*` de git, rotar credenciales versionadas | DevOps / Arquitectura | **Crítica** |
| 2 | Definir proveedor de Secret Manager | Arquitectura | Alta |
| 3 | Restringir permisos de BD por módulo (sin superusuario compartido) | DevOps | Alta |
| 4 | Diseñar rate limiting y TLS en el gateway (capa app) | DevOps | Media |
| 5 | Diseñar rotación de claves RSA (proceso operativo) | Seguridad | Media |
| 6 | Especificar 2FA TOTP para V2 | Seguridad | Baja (V2) |

## Referencias

- [cross-cutting.md](./cross-cutting.md) — controles transversales (IAM, auditoría, idempotencia)
- [ADR-003](./decisions/records/ADR-003-object-storage.md) — URLs firmadas (I-04) · [ADR-004](./decisions/records/ADR-004-status-parametrization-and-audit-standard.md) — auditoría (T-02, R-01)
- [00-governance/security-rules.md](../00-governance/security-rules.md) — reglas documentales de seguridad
- [10-devops/environments.md](../10-devops/environments.md) — `.env` por ambiente y advertencia de secretos
- OWASP Top 10: https://owasp.org/www-project-top-ten/
- Ley 1581 de 2012: https://www.funcionpublica.gov.co/eva/gestornormativo/norma.php?i=49981
