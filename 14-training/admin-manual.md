# Manual de Administración y Soporte

> Estado: 🟡 En progreso | Última actualización: 2026-08-01
> Autor: Jesús Ariel González Bonilla (PM + Arquitecto) | Equipo: Formación

Manual para **administradores del sistema y equipo de soporte** de la plataforma SENA — Gestión de Horarios.

> **Estado real del proyecto: la aplicación aún no existe.** Hoy solo está construida la capa de datos. Este documento define el **marco y la estructura** del manual (índice y secciones previstas por funcionalidad del dominio). Los procedimientos paso a paso con pantallas reales **se completan cuando la app esté construida.** Aquí **no** se describen botones ni pantallas ficticias como si existieran.

## Audiencia y alcance

- **Administrador de Centro / sistema:** gestión de usuarios, roles, datos de referencia y visibilidad de indicadores.
- **Equipo de soporte:** atención a usuarios, resolución de dudas frecuentes y escalamiento.

Para respuesta a incidentes operativos ver [`13-operations/`](../13-operations/). Para el manual de usuarios finales ver [user-manual.md](./user-manual.md).

## Índice previsto del manual

### 1. Introducción
- Propósito de la plataforma y roles del sistema (`COORDINATOR`, `INSTRUCTOR`, `LEARNER`, `ADMIN`).
- Alcance del MVP ([discovery-brief.md](../03-product/discovery-brief.md)).
- _Pendiente: capturas y recorrido de la interfaz de administración._

### 2. Gestión de usuarios y accesos (IAM)
- Alta, baja y modificación de usuarios del centro.
- Asignación de roles y permisos.
- Restablecimiento de acceso.
- _Pendiente: procedimientos concretos una vez exista `iam-service` y su UI._

### 3. Datos de referencia y parametrización
- Mantenimiento de catálogos de dominio: tipos de vinculación, niveles de formación, modalidades, estados de ficha, tipos de ambiente ([discovery M4](../03-product/discovery-brief.md)).
- Jerarquía institucional (Macrorregión → … → Centro → Unidad).
- Nota: los **estados de negocio** se administran como catálogo parametrizable `status`, no se hardcodean ([modeling-conventions §1](../06-data/modeling-conventions.md)).

### 4. Gestión de ambientes de formación
- Registro de ambientes: tipo, capacidad, equipamiento.
- Reglas de disponibilidad por día y mantenimientos.

### 5. Gestión académica (programas y fichas)
- Programas de formación, competencias y RAP.
- Fichas de caracterización y sus estados (Ejecución, Etapa Productiva, Terminada).

### 6. Soporte al flujo de horarios
- Acompañamiento al coordinador en creación → validación → publicación.
- Interpretación de conflictos y del ciclo `BORRADOR → EN_REVISIÓN → PUBLICADO`.
- Versionado de horarios publicados.

### 7. Indicadores y seguimiento
- Lectura de KPIs: utilización de ambientes, carga de instructores, asistencia y avance ([discovery M9](../03-product/discovery-brief.md)).
- Interpretación de alertas.

### 8. Procedimientos de soporte
- Registro y clasificación de solicitudes.
- Preguntas frecuentes (FAQ).
- **Escalamiento:** criterios y ruta hacia el equipo técnico / operaciones ([13-operations](../13-operations/)).

### 9. Anexos
- Glosario de dominio ([glosario](../01-context/glossary.md)).
- Referencias de arquitectura y datos.

## Cómo se completará este manual

Cada sección se redactará con procedimientos paso a paso, capturas reales y casos de error **a medida que cada servicio y su interfaz se construyan**. Mientras tanto, este índice sirve de guía y de checklist de cobertura para el equipo de soporte.

## Referencias

- [user-manual.md](./user-manual.md) · [technical-onboarding.md](./technical-onboarding.md)
- [discovery-brief.md](../03-product/discovery-brief.md) · [glosario](../01-context/glossary.md) · [13-operations](../13-operations/)
