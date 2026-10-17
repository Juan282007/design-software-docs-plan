## Rama destino

- [ ] `dev` — trabajo en curso (caso normal)
- [ ] `qa` — validación (solo desde `hu-*-qa` o cherry-pick aprobado desde `dev`)
- [ ] `main` — solo desde `release/*` o `fix/doc-*` de hotfix

## Tipo de cambio

- [ ] Documento nuevo
- [ ] Actualización de documento existente
- [ ] Corrección menor
- [ ] Reorganización / archivo movido
- [ ] Diagrama o asset

## Secciones afectadas

<!-- Ejemplo: 04-requirements, 08-uml, 13-operations -->

## Definition of Ready — ¿el documento está listo para revisión?

Ver criterios completos en [00-governance/definition-of-ready.md](../00-governance/definition-of-ready.md)

- [ ] Tiene título, estado, fecha y autor o equipo responsable
- [ ] Explica su contexto y propósito
- [ ] El contenido cubre el alcance mínimo acordado para revisión
- [ ] Está enlazado desde el `README.md` de su sección
- [ ] No contiene información sensible

## Checklist obligatorio

- [ ] El documento tiene estado y última actualización.
- [ ] El documento está registrado en el README.md de su sección.
- [ ] Los commits siguen Conventional Commits (obligatorio): formato `<type>(NN-section): short description in English`. Ver [00-governance/git-conventions.md](../00-governance/git-conventions.md).
- [ ] Si se documentan varios microservicios, se estructuró con un commit por microservicio.
- [ ] No se incluyen credenciales, tokens, llaves privadas, datos personales ni URLs privadas sensibles.
- [ ] Los enlaces relativos funcionan.
- [ ] Si hay diagramas, se incluye fuente editable (`.wsd`) y exportación (`.svg`) en `08-uml/`.
- [ ] Si es ADR, se registró en `05-architecture/decisions/README.md`.
- [ ] Si es microservicio, se registró en `09-microservices/service-catalog.md`.
- [ ] CHANGELOG.md fue actualizado si el cambio es relevante para todo el equipo.

## Definition of Done — para el revisor

Ver criterios completos en [00-governance/definition-of-done.md](../00-governance/definition-of-done.md)

- [ ] Aprobado por al menos un revisor del equipo responsable de la sección
- [ ] El estado del documento cambia a 🟢 si corresponde
- [ ] Los enlaces relativos funcionan en la rama destino

## Notas para revisores

<!-- Riesgos, pendientes conocidos o contexto necesario para revisar. -->
