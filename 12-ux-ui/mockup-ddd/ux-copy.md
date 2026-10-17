<!-- RESUMEN-EJECUTIVO
agente: Claude (asistente de documentación UX)
capacidad: Guía de UX copy / microcopy transversal para las 45 pantallas del mockup DDD
fase: diseño (UX/UI)
estado: draft
dependencias_entrada: 12-ux-ui/design-system.md; 12-ux-ui/mockup-ddd/flows/{01-auth,02-coordinador,03-instructor,04-aprendiz,05-administrador,06-backoffice}.md; 07-api/contracts/openapi/{iam,_shared}.yaml; 01-context/glossary.md
consumidores_siguientes: implementación de UI (todas las MFE), Google Stitch (prompts de pantalla), QA de contenido
tldr: Mapa único de mensajes de error por `error.code`, estados vacíos, confirmaciones, toasts y labels de botones para hablar con una sola voz en las 45 pantallas; incluye tabla de estados de negocio del horario (BORRADOR/EN_REVISIÓN/PUBLICADO/CONFLICTO).
decisiones_clave: una sola forma canónica por mensaje/acción (se documentan y resuelven las variantes encontradas en los flujos, ej. "Crear horario" vs "Nuevo horario"); los mensajes de error nunca exponen `message` técnico crudo del backend, solo `trace_id` como "Código de referencia"; todo estado se comunica con icono + texto, nunca solo color (WCAG AA, design-system.md §2)
halts_registrados: varios textos de toast/confirmación no tienen redacción literal en los flujos de origen (endpoint documentado, texto de UI no) — se marcan **[propuesto]** y quedan a validar en la fase de diseño de la app
-->

# UX Copy — Horarios SENA

> **ESTADO: PRELIMINAR (v0)**

Guía de microcopy en español para las 45 pantallas del mockup DDD (`12-ux-ui/mockup-ddd/flows/*.md`).
Consolida en un solo lugar los mensajes de error, estados vacíos, confirmaciones, toasts, labels de
botones y etiquetas de estado de negocio, para que ningún equipo/MFE redacte una variante distinta del
mismo mensaje.

**Tono:** institucional, claro, en español con el vocabulario del dominio SENA (ficha, ambiente,
competencia, RAP — ver [glossary.md](../../01-context/glossary.md)); nunca acusatorio; el código
(`error.code`, nombres de campo) va en inglés, el texto que ve el usuario siempre en español
([design-system.md §1.5](../design-system.md)).

**Regla de oro:** cada mensaje o acción tiene **una sola redacción canónica**. Donde los flujos de
origen usan variantes distintas para la misma acción, esta guía elige una y señala el reemplazo.

**Convención de esta tabla:** el texto entre comillas es literal, tomado de los flujos de origen
(`flows/*.md`) o de los contratos OpenAPI. Los textos marcados **[propuesto]** no tienen redacción
literal en el origen — el endpoint/acción existe, pero el copy exacto se define aquí por primera vez,
siguiendo el mismo tono de los textos ya existentes.

---

## 1. Mensajes de error por código

Basado en el envelope común `Error { error: { code, message, details[], trace_id } }`
([`_shared.yaml#/components/schemas/Error`](../../07-api/contracts/openapi/_shared.yaml)). La UI
**nunca** muestra `error.message` del backend tal cual si es técnico; traduce a estos textos. Cuando
el backend ya envía un `message` en español legible (ej. el ejemplo del propio contrato,
`"El instructor ya tiene una sesión en ese bloque horario."`), ese texto es aceptable siempre que
coincida con el patrón de tono de esta tabla.

