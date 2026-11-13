# Especificación del Modelo de Datos

MS01 tiene su propia base de datos PostgreSQL ([RNF-MS01-01](1_requirements.md#requerimientos-no-funcionales-rnf)). Ningún otro microservicio la lee ni la modifica: solo se accede a ella mediante la API. Esquema desplegado: `database/` del repositorio de MS01. Diagrama: [modelo físico](diagramas/ms01_er_fisico.drawio.svg).

## Índice de Contenido
1. [Alcance](#alcance)
2. [Convenciones](#convenciones)
3. [Almacenamiento seguro](#almacenamiento-seguro)
4. [Tablas](#tablas)
    - [usuarios](#usuarios)
    - [roles](#roles)
    - [usuario_rol](#usuario_rol)
    - [codigos_recuperacion](#codigos_recuperacion)
5. [Relaciones](#relaciones)
6. [Correspondencia entre API y columnas](#correspondencia-entre-api-y-columnas)
7. [Reglas de integridad](#reglas-de-integridad)
8. [Operaciones transaccionales y consultas](#operaciones-transaccionales-y-consultas)
9. [Configuración](#configuración)
10. [Datos iniciales](#datos-iniciales)

---

## Alcance

**Se almacena:** usuarios, los roles (con su descripción y los permisos de cada uno), la asignación de roles a usuarios, las marcas de corte de sesión y los códigos de recuperación de contraseña.

**No se almacena:**
* **Auditoría:** la delega MS08 ([RNF-MS01-10](1_requirements.md#requerimientos-no-funcionales-rnf)).
* **El catálogo de permisos:** los códigos de permiso y sus descripciones están versionados en el código (enum `Permission`, solo lectura). La base solo guarda qué permisos tiene cada rol ([RN-MS01-14](1_requirements.md#rf-ms01-05-gestión-de-roles-y-permisos)).
* **Sesiones y tokens:** el token es stateless y no se persiste ([`3_jwt-spec.md`](3_jwt-spec.md)). Solo se guardan las marcas `valido_desde` que cortan sesiones ([RF-MS01-11](1_requirements.md#rf-ms01-11-corte-de-sesión-por-cambios-de-acceso)).
* **Datos de otros dominios** (alertas, check-ins, etc.): pertenecen a sus propios microservicios.

---

## Convenciones

* Tablas y columnas en `snake_case` y en español, alineadas con el contrato OpenAPI.
* Fechas y horas en `TIMESTAMPTZ`, guardadas en UTC.
* Los identificadores son UUID. La aplicación los genera al insertar, y las columnas también tienen `DEFAULT gen_random_uuid()`. El `id` del usuario es el mismo valor del `id` de la API, del claim `sub` y de `X-User-Id`.
* El correo se guarda en minúsculas y sin espacios en los extremos ([RG-01](2_forms-inputs.md#reglas-generales-de-entrada)).
* Los roles son filas de la tabla `roles` y se identifican por su nombre (texto en MAYÚSCULAS). `usuario_rol.rol` guarda ese nombre y la base de datos lo valida con una clave foránea, no con una lista fija: crear, renombrar o borrar un rol no exige cambiar código ni el esquema. Los permisos de cada rol son una lista de texto (`TEXT[]`) sin ninguna restricción sobre los valores: el catálogo de permisos cambia con el código, nunca con SQL.
* Las restricciones llevan prefijo por tipo (`pk_`, `fk_`, `uq_`, `chh_` para CHECK). Los CHECK de `roles` protegen el formato, el nombre reservado `DETAILS` y la regla de `ms01:role:manage`; los valores de los permisos los valida el servicio contra el catálogo.
* `valido_desde` guarda el instante de corte de sesión truncado a segundos (el mismo tipo de dato que el `iat` del token) y es `NULL` cuando no hay corte. Lo escribe MS01 con la hora de la base de datos, la misma con la que fija el `iat` de las credenciales ([corte de sesión](3_jwt-spec.md#corte-de-sesión)), así que todas las instancias de MS01 comparten un solo reloj.
* Los usuarios no se borran físicamente: se inactivan ([RN-MS01-17](1_requirements.md#rf-ms01-07-administración-de-usuarios)).
* El esquema se versiona como scripts SQL numerados en `database/` (`001_create_schema.sql`, `002_index_schema.sql`, `003_codigos_recuperacion.sql`). Las bases nuevas se crean con esos tres. `database/migraciones/004_migracion_roles_permisos.sql` es el script manual, de una sola transacción, que lleva una base ya desplegada al esquema nuevo (tabla `roles`, columnas `valido_desde`, clave foránea); no se usa en bases nuevas y por eso está en la subcarpeta `migraciones/`, que Postgres no ejecuta al crear una base. La aplicación no crea ni modifica el esquema: lo valida al arrancar (`spring.jpa.hibernate.ddl-auto: validate`).

---

## Almacenamiento seguro

Detalle técnico de [RN-MS01-04](1_requirements.md#rf-ms01-01-registrar-usuario-institucional):
* **Contraseñas:** solo se guarda su hash BCrypt (`usuarios.password_hash`). La contraseña en claro nunca se guarda, no se registra en logs y nunca se devuelve.
* **Códigos de recuperación:** solo se guarda su hash BCrypt (`codigos_recuperacion.codigo_hash`). El código en claro solo viaja en el correo al usuario.

---

## Tablas

### usuarios
Una fila por persona registrada.

| Columna | Tipo | Nulo | Por defecto | Restricciones | Descripción |
|---|---|---|---|---|---|
| `id` | UUID | No | `gen_random_uuid()` | PK | Identificador del usuario. |
| `primer_nombre` | VARCHAR(30) | No | — | CHECK 2 a 30 caracteres | [F-01](2_forms-inputs.md#f-01-registro-de-usuario). |
| `segundo_nombre` | VARCHAR(30) | Sí | NULL | CHECK 2 a 30 si existe | Opcional. |
| `primer_apellido` | VARCHAR(30) | No | — | CHECK 2 a 30 caracteres | |
| `segundo_apellido` | VARCHAR(30) | Sí | NULL | CHECK 2 a 30 si existe | Opcional. |
| `correo` | VARCHAR(100) | No | — | UNIQUE (`uq_users_correo`); CHECK `correo = lower(correo)` | Correo institucional. No cambia después del registro ([RN-MS01-15](1_requirements.md#rf-ms01-06-gestión-del-perfil)). |
| `password_hash` | VARCHAR(100) | No | — | — | Ver [almacenamiento seguro](#almacenamiento-seguro). |
| `estado` | VARCHAR(10) | No | `'ACTIVO'` | CHECK IN (`ACTIVO`, `INACTIVO`) | [RN-MS01-17](1_requirements.md#rf-ms01-07-administración-de-usuarios). |
| `creado_en` | TIMESTAMPTZ | No | `CURRENT_TIMESTAMP` | — | Fecha de registro. |
| `actualizado_en` | TIMESTAMPTZ | No | `CURRENT_TIMESTAMP` | — | Se actualiza en cada modificación del usuario (perfil, estado, contraseña). |
| `valido_desde` | TIMESTAMPTZ | Sí | NULL | — | Corte de sesión del usuario: las credenciales con `iat` menor se rechazan. Lo escribe MS01 al cambiar sus roles o su estado, o al renombrar o borrar uno de sus roles ([RN-MS01-35](1_requirements.md#rf-ms01-11-corte-de-sesión-por-cambios-de-acceso)). NULL = sin corte. Truncado a segundos. No se expone en la API. |

**Índices:** PK `id`; único `correo`; `idx_usuarios_correo_trgm` (GIN `pg_trgm`, para la búsqueda parcial por correo); `idx_usuarios_estado (estado)`; `idx_usuarios_valido_desde (valido_desde) WHERE valido_desde IS NOT NULL` (parcial: solo indexa los usuarios con corte, que son pocos, para armar la instantánea de cortes).

### roles
Una fila por rol. Es la fuente de los roles y de los permisos de cada rol ([RF-MS01-10](1_requirements.md#rf-ms01-10-administración-de-roles-y-catálogo-de-permisos)).

| Columna | Tipo | Nulo | Por defecto | Restricciones | Descripción |
|---|---|---|---|---|---|
| `nombre` | VARCHAR(30) | No | — | PK; CHECK `chh_role_name` (`nombre ~ '^[A-Z][A-Z0-9_]{1,29}$'`); CHECK `chh_role_name_reserved` (`nombre <> 'DETAILS'`) | Nombre del rol y clave. 2 a 30 caracteres: mayúsculas, dígitos y guion bajo, empezando con una letra. No puede ser `DETAILS`, nombre reservado de la ruta `GET /roles/details`. El servicio recorta los espacios y lo pasa a mayúsculas antes de guardarlo ([RG-08](2_forms-inputs.md#reglas-generales-de-entrada)); la base solo acepta el valor ya normalizado. Es lo que viaja en el claim `roles` y en `X-User-Role`. Puede cambiar (salvo en los roles de sistema); `usuario_rol` lo sigue por la clave foránea. |
| `descripcion` | VARCHAR(200) | No | — | CHECK `chh_role_description` (3 a 200 caracteres) | Texto que muestra la interfaz de administración. Editable. |
| `permisos` | TEXT[] | No | — | CHECK `chh_role_permissions` (`cardinality(permisos) >= 1`); CHECK `chh_role_manage_only_admin` | Códigos de permiso del rol (p. ej. `ms04:checkin:write`). Sin repetidos y ordenados al guardar. Ver "Por qué `permisos` es un arreglo". |
| `es_sistema` | BOOLEAN | No | `FALSE` | Trigger `trg_protect_system_roles` | `TRUE` solo en `ESTUDIANTE` y `ADMIN`. Solo lo escribe la siembra; ninguna petición de la API lo recibe. |
| `valido_desde` | TIMESTAMPTZ | Sí | NULL | — | Corte de sesión del rol: las credenciales con `iat` menor, de usuarios que tienen este rol, se rechazan. Solo cambia cuando cambian los permisos del rol ([RN-MS01-33](1_requirements.md#rf-ms01-10-administración-de-roles-y-catálogo-de-permisos)). NULL = sin corte. Truncado a segundos. |
| `creado_en` | TIMESTAMPTZ | No | `CURRENT_TIMESTAMP` | — | Fecha de creación. |
| `actualizado_en` | TIMESTAMPTZ | No | `CURRENT_TIMESTAMP` | — | Se actualiza en cada modificación del rol. |

**Índices:** PK `nombre`; `idx_roles_valido_desde (valido_desde) WHERE valido_desde IS NOT NULL` (parcial, para la instantánea de cortes).

**Restricciones, una por una:**

| Restricción | Qué garantiza | Regla |
|---|---|---|
| `chh_role_name` | El formato del nombre | [RN-MS01-28](1_requirements.md#rf-ms01-10-administración-de-roles-y-catálogo-de-permisos) |
| `chh_role_name_reserved` | Que ningún rol se llame `DETAILS`, el segmento fijo de la ruta `GET /roles/details` | [RN-MS01-28](1_requirements.md#rf-ms01-10-administración-de-roles-y-catálogo-de-permisos) |
| `chh_role_description` | La longitud de la descripción | [RN-MS01-28](1_requirements.md#rf-ms01-10-administración-de-roles-y-catálogo-de-permisos) |
| `chh_role_permissions` | Que el rol tenga al menos un permiso. `NOT NULL` no basta, porque un arreglo vacío `{}` no es nulo | [RN-MS01-28](1_requirements.md#rf-ms01-10-administración-de-roles-y-catálogo-de-permisos) |
| `chh_role_manage_only_admin` | `(nombre = 'ADMIN') = ('ms01:role:manage' = ANY(permisos))`: `ms01:role:manage` está en `ADMIN` y en ningún otro rol, y `ADMIN` nunca lo pierde | [RN-MS01-30](1_requirements.md#rf-ms01-10-administración-de-roles-y-catálogo-de-permisos) |
| `trg_protect_system_roles` | Un rol con `es_sistema = TRUE` no se borra, no se renombra y no cambia su marca. El trigger (`BEFORE UPDATE OF nombre, es_sistema OR DELETE`, función `protect_system_roles()`) lanza un error con SQLSTATE `23S01` | [RN-MS01-29](1_requirements.md#rf-ms01-10-administración-de-roles-y-catálogo-de-permisos) |

**Por qué `permisos` es un arreglo (`TEXT[]`) y no una tabla:** una lista corta de códigos se lee y se escribe siempre completa y junta con el rol, y el token y las cabeceras la usan tal cual. No hay consultas por permiso individual, así que una tabla `rol_permiso` solo agregaría uniones sin beneficio.

**Por qué no hay un CHECK sobre los valores de los permisos:** el catálogo de permisos vive en el código (enum `Permission`) y crece con cada microservicio. Un CHECK con la lista obligaría a cambiar el esquema en cada permiso nuevo, y es justo lo que se quiere evitar. El servicio valida cada código contra el catálogo antes de guardar; si un código guardado deja de existir en el código, MS01 lo ignora al emitir el token y al responder, con un aviso en el log.

**Por qué hay una columna `es_sistema` además del trigger:** la columna dice qué roles son de sistema (y permite que la interfaz deshabilite el borrado); el trigger lo hace cumplir aunque alguien escriba directamente en la base. El servicio también lo verifica primero, para responder con un error claro (409) en vez del error de la base.

**Roles de sistema:** `ESTUDIANTE` (rol inicial de todo registro y rol de respaldo al borrar un rol) y `ADMIN` (único dueño de `ms01:role:manage`).

### usuario_rol
Roles de cada usuario, por el nombre del rol (`roles.nombre`). Un usuario puede tener varios, hasta 50 ([RN-MS01-13 y RN-MS01-40](1_requirements.md#rf-ms01-05-gestión-de-roles-y-permisos)); el tope lo verifica el servicio, la base de datos no lo restringe.

| Columna | Tipo | Nulo | Por defecto | Restricciones | Descripción |
|---|---|---|---|---|---|
| `usuario_id` | UUID | No | — | PK (`pk_rol`, con `rol`); FK → `usuarios.id` (`fk_rol`) | |
| `rol` | VARCHAR(30) | No | — | PK (con `usuario_id`); FK → `roles.nombre` (`fk_usuario_rol_roles`, `ON UPDATE CASCADE`, sin `ON DELETE`) | Nombre del rol. Ya no hay una lista fija de ocho nombres: la clave foránea exige que el rol exista. Renombrar un rol actualiza estas filas solas; borrar un rol que todavía tiene usuarios la base lo rechaza, por eso el servicio reasigna antes de borrar. |
| `asignado_en` | TIMESTAMPTZ | No | `CURRENT_TIMESTAMP` | — | Fecha de asignación. |

**Índices:** la PK compuesta (`usuario_id`, `rol`), que impide roles repetidos; `idx_usuario_rol_rol (rol, usuario_id)` para el filtro por rol.

### codigos_recuperacion
Códigos de un solo uso para restablecer la contraseña ([RF-MS01-08](1_requirements.md#rf-ms01-08-recuperación-y-cambio-de-contraseña)).

| Columna | Tipo | Nulo | Por defecto | Restricciones | Descripción |
|---|---|---|---|---|---|
| `id` | UUID | No | `gen_random_uuid()` | PK | |
| `usuario_id` | UUID | No | — | FK → `usuarios.id` (`fk_codigos_usuario`) | Dueño del código. |
| `codigo_hash` | VARCHAR(100) | No | — | — | Ver [almacenamiento seguro](#almacenamiento-seguro). |
| `intentos` | INTEGER | No | `0` | CHECK `intentos >= 0` | Intentos fallidos con este código. |
| `expira_en` | TIMESTAMPTZ | No | — | — | Creación + vigencia ([configuración](#configuración)). |
| `usado_en` | TIMESTAMPTZ | Sí | NULL | — | NULL = no usado. Con valor = consumido o invalidado. |
| `creado_en` | TIMESTAMPTZ | No | `CURRENT_TIMESTAMP` | — | También mide la espera entre solicitudes. |

**Índices:** PK `id`; `idx_codigos_usuario (usuario_id, expira_en)`.
**Vigente** = `usado_en IS NULL` y `expira_en > now()`.

---

## Relaciones

| Relación | Cardinalidad | Borrado |
|---|---|---|
| `usuarios` → `usuario_rol` | 1 a N (mínimo 1 fila por usuario, validado en el servicio) | Sin cascada (no hay borrado de usuarios) |
| `roles` → `usuario_rol` | 1 a N | Restringido: un rol con usuarios no se borra. Renombrar propaga (`ON UPDATE CASCADE`) |
| `usuarios` → `codigos_recuperacion` | 1 a N (a lo sumo uno sin usar a la vez) | Sin cascada |

---

## Correspondencia entre API y columnas

Los campos del contrato tienen el mismo nombre que su columna en `usuarios` (`id`, nombres y apellidos, `correo`, `estado`), salvo estos:

| Campo del contrato | Columna | Notas |
|---|---|---|
| `contrasena`, `nueva_contrasena` | `usuarios.password_hash` | Solo entra. No se devuelve. |
| `roles` (de un usuario) | `usuario_rol.rol` (filas del usuario) | Arreglo en orden alfabético (igual que en el token). |
| `rol` (en los roles) | `roles.nombre` | Nombre del rol. |
| `descripcion` | `roles.descripcion` | |
| `permisos` | `roles.permisos` | Arreglo de códigos de permiso, ordenado y sin repetidos. |
| `es_sistema` | `roles.es_sistema` | Solo se devuelve; nunca se recibe. |
| `permissions` (token) | `roles.permisos` de todos los roles del usuario | Unión ordenada y sin repetidos; se calcula al emitir el token. |
| `valido_desde` | `usuarios.valido_desde` y `roles.valido_desde` | Solo viaja en la ruta interna de cortes de sesión (usuarios y roles con corte), nunca en la API pública. |
| `codigo` | `codigos_recuperacion.codigo_hash` | Se compara contra el hash. No se devuelve. |
| `expira_en` (login) | — | Sale del claim `exp` del token, no de la BD. |

---

## Reglas de integridad

| Regla | Dónde se garantiza |
|---|---|
| [RN-MS01-01](1_requirements.md#rf-ms01-01-registrar-usuario-institucional) correo único | BD: UNIQUE + `correo = lower(correo)`. El servicio normaliza el correo antes de guardar. |
| [RN-MS01-17](1_requirements.md#rf-ms01-07-administración-de-usuarios) estados válidos | BD: CHECK. |
| [RN-MS01-14](1_requirements.md#rf-ms01-05-gestión-de-roles-y-permisos) roles existentes, sin repetir | BD: FK `fk_usuario_rol_roles` y PK compuesta. El catálogo de permisos es el enum `Permission`. |
| [RN-MS01-40](1_requirements.md#rf-ms01-05-gestión-de-roles-y-permisos) máximo de 50 roles por usuario | Servicio: calcula los roles que cada usuario tendría después del cambio y, si alguno supera 50, rechaza toda la petición con 422 antes de escribir. La BD no lo restringe. |
| [RN-MS01-28](1_requirements.md#rf-ms01-10-administración-de-roles-y-catálogo-de-permisos) formato del nombre, nombre reservado, descripción y permisos de un rol | BD: `chh_role_name`, `chh_role_name_reserved`, `chh_role_description`, `chh_role_permissions`. Servicio: que cada permiso exista en el catálogo (la BD no lo valida) y que el nombre sea único (clave primaria). |
| [RN-MS01-29](1_requirements.md#rf-ms01-10-administración-de-roles-y-catálogo-de-permisos) roles de sistema | BD: trigger `trg_protect_system_roles`. Servicio: lo verifica primero y responde 409. La siembra es la única que escribe `es_sistema`. |
| [RN-MS01-30](1_requirements.md#rf-ms01-10-administración-de-roles-y-catálogo-de-permisos) `ms01:role:manage` solo en ADMIN | BD: `chh_role_manage_only_admin`. Servicio: lo verifica primero y responde 422. |
| [RN-MS01-31](1_requirements.md#rf-ms01-10-administración-de-roles-y-catálogo-de-permisos) borrar un rol sin dejar usuarios sin rol | BD: FK sin `ON DELETE`. Servicio: reasigna a ESTUDIANTE o retira el rol y borra, todo en una transacción. |
| [RN-MS01-35](1_requirements.md#rf-ms01-11-corte-de-sesión-por-cambios-de-acceso) cortes de sesión | Servicio: escribe `valido_desde` en la misma transacción del cambio. BD: solo los índices parciales. |
| Al menos un rol por usuario; rol inicial ESTUDIANTE ([RN-MS01-02](1_requirements.md#rf-ms01-01-registrar-usuario-institucional)) | Servicio, dentro de la transacción. |
| Nadie cambia sus propios roles ni su estado ([RN-MS01-12](1_requirements.md#rf-ms01-05-gestión-de-roles-y-permisos), [RN-MS01-19](1_requirements.md#rf-ms01-07-administración-de-usuarios)) | Servicio: compara el id del solicitante con los ids objetivo. |
| [RN-MS01-21](1_requirements.md#rf-ms01-08-recuperación-y-cambio-de-contraseña) un solo código sin usar y un solo uso | Servicio: un código nuevo marca `usado_en` en los anteriores; el restablecimiento marca el consumido. |
| [RN-MS01-03](1_requirements.md#rf-ms01-01-registrar-usuario-institucional) política de contraseña | Servicio, antes de calcular el hash. La BD solo ve el hash. |

---

## Operaciones transaccionales y consultas

Cada operación de escritura es una sola transacción: se aplica completa o no se aplica.

| Operación | Lecturas | Escrituras |
|---|---|---|
| Registro ([RF-01](1_requirements.md#rf-ms01-01-registrar-usuario-institucional)) | ¿Existe el `correo`? | INSERT `usuarios` (ACTIVO) + INSERT `usuario_rol` (ESTUDIANTE) |
| Login ([RF-02](1_requirements.md#rf-ms01-02-autenticar-usuario-login)) | `usuarios` por `correo` + sus roles + los `permisos` de esos roles | Ninguna |
| Actualizar perfil ([RF-06](1_requirements.md#rf-ms01-06-gestión-del-perfil)) | `usuarios` por id | UPDATE de los 4 campos de nombre + `actualizado_en` |
| Cambio de estado en lote ([RF-07](1_requirements.md#rf-ms01-07-administración-de-usuarios)) | Usuarios objetivo con bloqueo de escritura | UPDATE `estado` y `valido_desde` de cada usuario |
| Cambio de roles en lote ([RF-05](1_requirements.md#rf-ms01-05-gestión-de-roles-y-permisos)) | Usuarios objetivo con bloqueo de escritura | Actualiza las filas de `usuario_rol` de los usuarios cuyos roles cambian (`agregar` / `retirar`) y su `valido_desde` |
| Solicitar código ([RF-08](1_requirements.md#rf-ms01-08-recuperación-y-cambio-de-contraseña)) | `usuarios` por `correo`; último código del usuario (espera) | Si está ACTIVO y fuera de la espera: marca `usado_en` en los códigos sin usar + INSERT del nuevo |
| Restablecer contraseña | Último código sin usar, con bloqueo de escritura | Código incorrecto: `intentos + 1` (y `usado_en` al llegar al máximo). Correcto: UPDATE `password_hash` + `usado_en` del código y de los demás sin usar |
| Cambiar contraseña autenticado | `usuarios` por id | UPDATE `password_hash` |
| Consultar roles y permisos ([RF-05](1_requirements.md#rf-ms01-05-gestión-de-roles-y-permisos), [RF-10](1_requirements.md#rf-ms01-10-administración-de-roles-y-catálogo-de-permisos)) | `roles` (uno o todos) | Ninguna |
| Crear rol ([RF-10](1_requirements.md#rf-ms01-10-administración-de-roles-y-catálogo-de-permisos)) | ¿Existe el `nombre`? | INSERT `roles` (`es_sistema` falso, `valido_desde` nulo) |
| Renombrar o describir un rol ([RF-10](1_requirements.md#rf-ms01-10-administración-de-roles-y-catálogo-de-permisos)) | `roles` por `nombre`; ¿existe el nombre nuevo? | UPDATE `roles` (`nombre`, `descripcion`, `actualizado_en`); si el nombre cambia, `usuario_rol.rol` se actualiza por cascada y se escribe `valido_desde` de cada usuario con ese rol |
| Cambiar permisos de un rol ([RF-10](1_requirements.md#rf-ms01-10-administración-de-roles-y-catálogo-de-permisos)) | `roles` por `nombre` | UPDATE `roles` (`permisos`, `valido_desde`, `actualizado_en`) |
| Borrar un rol ([RF-10](1_requirements.md#rf-ms01-10-administración-de-roles-y-catálogo-de-permisos)) | `roles` por `nombre`; usuarios que lo tienen | Por cada usuario: UPDATE `usuario_rol` a ESTUDIANTE (si era su único rol) o DELETE de esa fila (si tenía varios) y UPDATE `usuarios.valido_desde`; luego DELETE `roles`. Una sola transacción: si la BD rechaza algo, se revierte todo |
| Consulta interna de cortes de sesión ([RF-11](1_requirements.md#rf-ms01-11-corte-de-sesión-por-cambios-de-acceso)) | `usuarios` y `roles` con `valido_desde` dentro de la ventana de 3700 s | Ninguna |
| Consulta interna por ids ([RF-09](1_requirements.md#rf-ms01-09-resolución-interna-de-usuarios)) | `usuarios` y sus roles con `id` en la lista (hasta 100) | Ninguna |
| Consulta interna por rol ([RF-09](1_requirements.md#rf-ms01-09-resolución-interna-de-usuarios)) | `usuarios` con una fila del rol en `usuario_rol` y, si hay fecha de corte, `creado_en <= corte`; orden `id` ascendente | Ninguna |

**Listado `GET /users`** (parámetros en [F-07](2_forms-inputs.md#f-07-búsqueda-y-filtros-de-usuarios)):
* `correo`: coincidencia parcial sin distinguir mayúsculas (`LIKE '%texto%'` sobre el correo en minúsculas).
* `estado` / `rol`: igualdad; `rol` busca usuarios con una fila de ese rol en `usuario_rol`.
* Orden: `creado_en` descendente y luego `id`.
* Cada elemento incluye todos sus roles, no solo el rol filtrado.

---

## Configuración

Variables de entorno que afectan los datos (valores por defecto de `application.yaml`):

| Variable | Por defecto | Uso |
|---|---|---|
| `INSTITUTIONAL_EMAIL_DOMAINS` | `escuelaing.edu.co,mail.escuelaing.edu.co` | Dominios permitidos en el registro. |
| `RECOVERY_CODE_TTL_MINUTES` | `15` | Vigencia del código ([RN-MS01-21](1_requirements.md#rf-ms01-08-recuperación-y-cambio-de-contraseña)). |
| `RECOVERY_MAX_ATTEMPTS` | `5` | Intentos fallidos permitidos antes de invalidar el código. |
| `RECOVERY_RESEND_COOLDOWN_SECONDS` | `60` | Espera mínima entre dos códigos del mismo usuario (`0` la desactiva). |

---

## Datos iniciales

1. **Roles iniciales** ([RN-MS01-34](1_requirements.md#rf-ms01-10-administración-de-roles-y-catálogo-de-permisos)): los scripts `001` (bases nuevas) y `004` (base desplegada) insertan los ocho roles con la asignación de permisos de la [Matriz Rol‐Permiso](https://github.com/CampusSeguro-DOSW-2026-2/documentation/wiki/Matriz-Rol%E2%80%90Permiso). Es una siembra única: después de cargada, la fuente es la tabla `roles` y el administrador puede cambiar descripciones y permisos. Solo `ESTUDIANTE` y `ADMIN` se siembran con `es_sistema = TRUE`.

   | Rol | `es_sistema` | Permisos | Descripción inicial |
   |---|---|---|---|
   | `ESTUDIANTE` | Sí | 15 | Estudiante de la comunidad universitaria |
   | `PROFESOR` | No | 16 | Profesor de la comunidad universitaria |
   | `ADMINISTRATIVO` | No | 15 | Personal administrativo de la comunidad universitaria |
   | `BRIGADISTA` | No | 23 | Brigadista de emergencias del campus |
   | `COORDINADOR` | No | 34 | Coordinador de emergencias: gestiona alertas, brigadistas y asistencia |
   | `ORGANIZADOR` | No | 25 | Organizador de simulacros y eventos de evacuación |
   | `ADMIN` | Sí | 21 | Administrador del sistema: gestiona usuarios, roles, campus y contenidos |
   | `AUDITOR` | No | 17 | Auditor: consulta históricos, indicadores y auditoría (solo lectura) |

   Los códigos de permiso de cada rol están en el script y en la matriz de la wiki (48 permisos en el catálogo). `ms01:role:manage` queda solo en `ADMIN`.
2. **Primer administrador** ([RN-MS01-12](1_requirements.md#rf-ms01-05-gestión-de-roles-y-permisos)): se crea al arrancar solo si no hay ningún usuario con el rol ADMIN y están definidas `BOOTSTRAP_ADMIN_EMAIL` y `BOOTSTRAP_ADMIN_PASSWORD` (nunca en el repositorio). Queda ACTIVO, con el rol ADMIN. `BOOTSTRAP_ADMIN_FIRST_NAME` y `BOOTSTRAP_ADMIN_LAST_NAME` son opcionales (por defecto "Administrador" e "Inicial"). Si el correo ya pertenece a otro usuario, no se crea ni se le asciende. Como no pasa por el registro, no se le aplican la política de contraseña ([RN-MS01-03](1_requirements.md#rf-ms01-01-registrar-usuario-institucional)) ni el dominio institucional: quien configura las variables debe usar un correo institucional y una contraseña que los cumpla.
3. **Limpieza opcional:** una tarea periódica puede eliminar los códigos usados o vencidos con más de 24 horas de antigüedad (no implementada).
