# Guía de Patrones — SENA Gestión de Horarios

> Fase: 03-Design | Agente: A06 | Estado: 🟡 Borrador
> Fecha: 2026-06-17
> Prerequisitos: [discovery-brief.md](../03-product/discovery-brief.md) · [problem-framing.md](../03-product/problem-framing.md)

## Patrón seleccionado

**Arquitectura de Microservicios con DDD (Domain-Driven Design) y Arquitectura Hexagonal por servicio**

### Justificación

| Fuerza del problema | Por qué justifica este patrón |
|--------------------|-------------------------------|
| 9 dominios de negocio claramente delimitados (M1-M9) | DDD Bounded Contexts mapean 1:1 con los dominios |
| Equipos independientes por dominio | Cada microservicio puede tener equipo y ciclo de release propio |
| Picos de carga distintos por capacidad (motor de horarios vs. catálogos) | Escala independiente por servicio |
| Lógica de negocio compleja en scheduling y monitoring | Hexagonal separa lógica de dominio de infraestructura |
| Múltiples formas de comunicación: REST sync + eventos async | Puertos y adaptadores permiten intercambiar implementaciones |

### Por qué no otras alternativas

| Alternativa | Por qué se descarta |
|-------------|---------------------|
| Monolito modular | Escalado independiente imposible; equipo demasiado acoplado en un repo |
| Microservicios sin DDD (por módulo técnico) | Fronteras basadas en capas técnicas producen servicios anémicos y alto acoplamiento |
| SOA tradicional | ESB centralizado = punto único de fallo y cuello de botella de gobernanza |
| Serverless puro | Complejidad de estado distribuido (motor de horarios) no es adecuada para Lambda/Functions |

---

## Estructura de capas — Arquitectura Hexagonal por microservicio

```
┌─────────────────────────────────────────────────────────┐
│                    ADAPTADORES DE ENTRADA                │
│         (HTTP Controllers, Message Consumers)            │
│                   [Capa de Interfaz]                     │
└──────────────────────┬──────────────────────────────────┘
                       │ llama a
┌──────────────────────▼──────────────────────────────────┐
│                   CASOS DE USO / APLICACIÓN             │
│         (Application Services, Command Handlers)         │
│                  [Capa de Aplicación]                    │
└──────────────────────┬──────────────────────────────────┘
                       │ usa interfaces definidas en
┌──────────────────────▼──────────────────────────────────┐
│                    DOMINIO PURO                          │
│      (Entities, Value Objects, Domain Services,          │
│       Aggregates, Domain Events)                         │
│                   [Capa de Dominio]                      │
└──────────────────────┬──────────────────────────────────┘
                       │ implementado por
┌──────────────────────▼──────────────────────────────────┐
│                ADAPTADORES DE SALIDA                     │
│    (Repositorios PostgreSQL, Clientes HTTP, Producers)   │
│               [Capa de Infraestructura]                  │
└─────────────────────────────────────────────────────────┘
```

### Paquetes de referencia por capa

```
<service-name>/
  src/
    domain/          ← entidades, value objects, interfaces de repositorio
    application/     ← casos de uso, DTOs de entrada/salida, validaciones
    infrastructure/
      persistence/   ← implementaciones de repositorio (PostgreSQL)
      http/          ← clientes HTTP hacia otros servicios
      messaging/     ← producers/consumers de eventos
    api/             ← controllers REST, middleware de autenticación
```

---

## Reglas de dependencia

### Regla fundamental — Inversión de dependencias

> El dominio no depende de infraestructura. La infraestructura implementa las interfaces definidas en el dominio.

```
Dominio ←──────── Aplicación ←──────── Infraestructura / API
  ↑                  ↑                       ↑
Sin imports       Solo imports             Solo imports
de BD o HTTP      del dominio            del dominio y aplicación
```

**Violación detectada en PR → bloquear merge.** Ningún archivo de `domain/` puede tener `import` de `infrastructure/` o de librerías de BD/HTTP.

### Regla de comunicación entre servicios

| Tipo de comunicación | Cuándo usar | Implementación |
|---------------------|-------------|----------------|
| REST sync | Cuando el llamante necesita la respuesta para continuar | HTTP con timeout 5 s, circuit breaker |
| Evento async | Cuando el llamante no necesita respuesta inmediata | Broker de mensajes (Kafka / RabbitMQ) |
| Read model / caché | Para queries de alta frecuencia sobre datos de otro servicio | Redis con TTL; datos eventualmente consistentes |

### Regla de dependencias sync — Máximo 2 por servicio

> Ningún servicio puede tener más de 2 dependencias síncronas hacia otros servicios (excluyendo `iam-service` para autenticación).

Excepción actual: `scheduling-service` depende síncronamente de `actors-service`, `training-environment-service` y `academic-management-service` → **ver ADR-002** (resolución con read models).

### Regla de propiedad de datos

> Cada entidad de dominio tiene exactamente **un** servicio propietario. Ningún otro servicio puede escribir en esa entidad directamente. Los demás consumen el dato vía API o evento.

---

## Bounded Contexts — Mapa de límites

