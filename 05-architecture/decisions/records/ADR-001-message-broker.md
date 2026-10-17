# ADR-001: Selección del broker de mensajes

**Estado:** PROPOSED
**Fecha:** 2026-06-17
**Autores:** Jesús Ariel González Bonilla (Arquitecto)
**Equipos involucrados:** Arquitectura, DevOps, Backend

---

## Contexto

La arquitectura de microservicios usa comunicación asíncrona por eventos para los caminos no críticos (auditoría, seguimiento, notificaciones). El [event-catalog.md](../../../09-microservices/event-catalog.md) define 28 eventos publicados por 8 servicios y consumidos principalmente por `audit-service` y `monitoring-service`.

Requisitos del broker:
- **At-least-once delivery** con soporte de idempotencia (los consumers ya manejan `event_id` único)
- **Dead Letter Queue** por consumer
- **Fan-out**: `audit-service` consume todos los topics (wildcard o suscripción múltiple)
- **Orden por partición**: eventos de una misma ficha deben procesarse en orden
- Operación viable para un equipo pequeño sin SRE dedicado
- Compatible con el patrón Outbox para garantizar entrega desde `scheduling-service`

Hay que decidir el broker antes de iniciar la fase de Build, porque los contratos de eventos y los adaptadores de mensajería dependen de él.

## Decisión

Se decide usar **RabbitMQ** como broker de mensajes para el MVP.

RabbitMQ ofrece el balance correcto para este proyecto: enrutamiento flexible (exchanges topic para el fan-out de auditoría), DLQ nativa, confirmaciones de entrega (publisher confirms + consumer ack) y una curva operativa moderada. El volumen esperado (decenas de eventos por minuto en un centro, cientos a nivel nacional) está muy por debajo del punto donde Kafka aporta ventajas de throughput.

El patrón de consumo se modela así:
- Un **topic exchange** por dominio de eventos (`scheduling-events`, `academic-events`, etc.)
- `audit-service` se suscribe con routing key `#` (wildcard) a un exchange de fan-out global
- Orden por ficha se logra con routing keys que incluyen `ficha_id` y colas con consumidor único por partición lógica cuando el orden importa

## Consecuencias

### Positivas

- Enrutamiento topic flexible cubre el fan-out de auditoría sin lógica adicional
- DLQ y reintentos nativos; menos código de resiliencia propio
- Operación más simple que Kafka para un equipo sin SRE dedicado
- Amplio soporte de clientes en Node.js y Python (stacks candidatos)
- Management UI incluida facilita debugging en DEV/QA

### Negativas / Trade-offs

- Menor throughput máximo que Kafka (irrelevante al volumen actual, relevante si el sistema escala a todo el SENA nacional con alto volumen)
- La retención de mensajes no es un log persistente como Kafka; si se requiere re-procesar histórico de eventos, RabbitMQ no es ideal
- El orden estricto requiere diseño cuidadoso de colas (consumidor único por partición)

### Riesgos

- Si el alcance crece a procesamiento de streams analíticos en tiempo real, habría que migrar a Kafka. Mitigación: los adaptadores de mensajería se aíslan tras un puerto (`EventPublisher` / `EventConsumer`) en la capa hexagonal, lo que permite cambiar la implementación sin tocar el dominio.

## Alternativas consideradas

| Alternativa | Por qué se descartó |
|-------------|---------------------|
| Apache Kafka | Sobredimensionado para el volumen actual; mayor complejidad operativa (Zookeeper/KRaft, gestión de particiones, retención); valor de su log persistente no se necesita en el MVP |
| AWS SQS + SNS | Acopla el diseño a AWS desde el MVP; el proyecto aún no define nube; SNS+SQS para fan-out funciona pero el ordenamiento (FIFO) tiene límites de throughput |
| Redis Streams | Redis ya se usa para caché; reutilizarlo como broker mezcla responsabilidades y compromete la durabilidad de eventos críticos de auditoría |
| Comunicación síncrona (sin broker) | Viola el principio de desacoplamiento; haría que `scheduling-service` dependa síncronamente de monitoring y audit, aumentando el acoplamiento y los puntos de fallo |

## Referencias

- [event-catalog.md](../../../09-microservices/event-catalog.md)
- [communication-patterns.md](../../../09-microservices/communication-patterns.md)
- [pattern-guide.md](../../pattern-guide.md) — sección Patrones de resiliencia
