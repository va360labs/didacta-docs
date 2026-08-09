# La licencia Enterprise

La licencia de Didacta es un **JWT firmado con ES256** (ECDSA P-256) por VA360 LABS. La instancia lo verifica localmente con las claves públicas embebidas en el producto — **no hay llamadas a un servidor de licencias** en runtime.

## Activar la licencia

Hay **dos caminos**, y la precedencia entre ellos es fija: **el entorno gana siempre sobre el panel**.

=== "Desde el panel (recomendado)"

    **Administración → Licencia** (`/admin/licencia`, solo `super_admin`) muestra el estado
    actual y permite pegar la clave, refrescarla o borrarla.

    - La clave se **valida en vivo antes de persistirse**: una clave inválida responde 400 y
      **no pisa la que ya tenías**.
    - Se guarda cifrada como ajuste de instancia y la licencia se **recarga en caliente**:
      **no hace falta reiniciar el contenedor**.
    - Endpoints: `GET`/`PUT`/`DELETE /api/v1/admin/license` y `POST /api/v1/admin/license/refresh`.

=== "Por variable de entorno"

    ```bash
    # En tu .env
    DIDACTA_LICENSE_KEY=eyJhbGciOiJFUzI1NiIs...
    ```

    ```bash
    docker compose -f docker-compose.alpha.yml up -d didacta
    ```

    Por esta vía la licencia se lee **al arrancar**, así que **cambiarla exige reiniciar** el
    contenedor. El estado queda en el log de arranque (`License: active`,
    `License: community (no key set)`…).

!!! note "El entorno gana sobre el panel"
    Si `DIDACTA_LICENSE_KEY` está definida, el panel pasa a **solo lectura**: oculta el
    formulario de activación, muestra la licencia con el distintivo de definida por el
    operador, y los intentos de editarla o borrarla responden **409**. Es lo que permite a un
    operador fijar la licencia por despliegue sin que nadie la cambie desde la interfaz.

## Qué contiene

El payload del JWT incluye: identificadores de licencia y cliente, organización, plan y edición, fechas de emisión y expiración, días de gracia, la lista de **capabilities** contratadas y, opcionalmente, restricciones (dominios permitidos, entorno, versión) y datos de soporte (tier, SLA).

## Estados

| Estado | Cuándo | Capabilities |
| --- | --- | --- |
| `community` | Sin `DIDACTA_LICENSE_KEY` | Ninguna — Community completo. |
| `active` | Licencia válida y vigente | Las del payload. |
| `grace` | Expirada, dentro del periodo de gracia (30 días por defecto) | Las del payload, con aviso. |
| `expired` | Expirada, gracia agotada | Ninguna. |
| `invalid` | Firma o payload inválidos | Ninguna (los endpoints gateados responden 401/402). |
| `dev` | `DIDACTA_DEV_BYPASS=true` (solo fuera de producción) | **Todas** — bypass de desarrollo. |

La instancia avisa 30 días antes de la expiración. El estado es consultable sin autenticación en `GET /api/license` (status, capabilities, avisos, expiración — nunca secretos).

## Verificación

- Algoritmo fijo **ES256**; issuer `didacta.io`, audiencia `didacta-runtime`.
- Claves públicas incluidas en el producto (`packages/license-sdk`), seleccionadas por el `kid` del token.
- El payload se valida además contra un schema estricto; cualquier fallo deja la licencia en `invalid` sin romper la instancia.

## Desarrollo y testing

`DIDACTA_DEV_BYPASS=true` activa todas las capabilities para desarrollar y probar features Enterprise. La guarda es dura: **se ignora con `NODE_ENV=production`** y emite un aviso visible al arrancar. La licencia Enterprise permite usar el código EE para desarrollo y testing sin suscripción; producción requiere licencia válida (ver [Licencias](../comunidad/licencias.md)).

## Obtener una licencia

Contacta con `licensing@didacta.io` o consulta [didacta.io](https://didacta.io).
