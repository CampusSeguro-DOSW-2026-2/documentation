# Seguridad — MS04 Evacuación y check-in

Cómo MS04 recibe la identidad del usuario y del servicio que llama, cómo autoriza cada operación, cómo llama a otros microservicios y cómo protege los datos de las personas durante un evento. Los valores transversales están en los [Estándares de Integración](../integration/standards.md); cómo se forma la credencial de acceso, en la [seguridad de MS01](../ms01-identity-and-access/security.md#formación-del-jwt); y lo que hace el Gateway con ella, en la [seguridad del Gateway](../api-gateway/security.md).

## Índice

1. [Registro, autenticación y JWT](#registro-autenticación-y-jwt)
2. [Identidad del usuario](#identidad-del-usuario)
3. [Autorización por permiso](#autorización-por-permiso)
4. [Rutas internas de MS06](#rutas-internas-de-ms06)
5. [Llamadas a otros microservicios y clave de servicio](#llamadas-a-otros-microservicios-y-clave-de-servicio)
6. [CORS y mTLS](#cors-y-mtls)
7. [Datos sensibles](#datos-sensibles)
8. [Otras medidas](#otras-medidas)

---

## Registro, autenticación y JWT

MS04 no registra usuarios, no autentica credenciales y nunca recibe ni valida el JWT. MS01 registra a las personas y emite el token RS256 con `sub`, `roles` y `permissions` ([formación del JWT](../ms01-identity-and-access/security.md#formación-del-jwt)); el Gateway lo valida, aplica los cortes de sesión y lo convierte en cabeceras ([validación del JWT](../api-gateway/security.md#validación-del-jwt)). MS04 solo lee esas cabeceras ([RNF-MS04-10](1_requirements.md#requerimientos-no-funcionales-rnf)).

## Identidad del usuario

En las rutas protegidas, el filtro de Spring Security de MS04 (`IdentityHeaderParser`) construye la identidad de la petición con las tres cabeceras que inyecta el Gateway:

| Cabecera | Validación en MS04 | Si falta o es inválida |
|---|---|---|
| `X-User-Id` | UUID | `401 NO_AUTENTICADO`, mensaje `La cabecera X-User-Id no es válida.` |
| `X-User-Role` | Lista separada por coma de nombres con el formato `^[A-Z][A-Z0-9_]{1,29}$`; se recortan los espacios y se ignoran los repetidos | `401 NO_AUTENTICADO`, mensaje `La cabecera X-User-Role no es válida.` |
| `X-User-Permission` | Lista separada por coma de códigos con el formato `ms<dos dígitos>:<recurso>:<acción>`; se recortan los espacios y se ignoran los repetidos | `401 NO_AUTENTICADO`, mensaje `La cabecera X-User-Permission no es válida.` |

Si falta cualquiera de las tres, MS04 responde `401 NO_AUTENTICADO` con el mensaje `Faltan las cabeceras de identidad (X-User-Id / X-User-Role / X-User-Permission).` La identidad nunca viaja en el cuerpo ni en los parámetros ([RN-MS04-06](1_requirements.md#rn-ms04-06)): el check-in siempre se registra a nombre de `X-User-Id`, así que nadie puede reportar el estado de otra persona.

## Autorización por permiso

Cada operación protegida exige un permiso de la [matriz de roles y permisos](../integration/permissions-matrix.md) contra `X-User-Permission` (`x-permiso` del [contrato](api/openapi.yml)); si falta, responde `403 PROHIBIDO`:

| Permiso | Operaciones |
|---|---|
| `ms04:checkin:write` | Registrar el propio check-in |
| `ms04:assistance:request` | Además del anterior, registrar un check-in `NEEDS_ASSISTANCE` (solicitud de asistencia) |
| `ms04:checkin:read_own` | Eventos activos que le aplican al usuario y su propio estado vigente |
| `ms04:dashboard:read` | Consulta de eventos, consolidado, personas, pendientes y asistencia |
| `ms04:history:read` | Histórico de check-ins y resultados del evento |

`X-User-Role` no autoriza: se usa para saber si un evento le aplica al usuario (su rol está entre los `rolesConvocados`). El alcance de las consultas también depende de los permisos: quien tiene `ms04:dashboard:read` sin `ms04:history:read` (el BRIGADISTA en la asignación inicial) solo ve lo que corresponde a las zonas que MS05 le asignó en ese evento y no recibe la lista de pendientes ([RN-MS04-14](1_requirements.md#rn-ms04-14)).

## Rutas internas de MS06

`PUT /chi/eventos/{eventoId}/estado` y `GET /chi/eventos/{eventoId}/checkins` las usa solo MS06 ([RN-MS04-23](1_requirements.md#rn-ms04-23)). MS04 las protege con una cadena de seguridad propia, que se elige por método y ruta:

1. MS06 llama al Gateway con su clave de servicio; el Gateway la valida, la elimina e inyecta `X-Service-Name: ms06`.
2. `InternalServiceAuthenticationFilter` lee `X-Service-Name` y lo compara, sin distinguir mayúsculas, con `INTERNAL_ALLOWED_SERVICES` (`ms06` por defecto).
3. Sin la cabecera responde `401 NO_AUTENTICADO` (`Falta la cabecera X-Service-Name.`); con otro servicio, `403 PROHIBIDO`.

Estas rutas no usan permisos de usuario. Una petición con JWT llega sin `X-Service-Name` y recibe `401`, porque el Gateway elimina cualquier `X-Service-Name` que envíe un cliente externo.

## Llamadas a otros microservicios y clave de servicio

MS04 nunca llama directamente a otro microservicio: todas sus llamadas salen al Gateway (`GATEWAY_URL`) con su clave de servicio (`GATEWAY_SERVICE_KEY`, de al menos 32 caracteres), el `X-Correlation-ID` de la petición original y un timeout de `GATEWAY_TIMEOUT_MS` (2000 ms):

| Destino | Ruta | Identidad que propaga |
|---|---|---|
| MS01 | `GET /api/v1/users/by-role`, `POST /api/v1/users/batch` (población objetivo y nombres de los participantes) | Ruta interna: MS01 autoriza por `X-Service-Name: ms04`. Si la consulta la hace un usuario, también propaga sus `X-User-*` |
| MS02 | `GET /api/v1/safe-zones/{id}` (punto de encuentro) | `X-User-Id`, `X-User-Role` y `X-User-Permission` del usuario que hace el check-in |
| MS05 | `POST /api/v1/incidentes/eventos/{eventoId}` (solicitud de asistencia) | Las tres `X-User-*` del usuario que pidió asistencia; MS05 la registra a su nombre |
| MS05 | `GET /api/v1/responsabilidades/mias` (zonas del brigadista) | Las tres `X-User-*` del brigadista |
| MS08 | `POST /api/v1/audit/events` | El usuario de la acción, si existe |

MS04 reenvía las cabeceras de identidad tal como las recibió, sin modificarlas. Si un destino no responde, MS04 responde con su propio `ErrorResponse` (`503 DEPENDENCIA_NO_DISPONIBLE`) y nunca reenvía el cuerpo de error de otro servicio. La solicitud de asistencia que MS05 no recibe queda pendiente y se reenvía periódicamente; el check-in se guarda igual ([RNF-MS04-03](1_requirements.md#requerimientos-no-funcionales-rnf)).

## CORS y mTLS

* **CORS:** MS04 no configura CORS. El navegador nunca llama a MS04: la única política CORS es la del Gateway ([CORS](../api-gateway/security.md#cors)).
* **Aislamiento de red:** MS04 confía en `X-User-*` y `X-Service-Name` porque solo el Gateway puede llegar a él. En local su puerto no se publica fuera de la red Docker. En la nube, el firewall de su VM solo acepta la IP del Gateway y su Caddy de borde exige el certificado de cliente del Gateway (mTLS), firmado por la CA privada del entorno ([aislamiento en la nube](../api-gateway/contracts.md#7-aislamiento-en-la-nube)).
* **Llamadas salientes:** van al Gateway por HTTPS con la clave de servicio de MS04.

## Datos sensibles

| Dato | Tratamiento |
|---|---|
| Identidad del usuario | MS04 solo guarda el UUID (`usuario_id`). Los nombres y el correo no se guardan: se piden a MS01 al consultar y solo viajan en la respuesta; si MS01 no responde, la lista se entrega con el nombre y el correo nulos |
| Identidad en las solicitudes de asistencia | Cada solicitud guarda los roles y los permisos de quien la pidió (`roles_solicitante`, `permisos_solicitante`), los mismos valores de sus cabeceras, para que el reenvío periódico a MS05 propague su identidad; no guarda el token |
| Ubicación | Un check-in `NEEDS_ASSISTANCE` guarda el tipo y el id de la ubicación en el campus (edificio, espacio o punto de encuentro de MS02), no coordenadas del dispositivo |
| Estado de las personas | Los check-ins solo se agregan: la base de datos rechaza con triggers todo `UPDATE`, `DELETE` y `TRUNCATE` sobre ellos ([RNF-MS04-04](1_requirements.md#requerimientos-no-funcionales-rnf)), y un evento `FINALIZADO` no admite escrituras ([RNF-MS04-05](1_requirements.md#requerimientos-no-funcionales-rnf)) |
| Credenciales de la base de datos y clave de servicio | Variables de entorno, nunca en el repositorio ([RNF-MS04-13](1_requirements.md#requerimientos-no-funcionales-rnf)) |
| Logs | Cada línea lleva el Correlation ID; no registran tokens, nombres ni correos ([RNF-MS04-12](1_requirements.md#requerimientos-no-funcionales-rnf)) |
| Errores | Todo error, incluidos los del framework, sale como `ErrorResponse` sin trazas ni detalles internos ([RNF-MS04-11](1_requirements.md#requerimientos-no-funcionales-rnf)) |

## Otras medidas

* **Validación en el servidor:** los cuerpos rechazan campos desconocidos (`fail-on-unknown-properties`) y cada campo se valida con las reglas de [2_forms-inputs.md](2_forms-inputs.md); el error indica el campo y su mensaje. El cuerpo que envía MS06 debe tener el mismo `eventoId` de la ruta.
* **Concurrencia:** cada check-in toma un bloqueo del usuario en el evento antes de leer su estado vigente, así que reportes simultáneos de la misma persona se serializan ([RNF-MS04-02](1_requirements.md#requerimientos-no-funcionales-rnf)).
* **Integridad en la base de datos:** restricciones `CHECK`, claves foráneas, la vista de estados vigentes y los triggers de solo inserción ([modelo de datos](3_data-model.md)). La aplicación no crea ni modifica el esquema (`ddl-auto: validate`).
* **Superficie expuesta:** Actuator solo expone `/actuator/health`, sin detalles; Swagger UI y la especificación OpenAPI en tiempo de ejecución están desactivados salvo que se active `SWAGGER_ENABLED` en desarrollo local.
* **Auditoría:** el registro y el cambio de estado de los check-ins, las solicitudes de asistencia y los cambios de estado de los eventos se emiten a MS08 con su resultado y el Correlation ID ([catálogo de auditoría](../integration/audit-catalog.md#ms04--evacuación-y-check-in)).
* **Análisis estático:** PMD, SpotBugs y el Quality Gate de SonarCloud revisan el código en cada cambio ([quality-and-tests.md](quality-and-tests.md)).
