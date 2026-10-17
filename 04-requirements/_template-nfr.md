# Requisitos no funcionales — <PROJECT_KEY>

> **PLANTILLA** — Copiar como `non-functional.md` y completar.
> Eliminar esta línea antes de hacer commit.

> Estado: 🔴 Pendiente | Última actualización: YYYY-MM-DD
> Autor: Por definir | Equipo: Arquitectura + Producto

## Rendimiento

| NFR | Objetivo | Estrategia | Verificación |
|-----|----------|------------|--------------|
| Latencia p95 API | < [X]ms | Cache, índices, async | Test de carga |
| Throughput | [X req/s] | Pool de conexiones | JMeter / k6 |
| Tiempo de carga UI | < [X]s | Lazy loading, CDN | Lighthouse |

## Disponibilidad y confiabilidad

| NFR | Objetivo | Estrategia | Verificación |
|-----|----------|------------|--------------|
| Disponibilidad | [99.9%] | Réplicas, healthchecks | Uptime monitor |
| RTO | [30 min] | Runbook, failover | Simulacro |
| RPO | [5 min] | Backups, réplica | Restore test |
| MTTR | [< 1h] | Alertas, runbook | Postmortem |

## Escalabilidad

| NFR | Objetivo | Estrategia |
|-----|----------|------------|
| Usuarios concurrentes | [X] | Stateless backend, autoescalado |
| Crecimiento de datos | [X GB/año] | Particionamiento, archivado |

## Seguridad

| NFR | Requisito | Referencia |
|-----|-----------|------------|
| Autenticación | [OAuth2 / JWT / otro] | security-threat-model.md |
| Autorización | [RBAC / ABAC] | security-threat-model.md |
| Cifrado en tránsito | TLS 1.2+ | |
| Cifrado en reposo | [AES-256 para PII] | |
| Rate limiting | [X req/min por usuario] | |

## Mantenibilidad

| NFR | Objetivo |
|-----|----------|
| Cobertura de pruebas | > [80%] |
| Tiempo de onboarding | < [2 días] para nuevo desarrollador |
| Documentación | Actualizada antes de cada release |

## Observabilidad

| NFR | Requisito |
|-----|-----------|
| Logs | Estructurados JSON; campos obligatorios: timestamp, level, service, requestId |
| Métricas | RED por endpoint; USE por recurso |
| Trazas | 100% en errores; sampling adaptativo en normales |
| Alertas | P0 notifica en < 5 min |

## Compliance

| Regulación | Aplica | Controles requeridos |
|------------|--------|---------------------|
| GDPR | Sí / No | |
| PCI-DSS | Sí / No | |
| [Regulación local] | Sí / No | |

## Referencias

- [PRD](../03-product/prd.md)
- [Architecture](../05-architecture/architecture.md)
- [Security Threat Model](../05-architecture/security-threat-model.md)
