# Especificación técnica de la sesión (JWT) — MS01

Detalle técnico de [RF-MS01-03](1_requirements.md#rf-ms01-03-gestión-de-sesión) (credencial de acceso), [RN-MS01-07 a 09](1_requirements.md#rf-ms01-03-gestión-de-sesión), [RN-MS01-23](1_requirements.md#rf-ms01-03-gestión-de-sesión) y [RN-MS01-27](1_requirements.md#rf-ms01-03-gestión-de-sesión), y de [RF-MS01-11](1_requirements.md#rf-ms01-11-corte-de-sesión-por-cambios-de-acceso) (corte de sesión). El contrato que verifica el Gateway (algoritmo, `iss`, `aud`, vigencia, tolerancia de reloj, cabeceras) está en los [Estándares](https://github.com/CampusSeguro-DOSW-2026-2/documentation/wiki/Est%C3%A1ndares-de-Integraci%C3%B3n-y-Contratos-Transversales) y no se repite aquí.

---

## Token emitido

JWT (RFC 7519) firmado con RS256 y una llave RSA de al menos 2048 bits.

**Header**
```json
{ "alg": "RS256", "typ": "JWT", "kid": "ms01-2026-01" }
```

**Payload**
```json
{
  "iss": "ms01-identity-and-access",
  "aud": "campusseguro",
  "sub": "3f6c2a9e-8b1d-4c7a-9e2f-1a5b7c9d0e42",
  "email": "maria.perez@escuelaing.edu.co",
  "roles": ["BRIGADISTA", "ESTUDIANTE"],
  "permissions": [
    "ms01:password:change",
    "ms01:profile:read",
    "ms01:profile:write",
    "ms01:role:read_own",
    "ms01:session:logout",
    "ms02:campus:read",
    "ms02:preventive_info:read",
    "ms02:route:read",
    "ms03:alert:receive",
    "ms04:assistance:request",
    "ms04:checkin:read_own",
    "ms04:checkin:write",
    "ms04:dashboard:read",
    "ms05:assistance:read",
    "ms05:assistance:register",
    "ms05:assistance:update_status",
    "ms05:incident:read",
    "ms05:incident:report",
    "ms05:incident:update",
    "ms05:panel:read_own",
    "ms07:content:read",
    "ms07:quiz:take",
    "ms07:tutor:use"
  ],
  "iat": 1792510000,
  "exp": 1792513600,
  "jti": "5b1f0c2e-8d4a-4c1b-9a77-3f2e6d1a0b9c"
}
```

| Claim | Regla de emisión en MS01 |
|---|---|
| `iss` / `aud` | Fijos en el código (`util/GlobalValues.java`), con los valores de los Estándares. |
| `sub` | UUID del usuario (`usuarios.id`). |
| `roles` | Roles vigentes al momento del login, en orden alfabético. Mínimo 1. Sus nombres son los de la tabla `roles` ([modelo de datos](4_modelo-datos.md#roles)); ya no son un conjunto fijo. |
| `permissions` | Permisos efectivos del usuario: la **unión** de los permisos de todos sus roles, tomados de la tabla `roles` al emitir. Orden alfabético, sin repetidos, mínimo 1. Cada permiso tiene el formato `ms<dos dígitos>:<recurso>:<acción>` (p. ej. `ms04:checkin:write`). Ningún rol hereda de otro. MS01 ignora, con un aviso en el log, los códigos guardados que ya no existen en el catálogo de permisos. |
| `iat` / `exp` | `exp = iat + 3600 s`. La vigencia está fija en el código. |
| `email` | Correo institucional (el Gateway no lo usa). |
| `jti` | UUID aleatorio por token (reservado; el corte de sesión no lo usa, compara `iat`). |

**No incluye** contraseñas, nombres ni estado ([RN-MS01-08](1_requirements.md#rf-ms01-03-gestión-de-sesión)). Los permisos que lleva son los efectivos de sus roles al emitir ([RN-MS01-27](1_requirements.md#rf-ms01-03-gestión-de-sesión)); la [Matriz Rol‐Permiso](https://github.com/CampusSeguro-DOSW-2026-2/documentation/wiki/Matriz-Rol%E2%80%90Permiso) solo describe la asignación inicial, porque después el administrador puede cambiarla.

**Respuesta de login:** `token`, `tipo: Bearer`, `expira_en` (= `exp` en ISO 8601 UTC) y `usuario` (`id`, `correo`, `roles`). No cambia: los permisos viajan dentro del token y el cliente los lee de su payload. Ver [`api/swagger.yml`](api/swagger.yml).

### Qué es cada parte del token

| Parte | Contenido | Para qué sirve |
|---|---|---|
| **Header** | `alg` (RS256), `typ` (JWT) y `kid` (identificador de la llave) | Dice con qué algoritmo y con qué llave se firmó. El Gateway solo acepta RS256. |
| **Payload** | Los claims de la tabla anterior | Es la identidad que el Gateway convierte en cabeceras. `roles` dice quién es (nombres); `permissions` dice qué puede hacer (la unión de los permisos de sus roles). |
| **Firma** | Firma RS256 del header y el payload con la llave privada de MS01 | Garantiza que nadie cambió los roles ni los permisos. Solo el Gateway la verifica, con la llave pública. |

### Cómo se calcula `permissions`

1. Al emitir el token, MS01 lee de la tabla `roles` los permisos de cada rol del usuario.
2. Hace la unión de todas las listas. Si un permiso no está en ninguno de sus roles, no está en el token.
3. Descarta los códigos que ya no existan en el catálogo de permisos (enum `Permission`) y deja un aviso en el log, para que retirar un permiso del código no impida iniciar sesión.
4. Ordena alfabéticamente y quita repetidos.

Ejemplo: un usuario con `PROFESOR` (16 permisos: los 15 de `ESTUDIANTE` más `ms05:incident:report`) y `AUDITOR` (17 permisos, 8 de ellos compartidos con `ESTUDIANTE`) recibe 25 permisos: los 16 de `PROFESOR` más los 9 propios de `AUDITOR` (`ms01:user:read`, `ms03:alert:read_history`, `ms04:dashboard:read`, `ms04:history:read`, `ms05:incident:read`, `ms06:drill:read`, `ms06:drill:compare_report`, `ms08:indicators:read` y `ms08:audit:read`).

**Tamaño:** cada código pesa en promedio unos 23 bytes con su coma. El catálogo completo (48 permisos) ocupa cerca de 1,1 KB en la cabecera `X-User-Permission` y alrededor de 1,5 KB más en el JWT. Los límites habituales de cabeceras (8 KB) no se alcanzan con esos valores (aproximación, no medida).

---

## Identidad en MS01

MS01 nunca recibe ni valida el token. Lee las cabeceras que inyecta el Gateway, con el formato de los Estándares: `X-User-Id`, `X-User-Role` y `X-User-Permission` en los endpoints protegidos, y `X-Service-Name` en los internos ([RF-MS01-09](1_requirements.md#rf-ms01-09-resolución-interna-de-usuarios)), que no exigen identidad de usuario:

| Cabecera | Contenido | Si falta o es inválida |
|---|---|---|
| `X-User-Id` | UUID del usuario (`sub`). | 401 `NO_AUTENTICADO` |
| `X-User-Role` | Roles separados por coma, en orden alfabético (p. ej. `BRIGADISTA,ESTUDIANTE`). MS01 no exige el orden, recorta los espacios e ignora los repetidos. Un nombre con formato inválido (`^[A-Z][A-Z0-9_]{1,29}$`) invalida la cabecera; MS01 no lo compara contra la tabla `roles`. | 401 `NO_AUTENTICADO` |
| `X-User-Permission` | Permisos separados por coma, en orden alfabético y sin espacios (p. ej. `ms01:profile:read,ms04:checkin:write`). Es el contenido del claim `permissions`. MS01 la convierte en un conjunto, recorta los espacios e ignora los repetidos. Un código con formato inválido (`ms<dos dígitos>:<recurso>:<acción>`) invalida la cabecera. | 401 `NO_AUTENTICADO` |
| `X-Service-Name` | Solo en endpoints internos: servicio que llama (`ms01`…`ms08`), derivado por el Gateway de su clave de servicio. Se compara sin distinguir mayúsculas con `INTERNAL_ALLOWED_SERVICES`. La ruta de cortes de sesión solo acepta `gateway`, que fija el propio Gateway cuando llama a MS01. | Falta o vacía: 401 `NO_AUTENTICADO`. Servicio no autorizado: 403 `PROHIBIDO`. |
| `X-Correlation-ID` | UUID v4. Si falta o no es un UUID v4, MS01 genera uno. | Nunca rechaza. |

MS01 autoriza cada operación con los permisos de `X-User-Permission`, que ya son la unión de los de todos los roles del usuario; no los recalcula a partir de los roles ni consulta la base para autorizar. Si falta el permiso exigido, responde 403 `PROHIBIDO`. `X-User-Role` se usa solo donde la respuesta depende de los roles del solicitante, como `GET /roles`. Los endpoints internos no usan permisos: los autoriza el servicio que llama, con la lista `INTERNAL_ALLOWED_SERVICES` (MS03 y MS05) en `/users/batch` y `/users/by-role`, y solo `gateway` en la consulta de cortes de sesión.

MS01 confía en estas cabeceras porque solo el Gateway puede llegar a él ([Estándares §2](https://github.com/CampusSeguro-DOSW-2026-2/documentation/wiki/Est%C3%A1ndares-de-Integraci%C3%B3n-y-Contratos-Transversales)):
* **En local:** su puerto no se publica fuera de la red Docker.
* **En la nube:** el firewall de su VM solo acepta la IP del Gateway, y su Caddy de borde exige el certificado de cliente del Gateway (mTLS). Detalle en [ADR-006](../api-gateway/adr/ADR-006-aislamiento-en-la-nube.md) y en la guía [Conexión al API Gateway](https://github.com/CampusSeguro-DOSW-2026-2/documentation/wiki/Conexi%C3%B3n-al-API-Gateway).

Podría falsificarlas quien entre a la VM o robe el certificado del Gateway.

La ruta interna de cortes de sesión no tiene otra defensa: `X-Service-Name: gateway` es una cabecera más y MS01 no la comprueba con ninguna credencial. Su seguridad depende de este mismo aislamiento de red. Quien llegue a MS01 sin pasar por el Gateway podría presentarse como `gateway` y leer los cortes (identificadores de usuario y de rol con la fecha de su último cambio), igual que podría falsificar las demás cabeceras. Por eso MS01 nunca debe quedar accesible fuera de la red interna.

---

## Ciclo de vida

1. **Emisión:** solo en un `POST /auth/login` exitoso de una cuenta ACTIVA ([RF-MS01-02](1_requirements.md#rf-ms01-02-autenticar-usuario-login)), con los roles y los permisos del momento.
2. **Cambios de roles de un usuario, de permisos de un rol o de estado:** cortan la sesión de los usuarios afectados en pocos segundos ([RN-MS01-09](1_requirements.md#rf-ms01-03-gestión-de-sesión), [RF-MS01-11](1_requirements.md#rf-ms01-11-corte-de-sesión-por-cambios-de-acceso)); el siguiente login emite un token con los datos nuevos. Ver [Corte de sesión](#corte-de-sesión).
3. **Logout, cambio o restablecimiento de contraseña:** no revocan los tokens emitidos; siguen válidos hasta vencer ([RN-MS01-10](1_requirements.md#rf-ms01-04-cerrar-sesión-logout)). La ventana máxima es la vigencia del token.
4. **Token inválido, expirado o cortado:** el Gateway lo rechaza con 401; nunca llega a MS01.

**Por qué así:** el Gateway es el único que verifica el token con la llave pública. Así los microservicios solo leen cabeceras y las reglas de validación cambian en un solo punto. Como los permisos de un rol se pueden cambiar en caliente, esperar a que el token venza (60 min) sería inaceptable; por eso hay un corte de sesión que el Gateway consulta cada pocos segundos, sin lista negra de tokens y sin consultar a MS01 en cada petición.

---

## Corte de sesión

Detalle técnico de [RF-MS01-11](1_requirements.md#rf-ms01-11-corte-de-sesión-por-cambios-de-acceso).

**Dónde se guarda:** en las mismas tablas, como la columna `valido_desde` (`TIMESTAMPTZ`, nula por defecto) de `usuarios` y de `roles` ([modelo de datos](4_modelo-datos.md#tablas)). `NULL` significa "sin corte". No hay tabla propia ni lista de tokens.

**Cuándo se escribe** (siempre "ahora" según la hora de la base de datos, truncada a segundos, en la misma transacción del cambio):

| Evento | Marca que se escribe | A quién corta |
|---|---|---|
| Agregar o retirar roles a un usuario | `usuarios.valido_desde` de cada usuario afectado | Ese usuario |
| Activar o inactivar una cuenta | `usuarios.valido_desde` de cada usuario afectado | Ese usuario |
| Cambiar los permisos de un rol | `roles.valido_desde` del rol | Todos los que tienen ese rol |
| Renombrar un rol | `usuarios.valido_desde` de cada usuario que lo tiene | Los usuarios con ese rol (su token lleva el nombre viejo) |
| Borrar un rol | `usuarios.valido_desde` de cada usuario afectado | Los usuarios con ese rol |
| Crear un rol, cambiar solo su descripción, logout, cambiar o restablecer la contraseña | Ninguna | Nadie |

**Regla en el Gateway:** rechaza con 401 `NO_AUTENTICADO` (mensaje genérico) un token cuyo `iat` sea estrictamente menor que `valido_desde` del usuario (`sub`) o que el de alguno de los roles del claim `roles`. Si son iguales, el token se acepta.

**Por qué se trunca a segundos:** `iat` es un entero en segundos. Con una marca con fracción (23:05:12,700), un usuario que inicia sesión de nuevo en 23:05:12,900 recibiría un token con `iat` 23:05:12 y sería rechazado aunque nació después del cambio. Con la marca truncada (23:05:12) se acepta. El costo es una ventana residual menor a 1 segundo.

**Un solo reloj:** el `iat` del token y la marca `valido_desde` salen de la hora de la base de datos (`floor(extract(epoch FROM clock_timestamp()))`, una consulta por token y por cambio), no del reloj de cada instancia de MS01. Con varias instancias, un desfase entre sus relojes haría que un token emitido después de un cambio llevara un `iat` menor que la marca y se rechazara, o que uno emitido antes se aceptara. Con un solo reloj las dos marcas son siempre comparables. El `exp` parte del mismo `iat`. El Gateway solo compara y usa su propio reloj, con la tolerancia de 30 segundos, para `exp`, `nbf` y un `iat` en el futuro.

**Cómo llega al Gateway:** cada 4 segundos el Gateway llama a una ruta interna de MS01 (`GET /api/v1/internal/session-cuts`, no se publica en el Gateway ni se audita) con `X-Service-Name: gateway`. La ruta solo acepta ese servicio, aparte de la lista `INTERNAL_ALLOWED_SERVICES`, y se protege con el aislamiento de red de [Identidad en MS01](#identidad-en-ms01). La respuesta es la instantánea completa de los cortes de los últimos 3700 segundos (3600 de vigencia + 30 de tolerancia de reloj + 70 de margen):

```json
{
  "generado_en": "2026-11-10T23:05:20Z",
  "ventana_segundos": 3700,
  "usuarios": [
    { "id": "3f6c2a9e-8b1d-4c7a-9e2f-1a5b7c9d0e42", "valido_desde": "2026-11-10T23:04:57Z" }
  ],
  "roles": [
    { "rol": "BRIGADISTA", "valido_desde": "2026-11-10T22:58:03Z" }
  ]
}
```

El Gateway reemplaza por completo sus dos mapas en memoria (usuario → corte y rol → corte) con cada instantánea. Un corte sale solo de la ventana cuando ya venció todo token al que afectaba.

**Si MS01 no responde:** el Gateway conserva la última instantánea, deja un aviso en el log y no bloquea el tráfico. Al arrancar hace una consulta inicial.

**Efecto:** con MS01 respondiendo, un cambio se aplica como máximo 7 segundos después: 4 del intervalo entre consultas y hasta 3 de espera de la consulta. Si MS01 no responde, el Gateway conserva los cortes anteriores y el cambio se aplica cuando vuelva a responder. Al usuario le llega el mismo 401 de un token inválido y debe iniciar sesión de nuevo.

El corte no distingue quién hizo el cambio: si el administrador que cambia los permisos de un rol, lo renombra o lo borra también tiene ese rol, su propia sesión se corta y debe iniciar sesión de nuevo.

---

## Configuración (variables de entorno, nunca en el repositorio)

| Variable | Servicio | Valor por defecto | Descripción |
|---|---|---|---|
| `JWT_PRIVATE_KEY` | MS01 | — (obligatoria fuera de desarrollo) | Llave privada RSA (≥ 2048 bits) en PEM PKCS#8 (`BEGIN PRIVATE KEY`; PKCS#1 se rechaza). Admite saltos de línea escritos como `\n` o solo el contenido Base64, sin las líneas BEGIN/END. Sin ella, MS01 genera un par efímero solo para desarrollo, que el Gateway no puede verificar. |
| `JWT_KEY_ID` | MS01 | `ms01-2026-01` | Valor del `kid` (informativo). |
| `JWT_PUBLIC_KEY` | Gateway | — | Llave pública en PEM del mismo par. |
| `INTERNAL_ALLOWED_SERVICES` | MS01 | `ms03,ms05` | Servicios autorizados para los endpoints internos de consulta de usuarios (`/users/batch` y `/users/by-role`), separados por coma. No aplica a la ruta de cortes de sesión, que solo acepta `gateway`. |
| `SESSION_CUTS_URL` | Gateway | Dirección interna de MS01 + `/api/v1/internal/session-cuts` | A dónde consulta los cortes de sesión. |
| `SESSION_CUTS_INTERVAL_SECONDS` | Gateway | `4` | Cada cuántos segundos consulta. |
| `SESSION_CUTS_TIMEOUT_SECONDS` | Gateway | `3` | Tiempo máximo de espera por consulta. Con el intervalo, fija el retraso máximo de un corte: 7 segundos con los valores por defecto. |

`iss`, `aud` y la vigencia no son configurables en MS01. La ventana de 3700 segundos de la instantánea tampoco: sale de la vigencia, la tolerancia de reloj y un margen fijos.

Generación del par de llaves:
```bash
openssl genpkey -algorithm RSA -pkeyopt rsa_keygen_bits:2048 -out private.pem
openssl rsa -in private.pem -pubout -out public.pem
```

**Cambio de llave (sin rotación en esta versión):** se genera un par nuevo y se despliegan a la vez la llave privada en MS01 y la pública en el Gateway. Los tokens vigentes dejan de ser válidos y los usuarios inician sesión de nuevo.