| `error.code` | HTTP | Mensaje de UI (canónico) | Dónde se usa | Notas |
|---|---|---|---|---|
| `INVALID_CREDENTIALS` | 401 | "Correo o contraseña incorrectos." | Login (banner bajo el formulario) | Nunca precisar cuál de los dos campos falló (seguridad) — `flows/01-auth.md` Pantalla 1 |
| `ACCOUNT_LOCKED` | 401 | "Tu cuenta está bloqueada temporalmente por demasiados intentos fallidos. Intenta de nuevo más tarde." | Login | Mismo banner que `INVALID_CREDENTIALS`, ícono + texto distinto — `flows/01-auth.md` Pantalla 1 |
| `ACCOUNT_INACTIVE` | 401 | "Tu cuenta está inactiva. Contacta al administrador de tu centro." | Login | Cuenta desactivada por un `ADMIN_STAFF`/`SYSTEM_ADMIN` — `flows/01-auth.md` Pantalla 1 |
| `TOKEN_EXPIRED` (sesión) | 401 (en `POST /auth/refresh`) | "Tu sesión expiró. Por seguridad, cerramos tu sesión. Ingresa de nuevo para continuar." | Página de sistema "Sesión expirada" (transversal, sin top bar) | Redirección automática a `/login` tras una breve pausa, o botón "Ir a iniciar sesión" — `flows/01-auth.md` Pantalla 6 |
| `RESET_TOKEN_EXPIRED` | 400 | "El enlace expiró o ya fue usado." | Nueva contraseña (`/reset-password`) | Acompañado del botón "Solicitar uno nuevo" — `flows/01-auth.md` Pantalla 3 |
| `INSUFFICIENT_PERMISSIONS` / Forbidden | 403 | "No tienes permiso para ver esto." · cuerpo: "Tu rol no tiene acceso a esta sección." | Página de sistema "403" (transversal, conserva top bar) | Botón "Volver al inicio"; también aparece como banner inline puntual: "No tienes permiso para esta acción." — `flows/01-auth.md` Pantalla 6, `flows/05-administrador.md` |
| `NOT_FOUND` | 404 | "No encontramos esta página." · cuerpo: "El recurso que buscas no existe o ya no está disponible." | Página de sistema "404" (transversal) | Variantes contextuales del mismo patrón: "Esta sesión ya no está disponible" (detalle de clase), "Esta notificación ya no está disponible" (detalle de notificación) — `flows/01-auth.md` Pantalla 6, `flows/04-aprendiz.md` |
| `VALIDATION_ERROR` / 422 | 422 | Inline, bajo el campo: texto específico del campo (ej. "La fecha de fin debe ser posterior a la de inicio."). Banner general si aplica a varios campos: "Revisa los campos marcados." | Cualquier formulario/modal de alta o edición | Siempre icono + texto en rojo bajo el campo, **nunca solo borde rojo** — regla transversal de todos los flujos |
| `SCHEDULE_CONFLICT` (doble-booking: `INSTRUCTOR_DOUBLE_BOOKED` / `ENVIRONMENT_DOUBLE_BOOKED`) | 409 | "Este instructor ya tiene una sesión en este horario." / "Este ambiente ya está reservado en este horario." | Modal Agregar/editar sesión, bajo el campo Instructor o Ambiente | Ejemplo del propio contrato: `"El instructor ya tiene una sesión en ese bloque horario."` — `_shared.yaml`, `flows/02-coordinador.md` Pantalla 5 |
| `SCHEDULE_IMMUTABLE` | 409 | "Este horario ya no admite cambios." | Editar/eliminar sesión cuando `schedule.status` ya no es `DRAFT` | Se muestra como banner (cierra el modal) — `flows/02-coordinador.md` Pantalla 4/5 |
| `UNRESOLVED_CONFLICTS` | 409 | "Este horario tiene {N} conflictos sin resolver." | Botón Publicar (deshabilitado con tooltip) y Modal Confirmar publicación | Bloquea la publicación; siempre con acceso directo "Ir al Panel de conflictos" — `flows/02-coordinador.md` Pantallas 4 y 6 |
| `ROW_VERSION_MISMATCH` | 409 | "El horario cambió, recarga e intenta de nuevo." | Guardar / Publicar sobre un horario editado por otra persona | Control de concurrencia optimista (`row_version`) — `flows/02-coordinador.md` Pantalla 6 |
| Conflicto ya resuelto (409, sin código propio en el contrato) | 409 | "Este conflicto ya fue resuelto." | Modal Resolver conflicto | Botón de salida "Entendido", refresca la lista al cerrar — `flows/02-coordinador.md` Pantalla 8 |
| Correo duplicado (409, alta de usuario) | 409 | "Este correo ya está registrado." | Form Crear/editar usuario, modo alta | `flows/05-administrador.md` Pantalla 4 |
| Rol ya asignado (409) | 409 | "Este usuario ya tiene el rol «{nombre del rol}»." | Modal Asignar/revocar rol, modo asignar | `flows/05-administrador.md` Pantalla 6 |
| Código/clave duplicado (409, catálogos/plantillas/parámetros) | 409 | "Este código ya existe en este catálogo." / "Esta clave ya existe." | Editor de catálogo, plantilla o parámetro | `flows/05-administrador.md` §parametrización, `flows/06-backoffice.md` Pantallas 2 y 5 |
| Documento no disponible para descarga (409) | 409 | "Este documento aún no está disponible para descargar." | Botón Descargar cuando `status ≠ AVAILABLE` | Botón se deshabilita en vez de solo fallar — `flows/06-backoffice.md` Pantalla 1 |
| `GENERATION_FAILED` (estado de negocio, no HTTP) | — (`status = GENERATION_FAILED`) | "No se pudo generar el documento." | Badge de fila + banner en el modal al reintentar | Acción "Reintentar" reabre el modal Generar documento prellenado — `flows/06-backoffice.md` Pantalla 1 |
| `RATE_LIMIT_EXCEEDED` | 429 | "Demasiados intentos. Espera un momento y vuelve a intentarlo." | Login, recuperar contraseña, reportes de alto volumen | `flows/01-auth.md` Pantallas 1–2, `flows/05-administrador.md` (reportes) |
| Error genérico de servidor/red | 500 / sin código tipado | "Algo salió mal de nuestro lado." · cuerpo: "Ocurrió un error inesperado. Intenta de nuevo en unos minutos." | Página de sistema "500" (transversal) y banners inline de widget | Botón "Reintentar" repite la última acción; muestra "Código de referencia: {trace_id}" solo si el backend lo entrega — nunca el `message` técnico crudo — `flows/01-auth.md` Pantalla 6 |

