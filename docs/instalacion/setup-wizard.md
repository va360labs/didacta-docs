# Asistente de configuración

El primer arranque de Didacta **no usa seeds de datos**: la instancia se configura con el asistente web (`/setup`), que crea la organización y la cuenta del primer administrador.

## Cuándo aparece

Mientras la instancia no tenga ningún tenant, la web redirige **cualquier ruta** al asistente. En cuanto el asistente termina, deja de estar disponible para siempre: el endpoint de inicialización responde `409 ALREADY_INITIALIZED` si ya existe una organización.

## Los pasos

1. **Bienvenida** — presentación del asistente.
2. **Organización** — nombre de la organización (obligatorio) y dominio público, pre-rellenado con el host desde el que accedes. En «Configuración avanzada» puedes ajustar el *slug* (autogenerado, apto para DNS).
3. **Módulos** — lista de módulos disponibles. Los módulos **core** (cursos, aprendizaje, evaluaciones…) aparecen pre-marcados y no se pueden desmarcar; los opcionales (comunidad, gamificación, Fundae, aula virtual…) se activan o no a tu gusto. Todo se puede cambiar después en **Administración → Módulos**.
4. **Tu cuenta** — nombre, email y contraseña del primer administrador (**mínimo 12 caracteres**, con medidor de fuerza).
5. **Listo** — tour rápido (Cursos, Comunidad, Administración) y recomendación de activar MFA.

## Qué crea exactamente

Todo en una única transacción:

- Los 6 **roles de sistema**: `super_admin`, `tenant_admin`, `formador`, `alumno`, `auditor`, `empresa_manager`.
- El **tenant** (organización) con su **dominio primario verificado** — el hostname que usaste para acceder. Si no es `localhost`, se añade también `localhost` como dominio secundario (útil para administración local).
- El **usuario administrador** con rol `super_admin` y contraseña hasheada con argon2.
- La activación de los **módulos** elegidos (los core siempre).
- Un registro de **auditoría** `setup.initialized`.
- Una **sesión iniciada**: al terminar entras directamente, sin volver a hacer login.

!!! tip "El dominio importa"
    Didacta resuelve el tenant por el **host** de la petición. Accede al asistente desde el dominio definitivo de tu instalación (no desde una IP provisional) para que quede registrado como dominio primario. Siempre puedes añadir dominios después en **Administración → Organizaciones**.

## Después del asistente

1. **Administración → Marca** — logo, colores y textos de la pantalla de acceso.
2. **Administración → SMTP** — servidor de correo real (por defecto apunta a Mailpit).
3. **Administración → IA** — proveedor y clave de IA si vas a usar tutor/corrección/generación IA.
4. **Administración → Módulos** — ajustar qué módulos están activos.

→ Sigue el [recorrido visual](recorrido-visual.md) para ver estos pasos y todo el camino hasta el primer certificado emitido, captura a captura.
