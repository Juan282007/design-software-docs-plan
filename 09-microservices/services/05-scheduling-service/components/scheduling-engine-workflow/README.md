# scheduling-engine-workflow

> Estado: 🟡 En progreso | Última actualización: 2026-08-01
> Autor: Jesús Ariel González Bonilla (PM + Arquitecto) | Equipo: Arquitectura

> **Diseño previsto — no implementado.** La capa de aplicación aún no se construye y no se ha elegido lenguaje ni framework de backend. Este documento describe la orquestación a nivel de protocolo (pasos, eventos), agnóstico de la tecnología.

## Tipo de componente

- [ ] `-api` — REST API sincrónica
- [ ] `-worker` — Consumidor de eventos / cola
- [x] `-workflow` — Orquestación de pasos con compensaciones
- [ ] `-scheduler` — Tarea periódica
- [ ] `-notifier` — Envío de notificaciones salientes
- [ ] `-gateway` — Punto de entrada / proxy

## Responsabilidad

Orquesta como saga la generación de un horario: valida la ficha, resuelve las disponibilidades de instructores y ambientes contra los read models locales, crea las sesiones de clase del borrador y compensa (revierte) los pasos ya aplicados si algún paso falla.

## Tecnologías

| Capa | Tecnología | Versión |
|------|-----------|---------|
| Runtime | Agnóstico — cualquier backend (a definir) | — |
| Framework | A definir (orquestador de saga a elegir) | — |
| Base de datos | PostgreSQL — `scheduling_db` (read models + sesiones) | 16 |
| Transporte de eventos | Broker AMQP (RabbitMQ, [ADR-001](../../../../../05-architecture/decisions/records/ADR-001-message-broker.md)) | — |

Los read models locales (instructores/competencias y ambientes/disponibilidad) que este componente consulta se pueblan por eventos según [ADR-002](../../../../../05-architecture/decisions/records/ADR-002-scheduling-read-models.md).

## Variables de entorno requeridas

> Nombres genéricos e ilustrativos; el naming exacto lo fija el stack cuando se elija.

| Variable | Descripción | Ejemplo |
|----------|-------------|---------|
| `SERVICE_PORT` | Puerto de escucha (API interna / health) | `8080` |
| `DB_URL` | Cadena de conexión a `scheduling_db` | `postgresql://host:5432/scheduling_db` |
| `BROKER_URL` | URL del broker AMQP | `amqp://host:5672` |
| `ACADEMIC_SERVICE_URL` | Base URL de `academic-management-service` (única dependencia síncrona) | `http://academic:8080` |
| `STEP_TIMEOUT` | Timeout por paso de la saga | `10s` |
| `LOG_LEVEL` | Nivel de log | `info` |

## Escalado, salud y observabilidad

- **Escalado:** cada ejecución de saga es independiente por `schedule_id`; escala horizontalmente. El estado de la saga se persiste para permitir reintentos y compensaciones idempotentes.
- **Salud:** sondas de *liveness* y *readiness* (readiness verifica `scheduling_db`, broker y disponibilidad de `academic-management-service`).
- **Observabilidad:** cada paso emite logs estructurados con `trace_id` y `schedule_id`; métricas de duración por paso y tasa de compensación, según [13-operations/observability.md](../../../../../13-operations/observability.md).

## Contrato

Ver [contract.md](./contract.md)

## Referencias

- Modelo de datos del servicio: [../../data-model.md](../../data-model.md)
- Eventos del servicio: [../../events.md](../../events.md)
- ADR-002 (read models): [../../../../../05-architecture/decisions/records/ADR-002-scheduling-read-models.md](../../../../../05-architecture/decisions/records/ADR-002-scheduling-read-models.md)