---

## 2. Estados vacíos

Patrón general: **texto explicativo + acción sugerida** (nunca un vacío mudo). Los estados vacíos
*positivos* (cero conflictos) usan color de éxito + ícono de check; los estados vacíos *neutros* (sin
resultados de filtro) usan tono neutro, nunca alarmante.

| Tipo de pantalla | Mensaje canónico | Acción sugerida | Dónde se usa |
|---|---|---|---|
| Lista sin resultados (filtro sin coincidencias) | "No hay {entidad} que coincidan con los filtros." (ej. "No hay horarios…", "No hay fichas…", "No hay documentos…") | Botón/enlace **Limpiar filtros** [propuesto — no existe hoy como acción explícita en los flujos, pero todas las listas con filtros lo necesitan para ser consistentes] | Horarios, Fichas, Documentos, Plantillas, Disponibilidad — `flows/02-coordinador.md`, `flows/06-backoffice.md` |
| Sin conflictos (positivo) — panel/dashboard | "Sin conflictos pendientes" (dashboard, ícono de check) / "Sin conflictos — este horario puede publicarse" (Panel de conflictos, mensaje ampliado) | Ninguna (es el estado deseado) | `flows/02-coordinador.md` Pantallas 1 y 7 |
| Sin notificaciones | "No tienes notificaciones nuevas" | Ninguna | Panel de notificaciones (dropdown del top bar) — `flows/01-auth.md` Pantalla 5 |
| Sin sesiones en un horario nuevo | "Aún no hay sesiones — agrega la primera" | Botón **Agregar sesión** | Crear/editar horario — `flows/02-coordinador.md` Pantalla 4 |
| Sin horarios en borrador (widget dashboard) | "No hay horarios en borrador" | Ninguna (o "Crear horario" si se quiere impulsar la acción) | Dashboard Coordinador — `flows/02-coordinador.md` Pantalla 1 |
| Horario archivado sin sesiones activas | "Este horario no tiene sesiones activas" | Ninguna (pantalla de solo lectura) | Detalle de horario — `flows/02-coordinador.md` Pantalla 3 |
| Ficha sin horarios asociados | "Esta ficha aún no tiene horarios" | Botón **Crear horario** (preasigna `ficha_id`) | Detalle de ficha — `flows/02-coordinador.md` Pantalla 12 |
| Ambiente sin franjas configuradas | "Este ambiente no tiene franjas de disponibilidad configuradas" | Ninguna | Detalle de ambiente — `flows/02-coordinador.md` Pantalla 10 |
| Ambientes/instructores sin coincidencias | "No hay ambientes/instructores que cumplan el filtro" | **Limpiar filtros** [propuesto] | Disponibilidad — `flows/02-coordinador.md` Pantalla 9 |
| Sin roles asignados en la sesión de trabajo | "Aún no se asignaron roles en esta sesión de trabajo" | Botón **Asignar rol** | Detalle de usuario, pestaña Roles — `flows/05-administrador.md` Pantalla 5 |
| Sin sesiones activas (usuario) | "Sin sesiones activas" | Ninguna | Detalle de usuario, pestaña Sesiones — `flows/05-administrador.md` Pantalla 5 |
| Sin fichas en seguimiento (filtro) | "No hay fichas en seguimiento para este filtro" | **Limpiar filtros** [propuesto] | Panel de indicadores — `flows/05-administrador.md` Pantalla 1 |
| Catálogo sin valores / sin parámetros | "Este catálogo aún no tiene valores" / "No hay parámetros registrados" | Botón **Agregar valor** / **Crear parámetro** | Parametrización — `flows/05-administrador.md`, `flows/06-backoffice.md` Pantalla 4 |
| Documentos sin coincidencias | "No hay documentos que coincidan con los filtros" | **Limpiar filtros** [propuesto] · botón **Generar documento** siempre visible en la barra | Documentos — `flows/06-backoffice.md` Pantalla 1 |

