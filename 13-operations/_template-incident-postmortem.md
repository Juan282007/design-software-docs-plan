# Postmortem de incidente — <PROJECT_KEY> — [Fecha]

> **PLANTILLA** — Copiar como `postmortem-YYYY-MM-DD-titulo-corto.md` y completar.
> Eliminar esta línea antes de hacer commit.

> Estado: 🔴 Pendiente | Última actualización: YYYY-MM-DD
> Autor: Por definir | Equipo: DevOps / Tech Lead

## Resumen ejecutivo

| Campo | Valor |
|-------|-------|
| Severidad | P0 / P1 |
| Fecha de inicio | YYYY-MM-DD HH:MM UTC |
| Fecha de resolución | YYYY-MM-DD HH:MM UTC |
| Duración | [X horas Y minutos] |
| Servicios afectados | |
| Usuarios impactados | [estimado] |
| SLO violado | Sí / No |

## Línea de tiempo

| Timestamp | Evento | Responsable |
|-----------|--------|-------------|
| HH:MM | Alerta detectada | [sistema / persona] |
| HH:MM | Equipo notificado | |
| HH:MM | Causa raíz identificada | |
| HH:MM | Mitigación aplicada | |
| HH:MM | Servicio restaurado | |
| HH:MM | Verificación completada | |

## Causa raíz

**Causa inmediata:** [qué desencadenó el incidente]

**Causa raíz:** [por qué ocurrió el desencadenante]

**Causa sistémica (si aplica):** [por qué el sistema permitió que esto ocurriera]

## Impacto

| Métrica | Valor |
|---------|-------|
| Tiempo de indisponibilidad | [X min] |
| Requests afectados | [N] |
| Error budget consumido | [X min de Y disponibles en el mes] |
| Pérdida económica / SLA | [si aplica] |

## Qué funcionó bien

- [Qué ayudó a detectar o resolver el incidente rápidamente]

## Qué no funcionó bien

- [Qué retrasó la detección o resolución]

## Acciones correctivas

| Acción | Responsable | Fecha límite | Ticket |
|--------|-------------|--------------|--------|
| [Agregar alerta preventiva para X] | | YYYY-MM-DD | |
| [Mejorar runbook para escenario Y] | | YYYY-MM-DD | |

## Referencias

- [Runbook](./runbook.md)
- [SLA/SLO/SLI](./sla-slo-sli.md)
