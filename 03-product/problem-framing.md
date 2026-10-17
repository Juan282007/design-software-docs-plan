# Problem Framing — SENA Gestión de Horarios

> Fase: 01-Discovery | Agente: A05 | Estado: 🟡 Borrador
> Fecha: 2026-06-17
> Prerequisito: [discovery-brief.md](./discovery-brief.md)

## Definición del problema

**Para:** Coordinadores académicos de centros de formación SENA

**Quienes:** Necesitan crear y mantener horarios de formación para múltiples fichas activas simultáneamente

**El problema de:** No contar con una herramienta que valide automáticamente conflictos de recursos (instructor, ambiente, franja horaria) y que comunique el horario a los actores involucrados

**Impacta en:** La calidad del servicio de formación y la eficiencia operativa del centro

**Una solución exitosa:** Permitirá crear un horario válido y publicarlo en menos de 1 hora, con cero conflictos detectados post-publicación, y con visibilidad inmediata para instructores y aprendices

---

## Análisis de causa raíz — 5 Whys

**Síntoma observado**: Conflictos de horario detectados después de publicar (instructor en dos lugares, ambiente sobre-asignado)

| # | Por qué | Causa identificada |
|---|---------|-------------------|
| 1 | ¿Por qué hay conflictos post-publicación? | Porque la validación es manual y sujeta a error humano |
| 2 | ¿Por qué la validación es manual? | Porque no existe sistema que centralice disponibilidad de instructores, ambientes y franjas |
| 3 | ¿Por qué no existe ese sistema? | Porque cada dominio (instructores, ambientes, horarios) está gestionado en herramientas separadas o en papel |
| 4 | ¿Por qué están separados? | Porque el sistema legado (SOFIA Plus) no fue diseñado para gestión operativa de horarios por centro |
| 5 | ¿Por qué el sistema legado no cubre esto? | Porque fue diseñado para registro académico nacional, no para la coordinación operativa diaria de un centro |

**Causa raíz**: La ausencia de un sistema operativo de centro que integre la visibilidad de recursos (instructores, ambientes) con el proceso de construcción de horarios y su publicación a los actores.

---

## Mapa de impacto

```
Causa raíz
│
├─▶ Para COORDINADORES
│     ├─ 4–8 h/semana en construcción y reconciliación de horarios
│     ├─ Errores detectados tarde (llaman a instructor para avisar del conflicto)
│     └─ Sin historial de cambios ni responsables
│
├─▶ Para INSTRUCTORES  
│     ├─ Desconocen su carga horaria con antelación
│     ├─ No pueden planificar sus actividades fuera del centro
│     └─ Reciben cambios de horario de forma informal (llamada/WhatsApp)
│
├─▶ Para APRENDICES
│     ├─ Desconocen cuándo y dónde tienen clase
│     ├─ No reciben alertas de cambios de horario
│     └─ Confusión cuando el instructor no aparece por un conflicto no resuelto
│
└─▶ Para ADMINISTRADORES
      ├─ Sin métricas de utilización real de ambientes
      ├─ Sin indicadores de carga de instructores
      └─ Incapacidad de detectar cuellos de botella antes de que ocurran
```

---

## Criterios de éxito

### Criterios funcionales (MVP)

| Criterio | Condición verificable |
|----------|----------------------|
| CF-01 | El coordinador puede crear un borrador de horario para una ficha activa |
| CF-02 | El sistema detecta conflictos (instructor doble-asignado, ambiente sobre-asignado, franja solapada) antes de publicar |
| CF-03 | El coordinador puede publicar un horario sin conflictos con un solo acción |
| CF-04 | El instructor puede consultar su horario vigente por semana |
| CF-05 | El aprendiz puede consultar el horario de su ficha |
| CF-06 | Un horario `PUBLICADO` no puede ser modificado; los cambios crean una nueva versión |

### Criterios no funcionales (MVP)

| Criterio | Métrica |
|----------|---------|
| CNF-01 Latencia de disponibilidad | `GET /environments/available` y `GET /instructors/available` responden en < 300 ms (p95) |
| CNF-02 Disponibilidad del API | > 99 % en horario laboral (07:00–20:00 hora Colombia) |
| CNF-03 Detección de conflictos | La validación completa de un horario con 20 sesiones tarda < 2 s |
| CNF-04 Seguridad | Autenticación requerida en todos los endpoints; autorización por rol |
| CNF-05 Observabilidad | Cada operación tiene request ID trazable en logs |

### Criterios de exclusión (fuera del MVP)

- Sugerencia automática de horario (scheduling IA)
- Integración con SOFIA Plus
- Notificaciones push móviles
- Exportación a Google Calendar / Outlook
- Soporte offline completo (PWA)
- Gestión de etapa productiva de aprendices

---

## Supuestos del MVP

1. Los coordinadores usan el sistema desde navegador web (desktop-first)
2. Los instructores y aprendices consultan desde navegador o dispositivo móvil (responsive web)
3. No hay datos históricos a migrar; el sistema parte de cero
4. Los centros tienen conectividad suficiente para operar online durante horas de trabajo
5. Los catálogos de referencia (tipos de ambiente, modalidades, etc.) se cargan manualmente en fase inicial

---

## Riesgos identificados en esta fase

| Riesgo | Probabilidad | Impacto | Mitigación |
|--------|-------------|---------|------------|
| Resistencia del coordinador al cambio de herramienta | Alta | Alto | Involucrar coordinadores en sesiones de validación de UX desde sprint 1 |
| Reglas de disponibilidad de ambientes más complejas de lo documentado | Media | Medio | Modelar primero las 24 reglas conocidas y dejar extensibilidad en el motor |
| Integración con SOFIA Plus requerida en MVP | Media | Alto | Confirmar con sponsor antes del cierre de discovery |
| Conectividad insuficiente en centros rurales para operación online | Baja | Alto | Diseñar contratos de API pensando en eventual modo offline (v2) |