---

## 3. Confirmaciones (acciones irreversibles o de alto impacto)

Formato estándar: **título corto (verbo + entidad)** + **cuerpo explicando la consecuencia** + **dos
botones** (secundario "Cancelar" siempre a la izquierda/primero, primario a la derecha/último). Las
acciones destructivas usan botón primario en color crítico ("Desactivar", "Revocar", "Archivar");
las no destructivas usan color de marca ("Confirmar publicación", "Generar").

| Acción | Título | Cuerpo | Botón secundario | Botón primario | Dónde se usa |
|---|---|---|---|---|---|
| Publicar horario (con conflictos) | "Publicar horario" | Resumen de solo lectura (ficha, período, N sesiones activas) + banner crítico: "Este horario tiene {N} conflictos sin resolver." | Cancelar | Confirmar publicación *(deshabilitado con candado mientras haya conflictos)* | Modal Confirmar publicación — `flows/02-coordinador.md` Pantalla 6 |
| Publicar horario (sin conflictos) | "Publicar horario" | Resumen + mensaje positivo: "{N} sesiones activas · sin conflictos pendientes" | Cancelar | Confirmar publicación | Modal Confirmar publicación — `flows/02-coordinador.md` Pantalla 6 |
| Resolver conflicto | "Resolver conflicto" | Descripción completa del conflicto (`description`, tipo, sesiones involucradas, fecha de detección); si es el último conflicto pendiente, añade: "Al resolverlo, este horario podrá publicarse." | Cancelar | Confirmar resolución | Modal Resolver conflicto — `flows/02-coordinador.md` Pantalla 8 |
| Desactivar usuario | "Desactivar usuario" [propuesto — el texto exacto no está literal en el flujo, solo "modal de confirmación antes de desactivar"] | "¿Desactivar a {full_name}? Sus sesiones activas se cerrarán y no podrá iniciar sesión." [propuesto] | Cancelar | Desactivar *(rojo)* | Detalle de usuario — `flows/05-administrador.md` Pantalla 5 |
| Revocar rol | "Revocar rol" | "¿Revocar el rol «{role_name}» de {full_name}?" (+ detalle de centro/expiración si aplica) | Cancelar | Revocar *(rojo)* | Modal Asignar/revocar rol, modo revocar — `flows/05-administrador.md` Pantalla 6 |
| Asignar rol | "Asignar rol" | Formulario (Rol, Centro opcional, Expira el opcional) — **no es un diálogo de confirmación**, es alta directa; no requiere doble paso | Cancelar | Asignar | Modal Asignar/revocar rol, modo asignar — `flows/05-administrador.md` Pantalla 6 |
| Revocar sesión de usuario | "Revocar sesión" [propuesto] | "¿Cerrar esta sesión activa de {full_name}? El dispositivo deberá iniciar sesión de nuevo." [propuesto] | Cancelar | Revocar *(rojo)* | Detalle de usuario, pestaña Sesiones — `flows/05-administrador.md` Pantalla 5 |
| Eliminar sesión de clase (horario en borrador) | "Eliminar sesión" [propuesto] | "¿Eliminar esta sesión del horario? Esta acción no se puede deshacer." [propuesto] | Cancelar | Eliminar *(rojo)* | Crear/editar horario, fila de sesión — `flows/02-coordinador.md` Pantalla 4 |
| Eliminar excepción de disponibilidad | "Eliminar excepción" [propuesto] | "¿Eliminar este bloqueo de disponibilidad?" [propuesto] | Cancelar | Eliminar *(rojo)* | Mi disponibilidad (Instructor) — `flows/03-instructor.md` Pantalla 3 |
| Desactivar plantilla de documento | "Desactivar plantilla" [propuesto] | "¿Desactivar esta plantilla? Los documentos ya generados no se ven afectados; dejará de estar disponible para generar nuevos." [propuesto] | Cancelar | Desactivar *(rojo)* | Plantillas de documento — `flows/06-backoffice.md` Pantalla 2 |
| Archivar documento | "Archivar documento" [propuesto] | "¿Archivar este documento? Ya no estará disponible para descarga." [propuesto] | Cancelar | Archivar *(rojo)* | Documentos — `flows/06-backoffice.md` Pantalla 1 |
| Desactivar catálogo / valor / sede | "Desactivar {catálogo / valor / sede}" [propuesto] | "¿Desactivar «{nombre}»? Podrás reactivarlo más adelante si es necesario." [propuesto] | Cancelar | Desactivar *(rojo)* | Parametrización — `flows/05-administrador.md`, `flows/06-backoffice.md` Pantalla 4 |
| Generar documento | "Generar documento" | Formulario (Plantilla, Dominio, Servicio/Entidad propietaria, Título, Datos opcionales) + nota: "La generación toma unos segundos; verás el estado en la lista." [propuesto, dado que es asíncrono — `202 GENERATING`] | Cancelar | Generar | Modal Generar documento — `flows/06-backoffice.md` Pantallas 1 y 6 |
| Salir sin guardar (editor de plantilla) | "¿Salir sin guardar?" [propuesto] | "Tienes cambios sin guardar en esta plantilla. Si sales ahora, se perderán." [propuesto] | Seguir editando | Salir sin guardar *(rojo)* | Editor de plantilla — `flows/06-backoffice.md` Pantalla 5 |

