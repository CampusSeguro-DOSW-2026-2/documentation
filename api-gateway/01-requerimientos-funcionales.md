# Requerimientos funcionales — API Gateway (GW)

* Los valores transversales vienen de los [Estándares de Integración](https://github.com/CampusSeguro-DOSW-2026-2/documentation/wiki/Est%C3%A1ndares-de-Integraci%C3%B3n-y-Contratos-Transversales) (EST).
* La tabla de rutas, los errores propios y las variables de configuración del GW están en [02-contratos-integracion.md](02-contratos-integracion.md) (GW-C).
* Los RF citan variables (p. ej. `GATEWAY_DOWNSTREAM_TIMEOUT_MS`), no valores, para que ningún número quede duplicado.

## Índice
| RF | Nombre | Prioridad |
|---|---|---|
| [RF-GW-01](#rf-gw-01) | Enrutamiento por contrato | Alta |
| [RF-GW-02](#rf-gw-02) | Validación del JWT | Alta |
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
| **Flujos alternos** | **A1, ruta sin destino:** `404 NO_ENCONTRADO`, sin llamar a ningún MS.<br>**A2, conexión rechazada o DNS no resuelto:** `502 SERVICIO_NO_DISPONIBLE`.<br>**A3, timeout:** `504 TIEMPO_AGOTADO`; se liberan los recursos de la llamada.<br>**A4, el cliente se desconecta:** se cancela la llamada downstream y se registra el evento.<br>**A5, configuración de rutas inválida** (destino ausente, URI mal formada, prefijos duplicados): el GW no arranca y el log indica la causa. |
| **Reglas de negocio** | <a id="rn-gw-01"></a>**RN-GW-01:** se preservan el método, la ruta completa (incluido `/api/v1`), los query params y el cuerpo. Solo cambian las cabeceras técnicas de EST.<br><a id="rn-gw-02"></a>**RN-GW-02:** las rutas y los destinos se configuran fuera del código y se validan al arrancar. Cambiarlos exige reiniciar, no recompilar.<br><a id="rn-gw-03"></a>**RN-GW-03:** solo se enrutan rutas publicadas en un `swagger.yml` del repositorio documentation. Se prohíbe inventar prefijos.<br><a id="rn-gw-04"></a>**RN-GW-04:** el GW no reintenta solicitudes.<br><a id="rn-gw-05"></a>**RN-GW-05:** el GW no ejecuta lógica de dominio, no accede a bases de datos ni importa DTOs o librerías de los MS, y no restringe métodos HTTP (un `405` lo responde el MS). |
| **Postcondiciones** | El cliente nunca conoce las URI, IPs ni puertos internos. |
| **Criterios de aceptación** | **CA-GW-01-01:** `POST /api/v1/auth/login` llega a MS01 con el método, la ruta, el cuerpo y los query params sin cambios.<br>**CA-GW-01-02:** una ruta no configurada devuelve `404 NO_ENCONTRADO` sin invocar a un MS.<br>**CA-GW-01-03:** un destino caído devuelve `502` y un destino que demora más que `GATEWAY_DOWNSTREAM_TIMEOUT_MS` devuelve `504`.<br>**CA-GW-01-04:** ninguna respuesta expone URI internas, IPs, nombres de contenedores ni clases.<br>**CA-GW-01-05:** una configuración con prefijos duplicados, destino ausente o URI inválida impide el arranque y el log indica la causa. |
| **Requerimientos asociados** | RF-GW-02…08, RNF-GW-01, RNF-GW-02, RNF-GW-06. |
| **Estado** | Pendiente / Equipo API Gateway |

<a id="rf-gw-02"></a>
### RF-GW-02 Validación del JWT

| Atributo | Detalle |
|---|---|
| **Descripción** | En las rutas protegidas, el GW exige y valida localmente el JWT emitido por MS01, según EST, sin consultar a MS01. |
| **Actor(es)** | Usuario autenticado; MS01 como emisor. |
| **Precondiciones** | `JWT_PUBLIC_KEY`, `JWT_ISSUER`, `JWT_AUDIENCE`, `JWT_CLOCK_SKEW_SECONDS` y `JWT_MAX_LIFETIME_SECONDS` configurados. |
| **Flujo base** | 1. Extrae `Authorization: Bearer <JWT>`.<br>2. Exige `alg = RS256` y verifica la firma con `JWT_PUBLIC_KEY`.<br>3. Valida `iss`, `aud`, `exp`, `iat` y `nbf` (si existe) con la tolerancia `JWT_CLOCK_SKEW_SECONDS`.<br>4. Valida que `sub` sea UUID, que `roles` sea un arreglo no vacío con elementos `^[A-Z][A-Z0-9_]*$`, y que `exp − iat ≤ JWT_MAX_LIFETIME_SECONDS`.<br>5. Continúa con RF-GW-03. |
| **Flujos alternos** | **A1, falta `Authorization` o el esquema no es Bearer:** `401 NO_AUTENTICADO`.<br>**A2, token mal formado, expirado, con firma inválida, algoritmo distinto de RS256 (incluido `none`), `iss`/`aud` incorrectos o claims inválidos:** `401 NO_AUTENTICADO`, sin llamar al MS.<br>**A3, más de una cabecera `Authorization`:** `400 VALIDACION`.<br>**A4, ruta pública:** no se exige ni se valida el token. |
| **Reglas de negocio** | <a id="rn-gw-06"></a>**RN-GW-06:** la validación no tiene estado: no hay llamada a MS01, ni introspección, ni lista de revocación. Un token válido se acepta hasta su `exp`, aunque el usuario cambie de rol o sea inactivado.<br><a id="rn-gw-07"></a>**RN-GW-07:** el GW no crea usuarios, no genera ni renueva tokens y no valida contraseñas.<br><a id="rn-gw-08"></a>**RN-GW-08:** toda ruta no declarada como pública o interna en GW-C es protegida. Agregar una ruta pública exige un cambio en EST por PR.<br><a id="rn-gw-09"></a>**RN-GW-09:** el GW valida el formato de los roles, pero no mantiene un catálogo ni una matriz; la autorización por rol es de cada MS. |
| **Postcondiciones** | Las solicitudes protegidas rechazadas no consumen recursos de los MS. |
| **Criterios de aceptación** | **CA-GW-02-01:** un JWT válido emitido por MS01 permite la solicitud.<br>**CA-GW-02-02:** sin `Authorization`, la respuesta es `401` y el MS no recibe la solicitud.<br>**CA-GW-02-03:** un token expirado, alterado, firmado con otra llave, con `alg: none`/`HS256`, o con `iss`/`aud` incorrectos devuelve `401`.<br>**CA-GW-02-04:** un token con `sub` que no es UUID, `roles` vacío o vigencia mayor a `JWT_MAX_LIFETIME_SECONDS` devuelve `401`.<br>**CA-GW-02-05:** las cuatro rutas públicas de MS01 funcionan sin JWT.<br>**CA-GW-02-06:** validar un JWT no genera ninguna llamada de red a MS01. |
| **Requerimientos asociados** | RF-GW-01, RF-GW-03, RNF-GW-03; RF-MS01-03 (emisión). |
| **Estado** | Pendiente / Equipo API Gateway, en coordinación con MS01 |

<a id="rf-gw-03"></a>
### RF-GW-03 Propagación de identidad y anti-suplantación

| Atributo | Detalle |
|---|---|
| **Descripción** | El GW construye las cabeceras técnicas de EST hacia el MS. La identidad solo puede venir del JWT validado o de un MS autenticado con su clave de servicio (RF-GW-08), nunca del cliente externo. |
| **Actor(es)** | Cliente, GW y MS01–MS08. |
| **Precondiciones** | RF-GW-02 o RF-GW-08 se cumplieron, o la ruta es pública. |
| **Flujo base** | 1. Elimina, sin distinguir mayúsculas/minúsculas, todo `X-User-Id`, `X-User-Role` y `X-Service-Name` del cliente externo.<br>2. Elimina `Authorization` y `X-Service-Key`.<br>3. En rutas protegidas con JWT, inyecta `X-User-Id` = `sub` y `X-User-Role` = `roles` unidos por coma, sin espacios, en el orden del token.<br>4. Elimina las cabeceras hop-by-hop (`Connection`, `Keep-Alive`, `Proxy-Authenticate`, `Proxy-Authorization`, `TE`, `Trailer`, `Transfer-Encoding`, `Upgrade`).<br>5. Agrega `X-Correlation-ID` (RF-GW-06). |
| **Flujos alternos** | **A1, ruta pública:** se ejecutan los pasos 1, 2, 4 y 5; no se inyecta identidad.<br>**A2, cabeceras de identidad duplicadas o con otras mayúsculas:** se eliminan todas y queda una sola versión canónica. |
| **Reglas de negocio** | <a id="rn-gw-10"></a>**RN-GW-10:** la identidad nunca se toma del cuerpo, los query params, las cookies ni de cabeceras del cliente externo.<br><a id="rn-gw-11"></a>**RN-GW-11:** el JWT nunca se reenvía a un MS ni se registra en logs.<br><a id="rn-gw-12"></a>**RN-GW-12:** el GW no autoriza por rol; cada MS autoriza con la matriz. |
| **Postcondiciones** | El MS recibe una identidad confiable sin conocer el token. |
| **Criterios de aceptación** | **CA-GW-03-01:** un token con `sub = 3f6c2a9e-…` y `roles = [BRIGADISTA, ESTUDIANTE]` produce `X-User-Id: 3f6c2a9e-…` y `X-User-Role: BRIGADISTA,ESTUDIANTE`.<br>**CA-GW-03-02:** si el cliente envía `X-User-Role: ADMIN` con un token de `ESTUDIANTE`, el MS recibe solo `ESTUDIANTE`.<br>**CA-GW-03-03:** el MS nunca recibe `Authorization` ni `X-Service-Key`.<br>**CA-GW-03-04:** una ruta pública no recibe `X-User-Id` ni `X-User-Role`, aunque el cliente los envíe.<br>**CA-GW-03-05:** un `X-Service-Name` enviado por un cliente externo nunca llega al MS. |
| **Requerimientos asociados** | RF-GW-02, RF-GW-06, RF-GW-08. |
| **Estado** | Pendiente / Equipo API Gateway |

<a id="rf-gw-04"></a>
### RF-GW-04 CORS centralizado

| Atributo | Detalle |
|---|---|
| **Descripción** | El GW aplica la única política CORS y resuelve el preflight sin JWT y sin llamar al MS. |
| **Actor(es)** | Navegador del frontend. |
| **Precondiciones** | `CORS_ALLOWED_ORIGINS`, `CORS_ALLOW_CREDENTIALS` y `CORS_MAX_AGE_SECONDS` configurados. |
| **Flujo base** | 1. Si la solicitud tiene `Origin` permitido y es un preflight `OPTIONS` válido (método y cabeceras permitidos) a una ruta existente, responde `200` sin cuerpo, con las cabeceras CORS y `Vary: Origin`.<br>2. Si es una solicitud real con un `Origin` permitido, agrega `Access-Control-Allow-Origin` y expone `X-Correlation-ID`. |
| **Flujos alternos** | **A1, origen, método o cabeceras no permitidos en el preflight:** `403 PROHIBIDO`.<br>**A2, solicitud real con origen no permitido:** se procesa sin cabeceras CORS (el navegador bloquea la lectura).<br>**A3, sin `Origin`:** se procesa sin cabeceras CORS.<br>**A4, preflight a una ruta inexistente:** `404 NO_ENCONTRADO`.<br>**A5, credenciales habilitadas con origen `*`:** el GW no arranca. |
| **Reglas de negocio** | <a id="rn-gw-13"></a>**RN-GW-13:** CORS no sustituye la autenticación. `X-Service-Key`, `X-User-Id` y `X-User-Role` no son cabeceras permitidas para el navegador. |
| **Criterios de aceptación** | **CA-GW-04-01:** un preflight desde un origen permitido responde `200` con `Access-Control-Allow-Origin`, `-Methods` y `-Headers`, sin llamar al MS.<br>**CA-GW-04-02:** un preflight desde un origen no permitido responde `403` sin `Access-Control-Allow-Origin`.<br>**CA-GW-04-03:** una solicitud real permitida expone `X-Correlation-ID`.<br>**CA-GW-04-04:** la configuración `*` con credenciales impide el arranque. |
| **Requerimientos asociados** | RF-GW-01, RF-GW-05. |
| **Estado** | Pendiente / Equipo API Gateway |

<a id="rf-gw-05"></a>
### RF-GW-05 Normalización de errores

| Atributo | Detalle |
|---|---|
| **Descripción** | Todo error del GW usa el `ErrorResponse` de EST con los códigos de GW-C. Los errores de los MS se propagan solo si cumplen el contrato; si no, se sustituyen por `502`. |
| **Actor(es)** | Clientes y MS downstream. |
| **Precondiciones** | La correlación ya está asignada (RF-GW-06). |
| **Flujo base** | 1. Las respuestas `2xx`/`3xx` del MS se devuelven sin cambios.<br>2. Ante una respuesta `4xx`/`5xx` del MS, se verifica que sea JSON con los cinco campos, que `status` sea igual al código HTTP y que `correlation_id` sea igual al de la solicitud.<br>3. Si es válida, se devuelve sin cambios.<br>4. Los errores propios del GW se construyen con `Content-Type: application/json` y `X-Correlation-ID`. |
| **Flujos alternos** | **A1, error del MS no conforme** (cuerpo no JSON, campos faltantes o adicionales, `status` distinto, `correlation_id` distinto): `502 SERVICIO_NO_DISPONIBLE` con mensaje genérico; el log registra el motivo, nunca el cuerpo.<br>**A2, excepción no controlada en el GW:** `500 ERROR_INTERNO`; el stack trace solo va al log.<br>**A3, respuesta ya iniciada:** se registra el evento y se cierra la conexión. |
| **Reglas de negocio** | <a id="rn-gw-14"></a>**RN-GW-14:** un error del GW tiene exactamente los cinco campos de EST y nunca expone stack traces, clases, rutas internas, IPs, contenedores ni secretos.<br><a id="rn-gw-15"></a>**RN-GW-15:** los errores de negocio válidos de un MS (`400`, `401`, `403`, `404`, `405`, `409`, `415`, `422`, `5xx`) llegan al cliente sin cambios. |
| **Criterios de aceptación** | **CA-GW-05-01:** un error de autenticación del GW contiene los cinco campos y `codigo: NO_AUTENTICADO`.<br>**CA-GW-05-02:** un `409 CONFLICTO` válido de MS01 llega sin cambios en el estado ni en el cuerpo.<br>**CA-GW-05-03:** un `405` del MS con un `ErrorResponse` válido llega sin cambios; un `405` con el cuerpo por defecto de Spring se convierte en `502`.<br>**CA-GW-05-04:** un error del MS sin `correlation_id`, o con otro valor, se convierte en `502`.<br>**CA-GW-05-05:** el `correlation_id` de un error del GW es igual a su cabecera `X-Correlation-ID`. |
| **Requerimientos asociados** | RF-GW-01, RF-GW-06. |
| **Estado** | Pendiente / Equipo API Gateway |

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
| **Estado** | Pendiente / Equipo API Gateway |

<a id="rf-gw-07"></a>
### RF-GW-07 Logging técnico y healthcheck

| Atributo | Detalle |
|---|---|
| **Descripción** | El GW registra un log estructurado por solicitud y expone su propio healthcheck. No audita. |
| **Actor(es)** | DevOps, monitoreo y soporte. |
| **Precondiciones** | Spring Boot Actuator habilitado; puerto de gestión `MANAGEMENT_SERVER_PORT` interno. |
| **Flujo base** | 1. Al terminar cada solicitud, registra los campos mínimos de EST, con el nivel según el estado.<br>2. `GET /actuator/health` en el puerto `SERVER_PORT` responde `200 {"status":"UP"}` sin detalles y sin consultar a otros servicios.<br>3. Las métricas y el resto de endpoints de Actuator solo responden en `MANAGEMENT_SERVER_PORT`. |
| **Flujos alternos** | **A1, login o registro:** nunca se registra el cuerpo.<br>**A2, falla el sistema de logs:** la respuesta no cambia y no se bloquea la solicitud.<br>**A3, un MS está caído:** el healthcheck del GW sigue en `UP`; reporta solo su propio estado. |
| **Reglas de negocio** | <a id="rn-gw-17"></a>**RN-GW-17:** `duration_ms` se mide desde que la solicitud entra al GW hasta que termina la respuesta.<br><a id="rn-gw-18"></a>**RN-GW-18:** el GW no envía eventos a MS08 ni depende de él. |
| **Criterios de aceptación** | **CA-GW-07-01:** una solicitud genera un log con todos los campos mínimos de EST, incluidos `upstream` y `origen`.<br>**CA-GW-07-02:** los logs de login y registro no contienen contraseña, token, `Authorization` ni `X-Service-Key`.<br>**CA-GW-07-03:** `GET /actuator/health` responde `200 {"status":"UP"}` sin JWT en `≤ 200 ms`.<br>**CA-GW-07-04:** `GET /actuator/metrics` en el puerto `SERVER_PORT` responde `404`. |
| **Requerimientos asociados** | RF-GW-06, RNF-GW-04. |
| **Estado** | Pendiente / Equipo API Gateway |

<a id="rf-gw-08"></a>
### RF-GW-08 Llamadas internas entre servicios

| Atributo | Detalle |
|---|---|
| **Descripción** | Un MS que llama a otro MS lo hace a través del GW y se autentica con su clave de servicio. El GW la valida, identifica al llamante y reenvía la identidad del usuario original. |
| **Actor(es)** | MS01–MS08 como llamantes. |
| **Precondiciones** | `GW_SERVICE_KEY_MS01` … `GW_SERVICE_KEY_MS08` configuradas. La llamada está registrada en EST. |
| **Flujo base** | 1. Recibe `X-Service-Key` y, si el MS actúa en nombre de un usuario, `X-User-Id` y `X-User-Role`.<br>2. Compara la clave en tiempo constante contra las configuradas y obtiene el nombre del llamante (`ms01`…`ms08`).<br>3. Valida el formato de `X-User-Id` (UUID) y de `X-User-Role` si vienen.<br>4. Elimina `X-Service-Key`, inyecta `X-Service-Name` y conserva `X-User-Id`/`X-User-Role`.<br>5. Continúa con el enrutamiento (RF-GW-01). |
| **Flujos alternos** | **A1, clave inválida:** `401 NO_AUTENTICADO`.<br>**A2, `Authorization` y `X-Service-Key` en la misma solicitud:** `400 VALIDACION`.<br>**A3, `X-User-Id` sin `X-User-Role` (o al revés), o con formato inválido:** `400 VALIDACION`.<br>**A4, ruta interna sin `X-Service-Key`:** `401 NO_AUTENTICADO`. |
| **Reglas de negocio** | <a id="rn-gw-19"></a>**RN-GW-19:** una ruta **interna** solo acepta clave de servicio; una ruta **protegida** acepta JWT o clave de servicio.<br><a id="rn-gw-20"></a>**RN-GW-20:** las claves nunca se registran en logs ni se reenvían. Cada MS tiene su propia clave, de al menos 32 caracteres.<br><a id="rn-gw-21"></a>**RN-GW-21:** el GW no decide qué servicio puede llamar a cuál; el MS destino autoriza por `X-Service-Name` o por `X-User-Role`. |
| **Postcondiciones** | La llamada interna queda en el log con `origen = msNN` y la misma correlación. |
| **Criterios de aceptación** | **CA-GW-08-01:** MS04 con una clave válida y `X-User-*` de un ESTUDIANTE llega a MS05 con `X-Service-Name: ms04` y la misma identidad.<br>**CA-GW-08-02:** una clave inválida devuelve `401` sin llamar al destino.<br>**CA-GW-08-03:** un cliente externo sin clave recibe `401` en `POST /api/v1/audit/events`.<br>**CA-GW-08-04:** el MS destino nunca recibe `X-Service-Key`. |
| **Requerimientos asociados** | RF-GW-01, RF-GW-03, RF-GW-07. |
| **Estado** | Pendiente / Equipo API Gateway |
