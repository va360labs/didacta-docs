# Modo sencillo

El **modo sencillo** es una forma corta de usar el panel pensada para quien monta su academia por primera vez y quiere ir de cero a **vender** sin tocar nada técnico. No es otra edición ni otra instalación: es un **interruptor por academia** que pliega lo avanzado. **No se borra nada**: todo lo del modo completo sigue ahí y se puede volver a él cuando quieras.

## Quién lo tiene encendido

- Es un ajuste de la academia (`simple_mode`) y **está apagado por defecto**: una instalación nueva arranca en **modo completo**, con el menú lateral de siempre.
- Una plataforma que aprovisiona academias para sus creadores (como Didacta Cloud, véase [Tu academia](../cloud/your-academy.md)) puede crearlas ya en modo sencillo.
- Quien administra el aula lo enciende y lo apaga sin pedir permiso a nadie:
    - **Encender**: en modo completo, **Configuración → «Volver al modo sencillo»**.
    - **Apagar**: en modo sencillo, **Ajustes → «Usar el modo completo»** (**«Cambiar al modo completo»**).
    - Por API: `GET`/`PUT /api/v1/admin/simple-mode` (`{ "enabled": true }`).

## Qué cambia

| | Modo completo | Modo sencillo |
| --- | --- | --- |
| Navegación | Menú lateral con todos los grupos | **Pestañas arriba**: Empieza aquí, Inicio, Comunidad, Cursos, Directos, Miembros, Ranking, Alumnos, Ventas, Ajustes y «Más» (lo demás, con el mismo nombre que en el menú completo) |
| Primer uso | El [asistente de configuración](../installation/setup-wizard.md) y a explorar | **Alta en 3 preguntas** y la pantalla «Empieza aquí» |
| Editor de cursos, Ajustes, Ventas, Alumnos, Directos, comunidad y Referidos | Pantallas completas | Versiones simplificadas (el resto de rutas `/admin/…`, `/studio/…` siguen abriéndose desde «Más») |
| Avatar | Perfil, certificados… | Además «Mi progreso» e **«Invita y gana»** (referidos) |

Las **rutas son las mismas** en los dos modos; un enlace a `/admin/payments` o a `/studio/courses` funciona igual. Las páginas del catálogo de módulos llevan capturas del modo completo salvo que digan lo contrario.

### Alta en 3 preguntas

La primera vez que entra quien administra un aula en modo sencillo, se le hacen tres preguntas:

1. **¿Cómo se llama tu academia?** (y el **idioma de la academia**: español, inglés o portugués de Brasil; se puede cambiar en Ajustes).
2. **¿Qué vas a vender?** Un **curso** (pago único) o una **membresía** (pago mensual), con su título.
3. **¿Cuánto cuesta?** Precio y moneda; la pantalla enseña cuánto queda de cada venta.

Al terminar crea, **en borrador**, el curso o el plan, los espacios básicos de la comunidad y un post de bienvenida. Es idempotente: repetirlo no duplica nada. Un aula que ya tenía cursos, alumnos o ventas antes de pasar al modo sencillo se **adopta tal cual**, sin alta.

### «Empieza aquí»

Seis pasos hasta la primera venta, con el estado de cada uno:

1. **Tu academia**: logo y color.
2. **Tu primer curso**: con una lección basta (vídeo subido o enlazado, o texto).
3. **Precio**: el del alta es solo un precio de partida; hasta que se confirma, el paso no cuenta como hecho.
4. **Conecta tus cobros**: con [cobros de plataforma](../modules/catalog/payouts.md), la tarjeta «Conectar cobros» (y después los datos fiscales); sin ellos, las claves de Stripe propias ([Pagos](../modules/catalog/billing.md)).
5. **Publicar**: genera el enlace de venta (`/catalog/<slug>` o `/join`).
6. **Invita** a tus primeros alumnos.

Tras la primera venta la pantalla pasa a un resumen (ventas, alumnos, próximo directo).

## Dónde quedan las opciones avanzadas

En **Ajustes → «Todas las opciones avanzadas»** (correo, cobros con tu propia cuenta, Zoom, idioma y más), y con **Ajustes → «Usar el modo completo»** vuelve el menú lateral. Los mensajes automáticos de [mod.retention](../modules/catalog/retention.md) y los premios de [mod.gamification](../modules/catalog/gamification.md) se gestionan en **Ajustes** del modo sencillo. Los [niveles de la membresía](../modules/catalog/subscriptions.md#niveles) están en **Ventas** (sección «Niveles»), y la [dirección de la academia](branding.md#direccion-de-la-academia-subdominio), en **Ajustes** si la instalación lo permite.

## Para el operador

- Qué módulos tiene activos cada aula no cambia con el modo: es independiente de [Módulos](../modules/managing.md).
- El modo se puede fijar al crear una academia desde la administración de instancia (`simpleMode` al crear el tenant).
- Es una preferencia **de la academia**, no de cada persona: la ven todos los administradores y formadores igual.
