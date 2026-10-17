# Arquitectura del Sistema — SENA Gestión de Horarios

> Fase: 03-Design | Agente: A06 | Estado: 🟡 Borrador
> Fecha: 2026-06-17
> Prerequisitos: [pattern-guide.md](./pattern-guide.md) · [discovery-brief.md](../03-product/discovery-brief.md)

## Resumen ejecutivo

El sistema de Gestión de Horarios SENA es una plataforma de microservicios compuesta por **9 servicios independientes** organizados en torno a 9 bounded contexts de negocio. La arquitectura combina comunicación REST síncrona para operaciones de usuario y eventos asincrónicos para propagación de cambios entre servicios. Cada servicio tiene su propia base de datos PostgreSQL.

| Dimensión | Decisión |
|-----------|----------|
| Estilo arquitectural | Microservicios con DDD + Hexagonal por servicio |
| Comunicación sync | REST/HTTP con circuit breaker y timeout 5 s |
| Comunicación async | Broker de mensajes (Kafka / RabbitMQ — ver ADR-001) |
| Persistencia | PostgreSQL por servicio (base de datos propia) |
| Autenticación | JWT Bearer (emitido por `iam-service`) |
| Almacenamiento de objetos | Object Storage S3/MinIO (via `document-service`) |
| Caché | Redis por consumidor, TTL según volatilidad del dato |
| Observabilidad | OpenTelemetry + logs estructurados JSON + métricas RED |

---

## Visión de componentes — C4 Nivel 2 (Contenedores)

```
                           ┌─────────────────────────────┐
                           │         USUARIOS             │
                           │  Coordinador / Instructor /  │
                           │  Aprendiz / Administrador    │
                           └──────────────┬──────────────┘
                                          │ HTTPS
                                          ▼
                           ┌─────────────────────────────┐
                           │         API GATEWAY          │
                           │   Routing · Rate Limiting    │
                           │   SSL Termination · CORS     │
                           └──────────────┬──────────────┘
                                          │
              ┌───────────────────────────┼────────────────────────────┐
              │                           │                            │
              ▼                           ▼                            ▼
  ┌───────────────────┐      ┌───────────────────────┐    ┌───────────────────┐
  │   iam-service     │      │  scheduling-service    │    │  actors-service   │
  │  :8001            │      │  :8005                 │    │  :8006            │
  │  JWT · Roles      │      │  Motor horarios        │    │  Instructores     │
  │  PostgreSQL       │      │  PostgreSQL             │    │  Aprendices       │
  └───────────────────┘      │  + Redis caché          │    │  PostgreSQL       │
                             └────────────┬────────────┘    └───────────────────┘
                                          │ sync
              ┌───────────────────────────┼──────────────────────────┐
              ▼                           ▼                           ▼
  ┌───────────────────┐      ┌────────────────────────┐  ┌────────────────────┐
  │ reference-data-   │      │  training-environment- │  │ academic-management│
  │ service :8002     │      │  service :8004          │  │ -service :8003     │
  │  Catálogos M2+M4  │      │  Ambientes M3           │  │  Programas M5+M6   │
  │  PostgreSQL+Redis │      │  PostgreSQL             │  │  PostgreSQL        │
  └───────────────────┘      └────────────────────────┘  └────────────────────┘

  ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ BROKER DE MENSAJES ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─

  ┌───────────────────┐      ┌────────────────────────┐  ┌────────────────────┐
  │ monitoring-       │      │  document-service      │  │  audit-service     │
  │ service :8008     │      │  :8007                 │  │  :8009             │
  │  KPIs M9          │      │  PDFs · Object Storage │  │  Append-only log   │
  │  PostgreSQL       │      │  PostgreSQL + MinIO/S3 │  │  PostgreSQL        │
  └───────────────────┘      └────────────────────────┘  └────────────────────┘
```

---

## Flujos principales

### Flujo 1 — Crear y publicar un horario

