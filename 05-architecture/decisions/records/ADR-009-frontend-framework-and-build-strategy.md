# ADR-009: Framework de frontend y estrategia de build para micro-frontends

**Estado:** ACCEPTED (2026-09-09, rev. 2)
**Fecha:** 2026-09-09 (rev. 2: cierra 5 huecos de robustez detectados en autoauditoría — CSS, remotes
por ambiente, ruteo anidado, falla de carga, gate de compatibilidad)
**Autores:** Jesús Ariel González Bonilla (PM + Arquitecto)
**Equipos involucrados:** Arquitectura, Frontend, Formación (aprendices SENA)

---

## Contexto

[ADR-005](./ADR-005-microservices-topology-and-transversal-capabilities.md) fijó la topología de
frontend (9 remotes + 2 hosts vía Module Federation) y
[micro-frontends.md](../../../12-ux-ui/mockup-ddd/micro-frontends.md) fijó la arquitectura de
construcción (qué MFE posee qué pantalla, reglas de composición). Ninguno de los dos fija **con qué
framework** ni **con qué build tool** se construyen esos 13 componentes runtime — esa decisión nunca
se formalizó.

En la práctica, los 13 repos (`web-shell`, `mobile-shell`, `design-system`, `frontend-platform` y los
9 `-mfe`) ya tienen un esqueleto committeado el 2026-08-14 con **React 18 + TypeScript** de forma
uniforme (mismo `package.json`, mismo `tsconfig.json` con `moduleResolution: "Bundler"`). Esta ADR
**formaliza esa decisión ya vigente**, en vez de reabrirla — evita cargar a 13 repos ya consistentes
con el costo de una migración de framework sin beneficio real. No se aplica el mismo criterio
"polyglot fit-driven" de [ADR-006](./ADR-006-polyglot-stack-strategy.md) al frontend: a diferencia de
microservicios backend (procesos aislados), varios remotes comparten el **mismo runtime del
navegador** — mezclar frameworks en runtime implica cargar varios runtimes de UI a la vez, duplicar
el design system (framework-agnóstico o N implementaciones) y complicar el punto de mayor riesgo
técnico (el host montando/desmontando remotes). El costo de mezclar supera el valor formativo de la
variedad, que ya se cubre con el polyglot real del backend (Go/Java/Python/Node).

Auditoría del esqueleto real (2026-09-09) encontró que, aunque el framework está bien elegido, **no
hay convención de capas, sin harness de testing, y sin configuración real de Module Federation en
ningún repo** — todo lo que dice "Module Federation" hoy es descriptivo (README/comentarios), no
código que funcione. Cero de los 5 pilares (DDD interno, TDD, SDD conectado a código, SOLID
implementado, patrones de MFE) está resuelto todavía; esta ADR es el primer paso para resolverlos.

## Decisión

### 1. Framework: React 18 + TypeScript, único, para los 11 remotes/hosts

**React 18 + TypeScript** para `web-shell`, `mobile-shell` y los 9 `-mfe`. `design-system` y
`frontend-platform` se mantienen **framework-agnósticos** (TypeScript puro, sin dependencia de
React) — ya están así en el esqueleto actual, correcto: son consumidos por remotes que en el futuro
podrían no ser React, y el design system en particular se beneficia de ser Web Components si algún
día se reabre esta decisión.

No se fija un segundo framework ahora. Si en el futuro un clúster de 2-3 remotes justifica un segundo
framework (mismo criterio de ADR-006: amortización por clúster, no "cada quien el suyo"), se abre una
ADR nueva — no se reabre esta.

### 2. Build tool: Vite + Module Federation (`@module-federation/vite`)

Se elige **Vite** sobre webpack clásico: los `tsconfig.json` ya committeados usan
`"moduleResolution": "Bundler"` (idiomático de Vite/esbuild, no de webpack `node` resolution) — la
elección ya estaba implícita. Plugin: `@module-federation/vite` (Module Federation 2.0), que
sustituye al plugin nativo de webpack y es el estándar 2026 para MF sobre Vite.

