# Contexto

> Estado: 🟡 En progreso | Última actualización: 2026-08-01
> Autor: Jesús Ariel González Bonilla (PM + Arquitecto) | Equipo: Arquitectura

Índice de la sección de contexto del sistema de Gestión de Horarios SENA. Aquí se establece **por qué** existe el proyecto, **qué** cubre y **qué vocabulario** se usa, antes de entrar en dominio o arquitectura.

## Propósito

Dar el marco de negocio del sistema: la operación del SENA (asignar instructores, ambientes y franjas horarias a cada ficha activa), el problema que hoy se resuelve de forma manual, los objetivos medibles del proyecto y los límites del MVP. Es la puerta de entrada para cualquier persona que se incorpore, y la fuente de la que dependen las secciones de requisitos, dominio y arquitectura.

## Archivos

| Archivo | Descripción | Estado |
|---------|-------------|--------|
| [overview.md](./overview.md) | Contexto institucional SENA, problema, objetivos y referencias normativas | 🟡 |
| [scope.md](./scope.md) | Alcance del MVP, exclusiones, supuestos y restricciones técnicas/normativas | 🟡 |
| [glossary.md](./glossary.md) | Glosario compartido del dominio SENA y del sistema (lenguaje ubicuo) | 🟡 |

## Plantillas

| Plantilla | Descripción |
|-----------|-------------|
| [_template-project-profile.md](./_template-project-profile.md) | Perfil del proyecto: dimensiones de complejidad, microservicios y trackers |
| [_template-scope-declaration.md](./_template-scope-declaration.md) | Declaración de alcance: MVP, roles, supuestos y criterios de éxito |

## Cómo se relaciona

- El **alcance** ([scope.md](./scope.md)) delimita los 9 microservicios y las restricciones que hereda la [arquitectura](../05-architecture/); el mapeo de conceptos SENA ↔ técnico vive en [02-domain/domain-map.md](../02-domain/domain-map.md).
- El proyecto es un ejercicio formativo del programa **ADSO** (Análisis y Desarrollo de Software) sobre una necesidad real del SENA.
- Marco normativo de referencia: Acuerdo 00003/2012 (Estatuto de la Formación Profesional Integral), Decreto 249/2004, Circular 1/2014 y Ley 1581/2012 (protección de datos).
