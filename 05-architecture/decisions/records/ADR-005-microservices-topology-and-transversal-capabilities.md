# ADR-005: Topología de repos, micro-frontends y capacidades transversales

**Estado:** ACCEPTED (2026-08-14)
**Fecha:** 2026-08-14
**Autores:** Jesús Ariel González Bonilla (PM + Arquitecto)
**Equipos involucrados:** Arquitectura, Backend, Frontend, DevOps

---

## Contexto

El proyecto confirma **microservicios** (backend) + **micro-frontends** (frontend). Al aterrizar
la topología de repos surgieron ambigüedades e incoherencias que, de no fijarse, generan acople de
despliegue y "monolito distribuido":

1. **Frontera de repo vs de contenedor** sin definir: no estaba claro si `-api`/`-worker`/`-workflow`
   eran repos o componentes.
2. **Mismatch de slug** entre capas: `reference-data-service` ↔ `reference-mfe`,
   `academic-management-service` ↔ `academic-mfe`, `training-environment-service` ↔ `environment-mfe`.
3. **Doble fuente de contratos**: OpenAPI en `docs/07-api` y, a la vez, un repo `shared-contracts`.
4. **Gateway y BFF solapados**: un BFF único agregaba los 9 dominios (mini-monolito) y su relación
   con el gateway no estaba fijada.
5. **Capacidad transversal enterrada**: `notification` vivía como worker dentro de `monitoring`,
   aunque muchos dominios necesitan notificar (igual que generar PDF).
6. **Invariante anti-solape** de horarios resuelto con validación asíncrona (ventana de carrera).

## Decisión

### 1. Frontera de repo = bounded context; frontera de contenedor = proceso
Cada microservicio es **un repo**. Sus componentes desplegables (`-api`, `-worker`, `-workflow`)
son **contenedores dentro** del repo (cada uno su Dockerfile), comparten dominio + BD del servicio.
No se crean repos por componente (evita duplicar dominio / lib compartida = monolito distribuido).

### 2. Slug canónico único por dominio
Un dominio = un slug (el del repo `-db`, inmutable). Todas las capas lo derivan:
`<dominio>-db`, `<dominio>-service`, `<dominio>-mfe`. Se renombran 3 remotes:
`reference-mfe → reference-data-mfe`, `academic-mfe → academic-management-mfe`,
`environment-mfe → training-environment-mfe`.

### 3. Capacidades transversales = servicio propio con contrato público
Un worker/workflow **nunca** se invoca desde otro dominio; es privado de su servicio. Una capacidad
que necesitan varios dominios **es un servicio** con API + eventos; los demás son clientes.
- **PDF / documentos** → `document-service` (ya transversal).
- **Notificar (email/in-app)** → se **extrae** `notification-service` (+ `notification-db`); sale de
  `monitoring`. `monitoring` detecta (KPIs/alertas) y **publica eventos**; `notification` entrega.
  La UI de notificaciones (campana/panel/inbox) es **chrome global del `shell-host`**, sin mfe propio.

### 4. Micro-frontends = hosts + remotes (Module Federation)
- **Remotes**: un mfe por dominio (9), responsive, deploy independiente; consume solo SU contrato.
- **Hosts**: `web-shell` (fase 1) y `mobile-shell` (fase 2) embeben los remotes.
- **BFF por experiencia**: `web-bff` + `mobile-bff` (no un BFF único).

### 5. Contratos: fuente única build-time
`shared-contracts` es la **única** fuente (OpenAPI + envelope de eventos + error-codes),
versionado aditivo/retrocompatible (incompatible ⇒ ADR). `docs/07-api` **apunta** ahí, no duplica.

### 6. Camino de request fijado
`cliente → gateway (authN JWT · routing · rate-limit) → BFF (agrega) → microservicios`.
El BFF no re-verifica identidad ni se salta el gateway; el gateway no agrega datos.

### 7. Invariante anti-solape transaccional
La exclusión mutua (instructor/ambiente/franja) se enforce con **`EXCLUDE` constraint** en
`scheduling_db` (scheduling es dueño de todas las `ClassSession`). El `conflict-validator-worker`
queda como **UX** (mostrar conflictos al armar el borrador), no como mecanismo de enforcement.

## Consecuencias

