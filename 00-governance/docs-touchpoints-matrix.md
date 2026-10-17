# Matriz de docs-touchpoints (gobernanza docs-first)

> Estado: 🟢 Estable | Última actualización: 2026-08-28
> Autor: Jesús Ariel González Bonilla (PM + Arquitecto) | Equipo: Gobernanza

Toda HU y todo cierre de épica **propaga cambios a las carpetas de docs que le correspondan**, en la
**misma entrega**. El repo docs es la fuente de verdad (SSOT); si una carpeta deja de reflejar lo
construido, se pierde la trazabilidad de punta a punta. Esta matriz fija **qué carpeta es obligatorio
revisar/actualizar** según lo que la HU toque, para que la deriva no vuelva a ocurrir.

## Por qué existe esta regla (precedente)

En los dos primeros micros (**iam-service** 12 HU, **notification-service** 8 HU) la construcción avanzó
pero el docs quedó **congelado en el baseline del 1-ago**: 7 carpetas sin un solo cambio pese a 20 HU
entregadas (trazabilidad con 0/20 HU, `domain-events` omitiendo un evento en producción, `testing-strategy`
negando pruebas ya ejecutadas, `roadmap` afirmando que "no existe capa de aplicación"). La causa raíz no
fue error de contenido sino **falta de sync docs-first al cerrar cada HU/épica**. Esta matriz es el
guardarraíl que lo previene. Ver también [definition-of-done.md](./definition-of-done.md) y la regla de
prioridad de mantenimiento (docs → backlog/board → código).

## Principio

- **docs primero**: los cambios de docs que la HU toca son **parte prioritaria del entregable**, no opcionales.
- **El estado vivo del avance vive en el board** (GitHub Project #3). Los `.md` son SSOT del **contenido**
  (requisitos, dominio, contrato, arquitectura, calidad), **no** del estado de tareas: no deben duplicar el board.
- **La HU/épica no es Done** hasta que sus touchpoints obligatorios estén actualizados; el revisor central lo verifica al mergear.

## Touchpoints por CADA HU (mínimo obligatorio)

Aplica a toda HU, sin excepción:

| Carpeta / archivo | Cuándo | Qué actualizar |
|---|---|---|
| `04-requirements/traceability-matrix.md` + `user-stories.md` | **Siempre** | Fila de la HU (ID real `HU-<SVC>-###`, RF, épica, issue docs#, estado `Done·Dev`) + su historia con criterios de aceptación. |
| `09-microservices/services/<svc>/` | **Siempre** | `data-model.md`, `runbook.md`, `components/` y demás docs del servicio afectados por la HU. |
| `06-data/data-dictionary.md` + `09-.../<svc>/data-model.md` | Si toca **schema** (`-db`) | Tabla/columna/índice/grant nuevos, referenciando el changeset. **Regla endurecida** (ver DoD). |
| `07-api/contracts/` | Si toca **contrato** (endpoint/evento/error) | OpenAPI / event schema en shared-contracts + su reflejo en docs. |
| Board (GitHub Project #3) | **Siempre** | Estado de la HU (Status + Environment) e issue vinculado. |
| `design-software-hu-log/log/<svc>.md` | **Siempre** | Fila de la HU + hallazgos ("lo que aprendimos"). |

## Touchpoints por CIERRE DE ÉPICA / MICRO (barrido completo)

Al cerrar una épica (micro completo), además de lo anterior, barrer **todas** estas carpetas:

| Carpeta | Qué revisar al cerrar el micro |
|---|---|
| `01-context/glossary.md` | Términos nuevos introducidos por el micro. |
| `02-domain/` | `domain-events.md` (eventos reales emitidos/consumidos), `domain-map.md` (bounded context + relaciones/Published Language), `entities-and-rules.md` (RN nuevas). |
| `03-product/roadmap.md` | Avance de fase (micro `Done·Dev`), ubicación del micro. `product-backlog.md` NO duplica el board (es índice → Project #3). `vision.md` solo si cambia una capacidad. |
| `04-requirements/functional.md` + `non-functional.md` | RF del módulo + NFR con evidencia en dev. |
| `05-architecture/` | ADR nuevo si hubo decisión; C4/vista de componentes. |
| `08-uml/` | Diagrama(s) de secuencia de los flujos construidos (login, outbox→relay, consumo/DLQ, etc.) + ER derivable del schema. |
| `10-devops/` | `environments.md`, `local-setup.md` (bring-up del micro, gotchas de deploy), `ci-cd.md` (checks del micro, gap de CI). |
| `11-quality/` | `testing-strategy.md` (pirámide real + lecciones), `code-review.md` (checks reales). |
| `12-ux-ui/` | Si el micro tiene UI/mockup. |
| `15-project-control/` | `risks.md` / `dependencies.md` / `technical-backlog.md` (riesgos resueltos, dependencias en uso, progreso). |

> No toda HU toca todas las carpetas: la columna "Cuándo" manda. Pero **al cierre del micro, el barrido
> completo es obligatorio** — es el momento en que se detecta y corrige cualquier carpeta que quedó atrás.

## Convención de IDs (para que la trazabilidad no derive)

Un solo esquema de identificadores, el de ejecución, en docs y board:
`EP-<SVC>` (épica) · `HU-<SVC>-###` (historia) · issue `docs#` (tracker). No conviven esquemas paralelos
(`HU-##`/`EPC-##` legados): se deprecan o se mapean explícitamente.

## Enlaces

- [definition-of-done.md](./definition-of-done.md) — la HU/épica no cierra sin sus touchpoints.
- [documentation-rules.md](./documentation-rules.md) — naming, estados, estructura.
- [microservices-documentation.md](./microservices-documentation.md) — cómo documentar el servicio.
