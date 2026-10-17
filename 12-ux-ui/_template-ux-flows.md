# UX Flows — <PROJECT_KEY>

> **PLANTILLA** — Copiar como `ux-flows.md` y completar.
> Eliminar esta línea antes de hacer commit.

> Estado: 🔴 Pendiente | Última actualización: YYYY-MM-DD
> Autor: Por definir | Equipo: UX/UI

## Usuarios y contexto

| Rol | Dispositivo principal | Nivel técnico | Necesidad principal |
|-----|-----------------------|---------------|---------------------|
| [Instructor] | Desktop / Móvil | Básico | [Registrar disponibilidad] |

## Flujos principales

### Flujo: [Nombre del flujo]

**HU relacionada:** HU-PRJ-XXX-NNN
**Actor:** [Instructor]
**Objetivo:** [qué logra el usuario al completar este flujo]

```
[Pantalla inicial]
  ↓ [acción del usuario]
[Pantalla 2]
  ↓ [condición: si / no]
  ├─ [Camino feliz] → [Pantalla de éxito]
  └─ [Camino de error] → [Mensaje de error]
```

**Estados del flujo:**
- Inicio: [descripción]
- Happy path: [descripción]
- Error: [descripción del manejo de error]
- Fin: [estado final del sistema]

## Flujos secundarios

| Flujo | Actor | Trigger | Resultado |
|-------|-------|---------|-----------|
| | | | |

## Diagrama

> Diagrama fuente en `08-uml/diagrams/source/` — exportación en `08-uml/diagrams/exports/`.

## Referencias

- [UI Spec](./ui-spec.md)
- [PRD](../03-product/prd.md)
- [HU template](../04-requirements/_template-hu.md)
