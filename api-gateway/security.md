# Seguridad — API Gateway

Qué hace el Gateway (GW) con la credencial de acceso, cómo propaga la identidad a los microservicios, cómo autentica las llamadas entre servicios y cómo protege los datos sensibles. Los valores transversales están en los [Estándares de Integración](../integration/standards.md) (EST); la configuración, en [contracts.md](contracts.md) (GW-C); y los RF, en [1_requirements.md](1_requirements.md).

## Índice

1. [Pipeline de seguridad](#pipeline-de-seguridad)
2. [Formación del JWT en MS01](#formación-del-jwt-en-ms01)
3. [Validación del JWT](#validación-del-jwt)
4. [Cortes de sesión](#cortes-de-sesión)
5. [Propagación de la identidad por cabeceras](#propagación-de-la-identidad-por-cabeceras)
6. [Claves de servicio](#claves-de-servicio)
7. [CORS](#cors)
8. [mTLS y aislamiento en la nube](#mtls-y-aislamiento-en-la-nube)
9. [Datos sensibles](#datos-sensibles)
10. [Otras medidas](#otras-medidas)

---

## Pipeline de seguridad

Cada petición pasa por la cadena de filtros en este orden ([GW-C §1](contracts.md#1-pipeline-de-una-solicitud)):

| # | Filtro | Qué hace |
|---:|---|---|
| 1 | `CorrelationIdFilter` | Conserva un `X-Correlation-ID` UUID v4 válido y único, o genera uno nuevo |
| 2 | Resolución de ruta | Busca el prefijo más específico; si no hay destino, `404` sin llamar a ningún MS |
| 3 | `GatewayCorsProcessor` y `PreflightRouteFilter` | Aplica la política CORS y responde el preflight sin llamar al MS |
| 4 | `AccessControlFilter` | Resuelve la política de la ruta (pública, protegida o interna) y exige la credencial: JWT o clave de servicio |
| 5 | `IdentityHeadersFilter` | Elimina las cabeceras de identidad y de credenciales del cliente e inyecta la identidad validada |
| 6 | Reenvío | Llama al MS con `GATEWAY_DOWNSTREAM_TIMEOUT_MS`; en la nube, con mTLS |
| 7 | `DownstreamErrorValidationFilter` | Deja pasar un error del MS solo si cumple `ErrorResponse`; si no, `502` |
| 8 | `AccessLogFilter` | Registra un log técnico sin cuerpos ni credenciales |

## Formación del JWT en MS01

El GW no emite tokens: MS01 es el único emisor ([seguridad de MS01](../ms01-identity-and-access/security.md#formación-del-jwt), [jwt-spec.md](../ms01-identity-and-access/jwt-spec.md)). Lo que el GW necesita saber del token que forma MS01:

| Parte | Contenido |
|---|---|
| Header | `alg: RS256`, `typ: JWT` y `kid` (informativo) |
| Claims | `iss` = `ms01-identity-and-access`, `aud` = `campusseguro`, `sub` (UUID del usuario), `email`, `roles` (nombres en orden alfabético), `permissions` (unión de los permisos de sus roles, en orden alfabético), `iat` (hora de la base de datos de MS01, en segundos), `exp` = `iat + 3600` y `jti` |
| Firma | RS256 con la llave privada RSA de MS01 (al menos 2048 bits); el GW la verifica con la llave pública `JWT_PUBLIC_KEY` del mismo par |

## Validación del JWT

En una ruta protegida, sin clave de servicio, el GW exige `Authorization: Bearer <JWT>` y lo valida en local, sin llamar a MS01 ([RF-GW-02](1_requirements.md#rf-gw-02), `JwtValidator` con Nimbus JOSE + JWT):

1. Una sola cabecera `Authorization` (varias: `400 VALIDACION`) con el esquema `Bearer`, sin distinguir mayúsculas.
2. `alg` igual a `RS256`, comprobado antes de verificar la firma: un token con `none`, `HS256` u otro algoritmo se rechaza sin intentar verificarlo.
3. Firma verificada con `JWT_PUBLIC_KEY`, una llave RSA de al menos 2048 bits; el GW no arranca con una llave más corta.
4. `iss` y `aud` iguales a `JWT_ISSUER` y `JWT_AUDIENCE`.
5. `exp`, `iat` y `nbf` (si existe) con la tolerancia `JWT_CLOCK_SKEW_SECONDS` (30 s), y `exp − iat` no mayor que `JWT_MAX_LIFETIME_SECONDS` (3600 s).
6. `sub` es un UUID; `roles` es un arreglo no vacío de nombres `^[A-Z][A-Z0-9_]{1,29}$`; `permissions` es un arreglo no vacío de códigos `^ms[0-9]{2}:[a-z][a-z0-9_]*:[a-z][a-z0-9_]*$`.
7. `iat` no es anterior al corte de sesión del usuario ni al de ninguno de sus roles ([Cortes de sesión](#cortes-de-sesión)).

Cualquier falla responde `401 NO_AUTENTICADO` con el mensaje genérico `Token ausente, inválido o expirado.` y la petición no llega al MS. El motivo concreto solo queda en el log en nivel `DEBUG`, sin el token. El GW no autoriza por rol ni por permiso: cada MS autoriza con `X-User-Permission` ([RN-GW-12](1_requirements.md#rn-gw-12)).

## Cortes de sesión

Para que un cambio de roles, de permisos de un rol o de estado de una cuenta surta efecto antes de que el token venza, el GW consulta a MS01 al arrancar y luego cada `SESSION_CUTS_INTERVAL_SECONDS` (4 s), con un timeout de `SESSION_CUTS_TIMEOUT_SECONDS` (3 s), la ruta interna `GET /api/v1/internal/session-cuts` con `X-Service-Name: gateway` ([RF-GW-02](1_requirements.md#rf-gw-02)). Cada respuesta válida reemplaza la instantánea en memoria (usuario → corte y rol → corte), y el GW rechaza los tokens cuyo `iat` es estrictamente menor que alguno de los cortes que les aplican. Un corte se aplica en 7 segundos como máximo. Si MS01 no responde o responde un cuerpo inválido, el GW conserva la última instantánea, registra un `WARN` y sigue aceptando los tokens válidos. Esa ruta no se publica a través del GW: desde fuera responde `404`.

## Propagación de la identidad por cabeceras

`IdentityHeadersFilter` construye las cabeceras que recibe el MS ([RF-GW-03](1_requirements.md#rf-gw-03)):

| Paso | Cabeceras |
|---|---|
| Elimina, sin distinguir mayúsculas y en todas sus repeticiones | `X-User-Id`, `X-User-Role`, `X-User-Permission`, `X-Service-Name` que envíe el cliente |
| Elimina las credenciales | `Authorization`, `X-Service-Key` |
| Elimina las cabeceras hop-by-hop | `Connection`, `Keep-Alive`, `Proxy-Authenticate`, `Proxy-Authorization`, `TE`, `Trailer`, `Transfer-Encoding`, `Upgrade` |
| Inyecta, con un JWT válido | `X-User-Id` = `sub`, `X-User-Role` = `roles` y `X-User-Permission` = `permissions`, unidos por coma sin espacios |
| Inyecta, con una clave de servicio válida | `X-Service-Name` = nombre del servicio que llama y, si el MS actúa en nombre de un usuario, las tres `X-User-*` que envió, ya validadas |
| Agrega siempre | `X-Correlation-ID` |

Así el MS nunca recibe el token ni la clave de servicio, y la identidad que recibe solo puede venir del JWT validado o de un MS autenticado; nunca de cabeceras, cuerpo, parámetros ni cookies del cliente externo ([RN-GW-10](1_requirements.md#rn-gw-10), [RN-GW-11](1_requirements.md#rn-gw-11)). En una ruta pública no se inyecta identidad.

## Claves de servicio

Un MS que llama a otro lo hace a través del GW con su clave de servicio en `X-Service-Key` ([RF-GW-08](1_requirements.md#rf-gw-08)):

| Regla | Detalle |
|---|---|
| Configuración | Una clave por MS (`GW_SERVICE_KEY_MS01` … `GW_SERVICE_KEY_MS08`), de al menos 32 caracteres y sin repetidas; los nombres solo pueden ser `ms01` … `ms08`. Si no se cumple, el GW no arranca y el error nunca muestra las claves |
| Comparación | El GW guarda el SHA-256 de cada clave y compara el SHA-256 de la clave recibida con `MessageDigest.isEqual`, en tiempo constante, contra todas las configuradas |
| Credenciales mezcladas | `Authorization` junto con `X-Service-Key`, o más de una `X-Service-Key`: `400 VALIDACION` |
| Clave ausente o inválida | En una ruta interna o con `X-Service-Key` presente: `401 NO_AUTENTICADO` |
| Identidad propagada | Si viene alguna `X-User-*`, deben venir las tres, cada una una sola vez y con formato válido; si no, `400 VALIDACION` sin llamar al destino |
| Autorización | El GW no decide qué servicio puede llamar a cuál: el MS destino autoriza por `X-Service-Name` (rutas internas) o por `X-User-Permission` (rutas protegidas) ([RN-GW-21](1_requirements.md#rn-gw-21)) |

Las rutas internas ([GW-C §3](contracts.md#3-políticas-de-acceso)) solo aceptan clave de servicio; las protegidas aceptan JWT o clave de servicio. Las llamadas internas no llevan JWT, así que los cortes de sesión no se les aplican.

## CORS

El GW aplica la única política CORS de CampusSeguro; los MS no configuran CORS ([RF-GW-04](1_requirements.md#rf-gw-04)):

| Elemento | Valor |
|---|---|
| Orígenes | `CORS_ALLOWED_ORIGINS` (obligatoria, sin ruta ni barra final) |
| Métodos | `GET`, `POST`, `PUT`, `PATCH`, `DELETE`, `OPTIONS`, fijos en el código |
| Cabeceras permitidas | `Authorization`, `Content-Type`, `Accept`, `X-Correlation-ID`, fijas en el código |
| Cabecera expuesta | `X-Correlation-ID` |
| Credenciales | `CORS_ALLOW_CREDENTIALS` (`false` por defecto); con `true`, el origen `*` impide el arranque |
| Vigencia del preflight | `CORS_MAX_AGE_SECONDS` (3600 s) |

El preflight lo responde el GW sin JWT y sin llamar al MS. Un origen, método o cabecera no permitidos responden `403 PROHIBIDO`; en particular, el navegador no puede pedir `X-User-Id`, `X-User-Role`, `X-User-Permission` ni `X-Service-Key`. CORS no reemplaza la autenticación.

## mTLS y aislamiento en la nube

Los MS confían en `X-User-*` y `X-Service-Name` sin validar un JWT, así que nadie debe llegar a un MS sin pasar por el GW ([GW-C §7](contracts.md#7-aislamiento-en-la-nube)):

| Tramo | Protección |
|---|---|
| Cliente → GW | HTTPS con certificado de Let's Encrypt en el Caddy de borde de la VM del GW, más el JWT |
| MS → GW | HTTPS más `X-Service-Key` |
| GW → MS | Firewall (NSG o Security Group) que solo acepta la IP del GW, más mTLS: el Caddy del MS exige el certificado de cliente del GW |

* Una CA privada por entorno (prueba y producción) firma el certificado de cliente del GW (solo `clientAuth`) y el de servidor de cada MS (solo `serverAuth`, con la IP de su VM). Un GW de prueba no entra a un MS de producción.
* El perfil `cloud` carga el certificado de cliente con un *SSL bundle* de Spring Boot (`GW_CLIENT_KEYSTORE`) y solo confía en la CA del entorno (`GW_TRUSTED_CA`); si alguno falta o es inválido, el GW no arranca. Un TLS rechazado responde `502 SERVICIO_NO_DISPONIBLE`.
* La consulta de cortes de sesión a MS01 usa la misma conexión con mTLS.
* En local y en Docker, solo se publica el puerto `8080` del GW; los MS quedan en la red interna de Docker.

## Datos sensibles

| Dato | Tratamiento en el GW |
|---|---|
| JWT | Se valida en memoria, nunca se reenvía a un MS y nunca se registra |
| Claves de servicio | Solo se guarda su SHA-256 en memoria; nunca se reenvían ni se registran, ni siquiera en los errores de arranque |
| Cuerpos de petición y de respuesta | No se registran (incluidos login, registro y contraseñas); el log de cliente HTTP (*wiretap*) está desactivado |
| Errores de los MS | Un error que no cumple `ErrorResponse` (o mayor de 64 KiB) se sustituye por `502` con mensaje genérico; el motivo queda en el log, nunca el cuerpo |
| Errores propios | Exactamente los cinco campos de `ErrorResponse`, sin trazas, clases, URIs internas, IPs, nombres de contenedores ni secretos ([RN-GW-14](1_requirements.md#rn-gw-14)) |
| Log técnico | Por petición: `correlation_id`, método, ruta, estado, `duration_ms`, `upstream` y `origen`; sin cabeceras de credenciales ni datos personales |
| Secretos de configuración | `JWT_PUBLIC_KEY`, claves de servicio, certificado de cliente y su contraseña, CA del entorno: variables de entorno y GitHub Environments, nunca en el repositorio; en la VM, archivos con permisos `0600` |

## Otras medidas

* **Política cerrada:** toda ruta que no está en la lista de públicas o internas es protegida ([RN-GW-08](1_requirements.md#rn-gw-08)). Las listas se evalúan por método y ruta exacta y no admiten comodines; el GW no arranca si una los tiene.
* **Sin lógica de dominio:** el GW no accede a bases de datos, no crea usuarios, no emite tokens ni valida contraseñas ([RN-GW-05](1_requirements.md#rn-gw-05), [RN-GW-07](1_requirements.md#rn-gw-07)).
* **Health mínimo:** `GET /actuator/health` responde solo `{"status":"UP"}` o `{"status":"DOWN"}`, sin detalles. Las métricas de Actuator solo responden en el puerto de gestión `MANAGEMENT_SERVER_PORT` (9090), que no se publica.
* **Configuración validada al arrancar:** rutas, políticas, JWT, cortes de sesión, CORS, claves de servicio y, con `cloud`, certificado y CA. Una configuración inválida impide el arranque ([RNF-GW-06](quality-and-tests.md#rnf-gw-06)).
* **Sin reintentos:** el GW no repite peticiones, así que una operación no idempotente nunca se ejecuta dos veces por su causa.
* **Riesgos que quedan:** quien robe el certificado de cliente del GW o la llave de una CA puede hacerse pasar por el GW de ese entorno, y un MS con su clave de servicio puede propagar cualquier `X-User-*`. Por eso los secretos se guardan solo en los GitHub Environments y en archivos `0600` de la VM.
* **Análisis estático:** PMD, SpotBugs y el Quality Gate de SonarCloud revisan el código en cada cambio ([quality-and-tests.md](quality-and-tests.md)).
