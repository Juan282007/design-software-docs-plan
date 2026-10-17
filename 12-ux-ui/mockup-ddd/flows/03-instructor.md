<!-- RESUMEN-EJECUTIVO
agente: Jesús Ariel González Bonilla (PM + Arquitecto)
capacidad: DDD de flujo UX (Instructor) + prompts Stitch
fase: diseño (UX/UI)
estado: PRELIMINAR (v0) — maduración a cobertura completa (MFE + pantallas/modales faltantes)
dependencias_entrada: 07-api/contracts/openapi/scheduling.yaml, actors.yaml, monitoring.yaml; 09-microservices/services/05-scheduling-service, 06-actors-service, 08-monitoring-service/data-model.md; 12-ux-ui/design-system.md, navigation-map.md, mockup-ddd/micro-frontends.md; 04-requirements/user-stories.md
consumidores_siguientes: Google Stitch; validación de diseño; construcción de scheduling-mfe/actors-mfe/monitoring-mfe
tldr: 6 pantallas/modales del rol Instructor (Mi horario, Detalle de sesión, Mi disponibilidad, Crear excepción de disponibilidad, Seguimiento de ficha, Registrar seguimiento), derivadas de tres contratos reales (scheduling/actors/monitoring) y sus data-model, cada una con su MFE dueño (micro-frontends.md), con prompts listos para Stitch.
decisiones_clave: la "ficha" de cada sesión se resuelve cross-service (schedule_id → Schedule.ficha_id, sin FK física); "disponibilidad" en v0 solo modela excepciones/bloqueos puntuales (no hay endpoint de franjas positivas); "Seguimiento" asume que el scope OWN del JWT acota `listFichaTrackings` al instructor; el contrato de `InstructorAvailabilityException` no expone `PUT`/`PATCH` — "editar" una excepción es en realidad "eliminar y volver a crear" (gap documentado en Pantalla 4)
halts_registrados: `listFichaTrackings` exige el feature `MON_DASHBOARD_FULL`; no hay en el contrato un feature explícito "mis fichas" para instructor — queda como gap a confirmar con backend (ver Pantalla 5)
-->

> **ESTADO: PRELIMINAR (v0 · mockup-first · design-driven discovery).** Instrumento de
> descubrimiento para Google Stitch, no un diseño final. Cada pantalla es trazable a
> **HU (si existe) + endpoint(s) + tabla(s)** de los contratos reales; donde no existe
> `HU-##` para Instructor se marca `HU: pendiente`. Ver
> [README.md](../README.md) para el framework completo y el brief global de estilo.

# Flujo — Instructor

