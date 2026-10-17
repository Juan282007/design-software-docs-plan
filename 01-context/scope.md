# Alcance

> Estado: 🟡 Borrador | Última actualización: 2026-06-17
> Fuente: [problem-framing.md](../03-product/problem-framing.md) · [discovery-brief.md](../03-product/discovery-brief.md)

## En alcance (MVP)

| Capacidad | Detalle |
|-----------|---------|
| Identidad y acceso | Login JWT, RBAC por feature + scope, gestión de usuarios y roles |
| Datos de referencia | Jerarquía institucional y catálogos del sistema |
| Gestión académica | Programas, competencias, RAPs y fichas de caracterización |
| Ambientes | Ambientes, tipos, disponibilidad, mantenimientos y reservas |
| Motor de horarios | Crear borrador, agregar sesiones, detectar conflictos, publicar |
| Actores | Instructores (competencias, disponibilidad), aprendices, empresas |
| Documentos | Generación de PDFs (constancias, horarios) en object storage |
| Seguimiento | KPIs por ficha, alertas configurables, planes de mejoramiento |
| Auditoría | Log append-only de todas las acciones del sistema |
| Consulta de horario | Vista por instructor y por aprendiz |

## Fuera de alcance (MVP)

- Sugerencia automática de asignación de horario (scheduling con IA)
- Resolución automática de conflictos
- Integración con SOFIA Plus (sistema nacional SENA)
- Notificaciones push móviles
- Exportación a calendarios externos (Google Calendar, Outlook)
- Soporte offline completo (PWA)
- Autenticación de doble factor (2FA) — planificada para V2
- App móvil nativa

## Supuestos

1. Los coordinadores operan desde navegador web (desktop-first)
2. Instructores y aprendices consultan desde navegador o móvil (web responsive)
3. No hay datos históricos a migrar; el sistema parte de cero (greenfield)
4. Los centros tienen conectividad suficiente para operar online en horas laborales
5. Los catálogos de referencia se cargan manualmente en la fase inicial
6. Los números de ficha y códigos de programa provienen de SOFIA Plus (ingreso manual en MVP)

## Restricciones

| Tipo | Restricción |
|------|-------------|
| Técnica | Arquitectura de microservicios con DDD + Hexagonal (pattern-guide) |
| Técnica | Naming técnico en inglés (HALT-DB-NAMING); UI/documentación en español |
| Técnica | Una base de datos por servicio; sin BD compartida |
| Técnica | Máximo 2 dependencias síncronas por servicio (excepto iam para auth) |
| Normativa | Cumplimiento Ley 1581/2012 (protección de datos personales) |
| Normativa | Reglas de negocio según Acuerdo 00003/2012 y Circular 1/2014 SENA |
| Operativa | El SENA opera de lunes a sábado; no hay clases domingos ni festivos |
| Organizacional | Proyecto formativo ADSO; equipo en aprendizaje de prácticas de ingeniería |
