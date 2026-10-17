# Modelo de datos — <PROJECT_KEY>

> **PLANTILLA** — Copiar como `data-model.md` y completar.
> Eliminar esta línea antes de hacer commit.

> Estado: 🔴 Pendiente | Última actualización: YYYY-MM-DD
> Autor: Por definir | Equipo: Arquitectura de Datos

## Alcance del modelo

- **Bounded contexts cubiertos:** [listar]
- **Fuera de alcance:** [lo que no cubre este modelo global — los modelos transaccionales de cada servicio van en 09-microservices/]

## Glosario del dominio

| Término | Definición | Entidad técnica |
|---------|------------|-----------------|
| [Horario] | [Asignación de un instructor a un bloque de tiempo] | `schedule` |

## Entidades

### Tabla maestra de entidades

| Entidad | Tipo DDD | Bounded Context | Propósito | PII |
|---------|----------|-----------------|-----------|-----|
| [Customer] | Aggregate Root | [Sales] | | Sí |

### Definición detallada

#### Entidad: [nombre]

- **Tipo:** Aggregate Root / Entity / Value Object
- **Bounded context:** [nombre]
- **Propósito:** [una línea]
- **Identidad:** UUID (PK)

| Atributo | Tipo lógico | Obligatorio | PII | Constraint | Notas |
|----------|-------------|-------------|-----|------------|-------|
| id | UUID | Sí | No | PK | generado por app |
| created_at | timestamp | Sí | No | NOT NULL | UTC |
| updated_at | timestamp | Sí | No | NOT NULL | UTC, auto-update |

**Invariantes de negocio:**
- [Invariante 1]

**Política de borrado:** hard / soft / cascade — justificación: [...]

## Relaciones

| Origen | Destino | Cardinalidad | ON DELETE | Integridad |
|--------|---------|--------------|-----------|------------|
| | | 1:N | RESTRICT | obligatoria |

## Patrones de acceso

| # | Patrón de consulta | Frecuencia | Columnas filtro | Tabla(s) |
|---|--------------------|------------|-----------------|----------|
| 1 | [Buscar por email] | Alta | email | [tabla] |

## Privacidad y retención

| Entidad | Datos PII | Política de retención | TTL |
|---------|-----------|----------------------|-----|
| | | soft-delete N días → hard-delete | |

## Separación de responsabilidades

> Este modelo cubre datos **globales y compartidos** entre servicios.
> Los modelos transaccionales propios de cada servicio van en `09-microservices/services/<servicio>/data-model.md`.

## Referencias

- [PRD](../03-product/prd.md)
- [Architecture](../05-architecture/architecture.md)
- [DB Design](./db-design.md)
- [Security Threat Model](../05-architecture/security-threat-model.md)
