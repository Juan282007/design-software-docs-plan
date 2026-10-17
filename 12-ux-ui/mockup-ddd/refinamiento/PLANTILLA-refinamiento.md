---
tipo: refinamiento-requerimientos
proyecto: Sistema de Gestión de Horarios SENA (code-sena)
version_plantilla: 1.0
revisor:                 # << TU nombre y apellido
ficha:                   # << número de ficha
rol_en_proyecto:         # << ej. Modelador de datos / Analista / Front / Backend
fecha:                   # << AAAA-MM-DD
fuente_feedback: https://drive.google.com/drive/folders/17ek3tHMEv1tKWP7nYwjDp0POE3-iYzJi
mockup_revisado: https://code-sena.github.io/design-software-mockup/
---

# Refinamiento de Requerimientos — Revisión individual

> **Propósito.** Muchas personas usaron el mockup como usuario final y dejaron feedback en el Drive.
> Aquí **tú** (que conoces el core del negocio) evalúas cada propuesta con **criterio técnico**:
> la aceptas, la rechazas, la ajustas o la mandas a discutir. El objetivo es una **visión técnica
> precisa**; luego se **integran todos los `.md`** para cerrar pendientes **antes de iniciar backend**.
>
> **Orden de refinamiento (respétalo):** 1) Requerimientos (docs) → 2) Base de datos → 3) Mockup.
> Cada capa condiciona la siguiente: un requerimiento define el dato, el dato define la pantalla.

## Cómo llenar esta plantilla

1. Revisa el feedback del Drive y el mockup publicado.
2. Por cada **propuesta**, copia un bloque de item y complétalo. Usa IDs consecutivos por sección
   (`P-REQ-01`, `P-BD-01`, `P-MOCK-01`…). Si una propuesta abarca varias capas, regístrala en la capa
   de origen y **enlázala** en las otras.
3. Si detectas algo que **nadie propuso** pero es necesario, agrégalo en la Sección D.
4. Marca explícitamente lo que **bloquea backend** (Sección E se auto-alimenta de esos ítems).
5. Sé preciso en la **justificación técnica**: nombra la entidad/campo/endpoint/pantalla afectada.

## Vocabulario de decisión (usar EXACTO)

| Símbolo | Decisión | Cuándo |
|---|---|---|
| ✅ | **ACEPTAR** | La propuesta es correcta técnica y de negocio; entra tal cual. |
| 🔧 | **ACEPTAR CON AJUSTE** | La idea es válida pero se implementa distinto (escribe tu contrapropuesta). |
| ❌ | **RECHAZAR** | No procede; explica la razón técnica (rompe contrato/DDL/RBAC, fuera de alcance, etc.). |
| 💬 | **DISCUTIR** | Falta info o hay conflicto; requiere consenso del equipo. |

**Impacto (marca todas las que apliquen):** `Requerimiento` · `Contrato API` · `DDL/BD` · `RBAC` · `Mockup` · `Backend`
**Prioridad:** `Alta` · `Media` · `Baja`

---

## A. Refinamiento de REQUERIMIENTOS (repo `design-software-docs`)

> Propuestas sobre historias de usuario, reglas de negocio, funcional, RBAC/roles, alcance.

### Resumen de decisiones — Requerimientos
| ID | Propuesta (resumen) | Decisión | ¿Bloquea backend? |
|---|---|:--:|:--:|
| P-REQ-01 |  |  |  |
| P-REQ-02 |  |  |  |

### Detalle
#### [P-REQ-01] <título corto>
- **Fuente:** <autor> · <archivo/enlace en Drive>
- **Propuesta:** <qué se pide cambiar / agregar / quitar>
- **Decisión:** ✅ | 🔧 | ❌ | 💬
- **Justificación técnica:** <por qué; nombra HU/regla/rol afectado y su implicación>
- **Ajuste propuesto (si 🔧):** <tu contrapropuesta>
- **Impacto:** [ ] Requerimiento [ ] Contrato API [ ] DDL/BD [ ] RBAC [ ] Mockup [ ] Backend
- **Prioridad:** Alta | Media | Baja
- **¿Bloquea backend?** Sí | No

<!-- copia este bloque por cada propuesta: P-REQ-02, P-REQ-03, ... -->

---

## B. Refinamiento de BASE DE DATOS (repos `design-software-*-db`)

> Propuestas sobre el modelo de datos, tablas/columnas, tipos, relaciones (FKs), catálogos/seeds,
> estados. Recuerda: **cambio de BD desplegada = changeset NUEVO** (nunca editar lo aplicado).

