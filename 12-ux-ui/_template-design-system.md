# Design System — <PROJECT_KEY>

> **PLANTILLA** — Copiar como `design-system.md` y completar.
> Eliminar esta línea antes de hacer commit.

> Estado: 🔴 Pendiente | Última actualización: YYYY-MM-DD
> Autor: Por definir | Equipo: UX/UI

## Fundamentos

### Paleta de colores

| Token | Valor | Uso |
|-------|-------|-----|
| `color-primary` | `#[hex]` | Botones principales, CTAs |
| `color-secondary` | `#[hex]` | Acciones secundarias |
| `color-error` | `#[hex]` | Mensajes de error, bordes inválidos |
| `color-success` | `#[hex]` | Confirmaciones, estados OK |
| `color-neutral-100` | `#[hex]` | Fondos |
| `color-neutral-900` | `#[hex]` | Texto principal |

### Tipografía

| Uso | Fuente | Tamaño | Peso |
|-----|--------|--------|------|
| Títulos H1 | [fuente] | [X]px | Bold |
| Títulos H2 | [fuente] | [X]px | SemiBold |
| Cuerpo | [fuente] | [X]px | Regular |
| Caption | [fuente] | [X]px | Regular |

### Espaciado

| Token | Valor | Uso |
|-------|-------|-----|
| `spacing-xs` | 4px | |
| `spacing-sm` | 8px | |
| `spacing-md` | 16px | |
| `spacing-lg` | 24px | |
| `spacing-xl` | 32px | |

## Componentes base

| Componente | Variantes | Estado disponibles |
|------------|-----------|-------------------|
| Button | primary / secondary / danger / ghost | default / hover / disabled / loading |
| Input | text / email / password / number | default / focus / error / disabled |
| Card | default / bordered | — |
| Modal | small / medium / large | — |
| Toast | success / error / warning / info | — |

## Iconografía

| Librería | Versión | URL / Package |
|----------|---------|---------------|
| [Material Icons / Heroicons / otro] | | |

## Accesibilidad (WCAG 2.1 AA)

- Contraste mínimo: 4.5:1 para texto normal, 3:1 para texto grande
- Todos los componentes interactivos accesibles por teclado
- ARIA labels en formularios e iconos sin texto

## Herramientas

| Herramienta | Propósito | URL |
|-------------|-----------|-----|
| [Figma / Adobe XD / Sketch] | Diseño y prototipado | |
| [Storybook] | Documentación de componentes | |

## Referencias

- [UX Flows](./ux-flows.md)
- [UI Spec](./ui-spec.md)
