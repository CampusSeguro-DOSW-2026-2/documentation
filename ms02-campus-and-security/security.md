# Seguridad — MS02 Campus y seguridad

Cómo MS02 recibe la identidad del usuario, cómo autoriza cada operación, qué expone sin sesión para los códigos QR y cómo protege las imágenes y los datos del campus. Los valores transversales están en los [Estándares de Integración](../integration/standards.md); cómo se forma la credencial de acceso, en la [seguridad de MS01](../ms01-identity-and-access/security.md#formación-del-jwt); y lo que hace el Gateway con ella, en la [seguridad del Gateway](../api-gateway/security.md).

## Índice

1. [Registro, autenticación y JWT](#registro-autenticación-y-jwt)
2. [Identidad del usuario](#identidad-del-usuario)
3. [Autorización por permiso](#autorización-por-permiso)
4. [Consulta pública por QR](#consulta-pública-por-qr)
5. [Llamadas salientes](#llamadas-salientes)
6. [CORS y mTLS](#cors-y-mtls)
7. [Datos sensibles](#datos-sensibles)
8. [Otras medidas](#otras-medidas)

---

## Registro, autenticación y JWT

MS02 no registra usuarios, no autentica credenciales y nunca recibe ni valida el JWT. MS01 emite el token y el Gateway lo valida y lo convierte en cabeceras ([validación del JWT](../api-gateway/security.md#validación-del-jwt)). MS02 solo lee esas cabeceras.

## Identidad del usuario

`HeaderAuthenticationFilter` construye la identidad de la petición con las tres cabeceras que inyecta el Gateway:

| Cabecera | Validación en MS02 |
|---|---|
| `X-User-Id` | Obligatoria |
| `X-User-Role` | Lista separada por coma; cada rol con el formato `^[A-Z][A-Z0-9_]{1,29}$` |
| `X-User-Permission` | Lista separada por coma; cada permiso con el formato `ms<dos dígitos>:<recurso>:<acción>` |

Si falta alguna de las tres o un rol o permiso no tiene el formato, la petición queda sin identidad y una ruta protegida responde `401 NO_AUTENTICADO`. Otros microservicios (MS04, MS06) consultan a MS02 por el Gateway propagando la identidad del usuario, así que se autorizan igual que una petición del frontend.

## Autorización por permiso

La cadena de Spring Security exige un permiso de la [matriz de roles y permisos](../integration/permissions-matrix.md) por método y ruta; si falta, responde `403 PROHIBIDO`:

| Permiso | Operaciones |
|---|---|
| `ms02:campus:read` | Consultar campus, zonas abiertas, edificios, pisos, espacios, salidas, escaleras, zonas seguras e imágenes |
| `ms02:route:read` | Consultar las rutas de evacuación de un edificio, un espacio o una salida, y listar y ver rutas |
| `ms02:preventive_info:read` | Consultar las recomendaciones preventivas |
| `ms02:building:write` | Gestionar campus, zonas abiertas, edificios, pisos, salidas, escaleras, imágenes y recomendaciones preventivas |
| `ms02:space:write` | Gestionar espacios |
| `ms02:safe_zone:write` | Gestionar zonas seguras |
| `ms02:route:write` | Gestionar rutas de evacuación y generar las rutas exteriores |
| `ms02:route_status:write` | Cambiar el estado de una ruta de evacuación o la disponibilidad de una escalera |
| `ms02:qr:manage` | Generar el código QR de un espacio, una salida o una zona segura |

`X-User-Role` nunca se usa para autorizar. Cualquier ruta no listada exige al menos una identidad válida.

## Consulta pública por QR

`GET /api/v1/public/spaces`, `GET /api/v1/public/exits` y `GET /api/v1/public/safe-zones` no exigen sesión: los usa la página a la que lleva un código QR ([RF-MS02-16](1_requirements.md#rf-ms02-16)). El Gateway las declara públicas y MS02 las permite sin identidad. Solo responden con lectura y sin datos personales; cualquier otro método sobre `/api/v1/public/**` se rechaza.

## Llamadas salientes

MS02 no llama a otros microservicios ni envía auditoría a MS08. Sus llamadas salientes son a servicios externos:

| Destino | Uso | Configuración |
|---|---|---|
| Servidor OSRM de OpenStreetMap | Calcular el recorrido peatonal de las rutas exteriores | `NAVIGATION_BASE_URL` (por defecto `https://routing.openstreetmap.de/routed-foot`), con tiempos máximos de 3 s para conectar y 10 s para leer. Si no responde, MS02 responde `503 NAVEGACION_NO_DISPONIBLE` y las rutas ya guardadas se siguen consultando |
| Azure Blob Storage | Guardar las imágenes cuando `IMAGEN_STORAGE_TYPE=azure` | `AZURE_STORAGE_CONNECTION_STRING` y `AZURE_STORAGE_CONTAINER` |

## CORS y mTLS

* **CORS:** MS02 no configura CORS. El navegador nunca llama a MS02: la única política CORS es la del Gateway ([CORS](../api-gateway/security.md#cors)).
* **Aislamiento de red:** MS02 confía en `X-User-*` porque solo el Gateway puede llegar a él. En la nube, el firewall de su VM solo acepta la IP del Gateway y su Caddy de borde exige el certificado de cliente del Gateway (mTLS), firmado por la CA privada del entorno ([aislamiento en la nube](../api-gateway/contracts.md#7-aislamiento-en-la-nube)).

## Datos sensibles

| Dato | Tratamiento |
|---|---|
| Información del campus | MS02 no guarda datos personales: edificios, espacios, salidas, zonas y rutas son datos de infraestructura |
| Imágenes | Se aceptan solo JPEG, PNG o WEBP, reconocidos por su contenido y no por la extensión, de hasta 5 MB (`IMAGEN_MAX_SIZE_BYTES`). El nombre original no puede superar 255 caracteres |
| Credenciales de la base de datos y de Azure Storage | Variables de entorno (`DB_PASSWORD`, `AZURE_STORAGE_CONNECTION_STRING`), nunca en el repositorio |
| Logs | Cada línea lleva el Correlation ID |
| Errores | Todo error, incluidos los del framework, sale como `ErrorResponse` sin trazas ni detalles internos |

## Otras medidas

* **Validación en el servidor:** cada campo se valida con las reglas de [2_forms-inputs.md](2_forms-inputs.md); una regla de negocio incumplida responde `422 REGLA_NEGOCIO`.
* **Sin estado:** la cadena de seguridad es `STATELESS`, sin sesión HTTP, sin formulario de inicio de sesión, sin autenticación básica y sin CSRF, porque no usa cookies.
* **Correlation ID:** `CorrelationIdFilter` acepta `X-Correlation-ID` solo si es un UUID; si no, genera uno. Lo pone en el log y lo devuelve en la respuesta.
* **Esquema controlado:** Flyway crea el esquema y Hibernate solo lo valida (`ddl-auto=validate`).
* **Superficie expuesta:** Actuator solo expone `/actuator/health`. MS02 publica Swagger UI y la especificación OpenAPI en `/swagger-ui` y `/v3/api-docs` sin permiso; el Gateway no enruta esas rutas, así que solo son accesibles dentro de la red de MS02.
