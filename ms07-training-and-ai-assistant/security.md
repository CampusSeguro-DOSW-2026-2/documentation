# Seguridad — MS07 Formación y asistente de IA

Cómo MS07 recibe la identidad del usuario, cómo autoriza cada operación, cómo llama a MS08 y al proveedor del modelo de lenguaje y cómo protege las consultas al tutor. Los valores transversales están en los [Estándares de Integración](../integration/standards.md); cómo se forma la credencial de acceso, en la [seguridad de MS01](../ms01-identity-and-access/security.md#formación-del-jwt); y lo que hace el Gateway con ella, en la [seguridad del Gateway](../api-gateway/security.md).

## Índice

1. [Registro, autenticación y JWT](#registro-autenticación-y-jwt)
2. [Identidad del usuario](#identidad-del-usuario)
3. [Autorización por permiso](#autorización-por-permiso)
4. [Llamadas salientes y clave de servicio](#llamadas-salientes-y-clave-de-servicio)
5. [CORS y mTLS](#cors-y-mtls)
6. [Datos sensibles](#datos-sensibles)
7. [Otras medidas](#otras-medidas)

---

## Registro, autenticación y JWT

MS07 no registra usuarios, no autentica credenciales y nunca recibe ni valida el JWT. MS01 emite el token y el Gateway lo valida y lo convierte en cabeceras ([validación del JWT](../api-gateway/security.md#validación-del-jwt)). MS07 solo lee esas cabeceras.

## Identidad del usuario

`HeaderAuthenticationFilter` construye la identidad de la petición con las tres cabeceras que inyecta el Gateway:

| Cabecera | Uso en MS07 |
|---|---|
| `X-User-Id` | Identifica al usuario. Las operaciones que guardan datos del usuario (visualizaciones, intentos, progreso, consultas al tutor) exigen que sea un UUID; si no lo es, responden `400 VALIDACION` con `El identificador de usuario no es válido.` |
| `X-User-Role` | Debe venir, pero no autoriza nada |
| `X-User-Permission` | Lista separada por coma de los permisos; se recortan los espacios y se ignoran los repetidos |

Si falta `X-User-Id` o `X-User-Role`, o no llega `X-User-Permission`, la petición queda sin identidad y responde `401 NO_AUTENTICADO` con `Faltan las cabeceras de identidad.` El progreso y los intentos siempre son los de `X-User-Id`: no hay forma de consultar los de otra persona.

## Autorización por permiso

La cadena de Spring Security exige un permiso de la [matriz de roles y permisos](../integration/permissions-matrix.md) por método y ruta; si falta, responde `403 PROHIBIDO` con `No tiene permiso para realizar esta operación.`:

| Permiso | Operaciones |
|---|---|
| `ms07:content:read` | Listar categorías y contenidos, ver un contenido y registrar su visualización |
| `ms07:quiz:take` | Listar y ver evaluaciones, presentar un intento y consultar el progreso propio |
| `ms07:tutor:use` | Preguntar al tutor |
| `ms07:content:manage` | Crear, actualizar, desactivar y reactivar categorías, contenidos y evaluaciones; también lista y ve categorías, contenidos y evaluaciones, incluidos los inactivos (`includeInactive=true`) |

Una ruta que no está en la tabla anterior se rechaza (`denyAll`). `X-User-Role` nunca se usa para autorizar.

## Llamadas salientes y clave de servicio

| Destino | Ruta | Cómo |
|---|---|---|
| MS08, a través del Gateway | `POST /api/v1/audit/events` | Cada evento se guarda primero en la tabla local `audit_event` y, después del commit, se envía de forma asíncrona con `X-Service-Key` (`GATEWAY_SERVICE_KEY`) y el `X-Correlation-ID` de la petición; propaga `X-User-Id`, `X-User-Role` y `X-User-Permission` del usuario de la acción. Una falla del envío no cambia la respuesta al usuario |
| Proveedor del modelo de lenguaje (OpenAI o Azure OpenAI) | `LLM_URL` | Por HTTPS con `LLM_API_KEY`: `Authorization: Bearer` en OpenAI y la cabecera `api-key` en Azure OpenAI. Tiempo máximo `LLM_TIMEOUT` (8 s por defecto). Si el proveedor falla, tarda o no está configurado, MS07 responde `503 DEPENDENCIA_NO_DISPONIBLE` sin reenviar el error del proveedor |

## CORS y mTLS

* **CORS:** MS07 no configura CORS. El navegador nunca llama a MS07: la única política CORS es la del Gateway ([CORS](../api-gateway/security.md#cors)).
* **Aislamiento de red:** MS07 confía en `X-User-*` porque solo el Gateway puede llegar a él. En la nube, el firewall de su VM solo acepta la IP del Gateway y su Caddy de borde exige el certificado de cliente del Gateway (mTLS), firmado por la CA privada del entorno ([aislamiento en la nube](../api-gateway/contracts.md#7-aislamiento-en-la-nube)). Su base de datos PostgreSQL corre en el mismo stack y solo es visible dentro de él.

## Datos sensibles

| Dato | Tratamiento |
|---|---|
| Preguntas y respuestas del tutor | No se guardan. La auditoría de cada consulta solo registra el usuario, la acción, el resultado y el Correlation ID |
| Identidad del usuario | MS07 guarda el UUID del usuario en visualizaciones, intentos y auditoría; no guarda nombres ni correos |
| Clave del proveedor de IA, credenciales de la base de datos y clave de servicio | Variables de entorno (`LLM_API_KEY`, `DB_PASSWORD`, `GATEWAY_SERVICE_KEY`), nunca en el repositorio |
| Logs | Cada línea lleva el Correlation ID |
| Errores | Todo error, incluidos los del framework, sale como `ErrorResponse` sin trazas ni detalles internos; un error inesperado responde `500 ERROR_INTERNO` con `Ocurrió un error inesperado. Intente de nuevo más tarde.` |

## Otras medidas

* **Validación en el servidor:** cada campo se valida con las reglas de [2_forms-inputs.md](2_forms-inputs.md). La pregunta al tutor se limpia de caracteres de control e invisibles antes de enviarla al proveedor.
* **Alcance del tutor:** el tutor responde solo con los contenidos activos de MS07 y, si no encuentra información suficiente, remite al protocolo oficial de emergencias ([RF-13](1_requirements.md)).
* **Sin estado:** la cadena de seguridad es `STATELESS`, sin sesión HTTP, sin formulario de inicio de sesión, sin autenticación básica y sin CSRF, porque no usa cookies.
* **Correlation ID:** `CorrelationIdFilter` acepta `X-Correlation-ID` solo si es un UUID; si no, genera uno. Lo pone en el log y lo devuelve en la respuesta.
* **Esquema controlado:** Flyway crea el esquema y Hibernate solo lo valida (`ddl-auto=validate`).
* **Superficie expuesta:** Actuator solo expone `/actuator/health`, sin detalles. Swagger UI y la especificación OpenAPI en tiempo de ejecución están desactivados salvo que se active `SWAGGER_ENABLED`.
