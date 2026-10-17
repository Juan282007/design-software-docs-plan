# Documentación de microservicios

> Estado: 🟢 Estable | Última actualización: 2026-08-01
> Autor: Jesús Ariel González Bonilla (PM + Arquitecto) | Equipo: Por definir

Este documento define cuándo y cómo registrar documentación de microservicios.

## Regla crítica

No crear carpetas en `09-microservices/services/` hasta que el servicio exista en el repositorio de código o su creación esté formalmente aprobada por arquitectura.

No crear microservicios ficticios para llenar la estructura.

**Requisito de aprobación:** Todo servicio nuevo debe tener una ADR en `05-architecture/decisions/records/` o una decisión registrada en `15-project-control/open-questions.md` con estado RESUELTA antes de crear la carpeta en `09-microservices/services/`. Sin ese artefacto, el PR será rechazado.

## Naming de microservicios

El nombre de cada microservicio sigue el patrón:

```
<proyecto>-<contexto>-<sufijo>
```

- `<proyecto>`: nombre corto del sistema o producto (ej: `horarios`, `facturacion`).
- `<contexto>`: bounded context o dominio del servicio (ej: `scheduling`, `auth`, `notification`).
- `<sufijo>`: tipo funcional del servicio según la tabla siguiente.

### Sufijos estándar

| Sufijo | Responsabilidad |
|--------|-----------------|
| `-api` | Expone contrato HTTP/REST/GraphQL hacia clientes o frontends |
| `-worker` | Procesa tareas asíncronas desde cola o evento; no expone endpoints |
| `-workflow` | Orquesta flujos de larga duración, sagas o procesos multi-paso |
| `-db` | Gestiona esquema, migraciones y parametrización de base de datos |
| `-gateway` | Enruta, autentica y compone respuestas entre servicios |
| `-scheduler` | Ejecuta tareas programadas o disparadores temporales |
| `-notifier` | Gestiona envío de notificaciones hacia canales externos |

El sufijo describe la responsabilidad del servicio, no la tecnología usada internamente.

Un mismo dominio puede tener varios servicios con diferentes sufijos:

```
horarios-scheduling-api
horarios-scheduling-worker
horarios-scheduling-db
```

## Ubicación

Cada servicio real se documenta en:

```text
09-microservices/services/<nombre-del-servicio>/
```

El nombre de la carpeta debe coincidir con el nombre del repositorio de código.

## Flujo

### 1. Verificar catálogo

Abrir `09-microservices/service-catalog.md` y confirmar que el servicio no exista.

### 2. Copiar plantilla

```bash
cp -r 09-microservices/_template/ 09-microservices/services/<nombre-servicio>/
```

Ejemplo ilustrativo, no crear sin aprobación:

```bash
cp -r 09-microservices/_template/ 09-microservices/services/sena-horarios-scheduler/
```

### 3. Completar README del servicio

El `README.md` del servicio debe incluir:

- Responsabilidad del servicio.
- Bounded context.
- Owner.
- Repositorio de código.
- Dependencias.
- Enlaces a contrato API, modelo de datos, eventos y runbook.

### 4. Registrar en catálogo

Agregar fila en `09-microservices/service-catalog.md`:

```markdown
| <nombre-servicio> | <descripción> | <owner> | [repo](<url>) | 🟡 |
```

### 5. Completar archivos mínimos

| Archivo | Qué documentar |
|---------|----------------|
| `README.md` | Responsabilidad, owner, dependencias y links |
| `api-contract.md` | Endpoints, request, response y errores |
| `data-model.md` | Modelo transaccional propio del servicio |
| `events.md` | Eventos publicados y consumidos |
| `runbook.md` | Deploy, rollback, variables y troubleshooting |

`README.md` y `api-contract.md` deben estar al menos en 🟡 antes del primer merge a `main` del código del servicio.

## Commits

Usar Conventional Commits en inglés:

```bash
git checkout -b feat/doc-service-<nombre-servicio>
git add 09-microservices/services/<nombre-servicio>/
git add 09-microservices/service-catalog.md
git commit -m "docs(09-microservices): register <service-name> service"
git push origin feat/doc-service-<nombre-servicio>
```

Si se documentan varios servicios, usar un commit por microservicio cuando sea posible.

## Contrato API

- `09-microservices/services/<servicio>/api-contract.md` describe el contrato operativo del servicio.
- `07-api/contracts/openapi/` almacena archivos OpenAPI formales cuando existan.
- Mantener enlaces cruzados entre ambos cuando aplique.
