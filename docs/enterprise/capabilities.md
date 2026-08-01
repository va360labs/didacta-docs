# Capabilities Enterprise

La lista **cerrada** de capabilities Enterprise — 11 hoy, definidas en `packages/license-sdk/src/capabilities.ts`. Una licencia activa incluye en su payload las capabilities contratadas; todo lo demás es Community.

## Identidad y acceso

| Capability | Qué desbloquea |
| --- | --- |
| `feat:sso.saml` | Configuración de login corporativo **SAML 2.0** (`/admin/sso/saml`). |
| `feat:sso.oidc` | Configuración de login corporativo **OIDC** (`/admin/sso/oidc`). |
| `feat:scim` | **Aprovisionamiento SCIM 2.0** desde Okta/Entra: alta, baja y actualización automática de usuarios (`/scim/v2` + emisión del token). |
| `feat:mfa.enforcement` | **MFA obligatorio a nivel de organización** (política aplicada en el login). Community permite MFA opcional por usuario. |

## Plataforma

| Capability | Qué desbloquea |
| --- | --- |
| `feat:multi_tenant.real` | **Multi-tenant real**: crear múltiples organizaciones aisladas en una misma instancia. Community opera con una organización. |
| `feat:custom_domains` | **Dominios personalizados por tenant** con verificación CNAME gestionada desde el panel. |
| `feat:white_label` | **White-label completo**: sobrescribir la marca y ocultar «powered by Didacta». El branding básico (logo, colores, textos) es Community. |

## Cumplimiento

| Capability | Qué desbloquea |
| --- | --- |
| `feat:audit.long_retention` | Retención del log de auditoría de **7 años firmada**. Community: 90 días. |
| `feat:reports.advanced_signed` | **Exportes XLSX/PDF firmados criptográficamente** (verificables offline). |

## API

| Capability | Community | Con la capability |
| --- | --- | --- |
| `feat:api.webhooks.high_throughput` | 1 endpoint/organización, máx. 3 tipos de evento, entrega directa con 1 reintento | 20 endpoints, eventos ilimitados, cola con backoff, **firma HMAC-SHA256**, dead-letter con reintento |
| `feat:api.rate_limit.elevated` | 100 req/min autenticado · 30 anónimo | 1000 req/min autenticado · 300 anónimo |

## Cómo se aplica el gating

- **Backend**: los endpoints Enterprise llevan `@RequiresCapability(...)` (o `license.requireCapability(...)` en el service) y responden **402** con la capability requerida cuando no está activa:

    ```json
    {
      "statusCode": 402,
      "error": "CapabilityRequiredError",
      "capability": "feat:sso.saml",
      "path": "/api/v1/admin/sso/saml/config"
    }
    ```

- **Frontend**: la pantalla siempre existe; el panel real va dentro de un gate que, sin capability, muestra el aviso de upsell. La web consulta las capabilities activas en `GET /api/license` y `GET /api/v1/me/modules`.
- **Código**: solo las capabilities con código físicamente separado viven en ficheros `*.ee.*` dentro del core (cubiertos por la licencia Enterprise); el resto se gatea en línea. Ningún fichero `.ee` puede existir en `modules/` — lo garantiza el validador `ee-fence` en CI.
