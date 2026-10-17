# Guía de patrones — <PROJECT_KEY>

> **PLANTILLA** — Copiar como `pattern-guide.md` y completar.
> Eliminar esta línea antes de hacer commit.

> Estado: 🔴 Pendiente | Última actualización: YYYY-MM-DD
> Autor: Por definir | Equipo: Arquitectura

## Patrón seleccionado

| Campo | Valor |
|-------|-------|
| Patrón | [DDD Táctico / Hexagonal / Clean Architecture / Modular Monolith / CQRS] |
| Justificación | [por qué este patrón para este proyecto] |
| ADR de decisión | [ADR-NNN] |

## Capas del patrón

| Capa | Responsabilidad | Puede depender de | No puede depender de |
|------|-----------------|-------------------|----------------------|
| Domain | Entidades, lógica de negocio pura | Ninguna | Infraestructura, UI |
| Application | Casos de uso, orquestación | Domain | Infraestructura |
| Infrastructure | Persistencia, mensajería, APIs externas | Domain, Application | — |
| Interface | Controllers, CLI, eventos de entrada | Application | Domain directo |

## Reglas de dependencia

- Las dependencias apuntan hacia adentro (Infraestructura → Aplicación → Dominio).
- La capa de Dominio no importa nada de capas externas.
- Los puertos (interfaces) se definen en Dominio/Aplicación; los adaptadores en Infraestructura.

## Mapeo de entidades por capa

| Entidad / Concepto | Capa | Tipo DDD |
|-------------------|------|----------|
| [Horario] | Domain | Aggregate Root |
| [HorarioRepository] | Domain (port) | Repository interface |
| [PostgresHorarioRepository] | Infrastructure | Adapter |
| [CrearHorarioUseCase] | Application | Use Case |

## Bounded Contexts (si aplica DDD)

| Bounded Context | Responsabilidad | Equipo owner |
|-----------------|-----------------|--------------|
| [Scheduling] | Gestión de disponibilidad y horarios | |
| [Notifications] | Envío de alertas y notificaciones | |

## Anti-patrones prohibidos

- No llamar directamente a repositorios desde controllers sin pasar por casos de uso.
- No poner lógica de negocio en controllers o en la capa de infraestructura.
- No compartir entidades de dominio entre bounded contexts — usar DTOs o eventos.

## Referencias

- [Architecture](./architecture.md)
- [Data Model](../06-data/data-model.md)
- [ADRs](./decisions/README.md)
