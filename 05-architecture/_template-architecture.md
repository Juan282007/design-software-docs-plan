# Arquitectura — <PROJECT_KEY>

> **PLANTILLA** — Copiar como `architecture.md` y completar.
> Eliminar esta línea antes de hacer commit.

> Estado: 🔴 Pendiente | Última actualización: YYYY-MM-DD
> Autor: Por definir | Equipo: Arquitectura

## 1. Resumen ejecutivo

- **Patrón seleccionado:** [DDD / Hexagonal / Clean / Modular Monolith / CQRS]
- **Tipo de sistema:** [Monolito modular / Microservicios / Serverless / Híbrido]
- **Decisiones clave:**
  - [Decisión 1]
  - [Decisión 2]

## 2. Visión de componentes

### 2.1 Diagrama de alto nivel

```text
[Consumidores] --> [API Gateway] --> [Servicios] --> [Dominio] --> [Persistencia]
```

> Diagrama fuente en `08-uml/diagrams/source/` — exportación en `08-uml/diagrams/exports/`.

### 2.2 Componentes principales

| Componente | Responsabilidad | Tecnología | Interface | Dependencias |
|------------|-----------------|------------|-----------|--------------|
| API Gateway | Enrutamiento, rate limit, auth | | HTTP/REST | Backend |
| Backend API | Casos de uso, orquestación | | HTTP/REST | DB, Cache |
| Workers | Procesamiento asíncrono | | Queue | DB |
| Frontend | UI | | HTTPS | Backend API |
| Persistencia | Almacenamiento transaccional | Ver db-design.md | SQL/NoSQL | — |

## 3. Flujos principales

### 3.1 Flujo síncrono

```text
Usuario → Frontend → API Gateway → Backend → Dominio → DB
```

### 3.2 Flujo asíncrono

```text
Evento → Cola → Worker → Dominio → DB / Servicio externo
```

### 3.3 Flujo de autenticación

```text
[Describir: login, emisión de token, refresh, logout]
```

## 4. Contratos de interfaz

### APIs externas (públicas)

| Endpoint | Método | Auth | Propósito | SLA |
|----------|--------|------|-----------|-----|
| /api/v1/<recurso> | GET/POST | Bearer JWT | | p95 < 300ms |

> Contrato completo en `07-api/contracts/openapi/`.

### Eventos publicados

| Evento | Productor | Consumidores | Garantía |
|--------|-----------|--------------|----------|
| | | | At-least-once |

## 5. Restricciones técnicas (NFR)

| NFR | Objetivo | Estrategia |
|-----|----------|------------|
| Latencia p95 | < [X]ms | Cache, índices |
| Disponibilidad | [99.9%] | Réplicas, autoescalado |
| Observabilidad | 100% con trace | OpenTelemetry |

> Detalle completo en [04-requirements/non-functional.md](../04-requirements/non-functional.md).

## 6. Decisiones arquitectónicas

| # | Decisión | Alternativas | Trade-off | ADR |
|---|----------|-------------|-----------|-----|
| 1 | | | | ADR-001 |

## 7. Seguridad transversal

- **AuthN:** [OAuth2 / JWT]
- **AuthZ:** [RBAC / ABAC]
- **Secrets:** [Vault / SSM] — nunca en variables de entorno planas
- **Cifrado:** TLS en tránsito; cifrado en reposo para PII

> Detalle en [security-threat-model.md](./security-threat-model.md).

## 8. Riesgos técnicos

| Riesgo | Probabilidad | Impacto | Mitigación |
|--------|--------------|---------|------------|
| | Baja/Media/Alta | Bajo/Medio/Alto | |

## Referencias

- [Pattern Guide](./pattern-guide.md)
- [Data Model](../06-data/data-model.md)
- [DB Design](../06-data/db-design.md)
- [Security Threat Model](./security-threat-model.md)
- [NFR](../04-requirements/non-functional.md)
- [ADRs](./decisions/README.md)
