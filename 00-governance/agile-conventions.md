# Convenciones ágiles

> Estado: 🟢 Estable | Última actualización: 2026-08-01
> Autor: Jesús Ariel González Bonilla (PM + Arquitecto) | Equipo: Por definir

Este documento define cómo nombrar, estructurar y referenciar los artefactos ágiles del proyecto. **Este repositorio no persiste las historias de usuario**: el tracker externo (Jira, Trello, GitHub Projects, Azure DevOps u otro) es la fuente de verdad para el seguimiento. Aquí solo viven las convenciones y plantillas.

## Jerarquía de artefactos

El proyecto maneja **dos capas de artefactos** que conviven y se mapean entre sí:

**Capa de producto** (planeación — vive en este repo de docs):

```
Épica de producto   EPC-NN
  └── Feature          FEA-NNN
       └── Historia     HU-NN            (id de planeación del backlog)
```

**Capa de ejecución** (construcción — vive en el board GitHub Project #3, por microservicio):

```
Épica de ejecución   EP-<MICRO>-NN       (un microservicio puede tener N épicas)
  └── Historia         HU-<MICRO>-NNN     (numeración continua por microservicio)
       └── Criterio de aceptación   AC-NNN
            └── Caso de prueba        TC-NNN
```

Regla puente: cada `EP-<MICRO>-NN` **realiza** una o más `EPC-NN`, y cada `HU-<MICRO>-NNN`
traza a su requisito funcional (`RF-*`) y, cuando aplique, a su historia de planeación
(`HU-NN`). El mapeo canónico vive en [`04-requirements/traceability-matrix.md`](../04-requirements/traceability-matrix.md).

## Naming

### Project Key — ⚫ Deprecado

El formato `PRJ-<DOMINIO>-<PRODUCTO>` y el prefijo largo de HU `HU-PRJ-SENA-HORARIOS-NNN`
**quedan deprecados** y no se usan. Se reemplazan por los IDs por microservicio de la capa
de ejecución (`HU-<MICRO>-NNN`). No se elimina esta nota para preservar el histórico de la
decisión.

### Épicas de producto

Formato: `EPC-NN: Título descriptivo`

- `NN`: número secuencial (`EPC-01`, `EPC-02`…). Numeración continua en todo el proyecto.
- El título describe el objetivo de negocio, no la solución técnica.
- Viven en la capa de producto ([`03-product/product-backlog.md`](../03-product/product-backlog.md)).

| Ejemplo válido | Ejemplo inválido |
|----------------|-----------------|
| `EPC-01: Identidad y control de acceso` | `EPIC-1: Módulo login` |
| `EPC-09: Seguimiento, KPIs y alertas` | `EPC-9: Backend monitoreo` |

### Épicas de ejecución

Formato: `EP-<MICRO>-NN: Título`

- `<MICRO>`: microservicio en mayúsculas sin sufijo (`IAM`, `NOTIF`, `SCHEDULING`…).
- `NN`: secuencial **por microservicio** (`EP-IAM-01`, `EP-IAM-02`…).
- **Un microservicio puede tener varias épicas de ejecución.** Cada una realiza una o más
  `EPC-NN` (ver mapeo en la matriz de trazabilidad).
- Viven en el board GitHub Project #3.

| Ejemplo válido | Ejemplo inválido |
|----------------|-----------------|
| `EP-IAM-01: iam-service (identidad, autenticación y autorización)` | `EP-IAM` (sin número) |
| `EP-NOTIF-01: notification-service (entrega transversal)` | `EPC-IAM-01` (mezcla capas) |

### Features

Formato: `FEA-NNN: Título`

- Agrupan HUs dentro de una épica de producto.
- `NNN`: número secuencial global del proyecto.

### Historias de usuario

Las HU tienen **dos identificadores según la capa**:

- **Planeación (producto):** `HU-NN` — id del backlog en [`04-requirements/user-stories.md`](../04-requirements/user-stories.md).
- **Ejecución (construcción):** `HU-<MICRO>-NNN` — **id canónico de trabajo**, numeración
  continua por microservicio, en el board Project #3.
- El título sigue la forma: `[verbo infinitivo] [objeto] [contexto opcional]`.

| Ejemplo válido | Ejemplo inválido |
|----------------|-----------------|
| `HU-IAM-001: Login, bloqueo por intentos y auditoría` | `HU-PRJ-SENA-HORARIOS-001` (deprecado) |
| `HU-NOTIF-004: Resiliencia — reintentos, DLQ e idempotencia` | `HU-4: Feature reintentos` |

### Criterios de aceptación

Formato: `AC-NNN` — ligado a su HU.

### Casos de prueba

Formato: `TC-NNN` — ligado a su AC o HU.

### Severidades de defecto

| Severidad | Descripción |
|-----------|-------------|
| `P0` | Bloquea el release — corrección inmediata |
| `P1` | Debe resolverse en el mismo sprint |
| `P2` | Mejora priorizable para el siguiente sprint |
| `P3` | Deuda menor, se agenda sin urgencia |

## Sprint

- Formato sugerido: `Sprint NN` o según la herramienta del equipo.
- Las fechas y capacidad se definen en el tracker externo.
- Cada sprint tiene un `sprint-plan` asociado en `15-project-control/`.

## Estatus de HU

Los siguientes estatus aplican a todo el proyecto y deben reflejarse tanto en el tracker como en la documentación.

| Estatus | Descripción | Quién avanza |
|---------|-------------|--------------|
| `Backlog` | Identificada, sin analizar ni refinar | Producto |
| `Refinamiento` | En análisis activo con el equipo | Producto + Desarrollo |
| `Listo` | Refinada, estimada y aprobada para iniciar | Equipo completo |
| `En desarrollo` | Asignada y en trabajo activo | Desarrollo |
| `En revisión` | Implementación completa, en revisión de par | Revisor |
| `En QA` | En validación funcional y técnica | QA |
| `Hecho` | Validada, aceptada y cerrada | Producto |
| `Cancelada` | Descartada por cambio de alcance o prioridad | Producto |

Una HU no puede pasar a `Listo` sin cumplir el [Definition of Ready](./definition-of-ready.md).
Una HU no puede pasar a `Hecho` sin cumplir el [Definition of Done](./definition-of-done.md).

## Plantilla de HU

Usar la plantilla en [`04-requirements/_template-hu.md`](../04-requirements/_template-hu.md).

## Integración con el tracker

El tracker del proyecto es **GitHub Project #3** (org `code-sena`):
<https://github.com/orgs/code-sena/projects/3>. Este repositorio no duplica el contenido
del board; cada documento que referencie una HU incluye el enlace directo a su issue.

| Artefacto | Formato de enlace |
|-----------|------------------|
| Épica de ejecución | `[EP-<MICRO>-NN](https://github.com/orgs/code-sena/projects/3)` o al issue de la épica |
| Historia de usuario | `[HU-<MICRO>-NNN](https://github.com/code-sena/<repo>/issues/<numero>)` |

## Trazabilidad

Cada HU debe poder rastrearse hacia:

| Desde | Hacia |
|-------|-------|
| HU (`HU-<MICRO>-NNN`) | Épica de ejecución (`EP-<MICRO>-NN`) que la contiene |
| HU | Épica de producto (`EPC-NN`) que realiza |
| HU | Feature (`FEA-NNN`) a la que pertenece |
| HU | Sprint en que se planificó |
| HU | Requisito funcional en `04-requirements/functional.md` |
| HU | Caso de prueba (`TC-NNN`) en `11-quality/` |
| HU | ADR si generó una decisión arquitectónica |

La matriz completa vive en [`04-requirements/traceability-matrix.md`](../04-requirements/traceability-matrix.md).
