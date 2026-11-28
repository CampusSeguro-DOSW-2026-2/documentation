# Seguridad — MS08 Analítica y auditoría

Cómo MS08 recibe la identidad del servicio o del usuario que llama, cómo autoriza cada operación y cómo protege los registros de auditoría. Los valores transversales están en los [Estándares de Integración](../integration/standards.md); cómo se forma la credencial de acceso, en la [seguridad de MS01](../ms01-identity-and-access/security.md#formación-del-jwt); y lo que hace el Gateway con ella, en la [seguridad del Gateway](../api-gateway/security.md).

## Índice

1. [Registro, autenticación y JWT](#registro-autenticación-y-jwt)
2. [Identidad del servicio y del usuario](#identidad-del-servicio-y-del-usuario)
3. [Autorización por permiso](#autorización-por-permiso)
4. [Ruta interna de registro](#ruta-interna-de-registro)
5. [Llamadas a otros microservicios](#llamadas-a-otros-microservicios)
6. [CORS y mTLS](#cors-y-mtls)
7. [Datos sensibles](#datos-sensibles)
8. [Otras medidas](#otras-medidas)

---

## Registro, autenticación y JWT

MS08 no registra usuarios, no autentica credenciales y nunca recibe ni valida el JWT. MS01 emite el token y el Gateway lo valida y lo convierte en cabeceras ([validación del JWT](../api-gateway/security.md#validación-del-jwt)). MS08 solo lee esas cabeceras ([RN-AYA-12](1_requirements.md#rn-aya-12)).

## Identidad del servicio y del usuario

`GatewayAuthenticationFilter` construye la identidad de cada petición con las cabeceras que inyecta el Gateway:

| Caso | Cabeceras | Validación | Si es inválida |
|---|---|---|---|
| Un microservicio llama | `X-Service-Name` | Un valor de `ms01` a `ms08`, sin distinguir mayúsculas | `401 NO_AUTENTICADO` |
| Un usuario llama | `X-User-Id` y `X-User-Permission` | `X-User-Id` es un UUID y `X-User-Permission` trae al menos un permiso (lista separada por coma; se recortan los espacios y se ignoran los repetidos) | `401 NO_AUTENTICADO` |

Si llega `X-Service-Name`, la petición se trata como de un servicio aunque también traiga cabeceras de usuario. `X-User-Role` no se usa: MS08 nunca autoriza por el nombre del rol. Una petición sin ninguna de estas cabeceras queda anónima y solo puede llegar a `/actuator/health`; en cualquier otra ruta recibe `401 NO_AUTENTICADO`.

## Autorización por permiso

La cadena de Spring Security exige un permiso de la [matriz de roles y permisos](../integration/permissions-matrix.md) por ruta; si falta, responde `403 PROHIBIDO`:

| Permiso | Operaciones |
|---|---|
| `ms08:audit_drill:read` | Buscar y ver el detalle de los registros de simulacros |
| `ms08:audit_real_event:read` | Buscar y ver el detalle de los registros de emergencias reales |
| `ms08:indicators_drill:read` | Indicadores, comparaciones y ubicaciones de simulacros |
| `ms08:indicators_real_event:read` | Indicadores, comparaciones y ubicaciones de emergencias reales |

Además del permiso de la ruta, el servicio limita cada resultado al tipo de evento que el usuario puede ver ([RN-AYA-10](1_requirements.md#rn-aya-10), [RN-AYA-11](1_requirements.md#rn-aya-11)):

* **Auditoría (`AuditVisibility`):** quien tiene un solo permiso de auditoría solo ve ese tipo de registro; los registros administrativos, sin `tipo_evento`, exigen los dos. Filtrar por un tipo que no puede ver, o abrir el detalle de un registro de ese tipo, responde `403`.
* **Indicadores (`IndicatorVisibility`):** el tipo del evento se determina con sus registros; consultar un evento de un tipo sin su permiso responde `403`. Los permisos de auditoría no dan acceso a los indicadores, ni al revés.

## Ruta interna de registro

`POST /api/v1/audit/events` solo la usan los microservicios ([RF-AYA-01](1_requirements.md#rf-aya-01)):

1. El emisor llama al Gateway con su clave de servicio; el Gateway la valida, la elimina e inyecta `X-Service-Name`.
2. MS08 exige la autoridad de servicio: una petición con identidad de usuario recibe `403 PROHIBIDO` ([RN-AYA-13](1_requirements.md#rn-aya-13)).
3. El nombre del servicio verificado se guarda en el registro como origen, sin mostrarse en las consultas.

Del mismo modo, las rutas de consulta exigen un permiso de usuario, así que un microservicio recibe `403` si intenta consultarlas.

## Llamadas a otros microservicios

MS08 no llama a ningún otro microservicio ni usa clave de servicio: solo recibe registros y consultas.

## CORS y mTLS

* **CORS:** MS08 no configura CORS. El navegador nunca llama a MS08: la única política CORS es la del Gateway ([CORS](../api-gateway/security.md#cors)).
* **Aislamiento de red:** MS08 confía en `X-Service-Name`, `X-User-Id` y `X-User-Permission` porque solo el Gateway puede llegar a él. En la nube, el firewall de su VM solo acepta la IP del Gateway y su Caddy de borde exige el certificado de cliente del Gateway (mTLS), firmado por la CA privada del entorno ([aislamiento en la nube](../api-gateway/contracts.md#7-aislamiento-en-la-nube)).
* **Base de datos:** MS08 se conecta a Azure DocumentDB con la cadena de conexión de `MONGODB_URI`; ningún otro microservicio accede a ella ([RN-AYA-19](1_requirements.md#rn-aya-19)).

## Datos sensibles

| Dato | Tratamiento |
|---|---|
| Contenido de los registros | `detalle` y `atributos` solo admiten códigos, identificadores y conteos, nunca datos personales ni contenido de mensajes ([RN-AYA-09](1_requirements.md#rn-aya-09)). MS08 guarda el UUID del usuario, no su nombre ni su correo |
| Inmutabilidad | La API no ofrece operaciones para modificar ni borrar registros; el único borrado es el de retención a los 180 días ([RN-AYA-01](1_requirements.md#rn-aya-01)) |
| Origen verificado | El servicio que envió el registro se guarda para trazabilidad y no se expone en las consultas |
| Cadena de conexión | `MONGODB_URI` llega por variable de entorno, nunca en el repositorio |
| Logs | Cada línea lleva el Correlation ID. Los logs registran el método, la ruta, el código de error y, al guardar un registro, su `id`, módulo y acción; nunca cuerpos de petición, `detalle`, `atributos`, cabeceras ni credenciales |
| Errores | Todo error, incluidos los del framework (`ApiErrorController`), sale como `ErrorResponse` sin trazas ni detalles internos |

## Otras medidas

* **Validación en el servidor:** los cuerpos rechazan campos desconocidos (`fail-on-unknown-properties`), y los campos y filtros se validan con las reglas de [2_forms-inputs.md](2_forms-inputs.md); el error indica el campo y su mensaje.
* **Sin estado:** la cadena de seguridad es `STATELESS`, sin sesión HTTP, sin formulario de inicio de sesión, sin autenticación básica y sin CSRF, porque no usa cookies.
* **Correlation ID:** `CorrelationIdFilter` toma `X-Correlation-ID` de la petición o genera uno, lo pone en el log y lo devuelve en la respuesta.
* **Superficie expuesta:** Actuator solo expone `/actuator/health`, sin detalles. MS08 no publica Swagger UI ni la especificación OpenAPI en tiempo de ejecución.
* **Retención:** el índice TTL o la tarea diaria eliminan los registros vencidos sin intervención manual ([RF-AYA-07](1_requirements.md#rf-aya-07)).
