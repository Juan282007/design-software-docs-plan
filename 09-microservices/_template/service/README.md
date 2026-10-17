# [service-name] — [descripción en una línea]

> **PLANTILLA** — Copiar esta carpeta a `services/<nn>-<nombre>-service/` y completar.
> Ver instrucciones en [microservices-documentation.md](../../../00-governance/microservices-documentation.md)
>
> Estado: 🔴 Pendiente | Última actualización: YYYY-MM-DD
> Autor: Nombre Apellido | Equipo: nombre-del-equipo

## Responsabilidad

<!-- Una sola oración: qué hace este servicio y qué NO hace. -->

## Bounded context

<!-- Entidades que este servicio POSEE. Ningún otro servicio escribe directamente en estas tablas. -->

| Entidad | Descripción |
|---------|-------------|
| `[entidad]` | [descripción] |

## Módulo de origen

<!-- Módulo de dominio del que surge este bounded context (M1-M10) -->

## Dependencias

| Servicio | Tipo | Motivo |
|----------|------|--------|
| `iam-service` | sync | Validación de token JWT en cada request |

## Componentes desplegables

| Componente | Sufijo | Descripción |
|------------|--------|-------------|
| `[nombre]-api` | `-api` | API REST principal |

## Base de datos

- Nombre lógico: `[nombre]_db`
- Motor: (por definir)
- Esquema: (pendiente)

## Links

- Repo: (pendiente)
- Data model: [data-model.md](./data-model.md)
- Eventos: [events.md](./events.md)
- Runbook: [runbook.md](./runbook.md)
- Decisiones internas: [decisions.md](./decisions.md)
