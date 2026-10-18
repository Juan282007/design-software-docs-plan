# R5 - Contratos y compatibilidad de audit-service

> Estado: 🟡 En progreso | Última actualización: 2026-10-18
> Autor: Juan Pablo Gómez Perdomo | Equipo: Audit

## Contexto

El reto individual R5 exige que `audit-service` detecte incompatibilidades de contrato antes de integrar cambios a `develop`. El proveedor HTTP es `audit-query-api`; el consumidor implementado es `audit-mfe`. El worker no publica eventos: consume el envelope canónico que emiten los servicios de dominio.

## Contenido

### Límites de responsabilidad

| Artefacto | Dueño | Uso |
| --- | --- | --- |
| `api/openapi/audit.yaml` | audit-service | Contrato HTTP versionado de `audit-query-api`. `info.version` usa SemVer. |
| `pacts/audit-mfe-audit-query-api.json` | audit-mfe | Pact consumidor generado por la interacción real `GET /api/v1/audit-records`. |
| `contracts/pact/audit-mfe-audit-query-api.json` | audit-service | Copia versionada que el proveedor verifica en CI. Debe coincidir con el Pact generado por audit-mfe. |
| `api/events/audit-consumed-event.schema.json` | audit-service | JSON Schema del envelope canónico que recibe `audit-worker`. |

### Gates de pull request

El workflow `contract-compatibility.yml` de audit-service ejecuta tres checks para PR hacia `develop`:

1. Valida que `info.version` del OpenAPI sea SemVer y ejecuta `oasdiff breaking` contra el contrato de `develop`. Un cambio incompatible termina con código distinto de cero.
2. Ejecuta la prueba JSON Schema del envelope aceptado y rechazado, y verifica que el parser productivo mantenga la misma decisión.
3. Levanta un proveedor de prueba que usa el router, validación RS256 y autorización reales; Pact CLI verifica el contrato de `audit-mfe` contra ese proveedor.

El harness de Pact no forma parte de los binarios de producción ni abre rutas nuevas. Solo entrega datos deterministas a CI para que la interacción sea repetible.

### Demostración de incompatibilidad bloqueada

En una rama temporal, eliminar `GET /audit-records` o estrechar una respuesta de `audit.yaml`; al comparar esa versión contra `develop`, `oasdiff breaking` debe fallar. El cambio temporal no se integra. La demostración queda disponible mediante el workflow para revisión y sustentación.

### Verificación manual

```powershell
# audit-mfe: genera y verifica el contrato consumidor
npm ci
npm run test:contract

# audit-service: valida esquema y toda la suite Go
go test ./...

# audit-service: comprueba la interacción del proveedor local
go run ./cmd/audit-contract-provider
# En otra terminal: GET http://127.0.0.1:18080/api/v1/audit-records?limit=10
# con Accept: application/json y Authorization: Bearer pact-contract-token
```

Resultado esperado: `200`, `Content-Type: application/json`, una lista paginada y un registro con los UUID, `event_type`, `source_service` y `received_at` definidos en el Pact.

Para la verificación oficial, Docker debe estar iniciado y se ejecuta `pact_verifier_cli` como hace el workflow, usando `pactfoundation/pact-cli:1.5.0.5`. Su resultado esperado es una verificación exitosa del proveedor; si falla una coincidencia de status, headers o cuerpo, el check queda rojo.

## Referencias

- Plan de mejoramiento, numeral 12: R5 - Contratos y compatibilidad.
- [Guías API](../07-api/guidelines.md).
- [Estrategia de pruebas](./testing-strategy.md).
- [ADR-006](../05-architecture/decisions/records/ADR-006-polyglot-stack-strategy.md).