```
Coordinador
  │
  ├─▶ POST /auth/login            [iam-service] → JWT
  │
  ├─▶ GET /academic/fichas?status=active   [academic-management-service]
  │       └─ Retorna fichas activas del centro
  │
  ├─▶ GET /environments/available?...      [training-environment-service]
  │       └─ Retorna ambientes disponibles para franja horaria (< 300 ms)
  │
  ├─▶ GET /actors/instructors/available?... [actors-service]
  │       └─ Retorna instructores disponibles con competencia requerida (< 300 ms)
  │
  ├─▶ POST /schedules                      [scheduling-service]
  │       └─ Crea borrador; estado: BORRADOR
  │
  ├─▶ POST /schedules/{id}/sessions        [scheduling-service]
  │       └─ Agrega sesiones de clase al borrador
  │
  ├─▶ POST /schedules/{id}/validate        [scheduling-service]
  │       └─ Detecta conflictos → responde lista de conflictos o vacío
  │
  └─▶ POST /schedules/{id}/publish         [scheduling-service]
        ├─ Cambia estado a PUBLICADO
        └─ Publica evento: scheduling.schedule.published
                            │
                            ├─▶ [monitoring-service] consume → actualiza estado seguimiento ficha
                            ├─▶ [audit-service] consume → registra evento de auditoría
                            └─▶ [notification-service / notification-worker] → notifica a instructor/aprendiz
```

### Flujo 2 — Consultar horario (instructor)

```
Instructor
  │
  ├─▶ GET /auth/me                         [iam-service] → datos del actor
  │
  └─▶ GET /schedules/instructor/{id}/week?week=2026-W25  [scheduling-service]
        └─ Retorna sesiones de clase del instructor para la semana
```

### Flujo 3 — Seguimiento de ficha

```
Coordinador / Instructor
  │
  ├─▶ POST /monitoring/sessions            [monitoring-service]
  │       └─ Registra sesión de seguimiento (asistencia, avance)
  │
  └─▶ GET /monitoring/fichas/{id}/kpi      [monitoring-service]
        └─ Retorna KPIs: % asistencia, % avance, alertas activas
```

---

## Contratos y responsabilidades de borde

| Servicio | Expone | Consume (sync) | Consume (async) |
|----------|--------|----------------|-----------------|
| `iam-service` | JWT validation, user CRUD | — | — |
| `reference-data-service` | Catálogos, jerarquía institucional | — | — |
| `academic-management-service` | Programas, competencias, fichas | `iam-service` (auth) | — |
| `training-environment-service` | Disponibilidad de ambientes | `iam-service` (auth) | — |
| `actors-service` | Disponibilidad de instructores, datos de aprendices | `iam-service` (auth) | — |
| `scheduling-service` | Motor de horarios, detección de conflictos | `actors-service`, `training-environment-service`, `academic-management-service` | — |
| `document-service` | Generación de PDFs, almacenamiento de archivos | `iam-service` (auth) | `academic-events`, `scheduling-events` |
| `monitoring-service` | KPIs, alertas, seguimiento | `iam-service` (auth) | `scheduling-events`, `academic-events` |
| `audit-service` | (sin API pública) | — | todos los topics (wildcard) |

---

## Requisitos no funcionales

### Rendimiento

| Operación | Objetivo p95 | Justificación |
|-----------|-------------|---------------|
| `GET /environments/available` | < 300 ms | Operación crítica durante creación de horario |
| `GET /instructors/available` | < 300 ms | Operación crítica durante creación de horario |
| `POST /schedules/{id}/validate` | < 2 s | Validación de horario completo (20 sesiones) |
| Publicación de evento | < 100 ms al broker | Desacoplado de la respuesta HTTP al usuario |
| Consulta de horario (instructor/aprendiz) | < 500 ms | UX aceptable en consulta |

### Disponibilidad

