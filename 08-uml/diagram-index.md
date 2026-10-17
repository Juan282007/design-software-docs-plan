# Índice de diagramas

> Estado: 🟡 En progreso | Última actualización: 2026-08-28
>
> Primeros diagramas registrados: los tres flujos de secuencia **ya construidos** de `iam-service`
> y `notification-service`, expresados como bloques `mermaid` en Markdown dentro de
> [`diagrams/source/`](diagrams/source/). La carpeta [`diagrams/exports/`](diagrams/exports/)
> (PNG/SVG exportados) sigue preparada; al exportar, registrar la ruta en la columna correspondiente.
> Al agregar un diagrama, registrarlo como una fila en la tabla de abajo.

| Diagrama | Tipo | Archivo fuente | Exportación | Estado |
|----------|------|---------------|-------------|--------|
| Autenticación — login + emisión de JWT (`iam-service`) | Secuencia | [`diagrams/source/login-jwt.md`](diagrams/source/login-jwt.md) | _(pendiente)_ | 🟢 |
| Outbox → relay → `iam-events` (`iam-service`) | Secuencia | [`diagrams/source/outbox-relay-events.md`](diagrams/source/outbox-relay-events.md) | _(pendiente)_ | 🟢 |
| Consumo de notificación — entrega / retry / DLQ (`notification-service`) | Secuencia | [`diagrams/source/notification-consume.md`](diagrams/source/notification-consume.md) | _(pendiente)_ | 🟢 |