Cada remote expone su `App` vía `exposes`; el host consume vía `remotes`, con `react`/`react-dom`
marcados `shared: { singleton: true, requiredVersion: "^18.3.1" }` — sin esto, host + remote cargan
React duplicado y falla en runtime ("Invalid hook call"). Ver plantilla completa en la skill
`design-software-frontend-build`.

### 3. Capas internas por MFE (DDD aplicado a frontend — mismo principio de ADR-006)

Mismo principio rector de ADR-006 ("DDD + Hexagonal + TDD + SDD aplican por igual en todos los
stacks"), adaptado a frontend:

```
domain/       modelos, value objects, reglas de negocio de UI puras (sin React, sin fetch)
application/  hooks de caso de uso (uno por HU), puertos (interfaces de servicios que consume)
adapter/      in/ (componentes de presentación, páginas) · out/ (implementación de los puertos:
              llamadas HTTP reales vía frontend-platform, mapeo DTO↔dominio)
```

Regla de dependencia idéntica al backend: `componente de presentación → hook de aplicación →
dominio`; el dominio no importa React ni el `HttpClient` concreto — solo el puerto (interfaz).

### 4. TDD: Vitest + Testing Library

**Vitest** (nativo de Vite, mismo motor de build que dev/prod — sin config duplicada) +
**@testing-library/react** para tests de componentes orientados a comportamiento, no a implementación.
Todo hook de aplicación (caso de uso) lleva unit test; todo componente de adapter/in lleva un test de
integración con Testing Library. DoD de cada HU de frontend exige tests en verde, igual que backend.

### 5. SDD: contrato → tipos, spec de diseño → tokens

- **Tipos de API:** `openapi-typescript` genera tipos desde `shared-contracts/openapi/<dominio>.yaml`
  hacia `contracts/` en cada MFE (mismo patrón que `web-bff`/`mobile-bff` en ADR-006 §5) — nunca tipos
  escritos a mano que puedan divergir del contrato real.
- **Design tokens:** `design-system/src/tokens.ts` se genera/sincroniza desde
  [design-tokens.md](../../../12-ux-ui/mockup-ddd/design-tokens.md) (fuente del spec) — hoy el
  esqueleto tiene los tokens vacíos (`color: {}, space: {}, radius: {}`); es deuda inmediata a resolver
  al construir `design-system`, no un placeholder que se queda así.
- **Regla 4 de micro-frontends.md se mantiene:** el MFE se construye contra el contrato, nunca contra
  el mockup — el mockup es solo referencia de UX, no fuente de tipos ni de componentes.

### 6. Patrones de micro-frontend obligatorios

- **Facade** — `frontend-platform` es la fachada única sobre `fetch`/`localStorage`/mensajería
  (`AuthClient`, `HttpClient`, `EventBus`, `config`). Ningún MFE llama `fetch` directo ni lee
  `localStorage` directo — todo pasa por la fachada, inyectada vía React Context (Inversión de
  Dependencias, mismo rol que los `port/out` del backend).
- **Event Bus** (pub/sub, no acople directo remote↔remote) para comunicación cross-MFE — ya declarado
  en el alcance de `frontend-platform`, pendiente de implementar (hoy el `index.ts` solo tiene
  `AuthClient`/`HttpClient`, sin `EventBus`).
- **Container/Presentational** dentro de cada MFE: `adapter/in/pages` (containers, consumen hooks de
  `application/`) vs. componentes puramente presentacionales (reciben props, sin lógica de datos).
- **Error boundary por remote** — un remote caído no debe tumbar el shell ni a los demás remotes. Ver
  §8 para el mecanismo exacto (Suspense + ErrorBoundary, no solo uno de los dos).

### 7bis. Aislamiento de CSS (obligatorio, no opcional)

Ningún selector global fuera de `design-system` (que sí publica el reset/tokens base, por diseño —
es la única fuente de estilos compartidos). Cada MFE usa **CSS Modules** (`*.module.css`, soportado
nativo por Vite, cero dependencia extra): las clases se scope-an automáticamente por archivo, un
`.button` en `scheduling-mfe` no puede pisar un `.button` en `iam-mfe`. Regla verificable: ningún
archivo `.css` sin sufijo `.module.css` fuera de `design-system/src/`. Adicional: el contenedor raíz
de cada remote monta con `data-mfe="<dominio>"` (ya presente en el `App.tsx` actual) — sirve como
anclaje si algún día se necesita un selector defensivo adicional.

### 8. Carga de remotes: Suspense + ErrorBoundary (no uno solo)

Un Error Boundary **solo** atrapa errores de render — la carga de `remoteEntry.js` es una falla
**async** (remote caído, red, deploy a medias) que un Error Boundary por sí solo no cubre limpio. El
host monta cada remote así:

```tsx
const IamApp = lazy(() => import('iamMfe/App'))

<ErrorBoundary fallback={<RemoteUnavailable domain="iam" />}>
  <Suspense fallback={<LoadingRemote domain="iam" />}>
    <IamApp />
  </Suspense>
</ErrorBoundary>
```

`Suspense` cubre el estado de carga (incluye la descarga de `remoteEntry.js`); `ErrorBoundary` cubre
tanto el rechazo del `import()` (remote inalcanzable) como errores de render una vez montado.
`RemoteUnavailable`/`LoadingRemote` viven en `design-system` (son UI compartida, no lógica de MFE).

### 9. Resolución de remotes por ambiente (4 ambientes, no URLs fijas)

El proyecto opera en **4 ambientes** (dev/qa/staging/main, ver skill `design-software-project`) — la
URL de `remoteEntry.js` de cada remote **no puede** quedar hardcodeada en `vite.config.ts` (eso solo
sirve para dev local). El host resuelve remotes en runtime, no en build time, vía
`@module-federation/vite`'s `initManifest`/`init()` dinámico:

- Un archivo `remotes.<env>.json` (uno por ambiente, no versionado con secretos) lista
  `{ "iamMfe": "https://iam-mfe.<env>.horarios-sena.internal/remoteEntry.js", ... }` — el host lo
  carga según `import.meta.env.MODE` (o la env var que ya use el proyecto para distinguir ambiente,
  ver convención de `docker-infra`) **antes** de inicializar Module Federation.
- Dev local sigue usando `localhost:51xx` (§ Puertos, en la skill) vía el mismo mecanismo, solo que
  `remotes.dev.json` apunta a `localhost`.
- Ningún commit debe tener una URL de `remoteEntry` de staging/main hardcodeada en código — igual
  regla que "nunca secretos hardcodeados" de ADR-006, aplicada a endpoints de remotes.

### 10. Ruteo anidado: shell enruta por dominio, remote enruta internamente

`react-router-dom` se agrega a la lista de `shared: { singleton: true }` (junto a react/react-dom) —
sin esto, shell y remote traen cada uno su propio contexto de router y `useNavigate`/`Link` dejan de
interoperar. Convención de composición:

- El shell declara **una ruta comodín por dominio**: `<Route path="/scheduling/*" element={<IamApp />} />`.
- Cada remote trae su **propio árbol de rutas internas** bajo ese prefijo
  (`/scheduling`, `/scheduling/:id`, `/scheduling/new`), declarado dentro de su propio `<Routes>` — el
  remote es dueño de su sub-árbol, el shell no conoce las rutas internas de ningún dominio (mismo
  principio de "el shell no tiene lógica de dominio").
- Una sola instancia de `BrowserRouter`/historial — la crea el shell; los remotes nunca instancian su
  propio `Router` de nivel superior.

### 11. Gate de compatibilidad host↔remote (equivalente a Pact, resuelto — no queda pendiente)

Dos capas, no una sola:

1. **Contrato de tipos TS** — `frontend-platform/src/contracts/remote-props.ts` exporta una interfaz
   por dominio (`IamMfeProps`, `SchedulingMfeProps`, …) que describe la forma del componente que cada
   remote expone. El `App.tsx` real de cada remote debe **implementar** esa interfaz (`App: FC<IamMfeProps>`
   o equivalente); si un remote cambia su forma expuesta de modo incompatible, **su propio build de TS
   falla** antes de llegar a integrarse con el host — no hace falta esperar a runtime.
2. **Smoke test de integración en el CI del host** — el pipeline de `web-shell` importa dinámicamente
   el `remoteEntry.js` real de cada remote (contra el manifest de QA, no localhost) y lo monta con
   Testing Library, asertando que renderiza sin lanzar. Esto atrapa lo que los tipos no pueden
   (cambios runtime-only, un remote que rompe su propio bootstrap). Una falla aquí **bloquea el merge**
   del host — mismo principio que "build en rojo bloquea" del gate Pact backend (ADR-006 §7).

### 12. Convención de código = ADR-006 §"Prohibido" (mismo estándar, sin duplicar)

Código, nombres, comentarios y mensajes de commit en **inglés**, Conventional Commits
(`feat(<dominio>-mfe): ...`); secretos nunca hardcodeados; solo copy visible al usuario y
documentación funcional en español. Mismo `sena-platform`-style guardrail que backend.

## Consecuencias

### Positivas
- Formaliza lo que ya existe en 13 repos en vez de migrarlo — cero costo de reescritura.
- `design-system`/`frontend-platform` framework-agnósticos dejan la puerta abierta a revisar el
  framework único más adelante sin tocar la capa compartida.
- Mismo principio DDD/TDD/SDD que backend — transferible entre aprendices que rotan de capa.

### Negativas / Trade-offs
- Un solo framework reduce la variedad formativa en frontend frente al polyglot real del backend —
  mitigado explícitamente: la variedad de lenguajes ya se cubre en backend (4 stacks), frontend prioriza
  consistencia dado el riesgo técnico real de mezclar runtimes de UI.
- Migrar a Module Federation 2.0 (`@module-federation/vite`) en vez del plugin webpack clásico es
  menos "battle-tested" en tutoriales existentes — mitigado con la plantilla completa en la skill.

### Riesgos
- Repetir el mismo error que backend ya evitó con `shared-contracts`: MFEs que copian tipos a mano en
  vez de generarlos del contrato → deriva. Mitigación: regla 5 (SDD) + regla 11 (gate de compatibilidad
  host↔remote, contrato de tipos + smoke test en CI).
- Design system con tokens vacíos "por defecto" indefinidamente. Mitigación: es DoR explícito de la
  primera HU de `design-system` (no se cierra la HU sin migrar `design-tokens.md` a código).

## Alternativas consideradas

| Alternativa | Por qué se descartó |
|---|---|
| Polyglot frontend (React + Angular, uno por clúster de dominios) | Evaluado en detalle: el runtime compartido del navegador hace que mezclar frameworks cueste mucho más que mezclar lenguajes backend (procesos aislados). El design system tendría que ser Web Components desde ya o triplicarse. Se retira por ahora; reabrir con ADR nueva si aparece un fit real. |
| Webpack 5 + `ModuleFederationPlugin` (clásico) | Más tutoriales/ejemplos disponibles, pero el `tsconfig.json` ya committeado (`moduleResolution: Bundler`) apunta a Vite; usar webpack forzaría reescribir el tooling ya elegido de facto. |
| Sin capas internas (todo en componentes) | Es lo que hay hoy (`App.tsx` con TODO) — repite el error que backend ya corrigió con hexagonal: lógica de negocio mezclada con framework, imposible de testear en aislamiento. |
| Jest en vez de Vitest | Jest exige config de transform separada para ESM/Vite; Vitest reusa la config de Vite tal cual — menos superficie de mantenimiento para aprendices. |

## Pendientes (fuera de alcance de esta ADR)

- Framework de Web Components para `design-system` si se reabre la decisión de multi-framework.
- Observabilidad frontend (RUM/OTel en navegador) — no cubierto aquí; evaluar junto con
  [ADR-008](./ADR-008-observability-opentelemetry.md) cuando el primer remote piloto esté en QA.

## Referencias

- [ADR-005](./ADR-005-microservices-topology-and-transversal-capabilities.md) — topología micro-frontends (hosts+remotes)
- [ADR-006](./ADR-006-polyglot-stack-strategy.md) — principio DDD+Hexagonal+TDD+SDD, criterio de amortización por clúster
- [micro-frontends.md](../../../12-ux-ui/mockup-ddd/micro-frontends.md) — arquitectura de construcción (MFE↔pantallas)
- [design-tokens.md](../../../12-ux-ui/mockup-ddd/design-tokens.md) · [design-system.md](../../../12-ux-ui/design-system.md)
- Skill `design-software-frontend-build` — convenciones de código derivadas de esta ADR
