# Preguntas abiertas

> Estado: 🟡 En progreso | Última actualización: 2026-08-28
> Autor: Jesús Ariel González Bonilla (PM + Arquitecto) | Equipo: Arquitectura / Gestión

## Flujo de resolución

1. **Registrar:** cualquier equipo puede agregar una pregunta abierta con estado `ABIERTA`.
2. **Asignar:** el equipo de arquitectura o gestión asigna un responsable y una fecha límite de respuesta.
3. **Resolver:** el responsable responde y cambia el estado a `RESUELTA`.
4. **Escalar si aplica:** si la resolución genera una decisión técnica relevante, crear un ADR en `05-architecture/decisions/records/`. Si afecta requisitos, actualizar `04-requirements/`.
5. **Cerrar:** preguntas resueltas se mantienen en el historial (no eliminar). Mover a sección `## Resueltas` cuando el registro sea largo.

## Formato de registro

| # | Pregunta | Área | Responsable | Fecha límite | Estado | Resolución / ADR |
|---|----------|------|-------------|--------------|--------|------------------|

## Estados válidos

| Estado | Significado |
|--------|-------------|
| `ABIERTA` | Sin respuesta aún |
| `EN REVISIÓN` | Responsable asignado, trabajando en ella |
| `RESUELTA` | Tiene respuesta, puede generar ADR o actualización |
| `CANCELADA` | Ya no aplica (indicar por qué) |

---

## Preguntas abiertas

Registro central de los puntos abiertos (PA-NN) del producto. Origen: [03-product/vision.md](../03-product/vision.md) §8 (PA-01…PA-04) y [03-product/roadmap.md](../03-product/roadmap.md) (PA-03). Al resolverse, mover a `## Resueltas` y —si aplica— generar el ADR correspondiente.

| # | Pregunta | Área | Responsable | Fecha límite | Estado | Resolución / ADR |
|---|----------|------|-------------|--------------|--------|------------------|
| PA-01 | ¿El coordinador puede modificar un horario `PUBLISHED` directamente, o siempre debe crear una nueva versión? *(supuesto actual: siempre nueva versión — RN-SCH-01)* | Producto / Arquitectura | Producto | Antes de construir `scheduling-service` (Fase 2 / D) | ABIERTA | — |
| PA-02 | ¿Los instructores contratistas tienen un límite de horas distinto al de planta? *(supuesto: sí — RN-ACT-01)* | Producto / Normativa | Producto | Antes de `actors-service` (Fase 1 / C) | ABIERTA | — |
| PA-03 | ¿Se requiere integración con SOFIA Plus en el MVP o en fases posteriores? *(supuesto: fases posteriores)*. Si se adelanta, `academic-management` (EPC-04) debe incorporar el ACL correspondiente. | Producto / Sponsor | Sponsor | Confirmar antes de cerrar alcance MVP | ABIERTA | — |
| PA-04 | ¿Un aprendiz puede estar matriculado en más de una ficha activa a la vez? | Producto / Dominio | Producto | Antes de `actors-service` (Fase 1 / C) | ABIERTA | — |

---

## Resueltas

| # | Pregunta | Área | Resolución | ADR / Doc relacionado |
|---|----------|------|------------|-----------------------|
