# Wireframes

> Estado: 🟡 En progreso | Última actualización: 2026-08-01
> Autor: Jesús Ariel González Bonilla (PM + Arquitecto) | Equipo: UX-UI

Enfoque, principios y organización previstos para los wireframes de la plataforma SENA — Gestión de Horarios.

> **Estado real del proyecto: los wireframes concretos NO existen todavía.** Hoy solo está construida la capa de datos (repos `*-db`); no hay UI ni prototipos. Este documento define **cómo** se abordará el wireframing y **qué** pantallas se cubrirán, para no inventar pantallas inexistentes. Los artefactos visuales se producen en la **fase de diseño de la app**.

## Enfoque de wireframing

1. **De flujo a pantalla.** Partimos de los flujos de negocio ya documentados en [overview.md](../05-architecture/overview.md#flujos-principales) y del alcance del MVP ([discovery-brief.md](../03-product/discovery-brief.md)); cada wireframe responde a un paso de un flujo real, no a una idea suelta.
2. **Fidelidad creciente.** Primero low-fi (estructura y jerarquía), luego mid-fi (contenido real del dominio), y solo entonces alta fidelidad con los tokens del [design-system.md](./design-system.md).
3. **Por rol.** Los wireframes se agrupan por los roles del [navigation-map.md](./navigation-map.md): coordinador, instructor, aprendiz, administrador.
4. **Estados incluidos.** Cada pantalla contempla sus estados: carga, vacío, error de red y —cuando aplica— con conflictos, dada la conectividad variable de algunos centros.

## Cobertura prevista (backlog de wireframes)

Pantallas a wireframear, ordenadas por prioridad del MVP. **Ninguna está diseñada aún.**

### Prioridad 1 — Flujo del coordinador (núcleo del MVP)

| Wireframe previsto | Flujo asociado | Estado |
|--------------------|----------------|--------|
| Inicio del coordinador (fichas activas, borradores, conflictos) | Flujo 1 | ⬜ Pendiente |
| Lista de horarios por ficha y estado | Flujo 1 | ⬜ Pendiente |
| Crear/editar horario — agregar sesión (ficha+instructor+ambiente+franja) | Flujo 1 | ⬜ Pendiente |
| Selección de ambiente disponible por franja | Flujo 1 | ⬜ Pendiente |
| Selección de instructor disponible con competencia | Flujo 1 | ⬜ Pendiente |
| Panel de validación de conflictos | Flujo 1 | ⬜ Pendiente |
| Confirmación de publicación / versionado | Flujo 1 | ⬜ Pendiente |

### Prioridad 2 — Consulta de horario

| Wireframe previsto | Flujo asociado | Estado |
|--------------------|----------------|--------|
| Horario semanal del instructor | Flujo 2 | ⬜ Pendiente |
| Horario semanal del aprendiz | Flujo 2 | ⬜ Pendiente |

### Prioridad 3 — Seguimiento e indicadores

| Wireframe previsto | Flujo asociado | Estado |
|--------------------|----------------|--------|
| Registro de sesión de seguimiento | Flujo 3 | ⬜ Pendiente |
| Panel de KPIs de ficha (asistencia, avance, alertas) | Flujo 3 | ⬜ Pendiente |

## Convenciones para cuando se produzcan

- **Contenido real del dominio** en los wireframes (números de ficha, nombres de competencia, franjas), no "lorem ipsum", para validar densidad de información.
- **Anotaciones** de comportamiento y reglas junto a cada pantalla (p. ej. "un horario PUBLICADO es de solo lectura").
- **Handoff** con la plantilla [`_template-ui-spec.md`](./_template-ui-spec.md) y revisión con `design:design-critique` / `design:accessibility-review`.
- Los flujos de interacción se documentan aparte con [`_template-ux-flows.md`](./_template-ux-flows.md).

## Pendiente (fase de diseño de la app)

- [ ] Elegir herramienta de wireframing/prototipado.
- [ ] Producir los wireframes low-fi de Prioridad 1 y validarlos con coordinadores.
- [ ] Enlazar aquí cada artefacto una vez creado.

## Referencias

- [navigation-map.md](./navigation-map.md) · [design-system.md](./design-system.md)
- [overview.md (flujos)](../05-architecture/overview.md#flujos-principales) · [discovery-brief.md](../03-product/discovery-brief.md)
