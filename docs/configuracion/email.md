# Email

Didacta envía emails transaccionales — invitaciones, reset de contraseña, recordatorios de clase, digests de comunidad, avisos de suscripción — a través de SMTP, con **dos niveles de configuración**.

## 1. SMTP global (variables de entorno)

Es el transporte por defecto de toda la instancia. Requiere las **tres** variables a la vez:

```bash
SMTP_HOST=smtp.tu-proveedor.com
SMTP_PORT=587
SMTP_FROM="Mi Academia <no-reply@ejemplo.com>"
# Opcionales:
SMTP_USER=usuario
SMTP_PASS=contraseña
SMTP_SECURE=true      # fuerza TLS implícito; sin definir, lo decide el puerto
```

Si falta cualquiera de las tres primeras, **no hay SMTP global**: solo podrán enviar correo los tenants que configuren SMTP propio.

## 2. SMTP por tenant (panel de administración)

Cada organización puede definir su propio servidor en **Administración → Configuración → Notificaciones** (la pestaña «Notificaciones» de `/admin/configuracion`). Ese ajuste tiene prioridad sobre el global, y sus credenciales se guardan **cifradas at-rest** en la base de datos.

## En desarrollo: Mailpit

El compose oficial incluye [Mailpit](https://mailpit.axllent.org/): todos los correos de la instalación caen en su buzón web (`http://localhost:8025`) en lugar de salir a Internet. Perfecto para probar flujos completos sin enviar nada real.

!!! warning "Mailpit muestra todo"
    La UI de Mailpit muestra **todos** los correos, incluidos los resets de contraseña. Por eso el compose la publica solo en `127.0.0.1`. Nunca la expongas.

## Enlaces dentro de los emails

Los enlaces absolutos de los emails (botón de «Restablecer contraseña», invitaciones…) se construyen con `WEB_PUBLIC_URL`. **Sin esa variable, en producción los emails llevan enlaces a `localhost`.** Configúrala siempre:

```bash
WEB_PUBLIC_URL=https://campus.ejemplo.com
```

## Plantillas de email (Administración → Emails)

El **contenido** de los correos transaccionales se edita por tenant en **Administración → Emails**, sin tocar código: asunto y cuerpo admiten variables (`{{tenantName}}`, `{{userName}}`…) y cada plantilla tiene un default sensato. Las **partes estructurales** de cada email (el botón de acción, el código OTP, los botones de aprobar/rechazar) no son editables: garantizan que el flujo siga funcionando aunque el texto cambie.

Claves del núcleo:

| Clave | Cuándo se envía |
| --- | --- |
| `auth.password_reset` | Restablecer contraseña. |
| `enrollment.welcome` | Alta por invitación/API con enlace de acceso. |
| `membership.welcome` | Alta por compra de membresía (`mod.subscriptions`). |
| `billing.welcome` | Alta por compra pública de un curso (`mod.billing`): «Define tu contraseña». |
| `subscriptions.renewal_warning` | Aviso de renovación próxima. |
| `payment_connections.access_expiring` | Acceso derivado de una cuenta de pago externa a punto de expirar. |
| `subscriptions.admin_digest` | Resumen diario de suscripciones para admins. |

Los módulos añaden las suyas al mismo catálogo — por ejemplo, `mod.member-registration` registra sus 4 plantillas (`member_registration.otp_code`, `.approval_request`, `.welcome_approved`, `.rejection`). Los endpoints de lectura/edición están en [Referencia → Administración](../api/referencia/administracion.md).

## Si no hay SMTP configurado

Los emails no se envían pero **quedan registrados en los logs** — la plataforma no se bloquea. Aun así, flujos como la invitación de usuarios o el reset de contraseña dependen del correo: configúralo antes de dar de alta usuarios reales.
