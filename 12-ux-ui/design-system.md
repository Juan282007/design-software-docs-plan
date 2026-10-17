# Sistema de Diseño

> Estado: 🟡 En progreso | Última actualización: 2026-08-01
> Autor: Jesús Ariel González Bonilla (PM + Arquitecto) | Equipo: UX-UI

Guía de diseño de la plataforma **SENA — Gestión de Horarios**. Define los principios, tokens y patrones que gobernarán la interfaz cuando se construya la aplicación.

> **Estado real del proyecto:** hoy solo existe la **capa de datos** (repos `*-db` con Liquibase + PostgreSQL 16). **Aún no hay UI ni componentes implementados.** Este documento es la **guía de referencia** que orientará el diseño; los valores concretos (paleta final, librería de componentes, tipografía definitiva) se **ratifican en la fase de diseño de la app** y quedan marcados como pendientes.

## 1. Principios de diseño

1. **Claridad antes que densidad.** El coordinador toma decisiones sobre conflictos de horario; la interfaz debe resaltar el estado (BORRADOR / EN_REVISIÓN / PUBLICADO) y los conflictos por encima de todo lo demás.
2. **El conflicto es un ciudadano de primera clase.** Detectar y comunicar un conflicto (instructor doble-asignado, ambiente sobreprogramado, ficha solapada) es la razón de ser del sistema; su representación visual es prioritaria.
3. **Una interfaz por rol.** Coordinador, instructor, aprendiz y administrador ven vistas distintas con permisos distintos (ver [navigation-map.md](./navigation-map.md)). No se comparte la misma pantalla con controles ocultos.
4. **Tolerante a conectividad variable.** Algunos centros tienen conectividad intermitente ([discovery-brief.md](../03-product/discovery-brief.md)); estados de carga, reintentos y consulta offline básico condicionan el diseño desde el inicio.
5. **Lenguaje del dominio SENA.** La UI habla en español con el vocabulario del negocio (ficha, ambiente, competencia, RAP); el código va en inglés. El puente está en el [glosario](../01-context/glossary.md) y el lenguaje ubicuo del [domain-map](../02-domain/domain-map.md#lenguaje-ubicuo--mapeo-dominio--técnico).

## 2. Accesibilidad (objetivo: WCAG 2.1 AA)

- **Contraste** mínimo 4.5:1 en texto normal y 3:1 en texto grande y elementos de interfaz.
- **No depender solo del color** para comunicar estado o conflicto: acompañar siempre con icono y/o texto (relevante para daltonismo, dado que el estado es central).
- **Navegación por teclado** completa: foco visible, orden lógico de tabulación, atajos para el flujo de creación de horario.
- **Objetivos táctiles** de al menos 44×44 px, pensando en uso en tablet dentro de ambientes.
- **Etiquetas y roles ARIA** en formularios y tablas de horario; mensajes de error asociados al campo.
- El equipo dispone del skill `design:accessibility-review` para auditar cada pantalla antes del handoff.

## 3. Tokens de diseño (previstos)

Los tokens se definirán como variables (design tokens) para mantener coherencia y permitir tema claro/oscuro. Valores **tentativos, pendientes de ratificación**:

### Color

| Token | Uso | Nota |
|-------|-----|------|
| `color-brand` | Identidad SENA (verde institucional) | Valor exacto **pendiente** de lineamiento de marca SENA |
| `color-surface` / `color-background` | Fondos de tarjetas y página | — |
| `color-text` / `color-text-muted` | Texto principal / secundario | Contraste AA obligatorio |
| `color-status-draft` | Estado BORRADOR | Neutro |
| `color-status-review` | Estado EN_REVISIÓN | Advertencia |
| `color-status-published` | Estado PUBLICADO | Éxito |
| `color-conflict` | Conflicto detectado | Crítico, nunca solo color |
| `color-info` / `color-success` / `color-warning` / `color-danger` | Feedback | Semánticos |

> Los estados de negocio provienen del catálogo parametrizable `status` (ver [modeling-conventions §1](../06-data/modeling-conventions.md)); la UI **no** debe hardcodear la lista de estados, sino consumirla.

### Tipografía, espaciado y forma

| Grupo | Tokens previstos |
|-------|------------------|
| Tipografía | `font-family-base`, escala `font-size-{xs..2xl}`, `font-weight-{regular,medium,bold}`, `line-height-{tight,normal,relaxed}` |
| Espaciado | Escala base de 4 px: `space-{1..8}` |
| Radios | `radius-{sm,md,lg}` |
| Sombras | `shadow-{sm,md}` para elevación de tarjetas y menús |
| Z-index | Capas: base, dropdown, overlay, modal, toast |

> Familia tipográfica concreta: **pendiente** (candidata: una sans-serif del sistema para rendimiento en conectividad baja).

## 4. Componentes base (catálogo previsto)

Inventario objetivo de componentes; ninguno está implementado aún. Cada uno se especificará con la plantilla [`_template-ui-spec.md`](./_template-ui-spec.md) (estados, accesibilidad, variantes).

- **Estructura:** cabecera de aplicación, navegación por rol, breadcrumb, contenedor de página.
- **Datos:** tabla de horario (rejilla día × franja), tarjeta de sesión de clase, tarjeta de ficha, lista de ambientes/instructores disponibles.
- **Estado:** badge de estado (BORRADOR/EN_REVISIÓN/PUBLICADO), panel de conflictos, indicador de disponibilidad, alerta de seguimiento.
- **Entrada:** formularios de sesión (ficha + instructor + ambiente + franja + fecha), selectores con búsqueda, date/time pickers.
- **Feedback:** toasts, modales de confirmación (publicar/despublicar), estados vacíos, skeletons de carga, estados de error de red.

### Estados obligatorios por componente

Todo componente interactivo documenta: `default`, `hover`, `focus`, `active`, `disabled`, `loading`, `error` y `empty` cuando aplique.

## 5. Patrones transversales

- **Máquina de estados visible:** el ciclo `BORRADOR → EN_REVISIÓN → PUBLICADO` se refleja en la UI; un horario PUBLICADO es inmutable y sus cambios generan nueva versión ([discovery M8](../03-product/discovery-brief.md)).
- **Validación pre-publicación:** el usuario nunca publica a ciegas; siempre precede una vista de conflictos.
- **Feedback de latencia:** las consultas críticas de disponibilidad tienen presupuesto < 300 ms ([overview](../05-architecture/overview.md#rendimiento)); la UI muestra skeleton en vez de spinner bloqueante.

## Pendiente (fase de diseño de la app)

- [ ] Ratificar paleta y token de marca con lineamiento SENA.
- [ ] Elegir librería/base de componentes y stack de front.
- [ ] Definir tipografía y escala tipográfica final.
- [ ] Especificar cada componente con `_template-ui-spec.md`.
- [ ] Construir los wireframes concretos (ver [wireframes.md](./wireframes.md)).

## Referencias

- [navigation-map.md](./navigation-map.md) · [wireframes.md](./wireframes.md)
- [discovery-brief.md](../03-product/discovery-brief.md) · [domain-map.md](../02-domain/domain-map.md)
- [overview.md (arquitectura)](../05-architecture/overview.md) · [glosario](../01-context/glossary.md)
