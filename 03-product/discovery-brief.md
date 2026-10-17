# Discovery Brief — SENA Gestión de Horarios

> Fase: 01-Discovery | Agente: A05 | Estado: 🟡 Borrador
> Fecha: 2026-06-17

## Contexto

El SENA (Servicio Nacional de Aprendizaje) opera centros de formación a nivel nacional con decenas de fichas de caracterización activas simultáneamente. Cada ficha agrupa un conjunto de aprendices vinculados a un programa de formación específico y requiere la asignación periódica de:

- Instructores con competencias certificadas para las competencias del programa
- Ambientes físicos con capacidad y equipamiento adecuado
- Franjas horarias compatibles con la disponibilidad de todos los actores

Este proceso se realiza hoy de forma manual o semi-manual por coordinadores académicos, sin herramienta integrada que detecte conflictos o garantice visibilidad a los actores.

## Problema

**Los coordinadores académicos no tienen una herramienta centralizada para crear, validar y publicar horarios de formación**, lo que genera:

- Conflictos no detectados a tiempo: instructor doble-asignado, ambiente sobreprogramado
- Falta de visibilidad para instructores y aprendices sobre sus horarios vigentes
- Procesos manuales que consumen horas por semana en reconciliación
- Ausencia de trazabilidad sobre cambios de horario y sus causas

## Usuarios afectados

| Rol | Dolor principal |
|-----|----------------|
| Coordinador Académico | Crea horarios manualmente; detecta conflictos tarde o nunca |
| Instructor | No conoce su carga horaria con anticipación; no puede gestionar su disponibilidad |
| Aprendiz | Desconoce su horario de clases actualizado; no recibe alertas de cambios |
| Administrador de Centro | Sin métricas de utilización de ambientes ni carga de instructores |

## Resultado esperado

Una plataforma digital que permita al coordinador académico **crear, validar y publicar horarios de formación** con detección automática de conflictos, y que comunique el horario publicado a instructores y aprendices en tiempo real.

## MVP — Alcance mínimo viable

| Capacidad | Incluye | Excluye |
|-----------|---------|---------|
| Creación de horario | Asignación de ficha + instructor + ambiente + franja | Sugerencia automática de asignación |
| Validación de conflictos | Detección antes de publicación | Resolución automática de conflictos |
| Publicación | Cambio de estado BORRADOR → PUBLICADO | Notificación push móvil |
| Consulta de horario | Vista por instructor y por aprendiz | Exportación a calendario externo |
| Gestión de disponibilidad | Disponibilidad de ambientes e instructores | Restricciones complejas por modalidad contractual |

## KPIs base — línea cero

| KPI | Situación actual | Meta MVP |
|-----|-----------------|----------|
| Tiempo promedio de creación de horario semanal | No medido (estimado: 4–8 h) | < 1 h |
| % horarios publicados sin conflictos detectados post-publicación | No medido | > 95 % |
| % instructores que conocen su horario con ≥ 48 h de anticipación | No medido | > 90 % |
| Utilización promedio de ambientes en horas programadas | No medido | Medible desde sprint 1 |

## Hallazgos por dominio — Análisis de equipos

### M1 — IAM (Identidad y Acceso)

Los sistemas SENA actuales no tienen control de acceso granular por rol dentro de un centro. Coordinadores, instructores y aprendices comparten vistas sin diferenciación de permisos.

**Implicación**: Se requiere un servicio de identidad propio con roles `COORDINATOR`, `INSTRUCTOR`, `LEARNER`, `ADMIN`. JWT con refresh token es el mecanismo adecuado para sesiones de usuario.

### M2 — Estructura Institucional

La jerarquía SENA tiene 6 niveles: Macrorregión → Microrregión → Departamento → Municipio → Centro de Formación → Unidad Institucional. Un coordinador opera dentro de un centro; un instructor puede estar vinculado a varias unidades dentro del mismo centro.

**Implicación**: La jerarquía institucional es dato de referencia con baja frecuencia de cambio. Candidato a caché con TTL largo (10 min). No requiere servicio propio; se consolida con parametrización (M4).