---

## 4. Toasts de éxito

Formato: mensaje corto en pasado/participio, sin signos de exclamación, desaparece solo (no requiere
acción del usuario). Los textos marcados **[propuesto]** completan acciones cuyo endpoint está
documentado pero cuyo copy de toast no está escrito literalmente en el flujo de origen — se redactan
aquí siguiendo el mismo patrón que los toasts que sí tienen texto literal.

| Acción | Toast canónico | Dónde se usa |
|---|---|---|
| Guardar cambios de un horario/sesión | "Sesión agregada." / "Sesión actualizada." [propuesto — el flujo dice "confirmación breve tipo toast" sin texto literal] | Crear/editar horario, Modal Agregar/editar sesión — `flows/02-coordinador.md` Pantalla 5 |
| Publicar horario | "Horario publicado." [propuesto] | Modal Confirmar publicación — `flows/02-coordinador.md` Pantalla 6 |
| Registrar excepción de disponibilidad | "Excepción registrada." | Instructor — Mi disponibilidad — `flows/03-instructor.md` Pantalla 4 |
| Registrar seguimiento de ficha | "Seguimiento registrado." | Instructor — Seguimiento — `flows/03-instructor.md` Pantalla 6 |
| Guardar edición de usuario | "Usuario actualizado." | Form Crear/editar usuario, modo edición — `flows/05-administrador.md` Pantalla 4 |
| Crear usuario (alta) | *(no es toast — tarjeta de éxito inline con la contraseña temporal generada, se mantiene visible hasta que el administrador la copie/cierre)* | Form Crear/editar usuario, modo alta — `flows/05-administrador.md` Pantalla 4 |
| Asignar rol | "Rol asignado." [propuesto] | Modal Asignar/revocar rol — `flows/05-administrador.md` Pantalla 6 |
| Revocar rol | "Rol revocado." [propuesto] | Modal Asignar/revocar rol — `flows/05-administrador.md` Pantalla 6 |
| Revocar sesión de usuario | "Sesión cerrada." [propuesto] | Detalle de usuario, pestaña Sesiones — `flows/05-administrador.md` Pantalla 5 |
| Guardar centro/sede | "Cambios guardados." | Parametrización institucional — `flows/05-administrador.md` |
| Guardar valor de parámetro | "Cambios guardados." | Parametrización de sistema — `flows/05-administrador.md` |
| Guardar catálogo / valor de catálogo | "Catálogo guardado." / "Valor guardado." | Parametrización — `flows/06-backoffice.md` Pantalla 4 |
| Guardar plantilla de documento | "Plantilla guardada — versión {N}." | Editor de plantilla — `flows/06-backoffice.md` Pantalla 5 |
| Documento generado (al completar, `status = AVAILABLE`) | "Documento disponible." [propuesto — la fila pasa de badge "Generando…" a "Disponible", no hay texto de toast literal en el origen] | Documentos — `flows/06-backoffice.md` Pantalla 1 |
| Copiar al portapapeles (`event_id` / JSON de auditoría) | "Copiado al portapapeles." | Modal Detalle de registro de auditoría — `flows/06-backoffice.md` Pantalla 8 |

