<!-- RESUMEN-EJECUTIVO
agente: Jesús Ariel González Bonilla (PM + Arquitecto)
capacidad: estándar de estados por tipo de pantalla + Definition of Done del mockup
fase: diseño (UX/UI)
estado: accepted
dependencias_entrada: mockup-ddd/README.md (reglas/contrato de entrega); design-tokens.md; design-system.md; sitemap.md; ux-copy.md
consumidores_siguientes: generación y validación de cada pantalla del mockup
tldr: Define qué estados debe especificar cada tipo de pantalla y el checklist de aceptación (DoD) que una pantalla generada debe cumplir para marcarse aceptada.
decisiones_clave: matriz tipo×estado obligatoria; DoD verificable por pantalla; los estados no se fragmentan en pantallas, se documentan como estados; render verificado por estilos computados (no basta con que el archivo exista); lecciones de construcción post-generación (§B.1)
halts_registrados: ninguno
-->

# Estados obligatorios + Definition of Done (DoD) del mockup

> **ESTADO: accepted.** Reglas de verificación para dar una pantalla por **aceptada**.

## A. Estados obligatorios por tipo de pantalla

✅ obligatorio · ◦ si aplica · — no aplica

| Tipo | loading | empty | error | success | offline | sin-permiso (403) |
|---|:--:|:--:|:--:|:--:|:--:|:--:|
| **Lista / tabla** | ✅ (skeleton) | ✅ (sin resultados) | ✅ (reintentar) | — | ◦ | ✅ (oculta/403) |
| **Detalle** | ✅ | ◦ | ✅ | — | ◦ | ✅ |
| **Formulario / crear-editar** | ✅ (guardando) | — | ✅ (validación por campo + banner) | ✅ (toast) | ◦ | ✅ |
| **Dashboard / widgets** | ✅ (por widget) | ✅ (positivo: "Sin conflictos") | ✅ (por widget, no bloquea) | — | ✅ (banner) | ◦ |
| **Modal** | ◦ | — | ✅ (en el modal) | ✅ (cierra + toast) | — | — |
| **Calendario (semana)** | ✅ | ✅ (semana sin clases) | ✅ | — | ◦ | ✅ |

- El **estado de conflicto** (scheduling) es siempre color + icono + texto crítico (design-tokens).
- Los estados se documentan **dentro** de la pantalla (sección *Estados*); **no** se fragmentan en
  pantallas separadas (anti-patrón). Excepción: las **páginas de sistema** (403/404/500/sesión) sí
  son una pantalla propia transversal ([01-auth P6](flows/01-auth.md)).
- El banner **"Sin conexión"** es **condicional** (solo offline), no por defecto.

## B. Definition of Done — checklist por pantalla

Una pantalla del mockup se marca **✅ aceptada** cuando cumple TODO:

- [ ] **Deriva del spec:** endpoints y campos son los reales del contrato/data-model (no inventados);
      referencias cross-dominio como composición/BFF, no join físico.
- [ ] **Tokens + accesibilidad:** usa los [design-tokens](design-tokens.md); contraste ≥4.5:1,
      navegable por teclado, objetivos táctiles ≥44px, ARIA en formularios/tablas (WCAG 2.1 AA).
- [ ] **Estados requeridos** presentes según su tipo (tabla A).
- [ ] **Paginación:** listas con paginación real (‹ 1 2 3 › + `page_size` + "Mostrando X–Y de N") o
      cursor donde el contrato lo use; widgets con "top N · Ver todos" (sin paginador).
- [ ] **Sin anti-patrones:** una sola representación por dato/acción; coherencia numérica
      (contador ↔ lista); densidad moderada; rol/usuario solo en el menú superior; sin fila extra de
      accesos directos ([README §anti-patrones](README.md)).
- [ ] **RBAC:** muestra solo lo permitido para el rol (oculta lo no permitido; no controles
      deshabilitados escondidos); acceso directo no permitido → 403 ([sitemap](sitemap.md)).
