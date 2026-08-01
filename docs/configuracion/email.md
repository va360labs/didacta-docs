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

Cada organización puede definir su propio servidor en **Administración → SMTP**. Ese ajuste tiene prioridad sobre el global, y sus credenciales se guardan **cifradas at-rest** en la base de datos.

## En desarrollo: Mailpit

El compose oficial incluye [Mailpit](https://mailpit.axllent.org/): todos los correos de la instalación caen en su buzón web (`http://localhost:8025`) en lugar de salir a Internet. Perfecto para probar flujos completos sin enviar nada real.

!!! warning "Mailpit muestra todo"
    La UI de Mailpit muestra **todos** los correos, incluidos los resets de contraseña. Por eso el compose la publica solo en `127.0.0.1`. Nunca la expongas.

## Enlaces dentro de los emails

Los enlaces absolutos de los emails (botón de «Restablecer contraseña», invitaciones…) se construyen con `WEB_PUBLIC_URL`. **Sin esa variable, en producción los emails llevan enlaces a `localhost`.** Configúrala siempre:

```bash
WEB_PUBLIC_URL=https://campus.ejemplo.com
```

## Si no hay SMTP configurado

Los emails no se envían pero **quedan registrados en los logs** — la plataforma no se bloquea. Aun así, flujos como la invitación de usuarios o el reset de contraseña dependen del correo: configúralo antes de dar de alta usuarios reales.