---

## 5. Labels de acciones/botones (forma canónica única)

Cuando los flujos usaban más de una redacción para la misma acción, se elige **una** y se lista la(s)
variante(s) a reemplazar.

| Acción | Label canónico | Variantes encontradas a reemplazar | Uso |
|---|---|---|---|
| Crear un nuevo registro desde una lista | **"+ Nuevo/Nueva {entidad}"** (concordancia de género: "+ Nuevo horario", "+ Nuevo usuario", "+ Nueva sede", "+ Nueva plantilla", "+ Nueva excepción") | "Crear horario" (Dashboard, `flows/02-coordinador.md` Pantalla 1) y "Nuevo horario" (lista Horarios, Pantalla 2) referían a la misma acción con dos textos distintos — ambas se unifican en **"+ Nuevo horario"**; "Nueva plantilla" (sin "+") se unifica al mismo patrón | CTA primario de listas y del dashboard |
| Guardar cambios de un formulario/modal en modo edición | **"Guardar cambios"** | "Guardar" (a veces usado también para edición) | Modal Agregar/editar sesión (modo edición), formularios de edición |
| Guardar un campo/formulario simple (sin modo alta/edición diferenciado) | **"Guardar"** | — | Nombre del horario, catálogos, parámetros, sedes |
| Confirmar el alta desde un modal/formulario | **verbo + entidad** ("Agregar sesión", "Asignar", "Crear", "Generar") — nunca "Guardar" en modo alta | "Guardar" usado a veces para alta | Modales de creación |
| Validar un horario antes de publicar | **"Validar"** | — | Crear/editar horario (botón secundario en la barra fija) |
| Publicar un horario (dispara el modal de confirmación) | **"Publicar"** (botón en la pantalla) / **"Confirmar publicación"** (botón dentro del modal) | — | Crear/editar horario → Modal Confirmar publicación |
| Continuar un horario en borrador desde una lista/widget | **"Continuar edición"** | — | Widget "Horarios recientes en borrador" (Dashboard) |
| Ir al panel de conflictos desde una alerta/banner | **"Ver panel"** (enlace corto en tarjetas/filas) / **"Ir al Panel de conflictos"** (dentro de un modal o banner de bloqueo) | — | Dashboard, banners de conflictos sin resolver |
| Marcar un conflicto como resuelto (acción de tarjeta) | **"Marcar como resuelto"** | "Resolver" (evitar como label de botón — se reserva "Resolver conflicto" solo como *título del modal*, no como texto de botón) | Panel de conflictos, tarjeta de conflicto pendiente |
| Ver el listado completo desde un widget resumen | **"Ver todos"** (siempre junto al patrón "Mostrando X de N · Ver todos") | — | Todos los widgets tipo dashboard (nunca en pantallas de lista, que usan paginación real) |
| Ver el detalle completo de un registro | **"Ver detalle"** | "Ver" (a secas, en algunas filas de tabla) | Filas de tabla, paneles laterales rápidos |
| Limpiar los filtros aplicados en una lista | **"Limpiar filtros"** [propuesto — gap: ninguna pantalla de lista documenta hoy esta acción explícitamente, pero todas la necesitan junto al estado vacío de "sin coincidencias" (sección 2)] | — | Toda barra de filtros de pantallas de lista |
| Volver a la pantalla/lista anterior | **"‹ Volver a {Sección}"** (con el nombre de la sección de destino, ej. "‹ Volver a Horarios", "‹ Volver a Fichas", "‹ Volver a Disponibilidad", "‹ Volver a Mi horario") | "Volver" a secas | Pantallas de detalle/solo lectura |
| Cancelar un modal sin aplicar cambios | **"Cancelar"** (siempre primer botón, a la izquierda del primario) | — | Todos los modales |
| Reintentar tras un error de carga o de red | **"Reintentar"** | — | Banners de error, estado 500, widgets con error |
| Descargar la versión vigente de un documento | **"Descargar"** (en tabla/fila) / **"Descargar versión vigente"** (en panel de detalle, para distinguir de versiones históricas no descargables) | — | Documentos — `flows/06-backoffice.md` |

