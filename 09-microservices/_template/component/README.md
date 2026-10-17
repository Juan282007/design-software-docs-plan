# [component-name]

> **PLANTILLA** — Copiar a `services/<nn>-<nombre>-service/components/<componente>/` y completar.
>
> Estado: 🔴 Pendiente | Última actualización: YYYY-MM-DD
> Autor: Nombre Apellido | Equipo: nombre-del-equipo

## Tipo de componente

<!-- Marcar el sufijo que aplica -->
- [ ] `-api` — REST API sincrónica
- [ ] `-worker` — Consumidor de eventos / cola
- [ ] `-workflow` — Orquestación de pasos con compensaciones
- [ ] `-scheduler` — Tarea periódica
- [ ] `-notifier` — Envío de notificaciones salientes
- [ ] `-gateway` — Punto de entrada / proxy

## Responsabilidad

<!-- Una oración. Qué hace este componente y cuál es su frontera. -->

## Tecnologías

| Capa | Tecnología | Versión |
|------|-----------|---------|
| Runtime | (pendiente) | |
| Framework | (pendiente) | |
| BD / Queue | (pendiente) | |

## Variables de entorno requeridas

| Variable | Descripción | Ejemplo |
|----------|-------------|---------|
| `SERVICE_PORT` | Puerto de escucha | `8080` |
| `DB_URL` | Cadena de conexión | `postgresql://...` |

## Contrato

Ver [contract.md](./contract.md)
