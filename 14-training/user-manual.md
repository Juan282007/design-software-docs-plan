# Manual de Usuario

> Estado: 🟡 En progreso | Última actualización: 2026-08-01
> Autor: Jesús Ariel González Bonilla (PM + Arquitecto) | Equipo: Formación

Manual para **usuarios finales** de la plataforma SENA — Gestión de Horarios: coordinadores académicos, instructores y aprendices.

> **Estado real del proyecto: la aplicación aún no existe.** Hoy solo está construida la capa de datos. Este documento define el **marco y la estructura** del manual (índice y secciones previstas por funcionalidad del dominio). Las instrucciones paso a paso con pantallas reales **se completan cuando la app esté construida.** Aquí **no** se describen botones ni pantallas ficticias como si existieran.

## Cómo está organizado este manual

El manual se organiza **por rol**, porque cada usuario ve una interfaz distinta ([navigation-map.md](../12-ux-ui/navigation-map.md)). Cada sección se llenará con procedimientos y capturas cuando exista la aplicación.

## Índice previsto

### 1. Primeros pasos (todos los roles)
- Qué es la plataforma y qué permite hacer.
- Iniciar sesión y cerrar sesión.
- Conocer tu rol y lo que puedes hacer.
- _Pendiente: capturas del acceso una vez exista la UI._

### 2. Guía del Coordinador Académico
Rol central del MVP: crear, validar y publicar horarios ([discovery-brief.md](../03-product/discovery-brief.md)).
- Ver fichas activas del centro.
- Crear un horario para una ficha.
- Agregar sesiones de clase (ficha + instructor + ambiente + franja + fecha).
- Consultar disponibilidad de ambientes e instructores.
- Validar conflictos antes de publicar.
- Publicar el horario y entender el ciclo `BORRADOR → EN_REVISIÓN → PUBLICADO`.
- Crear una nueva versión de un horario ya publicado (un publicado es inmutable).
- _Pendiente: procedimientos paso a paso con la interfaz real._

### 3. Guía del Instructor
- Consultar mi horario semanal.
- Gestionar mi disponibilidad (según la fase).
- Registrar sesiones de seguimiento de la ficha (asistencia, avance).

### 4. Guía del Aprendiz
- Consultar mi horario de clases vigente.
- Recibir y entender avisos de cambios de horario.

### 5. Conceptos clave (glosario para el usuario)
Explicación en lenguaje sencillo de: ficha, programa, competencia, ambiente, horario, sesión de clase, conflicto, estado del horario. Basado en el [glosario](../01-context/glossary.md) del proyecto.

### 6. Preguntas frecuentes
- ¿Por qué no puedo editar un horario publicado?
- ¿Qué significa un conflicto y cómo lo resuelvo?
- ¿A quién contacto si algo falla? (ver [soporte](./admin-manual.md#8-procedimientos-de-soporte)).
- _Pendiente: ampliar con dudas reales tras las primeras pruebas con usuarios._

### 7. Soporte y contacto
- Cómo pedir ayuda y qué información incluir.
- Canales de atención (los define el equipo de soporte en [admin-manual.md](./admin-manual.md)).

## Cómo se completará este manual

Cada guía por rol se redactará con pasos concretos, capturas y ejemplos reales del dominio **a medida que las pantallas se construyan**. Este índice fija desde ya la cobertura esperada y el lenguaje orientado al usuario.

## Referencias

- [admin-manual.md](./admin-manual.md) · [navigation-map.md](../12-ux-ui/navigation-map.md)
- [discovery-brief.md](../03-product/discovery-brief.md) · [glosario](../01-context/glossary.md)
