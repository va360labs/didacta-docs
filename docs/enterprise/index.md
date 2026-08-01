# Didacta Enterprise

Enterprise añade a Community un conjunto cerrado de **capabilities transversales del core** pensadas para organizaciones grandes: federación de identidad, aprovisionamiento automático, marca blanca completa, auditoría extendida…

La regla de reparto es simple y no cambia: **los módulos son siempre Community**; lo único de pago son estas capabilities de plataforma. Ver [Ediciones](../primeros-pasos/ediciones.md).

## Qué incluye

| Área | Capabilities |
| --- | --- |
| Identidad corporativa | SSO SAML 2.0 · SSO OIDC · aprovisionamiento SCIM · MFA obligatorio por organización |
| Plataforma multi-organización | Multi-tenant real (varias organizaciones aisladas en una instancia) · dominios personalizados por tenant |
| Marca | White-label completo (ocultar la marca Didacta) |
| Cumplimiento | Auditoría con retención de 7 años firmada · exportes XLSX/PDF firmados criptográficamente |
| API | Webhooks de alto rendimiento (cola, HMAC, dead-letter) · límites de API elevados |

El detalle de cada una, con su efecto exacto y sus límites frente a Community: [Capabilities](capabilities.md).

## Cómo se comporta sin licencia

Didacta sigue la convención de n8n — **el gating nunca oculta**:

- Las pantallas Enterprise existen siempre, con su título y descripción; sin licencia muestran un aviso de qué desbloquean en lugar del panel.
- Los endpoints Enterprise responden **HTTP 402** con la capability requerida en el cuerpo.
- Community funciona **completa** sin licencia: sin límites de usuarios, sin trials, sin degradaciones.

## Cómo se activa

Con una licencia JWT firmada por VA360 LABS en la variable `DIDACTA_LICENSE_KEY`. El funcionamiento completo (estados, periodo de gracia, verificación) está en [Licencia](licencia.md).

## Contratación

Enterprise incluye además account manager dedicado, onboarding guiado e integraciones con sistemas existentes. Información y contacto en [didacta.io](https://didacta.io) · `licensing@didacta.io`.
