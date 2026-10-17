# Perfil del proyecto — <PROJECT_KEY>

> **PLANTILLA** — Copiar como `project-profile.md` y completar todos los campos.
> Eliminar esta línea antes de hacer commit.

> Estado: 🔴 Pendiente | Última actualización: YYYY-MM-DD
> Autor: Por definir | Equipo: Arquitectura

## Identificación

| Campo | Valor |
|-------|-------|
| Project key | `PRJ-<DOMINIO>-<PRODUCTO>` |
| Nombre del proyecto | |
| Tipo | `greenfield` / `legacy-rewrite` / `legacy-extend` / `integration` |
| Owner | |
| Fecha de kickoff | YYYY-MM-DD |

## Dimensiones de complejidad

| Dimensión | Valor | Implicancia |
|-----------|-------|-------------|
| Tiene interfaz de usuario | sí / no | Activa diseño UX en 12-ux-ui/ |
| Canal UI | web / móvil / ambos / ninguno | |
| Persistencia | sql / nosql / híbrido / ninguna | Activa diseño de datos en 06-data/ |
| Complejidad de dominio | trivial / media / alta | |
| Volumen esperado | < 10k usuarios / 10k-1M / > 1M | |
| PII / compliance | ninguno / básico / regulado | Regulado activa seguridad documental reforzada |
| Integraciones externas | 0 / 1-3 / 4+ | |
| Disponibilidad requerida | best-effort / 99% / 99.9% / 99.99% | |
| Multi-tenant | no / soft / hard isolation | |

## Microservicios del proyecto

| Nombre del servicio | Sufijo | Responsabilidad | Estado |
|--------------------|--------|-----------------|--------|
| `<proyecto>-<contexto>-api` | `-api` | | 🔴 |

> Ver convenciones de naming en [00-governance/microservices-documentation.md](../00-governance/microservices-documentation.md).

## Trackers externos

| Herramienta | URL | Propósito |
|-------------|-----|-----------|
| [Jira / Trello / GitHub Projects] | | Gestión de HUs y sprints |

## Referencias

- [Contexto del proyecto](./overview.md)
- [Alcance](./scope.md)
- [Convenciones ágiles](../00-governance/agile-conventions.md)
