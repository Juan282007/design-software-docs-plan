<!-- RESUMEN-EJECUTIVO
agente: Jesús Ariel González Bonilla (PM + Arquitecto)
capacidad: DDD (Design-Driven Document) para generación de mockup con Google Stitch
fase: diseño (UX/UI)
estado: draft
dependencias_entrada: 12-ux-ui/design-system.md, navigation-map.md; 09-microservices/services/*/openapi.yaml; 06-data + services/*/data-model.md; 04-requirements/user-stories.md
consumidores_siguientes: Google Stitch (generación de mockup), luego repo frontend (que NO conoce el mockup)
tldr: Documento que traduce el SSOT (design-system + contratos + data-model + HU) en prompts listos para Stitch y aloja los mockups generados, todo dentro de docs.
decisiones_clave: DDD y mockups viven en docs/12-ux-ui; frontend jamás conoce el mockup; cada pantalla es trazable a HU+endpoint+tabla
halts_registrados: ninguno
-->

# DDD — Design-Driven Document (mockup con Google Stitch)

> **ESTADO: PRELIMINAR (v0 · mockup-first · design-driven discovery).** Este mockup es un
> **instrumento de descubrimiento y feedback**, no un diseño final ni objeto construible. Se
> busca **precisión** (derivar cada pantalla de los contratos + data-model reales) para que el
> feedback sea de calidad, pero se espera iterar.
>
> **Las HU se construirán mucho más adelante** — todavía faltan varios frentes por comprender
> antes de escribir HU. Por eso el DDD **NO depende de HU**: la trazabilidad a **endpoint + tabla
> es completa ya** (eso lo tenemos 100%); la trazabilidad a **HU se cierra después**. Donde exista
> una `HU-##` en `04-requirements/user-stories.md` se cita; si no, se marca `HU: pendiente`.

