---
tipo: refinamiento-requerimientos
proyecto: Sistema de Gestión de Horarios SENA (code-sena)
version_plantilla: 1.0
revisor: EJEMPLO — Nombre Apellido
ficha: 3145555
rol_en_proyecto: Modelador de datos
fecha: 2026-08-07
fuente_feedback: https://drive.google.com/drive/folders/17ek3tHMEv1tKWP7nYwjDp0POE3-iYzJi
mockup_revisado: https://code-sena.github.io/design-software-mockup/
---

# Refinamiento de Requerimientos — Revisión individual (EJEMPLO)

> Este archivo es solo una **muestra** de cómo llenar la plantilla. Copia
> `PLANTILLA-refinamiento.md`, NO este ejemplo, para tu propia revisión.

## A. Refinamiento de REQUERIMIENTOS

### Resumen de decisiones — Requerimientos
| ID | Propuesta (resumen) | Decisión | ¿Bloquea backend? |
|---|---|:--:|:--:|
| P-REQ-01 | El coordinador debe poder duplicar un horario de una ficha a otra | 🔧 | No |

### Detalle
#### [P-REQ-01] Duplicar horario entre fichas
- **Fuente:** Laura V. · `feedback-coordinador.docx` (Drive)
- **Propuesta:** Botón "duplicar horario" que copie las sesiones a otra ficha.
- **Decisión:** 🔧 ACEPTAR CON AJUSTE
- **Justificación técnica:** La idea es válida (ahorra tiempo), pero copiar sesiones exige revalidar
  conflictos de instructor/ambiente en la ficha destino; no puede ser copia ciega. Debe pasar por el
  mismo motor de detección de conflictos (`scheduling.scheduling_conflict`).
- **Ajuste propuesto:** "Duplicar como borrador" → crea un `schedule` en estado `DRAFT` y corre la
  validación de conflictos antes de permitir publicar.
- **Impacto:** [x] Requerimiento [x] Contrato API [ ] DDL/BD [ ] RBAC [x] Mockup [x] Backend
- **Prioridad:** Media
- **¿Bloquea backend?** No

## B. Refinamiento de BASE DE DATOS

### Resumen de decisiones — Base de datos
| ID | Propuesta (resumen) | Entidad/Tabla | Decisión | ¿Bloquea backend? |
|---|---|---|:--:|:--:|
| P-BD-01 | Guardar el motivo de cancelación de una sesión | scheduling.class_session | ✅ | Sí |

### Detalle
#### [P-BD-01] Motivo de cancelación de sesión
- **Fuente:** Kevin C. · nota en Drive
- **Propuesta:** Poder registrar por qué se canceló una clase.
- **Entidad/Tabla afectada:** `scheduling.class_session` · nuevo campo `cancellation_reason`
- **Decisión:** ✅ ACEPTAR
- **Justificación técnica:** Hoy `class_session` tiene estado pero no motivo; el reporte de asistencia
  lo necesita. Campo `VARCHAR(200)` nullable (solo aplica si el estado es CANCELLED).
- **¿Requiere changeset nuevo?** Sí (id sugerido: `refinamiento-scheduling-970`)
- **Impacto:** [ ] Requerimiento [x] Contrato API [x] DDL/BD [ ] RBAC [x] Mockup [x] Backend
- **Prioridad:** Alta
- **¿Bloquea backend?** Sí

## C. Refinamiento del MOCKUP / UX

### Resumen de decisiones — Mockup
| ID | Propuesta (resumen) | Pantalla/Ruta | Decisión | ¿Bloquea backend? |
|---|---|---|:--:|:--:|
| P-MOCK-01 | Mostrar el cupo disponible en la lista de fichas | #/fichas | ❌ | No |

### Detalle
#### [P-MOCK-01] Cupo disponible en lista de fichas
- **Fuente:** Santiago P. · captura en Drive
- **Pantalla / ruta:** Coordinador › Fichas · `#/fichas`
- **Propuesta:** Añadir columna "cupos disponibles" en la tabla de fichas.
- **Decisión:** ❌ RECHAZAR
- **Justificación técnica:** El dato "cupos disponibles" no vive en academic-management (que solo
  guarda `max_capacity`); el conteo real de matriculados está en actors (`learner_ficha_enrollment`).
  Mostrarlo aquí obligaría a un join cross-servicio en una lista → viola el patrón BFF por composición.
  Va mejor en el **detalle** de ficha, no en la lista.
- **Depende de:** —
- **Impacto:** [ ] Requerimiento [ ] Contrato API [ ] DDL/BD [ ] RBAC [x] Mockup [ ] Backend
- **Prioridad:** Baja
- **¿Bloquea backend?** No

## D. Hallazgos NUEVOS

#### [H-01] Falta estado "reprogramada" en sesión — capa: BD
- **Descripción:** Una sesión puede moverse de fecha sin cancelarse; hoy no hay estado para eso.
- **Por qué importa (técnico):** El CHECK de `class_session.status` no contempla `RESCHEDULED`; sin él
  se pierde trazabilidad y el reporte confunde canceladas con movidas.
- **Propuesta de solución:** Agregar valor `RESCHEDULED` al CHECK vía changeset nuevo.
- **Impacto:** [ ] Requerimiento [x] Contrato API [x] DDL/BD [ ] RBAC [x] Mockup [x] Backend
- **Prioridad:** Media
- **¿Bloquea backend?** No

## E. BLOQUEANTES para iniciar BACKEND

| ID (ref) | Qué falta cerrar | Capa | Decisión |
|---|---|---|:--:|
| P-BD-01 | Campo `cancellation_reason` en `class_session` (changeset + contrato) | BD | ✅ |

## F. Cierre

- **Total de propuestas revisadas:** 3  (✅ 1 · 🔧 1 · ❌ 1 · 💬 0)
- **Hallazgos nuevos:** 1
- **Bloqueantes de backend identificados:** 1
- **Comentario general del revisor:** La mayoría del feedback es de UI; el dato realmente faltante es
  el motivo de cancelación. Cerrar eso antes de backend.

> **Firma:** Nombre Apellido · 3145555 · 2026-08-07
