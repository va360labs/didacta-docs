# Branding whitelabel

Didacta es **whitelabel de serie**: nada de la marca de una instalación concreta vive en el código. Todo lo visual y textual de tu plataforma es configuración de tenant.

## Marca por organización (Community)

En **Administración → Branding** cada organización configura:

- Logo y favicon.
- Color primario (el tema deriva el resto de la paleta).
- Textos de la pantalla de acceso (titular y subtítulo).

El módulo `mod.theming` amplía la personalización visual: fuentes, tokens de diseño y CSS custom sanitizado.

## Dominio propio

El tenant se resuelve por el **host** de la petición: cada organización tiene uno o más dominios verificados asociados (el primero se crea en el [asistente de configuración](../instalacion/setup-wizard.md)).

- Los administradores gestionan dominios adicionales desde el panel (**Administración → Tenants**).
- La gestión avanzada de **dominios personalizados por tenant** con verificación CNAME es una capability Enterprise (`feat:custom_domains`). La UI muestra el target CNAME configurable con `DIDACTA_CNAME_TARGET`.

!!! note "Sin comodines de subdominio"
    La resolución de tenant exige **match exacto** del dominio (decisión anti-phishing): cada subdominio que quieras usar debe estar dado de alta explícitamente.

## White-label completo (Enterprise)

Ocultar por completo la marca Didacta («powered by Didacta») es la capability Enterprise `feat:white_label`. Sin licencia, la pantalla de white-label existe y explica qué desbloquea — como todas las features Enterprise, [nunca se oculta](../primeros-pasos/ediciones.md). El resto del branding (logo, colores, textos) es Community y no requiere licencia.
