# Overview

> Estado: 🟡 Borrador | Última actualización: 2026-06-17
> Fuente: [discovery-brief.md](../03-product/discovery-brief.md) · [problem-framing.md](../03-product/problem-framing.md)

## Contexto institucional

El SENA (Servicio Nacional de Aprendizaje) es la entidad pública colombiana encargada de la formación profesional integral. Opera centros de formación a nivel nacional, cada uno con decenas de fichas de caracterización activas que agrupan aprendices vinculados a un programa de formación.

La operación diaria de un centro exige asignar, para cada ficha activa: instructores con competencias certificadas, ambientes físicos adecuados y franjas horarias compatibles. Este proceso de construcción de horarios se realiza hoy de forma manual o semi-manual, sin herramienta integrada que valide conflictos ni comunique el resultado a los actores.

Este proyecto corresponde al programa de formación **ADSO** (Análisis y Desarrollo de Software), como ejercicio de diseño y construcción de un sistema real bajo prácticas de ingeniería de software.

## Problema

Los coordinadores académicos no cuentan con una herramienta centralizada para **crear, validar y publicar horarios** de formación, lo que genera conflictos no detectados a tiempo (instructor doble-asignado, ambiente sobreprogramado), falta de visibilidad para instructores y aprendices, y horas de trabajo manual en reconciliación.

Detalle completo en [problem-framing.md](../03-product/problem-framing.md).

## Objetivos

1. Permitir crear un horario válido y publicarlo en **menos de 1 hora** (vs. 4–8 h manuales)
2. **Detectar automáticamente** los conflictos de recursos antes de publicar
3. Dar **visibilidad en tiempo real** del horario a instructores y aprendices
4. Habilitar el **seguimiento pedagógico** (asistencia, avance, alertas de deserción) de cada ficha
5. Garantizar **trazabilidad** completa de las acciones del sistema (auditoría)

## Alcance del sistema

Plataforma de microservicios (9 servicios) que cubre identidad, datos de referencia, gestión académica, ambientes, motor de horarios, actores, documentos, seguimiento y auditoría. Ver [scope.md](./scope.md).

## Referencias

- Acuerdo 009 de 2024 (SENA)
- Acuerdo 00003 de 2012 — Estatuto de la Formación Profesional Integral
- Decreto 249 de 2004 — Estructura interna del SENA
- Circular 1 de 2014 — Seguimiento a la formación
- Ley 1581 de 2012 — Protección de datos personales (Habeas Data)