### Resumen de decisiones — Base de datos
| ID | Propuesta (resumen) | Entidad/Tabla | Decisión | ¿Bloquea backend? |
|---|---|---|:--:|:--:|
| P-BD-01 |  |  |  |  |

### Detalle
#### [P-BD-01] <título corto>
- **Fuente:** <autor> · <enlace Drive>
- **Propuesta:** <qué cambia en el modelo: entidad, campo, tipo, relación, catálogo…>
- **Entidad/Tabla afectada:** <schema.tabla · campo>
- **Decisión:** ✅ | 🔧 | ❌ | 💬
- **Justificación técnica:** <por qué; congruencia con contrato/mockup; normalización; integridad>
- **Ajuste propuesto (si 🔧):** <p. ej. "usar status_id FK en vez de enum">
- **¿Requiere changeset nuevo?** Sí (id sugerido: `refinamiento-<dominio>-NNN`) | No
- **Impacto:** [ ] Requerimiento [ ] Contrato API [ ] DDL/BD [ ] RBAC [ ] Mockup [ ] Backend
- **Prioridad:** Alta | Media | Baja
- **¿Bloquea backend?** Sí | No

<!-- copia este bloque por cada propuesta: P-BD-02, ... -->

---

## C. Refinamiento del MOCKUP / UX

> Propuestas sobre pantallas, flujos, campos visibles, navegación, estados, textos (copy), parametrización.
> Referencia: mockup publicado + índice de 53 pantallas (`#/inventory`).

### Resumen de decisiones — Mockup
| ID | Propuesta (resumen) | Pantalla/Ruta | Decisión | ¿Bloquea backend? |
|---|---|---|:--:|:--:|
| P-MOCK-01 |  |  |  |  |

### Detalle
#### [P-MOCK-01] <título corto>
- **Fuente:** <autor> · <enlace Drive>
- **Pantalla / ruta:** <ej. Coordinador › Crear horario · #/horarios/nuevo>
- **Propuesta:** <qué cambia en la pantalla/flujo>
- **Decisión:** ✅ | 🔧 | ❌ | 💬
- **Justificación técnica:** <por qué; ¿se sostiene con el dato del DDL y el contrato? ¿RBAC lo permite?>
- **Ajuste propuesto (si 🔧):** <la variante>
- **Depende de:** <P-REQ-xx / P-BD-xx si el cambio de UI exige antes cambiar requerimiento o dato>
- **Impacto:** [ ] Requerimiento [ ] Contrato API [ ] DDL/BD [ ] RBAC [ ] Mockup [ ] Backend
- **Prioridad:** Alta | Media | Baja
- **¿Bloquea backend?** Sí | No

<!-- copia este bloque por cada propuesta: P-MOCK-02, ... -->

---

## D. Hallazgos NUEVOS (que nadie propuso pero detecté)

> Cosas que faltan o están mal y que como conocedor del core debes levantar. Mismo formato de item,
> usa prefijo `H-` (`H-01`, `H-02`…) e indica a qué capa pertenece (REQ/BD/MOCK).

#### [H-01] <título> — capa: REQ | BD | MOCK
- **Descripción:**
- **Por qué importa (técnico):**
- **Propuesta de solución:**
- **Impacto:** [ ] Requerimiento [ ] Contrato API [ ] DDL/BD [ ] RBAC [ ] Mockup [ ] Backend
- **Prioridad:** Alta | Media | Baja
- **¿Bloquea backend?** Sí | No

---

## E. BLOQUEANTES para iniciar BACKEND (consolidado)

> Lista SOLO los ítems que marcaste "¿Bloquea backend? = Sí". Esto es lo que se debe cerrar
> **antes** de escribir la primera línea de backend.

| ID (ref) | Qué falta cerrar | Capa | Decisión |
|---|---|---|:--:|
|  |  |  |  |

---

## F. Cierre

- **Total de propuestas revisadas:** ___  (✅ ___ · 🔧 ___ · ❌ ___ · 💬 ___)
- **Hallazgos nuevos:** ___
- **Bloqueantes de backend identificados:** ___
- **Comentario general del revisor:**

> **Firma:** <nombre> · <ficha> · <fecha>

---

### Guía de consolidación (para quien integra los `.md`)
1. Recolectar todos los `.md` (uno por revisor, en `respuestas/`).
2. Agrupar por ID de propuesta (P-REQ/P-BD/P-MOCK) y contar decisiones → mayoría / conflicto.
3. Los ✅ con consenso pasan a implementación (por capa, en orden REQ→BD→Mockup).
4. Los 💬 y los que tengan decisiones divididas van a reunión de cierre.
5. Consolidar la Sección E de todos → **backlog bloqueante de backend** (debe quedar en cero para arrancar).
