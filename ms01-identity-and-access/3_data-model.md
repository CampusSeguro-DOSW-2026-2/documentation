# Especificación del Modelo de Datos

MS01 tiene su propia base de datos PostgreSQL ([RNF-MS01-01](1_requirements.md#requerimientos-no-funcionales-rnf)). Ningún otro microservicio la lee ni la modifica: solo se accede a ella mediante la API. Esquema desplegado: `database/` del repositorio de MS01. Diagrama: [modelo físico](diagrams/ms01-er-physical.drawio.svg).

## Índice de Contenido
1. [Alcance](#alcance)
2. [Por qué una base de datos relacional](#por-qué-una-base-de-datos-relacional)
3. [Convenciones](#convenciones)
4. [Almacenamiento seguro](#almacenamiento-seguro)
5. [Tablas](#tablas)
    - [users](#users)
    - [roles](#roles)
    - [permissions](#permissions)
    - [role_permissions](#role_permissions)
    - [user_roles](#user_roles)
    - [password_reset_codes](#password_reset_codes)
    - [pending_registrations](#pending_registrations)
6. [Relaciones](#relaciones)
7. [Correspondencia entre API y columnas](#correspondencia-entre-api-y-columnas)
8. [Reglas de integridad](#reglas-de-integridad)
9. [Operaciones transaccionales y consultas](#operaciones-transaccionales-y-consultas)
10. [Configuración](#configuración)
11. [Datos iniciales](#datos-iniciales)

---

## Alcance

**Se almacena:** usuarios, los roles (con su descripción), el catálogo de permisos, los permisos de cada rol, la asignación de roles a usuarios, las marcas de corte de sesión, los códigos de recuperación de contraseña y los registros pendientes de verificación del correo.

**No se almacena:**
* **Auditoría:** la delega MS08 ([RNF-MS01-10](1_requirements.md#requerimientos-no-funcionales-rnf)).
* **Sesiones y tokens:** el token es stateless y no se persiste ([`jwt-spec.md`](jwt-spec.md)). Solo se guardan las marcas `valid_from` que cortan sesiones ([RF-MS01-11](1_requirements.md#rf-ms01-11-corte-de-sesión-por-cambios-de-acceso)).
* **Datos de otros dominios** (alertas, check-ins, etc.): pertenecen a sus propios microservicios.

**Fuente de verdad de los permisos:** la base de datos. La tabla `permissions` es el único catálogo de códigos de permiso y de sus descripciones, y `role_permissions` guarda qué permisos tiene cada rol ([RN-MS01-14](1_requirements.md#rf-ms01-05-gestión-de-roles-y-permisos)). El código de MS01 solo define sus propios permisos (`ms01:*`); quién escribe cada parte del catálogo está en [`permissions`](#permissions).

---

## Por qué una base de datos relacional

MS01 usa PostgreSQL, una base relacional, y no una base NoSQL (documental o de clave-valor), porque sus datos y sus reglas son relacionales:

| Necesidad de MS01 | Cómo la resuelve PostgreSQL | Qué pasaría con una base NoSQL |
|---|---|---|
| Un usuario tiene varios roles y un rol tiene varios permisos; los permisos efectivos son la unión de los permisos de todos los roles del usuario | Tablas de unión `user_roles` y `role_permissions` con claves foráneas, y una consulta con `JOIN` al iniciar sesión | Habría que copiar los roles y los permisos dentro de cada documento de usuario y mantener esas copias sincronizadas |
| Renombrar o borrar un rol, cambiar sus permisos o retirar un permiso del catálogo afecta a muchos usuarios a la vez | Un solo `UPDATE` o `DELETE`: las demás tablas apuntan al `id` del rol, y las cascadas y la transacción aplican todo o nada | El cambio tocaría cada documento que tiene la copia del rol o del permiso, sin la garantía de que todos queden iguales |
| Reglas que dependen de varias filas: todo rol conserva `ms01:profile:read` y `ms01:session:logout`, `ms01:role:manage` solo en `ADMIN`, los roles de sistema no se borran | Restricciones `CHECK`, `UNIQUE`, claves foráneas y triggers (uno de ellos diferido, que revisa el estado final de la transacción) | Las reglas quedarían solo en el código; una escritura directa o un script podría dejar datos inválidos |
| Unicidad del correo y del nombre de rol, incluso con peticiones simultáneas | `UNIQUE` (`uq_users_email`, `uq_roles_name`) y la llave primaria de `pending_registrations` con `INSERT ... ON CONFLICT` | La unicidad entre documentos depende de índices propios de cada producto y no cubre todas las combinaciones |
| Operaciones que escriben varias tablas y deben aplicarse completas: borrar un rol reasignando a sus usuarios, cambiar roles y registrar el corte de sesión, confirmar un registro | Transacciones ACID con bloqueo de filas (`SELECT ... FOR UPDATE`) | Las transacciones entre documentos son limitadas o más costosas, y un fallo a mitad podría dejar el cambio aplicado a medias |
| Un solo reloj para los cortes de sesión y el `iat` del token, compartido por todas las instancias de MS01 | `now()` de la base de datos, truncado a segundos | Cada instancia dependería de su propio reloj |
| Búsqueda parcial por correo y filtros por estado y rol, con paginación | Índice GIN `pg_trgm` sobre el correo, índices por estado y por rol, `ORDER BY` y `LIMIT/OFFSET` | Exigiría índices de texto propios del producto o recorrer documentos |

El volumen de datos de MS01 (usuarios de una universidad y unas decenas de roles y permisos) no necesita la escalabilidad horizontal que justifica una base NoSQL, y el esquema es estable y conocido de antemano. Las instancias de MS01 no guardan estado propio: todas comparten esta base, así que el servicio puede ejecutarse con varias instancias sin cambiar el modelo.

---

## Convenciones

* Tablas, columnas, restricciones, índices, funciones y triggers en `snake_case` y en inglés. Los campos del contrato OpenAPI se mantienen en español y su correspondencia con las columnas está en [Correspondencia entre API y columnas](#correspondencia-entre-api-y-columnas). Los valores que se guardan siguen en español: los estados (`ACTIVO`, `INACTIVO`), los nombres de los roles y las descripciones de roles y permisos.
* Fechas y horas en `TIMESTAMPTZ`, guardadas en UTC.
* Los identificadores son UUID. La aplicación los genera al insertar, y las columnas también tienen `DEFAULT gen_random_uuid()`. El `id` del usuario es el mismo valor del `id` de la API, del claim `sub` y de `X-User-Id`. Los `id` de `roles` y de `permissions` son internos: no salen en la API, en el token ni en las cabeceras.
* El correo se guarda en minúsculas y sin espacios en los extremos ([RG-01](2_forms-inputs.md#reglas-generales-de-entrada)).
* Los roles son filas de la tabla `roles`. Hacia afuera (API, token y cabeceras) se identifican por su nombre, en MAYÚSCULAS y único; por dentro las tablas se relacionan por el `id` UUID del rol, así que renombrar un rol es un UPDATE de `roles.name` que no toca ninguna otra tabla. Los permisos son filas de la tabla `permissions` y se identifican por su código; `role_permissions` une cada rol con sus permisos. La base de datos valida con claves foráneas, no con una lista fija, que cada rol de un usuario y cada permiso de un rol existan: crear, renombrar o borrar un rol, o agregar un permiso al catálogo, no exige cambiar código ni el esquema.
* Las restricciones llevan prefijo por tipo (`pk_`, `fk_`, `uq_`, `chh_` para CHECK y `trg_` para triggers). Los CHECK de `roles` protegen el formato del nombre, el nombre reservado `DETAILS` y la longitud de la descripción; el de `permissions`, el formato del código. Lo que depende de varias filas o de otra tabla no cabe en un CHECK y lo hacen cumplir triggers: la protección de los roles de sistema y las reglas de los permisos de cada rol ([`role_permissions`](#role_permissions)). Los errores de esos triggers usan SQLSTATE `23S01`.
* `valid_from` (`valido_desde` en la ruta interna de cortes) guarda el instante de corte de sesión truncado a segundos (el mismo tipo de dato que el `iat` del token) y es `NULL` cuando no hay corte. Lo escribe MS01 con la hora de la base de datos, la misma con la que fija el `iat` de las credenciales ([corte de sesión](jwt-spec.md#corte-de-sesión)), así que todas las instancias de MS01 comparten un solo reloj.
* Los usuarios no se borran físicamente: se inactivan ([RN-MS01-17](1_requirements.md#rf-ms01-07-administración-de-usuarios)).
* El esquema se versiona como scripts SQL numerados en `database/`. Las bases nuevas se crean con `001_create_schema.sql` y `002_index_schema.sql`: `001` incluye todas las tablas (también `password_reset_codes` y `pending_registrations`), funciones y triggers de este documento y la siembra del catálogo de permisos y de los ocho roles ([datos iniciales](#datos-iniciales)), todo en una sola transacción, y `002` crea los índices. Un script manual, de una sola transacción, no se usa en bases nuevas y por eso está en una subcarpeta que Postgres no ejecuta al crear una base:
    * `database/maintenance/cambio_permisos.sql`: el último cambio de la [matriz de roles y permisos](../integration/permissions-matrix.md), que lleva una base que ya existe (prueba y producción) desde la Matriz anterior a la actual con solo lo que cambió: agrega y retira permisos del catálogo, da y quita permisos a los ocho roles iniciales y corta las sesiones solo de los roles afectados. No toca los roles creados desde la API ni los demás cambios del administrador, y se puede aplicar más de una vez sin efecto. Cada cambio de la Matriz lo reemplaza; los anteriores quedan en el historial del repositorio.

  La carpeta solo conserva lo que está en uso: un cambio de esquema se escribe como un script que se aplica a cada base existente, luego se integra en `001` y `002` y se borra. La aplicación no crea ni modifica el esquema: lo valida al arrancar (`spring.jpa.hibernate.ddl-auto: validate`).

---

## Almacenamiento seguro

Detalle técnico de [RN-MS01-04](1_requirements.md#rf-ms01-01-registrar-usuario-institucional):
* **Contraseñas:** solo se guarda su hash BCrypt (`users.password_hash`). La contraseña en claro nunca se guarda, no se registra en logs y nunca se devuelve.
* **Códigos de recuperación:** solo se guarda su hash BCrypt (`password_reset_codes.code_hash`). El código en claro solo viaja en el correo al usuario.
* **Registros pendientes:** la contraseña y el código de verificación se guardan solo como hash BCrypt (`pending_registrations.password_hash` y `code_hash`), y el token de registro solo como su hash SHA-256 (`request_token_hash`); un token aleatorio de 32 bytes no necesita un hash lento. Al confirmar el código, el hash de la contraseña pasa tal cual a `users.password_hash`: la contraseña en claro no vuelve a existir.

---

## Tablas

### users
Una fila por persona registrada.

| Columna | Tipo | Nulo | Por defecto | Restricciones | Descripción |
|---|---|---|---|---|---|
| `id` | UUID | No | `gen_random_uuid()` | PK | Identificador del usuario. |
| `first_name` | VARCHAR(30) | No | — | CHECK `chh_user_first_name` (2 a 30 caracteres) | [F-01](2_forms-inputs.md#f-01-registro-de-usuario). |
| `second_name` | VARCHAR(30) | Sí | NULL | CHECK `chh_user_second_name` (1 a 30 si existe) | Opcional. El servicio exige de 2 a 30 ([F-01](2_forms-inputs.md#f-01-registro-de-usuario)); la BD acepta desde 1 para que los datos anteriores sigan siendo válidos. |
| `first_lastname` | VARCHAR(30) | No | — | CHECK `chh_user_first_lastname` (2 a 30 caracteres) | |
| `second_lastname` | VARCHAR(30) | Sí | NULL | CHECK `chh_user_second_lastname` (1 a 30 si existe) | Opcional. El servicio exige de 2 a 30; la BD acepta desde 1, como `second_name`. |
| `email` | VARCHAR(100) | No | — | UNIQUE (`uq_users_email`); CHECK `chh_user_email` (`email = lower(email)`); CHECK `chh_user_email_format` (`email ~ '^[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}$'`) | Correo institucional. No cambia después del registro ([RN-MS01-15](1_requirements.md#rf-ms01-06-gestión-del-perfil)). Los dos CHECK son independientes: uno garantiza las minúsculas y el otro la forma de un correo; el dominio institucional lo valida el servicio. |
| `password_hash` | VARCHAR(100) | No | — | — | Ver [almacenamiento seguro](#almacenamiento-seguro). |
| `status` | VARCHAR(20) | No | `'ACTIVO'` | CHECK `chh_user_status` IN (`ACTIVO`, `INACTIVO`) | [RN-MS01-17](1_requirements.md#rf-ms01-07-administración-de-usuarios). Es texto y no un booleano para poder agregar un estado ampliando el CHECK, sin cambiar el tipo de la columna. |
| `created_at` | TIMESTAMPTZ | No | `CURRENT_TIMESTAMP` | — | Fecha de registro. |
| `updated_at` | TIMESTAMPTZ | No | `CURRENT_TIMESTAMP` | Trigger `trg_users_updated_at` | Se actualiza en cada modificación del usuario (perfil, estado, contraseña). La fija la base de datos, no la aplicación. |
| `valid_from` | TIMESTAMPTZ | Sí | NULL | — | Corte de sesión del usuario: las credenciales con `iat` menor se rechazan. Lo escribe MS01 al cambiar sus roles o su estado, o al renombrar o borrar uno de sus roles ([RN-MS01-35](1_requirements.md#rf-ms01-11-corte-de-sesión-por-cambios-de-acceso)). NULL = sin corte. Truncado a segundos. No se expone en la API pública. |

**Índices:** PK `id`; único `email`; `idx_users_email_trgm` (GIN `pg_trgm`, para la búsqueda parcial por correo); `idx_users_status (status)`; `idx_users_valid_from (valid_from) WHERE valid_from IS NOT NULL` (parcial: solo indexa los usuarios con corte, que son pocos, para armar la instantánea de cortes).

**Trigger de `updated_at`:** `trg_users_updated_at` (BEFORE UPDATE, función `set_updated_at()`, la misma que usa `roles`) asigna `NEW.updated_at = CURRENT_TIMESTAMP` en cada UPDATE, de modo que el valor no depende de que cada consulta de la aplicación lo escriba.

### roles
Una fila por rol. Es la fuente de los roles ([RF-MS01-10](1_requirements.md#rf-ms01-10-administración-de-roles-y-catálogo-de-permisos)); sus permisos están en [`role_permissions`](#role_permissions).

| Columna | Tipo | Nulo | Por defecto | Restricciones | Descripción |
|---|---|---|---|---|---|
| `id` | UUID | No | `gen_random_uuid()` | PK | Identificador interno del rol. No sale en la API, en el token ni en las cabeceras: hacia afuera el rol se identifica por su `name`. |
| `name` | VARCHAR(30) | No | — | UNIQUE (`uq_roles_name`); CHECK `chh_role_name` (`name ~ '^[A-Z][A-Z0-9_]{1,29}$'`); CHECK `chh_role_name_reserved` (`name <> 'DETAILS'`) | Nombre del rol y clave hacia afuera. 2 a 30 caracteres: mayúsculas, dígitos y guion bajo, empezando con una letra. No puede ser `DETAILS`, nombre reservado de la ruta `GET /roles/details`. El servicio recorta los espacios y lo pasa a mayúsculas antes de guardarlo ([RG-08](2_forms-inputs.md#reglas-generales-de-entrada)); la base solo acepta el valor ya normalizado. Es lo que viaja en el claim `roles` y en `X-User-Role`. Puede cambiar (salvo en los roles de sistema): renombrar es un UPDATE de esta columna y las demás tablas siguen al rol por `id`. |
| `description` | VARCHAR(200) | No | — | CHECK `chh_role_description` (3 a 200 caracteres) | Texto que muestra la interfaz de administración. Editable. |
| `is_system` | BOOLEAN | No | `FALSE` | Trigger `trg_protect_system_roles` | `TRUE` solo en `ESTUDIANTE` y `ADMIN`. Solo lo escribe la siembra; ninguna petición de la API lo recibe. |
| `valid_from` | TIMESTAMPTZ | Sí | NULL | — | Corte de sesión del rol: las credenciales con `iat` menor, de usuarios que tienen este rol, se rechazan. Solo cambia cuando cambian los permisos del rol ([RN-MS01-33](1_requirements.md#rf-ms01-10-administración-de-roles-y-catálogo-de-permisos)) o cuando se retira un permiso del catálogo que el rol tenía ([RN-MS01-42](1_requirements.md#rf-ms01-10-administración-de-roles-y-catálogo-de-permisos)). NULL = sin corte. Truncado a segundos. |
| `created_at` | TIMESTAMPTZ | No | `CURRENT_TIMESTAMP` | — | Fecha de creación. |
| `updated_at` | TIMESTAMPTZ | No | `CURRENT_TIMESTAMP` | Trigger `trg_roles_updated_at` | Se actualiza en cada modificación de la fila del rol (nombre, descripción o corte). La fija la base de datos con la misma función `set_updated_at()` de `users`. |

**Índices:** PK `id`; único `name`; `idx_roles_valid_from (valid_from) WHERE valid_from IS NOT NULL` (parcial, para la instantánea de cortes).

**Restricciones, una por una:**

| Restricción | Qué garantiza | Regla |
|---|---|---|
| `uq_roles_name` | Que no haya dos roles con el mismo nombre | [RN-MS01-28](1_requirements.md#rf-ms01-10-administración-de-roles-y-catálogo-de-permisos) |
| `chh_role_name` | El formato del nombre | [RN-MS01-28](1_requirements.md#rf-ms01-10-administración-de-roles-y-catálogo-de-permisos) |
| `chh_role_name_reserved` | Que ningún rol se llame `DETAILS`, el segmento fijo de la ruta `GET /roles/details` | [RN-MS01-28](1_requirements.md#rf-ms01-10-administración-de-roles-y-catálogo-de-permisos) |
| `chh_role_description` | La longitud de la descripción | [RN-MS01-28](1_requirements.md#rf-ms01-10-administración-de-roles-y-catálogo-de-permisos) |
| `trg_protect_system_roles` | Un rol con `is_system = TRUE` no se borra, no se renombra y no cambia su marca. El trigger (`BEFORE UPDATE OF name, is_system OR DELETE`, función `protect_system_roles()`) lanza un error con SQLSTATE `23S01` | [RN-MS01-29](1_requirements.md#rf-ms01-10-administración-de-roles-y-catálogo-de-permisos) |
| `trg_roles_updated_at` | Que `updated_at` refleje siempre el último UPDATE (BEFORE UPDATE, función `set_updated_at()`) | — |

Los permisos de un rol y sus reglas (los dos permisos obligatorios de todo rol, `ms01:role:manage` solo en `ADMIN` y `ms01:profile:write` en `ESTUDIANTE` y `ADMIN`) se garantizan en [`role_permissions`](#role_permissions).

**Por qué los roles tienen un `id` además del nombre:** hacia afuera el nombre es la clave, y puede cambiar. Por dentro, `user_roles` y `role_permissions` apuntan al `id`, que no cambia, así que renombrar un rol es un solo UPDATE y no hay que propagar el cambio a otras tablas.

**Por qué hay una columna `is_system` además del trigger:** la columna dice qué roles son de sistema (y permite que la interfaz deshabilite el borrado); el trigger lo hace cumplir aunque alguien escriba directamente en la base. El servicio también lo verifica primero, para responder con un error claro (409) en vez del error de la base.

**Roles de sistema:** `ESTUDIANTE` (rol inicial de todo registro y rol de respaldo al borrar un rol; conserva siempre `ms01:profile:read`, `ms01:session:logout` y `ms01:profile:write`) y `ADMIN` (único dueño de `ms01:role:manage`, que nunca pierde; conserva siempre `ms01:profile:read`, `ms01:session:logout` y `ms01:profile:write`).

### permissions
Catálogo de permisos: una fila por código. Es la única fuente de los códigos de permiso y de sus descripciones ([RN-MS01-14](1_requirements.md#rf-ms01-05-gestión-de-roles-y-permisos)).

| Columna | Tipo | Nulo | Por defecto | Restricciones | Descripción |
|---|---|---|---|---|---|
| `id` | UUID | No | `gen_random_uuid()` | PK | Identificador interno del permiso. No sale en la API ni en el token. |
| `name` | VARCHAR(100) | No | — | UNIQUE (`uq_permissions_name`); CHECK `chh_permission_name` (`name ~ '^ms[0-9]{2}:[a-z][a-z0-9_]*:[a-z][a-z0-9_]*$'`) | Código del permiso, con el formato `<ms>:<recurso>:<acción>` (p. ej. `ms04:checkin:write`). Es lo que viaja en el claim `permissions`, en `X-User-Permission` y en el `x-permiso` de cada OpenAPI, y el mismo formato que valida el servicio. |
| `description` | VARCHAR(200) | No | — | — | Qué permite hacer el permiso. Es el texto que muestra la interfaz de administración (`GET /permissions`). |

**Índices:** PK `id`; único `name`.

**Quién escribe el catálogo:**
* **Permisos de MS01 (`ms01:*`):** MS01 los define en su código (enum `Permission`, con el código y la descripción de cada uno) y los sincroniza con la tabla al arrancar, en una sola transacción y solo sobre las filas con prefijo `ms01:`: inserta los que faltan, borra los que ya no están en el enum y no modifica los que ya existen (la descripción del enum solo se usa al insertar; después manda la base). Por cada permiso borrado deja un aviso en el log con los roles que lo tenían y registra el corte de sesión de esos roles. Si la base rechaza el borrado porque dejaría a un rol sin los permisos obligatorios o sin los mínimos de `ADMIN` o `ESTUDIANTE` ([`role_permissions`](#role_permissions)), la transacción se revierte y MS01 no arranca.
* **Permisos de `ms02` a `ms08`:** no están en el código de MS01. Los carga por primera vez la siembra de `001`, y el responsable de la base de datos los agrega y los retira en las bases que ya existen con `database/maintenance/cambio_permisos.sql`, generado a partir de la Matriz con solo lo que cambió ([datos iniciales](#datos-iniciales)). Para agregar o actualizar un permiso, el script usa `INSERT ... ON CONFLICT (name) DO UPDATE SET description`, que se puede repetir sin efecto.
* **Ninguna ruta, pública ni interna, escribe en `permissions`:** la interfaz solo lee el catálogo (`GET /permissions`) y asigna permisos a los roles (`PUT /roles/{rol}/permissions`).
* **Matriz de roles y permisos:** es el registro oficial. Cada squad mantiene en ella los permisos que su microservicio realmente exige. El responsable de la base de datos agrega a `permissions` los que aún no están y retira los que ya no figuran en la Matriz. Mientras un permiso figure en la Matriz se mantiene en la base, aunque ningún rol lo tenga o su microservicio ya no lo use: quitarlo de la Matriz le corresponde a cada squad. Las columnas de roles de la Matriz son indicativas, porque los roles se editan desde la interfaz.

**Retirar un permiso:** el script de retiro corre en una sola transacción y sigue este orden: actualiza `valid_from` de los roles que tienen el permiso (corte de sesión; este UPDATE es explícito en cada script de retiro, no lo hace ningún trigger), borra el permiso de `permissions` y la cascada de `role_permissions` lo quita de los roles. Si con eso un rol pierde `ms01:profile:read` o `ms01:session:logout`, o `ESTUDIANTE` o `ADMIN` pierde `ms01:profile:write`, o `ADMIN` pierde `ms01:role:manage`, el trigger diferido de [`role_permissions`](#role_permissions) rechaza la transacción: no se aplica ningún cambio y el error nombra el rol que lo impide ([RN-MS01-42](1_requirements.md#rf-ms01-10-administración-de-roles-y-catálogo-de-permisos)).

### role_permissions
Permisos de cada rol: una fila por cada par rol-permiso. Es la fuente de los permisos de un rol ([RF-MS01-10](1_requirements.md#rf-ms01-10-administración-de-roles-y-catálogo-de-permisos)).

| Columna | Tipo | Nulo | Por defecto | Restricciones | Descripción |
|---|---|---|---|---|---|
| `role_id` | UUID | No | — | PK (`pk_role_permissions`, con `permission_id`); FK → `roles.id` (`fk_role_permissions_roles`, `ON DELETE CASCADE`) | Rol dueño del permiso. Al borrar un rol se borran sus filas. |
| `permission_id` | UUID | No | — | PK (con `role_id`); FK → `permissions.id` (`fk_role_permissions_permissions`, `ON DELETE CASCADE`) | Permiso asignado. Al borrar un permiso del catálogo se quita de todos los roles. |

**Índices:** la PK compuesta (`role_id`, `permission_id`), que impide repetidos y sirve para leer los permisos de un rol; `idx_role_permissions_permission (permission_id)`, para buscar los roles que tienen un permiso y para el borrado en cascada.

**Reglas de los permisos de un rol:** las hace cumplir el trigger `trg_check_role_permissions` (función `check_role_permissions()`):

| Regla | Qué garantiza | Regla de negocio |
|---|---|---|
| Permisos obligatorios | Todo rol conserva siempre `ms01:profile:read` y `ms01:session:logout`, el mínimo para que cualquier cuenta consulte su perfil y cierre sesión | [RN-MS01-41](1_requirements.md#rf-ms01-10-administración-de-roles-y-catálogo-de-permisos) |
| Perfil editable de `ESTUDIANTE` y `ADMIN` | `ESTUDIANTE` y `ADMIN` conservan además `ms01:profile:write` | [RN-MS01-41](1_requirements.md#rf-ms01-10-administración-de-roles-y-catálogo-de-permisos) |
| Gestión de roles exclusiva | `ms01:role:manage` está en `ADMIN` y en ningún otro rol, y `ADMIN` nunca lo pierde | [RN-MS01-30](1_requirements.md#rf-ms01-10-administración-de-roles-y-catálogo-de-permisos) |

`trg_check_role_permissions` es un trigger de restricción `DEFERRABLE INITIALLY DEFERRED`, definido sobre `roles` (AFTER INSERT) y sobre `role_permissions` (AFTER INSERT, UPDATE o DELETE). Se evalúa al confirmar la transacción, sobre el estado final de los roles tocados, y no fila por fila: por eso `PUT /roles/{rol}/permissions`, que reemplaza la lista completa (borra unas filas e inserta otras), se acepta si el resultado final cumple las reglas aunque un paso intermedio no las cumpliera. Ignora los roles que ya no existen al confirmar (un rol borrado). Si una regla falla, lanza un error con SQLSTATE `23S01` cuyo mensaje nombra el rol que la incumple, y la transacción se revierte completa.

**Por qué un trigger diferido y no un CHECK:** un CHECK solo ve su propia fila, y estas reglas dependen de varias filas y de dos tablas. El trigger diferido mira el resultado final de la transacción, y así también protege la siembra, los scripts del catálogo y la sincronización de MS01 aunque escriban directamente en la base. El servicio verifica las mismas reglas primero, para responder con un error claro (422) en vez del error de la base.

**Permisos obligatorios al crear un rol:** como la base rechaza un rol que no tenga `ms01:profile:read` y `ms01:session:logout`, el servicio los agrega a todo rol nuevo aunque la petición no los incluya ([RN-MS01-41](1_requirements.md#rf-ms01-10-administración-de-roles-y-catálogo-de-permisos)). Al reemplazar los permisos de un rol no los agrega: si faltan en la lista, responde 422.

**Por qué los permisos de un rol son una tabla y no una lista dentro de `roles`:** la clave foránea garantiza que cada permiso asignado existe en el catálogo, retirar un permiso lo quita de todos los roles en la misma transacción, y se puede saber qué roles tienen un permiso sin recorrer listas.

### user_roles
Roles de cada usuario, por el `id` del rol (`roles.id`). Un usuario puede tener varios, hasta 50 ([RN-MS01-13 y RN-MS01-40](1_requirements.md#rf-ms01-05-gestión-de-roles-y-permisos)); el tope lo verifica el servicio, la base de datos no lo restringe.

| Columna | Tipo | Nulo | Por defecto | Restricciones | Descripción |
|---|---|---|---|---|---|
| `user_id` | UUID | No | — | PK (`pk_user_roles`, con `role_id`); FK → `users.id` (`fk_user_roles_users`) | |
| `role_id` | UUID | No | — | PK (con `user_id`); FK → `roles.id` (`fk_user_roles_roles`, sin `ON DELETE` ni `ON UPDATE`) | Rol asignado. No hay una lista fija de ocho roles: la clave foránea exige que el rol exista. Renombrar un rol no cambia estas filas, porque apuntan al `id`; borrar un rol que todavía tiene usuarios la base lo rechaza, por eso el servicio reasigna antes de borrar. |
| `assigned_at` | TIMESTAMPTZ | No | `CURRENT_TIMESTAMP` | — | Fecha de asignación. |

**Índices:** la PK compuesta (`user_id`, `role_id`), que impide roles repetidos; `idx_user_roles_role (role_id, user_id)` para el filtro por rol.

### password_reset_codes
Códigos de un solo uso para restablecer la contraseña ([RF-MS01-08](1_requirements.md#rf-ms01-08-recuperación-y-cambio-de-contraseña)).

| Columna | Tipo | Nulo | Por defecto | Restricciones | Descripción |
|---|---|---|---|---|---|
| `id` | UUID | No | `gen_random_uuid()` | PK | |
| `user_id` | UUID | No | — | FK → `users.id` (`fk_password_reset_codes_users`) | Dueño del código. |
| `code_hash` | VARCHAR(100) | No | — | — | Ver [almacenamiento seguro](#almacenamiento-seguro). |
| `attempts` | INTEGER | No | `0` | CHECK `chh_password_reset_codes_attempts` (`attempts >= 0`) | Intentos fallidos con este código. |
| `expires_at` | TIMESTAMPTZ | No | — | — | Creación + vigencia ([configuración](#configuración)). |
| `used_at` | TIMESTAMPTZ | Sí | NULL | — | NULL = no usado. Con valor = consumido o invalidado. |
| `created_at` | TIMESTAMPTZ | No | `CURRENT_TIMESTAMP` | — | También mide la espera entre solicitudes. |

**Índices:** PK `id`; `idx_password_reset_codes_user (user_id, expires_at)`.
**Vigente** = `used_at IS NULL` y `expires_at > now()`.

### pending_registrations
Registros que esperan la verificación del correo ([RF-MS01-01](1_requirements.md#rf-ms01-01-registrar-usuario-institucional), [RN-MS01-43](1_requirements.md#rf-ms01-01-registrar-usuario-institucional)). No son usuarios: el usuario se crea en `users` al confirmar el código, y en ese momento se borra la fila. No tiene claves foráneas, porque el usuario todavía no existe.

| Columna | Tipo | Nulo | Por defecto | Restricciones | Descripción |
|---|---|---|---|---|---|
| `email` | VARCHAR(100) | No | — | PK; CHECK `chh_pending_registrations_email` (`email = lower(email)`) y `chh_pending_registrations_email_format` | Correo que se verifica. Al ser la clave, cada correo tiene a lo sumo un registro pendiente. |
| `first_name` | VARCHAR(30) | No | — | CHECK `chh_pending_registrations_first_name` (2 a 30) | Mismas reglas que en `users`. |
| `second_name` | VARCHAR(30) | Sí | NULL | CHECK `chh_pending_registrations_second_name` (1 a 30) | |
| `first_lastname` | VARCHAR(30) | No | — | CHECK `chh_pending_registrations_first_lastname` (2 a 30) | |
| `second_lastname` | VARCHAR(30) | Sí | NULL | CHECK `chh_pending_registrations_second_lastname` (1 a 30) | |
| `password_hash` | VARCHAR(100) | No | — | — | Ver [almacenamiento seguro](#almacenamiento-seguro). |
| `code_hash` | VARCHAR(100) | No | — | — | Último código enviado. Un código nuevo reemplaza al anterior. |
| `request_token_hash` | VARCHAR(64) | No | — | — | SHA-256 en hexadecimal del token que recibió quien hizo el registro ([RN-MS01-46](1_requirements.md#rf-ms01-01-registrar-usuario-institucional)). Cambia en cada nuevo registro del correo; el reenvío no lo cambia. |
| `attempts` | INTEGER | No | `0` | CHECK `chh_pending_registrations_attempts` (`attempts >= 0`) | Intentos fallidos con el código actual. Vuelve a `0` con cada código nuevo. |
| `expires_at` | TIMESTAMPTZ | No | — | — | Envío + vigencia ([configuración](#configuración)). Al agotar los intentos se mueve al presente, lo que invalida el código. |
| `code_sent_at` | TIMESTAMPTZ | No | `CURRENT_TIMESTAMP` | — | Envío del último código; mide la espera mínima entre dos códigos. |
| `codes_sent` | INTEGER | No | `1` | CHECK `chh_pending_registrations_codes_sent` (`codes_sent >= 1`) | Códigos enviados en la ventana actual de 24 horas (límite diario). |
| `window_started_at` | TIMESTAMPTZ | No | `CURRENT_TIMESTAMP` | — | Inicio de la ventana de 24 horas del límite diario. |

**Índices:** PK `email`; `idx_pending_registrations_expires (expires_at)`, para la limpieza periódica.
**Vigente** = `expires_at > now()`. Todas las horas de esta tabla salen del reloj de la base de datos (`now()`).

---

## Relaciones

| Relación | Cardinalidad | Borrado |
|---|---|---|
| `users` → `user_roles` | 1 a N (mínimo 1 fila por usuario, validado en el servicio) | Sin cascada (no hay borrado de usuarios) |
| `roles` → `user_roles` | 1 a N | Restringido: un rol con usuarios no se borra. Renombrar no afecta a `user_roles`, que apunta al `id` |
| `roles` → `role_permissions` | 1 a N (mínimo 2 filas por rol, los permisos obligatorios, validado por el trigger diferido) | Cascada: al borrar un rol se borran sus permisos asignados |
| `permissions` → `role_permissions` | 1 a N | Cascada: al retirar un permiso se quita de todos los roles; el trigger diferido impide dejar un rol sin los permisos obligatorios o sin los mínimos de `ADMIN` y `ESTUDIANTE` |
| `users` → `password_reset_codes` | 1 a N (a lo sumo uno sin usar a la vez) | Sin cascada |

`pending_registrations` no se relaciona con ninguna tabla: guarda solicitudes de usuarios que todavía no existen. Su único vínculo con `users` es el correo, y el servicio verifica que no esté registrado al solicitar y al confirmar.

---

## Correspondencia entre API y columnas

Los campos del contrato están en español y las columnas en inglés. Los que no figuran aquí no se guardan: se calculan o vienen del token.

| Campo del contrato | Columna | Notas |
|---|---|---|
| `id` | `users.id` | |
| `primer_nombre`, `segundo_nombre`, `primer_apellido`, `segundo_apellido` | `users.first_name`, `users.second_name`, `users.first_lastname`, `users.second_lastname` | |
| `correo` | `users.email` | Se guarda en minúsculas. |
| `estado` | `users.status` | Los valores (`ACTIVO`, `INACTIVO`) son los mismos en la API y en la base. |
| `contrasena`, `nueva_contrasena` | `users.password_hash` | Solo entra. No se devuelve. En el registro se guarda primero en `pending_registrations.password_hash` y pasa a `users` al confirmar el código. |
| `roles` (de un usuario) | `roles.name` de las filas de `user_roles` del usuario (por `role_id`) | Arreglo de nombres en orden alfabético (igual que en el token). |
| `rol` (en los roles) | `roles.name` | Nombre del rol. |
| `descripcion` (de un rol) | `roles.description` | |
| `permisos` (de un rol) | `permissions.name` de las filas de `role_permissions` del rol | Arreglo de códigos de permiso, ordenado y sin repetidos. |
| `permiso`, `descripcion` (en `GET /permissions`) | `permissions.name`, `permissions.description` | Catálogo completo, ordenado por código. |
| `es_sistema` | `roles.is_system` | Solo se devuelve; nunca se recibe. |
| `permissions` (token) | `permissions.name` de las filas de `role_permissions` de todos los roles del usuario | Unión ordenada y sin repetidos; se calcula al emitir el token. |
| `valido_desde` | `users.valid_from` y `roles.valid_from` | Solo viaja en la ruta interna de cortes de sesión (usuarios y roles con corte), nunca en la API pública. |
| `codigo` | `password_reset_codes.code_hash` (recuperación) y `pending_registrations.code_hash` (verificación del registro) | Se compara contra el hash. No se devuelve. |
| `expira_en` (login) | — | Sale del claim `exp` del token, no de la BD. |

---

## Reglas de integridad

| Regla | Dónde se garantiza |
|---|---|
| [RN-MS01-01](1_requirements.md#rf-ms01-01-registrar-usuario-institucional) correo único | BD: UNIQUE `uq_users_email` + `chh_user_email` (`email = lower(email)`) + `chh_user_email_format` (forma de un correo). El servicio normaliza el correo antes de guardar y valida el dominio. |
| [RN-MS01-17](1_requirements.md#rf-ms01-07-administración-de-usuarios) estados válidos | BD: CHECK `chh_user_status`. |
| [RN-MS01-14](1_requirements.md#rf-ms01-05-gestión-de-roles-y-permisos) roles existentes, sin repetir; catálogo de permisos | BD: FK `fk_user_roles_roles` y PK compuesta en `user_roles`; tabla `permissions` (UNIQUE y CHECK de formato) y FK `fk_role_permissions_permissions`, así que un permiso asignado a un rol existe siempre en el catálogo. Quién escribe el catálogo: [`permissions`](#permissions). |
| [RN-MS01-40](1_requirements.md#rf-ms01-05-gestión-de-roles-y-permisos) máximo de 50 roles por usuario | Servicio: calcula los roles que cada usuario tendría después del cambio y, si alguno supera 50, rechaza toda la petición con 422 antes de escribir. La BD no lo restringe. |
| [RN-MS01-28](1_requirements.md#rf-ms01-10-administración-de-roles-y-catálogo-de-permisos) formato del nombre, nombre reservado, descripción y permisos de un rol | BD: `chh_role_name`, `chh_role_name_reserved`, `chh_role_description`, `uq_roles_name`; que cada permiso exista, FK `fk_role_permissions_permissions`. Servicio: lo verifica primero y responde 400 o 409. |
| [RN-MS01-29](1_requirements.md#rf-ms01-10-administración-de-roles-y-catálogo-de-permisos) roles de sistema | BD: trigger `trg_protect_system_roles`. Servicio: lo verifica primero y responde 409. La siembra es la única que escribe `is_system`. |
| [RN-MS01-30](1_requirements.md#rf-ms01-10-administración-de-roles-y-catálogo-de-permisos) `ms01:role:manage` solo en ADMIN | BD: trigger diferido `trg_check_role_permissions`. Servicio: lo verifica primero y responde 422. |
| [RN-MS01-41](1_requirements.md#rf-ms01-10-administración-de-roles-y-catálogo-de-permisos) todo rol conserva `ms01:profile:read` y `ms01:session:logout`, y `ESTUDIANTE` y `ADMIN` además `ms01:profile:write` | BD: trigger diferido `trg_check_role_permissions`. Servicio: al crear un rol agrega los dos obligatorios; al cambiar los permisos de un rol lo verifica primero y responde 422. |
| [RN-MS01-42](1_requirements.md#rf-ms01-10-administración-de-roles-y-catálogo-de-permisos) mantenimiento del catálogo de permisos | BD: FK con `ON DELETE CASCADE` en `role_permissions` y trigger diferido, que cancela el retiro de un permiso si deja a un rol sin los permisos obligatorios o sin sus mínimos. MS01 (`ms01:*`) y responsable de la base de datos (`ms02` a `ms08`): ver [`permissions`](#permissions). |
| [RN-MS01-31](1_requirements.md#rf-ms01-10-administración-de-roles-y-catálogo-de-permisos) borrar un rol sin dejar usuarios sin rol | BD: FK `fk_user_roles_roles` sin `ON DELETE`. Servicio: reasigna a ESTUDIANTE o retira el rol y borra, todo en una transacción. |
| [RN-MS01-35](1_requirements.md#rf-ms01-11-corte-de-sesión-por-cambios-de-acceso) cortes de sesión | Servicio: escribe `valid_from` en la misma transacción del cambio. BD: solo los índices parciales. |
| Al menos un rol por usuario; rol inicial ESTUDIANTE ([RN-MS01-02](1_requirements.md#rf-ms01-01-registrar-usuario-institucional)) | Servicio, dentro de la transacción. |
| Nadie cambia sus propios roles ni su estado ([RN-MS01-12](1_requirements.md#rf-ms01-05-gestión-de-roles-y-permisos), [RN-MS01-19](1_requirements.md#rf-ms01-07-administración-de-usuarios)) | Servicio: compara el id del solicitante con los ids objetivo. |
| [RN-MS01-21](1_requirements.md#rf-ms01-08-recuperación-y-cambio-de-contraseña) un solo código sin usar y un solo uso | Servicio: un código nuevo marca `used_at` en los anteriores; el restablecimiento marca el consumido. |
| [RN-MS01-03](1_requirements.md#rf-ms01-01-registrar-usuario-institucional) política de contraseña | Servicio, antes de calcular el hash. La BD solo ve el hash. |
| [RN-MS01-43](1_requirements.md#rf-ms01-01-registrar-usuario-institucional) un registro pendiente por correo, que se reemplaza | BD: PK `email` de `pending_registrations`. El servicio escribe con `INSERT ... ON CONFLICT (email) DO UPDATE`, una sola sentencia, así que dos solicitudes simultáneas nunca crean dos filas. |
| [RN-MS01-44](1_requirements.md#rf-ms01-01-registrar-usuario-institucional) espera mínima y límite diario de códigos | BD: la condición va en la misma sentencia que escribe el código (el `WHERE` del `ON CONFLICT DO UPDATE` y del `UPDATE` del reenvío), así que dos solicitudes simultáneas no envían más códigos de los permitidos. Servicio: cuenta los intentos fallidos con la fila bloqueada y, al llegar al máximo, invalida el código. |
| [RN-MS01-46](1_requirements.md#rf-ms01-01-registrar-usuario-institucional) solo quien hizo el registro lo confirma | Servicio: compara en tiempo constante el hash del token recibido con `request_token_hash` antes de mirar el código; un token que no corresponde no cuenta como intento. El reenvío solo actualiza la fila si el token coincide (`WHERE request_token_hash = ...`). |
| [RN-MS01-45](1_requirements.md#rf-ms01-01-registrar-usuario-institucional) limpieza de registros vencidos | Servicio: tarea programada que borra las filas con el código vencido y la ventana diaria terminada. |

---

## Operaciones transaccionales y consultas

Cada operación de escritura es una sola transacción: se aplica completa o no se aplica.

| Operación | Lecturas | Escrituras |
|---|---|---|
| Solicitar el registro ([RF-01](1_requirements.md#rf-ms01-01-registrar-usuario-institucional)) | ¿Existe el `email` en `users`? | `INSERT ... ON CONFLICT (email) DO UPDATE` de `pending_registrations` con los datos, los hashes (también el del token nuevo), `attempts = 0`, la vigencia, `code_sent_at` y el contador diario, solo si se cumplen la espera y el límite diario; si escribió la fila, envía el código después de confirmar la transacción |
| Confirmar el registro ([RF-01](1_requirements.md#rf-ms01-01-registrar-usuario-institucional)) | `pending_registrations` por `email`, con bloqueo de escritura; ¿coincide el token?; ¿existe el `email` en `users`? | Token que no corresponde: nada. Código incorrecto: `attempts + 1` (y `expires_at = now()` al llegar al máximo). Correcto: INSERT `users` (ACTIVO) + INSERT `user_roles` (ESTUDIANTE) + DELETE de la fila de `pending_registrations` |
| Reenviar el código ([RF-01](1_requirements.md#rf-ms01-01-registrar-usuario-institucional), A1) | — | UPDATE de `code_hash`, `attempts = 0`, `expires_at`, `code_sent_at` y el contador diario de la fila del `email`, solo si existe con ese token y se cumplen la espera y el límite diario |
| Limpiar registros pendientes (tarea periódica, [RN-MS01-45](1_requirements.md#rf-ms01-01-registrar-usuario-institucional)) | — | DELETE de `pending_registrations` con `expires_at < now()` y la ventana diaria terminada |
| Login ([RF-02](1_requirements.md#rf-ms01-02-autenticar-usuario-login)) | `users` por `email` + sus roles + los permisos de esos roles (`role_permissions` y `permissions`) | Ninguna |
| Actualizar perfil ([RF-06](1_requirements.md#rf-ms01-06-gestión-del-perfil)) | `users` por id | UPDATE de los 4 campos de nombre (`updated_at` lo fija el trigger) |
| Cambio de estado en lote ([RF-07](1_requirements.md#rf-ms01-07-administración-de-usuarios)) | Usuarios objetivo con bloqueo de escritura | UPDATE `status` y `valid_from` de cada usuario |
| Cambio de roles en lote ([RF-05](1_requirements.md#rf-ms01-05-gestión-de-roles-y-permisos)) | Usuarios objetivo con bloqueo de escritura | Actualiza las filas de `user_roles` de los usuarios cuyos roles cambian (`agregar` / `retirar`) y su `valid_from` |
| Solicitar código ([RF-08](1_requirements.md#rf-ms01-08-recuperación-y-cambio-de-contraseña)) | `users` por `email`; último código del usuario (espera) | Si está ACTIVO y fuera de la espera: marca `used_at` en los códigos sin usar + INSERT del nuevo |
| Restablecer contraseña | Último código sin usar, con bloqueo de escritura | Código incorrecto: `attempts + 1` (y `used_at` al llegar al máximo). Correcto: UPDATE `password_hash` + `used_at` del código y de los demás sin usar |
| Cambiar contraseña autenticado | `users` por id | UPDATE `password_hash` |
| Consultar roles y permisos ([RF-05](1_requirements.md#rf-ms01-05-gestión-de-roles-y-permisos), [RF-10](1_requirements.md#rf-ms01-10-administración-de-roles-y-catálogo-de-permisos)) | `roles` con sus permisos (uno o todos) y, para el catálogo, `permissions` | Ninguna |
| Crear rol ([RF-10](1_requirements.md#rf-ms01-10-administración-de-roles-y-catálogo-de-permisos)) | ¿Existe el `name`? Los permisos pedidos en `permissions` | INSERT `roles` (`is_system` falso, `valid_from` nulo) + INSERT `role_permissions` (una fila por permiso pedido, más `ms01:profile:read` y `ms01:session:logout` si no venían) |
| Renombrar o describir un rol ([RF-10](1_requirements.md#rf-ms01-10-administración-de-roles-y-catálogo-de-permisos)) | `roles` por `name`; ¿existe el nombre nuevo? | UPDATE `roles` (`name`, `description`; `updated_at` lo fija el trigger); `user_roles` no cambia porque apunta al `id`; si el nombre cambia, se escribe `valid_from` de cada usuario con ese rol |
| Cambiar permisos de un rol ([RF-10](1_requirements.md#rf-ms01-10-administración-de-roles-y-catálogo-de-permisos)) | `roles` por `name`; los permisos pedidos en `permissions` | DELETE de las filas de `role_permissions` que sobran + INSERT de las nuevas + UPDATE `roles.valid_from`. Las reglas de los permisos de un rol se evalúan al confirmar ([`role_permissions`](#role_permissions)) |
| Borrar un rol ([RF-10](1_requirements.md#rf-ms01-10-administración-de-roles-y-catálogo-de-permisos)) | `roles` por `name`; usuarios que lo tienen | Por cada usuario: UPDATE `user_roles` a ESTUDIANTE (si era su único rol) o DELETE de esa fila (si tenía varios) y UPDATE `users.valid_from`; luego DELETE `roles` (sus filas de `role_permissions` se borran en cascada). Una sola transacción: si la BD rechaza algo, se revierte todo |
| Sincronizar los permisos de MS01 (arranque, [RN-MS01-42](1_requirements.md#rf-ms01-10-administración-de-roles-y-catálogo-de-permisos)) | `permissions` con prefijo `ms01:` y los roles que tienen cada permiso por borrar | INSERT de los `ms01:*` que faltan; por cada `ms01:*` que ya no está en el enum: UPDATE `roles.valid_from` de los roles que lo tienen y DELETE de `permissions` (cascada sobre `role_permissions`). Una sola transacción: si la BD rechaza algo, se revierte todo y MS01 no arranca |
| Retirar un permiso (script del responsable de la base de datos, [RN-MS01-42](1_requirements.md#rf-ms01-10-administración-de-roles-y-catálogo-de-permisos)) | Los roles que tienen el permiso | UPDATE `roles.valid_from` de esos roles + DELETE de `permissions` (cascada sobre `role_permissions`). Una sola transacción: si deja a un rol sin los permisos obligatorios o sin los mínimos de `ADMIN` o `ESTUDIANTE`, se revierte todo y el error nombra el rol |
| Consulta interna de cortes de sesión ([RF-11](1_requirements.md#rf-ms01-11-corte-de-sesión-por-cambios-de-acceso)) | `users` y `roles` con `valid_from` dentro de la ventana de 3700 s | Ninguna |
| Consulta interna por ids ([RF-09](1_requirements.md#rf-ms01-09-resolución-interna-de-usuarios)) | `users` y sus roles con `id` en la lista (hasta 100) | Ninguna |
| Consulta interna por rol ([RF-09](1_requirements.md#rf-ms01-09-resolución-interna-de-usuarios)) | `users` con una fila en `user_roles` del rol pedido (se resuelve por `roles.name`) y, si hay fecha de corte, `created_at <= corte`; orden `id` ascendente | Ninguna |

**Listado `GET /users`** (parámetros en [F-07](2_forms-inputs.md#f-07-búsqueda-y-filtros-de-usuarios)):
* `correo` (columna `users.email`): coincidencia parcial sin distinguir mayúsculas (`LIKE '%texto%'` sobre el correo en minúsculas).
* `estado` (columna `users.status`) / `rol`: igualdad; `rol` busca usuarios con una fila de ese rol en `user_roles` (el nombre se resuelve a `roles.id`).
* Orden: `created_at` descendente y luego `id`.
* Cada elemento incluye todos sus roles, no solo el rol filtrado.

---

## Configuración

Variables de entorno que afectan los datos (valores por defecto de `application.yaml`):

| Variable | Por defecto | Uso |
|---|---|---|
| `INSTITUTIONAL_EMAIL_DOMAINS` | `escuelaing.edu.co,mail.escuelaing.edu.co` | Dominios permitidos en el registro. |
| `REGISTRATION_CODE_TTL_MINUTES` | `15` | Vigencia del código de verificación del registro ([RN-MS01-44](1_requirements.md#rf-ms01-01-registrar-usuario-institucional)). |
| `REGISTRATION_MAX_ATTEMPTS` | `5` | Intentos fallidos permitidos antes de invalidar el código de verificación. |
| `REGISTRATION_RESEND_COOLDOWN_SECONDS` | `60` | Espera mínima entre dos códigos de verificación del mismo correo (`0` la desactiva). |
| `REGISTRATION_MAX_CODES_PER_DAY` | `5` | Códigos de verificación por correo en una ventana de 24 horas. |
| `REGISTRATION_CLEANUP_INTERVAL_MINUTES` | `60` | Cada cuántos minutos se borran los registros pendientes vencidos ([RN-MS01-45](1_requirements.md#rf-ms01-01-registrar-usuario-institucional)). |
| `RECOVERY_CODE_TTL_MINUTES` | `15` | Vigencia del código ([RN-MS01-21](1_requirements.md#rf-ms01-08-recuperación-y-cambio-de-contraseña)). |
| `RECOVERY_MAX_ATTEMPTS` | `5` | Intentos fallidos permitidos antes de invalidar el código. |
| `RECOVERY_RESEND_COOLDOWN_SECONDS` | `60` | Espera mínima entre dos códigos del mismo usuario (`0` la desactiva). |

---

## Datos iniciales

1. **Catálogo de permisos y roles iniciales** ([RN-MS01-34](1_requirements.md#rf-ms01-10-administración-de-roles-y-catálogo-de-permisos)): el script `001` (bases nuevas) carga, en una sola transacción, los permisos del catálogo (de `ms01` a `ms08`) con su descripción, los ocho roles y la asignación de permisos de la [matriz de roles y permisos](../integration/permissions-matrix.md). Tiene que ser una sola transacción porque las reglas de los permisos de un rol ([`role_permissions`](#role_permissions)) se evalúan al confirmar. Es una siembra única: después de cargada, la fuente son las tablas `permissions`, `roles` y `role_permissions`, y el administrador puede cambiar descripciones y permisos de los roles. Solo `ESTUDIANTE` y `ADMIN` se siembran con `is_system = TRUE`.

   | Rol | `is_system` | Permisos | Descripción inicial |
   |---|---|---|---|
   | `ESTUDIANTE` | Sí | 18 | Estudiante de la comunidad universitaria |
   | `PROFESOR` | No | 19 | Profesor de la comunidad universitaria |
   | `ADMINISTRATIVO` | No | 18 | Personal administrativo de la comunidad universitaria |
   | `BRIGADISTA` | No | 25 | Brigadista de emergencias del campus |
   | `COORDINADOR` | No | 35 | Coordinador de emergencias: gestiona alertas, brigadistas y asistencia |
   | `ORGANIZADOR` | No | 34 | Organizador de simulacros y eventos de evacuación |
   | `ADMIN` | Sí | 27 | Administrador del sistema: gestiona usuarios, roles, campus y contenidos |
   | `AUDITOR` | No | 22 | Auditor: consulta históricos, indicadores y auditoría (solo lectura) |

   Los códigos de permiso de cada rol están en el script y en la matriz de roles y permisos (58 permisos en el catálogo). `ms01:role:manage` queda solo en `ADMIN`, los ocho roles llevan `ms01:profile:read` y `ms01:session:logout`, y `ESTUDIANTE` y `ADMIN` llevan además `ms01:profile:write`.
2. **Cambios de la Matriz:** `database/maintenance/cambio_permisos.sql` es un script manual, de una sola transacción, que lleva una base que ya existe (prueba y producción) desde la Matriz anterior a la actual con solo lo que cambió: agrega y retira permisos del catálogo, da y quita permisos a los ocho roles iniciales y corta las sesiones solo de los roles afectados. No toca los roles creados desde la API ni los demás cambios del administrador, y se puede aplicar más de una vez sin efecto. Los usuarios de los roles afectados inician sesión de nuevo y reciben sus permisos nuevos. Antes de aplicarlo conviene guardar una copia de la base de datos.
3. **Primer administrador** ([RN-MS01-12](1_requirements.md#rf-ms01-05-gestión-de-roles-y-permisos)): se crea al arrancar solo si no hay ningún usuario con el rol ADMIN y están definidas `BOOTSTRAP_ADMIN_EMAIL` y `BOOTSTRAP_ADMIN_PASSWORD` (nunca en el repositorio). Queda ACTIVO, con el rol ADMIN. `BOOTSTRAP_ADMIN_FIRST_NAME` y `BOOTSTRAP_ADMIN_LAST_NAME` son opcionales (por defecto "Administrador" e "Inicial"). Si el correo ya pertenece a otro usuario, no se crea ni se le asciende. Como no pasa por el registro, no se le aplican la política de contraseña ([RN-MS01-03](1_requirements.md#rf-ms01-01-registrar-usuario-institucional)) ni el dominio institucional: quien configura las variables debe usar un correo institucional y una contraseña que los cumpla.
4. **Códigos de recuperación:** los códigos usados o vencidos no se eliminan y permanecen en `password_reset_codes`. Los registros pendientes vencidos sí se limpian con una tarea programada ([RN-MS01-45](1_requirements.md#rf-ms01-01-registrar-usuario-institucional)).