**Qué es:** el documento que **dirige la generación del mockup** de todo el sistema con
[Google Stitch](https://stitch.withgoogle.com/). Traduce el conocimiento ya existente
(design-system + contratos de API + modelo de datos + navegación) en **prompts listos para pegar
en Stitch**, pantalla por pantalla.

## Regla de ubicación (decidida)

- **El DDD vive en `docs/12-ux-ui/mockup-ddd/`.**
- **El mockup generado vive en `docs/12-ux-ui/mockup-ddd/mockups/`.**
- **El repo `frontend/` es una capa limpia que JAMÁS conoce el mockup.** El frontend se
  construye contra los **contratos** (`09-microservices/**/openapi.yaml`) y el design-system,
  no contra estos artefactos. El mockup es insumo de diseño/validación, no dependencia de código.

## Estado del mockup (implementación adoptada)

El mockup generado y **adoptado** es una **SPA con hash-router** (no un build estático de un
archivo `.html` por ruta). Cómo se abre y qué entrega:

- **Punto de entrada:** `mockups/app/index.html`. Servir con `python -m http.server` y abrir
  `#/inventory` (inventario maestro de pantallas), o abrir directamente `review.html`.
- **Cobertura:** **45 pantallas + modales**. Validador: `node tools/validate-routes.js` → **45/45**.
- **Rutas de revisión (hash + query):** rol activo `?as=<rol>`; estados `?state=loading|empty|error`
  y `?offline=1`; modales `?modal=…`; panel de notificaciones `?overlay=notifications`; estados
  globales `#/system-states?variant=403|404|500|session`. Acceso directo con rol no autorizado → **403**.
- **Capturas:** **90 PNG** bajo `mockups/app/screenshots/` (**45 desktop** 1440×1000 + **45 móvil**
  390×844), indexadas en `screenshots/README.md`.

> Esto reemplaza cualquier descripción previa de un build estático (p. ej. "un `.html` por ruta").
> El **contrato de entrega** de más abajo describe el artefacto por prompt; el mockup adoptado lo
> materializa como esta SPA con las 90 capturas de referencia.

## Índice del spec del mockup

| Documento | Qué contiene |
|---|---|
| **README.md** (este) | framework, reglas de calidad/anti-patrón, contrato de entrega, brief global |
| [micro-frontends.md](micro-frontends.md) | arquitectura "por debajo": dominio → micro-frontend → pantalla |
| [screen-inventory.md](screen-inventory.md) | inventario maestro de las 45 pantallas (por flujo y por MFE) + estado |
| [component-inventory.md](component-inventory.md) | componentes compartidos (design-system) + patrones por MFE |
| [sitemap.md](sitemap.md) | mapa de rutas por rol + matriz de visibilidad RBAC |
| [design-tokens.md](design-tokens.md) | tokens ratificados (color/tipografía/espaciado) locked para el mockup |
| [ux-flows.md](ux-flows.md) | journeys end-to-end que conectan las pantallas |
| [ux-copy.md](ux-copy.md) | microcopy canónica (errores, vacíos, confirmaciones, labels) |
| [acceptance-and-states.md](acceptance-and-states.md) | estados obligatorios por tipo + Definition of Done |
| [discovery-findings.md](discovery-findings.md) | vacíos de contrato/RBAC detectados (feedback) |
| [refinamiento-alignment.md](refinamiento-alignment.md) | triage de las 22 retros × estado real + materialización en mockup + spec de cambio aditivo |
| [refinamiento-decisions-D4-D12.md](refinamiento-decisions-D4-D12.md) | registro oficial de las decisiones de negocio D-4…D-12 y su estado de implementación |
| `flows/01-auth.md … 06-backoffice.md` | las pantallas con su `PROMPT STITCH` (por flujo/rol) |

## Fuente única de verdad (de dónde deriva cada pantalla)

El DDD **no inventa** dominio ni datos; deriva del SSOT:

| Insumo | Aporta a la pantalla |
|---|---|
| `04-requirements/user-stories.md` | la **HU** (qué necesita el usuario) |
| `09-microservices/services/<svc>/components/*/openapi.yaml` | las **acciones/datos** (endpoints que consume) |
| `06-data/*` + `services/<svc>/data-model.md` | los **campos** que se muestran (nombres/tipos/enums reales) |
| `12-ux-ui/navigation-map.md` | la **ubicación** en la navegación por rol |
| `12-ux-ui/design-system.md` | **estilo, tokens, accesibilidad** (WCAG 2.1 AA) |

**Trazabilidad obligatoria por pantalla:** `pantalla → HU → endpoint(s) → tabla(s)`.

## Principios que Stitch debe respetar (del design-system)

1. **Claridad antes que densidad.** Estado (BORRADOR/EN_REVISIÓN/PUBLICADO) y **conflictos** por encima de todo.
2. **El conflicto es ciudadano de primera clase** — su representación visual es prioritaria; nunca solo por color.
3. **Una interfaz por rol** (Coordinador, Instructor, Aprendiz, Administrador) — sin controles ocultos.
4. **Tolerante a conectividad variable** — estados loading / empty / error / offline básicos.
5. **Lenguaje del dominio SENA en español** (ficha, ambiente, competencia, RAP); código en inglés.
6. **Accesibilidad WCAG 2.1 AA** — contraste ≥4.5:1, teclado, objetivos táctiles 44×44, ARIA.
7. **Estados de negocio parametrizables** — la UI consume el catálogo `status`, no lo hardcodea.

## Reglas de calidad para Stitch (anti-patrones) — obligatorias en TODO prompt

> Derivadas de la revisión del mockup **v1** (elementos desproporcionados, sin paginación,
> redundancia/sobrecarga de elementos similares). Todo `PROMPT STITCH` debe respetarlas y, cuando
> aplique, enunciarlas explícitamente al final del prompt.

1. **Una sola representación por dato o acción.** El mismo dato o la misma acción **no** se muestra
   en dos lugares. Ej.: los **conflictos** son **UNA** sección destacada, **no** una tarjeta KPI
   *y además* un panel. Cada CTA aparece **una vez** (prohibido: botón superior + fila inferior de
   "accesos directos" que lo repita).
2. **Coherencia numérica.** Si un contador dice "N", los ítems listados **deben ser N**; si se
   muestran menos, usar explícito **"mostrando X de N · Ver todos"**. Nunca un número que no calce
   con lo listado (v1 decía "4 conflictos" y listaba 3 — prohibido).
3. **Paginación / conteo en toda lista.**
   - **Pantallas de LISTA** (Horarios, Documentos, Auditoría, Usuarios, Fichas, etc.) →
     **paginación REAL y visible**: barra de paginación (‹ anterior · 1 2 3 … › siguiente),
     **selector de tamaño de página** (`page_size`: 10/20/50) y texto **"Mostrando X–Y de N"**,
     alineado a los parámetros `page`/`page_size` del contrato (o `cursor`/`limit` donde el
     contrato use cursor: audit, sessions). Nunca una tabla infinita sin control.
   - **WIDGETS de resumen** (dashboard) → **NO** llevan paginación; muestran **"top N · Ver
     todos"** con "Mostrando X de N" (como el dashboard v2). El "Ver todos" navega a la pantalla
     de lista real (que sí pagina).
4. **Densidad controlada / jerarquía.** Máx **3–4** tarjetas KPI; **una** sección principal por
   pantalla; **sin** filas de accesos directos que dupliquen la nav lateral o los enlaces de las
   tarjetas. Evitar repetir muchos elementos del mismo tipo (no sobrecargar).
5. **Identidad/rol una sola vez.** Nombre y rol del usuario van en el **menú de usuario** (barra
   superior). No repetirlos en el sidebar ni en saludos redundantes.
6. **Datos de ejemplo realistas y proporcionados.** Cantidades verosímiles, textos de longitud
   real (no bloques enormes), números coherentes entre sí y con los contadores.
7. **Sin tercera capa de navegación.** Las acciones viven en la nav lateral + los botones primarios
   de la página; no agregar una capa extra que repita esos destinos.

## Contrato de entrega (output de CADA prompt) — tool-agnóstico

Independiente de la herramienta (Google Stitch **o** ChatGPT/GPT u otra), **cada prompt debe
ENTREGAR SIEMPRE**:

1. **Un ZIP** autocontenido con:
   - **Imágenes PNG** por pantalla: **una desktop (~1440px de ancho)** y **una móvil (~390px)**.
   - **HTML iterativo y funcional** (HTML + CSS + JS), autocontenido o con `assets/` locales, que
     abra en el navegador y refleje los estados (loading/empty/error/interacciones básicas).
2. **Datos de ejemplo realistas** — nombres, fichas, fechas, conteos verosímiles y **coherentes**
   entre sí; **muy cercano a la realidad** del dominio SENA (no lorem ipsum, no cifras absurdas).
3. **Fidelidad al design-system** (tokens, WCAG AA) y a estas reglas de calidad.

> Si la herramienta no exporta ZIP directamente, entregar los mismos artefactos (PNG desktop+móvil
> + HTML) y empaquetarlos. El HTML es la fuente iterativa; los PNG son el render de referencia.

### Coletilla de calidad + entrega (pegar al FINAL de cada `PROMPT STITCH`)

```text
Reglas de calidad: no dupliques ningún dato ni acción (cada CTA una sola vez; no repitas botones que
ya están en la nav lateral o en las tarjetas; no agregues fila extra de accesos directos). Nombre y
rol del usuario solo en el menú superior. Máximo 3–4 tarjetas KPI y una sección principal; densidad
moderada, sin sobrecargar. Si es una PANTALLA DE LISTA, incluye paginación REAL visible (barra ‹
anterior · 1 2 3 · siguiente ›, selector de tamaño de página, y "Mostrando X–Y de N"); si es un WIDGET
de dashboard, usa "top N · Ver todos" sin paginador. Los números deben calzar con lo listado. Datos
de ejemplo realistas y proporcionados (muy cercano a la realidad, sin lorem ipsum).
ENTREGA SIEMPRE: un ZIP con las imágenes PNG (una desktop ~1440px y una móvil ~390px por pantalla) +
el HTML/CSS/JS iterativo y funcional (autocontenido o con assets locales) que abra en el navegador.
```

## Cómo se usa este DDD con Stitch (workflow)

1. **Brief global de estilo** (una vez): se pega al iniciar el proyecto en Stitch —
   ver [§ Brief global de estilo](#brief-global-de-estilo). Fija paleta, tipografía,
   tono, accesibilidad y layout base (app shell por rol).
2. **Prompt por pantalla**: cada archivo en `flows/` trae, por pantalla, un bloque
   **`PROMPT STITCH`** listo para pegar. El prompt describe: propósito, layout, datos
   reales (campos del data-model), acciones (botones ↔ endpoints), y los estados
   loading/empty/error.
3. **Guardar el resultado**: el mockup generado (imagen/HTML export de Stitch) se guarda en
   `mockups/<flujo>/<pantalla>.<png|html>` y se enlaza desde el archivo del flujo.

## Brief global de estilo (pegar primero en Stitch)

> App web responsive para **gestión de horarios de formación del SENA (Colombia)**. Público:
> personal académico. Idioma de la interfaz: **español**. Tono: institucional, claro, sobrio.
> **Design system:** color de marca verde institucional SENA (placeholder hasta ratificar);
> superficies claras, texto de alto contraste (**WCAG 2.1 AA**, ≥4.5:1). Escala de espaciado de
> 4px, esquinas suaves (radius sm/md), sombras sutiles para tarjetas y menús. Tipografía
> sans-serif del sistema (rendimiento en conectividad baja). **Estados de negocio** con color
> semántico + **icono + texto** (nunca solo color): BORRADOR (neutro), EN_REVISIÓN (advertencia),
> PUBLICADO (éxito), **CONFLICTO (crítico)**. Layout: **app shell** con barra superior (marca,
> usuario, notificaciones) y **navegación lateral por rol**. Componentes: tablas densas con
> filtros y paginación, formularios con validación en línea, panel de detalle lateral, toasts,
> modales de confirmación. Todo navegable por teclado, objetivos táctiles ≥44px (uso en tablet).

## Inventario de pantallas por rol (deriva de navigation-map)

| Rol | Flujo (archivo) | Pantallas |
|---|---|---|
| Transversal | [flows/01-auth.md](flows/01-auth.md) | Login · Recuperar contraseña · Nueva contraseña · App shell |
| **Coordinador** (núcleo MVP) | flows/02-coordinador.md | Dashboard · Horarios (lista) · Crear/editar horario · **Panel de conflictos** · Disponibilidad · Fichas |
| Instructor | flows/03-instructor.md | Mi horario (semana) · Mi disponibilidad · Seguimiento de ficha |
| Aprendiz | flows/04-aprendiz.md | Mi horario (semana) · Notificaciones |
| Administrador | flows/05-administrador.md | Panel de indicadores (KPIs) · Administración (usuarios, datos de referencia) |
| Soporte back-office | flows/06-backoffice.md | Documentos · Auditoría · Catálogos/parametrización |

> El orden de prioridad para generar mockups sigue el MVP: **Auth → Coordinador → Instructor →
> Aprendiz → Administrador → back-office.**

## Convención de cada archivo de flujo (`flows/*.md`)

Por cada pantalla:
- **Encabezado:** nombre, ruta, rol, **HU**, **endpoints** (del openapi), **tabla(s)/campos** (del data-model).
- **Propósito** (1–2 líneas).
- **Layout** (regiones y componentes).
- **Datos mostrados** (tabla campo→origen real).
- **Acciones** (botón/control → endpoint → resultado).
- **Estados** (loading / empty / error / success; y conflicto donde aplique).
- **`PROMPT STITCH`** (bloque listo para pegar).
- **Mockup:** enlace a `mockups/...` una vez generado.
