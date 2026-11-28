# Seguridad — MS03 Alertas y emergencias

Cómo MS03 recibe la identidad del usuario, cómo autoriza cada operación, cómo llama a otros microservicios y al proveedor de correo, y cómo protege los datos de los destinatarios. Los valores transversales están en los [Estándares de Integración](../integration/standards.md); cómo se forma la credencial de acceso, en la [seguridad de MS01](../ms01-identity-and-access/security.md#formación-del-jwt); y lo que hace el Gateway con ella, en la [seguridad del Gateway](../api-gateway/security.md).

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

MS03 no registra usuarios, no autentica credenciales y nunca recibe ni valida el JWT. MS01 emite el token y el Gateway lo valida y lo convierte en cabeceras ([validación del JWT](../api-gateway/security.md#validación-del-jwt)). MS03 solo lee esas cabeceras.

## Identidad del usuario

`IdentityHeadersAuthenticationFilter` construye la identidad de la petición con las cabeceras que inyecta el Gateway:

| Cabecera | Validación en MS03 | Si falta o es inválida |
|---|---|---|
| `X-User-Id` | UUID | `401 NO_AUTENTICADO` con `La cabecera X-User-Id es obligatoria` o `La cabecera X-User-Id no tiene un formato válido` |
| `X-User-Role` | Lista separada por coma con al menos un rol, de longitud acotada | `401 NO_AUTENTICADO` con el mensaje de la cabecera |
| `X-User-Permission` | Lista separada por coma con al menos un permiso, de longitud acotada | `401 NO_AUTENTICADO` con el mensaje de la cabecera |
| `X-Service-Name` | Opcional. Cuando otro microservicio llama en nombre de un usuario (MS06 al crear o activar las alertas de un evento), el Gateway la inyecta | No autoriza: se guarda en la auditoría como `<servicio>:<usuario>` |

Las únicas rutas sin identidad son `/actuator/health`, `/actuator/info` y `/error`.

## Autorización por permiso

Cada operación exige un permiso de la [matriz de roles y permisos](../integration/permissions-matrix.md) (`@PreAuthorize`); si falta, responde `403 PROHIBIDO`:

| Permiso | Operaciones |
|---|---|
| `ms03:alert:manage` | Crear, editar, programar, activar y archivar cualquier alerta; ver los tipos de alerta |
| `ms03:alert:manage_drill` | Las mismas operaciones, solo sobre alertas de simulacro |
| `ms03:alert:read_history` | Listar todas las alertas, sus filtros y su auditoría, y ver cualquier alerta |
| `ms03:alert:receive` | Bandeja propia (`/mias`), ver una alerta de la que es destinatario, confirmarla y ver los tipos de alerta |

Además del permiso, `AlertAccessPolicy` comprueba el destinatario: sin `ms03:alert:read_history`, un usuario solo ve o confirma una alerta si es destinatario directo, si uno de sus roles es destinatario o si recibió la notificación. Si no, MS03 responde con error sin mostrar la alerta. `X-User-Role` solo se usa para esa comprobación de destinatarios, nunca para autorizar una operación.

## Llamadas salientes y clave de servicio

MS03 nunca llama directamente a otro microservicio: sus llamadas van al Gateway con su clave de servicio (`MS_SERVICE_KEY`) en `X-Service-Key` y el `X-Correlation-ID` de la operación, con tiempos máximos de 2 s para conectar y para leer (5 s de lectura para la auditoría):

| Destino | Ruta | Identidad que envía |
|---|---|---|
| MS01 | `POST /api/v1/users/batch`, `GET /api/v1/users/by-role` (correo y nombre de los destinatarios) | Ruta interna: MS01 autoriza por `X-Service-Name: ms03` |
| MS02 | `GET /api/v1/buildings/{id}`, `GET /api/v1/safe-zones/{id}` (validar el alcance de la alerta) | `X-User-Id`, `X-User-Role` y `X-User-Permission` del usuario que crea o edita la alerta |
| MS06 | `GET /api/v1/tipos-alerta` (sincronizar el catálogo de tipos) | Una identidad de sistema configurable (`MS06_SYSTEM_USER_ID`, `MS06_SYSTEM_USER_ROLE`, `MS06_SYSTEM_USER_PERMISSIONS`) |
| MS08 | `POST /api/v1/audit/events` | El usuario de la acción |
| Brevo | API de Brevo (`BREVO_API_KEY`) o SMTP con STARTTLS obligatorio (`MAIL_HOST`, `MAIL_USERNAME`, `MAIL_PASSWORD`) | Credencial del proveedor de correo |

La auditoría no se envía dentro de la operación: se guarda en una tabla de salida (*outbox*) y un proceso la despacha a MS08 cada 5 s, con reintentos y espera creciente, así que una caída de MS08 no afecta al usuario.

## CORS y mTLS

* **CORS:** MS03 no configura CORS. El navegador nunca llama a MS03: la única política CORS es la del Gateway ([CORS](../api-gateway/security.md#cors)).
* **Aislamiento de red:** MS03 confía en `X-User-*` y `X-Service-Name` porque solo el Gateway puede llegar a él. En la nube, el firewall de su VM solo acepta la IP del Gateway y su Caddy de borde exige el certificado de cliente del Gateway (mTLS), firmado por la CA privada del entorno ([aislamiento en la nube](../api-gateway/contracts.md#7-aislamiento-en-la-nube)). Su base de datos PostgreSQL corre en el mismo stack y solo es visible dentro de él.

## Datos sensibles

| Dato | Tratamiento |
|---|---|
| Correo de los destinatarios | MS03 lo obtiene de MS01 al notificar y lo guarda junto a cada entrega de notificación para el reintento y la trazabilidad del envío |
| Credenciales | La clave de servicio, la clave de la API de Brevo, la contraseña SMTP y las credenciales de la base de datos llegan por variables de entorno, nunca en el repositorio |
| Logs | Cada línea lleva el Correlation ID. Los logs del envío de correo registran cantidades y códigos de respuesta del proveedor, no direcciones ni contenido |
| Errores | Todo error, incluidos los del framework, sale como `ErrorResponse` sin trazas ni detalles internos |

## Otras medidas

* **Validación en el servidor:** cada campo se valida con las reglas de [2_forms-inputs.md](2_forms-inputs.md).
* **Sin estado:** la cadena de seguridad es `STATELESS`, sin sesión HTTP, sin formulario de inicio de sesión, sin autenticación básica y sin CSRF, porque no usa cookies.
* **Esquema controlado:** Flyway crea el esquema y Hibernate solo lo valida (`ddl-auto: validate`).
* **Superficie expuesta:** Actuator solo expone `/actuator/health`, sin detalles. Swagger UI y la especificación OpenAPI en tiempo de ejecución están desactivados salvo que se active `SPRINGDOC_ENABLED`.
* **Análisis estático:** SpotBugs revisa el código en cada build y lo hace fallar con un hallazgo ([quality-and-tests.md](quality-and-tests.md)).