---

## 6. Estados de negocio (badge visible + significado)

Ciclo principal del dominio ([design-system.md §5](../design-system.md),
[glossary.md](../../01-context/glossary.md)): `DRAFT → UNDER_REVIEW → PUBLISHED (→ ARCHIVED)`. La UI
**no** hardcodea esta lista — la consume del catálogo `status` parametrizable — pero el texto/ícono de
cada valor conocido debe ser este, en todas las pantallas donde aparece (Horarios, Dashboard, Detalle
de ficha).

| Estado (`schedule.status`) | Etiqueta visible | Ícono/color | Significado para el usuario |
|---|---|---|---|
| `DRAFT` | **Borrador** | Círculo neutro (gris) | El horario se está construyendo; sesiones editables; aún no visible para instructores/aprendices |
| `UNDER_REVIEW` | **En revisión** | Triángulo de advertencia (ámbar) | El horario ya fue validado (`POST /validate`); puede tener conflictos sin resolver; las sesiones ya no se editan desde aquí — los conflictos se resuelven en el Panel de conflictos |
| `PUBLISHED` | **Publicado** | Check (verde/éxito) | El horario es definitivo e **inmutable**; visible para instructores y aprendices; cualquier cambio requiere una nueva versión |
| `ARCHIVED` | **Archivado** | Candado neutro atenuado (gris) | El horario quedó fuera de vigencia; solo consulta histórica, sin acciones de mutación |
| **Conflicto** (no es un valor de `status`, es una condición transversal: `scheduling_conflict.is_resolved = false`) | **Pendiente** (tarjeta/badge crítico) | Ícono según `conflict_type` + color crítico (rojo/naranja) | Hay un choque real detectado (instructor o ambiente doblemente asignado, o sesiones solapadas) que **bloquea la publicación** hasta resolverse; nunca se comunica solo con color — siempre ícono + texto explícito del tipo |
| Conflicto resuelto | **Resuelto** (tarjeta/badge atenuado) | Check verde, tarjeta atenuada | El conflicto fue marcado como resuelto (`POST /conflicts/{id}/resolve`); queda en el historial del Panel de conflictos |

**Otros catálogos de estado que aparecen en las 45 pantallas** (mismo principio — ícono + texto, un
solo badge por valor, ver secciones 1–2 para sus mensajes de error/vacío asociados):

| Dominio | Valores | Etiqueta visible |
|---|---|---|
| Ficha (`enrollment_ficha.status`) | `INDUCTION` / `EXECUTION` / `PRODUCTIVE_STAGE` / `COMPLETED` / `CANCELLED` | Inducción / Ejecución / Etapa productiva / Finalizada / Cancelada |
| Documento (`document.status`) | `GENERATING` / `AVAILABLE` / `ARCHIVED` / `EXPIRED` / `GENERATION_FAILED` | Generando… (spinner) / Disponible / Archivado / Expirado / Error de generación |
| Seguimiento de ficha (`kpi_status` / `risk_level`) | `ON_TRACK` / `AT_RISK` / `CRITICAL` | En seguimiento / En riesgo / Crítico |
| Notificación (`send_status`) | `PENDING` / `SENT` / `FAILED` | Enviando… / Enviado / No se pudo entregar |
| Usuario (`is_active`) | `true` / `false` | Activo / Inactivo |

---

## Referencias

- [design-system.md](../design-system.md) — principios de lenguaje, accesibilidad, estados obligatorios por componente
- [glossary.md](../../01-context/glossary.md) — vocabulario de dominio SENA
- `flows/01-auth.md` … `flows/06-backoffice.md` — las 45 pantallas de origen, con su endpoint/tabla real
- [`_shared.yaml`](../../07-api/contracts/openapi/_shared.yaml) — envelope `Error`, respuestas 400/401/403/404/409/422/429
- [`iam.yaml`](../../07-api/contracts/openapi/iam.yaml) — `outcome` de auditoría de login (`SUCCESS`/`INVALID_PASSWORD`/`USER_NOT_FOUND`/`ACCOUNT_LOCKED`/`TOKEN_EXPIRED`)
