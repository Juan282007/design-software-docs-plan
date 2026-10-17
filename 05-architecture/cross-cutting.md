# Preocupaciones transversales

> Estado: 🟡 En progreso | Última actualización: 2026-08-01
> Autor: Jesús Ariel González Bonilla (PM + Arquitecto) | Equipo: Arquitectura

Preocupaciones que atraviesan a los 9 microservicios y que, por tanto, se estandarizan una sola vez en lugar de resolverse servicio por servicio. Complementa a [overview.md](./overview.md), [pattern-guide.md](./pattern-guide.md) y a los ADR de la sección.

> **Estado real:** hoy solo existe la **capa de datos** (`*-db`, Liquibase + PostgreSQL 16, un schema por módulo). Varias de estas preocupaciones ya tienen soporte en el modelo de datos (columnas de auditoría, catálogo de estados, `TIMESTAMPTZ`); otras (RBAC en runtime, observabilidad, broker) pertenecen a la **capa de aplicación aún no construida** y se documentan como diseño previsto. Cada apartado marca lo que existe vs lo previsto.

---

## 1. Seguridad — Autenticación y autorización (IAM / RBAC)

- **Autenticación (previsto):** `iam-service` emite un JWT firmado (RS256). Cada servicio lo verifica de forma stateless con la clave pública; solo `iam-service` posee la clave privada. Ver [security-threat-model.md](./security-threat-model.md).
- **Autorización (previsto):** modelo **RBAC por Feature + Scope**. El JWT transporta los `features` pre-calculados; el `scope` (p. ej. `TRAINING_CENTER`, `OWN_*`) restringe las filas visibles. Un coordinador solo opera dentro de su centro.
- **Estado actual (existe):** el modelo de datos de identidad ya reserva los schemas `identity`/`rbac`/`session` y contempla usuarios, roles y features. La verificación en runtime depende de la capa app.
- **Regla transversal:** ningún endpoint (salvo login) opera sin `Authorization: Bearer <JWT>`; el filtro de `scope` en las consultas es obligatorio, no opcional.

## 2. Auditoría (audit-service + estándar por servicio)

Auditoría en dos niveles, según [ADR-004](./decisions/records/ADR-004-status-parametrization-and-audit-standard.md):

- **Auditoría de registro (existe en el modelo):** toda tabla transaccional lleva `created_at/created_by`, `updated_at/updated_by`, `deleted_at/deleted_by`, `is_active` y `row_version`. El **borrado es lógico** (soft delete): se preserva la historia y se permite recuperación. Las acciones automáticas del sistema usan un **UUID de actor reservado**.
- **Auditoría de eventos (previsto):** `audit-service` es un consumidor universal que se suscribe a **todos los topics** (fan-out, ver [ADR-001](./decisions/records/ADR-001-message-broker.md)) y persiste un log **append-only** (`audit_record`, solo INSERT; sin UPDATE/DELETE ni endpoint de escritura externo). El `event_id` es único → sirve además como llave de idempotencia (ver §6).
- **Motivación normativa:** trazabilidad exigida por la Ley 1581/2012 y por el estatuto SENA; se debe poder responder "quién hizo qué y cuándo".

## 3. Observabilidad (previsto)

Pertenece a la capa app; hoy no hay servicios emitiendo telemetría.

- **Logs estructurados** en JSON con campos mínimos: `timestamp`, `level`, `service`, `request_id`, `event`. **PII nunca en logs**, solo identificadores (ver §Manejo de errores y el threat model I-01).
- **Correlación:** cada request lleva un `X-Request-ID`/`trace_id` propagado entre servicios y hacia los eventos, de modo que un flujo multi-salto (crear → validar → publicar horario) sea reconstruible.
- **Métricas RED** (Rate, Errors, Duration) por endpoint y **trazas distribuidas** para los flujos async.
- **Nivel de datos (existe):** el historial de Liquibase (`databasechangelog`, aislado por módulo) es la fuente de verdad observable del estado del esquema por módulo.

## 4. Gestión de estados parametrizables (ADR-004)

Patrón transversal para no dispersar los estados de negocio como `VARCHAR + CHECK` inline:

- **Existe en el modelo:** cada bounded context implementa en su propia BD las entidades `status_category` (parametriza tipos de estado), `status` (valores) y opcionalmente `status_transition` (transiciones gobernadas). Los agregados con estado de negocio referencian `status_id`.
- **Replicado por servicio, no global:** el patrón se copia en cada schema para no violar "DB por servicio" ni la propiedad de datos. La duplicación es **estructural intencional**, no de datos.
- **Tres ejes ortogonales, no confundir:**
  1. Ciclo de vida del **registro** (técnico): `is_active` + `deleted_at`.
  2. Estado de **negocio** (parametrizable): `status_id`.
  3. Enum técnico **cerrado** (inmutable): `VARCHAR + CHECK` (p. ej. `document_type`, `channel`). Estos **no** migran al catálogo.
