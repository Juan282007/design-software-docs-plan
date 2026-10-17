# Gestión de incidentes

> Estado: 🟡 En progreso | Última actualización: 2026-08-01
> Autor: Jesús Ariel González Bonilla (PM + Arquitecto) | Equipo: Operaciones

Cómo se clasifica, responde, comunica y aprende de los incidentes del sistema **Horarios SENA**. En el estado actual —**solo capa de datos** (PostgreSQL 16 + Liquibase en Docker)— los incidentes plausibles son de base de datos y de migraciones (BD caída, migración fallida, disco lleno, dato de catálogo corrupto). Cuando exista la capa de aplicación, este mismo marco cubrirá también los servicios Go y el broker de eventos.

## Severidades

| Sev | Definición | Ejemplos (hoy, capa de datos) | Respuesta |
|-----|------------|-------------------------------|-----------|
| **P0** | Pérdida total o corrupción de datos; BD caída sin acceso | PostgreSQL no responde; corrupción de datos; pérdida de backups | Atención inmediata; se moviliza el equipo |
| **P1** | Degradación grave o riesgo alto sin pérdida aún | Migración fallida que deja un esquema inconsistente; disco > 85%; seed que rompió un catálogo | Atención el mismo día |
| **P2** | Impacto acotado, con workaround | Consultas lentas sostenidas; changeset con rollback defectuoso detectado en local | Priorizable en el sprint |
| **P3** | Impacto menor / cosmético | Discrepancia menor en datos de prueba no productivos | Deuda menor, se agenda |

> Los rangos numéricos de disponibilidad se comprometerán con los SLO por servicio (ver [_template-sla-slo-sli.md](./_template-sla-slo-sli.md)); hoy la clasificación es por impacto sobre la integridad de los datos.

## Roles

| Rol | Responsabilidad |
|-----|-----------------|
| **Incident Commander (IC)** | Coordina la respuesta, decide mitigaciones, mantiene la línea de tiempo. Un único responsable a la vez. |
| **Responsable técnico** | Ejecuta el diagnóstico y las acciones sobre la BD/migraciones (o el servicio afectado en el futuro). |
| **Comunicaciones** | Informa a stakeholders y mantiene el registro de actualizaciones. En incidentes pequeños puede coincidir con el IC. |
| **Escalamiento** | Arquitectura/Datos para decisiones sobre esquema, y seguridad si hay exposición de datos (ver [security-rules.md](../00-governance/security-rules.md)). |

> **Punto abierto:** la rotación de guardia (on-call) y los handles/canales reales de cada rol están por definir.

## Flujo de respuesta

1. **Detección.** Alerta automática (ver [observability.md](./observability.md)) o reporte manual.
2. **Triaje.** Asignar severidad y designar IC. Para P0/P1 se abre canal dedicado del incidente.
3. **Contención.** Frenar el daño antes de arreglar la causa. Ejemplos de datos:
   - Migración fallida: **no** reintentar a ciegas; revisar `databasechangelog` del módulo y, en local, revertir con el `rollback` del changeset (en `qa`/`staging`/`main` las migraciones son forward-only: se prepara un forward-fix, no un rollback en caliente).
   - Corrupción/pérdida: aislar la BD y preparar restauración (ver [backup-and-recovery.md](./backup-and-recovery.md)).
   - Disco lleno: liberar espacio/rotar logs antes de reanudar escrituras.
4. **Diagnóstico.** Identificar causa inmediata y causa raíz.
5. **Mitigación / resolución.** Aplicar el fix (forward-fix de migración, restauración, ajuste de configuración) y verificar con healthchecks (`pg_isready`, `status --verbose`).
6. **Verificación y cierre.** Confirmar integridad de datos y estabilidad; comunicar resolución.
7. **Postmortem** para P0/P1 (obligatorio).

## Comunicación

| Momento | Contenido |
|---------|-----------|
| Inicio (P0/P1) | Qué falla, severidad, impacto conocido, IC asignado |
| Cada intervalo acordado | Causa (si se conoce), acción en curso, próximo paso, ETA |
| Resolución | Qué se restauró, duración, causa raíz preliminar, si hubo impacto en datos |

Regla de seguridad: si el incidente involucra exposición de datos sensibles o una credencial, **rotar primero, avisar después** y seguir el procedimiento de fuga de [security-rules.md](../00-governance/security-rules.md). No pegar datos reales ni secretos en los canales del incidente.

## Postmortem

Obligatorio para P0/P1, **sin culpables** (blameless). Se documenta con la plantilla [_template-incident-postmortem.md](./_template-incident-postmortem.md): resumen ejecutivo, línea de tiempo, causa inmediata / raíz / sistémica, impacto (incluida integridad de datos y, si aplica, error budget), qué funcionó y qué no, y **acciones correctivas** con responsable y fecha.

Regla de mejora continua: toda acción correctiva se convierte en un ticket rastreable; un incidente sin acciones correctas registradas no se considera cerrado.

## Puntos abiertos

- Guardia on-call, canales reales y herramienta de alerting/paging.
- Umbrales de severidad ligados a SLO por servicio.

## Referencias

- [_template-incident-postmortem.md](./_template-incident-postmortem.md)
- [_template-runbook.md](./_template-runbook.md)
- [observability.md](./observability.md)
- [backup-and-recovery.md](./backup-and-recovery.md)
- [security-rules.md](../00-governance/security-rules.md)
