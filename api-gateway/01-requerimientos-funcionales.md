# Requerimientos funcionales — API Gateway (GW)

* Los valores transversales vienen de los [Estándares de Integración](https://github.com/CampusSeguro-DOSW-2026-2/documentation/wiki/Est%C3%A1ndares-de-Integraci%C3%B3n-y-Contratos-Transversales) (EST).
* La tabla de rutas, los errores propios y las variables de configuración del GW están en [02-contratos-integracion.md](02-contratos-integracion.md) (GW-C).
* Los RF citan variables (p. ej. `GATEWAY_DOWNSTREAM_TIMEOUT_MS`), no valores, para que ningún número quede duplicado.

## Índice
| RF | Nombre | Prioridad |
|---|---|---|
| [RF-GW-01](#rf-gw-01) | Enrutamiento por contrato | Alta |
| [RF-GW-02](#rf-gw-02) | Validación del JWT y cortes de sesión | Alta |
| [RF-GW-03](#rf-gw-03) | Propagación de identidad y anti-suplantación | Alta |
| [RF-GW-04](#rf-gw-04) | CORS centralizado | Alta |
| [RF-GW-05](#rf-gw-05) | Normalización de errores | Alta |
| [RF-GW-06](#rf-gw-06) | Correlación de solicitudes | Alta |
| [RF-GW-07](#rf-gw-07) | Logging técnico y healthcheck | Alta |
| [RF-GW-08](#rf-gw-08) | Llamadas internas entre servicios | Alta |

---

<a id="rf-gw-01"></a>
### RF-GW-01 Enrutamiento por contrato

| Atributo | Detalle |
|---|---|
| **Descripción** | El GW es el único punto de entrada HTTP, tanto externo como entre servicios. Selecciona el MS destino con la tabla de rutas de GW-C y reenvía la solicitud sin alterar su semántica. |
| **Actor(es)** | Frontend, clientes HTTP externos y MS01–MS08 (como llamantes, RF-GW-08). |
| **Precondiciones** | La tabla de rutas y los destinos `MSnn_URI` están en configuración externa. Solo se incluyen rutas publicadas en el OpenAPI del MS. |
| **Flujo base** | 1. Asigna la correlación (RF-GW-06).<br>2. Busca la ruta con el prefijo más específico.<br>3. Aplica CORS (RF-GW-04).<br>4. Determina la política de acceso y la valida (RF-GW-02 o RF-GW-08).<br>5. Prepara las cabeceras (RF-GW-03).<br>6. Reenvía al destino con el timeout `GATEWAY_DOWNSTREAM_TIMEOUT_MS`.<br>7. Devuelve la respuesta (RF-GW-05).<br>8. Registra el log (RF-GW-07). |
| **Flujos alternos** | **A1, ruta sin destino:** `404 NO_ENCONTRADO`, sin llamar a ningún MS.<br>**A2, conexión rechazada, DNS no resuelto, conexión cortada por el MS o TLS rechazado (mTLS en la nube, [ADR-006](adr/ADR-006-aislamiento-en-la-nube.md)):** `502 SERVICIO_NO_DISPONIBLE`.<br>**A3, timeout:** `504 TIEMPO_AGOTADO`; se liberan los recursos de la llamada.<br>**A4, el cliente se desconecta:** se cancela la llamada downstream y se registra el evento; el log de acceso queda con `status` `499`.<br>**A5, configuración de rutas inválida** (tabla vacía, id o prefijo repetido, prefijo fuera de `/api/v1/`, destino ausente o que no sea una URI `http(s)` sin ruta ni query, timeout ≤ 0, o una ruta pública o interna con comodines): el GW no arranca y el log indica la causa. |
| **Reglas de negocio** | <a id="rn-gw-01"></a>**RN-GW-01:** se preservan el método, la ruta completa (incluido `/api/v1`), los query params y el cuerpo. Solo cambian las cabeceras técnicas de EST.<br><a id="rn-gw-02"></a>**RN-GW-02:** las rutas y los destinos se configuran fuera del código y se validan al arrancar. Cambiarlos exige reiniciar, no recompilar.<br><a id="rn-gw-03"></a>**RN-GW-03:** solo se enrutan rutas publicadas en un `swagger.yml` del repositorio documentation. Se prohíbe inventar prefijos.<br><a id="rn-gw-04"></a>**RN-GW-04:** el GW no reintenta solicitudes.<br><a id="rn-gw-05"></a>**RN-GW-05:** el GW no ejecuta lógica de dominio, no accede a bases de datos ni importa DTOs o librerías de los MS, y no restringe métodos HTTP (un `405` lo responde el MS). |
| **Postcondiciones** | El cliente nunca conoce las URI, IPs ni puertos internos. |
| **Criterios de aceptación** | **CA-GW-01-01:** `POST /api/v1/auth/login` llega a MS01 con el método, la ruta, el cuerpo y los query params sin cambios.<br>**CA-GW-01-02:** una ruta no configurada devuelve `404 NO_ENCONTRADO` sin invocar a un MS.<br>**CA-GW-01-03:** un destino caído devuelve `502` y un destino que demora más que `GATEWAY_DOWNSTREAM_TIMEOUT_MS` devuelve `504`.<br>**CA-GW-01-04:** ninguna respuesta expone URI internas, IPs, nombres de contenedores ni clases.<br>**CA-GW-01-05:** una configuración con prefijos duplicados, destino ausente o URI inválida impide el arranque y el log indica la causa.<br>**CA-GW-01-06:** `/api/v1/roles`, `/api/v1/roles/**` y `/api/v1/permissions` llegan a MS01 con el método, la ruta y el cuerpo sin cambios; `/api/v1/rolesx` y `/api/v1/permissions/ms01` no tienen destino y devuelven `404`.<br>**CA-GW-01-07:** `GET /api/v1/internal/session-cuts` no se enruta: devuelve `404 NO_ENCONTRADO` sin llamar a MS01, con o sin clave de servicio. |
| **Requerimientos asociados** | RF-GW-02…08, RNF-GW-01, RNF-GW-02, RNF-GW-06. |
| **Estado** | Implementado / Equipo API Gateway |

<a id="rf-gw-02"></a>
### RF-GW-02 Validación del JWT y cortes de sesión

| Atributo | Detalle |
|---|---|
| **Descripción** | En las rutas protegidas, el GW exige y valida el JWT emitido por MS01, según EST. La firma y los claims se verifican localmente y los cortes de sesión que publica MS01 se leen de una instantánea en memoria que el GW actualiza en segundo plano, de modo que validar un token no genera ninguna llamada de red. |
| **Actor(es)** | Usuario autenticado; MS01 como emisor del JWT y fuente de los cortes de sesión. |
| **Precondiciones** | `JWT_PUBLIC_KEY`, `JWT_ISSUER`, `JWT_AUDIENCE`, `JWT_CLOCK_SKEW_SECONDS` y `JWT_MAX_LIFETIME_SECONDS` configurados, y `SESSION_CUTS_URL`, `SESSION_CUTS_INTERVAL_SECONDS` y `SESSION_CUTS_TIMEOUT_SECONDS` para la consulta de cortes. |
| **Flujo base** | 1. Extrae `Authorization: Bearer <JWT>`.<br>2. Exige `alg = RS256` y verifica la firma con `JWT_PUBLIC_KEY`.<br>3. Valida `iss`, `aud`, `exp`, `iat` y `nbf` (si existe) con la tolerancia `JWT_CLOCK_SKEW_SECONDS`.<br>4. Valida que `sub` sea UUID, que `roles` sea un arreglo no vacío con elementos `^[A-Z][A-Z0-9_]{1,29}$`, que `permissions` sea un arreglo no vacío con elementos `^ms[0-9]{2}:[a-z][a-z0-9_]*:[a-z][a-z0-9_]*$` y que `exp − iat ≤ JWT_MAX_LIFETIME_SECONDS`.<br>5. Compara `iat` con la instantánea de cortes: rechaza el token si `iat` es estrictamente menor que el corte del usuario (`sub`) o el de alguno de sus `roles`.<br>6. Continúa con RF-GW-03.<br>**Consulta de cortes (en segundo plano, independiente de las solicitudes):**<br>a. Al arrancar y luego cada `SESSION_CUTS_INTERVAL_SECONDS`, llama a `GET SESSION_CUTS_URL` con `X-Service-Name: gateway`, un `X-Correlation-ID` propio y el timeout `SESSION_CUTS_TIMEOUT_SECONDS`.<br>b. Cada respuesta válida (`generado_en`, `ventana_segundos`, `usuarios` con `id` y `valido_desde`, y `roles` con `rol` y `valido_desde`) reemplaza por completo la instantánea anterior. |
| **Flujos alternos** | **A1, falta `Authorization` o el esquema no es `Bearer` (sin distinguir mayúsculas):** `401 NO_AUTENTICADO`.<br>**A2, token mal formado, expirado, con firma inválida, algoritmo distinto de RS256 (incluido `none`), `iss`/`aud` incorrectos o claims inválidos (entre ellos `roles` o `permissions` ausentes, vacíos o con elementos de formato inválido):** `401 NO_AUTENTICADO`, sin llamar al MS.<br>**A3, más de una cabecera `Authorization`:** `400 VALIDACION`.<br>**A4, ruta pública:** no se exige ni se valida el token.<br>**A5, token con `iat` anterior al corte del usuario o al de alguno de sus roles:** `401 NO_AUTENTICADO` con el mismo mensaje genérico de cualquier token inválido, sin llamar al MS; el motivo queda solo en el log (`DEBUG`).<br>**A6, MS01 no responde a la consulta de cortes, responde un error o un cuerpo inválido:** el GW conserva la última instantánea, registra un aviso (`WARN`) con el motivo, sigue aceptando los tokens válidos y reintenta en el siguiente ciclo. Si falla la consulta inicial, el GW arranca con una instantánea vacía. |
| **Reglas de negocio** | <a id="rn-gw-06"></a>**RN-GW-06:** validar un token no hace ninguna llamada de red a MS01 ni introspección: la firma y los claims se verifican con `JWT_PUBLIC_KEY` y los cortes se leen de la instantánea en memoria. Un token válido se acepta hasta su `exp`, salvo que un corte de sesión lo invalide (A5). El logout y el cambio de contraseña no cortan la sesión.<br><a id="rn-gw-07"></a>**RN-GW-07:** el GW no crea usuarios, no genera ni renueva tokens y no valida contraseñas.<br><a id="rn-gw-08"></a>**RN-GW-08:** toda ruta no declarada como pública o interna en GW-C es protegida. Agregar una ruta pública exige un cambio en EST por PR.<br><a id="rn-gw-09"></a>**RN-GW-09:** el GW valida el formato de los roles y de los permisos, pero no mantiene un catálogo ni una matriz: son datos de MS01 y la autorización es de cada MS con `X-User-Permission`.<br><a id="rn-gw-22"></a>**RN-GW-22:** un corte se aplica en el GW a más tardar un ciclo de consulta después de registrarse en MS01 (`SESSION_CUTS_INTERVAL_SECONDS` más `SESSION_CUTS_TIMEOUT_SECONDS`: 4 s más 3 s, 7 s con los valores por defecto). La instantánea cubre los últimos `ventana_segundos` (3700 s), más que la vigencia máxima de un token. Un token emitido en el mismo segundo del corte, o después, se acepta. El `iat` del token y el corte salen de la hora de la base de datos de MS01, un solo reloj para todas sus instancias: el GW solo los compara y no depende de que esas instancias estén sincronizadas. Su propio reloj, con `JWT_CLOCK_SKEW_SECONDS`, solo interviene en `exp`, `nbf` y un `iat` en el futuro.<br><a id="rn-gw-23"></a>**RN-GW-23:** los cortes no se aplican a las llamadas internas con clave de servicio (RF-GW-08), que no llevan JWT. La consulta de cortes es una llamada del propio GW a MS01: no usa clave de servicio, viaja por la misma conexión que las rutas hacia MS01 (con mTLS en la nube, [ADR-006](adr/ADR-006-aislamiento-en-la-nube.md)) y su ruta no se publica a través del GW (CA-GW-01-07). |
| **Postcondiciones** | Las solicitudes protegidas rechazadas no consumen recursos de los MS. |
| **Criterios de aceptación** | **CA-GW-02-01:** un JWT válido emitido por MS01 permite la solicitud.<br>**CA-GW-02-02:** sin `Authorization`, la respuesta es `401` y el MS no recibe la solicitud.<br>**CA-GW-02-03:** un token expirado, alterado, firmado con otra llave, con `alg: none`/`HS256`, o con `iss`/`aud` incorrectos devuelve `401`.<br>**CA-GW-02-04:** un token con `sub` que no es UUID, `roles` o `permissions` ausente, vacío o con elementos de formato inválido, o vigencia mayor a `JWT_MAX_LIFETIME_SECONDS` devuelve `401`.<br>**CA-GW-02-05:** las cuatro rutas públicas de MS01 funcionan sin JWT.<br>**CA-GW-02-06:** validar un JWT no genera ninguna llamada de red a MS01.<br>**CA-GW-02-07:** un token emitido antes del corte del usuario, o antes del corte de alguno de sus roles, devuelve `401 NO_AUTENTICADO` con el mensaje genérico y el MS no recibe la solicitud; uno emitido en el mismo segundo del corte o después se acepta.<br>**CA-GW-02-08:** tras un cambio de roles, de permisos de un rol o de estado de una cuenta en MS01, el GW rechaza los tokens anteriores en el siguiente ciclo de consulta (`SESSION_CUTS_INTERVAL_SECONDS` más `SESSION_CUTS_TIMEOUT_SECONDS` como máximo: 7 s con los valores por defecto).<br>**CA-GW-02-09:** con MS01 sin responder, el GW sigue aceptando los tokens válidos con los últimos cortes, registra un `WARN` por cada consulta fallida y vuelve a aplicar los cortes nuevos en la primera consulta que MS01 responde. |
| **Requerimientos asociados** | RF-GW-01, RF-GW-03, RNF-GW-03; RF-MS01-03 (emisión del JWT), RF-MS01-11 (cortes de sesión). |
| **Estado** | Implementado / Equipo API Gateway, en coordinación con MS01 |

<a id="rf-gw-03"></a>
### RF-GW-03 Propagación de identidad y anti-suplantación

| Atributo | Detalle |
|---|---|
| **Descripción** | El GW construye las cabeceras técnicas de EST hacia el MS. La identidad solo puede venir del JWT validado o de un MS autenticado con su clave de servicio (RF-GW-08), nunca del cliente externo. |
| **Actor(es)** | Cliente, GW y MS01–MS08. |
| **Precondiciones** | RF-GW-02 o RF-GW-08 se cumplieron, o la ruta es pública. |
| **Flujo base** | 1. Elimina, sin distinguir mayúsculas/minúsculas, todo `X-User-Id`, `X-User-Role`, `X-User-Permission` y `X-Service-Name` del cliente externo.<br>2. Elimina `Authorization` y `X-Service-Key`.<br>3. En rutas protegidas con JWT, inyecta `X-User-Id` = `sub`, `X-User-Role` = `roles` y `X-User-Permission` = `permissions`, los dos últimos unidos por coma, sin espacios, en el orden del token (MS01 los emite en orden alfabético).<br>4. Elimina las cabeceras hop-by-hop (`Connection`, `Keep-Alive`, `Proxy-Authenticate`, `Proxy-Authorization`, `TE`, `Trailer`, `Transfer-Encoding`, `Upgrade`).<br>5. Agrega `X-Correlation-ID` (RF-GW-06). |
| **Flujos alternos** | **A1, ruta pública:** se ejecutan los pasos 1, 2, 4 y 5; no se inyecta identidad.<br>**A2, cabeceras de identidad duplicadas o con otras mayúsculas:** se eliminan todas y queda una sola versión canónica. |
| **Reglas de negocio** | <a id="rn-gw-10"></a>**RN-GW-10:** la identidad nunca se toma del cuerpo, los query params, las cookies ni de cabeceras del cliente externo.<br><a id="rn-gw-11"></a>**RN-GW-11:** el JWT nunca se reenvía a un MS ni se registra en logs.<br><a id="rn-gw-12"></a>**RN-GW-12:** el GW no autoriza, ni por rol ni por permiso; cada MS autoriza con `X-User-Permission`. |
| **Postcondiciones** | El MS recibe una identidad confiable sin conocer el token. |
| **Criterios de aceptación** | **CA-GW-03-01:** un token con `sub = 3f6c2a9e-…`, `roles = [BRIGADISTA, ESTUDIANTE]` y `permissions = [ms04:checkin:write, ms07:content:read]` produce `X-User-Id: 3f6c2a9e-…`, `X-User-Role: BRIGADISTA,ESTUDIANTE` y `X-User-Permission: ms04:checkin:write,ms07:content:read`.<br>**CA-GW-03-02:** si el cliente envía `X-User-Role: ADMIN` o `X-User-Permission: ms01:role:manage` con un token de `ESTUDIANTE`, el MS recibe solo los roles y los permisos del token.<br>**CA-GW-03-03:** el MS nunca recibe `Authorization` ni `X-Service-Key`.<br>**CA-GW-03-04:** una ruta pública no recibe `X-User-Id`, `X-User-Role` ni `X-User-Permission`, aunque el cliente los envíe.<br>**CA-GW-03-05:** un `X-Service-Name` enviado por un cliente externo nunca llega al MS. |
| **Requerimientos asociados** | RF-GW-02, RF-GW-06, RF-GW-08. |
| **Estado** | Implementado / Equipo API Gateway |

<a id="rf-gw-04"></a>
### RF-GW-04 CORS centralizado

| Atributo | Detalle |
|---|---|
| **Descripción** | El GW aplica la única política CORS y resuelve el preflight sin JWT y sin llamar al MS. |
| **Actor(es)** | Navegador del frontend. |
| **Precondiciones** | `CORS_ALLOWED_ORIGINS`, `CORS_ALLOW_CREDENTIALS` y `CORS_MAX_AGE_SECONDS` configurados. |
| **Flujo base** | 1. Si la solicitud tiene `Origin` permitido y es un preflight `OPTIONS` válido (método y cabeceras permitidos) a una ruta existente, responde `200` sin cuerpo, con las cabeceras CORS y `Vary: Origin`.<br>2. Si es una solicitud real con un `Origin` permitido, agrega `Access-Control-Allow-Origin` y expone `X-Correlation-ID`. |
| **Flujos alternos** | **A1, origen, método o cabeceras no permitidos en el preflight:** `403 PROHIBIDO`.<br>**A2, solicitud real con origen no permitido:** se procesa sin cabeceras CORS (el navegador bloquea la lectura).<br>**A3, sin `Origin`:** se procesa sin cabeceras CORS.<br>**A4, preflight a una ruta inexistente:** `404 NO_ENCONTRADO`.<br>**A5, credenciales habilitadas con origen `*`:** el GW no arranca. |
| **Reglas de negocio** | <a id="rn-gw-13"></a>**RN-GW-13:** CORS no sustituye la autenticación. `X-Service-Key`, `X-User-Id`, `X-User-Role` y `X-User-Permission` no son cabeceras permitidas para el navegador. |
| **Criterios de aceptación** | **CA-GW-04-01:** un preflight desde un origen permitido responde `200` con `Access-Control-Allow-Origin`, `-Methods` y `-Headers`, sin llamar al MS.<br>**CA-GW-04-02:** un preflight desde un origen no permitido, con un método no permitido o que pide una cabecera interna (`X-User-Role`, `X-User-Permission`, `X-Service-Key`) responde `403` sin `Access-Control-Allow-Origin`.<br>**CA-GW-04-03:** una solicitud real permitida expone `X-Correlation-ID`.<br>**CA-GW-04-04:** la configuración `*` con credenciales impide el arranque. |
| **Requerimientos asociados** | RF-GW-01, RF-GW-05. |
| **Estado** | Implementado / Equipo API Gateway |

<a id="rf-gw-05"></a>
### RF-GW-05 Normalización de errores

| Atributo | Detalle |
|---|---|
| **Descripción** | Todo error del GW usa el `ErrorResponse` de EST con los códigos de GW-C. Los errores de los MS se propagan solo si cumplen el contrato; si no, se sustituyen por `502`. |
| **Actor(es)** | Clientes y MS downstream. |
| **Precondiciones** | La correlación ya está asignada (RF-GW-06). |
| **Flujo base** | 1. Las respuestas `2xx`/`3xx` del MS se devuelven sin cambios.<br>2. Ante una respuesta `4xx`/`5xx` del MS, se verifica que sea JSON con los cinco campos, que `status` sea igual al código HTTP y que `correlation_id` sea igual al de la solicitud.<br>3. Si es válida, se devuelve sin cambios.<br>4. Los errores propios del GW se construyen con `Content-Type: application/json` y `X-Correlation-ID`. |
| **Flujos alternos** | **A1, error del MS no conforme** (cuerpo no JSON o mayor de 64 KiB, campos faltantes o adicionales, `codigo` o `mensaje` que no son texto, `errores` que no es una lista, `status` distinto, `correlation_id` distinto): `502 SERVICIO_NO_DISPONIBLE` con mensaje genérico; el log registra el motivo, nunca el cuerpo.<br>**A2, excepción no controlada en el GW:** `500 ERROR_INTERNO`; el stack trace solo va al log.<br>**A3, respuesta ya iniciada:** se registra el evento y se cierra la conexión. |
| **Reglas de negocio** | <a id="rn-gw-14"></a>**RN-GW-14:** un error del GW tiene exactamente los cinco campos de EST y nunca expone stack traces, clases, rutas internas, IPs, contenedores ni secretos.<br><a id="rn-gw-15"></a>**RN-GW-15:** los errores de negocio válidos de un MS (`400`, `401`, `403`, `404`, `405`, `409`, `415`, `422`, `5xx`) llegan al cliente sin cambios. |
| **Criterios de aceptación** | **CA-GW-05-01:** un error de autenticación del GW contiene los cinco campos y `codigo: NO_AUTENTICADO`.<br>**CA-GW-05-02:** un `409 CONFLICTO` válido de MS01 llega sin cambios en el estado ni en el cuerpo.<br>**CA-GW-05-03:** un `405` del MS con un `ErrorResponse` válido llega sin cambios; un `405` con el cuerpo por defecto de Spring se convierte en `502`.<br>**CA-GW-05-04:** un error del MS sin `correlation_id`, o con otro valor, se convierte en `502`.<br>**CA-GW-05-05:** el `correlation_id` de un error del GW es igual a su cabecera `X-Correlation-ID`. |
| **Requerimientos asociados** | RF-GW-01, RF-GW-06. |
| **Estado** | Implementado / Equipo API Gateway |

<a id="rf-gw-06"></a>
### RF-GW-06 Correlación de solicitudes

| Atributo | Detalle |
|---|---|
| **Descripción** | Toda solicitud y toda respuesta del GW llevan un `X-Correlation-ID` UUID v4. |
| **Actor(es)** | Clientes, GW y MS01–MS08. |
| **Precondiciones** | El filtro de correlación se ejecuta antes que cualquier otro. |
| **Flujo base** | 1. Lee `X-Correlation-ID` sin distinguir mayúsculas/minúsculas.<br>2. Si es un UUID v4 válido y único, lo conserva; si no, genera uno nuevo.<br>3. Lo guarda en el contexto, lo envía al MS, lo usa en logs y errores y lo devuelve al cliente. |
| **Flujos alternos** | **A1, ausente, vacío, inválido o duplicado:** se genera un UUID v4 nuevo; la solicitud nunca se rechaza por la correlación.<br>**A2, error antes de llamar al MS:** el error lleva el identificador del contexto. |
| **Reglas de negocio** | <a id="rn-gw-16"></a>**RN-GW-16:** la cabecera `X-Correlation-ID` de la respuesta siempre es la del contexto del GW. Un MS que devuelve otro valor en el cuerpo de un error incumple el contrato (RF-GW-05, A1). |
| **Criterios de aceptación** | **CA-GW-06-01:** sin cabecera, se genera un UUID v4 que llega al MS y vuelve al cliente.<br>**CA-GW-06-02:** un UUID v4 válido se conserva de extremo a extremo.<br>**CA-GW-06-03:** un valor inválido o duplicado se reemplaza sin rechazar la solicitud. |
| **Requerimientos asociados** | RF-GW-05, RF-GW-07. |
| **Estado** | Implementado / Equipo API Gateway |

<a id="rf-gw-07"></a>
### RF-GW-07 Logging técnico y healthcheck

| Atributo | Detalle |
|---|---|
| **Descripción** | El GW registra un log estructurado por solicitud y expone su propio healthcheck. No audita. |
| **Actor(es)** | DevOps, monitoreo y soporte. |
| **Precondiciones** | Spring Boot Actuator habilitado; puerto de gestión `MANAGEMENT_SERVER_PORT` interno. |
| **Flujo base** | 1. Al terminar cada solicitud, registra los campos mínimos de EST, con el nivel según el estado.<br>2. `GET /actuator/health` en el puerto `SERVER_PORT` responde `200 {"status":"UP"}` (o `503 {"status":"DOWN"}` si el propio GW no está sano), sin detalles y sin consultar a otros servicios.<br>3. Las métricas y el resto de endpoints de Actuator solo responden en `MANAGEMENT_SERVER_PORT`. |
| **Flujos alternos** | **A1, login o registro:** nunca se registra el cuerpo.<br>**A2, falla el sistema de logs:** la respuesta no cambia y no se bloquea la solicitud.<br>**A3, un MS está caído:** el healthcheck del GW sigue en `UP`; reporta solo su propio estado.<br>**A4, MS01 no responde a la consulta de cortes de sesión (RF-GW-02, A6):** el aviso queda en el log técnico (`WARN`); el healthcheck sigue en `UP` y las solicitudes no se bloquean. |
| **Reglas de negocio** | <a id="rn-gw-17"></a>**RN-GW-17:** `duration_ms` se mide desde que la solicitud entra al GW hasta que termina la respuesta.<br><a id="rn-gw-18"></a>**RN-GW-18:** el GW no envía eventos a MS08 ni depende de él. |
| **Criterios de aceptación** | **CA-GW-07-01:** una solicitud genera un log con todos los campos mínimos de EST, incluidos `upstream` y `origen`.<br>**CA-GW-07-02:** los logs de login y registro no contienen contraseña, token, `Authorization` ni `X-Service-Key`.<br>**CA-GW-07-03:** `GET /actuator/health` responde `200 {"status":"UP"}` sin JWT en `≤ 200 ms`.<br>**CA-GW-07-04:** `GET /actuator/metrics` en el puerto `SERVER_PORT` responde `404`. |
| **Requerimientos asociados** | RF-GW-06, RNF-GW-04. |
| **Estado** | Implementado / Equipo API Gateway |

<a id="rf-gw-08"></a>
### RF-GW-08 Llamadas internas entre servicios

| Atributo | Detalle |
|---|---|
| **Descripción** | Un MS que llama a otro MS lo hace a través del GW y se autentica con su clave de servicio. El GW la valida, identifica al llamante y reenvía la identidad del usuario original. |
| **Actor(es)** | MS01–MS08 como llamantes. |
| **Precondiciones** | La clave `GW_SERVICE_KEY_MSnn` del MS llamante está configurada (solo son obligatorias las de los MS que llaman). La llamada está registrada en EST. |
| **Flujo base** | 1. Recibe `X-Service-Key` y, si el MS actúa en nombre de un usuario, `X-User-Id`, `X-User-Role` y `X-User-Permission`.<br>2. Compara la clave en tiempo constante contra las configuradas y obtiene el nombre del llamante (`ms01`…`ms08`).<br>3. Si viene alguna de las tres cabeceras de identidad, exige que vengan las tres, cada una una sola vez, y valida su formato: `X-User-Id` UUID; `X-User-Role` con roles `^[A-Z][A-Z0-9_]{1,29}$` y `X-User-Permission` con permisos `^ms[0-9]{2}:[a-z][a-z0-9_]*:[a-z][a-z0-9_]*$`, ambas separadas por coma.<br>4. Elimina `X-Service-Key`, inyecta `X-Service-Name` y conserva `X-User-Id`, `X-User-Role` y `X-User-Permission`.<br>5. Continúa con el enrutamiento (RF-GW-01). |
| **Flujos alternos** | **A1, clave inválida:** `401 NO_AUTENTICADO`.<br>**A2, `Authorization` y `X-Service-Key` en la misma solicitud, o más de una `X-Service-Key`:** `400 VALIDACION`.<br>**A3, solo algunas de las tres cabeceras de identidad, alguna repetida o con formato inválido:** `400 VALIDACION`, sin llamar al destino.<br>**A4, ruta interna sin `X-Service-Key`:** `401 NO_AUTENTICADO`. |
| **Reglas de negocio** | <a id="rn-gw-19"></a>**RN-GW-19:** una ruta **interna** solo acepta clave de servicio; una ruta **protegida** acepta JWT o clave de servicio.<br><a id="rn-gw-20"></a>**RN-GW-20:** las claves nunca se registran en logs ni se reenvían. Cada MS tiene su propia clave, de al menos 32 caracteres. Al arrancar se valida que los nombres sean `ms01`…`ms08`, que cada clave tenga al menos 32 caracteres y que no haya claves repetidas; si no, el GW no arranca y el error nunca muestra las claves.<br><a id="rn-gw-21"></a>**RN-GW-21:** el GW no decide qué servicio puede llamar a cuál; el MS destino autoriza por `X-Service-Name` (ruta interna) o por `X-User-Permission` (ruta protegida). |
| **Postcondiciones** | La llamada interna queda en el log con `origen = msNN` y la misma correlación. |
| **Criterios de aceptación** | **CA-GW-08-01:** MS04 con una clave válida y las tres `X-User-*` de un ESTUDIANTE llega a MS05 con `X-Service-Name: ms04` y la misma identidad.<br>**CA-GW-08-02:** una clave inválida devuelve `401` sin llamar al destino.<br>**CA-GW-08-03:** un cliente externo sin clave recibe `401` en una ruta interna con destino (p. ej. `POST /api/v1/users/batch`).<br>**CA-GW-08-04:** el MS destino nunca recibe `X-Service-Key`.<br>**CA-GW-08-05:** una llamada interna con `X-User-Id` pero sin `X-User-Permission`, o con cualquiera de las tres cabeceras repetida o con formato inválido, devuelve `400` sin llamar al destino. |
| **Requerimientos asociados** | RF-GW-01, RF-GW-03, RF-GW-07. |
| **Estado** | Implementado / Equipo API Gateway |
