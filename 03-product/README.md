# Producto

> Estado: 🟡 En progreso | Última actualización: 2026-08-28
> Autor: Jesús Ariel González Bonilla (PM + Arquitecto) | Equipo: Producto

## Contenido

Documenta la intención del producto **SENA Gestión de Horarios**: qué problema resuelve, para quién, cómo evoluciona y qué se construye en qué orden. Es el puente entre el descubrimiento ([discovery-brief.md](./discovery-brief.md), [problem-framing.md](./problem-framing.md)) y los requisitos ejecutables ([04-requirements/](../04-requirements/)).

> **Marco estable vs. estado vivo:** estos documentos son el **marco estable** del producto (visión, fases, épicas). El **estado vivo del avance** —qué HU está en qué ambiente— se lleva en el **board (GitHub Project #3), que es la única fuente de verdad (SSOT) del progreso**. Los `.md` no replican el board.

## Archivos

| Archivo | Descripción | Estado |
|---------|-------------|--------|
| [vision.md](./vision.md) | Visión, objetivos, usuarios, propuesta de valor y alcance MVP vs. futuro | 🟡 |
| [roadmap.md](./roadmap.md) | Fases (0–4), entregables por fase y dependencias | 🟡 |
| [product-backlog.md](./product-backlog.md) | Backlog priorizado por épicas (EPC-NN) con MoSCoW | 🟡 |
| [discovery-brief.md](./discovery-brief.md) | Discovery: contexto, problema, MVP y hallazgos por dominio | 🟡 |
| [problem-framing.md](./problem-framing.md) | Definición del problema, 5 Whys y criterios de éxito | 🟡 |

## Cómo se relacionan

```
discovery-brief ─▶ problem-framing ─▶ vision ─▶ roadmap ─▶ product-backlog
                                                                │
                                                                ▼
                                          04-requirements (functional / user-stories / traceability)
```

## Plantillas

| Plantilla | Descripción |
|-----------|-------------|
| [_template-discovery-brief.md](./_template-discovery-brief.md) | Discovery brief: problema, resultado esperado y hallazgos |
| [_template-problem-framing.md](./_template-problem-framing.md) | Problem framing: definición del problema, 5 Whys y criterios de éxito |
| [_template-prd.md](./_template-prd.md) | PRD: épicas (EPC-NNN), features (FEA-NNN), HUs, criterios de aceptación y NFRs |
| [_template-backlog.md](./_template-backlog.md) | Backlog priorizado con MoSCoW, mapa de dependencias y resumen por release |