| Servicio | SLO Disponibilidad | Horario de criticidad |
|----------|-------------------|----------------------|
| `iam-service` | 99.9 % | 24/7 |
| `scheduling-service` | 99.5 % | 06:00–22:00 hora Colombia |
| `reference-data-service` | 99.5 % | Horas laborales |
| Resto de servicios | 99 % | Horas laborales |

### Seguridad

- Todos los endpoints (excepto `POST /auth/login`) requieren `Authorization: Bearer <JWT>`
- Validación de JWT en cada servicio (stateless); `iam-service` emite, cada servicio verifica
- HTTPS en todos los canales; HTTP Strict Transport Security habilitado en gateway
- No se almacenan datos sensibles en logs (ver [security-rules.md](../00-governance/security-rules.md))

### Observabilidad

- Cada request tiene `X-Request-ID` propagado entre servicios (correlation)
- Logs estructurados en JSON: `timestamp`, `level`, `service`, `request_id`, `event`
- Métricas RED (Rate, Errors, Duration) por endpoint
- Trazas distribuidas con OpenTelemetry; exportación a Jaeger / Tempo

---

## Decisiones de arquitectura — Índice de ADRs

| ADR | Decisión | Estado |
|-----|----------|--------|
| [ADR-001](./decisions/records/ADR-001-message-broker.md) | Selección de broker de mensajes (RabbitMQ) | 🟡 PROPOSED |
| [ADR-002](./decisions/records/ADR-002-scheduling-read-models.md) | Reducción de dependencias síncronas en scheduling-service (read models) | 🟡 PROPOSED |
| [ADR-003](./decisions/records/ADR-003-object-storage.md) | Estrategia de almacenamiento de objetos en document-service (MinIO/S3) | 🟡 PROPOSED |
| [ADR-004](./decisions/records/ADR-004-status-parametrization-and-audit-standard.md) | Parametrización de estados (status_category/status) y estándar de auditoría | 🟡 PROPOSED |

---

## Topología de despliegue

```
┌─────────────────────────────────────────────────────────┐
│                  AMBIENTE DE PRODUCCIÓN                  │
│                                                         │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌────────┐  │
│  │iam-api   │  │sched-api │  │actors-api│  │...     │  │
│  │ x2       │  │ x2       │  │ x2       │  │        │  │
│  └──────────┘  └──────────┘  └──────────┘  └────────┘  │
│                                                         │
│  ┌────────────────────────────────────────────────────┐ │
│  │              Broker de mensajes (cluster)           │ │
│  └────────────────────────────────────────────────────┘ │
│                                                         │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐              │
│  │iam-db    │  │sched-db  │  │actors-db │  ... (1 BD   │
│  │PostgreSQL│  │PostgreSQL│  │PostgreSQL│  por servicio)│
│  └──────────┘  └──────────┘  └──────────┘              │
│                                                         │
│  ┌──────────┐  ┌──────────┐                            │
│  │Redis     │  │MinIO/S3  │                            │
│  │(caché)   │  │(objetos) │                            │
│  └──────────┘  └──────────┘                            │
└─────────────────────────────────────────────────────────┘
```

Rama de despliegue por ambiente: `dev` → `qa` → `main` (producción). Ver [deployment.md](./deployment.md).

---

## Riesgos arquitecturales

| Riesgo | Probabilidad | Mitigación |
|--------|-------------|------------|
| `scheduling-service` con 3 deps síncronas (excede límite) | Confirmado | ADR-002: introducir read models para reducir a ≤ 2 |
| Latencia acumulada en flujos con múltiples saltos de servicio | Media | Medir p95 end-to-end desde sprint 1; budget: 1.5 s total para creación de sesión |
| Split-brain si el broker de mensajes falla durante publicación | Media | Outbox pattern en `scheduling-service` para garantizar at-least-once delivery |
| Complejidad operacional de 9 bases de datos independientes | Alta | Automigración gestionada por cada servicio (Flyway / Liquibase); no hay DBA centralizado |
