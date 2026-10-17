# Modelo de amenazas de seguridad — <PROJECT_KEY>

> **PLANTILLA** — Copiar como `security-threat-model.md` y completar.
> Eliminar esta línea antes de hacer commit.

> Estado: 🔴 Pendiente | Última actualización: YYYY-MM-DD
> Autor: Por definir | Equipo: Arquitectura + Seguridad

## Superficie de ataque

| Componente | Tipo de exposición | Datos en riesgo |
|------------|-------------------|-----------------|
| API Gateway | Pública (Internet) | Todos los datos de la API |
| Base de datos | Interna | PII, datos transaccionales |
| Servicio de autenticación | Pública | Credenciales, tokens |

## Amenazas identificadas (STRIDE)

| ID | Categoría STRIDE | Amenaza | Componente afectado | Probabilidad | Impacto |
|----|-----------------|---------|---------------------|--------------|---------|
| T-001 | Spoofing | Suplantación de identidad por token robado | API Gateway | Media | Alto |
| T-002 | Tampering | Modificación de datos en tránsito | API REST | Baja | Alto |
| T-003 | Information Disclosure | Exposición de PII en logs | Backend | Media | Alto |
| T-004 | Denial of Service | Saturación de endpoints públicos | API Gateway | Alta | Medio |

## Controles de seguridad

### Autenticación y autorización

| Control | Implementación | Estado |
|---------|----------------|--------|
| Autenticación | [OAuth2 / JWT / otro] | Pendiente |
| Autorización | [RBAC / ABAC] — matriz rol x recurso | Pendiente |
| MFA | [Sí / No / Opcional] | Pendiente |
| Session timeout | [X minutos de inactividad] | Pendiente |

### Protección de datos

| Control | Dato protegido | Mecanismo |
|---------|----------------|-----------|
| Cifrado en tránsito | Todo | TLS 1.2+ |
| Cifrado en reposo | PII | AES-256 |
| Tokenización | Datos de pago (si aplica) | [Proveedor] |
| Secrets management | Credenciales, API keys | [Vault / SSM / otro] |

### Rate limiting y protección perimetral

| Regla | Límite | Respuesta |
|-------|--------|-----------|
| Por IP (global) | [X req/hora] | 429 + Retry-After |
| Por usuario autenticado | [X req/min] | 429 |
| Login fallido | [5 intentos] | Bloqueo temporal |

## Datos sensibles (PII / PCI)

| Dato | Clasificación | Almacenamiento | Acceso permitido |
|------|---------------|----------------|------------------|
| Email | PII contacto | Cifrado en reposo | Owner + Admin |
| Documento | PII identidad | Cifrado en reposo | Owner + Admin |
| Contraseña | Credencial | Hash bcrypt (nunca plano) | Nadie (solo verificación) |

## Procedimiento ante incidente de seguridad

Ver [00-governance/security-rules.md](../00-governance/security-rules.md) para el procedimiento completo.

## Referencias

- [NFR](../04-requirements/non-functional.md)
- [Architecture](./architecture.md)
- [Security Rules](../00-governance/security-rules.md)
- OWASP Top 10: https://owasp.org/www-project-top-ten/