**Rol:** `INSTRUCTOR` · **Navegación:** [navigation-map.md § Instructor](../../navigation-map.md#instructor)

Este flujo consume **tres servicios** distintos, cada uno con su propio contrato, modelo de
datos y **micro-frontend dueño** (ver [`micro-frontends.md`](../micro-frontends.md)):

| Pantalla/modal | Servicio | MFE | Contrato | Modelo |
|---|---|---|---|---|
| Mi horario (semana) | scheduling-service | `scheduling-mfe` | [`../../../07-api/contracts/openapi/scheduling.yaml`](../../../07-api/contracts/openapi/scheduling.yaml) | [`../../../09-microservices/services/05-scheduling-service/data-model.md`](../../../09-microservices/services/05-scheduling-service/data-model.md) |
| Detalle de sesión | scheduling-service | `scheduling-mfe` | [`../../../07-api/contracts/openapi/scheduling.yaml`](../../../07-api/contracts/openapi/scheduling.yaml) | [`../../../09-microservices/services/05-scheduling-service/data-model.md`](../../../09-microservices/services/05-scheduling-service/data-model.md) |
| Mi disponibilidad | actors-service | `actors-mfe` | [`../../../07-api/contracts/openapi/actors.yaml`](../../../07-api/contracts/openapi/actors.yaml) | [`../../../09-microservices/services/06-actors-service/data-model.md`](../../../09-microservices/services/06-actors-service/data-model.md) |
| Crear excepción de disponibilidad | actors-service | `actors-mfe` | [`../../../07-api/contracts/openapi/actors.yaml`](../../../07-api/contracts/openapi/actors.yaml) | [`../../../09-microservices/services/06-actors-service/data-model.md`](../../../09-microservices/services/06-actors-service/data-model.md) |
| Seguimiento de ficha | monitoring-service | `monitoring-mfe` | [`../../../07-api/contracts/openapi/monitoring.yaml`](../../../07-api/contracts/openapi/monitoring.yaml) | [`../../../09-microservices/services/08-monitoring-service/data-model.md`](../../../09-microservices/services/08-monitoring-service/data-model.md) |
| Registrar seguimiento | monitoring-service | `monitoring-mfe` | [`../../../07-api/contracts/openapi/monitoring.yaml`](../../../07-api/contracts/openapi/monitoring.yaml) | [`../../../09-microservices/services/08-monitoring-service/data-model.md`](../../../09-microservices/services/08-monitoring-service/data-model.md) |

---

## Pantalla 1 — Mi horario (semana)

- **Ruta:** `/instructor/mi-horario` · **Rol:** Instructor · **MFE:** `scheduling-mfe`
  (ver [micro-frontends.md](../micro-frontends.md#mapa-dominio--micro-frontend--pantallas-dueño--microservicio))
  · **HU:** HU-15 (*"Como instructor
  quiero consultar mi horario vigente por semana para conocer mi carga con anticipación"*)
- **Servicio:** scheduling-service
- **Endpoint:** `GET /sessions` (`operationId: listSessions`) — filtros `instructor_id` (el
  propio), `from`/`to` (rango `session_date` de la semana visible), `status` (`ACTIVE` por
  defecto). Detalle de una sesión: `GET /sessions/{id}` (`operationId: getSession`).
- **Alcance (scope del JWT):** el `scope_type = OWN_SCHEDULE` asociado al rol Instructor
  acota el resultado a sesiones cuyo `schedule` padre está `PUBLISHED` (RN-SCH-01, HU-15:
  *"veo solo sesiones de horarios PUBLISHED"*) — el filtro lo aplica el servicio, no la UI.
- **Campos reales (`Session`, scheduling.yaml):** `id`, `schedule_id`, `competency_id`,
  `environment_id`, `instructor_id`, `time_slot_id`, `session_date`, `day_of_week`
  (1=lunes…7=domingo), `start_time`, `end_time`, `status` (`ACTIVE`/`CANCELLED`), `notes`,
  `updated_at`.
- **Resolución cross-service (sin FK física, por diseño):** `Session` no trae `ficha_id`
  directo — se resuelve vía `schedule_id → GET /schedules/{id}` (`Schedule.ficha_id`, mismo
  contrato). El nombre de la competencia (`competency_id`) y el nombre/ubicación del
  ambiente (`environment_id`) se resuelven contra academic-management-service y
  training-environment-service respectivamente. **Supuesto v0:** una capa de agregación
  (BFF) entrega estos nombres ya resueltos a la tarjeta; el endpoint `listSessions` por sí
  solo solo expone los UUID.

**Propósito:** que el instructor conozca su carga semanal de sesiones publicadas con
anticipación (meta ≥ 48 h), sin poder editarla.

**Layout:** selector de semana (flechas anterior/siguiente + rango de fechas visible);
rejilla semanal (7 columnas día × filas de franja horaria); tarjeta de sesión dentro de la
celda correspondiente con: franja (`start_time`–`end_time`), **ficha** (resuelta),
**competencia**, **ambiente**, badge de **estado** (`ACTIVE` normal, `CANCELLED` tachado +
icono, nunca solo color). Clic en una tarjeta abre panel lateral de detalle (`notes`,
ambiente completo, competencia completa).

**Acciones:**
| Control | Endpoint | Resultado |
|---|---|---|
| ◀ Semana anterior / Semana siguiente ▶ | `GET /sessions?instructor_id=&from=&to=` | recarga la rejilla con la semana elegida |
| Clic en tarjeta de sesión | `GET /sessions/{id}` | abre panel de detalle |

**Estados:** *loading* (skeleton de rejilla) · *empty* ("No tienes sesiones publicadas esta
semana") · *error* (banner + reintentar) · sesión `CANCELLED` con tratamiento visual
distinto (tachado + icono, no solo opacidad/color).

```text
PROMPT STITCH
Pantalla "Mi horario" para el rol Instructor de la plataforma SENA — Gestión de Horarios, en
español, responsive. Barra superior con selector de semana: flechas "◀" y "▶" alrededor de un
texto "10 – 16 de agosto de 2026". Debajo, una rejilla semanal (calendario) con 7 columnas
(lunes a domingo) y filas por franja horaria. Dentro de las celdas, tarjetas de sesión de
clase mostrando: horario ("07:00 – 09:00"), ficha ("Ficha 2758543"), competencia
("Programación de software"), ambiente ("Ambiente 204 - Sistemas") y una insignia de estado;
una tarjeta de ejemplo con estado "Cancelada" debe verse tachada y con un icono de alerta
(no solo un color distinto). Al hacer clic en una tarjeta se abre un panel lateral derecho
con el detalle completo de la sesión y sus notas. Incluir el estado vacío: mensaje centrado
"No tienes sesiones publicadas esta semana" con un icono de calendario. Estilo institucional
sobrio, verde SENA (placeholder), alto contraste WCAG AA, objetivos táctiles ≥44px. Mostrar
también skeleton de carga de la rejilla. Variante móvil: la rejilla colapsa a una lista de
tarjetas agrupadas por día.

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

---

## Pantalla 2 — Detalle de sesión

- **Ruta:** panel lateral derecho sobre `/instructor/mi-horario` (no es una ruta propia —
  se abre desde la tarjeta de sesión de la Pantalla 1; deep-link opcional
  `/instructor/mi-horario?sesion={id}`) · **Rol:** Instructor · **MFE:** `scheduling-mfe`
  · **HU:** HU-15 (misma HU de Pantalla 1 — el detalle es parte del mismo objetivo de
  "conocer mi carga con anticipación")
- **Servicio:** scheduling-service
- **Endpoint:** `GET /sessions/{id}` (`operationId: getSession`) — lectura de **una**
  `class_session`. Sin acciones de escritura: el instructor no puede editar/cancelar sus
  propias sesiones (RN-SCH-01, `scope_type = OWN_SCHEDULE` es de solo lectura).
- **Campos reales (`Session`, scheduling.yaml):** `id`, `schedule_id`, `competency_id`,
  `environment_id`, `instructor_id`, `time_slot_id`, `session_date`, `day_of_week`,
  `start_time`, `end_time`, `status` (`ACTIVE`/`CANCELLED`), `notes`, `updated_at`.
- **Resolución cross-service (mismo supuesto que Pantalla 1):** `ficha` vía
  `schedule_id → GET /schedules/{id}` (`Schedule.ficha_id`); nombre de **competencia**
  (`competency_id`) vía academic-management-service; nombre/ubicación de **ambiente**
  (`environment_id`) vía training-environment-service. **Supuesto v0:** una capa de
  agregación (BFF) entrega estos nombres ya resueltos al panel; `getSession` por sí solo
  solo expone los UUID.

**Propósito:** que el instructor vea el detalle completo de una sesión puntual de su
horario (ficha, competencia, ambiente, franja y notas) sin salir de la vista semanal.

**Layout:** panel lateral derecho (~420px desktop, pantalla completa en móvil) con: botón
"Cerrar" (X) arriba a la derecha; encabezado con franja (`start_time`–`end_time`),
`session_date` (fecha larga) y badge de **estado** (`ACTIVE` normal, `CANCELLED` tachado +
icono, nunca solo color); sección **Ficha** (número resuelto); sección **Competencia**
(nombre completo resuelto); sección **Ambiente** (nombre + ubicación resuelta); sección
**Notas** (`notes`, o "Sin notas para esta sesión" si es `null`).

**Acciones:**
| Control | Endpoint | Resultado |
|---|---|---|
| Clic en tarjeta de sesión (Pantalla 1) | `GET /sessions/{id}` | abre el panel con el detalle |
| Cerrar (X) / clic fuera del panel | — | cierra el panel, vuelve a Mi horario |

**Estados:** *loading* (skeleton del panel: franja + 3 bloques de sección) · *error*
(banner + reintentar dentro del panel, no bloquea la rejilla de fondo) · sesión
`CANCELLED` con el mismo tratamiento visual que en Pantalla 1 (tachado + icono de alerta,
nunca solo color). No aplica estado *empty* (el panel solo se abre sobre una sesión
existente).

```text
PROMPT STITCH
Panel lateral "Detalle de sesión" para el rol Instructor de la plataforma SENA — Gestión de
Horarios, en español, responsive, que se abre desde una tarjeta de "Mi horario". Panel
deslizante desde la derecha (~420px en desktop, pantalla completa en móvil) con botón cerrar
"×" arriba a la derecha. Encabezado: horario "07:00 – 09:00", fecha "Lunes 10 de agosto de
2026" y una insignia de estado ("Activa", éxito). Debajo, tres bloques de sección con
etiqueta + valor: "Ficha" ("Ficha 2758543 - Análisis y Desarrollo de Software"),
"Competencia" ("Programación de software orientado a objetos") y "Ambiente" ("Ambiente 204 -
Bloque C, Sistemas"). Al final, sección "Notas" con un texto breve de ejemplo ("Traer
computador portátil, se hace laboratorio práctico") o el texto "Sin notas para esta sesión"
si no hay notas. Mostrar también una variante con estado "Cancelada": el encabezado tachado
con un icono de alerta junto a la insignia (no solo un color distinto). Estilo institucional
sobrio, verde SENA (placeholder), alto contraste WCAG AA, objetivos táctiles ≥44px, navegable
por teclado (foco atrapado dentro del panel, Escape cierra). Mostrar también el skeleton de
carga del panel (franja + 3 bloques).

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

---

## Pantalla 3 — Mi disponibilidad

- **Ruta:** `/instructor/mi-disponibilidad` · **Rol:** Instructor · **MFE:** `actors-mfe`
  · **HU:** HU-14 (*"Como
  instructor quiero declarar mis franjas disponibles y mis excepciones para que no me
  asignen clases cuando no puedo"*)
- **Servicio:** actors-service
- **Endpoints:**
  - `GET /instructors/{id}/availability-exceptions` (`operationId:
    listInstructorAvailabilityExceptions`) — parámetros `from`/`to` (rango sobre
    `start_datetime`/`end_datetime`).
  - `POST /instructors/{id}/availability-exceptions` (`operationId:
    createInstructorAvailabilityException`) — crea el bloqueo; emite
    `actors.instructor.availability_changed`.
  - `DELETE /instructors/{id}/availability-exceptions/{exception_id}` (`operationId:
    removeInstructorAvailabilityException`) — elimina un bloqueo registrado por error;
    emite el mismo evento.
  - `{id}` = el `instructor_id` propio del usuario autenticado. **Supuesto:** se resuelve
    desde el contexto de sesión (mapeo `user_id` de iam-service → `instructor.id` de
    actors-service); el contrato de auth (`GET /auth/me`) no expone `instructor_id`
    directamente — es un dato a confirmar con backend antes de construir.
- **Campos reales (`InstructorAvailabilityException`, actors.yaml):** `id`, `instructor_id`,
  `exception_type` (enum `SICK_LEAVE` / `VACATION` / `COMMISSION` / `PERSONAL_LEAVE` /
  `TRAINING` / `OTHER`), `start_datetime`, `end_datetime`, `description` (opcional),
  `approved_by` (uuid de usuario iam, nullable — quién aprobó), `created_at`.
- **Nota de alcance v0 (hallazgo del contrato):** el modelo solo cubre **excepciones**
  (bloqueos puntuales); no existe en el contrato actual un endpoint para declarar **franjas
  recurrentes disponibles de forma positiva**. El instructor se asume disponible por
  defecto en toda franja salvo que registre una excepción — distinto de lo que sugiere el
  texto de HU-14 ("declarar mis franjas disponibles"), que queda parcialmente cubierto.
- **Hallazgo de contrato — paginación (gap a confirmar con backend):**
  `listInstructorAvailabilityExceptions` hoy solo expone `from`/`to` y responde un arreglo
  plano (sin `page`/`page_size` ni sobre `pagination`). Esta pantalla es una **lista** (regla
  README anti-patrón #3), así que el mockup **sí** incluye paginación real; se modela con los
  parámetros estándar `page`/`page_size` del resto del contrato (mismo patrón que
  `listFichaTrackings`/`listTrackingSessions`) a la espera de que el endpoint los adopte.

**Propósito:** que el instructor declare bloqueos de disponibilidad para que el motor de
conflictos detecte `INSTRUCTOR_UNAVAILABLE` (RN-SCH-03) y el coordinador no lo asigne en
esas franjas al construir un horario.

**Layout:** lista cronológica de excepciones (vigentes y futuras primero), cada fila con
tipo (icono + texto), rango de fechas/horas, descripción y quién la aprobó (si aplica);
botón primario **+ Nueva excepción** abre el modal de la Pantalla 4 (Crear excepción de
disponibilidad); cada fila tiene acción **Eliminar**. Pie de lista con **paginación REAL
visible**: barra ‹ anterior · 1 2 3 … › siguiente, selector de tamaño de página
(`page_size`: 10/20/50) y texto "Mostrando X–Y de N", alineados a `page`/`page_size` (ver
hallazgo de contrato arriba).

**Acciones:**
| Control | Endpoint | Resultado |
|---|---|---|
| + Nueva excepción | — (abre Pantalla 4) | abre el modal de creación; ver Pantalla 4 para el detalle de `POST` |
| Eliminar (por fila) | `DELETE /instructors/{id}/availability-exceptions/{exception_id}` | quita el bloqueo; refresca la lista |

**Estados:** *loading* (skeleton de lista) · *empty* ("Sin excepciones registradas — estás
disponible en tu franja estándar") · *error* (banner + reintentar) · validación en línea
(fecha/hora fin posterior a inicio) · *success* (toast "Excepción registrada" /
"Excepción eliminada").

```text
PROMPT STITCH
Pantalla "Mi disponibilidad" para el rol Instructor de la plataforma SENA — Gestión de
Horarios, en español, responsive. Encabezado con título "Mi disponibilidad" y botón primario
"+ Nueva excepción" a la derecha. Debajo, una lista de tarjetas/filas cronológicas, cada una
mostrando: icono según tipo, etiqueta de tipo ("Incapacidad médica", "Vacaciones",
"Comisión", "Permiso personal", "Capacitación", "Otro"), rango de fecha y hora
("12 – 14 de agosto, 2026, 08:00 – 17:00"), una descripción breve y un botón "Eliminar" con
icono de papelera. Al hacer clic en "+ Nueva excepción" se abre un modal con: un selector
desplegable "Tipo de excepción", dos campos de fecha/hora ("Inicio" y "Fin"), un campo de
texto "Descripción (opcional)" y un botón primario "Guardar". Mostrar el estado vacío:
mensaje centrado "Sin excepciones registradas — estás disponible en tu franja estándar" con
icono de calendario con check. Estilo institucional sobrio, alto contraste WCAG AA,
objetivos táctiles ≥44px, navegable por teclado. Incluir un toast de confirmación
"Excepción registrada". Al final de la lista, incluir una barra de paginación real: flechas
"‹ anterior" y "siguiente ›" alrededor de números de página "1 2 3", un selector desplegable
de tamaño de página ("10 / 20 / 50 por página") y el texto "Mostrando 1–10 de 34".

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

---

## Pantalla 4 — Modal/Form: Crear excepción de disponibilidad

- **Ruta:** modal sobre `/instructor/mi-disponibilidad` (no es una ruta propia) · **Rol:**
  Instructor · **MFE:** `actors-mfe` · **HU:** HU-14 (misma HU de Pantalla 3)
- **Servicio:** actors-service
- **Endpoint:** `POST /instructors/{id}/availability-exceptions` (`operationId:
  createInstructorAvailabilityException`) — `{id}` = `instructor_id` propio, resuelto desde
  el contexto de sesión (mismo supuesto que Pantalla 3). Emite
  `actors.instructor.availability_changed` al guardar.
- **Campos del formulario (`InstructorAvailabilityExceptionCreate`, actors.yaml, campos
  requeridos en negrita):** **`exception_type`** (enum `SICK_LEAVE` / `VACATION` /
  `COMMISSION` / `PERSONAL_LEAVE` / `TRAINING` / `OTHER`), **`start_datetime`**,
  **`end_datetime`**, `description` (opcional, string). `approved_by` existe en el schema
  pero **no** se expone en este formulario: es un `uuid` de usuario iam que se asigna en
  flujo de aprobación (fuera de alcance de esta pantalla; el instructor solo declara, no
  aprueba).
- **Hallazgo de contrato — sin "editar" (gap documentado, no se inventa):** el contrato
  expone `POST` (crear) y `DELETE` (eliminar) para
  `/instructors/{id}/availability-exceptions{,/{exception_id}}`, pero **no** un `PUT`/
  `PATCH`. La "E" de "Crear/editar excepción" pedida en la maduración del flujo **no tiene
  endpoint real de edición**: el patrón v0 es **eliminar y volver a crear** (el botón
  "Eliminar" de la Pantalla 3 + este mismo modal de creación). Este modal, por tanto, solo
  cubre **creación**; queda como gap a confirmar con backend si se requiere edición in-place.

**Propósito:** que el instructor registre un bloqueo puntual de disponibilidad (incapacidad,
vacaciones, comisión, permiso, capacitación u otro) para que el motor de conflictos lo
excluya de asignaciones futuras en esa franja (RN-SCH-03).

**Layout:** modal centrado (o panel en móvil) con título "Nueva excepción de disponibilidad"
y botón cerrar (X); formulario de una columna: selector desplegable **Tipo de excepción**
(las 6 opciones, con icono por tipo), dos campos **Inicio** y **Fin** (fecha + hora cada
uno), campo de texto **Descripción** (opcional, placeholder "Ej. Cita médica especialista"),
pie con botón secundario "Cancelar" y botón primario **Guardar**.

**Acciones:**
| Control | Endpoint | Resultado |
|---|---|---|
| Guardar | `POST /instructors/{id}/availability-exceptions` | crea el bloqueo; cierra el modal; refresca la lista de Pantalla 3; toast "Excepción registrada" |
| Cancelar / cerrar (X) | — | descarta el formulario sin guardar, vuelve a Pantalla 3 |

**Estados:** *validación en línea* (`exception_type` requerido; `end_datetime` posterior a
`start_datetime`; ambos campos de fecha/hora requeridos) · *guardando* (botón "Guardar" con
spinner, deshabilitado) · *error* (mensaje en línea dentro del modal, ej. "No se pudo
registrar la excepción — intenta de nuevo"; el modal permanece abierto para no perder lo
digitado) · *success* (cierra el modal + toast "Excepción registrada" en Pantalla 3).

```text
PROMPT STITCH
Modal "Nueva excepción de disponibilidad" para el rol Instructor de la plataforma SENA —
Gestión de Horarios, en español, responsive, que se abre desde el botón "+ Nueva excepción"
de "Mi disponibilidad". Modal centrado (~480px) con título "Nueva excepción de
disponibilidad" y botón cerrar "×" arriba a la derecha. Formulario de una columna: selector
desplegable "Tipo de excepción" con las opciones "Incapacidad médica", "Vacaciones",
"Comisión", "Permiso personal", "Capacitación", "Otro"; dos campos de fecha y hora "Inicio"
(ej. "12 ago 2026, 08:00") y "Fin" (ej. "14 ago 2026, 17:00"); un campo de texto
"Descripción (opcional)" con placeholder "Ej. Cita médica especialista"; pie del modal con
botón secundario "Cancelar" y botón primario "Guardar". Mostrar validación en línea: mensaje
de error bajo "Fin" si la fecha es anterior o igual a "Inicio" ("La fecha de fin debe ser
posterior al inicio"). Mostrar también el estado "Guardando…" (botón primario con spinner,
deshabilitado) y un estado de error dentro del modal ("No se pudo registrar la excepción —
intenta de nuevo") sin cerrar el modal. Estilo institucional sobrio, alto contraste WCAG AA,
objetivos táctiles ≥44px, navegable por teclado (foco atrapado en el modal, Escape cierra).

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

---

## Pantalla 5 — Seguimiento de ficha

- **Ruta:** `/instructor/seguimiento` · **Rol:** Instructor · **MFE:** `monitoring-mfe`
  · **HU:** pendiente (relacionado
  con RF-MON-02: *"calcular KPIs por ficha... registrando cada medición como un nuevo
  registro append-only"*; no hay una `HU-##` de Instructor dedicada a este registro en
  `user-stories.md`)
- **Servicio:** monitoring-service
- **Endpoints:**
  - Selector de ficha: `GET /ficha-trackings` (`operationId: listFichaTrackings`), filtro
    `assigned_instructor_id`. **⚠ Hallazgo de contrato (halt registrado):** este endpoint
    exige el feature `MON_DASHBOARD_FULL`, que por nombre sugiere tablero **completo**
    (coordinación), sin un feature explícito "mis fichas" para Instructor. **Supuesto v0:**
    se asume que el `scope_type` `OWN` del JWT del instructor acota igual el resultado a sus
    fichas asignadas (mismo patrón que `OWN_SCHEDULE` en scheduling.yaml); si el backend no
    implementa esa acotación, este selector queda bloqueado y debe resolverse antes de
    construir.
  - Resumen de la ficha: `GET /ficha-trackings/{id}` (`operationId: getFichaTracking`) —
    requiere `MON_DASHBOARD_OWN` (coherente con acceso de Instructor a su propia ficha).
  - Histórico: `GET /ficha-trackings/{id}/tracking-sessions` (`operationId:
    listTrackingSessions`), filtros `session_type`, `from`/`to`, paginado con `page`/
    `page_size`/`sort` (respuesta `{ data, pagination }`).
  - Registrar: `POST /ficha-trackings/{id}/tracking-sessions` (`operationId:
    createTrackingSession`) — *"Normativa SENA: seguimiento mínimo mensual para fichas en
    Ejecución (Acuerdo 00003/2012). `attendance_percentage` es calculada por el servidor."*
  - Detalle de fila: `GET /ficha-trackings/{id}/tracking-sessions/{session_id}`
    (`operationId: getTrackingSession`).
- **Campos reales (`FichaTracking`):** `ficha_id`, `assigned_instructor_id`,
  `overall_status_id` (→ `kpi_status`/`risk_level`), `active_alert_count`,
  `last_tracking_date`, `next_tracking_date`.
- **Campos reales (`TrackingSession` / `TrackingSessionCreate`):** `id`,
  `ficha_tracking_id`, `instructor_id`, `session_date`, `session_type` (enum `ACADEMIC` /
  `WELLNESS` / `PROJECT` / `PRODUCTIVE_STAGE`), `attendance_count`, `total_learners`,
  `attendance_percentage` (**readOnly**, columna generada = `attendance_count /
  total_learners * 100`), `curriculum_progress_percentage`, `observations`,
  `requires_follow_up` (boolean — *"si true, debe generarse una alerta asociada"*),
  `created_at`.

**Propósito:** que el instructor registre sesiones periódicas de seguimiento (asistencia,
avance curricular) de su ficha, insumo append-only del cálculo de KPIs y alertas de riesgo.

**Layout:** selector de ficha (dropdown, fichas asignadas al instructor); encabezado-resumen
con `overall_status` (badge de riesgo), `last_tracking_date` / `next_tracking_date` y
`active_alert_count`; tabla histórica de `tracking_session` (más reciente primero) con
columnas: fecha, tipo, asistencia (`attendance_count`/`total_learners` → `%`), avance
curricular (%), "requiere seguimiento" (badge); pie de tabla con **paginación REAL visible**:
barra ‹ anterior · 1 2 3 … › siguiente, selector de tamaño de página (`page_size`: 10/20/50)
y texto "Mostrando X–Y de N", alineados a `page`/`page_size` de `listTrackingSessions`; botón
primario **+ Registrar seguimiento** abre el formulario de la Pantalla 6 (Registrar
seguimiento).

**Acciones:**
| Control | Endpoint | Resultado |
|---|---|---|
| Seleccionar ficha | `GET /ficha-trackings?assigned_instructor_id=` → `GET /ficha-trackings/{id}` | carga resumen |
| Cargar histórico | `GET /ficha-trackings/{id}/tracking-sessions?page=&page_size=` | llena la tabla (página actual) |
| Cambiar página / tamaño de página | `GET /ficha-trackings/{id}/tracking-sessions?page=&page_size=` | recarga la tabla con la página/tamaño elegidos |
| + Registrar seguimiento | — (abre Pantalla 6) | abre el formulario de creación; ver Pantalla 6 para el detalle de `POST` |
| Clic en fila | `GET /ficha-trackings/{id}/tracking-sessions/{session_id}` | abre panel de detalle |

**Estados:** *loading* (skeleton de resumen + tabla) · *empty* (ficha sin seguimientos aún;
si `next_tracking_date` está vencida, aviso de seguimiento pendiente > 35 días, RF-MON-01) ·
*error* (banner + reintentar) · validación en línea (`attendance_count ≤ total_learners`,
`curriculum_progress_percentage` entre 0–100) · *success* (toast "Seguimiento registrado").

```text
PROMPT STITCH
Pantalla "Seguimiento de ficha" para el rol Instructor de la plataforma SENA — Gestión de
Horarios, en español, responsive. Encabezado con un selector desplegable "Ficha: 2758543 -
Análisis y Desarrollo de Software" y, al lado, un resumen con una insignia de estado de
riesgo ("En riesgo", color ámbar + icono, nunca solo color), texto "Último seguimiento:
10 jul 2026" y "Próximo: 10 ago 2026". Botón primario "+ Registrar seguimiento" a la derecha.
Debajo, una tabla histórica con columnas "Fecha", "Tipo" (Académico/Bienestar/
Proyecto/Etapa productiva), "Asistencia" (ej. "18/20 - 90%"), "Avance curricular" (barra de
progreso, ej. 65%) y "Requiere seguimiento" (badge sí/no). Al hacer clic en "+ Registrar
seguimiento" se abre un formulario modal con: selector "Tipo de sesión", campo "Fecha",
dos campos numéricos "Asistentes" y "Total de aprendices", un campo numérico "Avance
curricular (%)", un textarea "Observaciones" y un checkbox "Requiere seguimiento adicional",
con botón primario "Guardar". Mostrar el estado vacío: "Esta ficha aún no tiene registros de
seguimiento" con un aviso de alerta si el seguimiento está vencido (más de 35 días sin
registrar). Estilo institucional sobrio, alto contraste WCAG AA, objetivos táctiles ≥44px,
navegable por teclado. Al final de la tabla histórica, incluir una barra de paginación real:
flechas "‹ anterior" y "siguiente ›" alrededor de números de página "1 2 3", un selector
desplegable de tamaño de página ("10 / 20 / 50 por página") y el texto "Mostrando 1–10 de 27".

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

---

## Pantalla 6 — Form: Registrar seguimiento

- **Ruta:** modal/formulario sobre `/instructor/seguimiento` (no es una ruta propia) ·
  **Rol:** Instructor · **MFE:** `monitoring-mfe` · **HU:** pendiente (misma HU pendiente de
  Pantalla 5)
- **Servicio:** monitoring-service
- **Endpoint:** `POST /ficha-trackings/{id}/tracking-sessions` (`operationId:
  createTrackingSession`) — `{id}` = `ficha_tracking_id` de la ficha seleccionada en
  Pantalla 5. Requiere el feature `MON_TRACKING_SESSION_CREATE`. *"Normativa SENA:
  seguimiento mínimo mensual para fichas en Ejecución (Acuerdo 00003/2012).
  `attendance_percentage` es calculada por el servidor."* Crea un registro **append-only**
  de `tracking_session` (no editable ni eliminable desde la UI); si `requires_follow_up =
  true`, el `alert-worker` puede generar una alerta asociada.
- **Campos del formulario (`TrackingSessionCreate`, monitoring.yaml, campos requeridos en
  negrita):** `instructor_id` (no se pide en el formulario — se autocompleta con el
  instructor autenticado, mismo patrón de resolución por contexto de sesión que Pantalla 4),
  **`session_date`**, **`session_type`** (enum `ACADEMIC` / `WELLNESS` / `PROJECT` /
  `PRODUCTIVE_STAGE`), `attendance_count` (entero ≥0), `total_learners` (entero ≥0),
  `curriculum_progress_percentage` (0–100), `observations` (texto libre),
  `requires_follow_up` (boolean, default `false`). `attendance_percentage` **no** es un
  campo del formulario — es `readOnly`, columna generada por PostgreSQL
  (`attendance_count / total_learners * 100`) que el servidor calcula al guardar.

**Propósito:** que el instructor registre, de forma periódica, una medición de asistencia y
avance curricular de su ficha — insumo append-only para el cálculo de KPIs y alertas de
riesgo (RF-MON-02).

**Layout:** modal (o panel en móvil) con título "Registrar seguimiento" y botón cerrar (X);
formulario de una columna: selector **Tipo de sesión** (las 4 opciones), campo **Fecha**
(`session_date`), fila con dos campos numéricos **Asistentes** (`attendance_count`) y
**Total de aprendices** (`total_learners`) — con vista previa en línea del `%` calculado
(ej. "18/20 → 90%", solo referencial, el valor real lo calcula el servidor), campo numérico
**Avance curricular (%)** (`curriculum_progress_percentage`), textarea **Observaciones**
(`observations`), checkbox **"Requiere seguimiento adicional"** (`requires_follow_up`); pie
con botón secundario "Cancelar" y botón primario **Guardar**.

**Acciones:**
| Control | Endpoint | Resultado |
|---|---|---|
| Guardar | `POST /ficha-trackings/{id}/tracking-sessions` | crea el registro append-only; cierra el modal; refresca el histórico y el resumen de Pantalla 5; toast "Seguimiento registrado" |
| Cancelar / cerrar (X) | — | descarta el formulario sin guardar, vuelve a Pantalla 5 |

**Estados:** *validación en línea* (`session_date` y `session_type` requeridos;
`attendance_count ≤ total_learners`; `curriculum_progress_percentage` entre 0–100) ·
*guardando* (botón "Guardar" con spinner, deshabilitado) · *error* (mensaje en línea dentro
del modal, ej. "No se pudo registrar el seguimiento — intenta de nuevo"; el modal permanece
abierto para no perder lo digitado) · *success* (cierra el modal + toast "Seguimiento
registrado" en Pantalla 5; si `requires_follow_up = true`, aviso adicional "Se generará una
alerta de seguimiento").

```text
PROMPT STITCH
Modal "Registrar seguimiento" para el rol Instructor de la plataforma SENA — Gestión de
Horarios, en español, responsive, que se abre desde el botón "+ Registrar seguimiento" de
"Seguimiento de ficha". Modal centrado (~520px) con título "Registrar seguimiento" y botón
cerrar "×" arriba a la derecha. Formulario de una columna: selector desplegable "Tipo de
sesión" con las opciones "Académico", "Bienestar", "Proyecto", "Etapa productiva"; campo
"Fecha" (ej. "6 ago 2026"); fila con dos campos numéricos "Asistentes" (ej. "18") y "Total de
aprendices" (ej. "20"), con un texto pequeño debajo mostrando el cálculo referencial
"→ 90% de asistencia"; un campo numérico "Avance curricular (%)" (ej. "65"); un textarea
"Observaciones" con placeholder "Ej. Se avanzó en la unidad 3, dos aprendices en riesgo de
deserción"; un checkbox "Requiere seguimiento adicional"; pie del modal con botón secundario
"Cancelar" y botón primario "Guardar". Mostrar validación en línea: mensaje de error si
"Asistentes" es mayor que "Total de aprendices" ("Los asistentes no pueden superar el total
de aprendices"). Mostrar también el estado "Guardando…" (spinner, botón deshabilitado) y un
estado de error dentro del modal ("No se pudo registrar el seguimiento — intenta de nuevo")
sin cerrar el modal. Estilo institucional sobrio, alto contraste WCAG AA, objetivos táctiles
≥44px, navegable por teclado (foco atrapado en el modal, Escape cierra).

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

---

## Mockups generados
_(pendiente — guardar en `../mockups/03-instructor/` y enlazar aquí: mi-horario.png,
detalle-sesion.png, mi-disponibilidad.png, crear-excepcion-disponibilidad.png,
seguimiento.png, registrar-seguimiento.png)_
