<!-- RESUMEN-EJECUTIVO
agente: Jesús Ariel González Bonilla (PM + Arquitecto)
capacidad: contrato de evento / mensajería async
fase: diseño
estado: accepted
dependencias_entrada: 09-microservices/event-catalog.md, la BD de audit-service
consumidores_siguientes: ninguno (sink terminal; no publica eventos)
tldr: Worker consumidor por wildcard que persiste cada evento del ecosistema como audit_record append-only.
decisiones_clave: Suscripción fan-out a todos los topics (no depende de nombres de evento específicos); append-only sin FKs; idempotencia por event_id UNIQUE (conflicto se ignora, no error).
halts_registrados: ninguno — al consumir por wildcard, el worker no depende de que cada evento esté registrado en event-catalog.md
-->

# Contrato — audit-worker

> Estado: 🟡 En progreso | Última actualización: 2026-08-06
> Autor: Jesús Ariel González Bonilla (PM + Arquitecto) | Equipo: Arquitectura

> **Fuente de verdad:** el inventario de eventos lo gobierna [`event-catalog.md`](../../../../event-catalog.md) y el envelope estándar [`_template/service/events.md`](../../../../_template/service/events.md). Ante discrepancia, esos documentos prevalecen sobre este contrato.

> **Implementación base — HU-AUD-002.** Contrato de mensajería para el worker Go/RabbitMQ.

## Eventos consumidos

Fan-out a **todos** los eventos mediante el topic exchange global durable `audit-events`. La cola durable propia se nombra con `BROKER_CONSUMER_GROUP` y se enlaza con routing key `#`; por tanto no hay una lista cerrada de exchanges ni `event_type` en código. Los productores enlazan sus eventos al exchange global como responsabilidad de infraestructura.

## Eventos producidos
Ninguno. El `audit-worker` es un **consumidor terminal** (sink): solo consume y persiste, no publica eventos (así lo confirma la nota del [catálogo](../../../../event-catalog.md#audit-service)).

## Envelope de evento consumido (estándar)
```json
{
  "event_id": "uuid-v4",
  "event_type": "<service>.<entity>.<action>",
  "version": "1.0",
  "timestamp": "2026-08-01T14:00:00Z",
  "source_service": "scheduling-service",
  "correlation_id": "uuid-v4",
  "payload": {
    "actor_id": "uuid",
    "entity_type": "class_session",
    "entity_id": "uuid",
    "training_center_id": "uuid | null"
  }
}
```

## Efecto (persistencia)
Inserta un `audit_record` extrayendo `event_type`, `source_service` y `timestamp` del envelope. `actor_id`, `entity_type`, `entity_id` y `training_center_id` se proyectan desde `payload` cuando el evento origen los incluye, y se conserva el `payload` completo como `JSONB` (schema-on-read). El worker no amplía el envelope común con campos de auditoría en el nivel superior.

`training_center_id` es nullable y se proyecta únicamente para seguridad de scope y consulta: eventos GLOBAL o sin centro se persisten con `null`. Si el campo está presente debe ser UUID válido; un valor inválido se rechaza antes de reservar `audit_event_seen` o insertar `audit_record`. El worker no infiere el centro consultando otros servicios o bases de datos.

## Reglas
- **Append-only:** solo `INSERT`. Sin `UPDATE`/`DELETE` ni endpoint de escritura externo.
- **Idempotencia:** la transacción inserta primero `event_id` en `audit.audit_event_seen` con `ON CONFLICT DO NOTHING`; si no gana esa reserva, no inserta `audit_record` y confirma la entrega como duplicada.
- **Sin FKs:** el registro es autónomo (sink desacoplado entre servicios).
- **Ack:** el ack AMQP se envía solo después del commit o de confirmar que se trata de un duplicado.
- **Privacidad:** `payload` se persiste como JSONB opaco, pero nunca se escribe en logs; los headers se usan para trazabilidad y no se añaden como columnas de `audit_record`.

## Política de reintentos

La cola principal dead-letterea fallos transitorios hacia `audit.retry`; la cola de retry aplica TTL (`WORKER_RETRY_DELAY_MS`) y devuelve el mensaje al exchange global. Se cuentan las muertes de cola y, al alcanzar `WORKER_MAX_RETRIES`, el worker publica el cuerpo y headers originales en la cola durable **`audit.dlq`** con una causa técnica saneada. Los envelopes inválidos se envían directamente a esa DLQ. Ninguna de estas rutas confirma el mensaje antes de que el broker acepte su destino.

| Variable | Significado | Valor por defecto no secreto |
| --- | --- | --- |
| `AUDIT_EVENTS_EXCHANGE` | Exchange topic global de auditoría | `audit-events` |
| `BROKER_CONSUMER_GROUP` | Nombre de la cola durable compartida por réplicas | obligatorio |
| `BROKER_TOPIC_PATTERN` | Binding AMQP | `#` (obligatorio) |
| `WORKER_MAX_RETRIES` | Reintentos transitorios antes de DLQ | `3` |
| `WORKER_RETRY_DELAY_MS` | Backoff TTL de retry | `5000` |