- [ ] **Copy consistente:** usa la forma canónica de mensajes/labels ([ux-copy](ux-copy.md)); datos de
      ejemplo realistas y proporcionados (sin lorem ipsum).
- [ ] **Responsive:** variante móvil (~390px) y escritorio (~1440px); tabla→tarjetas en móvil.
- [ ] **Contrato de entrega:** el prompt produjo un **ZIP** con PNG desktop + móvil **+ HTML/CSS/JS
      iterativo y funcional** que abre en el navegador ([README §contrato de entrega](README.md)).
- [ ] **Navegación cableada (sin enlaces muertos):** todo enlace de navegación apunta a un archivo
      real (según [sitemap §convención de build](sitemap.md)); **prohibido `href="#"`** para navegar
      (solo válido en toggles JS). Los modales se abren con `data-toggle`, no con enlaces muertos.
      Cada pantalla es **alcanzable** desde el índice y desde la nav de su rol.
- [ ] **Render verificado (no basta con que el archivo exista):** la pantalla se **abre en el
      navegador y se inspecciona el layout renderizado** (estilos computados), no solo su HTML. Toda
      clase referenciada (`.app-sidebar`, `.app-header`, shell, componentes) **tiene CSS definido** en
      el CSS compartido o inline; **cero clases "huérfanas"** (referidas en el markup pero sin regla),
      que renderizan como bloque sin estilo. Verificar: sidebar con ancho/borde real, iconos a tamaño
      fijo (no expandidos por `viewBox`), `body`/shell con layout (flex/grid), no un apilado plano.
- [ ] **Trazabilidad:** la pantalla declara `HU → endpoint → tabla → MFE` en su header.

## B.1 Lecciones de construcción verificadas (post-generación)

Hallazgos reales al auditar el mockup **ya generado** (herramienta agéntica). Se registran para que la
verificación no vuelva a asumir "archivo existe = pantalla funciona":

| # | Síntoma | Causa raíz | Regla que lo previene |
|---|---|---|---|
| L1 | Iconos de nav gigantes (~147px) | SVG con `viewBox` **sin `width/height`** y **sin** regla CSS de tamaño; además clase del markup (`.nav-item`) ≠ clase del CSS (`.sidebar-item`) | Regla de tamaño de icono en el CSS **compartido** por todas las páginas; nombres de clase markup↔CSS deben coincidir |
| L2 | 15 pantallas (admin/backoffice/learner) con **sidebar colapsado sin estilo** | Clases del shell (`.app-header/.app-body/.app-sidebar/.nav-group`) **referenciadas pero nunca definidas** en CSS → renderizan como bloque plano | Criterio **"Render verificado"**: inspeccionar estilos computados; cero clases huérfanas |
| L3 | Fondos incoherentes entre roles (blanco vs hueso) | Cada página duplicaba `:root` inline con **valores divergentes** (`--color-bg: #FFFFFF` vs `#F9FAFB`) | Tokens con **una sola fuente** (CSS compartido); si se duplica `:root`, los valores deben ser idénticos |
| L4 | Colores "mágicos" (hex inline) fuera de tokens | Generación escribió hex crudos en `style=`/`<style>` | DoD "Tokens": **uso 100% `var(--color-*)`**; hex solo en la definición `:root` |

> **Verificación mínima recomendada:** servir el mockup (`http.server`) + abrir cada pantalla y
> comprobar por estilos computados: (a) el contenedor de layout es `flex`/`grid` (no `block` plano),
> (b) el sidebar tiene ancho/borde, (c) los iconos miden ~20px, (d) no hay hex crudos en líneas de uso.

## C. Estado de aceptación
El estado (⬜/✅) de cada pantalla se lleva en [screen-inventory.md](screen-inventory.md). Una pantalla
solo pasa a ✅ cuando cumple el DoD de arriba.
