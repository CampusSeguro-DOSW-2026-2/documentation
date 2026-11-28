# Seguridad — MS05 Brigadistas e incidentes

Cómo MS05 recibe la identidad del usuario y del servicio que llama, cómo autoriza cada operación, cómo llama a otros microservicios y cómo protege las evidencias de los incidentes. Los valores transversales están en los [Estándares de Integración](../integration/standards.md); cómo se forma la credencial de acceso, en la [seguridad de MS01](../ms01-identity-and-access/security.md#formación-del-jwt); y lo que hace el Gateway con ella, en la [seguridad del Gateway](../api-gateway/security.md).

## Índice

1. [Registro, autenticación y JWT](#registro-autenticación-y-jwt)
2. [Identidad del usuario y del servicio](#identidad-del-usuario-y-del-servicio)
3. [Autorización por permiso](#autorización-por-permiso)
4. [Rutas internas de MS06](#rutas-internas-de-ms06)
5. [Llamadas a otros microservicios y clave de servicio](#llamadas-a-otros-microservicios-y-clave-de-servicio)
6. [CORS y mTLS](#cors-y-mtls)
7. [Datos sensibles](#datos-sensibles)
8. [Otras medidas](#otras-medidas)

---

## Registro, autenticación y JWT

MS05 no registra usuarios, no autentica credenciales y nunca recibe ni valida el JWT. MS01 emite el token y el Gateway lo valida y lo convierte en cabeceras ([validación del JWT](../api-gateway/security.md#validación-del-jwt)). MS05 solo lee esas cabeceras.

## Identidad del usuario y del servicio

`GatewayAuthenticationFilter` construye la identidad de la petición con las cabeceras que inyecta el Gateway:

| Caso | Cabeceras | Tratamiento |
|---|---|---|
| Un usuario llama | `X-User-Id` y `X-User-Permission` | Cada permiso con el formato `ms<dos dígitos>:<recurso>:<acción>` se convierte en una autoridad; los que no cumplen el formato se ignoran |
| MS06 llama | `X-Service-Name: ms06` | Recibe la autoridad interna `SERVICE_MS06`. Cualquier otro valor de `X-Service-Name` no da ninguna autoridad |

Una petición sin ninguna autoridad responde `401` con `Se requiere autenticación.` La única ruta sin identidad es `GET /actuator/health`.

## Autorización por permiso

Cada operación exige un permiso de la [matriz de roles y permisos](../integration/permissions-matrix.md) (`@PreAuthorize`); si falta, responde `403` con `No tiene permisos para realizar esta operación.`:

| Permiso | Operaciones |
|---|---|
| `ms05:brigadier:assign_to_drill`, `ms05:brigadier:assign_to_real` | Asignar, modificar y retirar brigadistas de un evento (simulacro o real), buscar brigadistas y consultar las asignaciones y el estado operativo |
| `ms05:panel:read_own` | Panel propio del brigadista: sus asignaciones, su estado y sus responsabilidades |
| `ms05:incident:report` | Registrar un incidente o una solicitud de asistencia, editarlo y cargar evidencias |
| `ms05:incident:read` | Consultar incidentes, su detalle, sus evidencias, su historial y el listado general de responsabilidades |
| `ms05:incident:manage_own` | Consultar incidentes (`/consulta`) |
| `ms05:incident:update` | Editar un incidente |
| `ms05:assistance:assign_to_drill`, `ms05:assistance:assign_to_real` | Asignar o reasignar el responsable de un incidente, cerrarlo y eliminarlo |
| `ms05:assistance:update_status` | Iniciar la atención y resolver un incidente |
| `ms08:audit_drill:read`, `ms08:audit_real_event:read` | Consultar la auditoría propia de MS05 (`/api/v1/bei/auditoria`) |

Los permisos `assign_to_drill` y `assign_to_real` se exigen según el tipo del evento: con un simulacro se pide el primero y con un evento real el segundo. `X-User-Role` nunca se usa para autorizar.

## Rutas internas de MS06

`PUT /api/v1/bei/eventos/{eventoId}/estado`, `GET /api/v1/bei/eventos/{eventoId}/brigadistas`, `GET /api/v1/bei/eventos/{eventoId}/indicadores` y `POST /api/v1/asignaciones/predeterminadas` exigen la autoridad `SERVICE_MS06`:

1. MS06 llama al Gateway con su clave de servicio; el Gateway la valida, la elimina e inyecta `X-Service-Name: ms06`.
2. MS05 da la autoridad `SERVICE_MS06` solo a ese valor.
3. Un usuario, aunque tenga permisos de MS05, recibe `403` en estas rutas.

## Llamadas a otros microservicios y clave de servicio

MS05 nunca llama directamente a otro microservicio: todas sus llamadas van al Gateway (`MS05_GATEWAY_URL`) con su clave de servicio (`GW_SERVICE_KEY_MS05`) en `X-Service-Key` y un tiempo máximo de 2000 ms para conectar y para leer:

| Destino | Ruta | Identidad que envía |
|---|---|---|
| MS01 | `GET /api/v1/users/by-role`, `POST /api/v1/users/batch` (brigadistas y nombres de usuarios) | Ruta interna: MS01 autoriza por `X-Service-Name: ms05` |
| MS02 | `GET /api/v1/buildings/{id}`, `GET /api/v1/safe-zones/{id}` (validar ubicaciones) | `X-User-Id`, `X-User-Role`, `X-User-Permission` y `X-Correlation-ID` de la petición en curso, sin cambios |
| MS03 | `POST /api/v1/alertas` (alertas a los brigadistas) | Las tres `X-User-*` del usuario, porque MS03 autoriza por su permiso |
| MS08 | `POST /api/v1/audit/events` | El usuario de la acción |

## CORS y mTLS

* **CORS:** MS05 no configura CORS. El navegador nunca llama a MS05: la única política CORS es la del Gateway ([CORS](../api-gateway/security.md#cors)).
* **Aislamiento de red:** MS05 confía en `X-User-*` y `X-Service-Name` porque solo el Gateway puede llegar a él. En la nube, el firewall de su VM solo acepta la IP del Gateway y su Caddy de borde exige el certificado de cliente del Gateway (mTLS), firmado por la CA privada del entorno ([aislamiento en la nube](../api-gateway/contracts.md#7-aislamiento-en-la-nube)). Su base de datos PostgreSQL corre en el mismo stack y solo es visible dentro de él.

## Datos sensibles

| Dato | Tratamiento |
|---|---|
| Evidencias | Máximo 5 por incidente. Se aceptan imágenes JPG, PNG, WEBP, GIF, BMP, TIFF o HEIC, PDF y MP4, según el tipo de contenido declarado; el archivo debe tener nombre y no estar vacío. Se guardan en un volumen del contenedor (`EVIDENCIAS_DIRECTORIO`), fuera de la base de datos, y solo se descargan con `ms05:incident:read` |
| Identidad | MS05 guarda el UUID de los usuarios; los nombres se consultan a MS01 al responder |
| Credenciales | La contraseña de la base de datos y la clave de servicio llegan por variables de entorno |
| Errores | Las respuestas de error salen como `ErrorResponse`, sin trazas internas |

## Otras medidas

* **Validación en el servidor:** cada campo se valida con las reglas de [2_forms-inputs.md](2_forms-inputs.md).
* **Sin estado:** la cadena de seguridad es `STATELESS`, sin sesión HTTP, sin formulario de inicio de sesión, sin autenticación básica y sin CSRF, porque no usa cookies.
* **Esquema controlado:** el esquema lo crean los scripts de `database/` al crear el volumen de la base de datos, y Hibernate solo lo valida (`ddl-auto=validate`).
* **Superficie expuesta:** Actuator solo expone `/actuator/health`, sin detalles.
* **Análisis estático:** SpotBugs revisa el código en cada build y lo hace fallar con un hallazgo ([quality-and-tests.md](quality-and-tests.md)).
