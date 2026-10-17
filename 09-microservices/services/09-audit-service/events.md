# Eventos — audit-service

> Estado: 🟢 Estable | Última actualización: 2026-10-11

---

## Eventos publicados

`audit-service` no publica eventos. Es el sumidero final del sistema de auditoría. Publicar eventos desde el `audit-service` crearía dependencias circulares y violaría su invariante de solo-escritura.

---

## Eventos consumidos

El `audit-worker` consume todos los eventos del ecosistema mediante la cola durable propia enlazada al topic exchange global `audit-events` con routing key `#`. No existe una lista cerrada de dominios ni de `event_type`: un evento nuevo bien formado se persiste sin cambios de código.

El payload se conserva íntegro como JSONB opaco. Los headers AMQP son operacionales —por ejemplo, `traceparent`— y no se convierten en campos de `audit_record` ni se registran junto con el payload.

---

## Suscripción wildcard

El componente `audit-worker` se suscribe a todos los routing keys del exchange global mediante:

```
#
```

Cada mensaje válido abre una transacción: reserva `event_id` en `audit_event_seen` y, solo si gana la reserva, inserta exactamente un `audit_record`. Un conflicto es un duplicado exitoso; un fallo del segundo INSERT revierte la reserva. El ack ocurre únicamente después del commit o de reconocer el duplicado.

Flujo de procesamiento por mensaje:

1. El broker entrega un mensaje desde `audit-events`.
2. El `audit-worker` valida el envelope canónico y normaliza solo los campos indexables.
3. En una transacción, reserva `event_id` en `audit_event_seen` con `ON CONFLICT DO NOTHING` e inserta el registro si corresponde.
4. Confirma manualmente la entrega solo después del commit o del duplicado. Un envelope inválido va a `audit.dlq`; un fallo transitorio va a `audit.retry` con TTL/backoff.

---

## Garantías de idempotencia

Si el broker re-entrega un evento (semántica at-least-once), `INSERT INTO audit.audit_event_seen ... ON CONFLICT DO NOTHING` no crea una segunda reserva. El duplicado no inserta `audit_record`, se reconoce correctamente y no se pierden ni duplican eventos.

La columna `event_id` corresponde al identificador único generado por el servicio publicador en el momento de la emisión. El `audit-service` no genera ni reasigna identificadores propios.

```sql
-- Reserva global que garantiza idempotencia incluso entre particiones
INSERT INTO audit.audit_event_seen (event_id) VALUES ($1)
ON CONFLICT (event_id) DO NOTHING;
```

Esta garantía cubre los siguientes escenarios:

| Escenario                                   | Resultado                          |
|---------------------------------------------|------------------------------------|
| Entrega exactamente una vez                 | INSERT exitoso, registro creado.   |
| Re-entrega tras fallo del consumer          | Reserva ignorada, ack AMQP confirmado. |
| Re-entrega tras fallo de red                | Reserva ignorada, ack AMQP confirmado. |
| Reprocesamiento desde RabbitMQ              | Reserva ignorada para eventos ya persistidos. |

---

## Retención

- **Mínimo legal**: 7 años desde la fecha de emisión del evento.
- **Particionamiento**: mensual recomendado por columna `emitted_at` (ver `data-model.md`).
- **Archivado a cold storage**: después de 2 años en almacenamiento activo, las particiones se mueven a almacenamiento en frío (e.g., S3 Glacier, Azure Archive).
- **Inmutabilidad**: ninguna fila de `audit_record` puede ser modificada ni eliminada antes de que expire su período de retención obligatorio. Las operaciones `UPDATE` y `DELETE` deben estar bloqueadas a nivel de política de base de datos o rol.
- **Acceso a cold storage**: bajo demanda mediante proceso de restauración documentado en `runbooks/audit-cold-restore.md`.