- **Transiciones:** las máquinas de estado se validan contra `status_transition` (parametrizable) y se ligan a features de RBAC, en vez de hardcodearse.

## 5. Manejo de errores (previsto)

- **Formato de error estándar** para todas las APIs: código de error de negocio, mensaje seguro y `trace_id` para correlación interna. **Sin stack traces ni detalles internos** en la respuesta (threat model I-05).
- **Errores de negocio como invariantes de dominio:** el dominio lanza errores tipados (p. ej. "horario ya publicado") que la capa de aplicación traduce al formato estándar; los controladores no contienen lógica de negocio.
- **Respuestas genéricas en autenticación** para no filtrar información (login 401 idéntico para usuario inexistente; reset siempre 202).
- **Resiliencia entre servicios:** timeout 5 s, circuit breaker y retry con backoff en las llamadas síncronas; DLQ por consumer en las asíncronas (ver [pattern-guide.md](./pattern-guide.md) §Resiliencia).

## 6. Idempotencia de eventos

- **Regla:** todo consumidor de eventos debe ser **idempotente**. El transporte garantiza **at-least-once** (ADR-001), por lo que un mismo evento puede llegar más de una vez (reintentos, replay).
- **Mecanismo:** deduplicación por `event_id` único. En `audit-service` el `event_id` es UNIQUE en `audit_record` (un evento repetido no crea un segundo registro); los demás consumidores mantienen su propia marca de "evento ya procesado".
- **Producción confiable:** para eventos críticos (p. ej. publicación de horario) se usa el **patrón Outbox** en el servicio productor, de modo que la escritura de estado y la emisión del evento sean atómicas y el envío sea recuperable ante caída del broker.
- **Orden:** cuando el orden importa (eventos de una misma ficha), se enruta por `ficha_id` para preservar el orden por partición lógica (ADR-001).

## 7. Internacionalización y zona horaria

- **Idioma:** el **dominio y la UI van en español** (SENA); el **código, nombres técnicos, eventos y contratos van en inglés** (HALT-DB-NAMING). Los eventos son contratos de integración, por eso su naming es inglés: `<context>.<entity>.<action>` en pasado.
- **Zona horaria (existe en el modelo):** todos los timestamps se almacenan como **`TIMESTAMPTZ`** (UTC en reposo). La **presentación** se hace en **hora de Colombia (America/Bogotá, UTC−5, sin horario de verano)**. La conversión es responsabilidad de la capa de presentación, no del almacenamiento.
- **Implicación operativa:** los SLO horarios (p. ej. criticidad de `scheduling-service` 06:00–22:00) y las reglas de negocio ("no hay clases domingos ni festivos", jornada lunes a sábado) se interpretan siempre en hora Colombia, aunque el dato persista en UTC.

---

## Matriz de responsabilidad transversal

| Preocupación | Dueño | Estado hoy |
|--------------|-------|-----------|
| Autenticación / emisión JWT | iam-service | Modelo de datos; runtime previsto |
| Autorización RBAC (Feature + Scope) | iam-service + cada servicio | Modelo de datos; runtime previsto |
| Auditoría de registro (soft delete + `_by`) | Cada servicio (estándar ADR-004) | ✅ En el modelo |
| Auditoría de eventos (append-only) | audit-service | Previsto (depende del broker) |
| Estados parametrizables | Cada servicio (estándar ADR-004) | ✅ En el modelo |
| Observabilidad (logs, trazas, métricas) | Todos los servicios | Previsto (capa app) |
| Idempotencia de eventos | Cada consumidor | Previsto (capa app) |
| Zona horaria (TIMESTAMPTZ → hora Colombia) | Datos + presentación | ✅ En el modelo (persistencia) |

## Referencias

- [ADR-001](./decisions/records/ADR-001-message-broker.md) · [ADR-004](./decisions/records/ADR-004-status-parametrization-and-audit-standard.md)
- [security-threat-model.md](./security-threat-model.md)
- [06-data/modeling-conventions.md](../06-data/modeling-conventions.md) — estándar de auditoría, estados y tipos temporales
- [02-domain/domain-events.md](../02-domain/domain-events.md) — significado de negocio de los eventos
- [00-governance/security-rules.md](../00-governance/security-rules.md)
