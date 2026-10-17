# ADR-002: Read models para reducir dependencias síncronas de scheduling-service

**Estado:** PROPOSED
**Fecha:** 2026-06-17
**Autores:** Jesús Ariel González Bonilla (Arquitecto)
**Equipos involucrados:** Arquitectura, Backend (scheduling, actors, environment, academic)

---

## Contexto

La regla de frontera de servicios establece un máximo de **2 dependencias síncronas** por servicio (excluyendo `iam-service` para autenticación). Ver [service-boundary-rules.md](../../../09-microservices/service-boundary-rules.md).

`scheduling-service` **viola** esta regla: durante la creación y validación de horarios depende síncronamente de **3** servicios:
1. `academic-management-service` — para validar fichas y competencias
2. `training-environment-service` — para consultar disponibilidad de ambientes
3. `actors-service` — para consultar disponibilidad de instructores y sus competencias

Esto genera:
- Acoplamiento temporal: si cualquiera de los 3 servicios está caído, no se pueden crear horarios
- Latencia acumulada: cada salto suma a la latencia del flujo de creación
- Riesgo de cascada: un timeout en uno propaga fallos al motor de horarios

Hay que decidir cómo reducir las dependencias síncronas a ≤ 2 sin perder la consistencia funcional que el motor de horarios necesita.

## Decisión

Se decide introducir **read models locales** en `scheduling-service`, poblados por eventos, para los datos de referencia de alta frecuencia de consulta.

`scheduling-service` mantendrá proyecciones locales (read models) de:
- **Instructores y sus competencias** — poblado por `actors.instructor.assigned`, `actors.competency.assigned`, `actors.instructor.updated`
- **Ambientes y reglas de disponibilidad** — poblado por `environment.environment.created`, `environment.availability.changed`, `environment.maintenance.started`

De este modo, las consultas de disponibilidad durante la construcción del horario se resuelven **localmente** contra los read models, sin llamadas síncronas a `actors-service` ni `training-environment-service`.

Se mantiene **una sola** dependencia síncrona: hacia `academic-management-service` para validar la ficha y sus competencias en el momento de crear el borrador (operación puntual, no de alta frecuencia). Esto cumple la regla (1 ≤ 2).

## Consecuencias

### Positivas

- `scheduling-service` cumple la regla de ≤ 2 dependencias síncronas (queda en 1)
- Las consultas de disponibilidad se resuelven en milisegundos contra datos locales (mejora el objetivo de < 300 ms y < 2 s de validación completa)
- El motor de horarios sigue funcionando aunque `actors-service` o `training-environment-service` estén temporalmente caídos
- Reduce el riesgo de fallo en cascada

### Negativas / Trade-offs

- **Consistencia eventual**: los read models pueden estar momentáneamente desactualizados respecto al servicio dueño. Una excepción de disponibilidad de instructor recién registrada podría no reflejarse por unos segundos
- Complejidad adicional: hay que mantener proyecciones y manejar su poblado inicial (replay de eventos o snapshot)
- Duplicación de datos: ambientes, instructores y competencias existen como copia de lectura en `scheduling_db`

### Riesgos

- **Conflicto por desactualización**: si el read model está obsoleto, el motor podría proponer una asignación que ya no es válida. Mitigación: la **validación final pre-publicación** (`POST /schedules/{id}/validate`) sí consulta síncronamente el estado fresco de los servicios dueños antes de permitir `publish`. Es decir: read models para construir rápido (DRAFT), verificación síncrona puntual al publicar.
- **Poblado inicial**: un servicio nuevo no tiene histórico. Mitigación: endpoint de snapshot en los servicios dueños para bootstrap inicial del read model.

## Alternativas consideradas

| Alternativa | Por qué se descartó |
|-------------|---------------------|
| Dejar las 3 dependencias síncronas | Viola la regla de frontera; mantiene el acoplamiento y el riesgo de cascada |
| Fusionar los 4 servicios en uno | Crea un servicio ómnibus que viola los bounded contexts; pierde la independencia de despliegue |
| Caché con TTL en lugar de read models por eventos | El TTL fijo no refleja cambios inmediatos; los read models por eventos se actualizan al momento del cambio (más frescos) |
| API composition / BFF que orqueste las 3 llamadas | Mueve el problema a otra capa; sigue habiendo 3 dependencias síncronas, solo cambia quién las hace |

## Referencias

- [service-boundary-rules.md](../../../09-microservices/service-boundary-rules.md)
- [dependency-map.md](../../../09-microservices/dependency-map.md)
- [05-scheduling-service/data-model.md](../../../09-microservices/services/05-scheduling-service/data-model.md)
- ADR-001 (broker que transporta los eventos de poblado)
