# Seguridad — MS06 Eventos

Cómo MS06 recibe la identidad del usuario, cómo autoriza cada operación según el tipo de evento, cómo coordina a los demás microservicios y qué cambia en el perfil de desarrollo local. Los valores transversales están en los [Estándares de Integración](../integration/standards.md); cómo se forma la credencial de acceso, en la [seguridad de MS01](../ms01-identity-and-access/security.md#formación-del-jwt); y lo que hace el Gateway con ella, en la [seguridad del Gateway](../api-gateway/security.md).

## Índice

1. [Registro, autenticación y JWT](#registro-autenticación-y-jwt)
2. [Identidad del usuario](#identidad-del-usuario)
3. [Autorización por permiso](#autorización-por-permiso)
4. [Llamadas a otros microservicios y clave de servicio](#llamadas-a-otros-microservicios-y-clave-de-servicio)
5. [Perfil local](#perfil-local)
6. [CORS y mTLS](#cors-y-mtls)
7. [Datos sensibles](#datos-sensibles)
8. [Otras medidas](#otras-medidas)

---

## Registro, autenticación y JWT

En la nube MS06 no registra usuarios, no autentica credenciales y nunca recibe ni valida el JWT de MS01. MS01 emite el token y el Gateway lo valida y lo convierte en cabeceras ([validación del JWT](../api-gateway/security.md#validación-del-jwt)). MS06 solo lee esas cabeceras. El contenedor corre con el perfil `prod`, que activa esa cadena de seguridad.

## Identidad del usuario

`IdentityHeadersAuthenticationFilter` construye la identidad de la petición con las cabeceras que inyecta el Gateway:

| Cabecera | Validación en MS06 |
|---|---|
| `X-User-Id` | UUID |
| `X-User-Role` | Lista separada por coma de roles; se recortan los espacios y se ignoran los repetidos |
| `X-User-Permission` | Lista separada por coma con al menos un permiso |

Una petición sin ninguna de las tres cabeceras queda anónima y solo llega a `/actuator/health`; en cualquier otra ruta responde `401`. Si llega alguna pero `X-User-Id` no es un UUID o no hay permisos, responde `401` directamente.

## Autorización por permiso

Cada operación exige un permiso de la [matriz de roles y permisos](../integration/permissions-matrix.md) (`@PreAuthorize`); si falta, responde `403`. Los permisos se dividen por tipo de evento: `ms06:drill:*` para simulacros y `ms06:real_event:*` para eventos reales.

| Operación | Permiso de la ruta |
|---|---|
| Crear y editar eventos | `ms06:drill:manage` o `ms06:real_event:manage` |
| Finalizar un evento | `ms06:drill:execute` o `ms06:real_event:execute` |
| Listar eventos y ver uno | `ms06:drill:read` o `ms06:real_event:read` |
| Eventos activos y panel de un evento en curso | Cualquier usuario autenticado |
| Listar tipos de evento y de alerta | `ms06:drill:read`, `ms06:real_event:read`, `ms06:drill:manage` o `ms06:real_event:manage` |
| Gestionar tipos de evento | `ms06:real_event:manage` |

Además del permiso de la ruta, `EventAccessPolicy` comprueba el permiso del tipo concreto del evento: con solo `ms06:drill:manage` se crean y editan simulacros pero no eventos reales. Los roles de `X-User-Role` se usan para saber a qué eventos fue convocado un usuario, no para autorizar.

## Llamadas a otros microservicios y clave de servicio

MS06 nunca llama directamente a otro microservicio: todas sus llamadas van al Gateway (`GATEWAY_URL`) con su clave de servicio (`GATEWAY_SERVICE_KEY`) en `X-Service-Key` y tiempos máximos de 2000 ms para conectar y para leer:

| Destino | Rutas | Identidad que envía |
|---|---|---|
| MS02 | Edificios y zonas del alcance del evento | La del usuario, o la identidad de sistema en los procesos automáticos |
| MS03 | `POST /api/v1/alertas`, `PATCH /api/v1/alertas/{id}/programar`, `/archivar` y `/activar` | La del usuario que actúa; en el inicio automático, la identidad de sistema |
| MS04 | `PUT /api/v1/chi/eventos/{id}/estado` y `GET /api/v1/chi/eventos/{id}/checkins` (rutas internas), y el consolidado del evento | MS04 autoriza las rutas internas por `X-Service-Name: ms06`; el consolidado se pide con la identidad de sistema |
| MS05 | `PUT /api/v1/bei/eventos/{id}/estado`, `GET /api/v1/bei/eventos/{id}/brigadistas` y `/indicadores`, `POST /api/v1/asignaciones/predeterminadas` (rutas internas) | MS05 las autoriza por `X-Service-Name: ms06` |
| MS08 | `POST /api/v1/audit/events` | El usuario de la acción |

La **identidad de sistema** es la que MS06 usa cuando no hay un usuario detrás de la acción (el inicio automático de un evento o una sincronización): `GATEWAY_SYSTEM_USER_ID`, `GATEWAY_SYSTEM_USER_ROLE` (`ORGANIZADOR` por defecto) y `GATEWAY_SYSTEM_USER_PERMISSIONS`, con los permisos de MS02, MS03 y MS04 que necesitan esas llamadas. La auditoría se guarda primero en una bandeja de salida en la base de datos y un proceso la envía a MS08 cada 15 s, así que una caída de MS08 no afecta la operación del usuario. Si MS04 o MS05 no responden, el evento cambia de estado igual y la llamada se guarda para reintentarla.

## Perfil local

Con el perfil `local` (el predeterminado al ejecutar `mvn spring-boot:run`), MS06 no lee las cabeceras del Gateway: valida un JWT firmado con HS256 con un secreto local (`JWT_LOCAL_SECRET`) y expone `/local/tokens` sin autenticación para emitir tokens de prueba, y los demás módulos se reemplazan por *stubs*. Ese perfil no se usa en la nube: el contenedor corre con `SPRING_PROFILES_ACTIVE=prod`, donde no existen ni el emisor de tokens ni la validación local.

## CORS y mTLS

* **CORS:** MS06 no configura CORS. El navegador nunca llama a MS06: la única política CORS es la del Gateway ([CORS](../api-gateway/security.md#cors)).
* **Aislamiento de red:** MS06 confía en `X-User-*` porque solo el Gateway puede llegar a él. En la nube, el firewall de su VM solo acepta la IP del Gateway y su Caddy de borde exige el certificado de cliente del Gateway (mTLS), firmado por la CA privada del entorno ([aislamiento en la nube](../api-gateway/contracts.md#7-aislamiento-en-la-nube)). Su base de datos PostgreSQL corre en el mismo stack y solo es visible dentro de él.

## Datos sensibles

| Dato | Tratamiento |
|---|---|
| Identidad | MS06 guarda el UUID de quien crea, edita o finaliza un evento; no guarda nombres ni correos |
| Credenciales | La contraseña de la base de datos, la clave de servicio y el secreto local llegan por variables de entorno |
| Errores | Todo error, incluidos los del framework, sale como `ErrorResponse` sin trazas ni detalles internos |

## Otras medidas

* **Validación en el servidor:** cada campo se valida con las reglas de [2_forms-inputs.md](2_forms-inputs.md).
* **Sin estado:** la cadena de seguridad es `STATELESS`, sin sesión HTTP, sin formulario de inicio de sesión, sin autenticación básica y sin CSRF, porque no usa cookies.
* **Correlation ID:** `CorrelationIdFilter` toma `X-Correlation-ID` de la petición o genera uno y lo devuelve en la respuesta.
* **Superficie expuesta:** Actuator solo expone `/actuator/health`, sin detalles. MS06 publica Swagger UI y la especificación OpenAPI en `/swagger-ui.html` y `/v3/api-docs` sin permiso; el Gateway no enruta esas rutas, así que solo son accesibles dentro de la red de MS06.