```
┌────────────────────┐     ┌──────────────────────────┐
│    iam-service     │────▶│  reference-data-service  │
│  (Identidad, Auth) │     │  (Jerarquía + Catálogos)  │
└─────────┬──────────┘     └─────────────┬────────────┘
          │ JWT                           │ datos ref
          ▼                               ▼
┌─────────────────────────────────────────────────────┐
│                   scheduling-service                 │
│           (Motor central de asignación)              │
│                                                     │
│  consume de: actors-service, training-environment   │
│  academic-management; publica: scheduling-events    │
└──────────┬─────────────────────┬────────────────────┘
           │                     │
     sync  │               async │ eventos
           ▼                     ▼
┌──────────────────┐   ┌─────────────────────────────┐
│  actors-service  │   │    monitoring-service        │
│  (M7: Actores)   │   │    (M9: Seguimiento)         │
└──────────────────┘   └─────────────────────────────┘
┌──────────────────────────────┐   ┌─────────────────┐
│  training-environment-service│   │  audit-service  │
│  (M3: Ambientes)             │   │  (Auditoría)    │
└──────────────────────────────┘   └─────────────────┘
┌─────────────────────────────┐    ┌─────────────────┐
│  academic-management-service│    │ document-service │
│  (M5+M6: Programas+Fichas)  │    │  (Documentos)   │
└─────────────────────────────┘    └─────────────────┘
```

---

## Ejemplos concretos de aplicación

### Ejemplo 1 — Caso de uso: Publicar horario

```
[API Controller] POST /schedules/{id}/publish
    │
    ▼ llama a
[PublishScheduleUseCase]  ← Capa Aplicación
    │
    ├── ScheduleRepository.findById(id)        ← Puerto (dominio)
    │       └── PostgresScheduleRepository     ← Adaptador (infra)
    │
    ├── schedule.validate()                    ← Lógica de dominio pura
    │       └── Detecta conflictos internos
    │
    ├── ConflictValidatorPort.validate(schedule)  ← Puerto (dominio)
    │       └── ConflictValidatorWorkerAdapter     ← Adaptador (infra → async)
    │
    ├── schedule.publish()                     ← Agrega SchedulePublishedEvent
    │
    ├── ScheduleRepository.save(schedule)      ← Puerto → Adaptador
    │
    └── EventPublisher.publish(SchedulePublishedEvent)  ← Puerto → Adaptador (broker)
```

### Ejemplo 2 — Entidad de dominio sin dependencias de infraestructura

```typescript
// domain/entities/Schedule.ts
export class Schedule {
  private status: ScheduleStatus;
  private sessions: ClassSession[];
  private domainEvents: DomainEvent[] = [];

  publish(): void {
    if (this.status !== ScheduleStatus.DRAFT) {
      throw new ScheduleAlreadyPublishedError(this.id);
    }
    this.status = ScheduleStatus.PUBLISHED;
    this.domainEvents.push(new SchedulePublishedEvent(this.id, this.fichaId));
  }

  // Sin imports de PostgreSQL, Kafka, ni HTTP
}
```

### Ejemplo 3 — Puerto e implementación

```typescript
// domain/ports/ScheduleRepository.ts  ← dominio define la interfaz
export interface ScheduleRepository {
  findById(id: ScheduleId): Promise<Schedule | null>;
  save(schedule: Schedule): Promise<void>;
}

// infrastructure/persistence/PostgresScheduleRepository.ts  ← infra implementa
export class PostgresScheduleRepository implements ScheduleRepository {
  constructor(private readonly db: DatabaseConnection) {}
  // ...implementación con SQL
}
```

---

## Aplicación de SOLID

| Principio | Aplicación en este sistema |
|-----------|---------------------------|
| **S** — Responsabilidad única | Cada caso de uso en su propio archivo; cada entidad solo conoce su propio estado |
| **O** — Abierto/cerrado | Nuevas reglas de conflicto se agregan como `ConflictRule` sin modificar el validador base |
| **L** — Sustitución de Liskov | Los adaptadores de repositorio son intercambiables (Postgres en prod, in-memory en tests) |
| **I** — Segregación de interfaces | `ScheduleRepository` tiene solo los métodos que usa su caso de uso; no hereda un CRUD genérico |
| **D** — Inversión de dependencias | El dominio define las interfaces; la infraestructura las implementa (ver Ejemplo 3) |

---

## Patrones de resiliencia por servicio

| Patrón | Aplica en | Implementación |
|--------|-----------|----------------|
| Circuit Breaker | Llamadas sync entre servicios | Resilience4j / equivalente |
| Retry con backoff exponencial | Producers de eventos ante fallo del broker | 3 reintentos: 1 s, 4 s, 16 s |
| Dead Letter Queue | Consumers de eventos | `<topic>.dlq` por cada consumer |
| Timeout | Cualquier llamada HTTP externa | 5 s máximo; configurado en cliente |
| Idempotency Key | Operaciones de escritura críticas | `event_id` como PK en `audit-service`; `schedule_id` en publicación |

---

## Checklist de conformidad — Por pull request

- [ ] Ningún archivo de `domain/` tiene imports de `infrastructure/`, `pg`, `axios`, `kafka`
- [ ] Cada caso de uso tiene su propio archivo en `application/`
- [ ] Las interfaces de repositorio están definidas en `domain/ports/`, no en `infrastructure/`
- [ ] Los controladores REST no contienen lógica de negocio (solo deserialización + llamada al caso de uso)
- [ ] Los eventos de dominio se emiten desde las entidades o servicios de dominio, no desde los controladores
- [ ] El servicio no tiene más de 2 dependencias síncronas a otros servicios (excluye iam-service)

---

## Riesgos del patrón seleccionado

| Riesgo | Probabilidad | Mitigación |
|--------|-------------|------------|
| Sobre-ingeniería en servicios de bajo volumen (reference-data, audit) | Media | Los servicios simples pueden omitir subcapas que no agregan valor; el patrón se aplica en su forma mínima |
| Complejidad de debugging en flujos async multi-servicio | Alta | Correlation ID + tracing distribuido (OpenTelemetry) desde sprint 1 |
| Deriva de las fronteras de bounded context con el tiempo | Media | ADRs para toda decisión que cruce fronteras + revisión de `data-ownership-matrix.md` en cada sprint |
| Curva de aprendizaje del equipo con DDD | Media | Sesión de kick-off técnico + este `pattern-guide.md` como referencia permanente |
