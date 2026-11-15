# Especificación del Modelo de Datos

MS01 tiene su propia base de datos PostgreSQL ([RNF-MS01-01](1_requirements.md#requerimientos-no-funcionales-rnf)). Ningún otro microservicio la lee ni la modifica: solo se accede a ella mediante la API. Esquema desplegado: `database/` del repositorio de MS01. Diagrama: [modelo físico](diagramas/ms01_er_fisico.drawio.svg).

## Índice de Contenido
1. [Alcance](#alcance)
2. [Convenciones](#convenciones)
3. [Almacenamiento seguro](#almacenamiento-seguro)
4. [Tablas](#tablas)
    - [usuarios](#usuarios)
    - [roles](#roles)
    - [permisos](#permisos)
    - [rol_permiso](#rol_permiso)
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

**Se almacena:** usuarios, los roles (con su descripción), el catálogo de permisos, los permisos de cada rol, la asignación de roles a usuarios, las marcas de corte de sesión y los códigos de recuperación de contraseña.

**No se almacena:**
* **Auditoría:** la delega MS08 ([RNF-MS01-10](1_requirements.md#requerimientos-no-funcionales-rnf)).
* **Sesiones y tokens:** el token es stateless y no se persiste ([`3_jwt-spec.md`](3_jwt-spec.md)). Solo se guardan las marcas `valido_desde` que cortan sesiones ([RF-MS01-11](1_requirements.md#rf-ms01-11-corte-de-sesión-por-cambios-de-acceso)).
* **Datos de otros dominios** (alertas, check-ins, etc.): pertenecen a sus propios microservicios.

**Fuente de verdad de los permisos:** la base de datos. La tabla `permisos` es el único catálogo de códigos de permiso y de sus descripciones, y `rol_permiso` guarda qué permisos tiene cada rol ([RN-MS01-14](1_requirements.md#rf-ms01-05-gestión-de-roles-y-permisos)). El código de MS01 solo define sus propios permisos (`ms01:*`); quién escribe cada parte del catálogo está en [`permisos`](#permisos).

---

## Convenciones

* Tablas y columnas en `snake_case` y en español, alineadas con el contrato OpenAPI.
* Fechas y horas en `TIMESTAMPTZ`, guardadas en UTC.
* Los identificadores son UUID. La aplicación los genera al insertar, y las columnas también tienen `DEFAULT gen_random_uuid()`. El `id` del usuario es el mismo valor del `id` de la API, del claim `sub` y de `X-User-Id`. Los `id` de `roles` y de `permisos` son internos: no salen en la API, en el token ni en las cabeceras.
* El correo se guarda en minúsculas y sin espacios en los extremos ([RG-01](2_forms-inputs.md#reglas-generales-de-entrada)).
* Los roles son filas de la tabla `roles`. Hacia afuera (API, token y cabeceras) se identifican por su nombre, en MAYÚSCULAS y único; por dentro las tablas se relacionan por el `id` UUID del rol, así que renombrar un rol es un UPDATE de `roles.nombre` que no toca ninguna otra tabla. Los permisos son filas de la tabla `permisos` y se identifican por su código; `rol_permiso` une cada rol con sus permisos. La base de datos valida con claves foráneas, no con una lista fija, que cada rol de un usuario y cada permiso de un rol existan: crear, renombrar o borrar un rol, o agregar un permiso al catálogo, no exige cambiar código ni el esquema.
* Las restricciones llevan prefijo por tipo (`pk_`, `fk_`, `uq_`, `chh_` para CHECK y `trg_` para triggers). Los CHECK de `roles` protegen el formato del nombre, el nombre reservado `DETAILS` y la longitud de la descripción; el de `permisos`, el formato del código. Lo que depende de varias filas o de otra tabla no cabe en un CHECK y lo hacen cumplir triggers: la protección de los roles de sistema y las reglas de los permisos de cada rol ([`rol_permiso`](#rol_permiso)). Los errores de esos triggers usan SQLSTATE `23S01`.
* `valido_desde` guarda el instante de corte de sesión truncado a segundos (el mismo tipo de dato que el `iat` del token) y es `NULL` cuando no hay corte. Lo escribe MS01 con la hora de la base de datos, la misma con la que fija el `iat` de las credenciales ([corte de sesión](3_jwt-spec.md#corte-de-sesión)), así que todas las instancias de MS01 comparten un solo reloj.
* Los usuarios no se borran físicamente: se inactivan ([RN-MS01-17](1_requirements.md#rf-ms01-07-administración-de-usuarios)).
* El esquema se versiona como scripts SQL numerados en `database/` (`001_create_schema.sql`, `002_index_schema.sql`, `003_codigos_recuperacion.sql`). Las bases nuevas se crean con esos tres: `001` incluye todas las tablas, funciones y triggers de este documento y la siembra del catálogo de permisos y de los ocho roles ([datos iniciales](#datos-iniciales)), todo en una sola transacción. `database/migraciones/004_migracion_roles_permisos.sql` es el script manual, de una sola transacción, que lleva una base ya desplegada al esquema nuevo (tablas `roles`, `permisos` y `rol_permiso`, columnas `valido_desde` y `rol_id`, claves foráneas, restricciones y triggers); no se usa en bases nuevas y por eso está en la subcarpeta `migraciones/`, que Postgres no ejecuta al crear una base. Los cambios posteriores del catálogo de permisos de `ms02` a `ms08` se escriben como scripts numerados y versionados (`005` en adelante), uno por cambio, que ejecuta en cada entorno el responsable de la base de datos ([`permisos`](#permisos)). La aplicación no crea ni modifica el esquema: lo valida al arrancar (`spring.jpa.hibernate.ddl-auto: validate`).

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
| `segundo_nombre` | VARCHAR(30) | Sí | NULL | CHECK 1 a 30 si existe | Opcional. Admite desde un carácter. |
| `primer_apellido` | VARCHAR(30) | No | — | CHECK 2 a 30 caracteres | |
| `segundo_apellido` | VARCHAR(30) | Sí | NULL | CHECK 1 a 30 si existe | Opcional. Admite desde un carácter. |
| `correo` | VARCHAR(100) | No | — | UNIQUE (`uq_users_correo`); CHECK `chh_correo_user` (`correo = lower(correo)`); CHECK `chh_correo_user_format` (`correo ~ '^[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}$'`) | Correo institucional. No cambia después del registro ([RN-MS01-15](1_requirements.md#rf-ms01-06-gestión-del-perfil)). Los dos CHECK son independientes: uno garantiza las minúsculas y el otro la forma de un correo; el dominio institucional lo valida el servicio. |
| `password_hash` | VARCHAR(100) | No | — | — | Ver [almacenamiento seguro](#almacenamiento-seguro). |
| `estado` | VARCHAR(20) | No | `'ACTIVO'` | CHECK `chh_user_state` IN (`ACTIVO`, `INACTIVO`) | [RN-MS01-17](1_requirements.md#rf-ms01-07-administración-de-usuarios). Es texto y no un booleano para poder agregar un estado ampliando el CHECK, sin cambiar el tipo de la columna. |
| `creado_en` | TIMESTAMPTZ | No | `CURRENT_TIMESTAMP` | — | Fecha de registro. |
| `actualizado_en` | TIMESTAMPTZ | No | `CURRENT_TIMESTAMP` | Trigger `trg_users_updated_at` | Se actualiza en cada modificación del usuario (perfil, estado, contraseña). La fija la base de datos, no la aplicación. |
| `valido_desde` | TIMESTAMPTZ | Sí | NULL | — | Corte de sesión del usuario: las credenciales con `iat` menor se rechazan. Lo escribe MS01 al cambiar sus roles o su estado, o al renombrar o borrar uno de sus roles ([RN-MS01-35](1_requirements.md#rf-ms01-11-corte-de-sesión-por-cambios-de-acceso)). NULL = sin corte. Truncado a segundos. No se expone en la API. |

**Índices:** PK `id`; único `correo`; `idx_usuarios_correo_trgm` (GIN `pg_trgm`, para la búsqueda parcial por correo); `idx_usuarios_estado (estado)`; `idx_usuarios_valido_desde (valido_desde) WHERE valido_desde IS NOT NULL` (parcial: solo indexa los usuarios con corte, que son pocos, para armar la instantánea de cortes).

**Trigger de `actualizado_en`:** `trg_users_updated_at` (BEFORE UPDATE, función `set_updated_at()`, la misma que usa `roles`) asigna `NEW.actualizado_en = CURRENT_TIMESTAMP` en cada UPDATE, de modo que el valor no depende de que cada consulta de la aplicación lo escriba.

### roles
Una fila por rol. Es la fuente de los roles ([RF-MS01-10](1_requirements.md#rf-ms01-10-administración-de-roles-y-catálogo-de-permisos)); sus permisos están en [`rol_permiso`](#rol_permiso).

| Columna | Tipo | Nulo | Por defecto | Restricciones | Descripción |
|---|---|---|---|---|---|
| `id` | UUID | No | `gen_random_uuid()` | PK | Identificador interno del rol. No sale en la API, en el token ni en las cabeceras: hacia afuera el rol se identifica por su `nombre`. |
| `nombre` | VARCHAR(30) | No | — | UNIQUE (`uq_roles_nombre`); CHECK `chh_role_name` (`nombre ~ '^[A-Z][A-Z0-9_]{1,29}$'`); CHECK `chh_role_name_reserved` (`nombre <> 'DETAILS'`) | Nombre del rol y clave hacia afuera. 2 a 30 caracteres: mayúsculas, dígitos y guion bajo, empezando con una letra. No puede ser `DETAILS`, nombre reservado de la ruta `GET /roles/details`. El servicio recorta los espacios y lo pasa a mayúsculas antes de guardarlo ([RG-08](2_forms-inputs.md#reglas-generales-de-entrada)); la base solo acepta el valor ya normalizado. Es lo que viaja en el claim `roles` y en `X-User-Role`. Puede cambiar (salvo en los roles de sistema): renombrar es un UPDATE de esta columna y las demás tablas siguen al rol por `id`. |
| `descripcion` | VARCHAR(200) | No | — | CHECK `chh_role_description` (3 a 200 caracteres) | Texto que muestra la interfaz de administración. Editable. |
| `es_sistema` | BOOLEAN | No | `FALSE` | Trigger `trg_protect_system_roles` | `TRUE` solo en `ESTUDIANTE` y `ADMIN`. Solo lo escribe la siembra; ninguna petición de la API lo recibe. |
| `valido_desde` | TIMESTAMPTZ | Sí | NULL | — | Corte de sesión del rol: las credenciales con `iat` menor, de usuarios que tienen este rol, se rechazan. Solo cambia cuando cambian los permisos del rol ([RN-MS01-33](1_requirements.md#rf-ms01-10-administración-de-roles-y-catálogo-de-permisos)) o cuando se retira un permiso del catálogo que el rol tenía ([RN-MS01-42](1_requirements.md#rf-ms01-10-administración-de-roles-y-catálogo-de-permisos)). NULL = sin corte. Truncado a segundos. |
| `creado_en` | TIMESTAMPTZ | No | `CURRENT_TIMESTAMP` | — | Fecha de creación. |
| `actualizado_en` | TIMESTAMPTZ | No | `CURRENT_TIMESTAMP` | Trigger `trg_roles_updated_at` | Se actualiza en cada modificación de la fila del rol (nombre, descripción o corte). La fija la base de datos con la misma función `set_updated_at()` de `usuarios`. |

**Índices:** PK `id`; único `nombre`; `idx_roles_valido_desde (valido_desde) WHERE valido_desde IS NOT NULL` (parcial, para la instantánea de cortes).

**Restricciones, una por una:**

| Restricción | Qué garantiza | Regla |
|---|---|---|
| `uq_roles_nombre` | Que no haya dos roles con el mismo nombre | [RN-MS01-28](1_requirements.md#rf-ms01-10-administración-de-roles-y-catálogo-de-permisos) |
| `chh_role_name` | El formato del nombre | [RN-MS01-28](1_requirements.md#rf-ms01-10-administración-de-roles-y-catálogo-de-permisos) |
| `chh_role_name_reserved` | Que ningún rol se llame `DETAILS`, el segmento fijo de la ruta `GET /roles/details` | [RN-MS01-28](1_requirements.md#rf-ms01-10-administración-de-roles-y-catálogo-de-permisos) |
| `chh_role_description` | La longitud de la descripción | [RN-MS01-28](1_requirements.md#rf-ms01-10-administración-de-roles-y-catálogo-de-permisos) |
| `trg_protect_system_roles` | Un rol con `es_sistema = TRUE` no se borra, no se renombra y no cambia su marca. El trigger (`BEFORE UPDATE OF nombre, es_sistema OR DELETE`, función `protect_system_roles()`) lanza un error con SQLSTATE `23S01` | [RN-MS01-29](1_requirements.md#rf-ms01-10-administración-de-roles-y-catálogo-de-permisos) |
| `trg_roles_updated_at` | Que `actualizado_en` refleje siempre el último UPDATE (BEFORE UPDATE, función `set_updated_at()`) | — |

Los permisos de un rol y sus reglas (mínimo un permiso, `ms01:role:manage` solo en `ADMIN`, mínimo de `ESTUDIANTE` y `ADMIN`) se garantizan en [`rol_permiso`](#rol_permiso).

**Por qué los roles tienen un `id` además del nombre:** hacia afuera el nombre es la clave, y puede cambiar. Por dentro, `usuario_rol` y `rol_permiso` apuntan al `id`, que no cambia, así que renombrar un rol es un solo UPDATE y no hay que propagar el cambio a otras tablas.

**Por qué hay una columna `es_sistema` además del trigger:** la columna dice qué roles son de sistema (y permite que la interfaz deshabilite el borrado); el trigger lo hace cumplir aunque alguien escriba directamente en la base. El servicio también lo verifica primero, para responder con un error claro (409) en vez del error de la base.

**Roles de sistema:** `ESTUDIANTE` (rol inicial de todo registro y rol de respaldo al borrar un rol; conserva siempre `ms01:profile:read` y `ms01:profile:write`) y `ADMIN` (único dueño de `ms01:role:manage`, que nunca pierde; conserva siempre `ms01:profile:read` y `ms01:profile:write`).

### permisos
Catálogo de permisos: una fila por código. Es la única fuente de los códigos de permiso y de sus descripciones ([RN-MS01-14](1_requirements.md#rf-ms01-05-gestión-de-roles-y-permisos)).

| Columna | Tipo | Nulo | Por defecto | Restricciones | Descripción |
|---|---|---|---|---|---|
| `id` | UUID | No | `gen_random_uuid()` | PK | Identificador interno del permiso. No sale en la API ni en el token. |
| `nombre` | VARCHAR(100) | No | — | UNIQUE (`uq_permisos_nombre`); CHECK `chh_permission_name` (`nombre ~ '^ms[0-9]{2}:[a-z][a-z0-9_]*:[a-z][a-z0-9_]*$'`) | Código del permiso, con el formato `<ms>:<recurso>:<acción>` (p. ej. `ms04:checkin:write`). Es lo que viaja en el claim `permissions`, en `X-User-Permission` y en el `x-permiso` de cada OpenAPI, y el mismo formato que valida el servicio. |
| `descripcion` | VARCHAR(200) | No | — | — | Qué permite hacer el permiso. Es el texto que muestra la interfaz de administración (`GET /permissions`). |

**Índices:** PK `id`; único `nombre`.

**Quién escribe el catálogo:**
* **Permisos de MS01 (`ms01:*`):** MS01 los define en su código (enum `Permission`, con el código y la descripción de cada uno) y los sincroniza con la tabla al arrancar, en una sola transacción y solo sobre las filas con prefijo `ms01:`: inserta los que faltan, borra los que ya no están en el enum y no modifica los que ya existen (la descripción del enum solo se usa al insertar; después manda la base). Por cada permiso borrado deja un aviso en el log con los roles que lo tenían y registra el corte de sesión de esos roles. Si la base rechaza el borrado porque dejaría a un rol sin permisos o sin los mínimos de `ADMIN` o `ESTUDIANTE` ([`rol_permiso`](#rol_permiso)), la transacción se revierte y MS01 no arranca.
* **Permisos de `ms02` a `ms08`:** no están en el código de MS01. Los agrega y los retira el responsable de la base de datos con scripts numerados y versionados. Para agregar o actualizar un permiso, el script usa `INSERT ... ON CONFLICT (nombre) DO UPDATE SET descripcion`, que se puede repetir sin efecto. La siembra de `001` los carga por primera vez.
* **Ninguna ruta, pública ni interna, escribe en `permisos`:** la interfaz solo lee el catálogo (`GET /permissions`) y asigna permisos a los roles (`PUT /roles/{rol}/permissions`).
* **Matriz Rol‐Permiso:** es el registro oficial. Cada squad mantiene en ella los permisos que su microservicio realmente exige. El responsable de la base de datos agrega a `permisos` los que aún no están y retira los que ya no figuran en la Matriz. Mientras un permiso figure en la Matriz se mantiene en la base, aunque ningún rol lo tenga o su microservicio ya no lo use: quitarlo de la Matriz le corresponde a cada squad. Las columnas de roles de la Matriz son indicativas, porque los roles se editan desde la interfaz.

**Retirar un permiso:** el script de retiro corre en una sola transacción y sigue este orden: actualiza `valido_desde` de los roles que tienen el permiso (corte de sesión; este UPDATE es explícito en cada script de retiro, no lo hace ningún trigger), borra el permiso de `permisos` y la cascada de `rol_permiso` lo quita de los roles. Si con eso un rol se queda sin permisos, o `ADMIN` pierde `ms01:role:manage`, o `ESTUDIANTE` o `ADMIN` pierde `ms01:profile:read` o `ms01:profile:write`, el trigger diferido de [`rol_permiso`](#rol_permiso) rechaza la transacción: no se aplica ningún cambio y el error nombra el rol que lo impide ([RN-MS01-42](1_requirements.md#rf-ms01-10-administración-de-roles-y-catálogo-de-permisos)).

### rol_permiso
Permisos de cada rol: una fila por cada par rol-permiso. Es la fuente de los permisos de un rol ([RF-MS01-10](1_requirements.md#rf-ms01-10-administración-de-roles-y-catálogo-de-permisos)).

| Columna | Tipo | Nulo | Por defecto | Restricciones | Descripción |
|---|---|---|---|---|---|
| `rol_id` | UUID | No | — | PK (`pk_rol_permiso`, con `permiso_id`); FK → `roles.id` (`fk_rol_permiso_roles`, `ON DELETE CASCADE`) | Rol dueño del permiso. Al borrar un rol se borran sus filas. |
| `permiso_id` | UUID | No | — | PK (con `rol_id`); FK → `permisos.id` (`fk_rol_permiso_permisos`, `ON DELETE CASCADE`) | Permiso asignado. Al borrar un permiso del catálogo se quita de todos los roles. |

**Índices:** la PK compuesta (`rol_id`, `permiso_id`), que impide repetidos y sirve para leer los permisos de un rol; `idx_rol_permiso_permiso (permiso_id)`, para buscar los roles que tienen un permiso y para el borrado en cascada.

**Reglas de los permisos de un rol:** las hace cumplir el trigger `trg_check_role_permissions` (función `check_role_permissions()`):

| Regla | Qué garantiza | Regla de negocio |
|---|---|---|
| Mínimo un permiso | Todo rol tiene al menos un permiso | [RN-MS01-28](1_requirements.md#rf-ms01-10-administración-de-roles-y-catálogo-de-permisos) |
| Gestión de roles exclusiva | `ms01:role:manage` está en `ADMIN` y en ningún otro rol, y `ADMIN` nunca lo pierde | [RN-MS01-30](1_requirements.md#rf-ms01-10-administración-de-roles-y-catálogo-de-permisos) |
| Mínimo de `ESTUDIANTE` y `ADMIN` | `ESTUDIANTE` y `ADMIN` conservan siempre `ms01:profile:read` y `ms01:profile:write` | [RN-MS01-41](1_requirements.md#rf-ms01-10-administración-de-roles-y-catálogo-de-permisos) |

`trg_check_role_permissions` es un trigger de restricción `DEFERRABLE INITIALLY DEFERRED`, definido sobre `roles` (AFTER INSERT) y sobre `rol_permiso` (AFTER INSERT, UPDATE o DELETE). Se evalúa al confirmar la transacción, sobre el estado final de los roles tocados, y no fila por fila: por eso `PUT /roles/{rol}/permissions`, que reemplaza la lista completa (borra unas filas e inserta otras), se acepta si el resultado final cumple las reglas aunque un paso intermedio no las cumpliera. Ignora los roles que ya no existen al confirmar (un rol borrado). Si una regla falla, lanza un error con SQLSTATE `23S01` cuyo mensaje nombra el rol que la incumple, y la transacción se revierte completa.

**Por qué un trigger diferido y no un CHECK:** un CHECK solo ve su propia fila, y estas reglas dependen de varias filas y de dos tablas. El trigger diferido mira el resultado final de la transacción, y así también protege la siembra, los scripts del catálogo y la sincronización de MS01 aunque escriban directamente en la base. El servicio verifica las mismas reglas primero, para responder con un error claro (422) en vez del error de la base.

**Por qué los permisos de un rol son una tabla y no una lista dentro de `roles`:** la clave foránea garantiza que cada permiso asignado existe en el catálogo, retirar un permiso lo quita de todos los roles en la misma transacción, y se puede saber qué roles tienen un permiso sin recorrer listas.

### usuario_rol
Roles de cada usuario, por el `id` del rol (`roles.id`). Un usuario puede tener varios, hasta 50 ([RN-MS01-13 y RN-MS01-40](1_requirements.md#rf-ms01-05-gestión-de-roles-y-permisos)); el tope lo verifica el servicio, la base de datos no lo restringe.

| Columna | Tipo | Nulo | Por defecto | Restricciones | Descripción |
|---|---|---|---|---|---|
| `usuario_id` | UUID | No | — | PK (`pk_rol`, con `rol_id`); FK → `usuarios.id` (`fk_rol`) | |
| `rol_id` | UUID | No | — | PK (con `usuario_id`); FK → `roles.id` (`fk_usuario_rol_roles`, sin `ON DELETE` ni `ON UPDATE`) | Rol asignado. No hay una lista fija de ocho roles: la clave foránea exige que el rol exista. Renombrar un rol no cambia estas filas, porque apuntan al `id`; borrar un rol que todavía tiene usuarios la base lo rechaza, por eso el servicio reasigna antes de borrar. |
| `asignado_en` | TIMESTAMPTZ | No | `CURRENT_TIMESTAMP` | — | Fecha de asignación. |

**Índices:** la PK compuesta (`usuario_id`, `rol_id`), que impide roles repetidos; `idx_usuario_rol_rol (rol_id, usuario_id)` para el filtro por rol.

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
| `roles` → `usuario_rol` | 1 a N | Restringido: un rol con usuarios no se borra. Renombrar no afecta a `usuario_rol`, que apunta al `id` |
| `roles` → `rol_permiso` | 1 a N (mínimo 1 fila por rol, validado por el trigger diferido) | Cascada: al borrar un rol se borran sus permisos asignados |
| `permisos` → `rol_permiso` | 1 a N | Cascada: al retirar un permiso se quita de todos los roles; el trigger diferido impide dejar un rol sin permisos o sin los mínimos de `ADMIN` y `ESTUDIANTE` |
| `usuarios` → `codigos_recuperacion` | 1 a N (a lo sumo uno sin usar a la vez) | Sin cascada |

---

## Correspondencia entre API y columnas

Los campos del contrato tienen el mismo nombre que su columna en `usuarios` (`id`, nombres y apellidos, `correo`, `estado`), salvo estos:

| Campo del contrato | Columna | Notas |
|---|---|---|
| `contrasena`, `nueva_contrasena` | `usuarios.password_hash` | Solo entra. No se devuelve. |
| `roles` (de un usuario) | `roles.nombre` de las filas de `usuario_rol` del usuario (por `rol_id`) | Arreglo de nombres en orden alfabético (igual que en el token). |
| `rol` (en los roles) | `roles.nombre` | Nombre del rol. |
| `descripcion` (de un rol) | `roles.descripcion` | |
| `permisos` (de un rol) | `permisos.nombre` de las filas de `rol_permiso` del rol | Arreglo de códigos de permiso, ordenado y sin repetidos. |
| `permiso`, `descripcion` (en `GET /permissions`) | `permisos.nombre`, `permisos.descripcion` | Catálogo completo, ordenado por código. |
| `es_sistema` | `roles.es_sistema` | Solo se devuelve; nunca se recibe. |
| `permissions` (token) | `permisos.nombre` de las filas de `rol_permiso` de todos los roles del usuario | Unión ordenada y sin repetidos; se calcula al emitir el token. |
| `valido_desde` | `usuarios.valido_desde` y `roles.valido_desde` | Solo viaja en la ruta interna de cortes de sesión (usuarios y roles con corte), nunca en la API pública. |
| `codigo` | `codigos_recuperacion.codigo_hash` | Se compara contra el hash. No se devuelve. |
| `expira_en` (login) | — | Sale del claim `exp` del token, no de la BD. |

---

## Reglas de integridad

| Regla | Dónde se garantiza |
|---|---|
| [RN-MS01-01](1_requirements.md#rf-ms01-01-registrar-usuario-institucional) correo único | BD: UNIQUE + `chh_correo_user` (`correo = lower(correo)`) + `chh_correo_user_format` (forma de un correo). El servicio normaliza el correo antes de guardar y valida el dominio. |
| [RN-MS01-17](1_requirements.md#rf-ms01-07-administración-de-usuarios) estados válidos | BD: CHECK `chh_user_state`. |
| [RN-MS01-14](1_requirements.md#rf-ms01-05-gestión-de-roles-y-permisos) roles existentes, sin repetir; catálogo de permisos | BD: FK `fk_usuario_rol_roles` y PK compuesta en `usuario_rol`; tabla `permisos` (UNIQUE y CHECK de formato) y FK `fk_rol_permiso_permisos`, así que un permiso asignado a un rol existe siempre en el catálogo. Quién escribe el catálogo: [`permisos`](#permisos). |
| [RN-MS01-40](1_requirements.md#rf-ms01-05-gestión-de-roles-y-permisos) máximo de 50 roles por usuario | Servicio: calcula los roles que cada usuario tendría después del cambio y, si alguno supera 50, rechaza toda la petición con 422 antes de escribir. La BD no lo restringe. |
| [RN-MS01-28](1_requirements.md#rf-ms01-10-administración-de-roles-y-catálogo-de-permisos) formato del nombre, nombre reservado, descripción y permisos de un rol | BD: `chh_role_name`, `chh_role_name_reserved`, `chh_role_description`, `uq_roles_nombre`; que el rol tenga al menos un permiso, `trg_check_role_permissions`; que cada permiso exista, FK `fk_rol_permiso_permisos`. Servicio: lo verifica primero y responde 400 o 409. |
| [RN-MS01-29](1_requirements.md#rf-ms01-10-administración-de-roles-y-catálogo-de-permisos) roles de sistema | BD: trigger `trg_protect_system_roles`. Servicio: lo verifica primero y responde 409. La siembra es la única que escribe `es_sistema`. |
| [RN-MS01-30](1_requirements.md#rf-ms01-10-administración-de-roles-y-catálogo-de-permisos) `ms01:role:manage` solo en ADMIN | BD: trigger diferido `trg_check_role_permissions`. Servicio: lo verifica primero y responde 422. |
| [RN-MS01-41](1_requirements.md#rf-ms01-10-administración-de-roles-y-catálogo-de-permisos) `ESTUDIANTE` y `ADMIN` conservan `ms01:profile:read` y `ms01:profile:write` | BD: trigger diferido `trg_check_role_permissions`. Servicio: lo verifica primero y responde 422. |
| [RN-MS01-42](1_requirements.md#rf-ms01-10-administración-de-roles-y-catálogo-de-permisos) mantenimiento del catálogo de permisos | BD: FK con `ON DELETE CASCADE` en `rol_permiso` y trigger diferido, que cancela el retiro de un permiso si deja a un rol sin permisos o sin sus mínimos. MS01 (`ms01:*`) y responsable de la base de datos (`ms02` a `ms08`): ver [`permisos`](#permisos). |
| [RN-MS01-31](1_requirements.md#rf-ms01-10-administración-de-roles-y-catálogo-de-permisos) borrar un rol sin dejar usuarios sin rol | BD: FK `fk_usuario_rol_roles` sin `ON DELETE`. Servicio: reasigna a ESTUDIANTE o retira el rol y borra, todo en una transacción. |
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
| Login ([RF-02](1_requirements.md#rf-ms01-02-autenticar-usuario-login)) | `usuarios` por `correo` + sus roles + los permisos de esos roles (`rol_permiso` y `permisos`) | Ninguna |
| Actualizar perfil ([RF-06](1_requirements.md#rf-ms01-06-gestión-del-perfil)) | `usuarios` por id | UPDATE de los 4 campos de nombre (`actualizado_en` lo fija el trigger) |
| Cambio de estado en lote ([RF-07](1_requirements.md#rf-ms01-07-administración-de-usuarios)) | Usuarios objetivo con bloqueo de escritura | UPDATE `estado` y `valido_desde` de cada usuario |
| Cambio de roles en lote ([RF-05](1_requirements.md#rf-ms01-05-gestión-de-roles-y-permisos)) | Usuarios objetivo con bloqueo de escritura | Actualiza las filas de `usuario_rol` de los usuarios cuyos roles cambian (`agregar` / `retirar`) y su `valido_desde` |
| Solicitar código ([RF-08](1_requirements.md#rf-ms01-08-recuperación-y-cambio-de-contraseña)) | `usuarios` por `correo`; último código del usuario (espera) | Si está ACTIVO y fuera de la espera: marca `usado_en` en los códigos sin usar + INSERT del nuevo |
| Restablecer contraseña | Último código sin usar, con bloqueo de escritura | Código incorrecto: `intentos + 1` (y `usado_en` al llegar al máximo). Correcto: UPDATE `password_hash` + `usado_en` del código y de los demás sin usar |
| Cambiar contraseña autenticado | `usuarios` por id | UPDATE `password_hash` |
| Consultar roles y permisos ([RF-05](1_requirements.md#rf-ms01-05-gestión-de-roles-y-permisos), [RF-10](1_requirements.md#rf-ms01-10-administración-de-roles-y-catálogo-de-permisos)) | `roles` con sus permisos (uno o todos) y, para el catálogo, `permisos` | Ninguna |
| Crear rol ([RF-10](1_requirements.md#rf-ms01-10-administración-de-roles-y-catálogo-de-permisos)) | ¿Existe el `nombre`? Los permisos pedidos en `permisos` | INSERT `roles` (`es_sistema` falso, `valido_desde` nulo) + INSERT `rol_permiso` (una fila por permiso) |
| Renombrar o describir un rol ([RF-10](1_requirements.md#rf-ms01-10-administración-de-roles-y-catálogo-de-permisos)) | `roles` por `nombre`; ¿existe el nombre nuevo? | UPDATE `roles` (`nombre`, `descripcion`; `actualizado_en` lo fija el trigger); `usuario_rol` no cambia porque apunta al `id`; si el nombre cambia, se escribe `valido_desde` de cada usuario con ese rol |
| Cambiar permisos de un rol ([RF-10](1_requirements.md#rf-ms01-10-administración-de-roles-y-catálogo-de-permisos)) | `roles` por `nombre`; los permisos pedidos en `permisos` | DELETE de las filas de `rol_permiso` que sobran + INSERT de las nuevas + UPDATE `roles.valido_desde`. Las reglas de los permisos de un rol se evalúan al confirmar ([`rol_permiso`](#rol_permiso)) |
| Borrar un rol ([RF-10](1_requirements.md#rf-ms01-10-administración-de-roles-y-catálogo-de-permisos)) | `roles` por `nombre`; usuarios que lo tienen | Por cada usuario: UPDATE `usuario_rol` a ESTUDIANTE (si era su único rol) o DELETE de esa fila (si tenía varios) y UPDATE `usuarios.valido_desde`; luego DELETE `roles` (sus filas de `rol_permiso` se borran en cascada). Una sola transacción: si la BD rechaza algo, se revierte todo |
| Sincronizar los permisos de MS01 (arranque, [RN-MS01-42](1_requirements.md#rf-ms01-10-administración-de-roles-y-catálogo-de-permisos)) | `permisos` con prefijo `ms01:` y los roles que tienen cada permiso por borrar | INSERT de los `ms01:*` que faltan; por cada `ms01:*` que ya no está en el enum: UPDATE `roles.valido_desde` de los roles que lo tienen y DELETE de `permisos` (cascada sobre `rol_permiso`). Una sola transacción: si la BD rechaza algo, se revierte todo y MS01 no arranca |
| Retirar un permiso (script del responsable de la base de datos, [RN-MS01-42](1_requirements.md#rf-ms01-10-administración-de-roles-y-catálogo-de-permisos)) | Los roles que tienen el permiso | UPDATE `roles.valido_desde` de esos roles + DELETE de `permisos` (cascada sobre `rol_permiso`). Una sola transacción: si deja a un rol sin permisos o sin los mínimos de `ADMIN` o `ESTUDIANTE`, se revierte todo y el error nombra el rol |
| Consulta interna de cortes de sesión ([RF-11](1_requirements.md#rf-ms01-11-corte-de-sesión-por-cambios-de-acceso)) | `usuarios` y `roles` con `valido_desde` dentro de la ventana de 3700 s | Ninguna |
| Consulta interna por ids ([RF-09](1_requirements.md#rf-ms01-09-resolución-interna-de-usuarios)) | `usuarios` y sus roles con `id` en la lista (hasta 100) | Ninguna |
| Consulta interna por rol ([RF-09](1_requirements.md#rf-ms01-09-resolución-interna-de-usuarios)) | `usuarios` con una fila en `usuario_rol` del rol pedido (se resuelve por `roles.nombre`) y, si hay fecha de corte, `creado_en <= corte`; orden `id` ascendente | Ninguna |

**Listado `GET /users`** (parámetros en [F-07](2_forms-inputs.md#f-07-búsqueda-y-filtros-de-usuarios)):
* `correo`: coincidencia parcial sin distinguir mayúsculas (`LIKE '%texto%'` sobre el correo en minúsculas).
* `estado` / `rol`: igualdad; `rol` busca usuarios con una fila de ese rol en `usuario_rol` (el nombre se resuelve a `roles.id`).
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

1. **Catálogo de permisos y roles iniciales** ([RN-MS01-34](1_requirements.md#rf-ms01-10-administración-de-roles-y-catálogo-de-permisos)): los scripts `001` (bases nuevas) y `004` (base desplegada) cargan, en una sola transacción, los 48 permisos del catálogo (de `ms01` a `ms08`) con su descripción, los ocho roles y la asignación de permisos de la [Matriz Rol‐Permiso](https://github.com/CampusSeguro-DOSW-2026-2/documentation/wiki/Matriz-Rol%E2%80%90Permiso). Tiene que ser una sola transacción porque las reglas de los permisos de un rol ([`rol_permiso`](#rol_permiso)) se evalúan al confirmar. Es una siembra única: después de cargada, la fuente son las tablas `permisos`, `roles` y `rol_permiso`, y el administrador puede cambiar descripciones y permisos de los roles. Solo `ESTUDIANTE` y `ADMIN` se siembran con `es_sistema = TRUE`.

   | Rol | `es_sistema` | Permisos | Descripción inicial |
   |---|---|---|---|
   | `ESTUDIANTE` | Sí | 16 | Estudiante de la comunidad universitaria |
   | `PROFESOR` | No | 16 | Profesor de la comunidad universitaria |
   | `ADMINISTRATIVO` | No | 16 | Personal administrativo de la comunidad universitaria |
   | `BRIGADISTA` | No | 23 | Brigadista de emergencias del campus |
   | `COORDINADOR` | No | 34 | Coordinador de emergencias: gestiona alertas, brigadistas y asistencia |
   | `ORGANIZADOR` | No | 28 | Organizador de simulacros y eventos de evacuación |
   | `ADMIN` | Sí | 22 | Administrador del sistema: gestiona usuarios, roles, campus y contenidos |
   | `AUDITOR` | No | 17 | Auditor: consulta históricos, indicadores y auditoría (solo lectura) |

   Los códigos de permiso de cada rol están en el script y en la matriz de la wiki (48 permisos en el catálogo). `ms01:role:manage` queda solo en `ADMIN`, y `ESTUDIANTE` y `ADMIN` llevan `ms01:profile:read` y `ms01:profile:write`.
2. **Primer administrador** ([RN-MS01-12](1_requirements.md#rf-ms01-05-gestión-de-roles-y-permisos)): se crea al arrancar solo si no hay ningún usuario con el rol ADMIN y están definidas `BOOTSTRAP_ADMIN_EMAIL` y `BOOTSTRAP_ADMIN_PASSWORD` (nunca en el repositorio). Queda ACTIVO, con el rol ADMIN. `BOOTSTRAP_ADMIN_FIRST_NAME` y `BOOTSTRAP_ADMIN_LAST_NAME` son opcionales (por defecto "Administrador" e "Inicial"). Si el correo ya pertenece a otro usuario, no se crea ni se le asciende. Como no pasa por el registro, no se le aplican la política de contraseña ([RN-MS01-03](1_requirements.md#rf-ms01-01-registrar-usuario-institucional)) ni el dominio institucional: quien configura las variables debe usar un correo institucional y una contraseña que los cumpla.
3. **Limpieza opcional:** una tarea periódica puede eliminar los códigos usados o vencidos con más de 24 horas de antigüedad (no implementada).
