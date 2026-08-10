# Aprovisionamiento SCIM 2.0

Didacta expone un servidor **SCIM 2.0** (RFC 7643 y RFC 7644) para que tu proveedor de identidad —Okta, Microsoft Entra ID (Azure AD), Auth0, Google Workspace— cree, actualice y desactive usuarios automáticamente, sin que nadie mantenga dos listas a mano.

Es una capability Enterprise: `feat:scim`. Sin ella los endpoints de usuarios responden **402** y el IdP no puede aprovisionar.

!!! warning "Lee primero qué no está soportado"
    El alcance es **deliberadamente acotado a usuarios**. Groups, `bulk`, `changePassword`, `sort` y `etag` no existen aquí, y un conector configurado para usarlos falla. La lista completa está en [Qué no está soportado](#que-no-esta-soportado); conviene leerla antes de configurar el conector y no durante la primera sincronización.

## Qué soporta, de un vistazo

Esto es literalmente lo que declara `GET /scim/v2/ServiceProviderConfig`, que es lo que consulta el IdP antes de dejarte guardar la configuración:

| Función | Estado |
| --- | --- |
| Recurso `User` | **Sí** — listar, leer, crear, modificar y borrar |
| Recurso `Group` | **No** |
| `patch` (PatchOp) | **Sí**, con un conjunto acotado de rutas |
| `filter` | **Sí**, solo `userName eq "…"` (`maxResults`: 200) |
| `bulk` | **No** (`maxOperations: 0`) |
| `changePassword` | **No** |
| `sort` | **No** |
| `etag` | **No** |
| Autenticación | `oauthbearertoken` — token estático **por organización** |

## La URL del endpoint

```text
https://<tu-dominio>/scim/v2
```

- **No lleva el prefijo `/api/v1`.** SCIM es una de las pocas rutas del producto fuera de él (junto con las sondas, `/metrics` y `/api/license`), porque muchos conectores rechazan una URL base que no termine exactamente en `/scim/v2`.
- **Es la misma para todas las organizaciones**: quien identifica la organización es el token, no la URL. Si usas dominios personalizados, vale cualquiera que resuelva a la instancia.
- El panel te la muestra ya construida con tu dominio, lista para copiar.

## Emitir el token

**Administración → Seguridad → Aprovisionamiento (SCIM)** (`/admin/scim`). Solo `super_admin` y `tenant_admin` pueden gestionarlo.

1. Pulsa **Generar token**. Se crea un secreto con el prefijo `scim_` y 43 caracteres aleatorios (256 bits de entropía).
2. El token se muestra **una sola vez**. Cópialo y pégalo en el IdP en ese momento.
3. A partir de ahí el panel solo enseña el **prefijo** (los 13 primeros caracteres), la fecha de creación y el estado.

De lo que se guarda en la base de datos solo hay un **hash SHA-256**: el token en claro no queda persistido en ninguna parte, así que nadie —tampoco quien tenga acceso a la base de datos— puede recuperarlo. Si lo pierdes, la única salida es generar uno nuevo.

| Acción | Efecto |
| --- | --- |
| **Generar** | Crea el token y lo revela una vez. |
| **Rotar** | Genera uno nuevo y **revoca el anterior en el acto**. |
| **Revocar** | Borra el token: el IdP empieza a recibir **401** inmediatamente. |

Los mismos tres botones existen como API, con JWT de administrador y `feat:scim`: `GET` · `POST` · `DELETE /api/v1/admin/scim/token`.

!!! warning "Un solo token activo por organización"
    Cada organización tiene **un único token SCIM**. Generar uno nuevo invalida el anterior sin periodo de solape: si tuvieras dos conectores apuntando a la misma organización, el segundo dejaría de funcionar en cuanto rotes.

!!! note "«Aún no usado por el IdP»"
    El panel muestra un campo de último uso que **hoy nunca se rellena**: el servidor no anota la fecha de la última petición SCIM autenticada, así que la etiqueta se queda siempre en «aún no usado». Para comprobar que el IdP está llegando, mira el log de auditoría (`scim.user.*`), no ese campo.

## Autenticación

Todas las peticiones, **incluidas las de discovery**, llevan:

```http
Authorization: Bearer scim_xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
```

!!! note "El discovery también pide token"
    RFC 7644 §4 permite que `ServiceProviderConfig`, `ResourceTypes` y `Schemas` sean anónimos. En Didacta **no lo son**: la guarda cubre todos los endpoints, para que nada de la configuración se filtre antes de tener credencial. Configura el token en el IdP **antes** de pulsar «Test connection», o la prueba devolverá 401. Okta y Entra ID lo aceptan sin problema.

Devuelven **401** la falta de cabecera `Authorization`, un esquema que no sea `Bearer` (`Basic`, `ApiKey`…), un token vacío y un token no reconocido (revocado, rotado o de otra instancia).

El token resuelve la organización y **todas las consultas quedan acotadas a ella**: el token de una organización no ve, no modifica y no encuentra usuarios de otra. Un `GET` de un identificador que existe en otra organización devuelve 404, no 403.

## Endpoints

| Método | Ruta | Qué hace | Capability |
| --- | --- | --- | --- |
| `GET` | `/scim/v2/ServiceProviderConfig` | Declara lo que soporta el servidor. | — |
| `GET` | `/scim/v2/ResourceTypes` | Un único tipo: `User`. | — |
| `GET` | `/scim/v2/Schemas` | Atributos del esquema `User` core. | — |
| `GET` | `/scim/v2/Users` | Lista paginada, con filtro opcional. | `feat:scim` |
| `POST` | `/scim/v2/Users` | Crea un usuario (**201**). | `feat:scim` |
| `GET` | `/scim/v2/Users/{id}` | Lee un usuario. | `feat:scim` |
| `PATCH` | `/scim/v2/Users/{id}` | Aplica un `PatchOp`. | `feat:scim` |
| `DELETE` | `/scim/v2/Users/{id}` | Baja del usuario (**204**). | `feat:scim` |

Los tres endpoints de discovery **no** están gateados por licencia (piden token, pero no capability): así el administrador del IdP puede validar la URL aunque la licencia aún no esté activa. Las cinco operaciones sobre `Users` sí lo están.

Cualquier otra ruta bajo `/scim/v2` —`/Groups`, `/Me`, `/Bulk`— **no existe**: el IdP recibe un **404** con el error genérico del API, no un cuerpo SCIM.

## Cómo se mapean los atributos

Didacta no añade tablas para SCIM: los usuarios que crea el IdP son los mismos que ves en **Administración → Usuarios**.

| Atributo SCIM | Campo en Didacta | Notas |
| --- | --- | --- |
| `userName` | Email | **Obligatorio y tiene que ser un email válido.** Se normaliza a minúsculas. |
| `emails` | Email | Solo se usa como respaldo si `userName` no es un email: primero el marcado `primary`, luego el primero de la lista. En las respuestas siempre se devuelve un único email `type: work`, `primary: true`. |
| `name.givenName` + `name.familyName` | Nombre | Didacta guarda **un solo campo de texto**. Al leer se parte con una heurística: la última palabra es `familyName` y el resto `givenName` (`Juan Carlos Pérez` → `Juan Carlos` + `Pérez`). |
| `name.formatted` | Nombre | Si viene, gana sobre `givenName`/`familyName` al crear. |
| `displayName` | Nombre | Respaldo si no viene `name`. En las respuestas es el nombre, o el email si el usuario no tiene nombre. |
| `active` | Estado | Ver la tabla siguiente. |
| `locale` | Idioma | Texto libre (`es-ES`, `en-GB`…). Si no viene al crear, queda `es-ES`. |
| `externalId` | — | **No se persiste.** En las respuestas se devuelve una copia del `id` interno, para que el IdP pueda guardar un solo identificador. |
| `meta.location` | — | Ruta **relativa** (`/scim/v2/Users/{id}`), no absoluta: la instancia no fuerza un dominio. |
| `password` | — | Se ignora. Didacta no acepta contraseñas por SCIM. |

## `active` y el ciclo de vida del usuario

| Operación | Estado resultante |
| --- | --- |
| `POST` con `active: true` (o sin el campo) | **Pendiente** — el usuario existe pero aún no ha entrado. |
| `POST` con `active: false` | **Desactivado**. |
| `PATCH active: false` | **Desactivado**, desde cualquier estado. |
| `PATCH active: true` sobre un usuario pendiente o desactivado | **Activo**. |
| `PATCH active: true` sobre un usuario **suspendido** | **Sigue suspendido**. |

Al leer, `active` vale `true` para los usuarios activos y pendientes, y `false` para los suspendidos y desactivados.

Tres consecuencias que conviene tener claras antes de la primera sincronización:

- **La suspensión manual gana.** Si un administrador suspende a alguien desde el panel, un `PATCH active: true` del IdP **no** lo rehabilita: hay que levantarlo desde **Administración → Usuarios**. Al revés sí funciona siempre: el IdP puede cortar el acceso de cualquiera.
- **Desactivar cierra la sesión al momento.** Un `PATCH active: false` borra las sesiones vivas del usuario, así que el corte desde el IdP es inmediato y no espera a que caduque el token.
- **Crear un usuario no le manda ningún correo ni le da ningún rol.** El alta por SCIM no envía invitación (se asume que el usuario llegará por SSO) y no asigna roles ni grupos: eso se sigue haciendo desde **Administración → Usuarios**. Un usuario recién aprovisionado existe, pero todavía no puede hacer nada.

`DELETE` hace una **baja lógica**: marca el usuario como borrado, lo deja desactivado y elimina todas sus sesiones. Deja de aparecer en `GET /Users` y su `id` pasa a devolver 404. Los IdPs no suelen usarlo —prefieren `PATCH active: false`—, pero está disponible para peticiones de borrado por RGPD.

## Listar: paginación y filtro

```http
GET /scim/v2/Users?startIndex=1&count=50&filter=userName%20eq%20%22ana@acme.com%22
```

| Parámetro | Valores | Notas |
| --- | --- | --- |
| `startIndex` | Entero **desde 1**, por defecto `1` | Es 1-based, como manda RFC 7644 §3.4.2. Un `0` responde 400. |
| `count` | `0`–`200`, por defecto `50` | Por encima de 200 responde 400. El `0` se acepta (los IdPs lo usan para preguntar solo por el total). |
| `filter` | Solo `userName eq "valor"` | El valor se compara con el email **sin distinguir mayúsculas**. |

Cualquier otro filtro (`displayName co "…"`, `active eq true`, expresiones con `and`…) responde **400** con `scimType: invalidFilter`. Es el único filtro que la mayoría de IdPs necesita: lo usan para comprobar si el usuario ya existe antes de crearlo.

El orden es siempre **por fecha de creación ascendente** y no se puede cambiar: `sort` no está soportado. Los parámetros `attributes` y `excludedAttributes` se ignoran; siempre se devuelve el recurso completo.

## `PATCH`: operaciones soportadas

El cuerpo es un `PatchOp` estándar con un máximo de **20 operaciones** por petición:

```json
{
  "schemas": ["urn:ietf:params:scim:api:messages:2.0:PatchOp"],
  "Operations": [
    { "op": "replace", "path": "active", "value": false }
  ]
}
```

| `path` | `op` admitidos | Valor |
| --- | --- | --- |
| `active` | `replace`, `add`, `remove` | Booleano. `remove` equivale a `false`. |
| `userName` | `replace`, `add` | Un email válido. Si ya lo usa otro usuario de la organización → **409**. `remove` → 400. |
| `displayName` | `replace`, `add`, `remove` | Texto. `remove` borra el nombre. |
| `name.givenName` | `replace`, `add`, `remove` | Texto. Recompone el nombre conservando el apellido actual. |
| `name.familyName` | `replace`, `add`, `remove` | Texto. Recompone el nombre conservando el nombre actual. |
| `locale` | `replace`, `add`, `remove` | Texto. `remove` lo devuelve a `es-ES`. |
| *(sin `path`)* | `replace`, `add` | Un objeto con `active`, `userName`, `locale`, `name` o `displayName`: se aplica como mezcla parcial. `remove` sin `path` → 400. |

Cualquier otra ruta responde **400** con `scimType: invalidPath`. Una petición cuyas operaciones no cambien nada devuelve **200** con el usuario tal cual, sin escribir en la base de datos ni dejar rastro en auditoría.

## Formato de error

Los errores propios de SCIM siguen RFC 7644 §3.12: `status` como cadena y, en los 4xx que lo tienen definido, un `scimType`.

```json
{
  "schemas": ["urn:ietf:params:scim:api:messages:2.0:Error"],
  "status": "409",
  "scimType": "uniqueness",
  "detail": "User with userName \"ana@acme.com\" already exists.",
  "statusCode": 409,
  "message": "Conflict Exception"
}
```

!!! note "Los dos campos de más"
    `statusCode` y `message` **no son SCIM**: los añade el normalizador de errores del API a todas las respuestas. Los campos que exige la especificación están todos, y los clientes SCIM ignoran los atributos que no conocen, así que en la práctica no molestan. Conviene saberlo si estás comparando la respuesta contra el RFC. Las respuestas se sirven además como `application/json`, no como `application/scim+json`.

| Código | `scimType` | Cuándo |
| --- | --- | --- |
| **400** | `invalidFilter` | Un `filter` distinto de `userName eq "…"`. |
| **400** | `invalidPath` | Una ruta de `PatchOp` no soportada. |
| **400** | `invalidSyntax` | `replace` sin `path` cuyo valor no es un objeto. |
| **400** | `invalidValue` | `userName` que no es un email; `active` que no es booleano; `locale` o `displayName` que no son texto. |
| **400** | `mutability` | Intento de eliminar `userName`. |
| **400** | `noTarget` | `remove` sin `path`. |
| **401** | — | Falta el token, no es `Bearer`, está vacío o no se reconoce. |
| **404** | — | El usuario no existe en esta organización (o está dado de baja). |
| **409** | `uniqueness` | Ya hay un usuario con ese `userName`. Es la respuesta a un alta repetida: el IdP debe tratarla como «ya existe», no como error. |

Tres tipos de respuesta **no** llevan el formato SCIM, porque los genera el API antes de llegar al controlador. Si tu IdP los muestra como texto crudo, es esperable:

- **402** (falta `feat:scim`): `{ "statusCode": 402, "error": "CapabilityRequiredError", "capability": "feat:scim", … }`.
- **429** (límite de peticiones): incluye `retryAfterSeconds` y la cabecera `Retry-After`.
- **400 de validación** del cuerpo o de la query (por ejemplo `count=500`): `{ "statusCode": 400, "message": …, "issues": [ … ] }`.

## Qué no está soportado {#que-no-esta-soportado}

| No soportado | Qué pasa si el IdP lo intenta |
| --- | --- |
| **Groups** (`/scim/v2/Groups`) | **404.** No hay sincronización de grupos ni de roles: las pertenencias se siguen gestionando en **Administración → Usuarios**. Desactiva «push groups» en el conector. |
| **`bulk`** (`/scim/v2/Bulk`) | **404.** El IdP debe usar operaciones individuales; los conectores estándar lo hacen por defecto. |
| **`PUT`** sobre un usuario | **404.** El reemplazo total del recurso no existe; usa `PATCH`. |
| **`changePassword`** | Las contraseñas no se gestionan por SCIM. El campo `password` de un alta se ignora en silencio. |
| **`sort`** (`sortBy`, `sortOrder`) | Se ignora: el orden es siempre por fecha de creación ascendente. |
| **`etag`** | No se emiten `ETag` ni se atienden `If-Match`: no hay control de concurrencia optimista. |
| **Filtros complejos** | **400** `invalidFilter`. Solo `userName eq "…"`. |
| **`/Me`** | **404.** |
| **Extensión `enterprise:2.0:User`** | Se acepta en el cuerpo y se ignora: `employeeNumber`, `department`, `manager`… no se guardan. |
| **`externalId` persistido** | Se acepta y se ignora; la respuesta devuelve el `id` interno en su lugar. Si tu IdP correlaciona por `externalId`, funcionará, pero contra el identificador de Didacta. |
| **`attributes` / `excludedAttributes`** | Se ignoran: siempre se devuelve el recurso completo. |
| **Aprovisionamiento inverso** (Didacta → IdP) | No existe. El flujo es siempre del IdP hacia Didacta. |

## Límite de peticiones

Las peticiones SCIM **cuentan contra el cupo público** de la instancia, porque el token SCIM no es una sesión de usuario:

| Plan | Límite |
| --- | --- |
| Community | 30 peticiones/minuto |
| Con `feat:api.rate_limit.elevated` | 300 peticiones/minuto |

Se ajustan con `RATE_LIMIT_COMMUNITY_PUBLIC_PER_MIN` y `RATE_LIMIT_ENTERPRISE_PUBLIC_PER_MIN` (ver [Variables de entorno](../configuracion/variables-de-entorno.md)). Sin Redis configurado, el limitador se abre y no cuenta nada.

!!! warning "La primera sincronización es la que duele"
    `feat:scim` y `feat:api.rate_limit.elevated` son capabilities **distintas**. Con solo la primera, una sincronización inicial de varios cientos de usuarios puede chocar con los 30 req/min y recibir **429**. Los conectores reintentan, pero el alta completa tardará; si vas a migrar un directorio grande, contrata también el límite elevado o súbelo por variable de entorno mientras dure la carga.

## Configurar tu IdP

Los nombres de los campos cambian un poco entre proveedores, pero los pasos son los mismos:

1. **Crea la aplicación SCIM** en el catálogo del IdP: busca «SCIM 2.0» o crea una aplicación propia con aprovisionamiento SCIM activado.
2. **Pega la URL base** en `SCIM Connector base URL` (Okta), `Tenant URL` (Entra ID) o el campo equivalente: `https://<tu-dominio>/scim/v2`.
3. **Pega el token** en `OAuth Bearer Token` o `Secret Token`, y marca el método de autenticación como **Bearer**.
4. **Mapea los atributos**: `userName` al email, `name.givenName` y `name.familyName` al nombre y los apellidos, `active` al estado del usuario. Deja sin mapear todo lo que no aparezca en la tabla de arriba.
5. **Prueba la conexión.** El botón «Test connection» del IdP hace un `GET /scim/v2/ServiceProviderConfig`; si responde 200, ya está.

Y antes de activarlo del todo: **desactiva la sincronización de grupos** en el conector, o el IdP intentará escribir en `/Groups` y acumulará errores.

## Auditoría

Todas las operaciones quedan en el log de auditoría, con prefijo para filtrarlas de un vistazo:

| Acción | Cuándo |
| --- | --- |
| `scim.user.created` | Alta por `POST /Users`. |
| `scim.user.updated` | `PATCH` que cambia algo (los que no cambian nada no se registran). |
| `scim.user.deleted` | `DELETE /Users/{id}`. |
| `scim.token.created` · `scim.token.rotated` · `scim.token.revoked` | Gestión del token desde el panel o el API de administración. |

Las operaciones que llegan por SCIM no tienen actor humano: se registran sin autor, con la marca de que vienen del IdP.

## Solución de problemas

| Síntoma | Causa habitual |
| --- | --- |
| «Test connection» devuelve **401** | El token no está puesto todavía en el IdP (el discovery también lo exige), está mal copiado, o lo rotaste y el conector sigue con el viejo. |
| Todo devuelve **402** | La licencia no incluye `feat:scim`, ha expirado o está en estado inválido. Compruébalo en **Administración → Licencia**. |
| El alta devuelve **409** | Ya existe un usuario con ese email en la organización. Es la respuesta correcta a un alta repetida. |
| El alta devuelve **400** `invalidValue` | `userName` no es un email. Cambia el mapeo del IdP para que envíe el correo electrónico. |
| **429** durante la carga inicial | Cupo público agotado. Ver [Límite de peticiones](#limite-de-peticiones). |
| El IdP acumula errores en grupos | La sincronización de grupos sigue activa en el conector. No está soportada: desactívala. |
| Un usuario reaparece como activo tras suspenderlo | Al revés: los suspendidos **no** se rehabilitan por SCIM. Si vuelve a estar activo, alguien lo reactivó desde el panel. |
| El usuario existe pero no puede entrar | Falta darle rol y grupos: SCIM no los asigna. Hazlo en **Administración → Usuarios**. |
| El panel dice «aún no usado por el IdP» aunque funciona | Ese campo no se actualiza nunca. Mira el log de auditoría. |

## Relacionado

- [Capabilities Enterprise](capabilities.md) — qué desbloquea cada capability.
- [La licencia Enterprise](licencia.md) — cómo se activa `feat:scim`.
- [Autenticación del API](../api/autenticacion.md) — los otros esquemas de credencial del producto.
- [Referencia: núcleo y transversales](../api/referencia/nucleo.md) — la tabla resumida de rutas SCIM junto al resto del API.
