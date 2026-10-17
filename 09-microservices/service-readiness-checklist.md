# Checklist de madurez de servicio

> Estado: 🟡 En progreso | Última actualización: 2026-08-26

Criterios de madurez aplicados individualmente a los **10 microservicios backend**. No se presupone que todos estén en el mismo nivel.

## Nivel 0 — Doc

- [ ] Servicio registrado en [service-catalog.md](./service-catalog.md).
- [ ] Responsabilidad y frontera explícitas.
- [ ] Ownership de datos definido.
- [ ] ADR cuando la creación/cambio cruza bounded contexts.

## Nivel 1 — Datos

- [ ] `data-model.md` coherente con migraciones reales.
- [ ] Schema/credencial propia según estrategia vigente.
- [ ] Constraints, auditoría, rollback y seeds revisados.

## Nivel 2 — Dev

- [ ] Componentes desplegables documentados.
- [ ] API/worker/workflow implementados.
- [ ] Contratos generados/validados desde `shared-contracts`.
- [ ] Unit tests + integración de adapters.
- [ ] Eventos con idempotencia, retry y DLQ donde aplique.

## Nivel 3 — QA

- [ ] Integración y e2e en `qa`.
- [ ] Pact/contract tests.
- [ ] OTel, logs estructurados, métricas y readiness.
- [ ] Runbook probado.

## Nivel 4 — Staging/Main

- [ ] Promoción `develop → qa → staging → main` conforme a política Git.
- [ ] Backup/restore probado.
- [ ] Secretos fuera de Git.
- [ ] SLO/alertas definidos.
- [ ] Rollback probado.

## Regla específica de Notification

Para considerar `notification-service` listo al menos en Dev:

- [ ] `notification-api` y `notification-worker` construyen y prueban.
- [ ] `notification_db` contiene `sent_notification`, `notification_template` y `outbox`.
- [ ] consume `monitoring.alert.triggered` sin acceso a `monitoring_db`;
- [ ] publica `notification.notification.sent` mediante Outbox;
- [ ] retry/backoff/DLQ cubiertos;
- [ ] SMTP/IN_APP y PII revisados;
- [ ] Monitoring no conserva endpoints/tablas/workers de notificación.