### M3 — Infraestructura / Ambientes

Los ambientes tienen tipo (aula, laboratorio, taller), capacidad, equipamiento y hasta 24 reglas de disponibilidad por día. Un ambiente puede estar "disponible en el sistema" pero en mantenimiento físico real.

**Implicación**: La consulta de disponibilidad de ambientes debe responder en < 300 ms porque es operación crítica durante la creación de horario. La disponibilidad efectiva debe calcularse restando mantenimientos y reservas confirmadas.

### M4 — Parametrización

Catálogos de dominio: tipos de vinculación, niveles de formación, modalidades, estados de ficha, tipos de ambiente, etc. Estos catálogos cambian raramente (trimestral o menos frecuente).

**Implicación**: Datos de referencia compartidos. Se consolidan con M2 en `reference-data-service`. Caché Redis con TTL 10 min en los consumidores.

### M5 — Programas de Formación

Estructura curricular: Línea Tecnológica → Red Tecnológica → Red de Conocimiento → Programa → Competencia → Resultado de Aprendizaje (RAP). Un programa tiene duración total en horas y nivel de formación (Auxiliar, Operario, Técnico, Tecnólogo).

**Implicación**: La competencia es el vínculo entre programa e instructor. El RAP es la unidad que conecta programa con sesión de clase. Este árbol es el núcleo de `academic-management-service`.

### M6 — Oferta / Fichas de Caracterización

Una ficha es la instancia de un programa en un centro para una cohorte específica. Tiene número de ficha SENA único, fecha de inicio, número de aprendices y estado (Ejecución, Etapa Productiva, Terminada).

**Implicación**: El horario se crea por ficha activa. Una ficha en Ejecución puede tener múltiples versiones de horario a lo largo de su ciclo de vida. Las fichas referencian programas de `academic-management-service`.

### M7 — Actores

Instructores: competencias certificadas, tipo de vinculación (planta / contratista), carga máxima en horas por semana. Aprendices: estado (activo, desertor, trasladado), etapa actual (lectiva / productiva). Empresas: participan en etapa productiva.

**Implicación**: La disponibilidad del instructor es dato crítico para el motor de horarios; debe consultarse con baja latencia (< 300 ms). `actors-service` expone el endpoint `GET /instructors/available` como contrato SLA.

### M8 — Horarios

Una sesión de clase vincula: ficha + instructor + ambiente + franja horaria + fecha. Conflictos posibles: instructor en dos lugares simultáneamente, ambiente sobre-asignado, ficha con sesiones solapadas. Ciclo de vida del horario: `BORRADOR → EN_REVISION → PUBLICADO`. Un horario `PUBLICADO` es inmutable; los cambios generan una nueva versión.

**Implicación**: El motor de horarios es el componente de mayor complejidad del sistema. La detección de conflictos debe ser pre-publicación y exhaustiva. `scheduling-service` es el bounded context core.

### M9 — Seguimiento / Analítica

Seguimiento periódico a la ficha: asistencia de instructores, presencia de aprendices, avance en competencias. Alertas cuando hay desviación: baja asistencia, bajo avance, riesgo de deserción. Planes de mejoramiento asociados a hallazgos.

**Implicación**: El seguimiento consume eventos del módulo de horarios. Los KPIs requieren datos agregados. `monitoring-service` es consumidor de eventos, no productor de horarios.

## Restricciones técnicas relevantes

- No hay sistema legado con API documentada disponible (proyecto greenfield)
- No hay base de datos existente a migrar en esta fase
- Conectividad variable en algunos centros de formación rurales
- El horario publicado debe estar disponible offline básico en el futuro (fuera del MVP, pero condiciona el diseño de contratos)

## Preguntas abiertas

1. ¿El coordinador puede modificar un horario `PUBLICADO` directamente o siempre debe crear uno nuevo?
2. ¿Los instructores contratistas tienen una restricción de horas máximas diferente a los de planta?
3. ¿Hay integración requerida con el sistema SOFIA Plus del SENA en esta fase o solo en fases posteriores?
4. ¿Un aprendiz puede estar matriculado en más de una ficha activa simultáneamente?
