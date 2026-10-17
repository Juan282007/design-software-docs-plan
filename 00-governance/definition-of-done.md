# Definition of Done

> Estado: 🟢 Estable | Última actualización: 2026-08-01
> Autor: Jesús Ariel González Bonilla (PM + Arquitecto) | Equipo: Por definir

Un documento está **done** cuando:

- [ ] Fue aprobado por al menos un revisor
- [ ] Está enlazado desde el `README.md` de su sección
- [ ] El estado cambió a 🟢
- [ ] Los enlaces relativos funcionan
- [ ] No contiene información sensible
- [ ] El CHANGELOG fue actualizado si el cambio modifica gobernanza, estructura de carpetas, o un contrato compartido (API, modelo de datos, convenciones de Git)

## Cambios que modifican schema de BD (regla endurecida 2026-08-25)

Toda **HU o cambio que modifique el schema de una base de datos** (nueva tabla, columna, índice o grant en un repo `-db`) **NO está done** hasta que, en la **misma entrega**, actualice:

- [ ] El `data-model.md` del servicio dueño (`09-microservices/services/<svc>/`).
- [ ] El diccionario de datos (`06-data/data-dictionary.md`).

...referenciando el **changeset** que introdujo el cambio (tabla/columna/índice: nombre, tipo, propósito). Sin este sync, el docs SSOT deja de reflejar la BD real y se pierde trazabilidad. Aplica a todos los servicios; el revisor central lo verifica antes de mergear.

## Sync de docs por HU y por cierre de épica (regla endurecida 2026-08-28)

Toda **HU** y todo **cierre de épica/micro** **NO está done** hasta que actualice, en la **misma entrega**,
las carpetas de docs que le correspondan según la [matriz de docs-touchpoints](./docs-touchpoints-matrix.md):

- [ ] **Por cada HU (mínimo):** `04-requirements` (trazabilidad + user-story con estado), `09-microservices/<svc>`, `06-data` si toca schema, `07-api` si toca contrato, el board (Project #3) y el hu-log.
- [ ] **Al cerrar el micro (barrido completo):** además `01-context`, `02-domain`, `03-product`, `05-architecture`, `08-uml`, `10-devops`, `11-quality`, `15-project-control` (y `12-ux-ui` si hay UI).

Precedente: en los 2 primeros micros el docs derivó por falta de este sync (7 carpetas congeladas pese a 20 HU).
El revisor central verifica los touchpoints antes de dar la HU/épica por cerrada. Orden de mantenimiento:
**docs SSOT primero → backlog/board → repos de código** (gobernanza de punta a punta).

## Done por ambiente — HU de código (regla 2026-09-09)

El `Done` de una **HU de código** es **relativo al ambiente** al que se promovió. El
registro de la HU (board Project #3 + hu-log) **debe declarar hasta qué ambiente llegó**:
`Done·Dev`, `Done·QA`, `Done·Staging` o `Done·Main`. Los criterios son **acumulativos**
(cada gate exige los anteriores) y se apoyan en el flujo de ambientes de
[`10-devops/environments.md`](../10-devops/environments.md) y las
[convenciones de Git](./git-conventions.md).

| Gate | Criterios adicionales para alcanzarlo |
|------|----------------------------------------|
| **`Done·Dev`** | HU implementada; tests unitarios + de integración verdes; migraciones aplicadas a la BD `develop`; contrato en `shared-contracts` actualizado si aplica; sync de docs por HU (secciones anteriores); PR aprobado y mergeado a `develop`; entrada creada en el `hu-log`. |
| **`Done·QA`** | Promovido a `qa` por PR/cherry-pick; migraciones **forward-only** aplicadas en la BD `qa`; validación funcional y técnica registrada. |
| **`Done·Staging`** | Promovido a `staging`; validación de preproducción registrada. |
| **`Done·Main`** (Producción) | Promovido a `main` vía `release/*`; documentación estable; release anotado. |

Reglas:

- Una HU **no puede marcarse `Hecho`** (estatus final en [agile-conventions](./agile-conventions.md)) sin declarar su `Done·<ambiente>`. `Hecho` sin ambiente es ambiguo y **no se acepta**.
- Aplica a los **repos de código** (`-db`/`-service`/`-mfe`/`-infra`). El repo `-docs` es main-only y usa el DoD documental del inicio de este archivo.
