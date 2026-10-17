# iam-api

> Estado: 🟡 En progreso | Última actualización: 2026-08-01
> Autor: Jesús Ariel González Bonilla (PM + Arquitecto) | Equipo: Arquitectura

> **Diseño previsto — no implementado.** La capa de aplicación aún no se construye y no se ha elegido lenguaje ni framework de backend. Este documento describe el contrato y las responsabilidades a nivel de protocolo, agnóstico de la tecnología de implementación.

## Tipo de componente

- [x] `-api` — REST API sincrónica
- [ ] `-worker` — Consumidor de eventos / cola
- [ ] `-workflow` — Orquestación de pasos con compensaciones
- [ ] `-scheduler` — Tarea periódica
- [ ] `-notifier` — Envío de notificaciones salientes
- [ ] `-gateway` — Punto de entrada / proxy

## Responsabilidad

Expone los endpoints de autenticación (login, refresh, logout) y de administración de usuarios, roles, features y scopes; emite el JWT con features y scope pre-calculados que el resto de servicios consume para autorizar.

## Tecnologías

> Ninguna decisión de runtime/framework está tomada; solo se fijan los componentes de infraestructura reales y transversales de la plataforma.

| Capa | Tecnología | Versión |
|------|-----------|---------|
| Runtime | Agnóstico — cualquier backend (a definir) | — |
| Framework | A definir | — |
| Base de datos | PostgreSQL — `iam_db` | 16 |
| Transporte de eventos | Broker AMQP (RabbitMQ, [ADR-001](../../../../../05-architecture/decisions/records/ADR-001-message-broker.md)) | — |

Publica eventos de dominio (`iam.user.created`, `iam.role.assigned`, `iam.session.started`…) al broker; ver [event-catalog.md](../../../../event-catalog.md).

## Variables de entorno requeridas

> Nombres genéricos e ilustrativos; el naming exacto lo fija el stack cuando se elija.

| Variable | Descripción | Ejemplo |
|----------|-------------|---------|
| `SERVICE_PORT` | Puerto de escucha del API | `8080` |
| `DB_URL` | Cadena de conexión a `iam_db` | `postgresql://host:5432/iam_db` |
| `BROKER_URL` | URL del broker AMQP | `amqp://host:5672` |
| `JWT_SIGNING_KEY` | Clave para firmar el access token | `(secreto)` |
| `JWT_ACCESS_TTL` | TTL del access token | `15m` |
| `JWT_REFRESH_TTL` | TTL del refresh token | `7d` |
| `LOG_LEVEL` | Nivel de log | `info` |

## Escalado, salud y observabilidad

- **Escalado:** sin estado en memoria (el estado de sesión vive en `refresh_token`); escala horizontalmente detrás de un balanceador. La verificación del JWT en servicios downstream no requiere llamar a `iam-api`.
- **Salud:** expone sondas de *liveness* y *readiness* (readiness incluye conectividad a `iam_db` y al broker).
- **Observabilidad:** logs estructurados con `trace_id`, métricas de latencia/errores por endpoint y trazas distribuidas, según el estándar transversal en [13-operations/observability.md](../../../../../13-operations/observability.md).

## Contrato

Ver [contract.md](./contract.md)

## Referencias

- Modelo de datos: [../../data-model.md](../../data-model.md)
- Diseño RBAC (features, scopes, matriz de roles): [../../rbac-design.md](../../rbac-design.md)
- Catálogo de eventos: [../../../../event-catalog.md](../../../../event-catalog.md)
