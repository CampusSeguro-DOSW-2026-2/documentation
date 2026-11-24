# MS01 — Identidad y acceso: guía de integración

Lo que los demás squads (MS02–MS08) y el Gateway necesitan de MS01. Las reglas generales (JWT, cabeceras, errores) están en los [Estándares](../integration/standards.md) y no se repiten aquí.

**Documentación completa y contrato:** [README de MS01](README.md) · [`openapi.yml`](api/openapi.yml)

## 1. Qué recibe su microservicio

MS01 emite el JWT al hacer login. El Gateway lo valida y su servicio **nunca lo ve**: solo recibe estas cabeceras:

| Cabecera | Ejemplo | Uso en su servicio |
|---|---|---|
| `X-User-Id` | `3f6c2a9e-8b1d-4c7a-9e2f-1a5b7c9d0e42` | UUID del usuario. Guárdelo como referencia (p. ej. quién hizo el check-in). |
| `X-User-Role` | `BRIGADISTA,ESTUDIANTE` | Roles separados por coma. Informativa: no autorice por el nombre del rol, porque los roles ya no son un conjunto fijo. |
| `X-User-Permission` | `ms04:checkin:write,ms07:content:read` | Permisos separados por coma: la unión de los permisos de todos los roles del usuario. **Autorice con esta cabecera**, contra el `x-permiso` de su operación. Los códigos y su asignación inicial a los roles están en la [matriz de roles y permisos](../integration/permissions-matrix.md). |
| `X-Correlation-ID` | `c19b472d-7a2a-4d3b-a12b-abcdef123456` | Inclúyalo en sus logs, errores y llamadas salientes. |

* Si faltan `X-User-Id`, `X-User-Role` o `X-User-Permission` en un endpoint protegido, responda `401 NO_AUTENTICADO`. Si el usuario no tiene el permiso que exige la operación, responda `403 PROHIBIDO`.
* Un cambio de los roles de un usuario, de los permisos de un rol o del estado de una cuenta corta su sesión en 7 segundos como máximo: el Gateway responde `401 NO_AUTENTICADO` y el usuario inicia sesión de nuevo. Su servicio no tiene que hacer nada ni consultar a MS01 para autorizar. El logout y el cambio de contraseña no cortan la sesión.
* **Pruebas locales:** no necesita MS01 ni un JWT; envíe las tres cabeceras a mano desde Postman.

### Roles y permisos: lo que cambia para su servicio

* Los roles son datos de MS01: el administrador puede crear roles, cambiar sus permisos y borrar o renombrar los que no son de sistema (`ESTUDIANTE` y `ADMIN` no). No dependa de una lista fija de nombres de rol. Un usuario puede tener hasta 50 roles y `DETAILS` es un nombre reservado que ningún rol puede usar. Todo rol, también los que crea el administrador, tiene siempre `ms01:profile:read` y `ms01:session:logout`, y `ESTUDIANTE` y `ADMIN` además `ms01:profile:write`: no los exija como permisos que un rol pueda no tener.
* Un permiso es un código `ms<NN>:<recurso>:<acción>` del catálogo, que vive en la base de datos de MS01 y es de solo lectura: ninguna ruta lo modifica. Para un permiso nuevo de su servicio: agréguelo a la Matriz y a su `x-permiso`; el responsable de la base de datos lo suma al catálogo a partir de la Matriz, y quítelo de la Matriz cuando su servicio deje de exigirlo para que se retire. Después el administrador concede el permiso nuevo a los roles que correspondan.
* El catálogo vigente de permisos está en `GET /api/v1/permissions` y los roles con sus permisos en `GET /api/v1/roles/details`, ambos con `ms01:role:read_all`.
* Los roles que pueden recibir alertas (los que tienen `ms03:alert:receive`) están en `GET /api/v1/roles/alert-recipients`, con `ms03:alert:manage` o `ms03:alert:manage_drill`: solo devuelve el nombre y la descripción de cada rol. Es el selector de destinatarios de las alertas; no use una lista fija de roles.

## 2. Datos de usuario

No duplique datos personales en su base de datos: almacene solo el UUID (`X-User-Id`). Para obtener datos de otros usuarios desde su servicio use las rutas internas de MS01, a través del Gateway con su clave de servicio ([Estándares §5](../integration/standards.md)):

| Ruta | Uso |
|---|---|
| `POST /api/v1/users/batch` | Hasta 100 usuarios por id. Devuelve `usuarios` y `no_encontrados`. |
| `GET /api/v1/users/by-role?rol=…` | Usuarios con un rol, paginado (p. ej. todos los `BRIGADISTA`). |

* Devuelven activos e inactivos: filtre por `estado`.
* Su servicio debe estar autorizado en MS01 (`ms03`, `ms04` y `ms05`). Para agregar otro, pídalo al squad de MS01 y registre la llamada en los Estándares §5.
* Contrato: tag *Interno (servicios)* del [`openapi.yml`](api/openapi.yml).

## 3. Endpoints y auditoría

* **Registro:** la cuenta solo se crea cuando la persona confirma el código de 6 dígitos que MS01 envía a su correo ([RF-MS01-01](1_requirements.md#rf-ms01-01-registrar-usuario-institucional)). Para crear usuarios de prueba use correos institucionales que pueda leer.
* **Gestión de roles y catálogo de permisos:** [RF-MS01-10](1_requirements.md#rf-ms01-10-administración-de-roles-y-catálogo-de-permisos) (rutas `/api/v1/roles` y `/api/v1/permissions`). **Corte de sesión:** [RF-MS01-11](1_requirements.md#rf-ms01-11-corte-de-sesión-por-cambios-de-acceso).
* **Endpoints, acceso y permisos:** el [`openapi.yml`](api/openapi.yml) lista cada endpoint con su permiso (`x-permiso`), sus esquemas y sus errores.
* **Auditoría:** MS01 no expone endpoints de auditoría. Las acciones que emite a MS08 están en el [catálogo de auditoría](../integration/audit-catalog.md#ms01--identidad-y-acceso) ([RNF-MS01-10](1_requirements.md#requerimientos-no-funcionales-rnf)). La consulta de esos eventos la ofrece MS08.
