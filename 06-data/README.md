# Datos

> Estado: 🟡 Revisión | Última actualización: 2026-10-07
> Autor: Kevin Andrey Culma (Arquitecto) | Equipo: Arquitectura.

## Contenido

Documenta modelos de datos, diccionario y estrategia de migración.

> **Diferencia con `02-domain`:** esta sección describe la implementación de persistencia (tablas, columnas, tipos, relaciones, índices, migraciones). Las entidades de negocio y sus reglas se documentan en [`02-domain/`](../02-domain/). Un mismo concepto (ej: "Horario") tendrá una definición de dominio allá y un esquema de base de datos aquí.

> **Diferencia con `09-microservices`:** aquí vive el modelo de datos **global** del sistema — diccionario conceptual compartido, estrategia de migración y analítica. El modelo **transaccional propio de cada servicio** (esquema local, tablas internas) se documenta en `09-microservices/services/<servicio>/data-model.md`. No duplicar: si un concepto es solo del servicio, va allá; si es compartido entre varios servicios, va aquí.

## Archivos

| Archivo | Descripción | Estado |
|---------|-------------|--------|
| [models.md](./models.md) | Modelos conceptuales, lógicos o físicos de datos | 🔴 |
| [modeling-conventions.md](./modeling-conventions.md) | Estándar transversal de modelado (estados, auditoría, estructura DDL) | 🟡 |
| [data-dictionary.md](./data-dictionary.md) | Índice de entidades por servicio y reglas comunes | 🟡 |
| [normalization-assessment.md](./normalization-assessment.md) | Evaluación de normalización (NF) y calidad del modelo real | 🟢 |
| [migration-strategy.md](./migration-strategy.md) | Estrategia Liquibase: orden DDL, seeds, rollbacks, ambientes | 🟡 |
| [database-map.md](./database-map.md) | Mapa actualizado de esquemas y tablas por microservicio | 🟡 |

## Plantillas

| Plantilla | Descripción |
|-----------|-------------|
| [_template-data-model.md](./_template-data-model.md) | Modelo de datos: entidades DDD, relaciones, patrones de acceso y privacidad |
| [_template-db-design.md](./_template-db-design.md) | DB Design: esquema físico, índices, FKs, migraciones y RTO/RPO |
| [_template-data-migration-plan.md](./_template-data-migration-plan.md) | Plan de migración: mapeo, pasos, validaciones y rollback |
