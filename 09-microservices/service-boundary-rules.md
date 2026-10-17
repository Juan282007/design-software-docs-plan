# Reglas de frontera de servicio

> Estado: 🟡 En progreso | Última actualización: 2026-08-26
> Autor: Jesús Ariel González Bonilla (PM + Arquitecto) | Equipo: Arquitectura

Reglas que preservan el desacople entre los 10 microservicios backend. Son **agnósticas de lenguaje**: aplican a cualquier backend.

## 1. Una base de datos por servicio
Cada servicio es dueño exclusivo de su esquema. Hoy conviven como **un schema por módulo** en una sola instancia PostgreSQL; el estado objetivo separa a instancia/credencial por servicio. Ningún servicio accede al schema de otro.

## 2. Propiedad de datos (data ownership)
Cada entidad tiene **un único servicio dueño** (ver [data-ownership-matrix.md](./data-ownership-matrix.md)). Los demás la referencian, no la copian como fuente de verdad.

## 3. Prohibido el join cross-servicio
No se hacen `JOIN` entre tablas de servicios distintos. La composición de datos se hace en la capa de aplicación (llamando a la API del dueño) o mediante **read models** locales alimentados por eventos ([ADR-002](../05-architecture/decisions/records/ADR-002-scheduling-read-models.md)).

## 4. Referencias cross-servicio por UUID sin FK física
Una referencia a una entidad de otro servicio se guarda como `UUID` **sin** `FOREIGN KEY` en la BD. La integridad se garantiza por contrato/evento, no por el motor. **Esto es correcto en microservicios**, no una violación de diseño.

## 5. Comunicación solo por API o eventos
- **Síncrono:** REST sobre el contrato publicado del servicio dueño (máx. ~2 dependencias síncronas por operación, ADR-002).
- **Asíncrono:** eventos por el broker (at-least-once, idempotencia por `event_id`). Ver [communication-patterns.md](./communication-patterns.md).
- Nunca acceso directo a la BD ajena.

## 6. Contratos versionados
Cambios de contrato (API o evento) son **retrocompatibles** o versionados (`/api/v1`, versión de evento). Un cambio incompatible requiere ADR.

## 7. Naming
Eventos y contratos en **inglés** (`<service>.<entity>.<action>`); el dominio y la UI en español.