### Positivas
- Coherencia 1:1 db ↔ service ↔ mfe sin contradicción de nombres.
- Sin acople de despliegue: dominio compartido solo dentro del repo; contratos versionados build-time.
- Capacidades transversales sin worker "omnisciente" ni acople cross-dominio.
- Invariante crítico garantizado por el motor, no por consistencia eventual.

### Negativas / Trade-offs
- +2 repos por extraer `notification` (`notification-service` + `notification-db`).
- Renombrar 3 remotes (y mapear las carpetas del mockup, que conservan nombre corto).
- 38 repos de código exigen gobierno uniforme (4 ambientes + git-flow) y tooling de CI/CD
  (ver [nota de actualización](#nota-de-actualización-2026-09-09--repos-de-soporte) para el inventario real).

### Riesgos
- **Sobre-fragmentación de contenedores**: mitigación — cada servicio arranca con su `-api`; los
  workers 🟡 (`conflict-validator`, `alert`, `scheduling-engine`) se extraen por necesidad probada.
- **Deriva de contratos**: mitigación — `shared-contracts` versionado + Pact broker en infra.

## Alternativas consideradas

| Alternativa | Por qué se descartó |
|-------------|---------------------|
| Modular monolith | Correcto a esta escala en mérito puro, pero el proyecto decide microservicios (incl. objetivo formativo). Se conservan sus fixes válidos (invariante transaccional, federar authN, BFF por experiencia). |
| 1 repo por componente desplegable (~17) | Componentes de un servicio comparten dominio+BD → duplicación o lib compartida = monolito distribuido. |
| BFF único | Reacopla los 9 dominios; anti-patrón. Se usa BFF por experiencia. |
| `notification` dentro de monitoring | Capacidad transversal enterrada; acopla a todos los que notifican. Se extrae. |
| MFE espejo con nombres cortos | Rompe el slug canónico 1:1. Se unifican a forma larga. |

## Nota de actualización (2026-09-09) — repos de soporte

Esta nota **no cambia la decisión**; corrige el inventario, que estaba incompleto frente a la
organización real. El árbol de dominio de este ADR cubre los repos de **código**; existen además
repos de **soporte** que operan hace tiempo y no estaban registrados aquí.

| Categoría | Cantidad | Modelo de ramas |
|---|---|---|
| Repos de código (10 `-db` + 10 `-service` + 3 transversales backend + 13 frontend + infra + e2e) | 38 | `develop → qa → staging → main` |
| Repos documentales / de soporte | 3 | **solo `main`** + rama efímera → PR → borrar |
| **Total en la org `code-sena`** | **41** | — |

Los 3 repos documentales son:

| Repo | Propósito |
|---|---|
| `design-software-docs` | Fuente de verdad documental (este repo). |
| `design-software-hu-log` | Registro de HU por microservicio + `STATUS.md`, el dashboard de estado repo × capa × ambiente. |
| `design-software-pages` | Mockups SDD publicados por GitHub Pages (renombrado desde `design-software-mockup` el 2026-09-09). |

El modelo de ramas por tipo de repo está fijado en
[`git-conventions.md`](../../../00-governance/git-conventions.md).

**Cómo se mantiene sincronizado.** El inventario canónico vive en
[`topology.json`](https://github.com/code-sena/design-software-hu-log/blob/main/topology.json)
del repo `hu-log`; su script `generate-status.ps1` compara esa lista contra la organización real y
reporta cualquier repo nuevo sin registrar, repo faltante o repo cuyo modelo de ramas no cumple.
Al crear un repo nuevo hay que registrarlo ahí **y** actualizar esta nota.

## Referencias

- `TOPOLOGIA-REPOS.md` (workspace `design-software/`, fuera del repo docs) — árbol de los 38 repos de código
- [ADR-001](./ADR-001-message-broker.md) · [ADR-002](./ADR-002-scheduling-read-models.md) · [ADR-003](./ADR-003-object-storage.md)
- [service-catalog.md](../../../09-microservices/service-catalog.md) · [data-ownership-matrix.md](../../../09-microservices/data-ownership-matrix.md) · [event-catalog.md](../../../09-microservices/event-catalog.md)
- [micro-frontends.md](../../../12-ux-ui/mockup-ddd/micro-frontends.md)
