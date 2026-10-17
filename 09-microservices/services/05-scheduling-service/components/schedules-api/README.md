# schedules-api

> Estado: 🟡 En progreso | Última actualización: 2026-08-01
> Autor: Jesús Ariel González Bonilla (PM + Arquitecto) | Equipo: Arquitectura

> **Diseño previsto — no implementado.** La capa de aplicación aún no se construye y no se ha elegido lenguaje ni framework de backend. Este documento describe responsabilidades y contrato a nivel de protocolo, agnóstico de la tecnología.

## Tipo de componente

- [x] `-api` — REST API sincrónica
- [ ] `-worker` — Consumidor de eventos / cola
- [ ] `-workflow` — Orquestación de pasos con compensaciones
- [ ] `-scheduler` — Tarea periódica
- [ ] `-notifier` — Envío de notificaciones salientes
- [ ] `-gateway` — Punto de entrada / proxy

## Responsabilidad

Expone la API REST del servicio de horarios: CRUD de borradores, gestión de sesiones de clase y franjas, consulta y resolución de conflictos, y las transiciones de estado del horario (validar, publicar, archivar); actúa además como relay del Outbox para publicar `scheduling.schedule.published`.

## Tecnologías

| Capa | Tecnología | Versión |
|------|-----------|---------|
| Runtime | Agnóstico — cualquier backend (a definir) | — |
| Framework | A definir | — |
| Base de datos | PostgreSQL — `scheduling_db` | 16 |
| Transporte de eventos | Broker AMQP (RabbitMQ, [ADR-001](../../../../../05-architecture/decisions/records/ADR-001-message-broker.md)) | — |

## Variables de entorno requeridas

> Nombres genéricos e ilustrativos; el naming exacto lo fija el stack cuando se elija.

| Variable | Descripción | Ejemplo |
|----------|-------------|---------|
| `SERVICE_PORT` | Puerto de escucha del API | `8080` |
| `DB_URL` | Cadena de conexión a `scheduling_db` | `postgresql://host:5432/scheduling_db` |
| `BROKER_URL` | URL del broker AMQP (publicación de eventos / relay Outbox) | `amqp://host:5672` |
| `IAM_JWKS_URL` | Endpoint para validar la firma del JWT emitido por `iam-service` | `http://iam/.well-known/jwks.json` |
| `ACADEMIC_SERVICE_URL` | Base URL de `academic-management-service` (validación de ficha) | `http://academic:8080` |
| `OUTBOX_POLL_INTERVAL` | Intervalo de sondeo del relay de Outbox | `2s` |
| `LOG_LEVEL` | Nivel de log | `info` |

## Escalado, salud y observabilidad

- **Escalado:** API sin estado; escala horizontalmente. El relay del Outbox debe ejecutarse con concurrencia controlada (líder único o bloqueo por fila) para no duplicar publicaciones; la deduplicación final la garantiza `event_id`.
- **Salud:** sondas de *liveness* y *readiness* (readiness verifica `scheduling_db` y el broker).
- **Observabilidad:** logs estructurados con `trace_id`, métricas por endpoint y latencia de publicación del Outbox, según [13-operations/observability.md](../../../../../13-operations/observability.md). SLO de validación de horario completo < 2 s.

## Contrato

Ver [contract.md](./contract.md)

## Referencias

- Modelo de datos del servicio: [../../data-model.md](../../data-model.md)
- Eventos del servicio: [../../events.md](../../events.md)
- RBAC (features y scopes): [../../../01-iam-service/rbac-design.md](../../../01-iam-service/rbac-design.md)
