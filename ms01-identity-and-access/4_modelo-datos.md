# Especificación del Modelo de Datos

MS01 tiene su propia base de datos PostgreSQL ([RNF-MS01-01](1_requirements.md#requerimientos-no-funcionales-rnf)). Ningún otro microservicio la lee ni la modifica: solo se accede a ella mediante la API. Esquema desplegado: `database/` del repositorio de MS01. Diagrama: [modelo físico](diagramas/ms01_er_fisico.drawio.svg).

## Índice de Contenido
1. [Alcance](#alcance)
2. [Convenciones](#convenciones)
3. [Almacenamiento seguro](#almacenamiento-seguro)
4. [Tablas](#tablas)
    - [usuarios](#usuarios)
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

**Se almacena:** usuarios, sus roles y los códigos de recuperación de contraseña.

**No se almacena:**
* **Auditoría:** la delega MS08 ([RNF-MS01-10](1_requirements.md#requerimientos-no-funcionales-rnf)).
* **Permisos:** la matriz rol–permiso está versionada en el código ([RN-MS01-14](1_requirements.md#rf-ms01-05-gestión-de-roles-y-permisos)).
* **Sesiones y tokens:** el token es stateless y no se persiste ([`3_jwt-spec.md`](3_jwt-spec.md)).
* **Datos de otros dominios** (alertas, check-ins, etc.): pertenecen a sus propios microservicios.

---

## Convenciones

* Tablas y columnas en `snake_case` y en español, alineadas con el contrato OpenAPI.
* Fechas y horas en `TIMESTAMPTZ`, guardadas en UTC.
* Los identificadores son UUID. La aplicación los genera al insertar, y las columnas también tienen `DEFAULT gen_random_uuid()`. El `id` del usuario es el mismo valor del `id` de la API, del claim `sub` y de `X-User-Id`.
* El correo se guarda en minúsculas y sin espacios en los extremos ([RG-01](2_forms-inputs.md#reglas-generales-de-entrada)).
* Los roles se guardan como texto (`@Enumerated(EnumType.STRING)`, nunca `ORDINAL`) y la base de datos los valida con un `CHECK`. Cambiar el catálogo exige actualizar el enum y el `CHECK` en el mismo Pull Request.
* Los usuarios no se borran físicamente: se inactivan ([RN-MS01-17](1_requirements.md#rf-ms01-07-administración-de-usuarios)).
* El esquema se versiona como scripts SQL numerados en `database/` (`001_create_schema.sql`, `002_index_schema.sql`, `003_codigos_recuperacion.sql`). La aplicación no crea ni modifica el esquema: lo valida al arrancar (`spring.jpa.hibernate.ddl-auto: validate`).

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

**Índices:** PK `id`; único `correo`; `idx_usuarios_correo_trgm` (GIN `pg_trgm`, para la búsqueda parcial por correo); `idx_usuarios_estado (estado)`.

### usuario_rol
Roles de cada usuario, con valores del enum `UserRole`. Un usuario puede tener varios ([RN-MS01-13](1_requirements.md#rf-ms01-05-gestión-de-roles-y-permisos)).

| Columna | Tipo | Nulo | Por defecto | Restricciones | Descripción |
|---|---|---|---|---|---|
| `usuario_id` | UUID | No | — | PK (`pk_rol`, con `rol`); FK → `usuarios.id` (`fk_rol`) | |
| `rol` | VARCHAR(20) | No | — | PK (con `usuario_id`); CHECK IN (`ESTUDIANTE`, `PROFESOR`, `ADMINISTRATIVO`, `BRIGADISTA`, `COORDINADOR`, `ORGANIZADOR`, `ADMIN`, `AUDITOR`) | Código del rol. |
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
| `usuarios` → `codigos_recuperacion` | 1 a N (a lo sumo uno sin usar a la vez) | Sin cascada |

---

## Correspondencia entre API y columnas

Los campos del contrato tienen el mismo nombre que su columna en `usuarios` (`id`, nombres y apellidos, `correo`, `estado`), salvo estos:

| Campo del contrato | Columna | Notas |
|---|---|---|
| `contrasena`, `nueva_contrasena` | `usuarios.password_hash` | Solo entra. No se devuelve. |
| `roles` | `usuario_rol.rol` (filas del usuario) | Arreglo en orden alfabético (igual que en el token). |
| `codigo` | `codigos_recuperacion.codigo_hash` | Se compara contra el hash. No se devuelve. |
| `expira_en` (login) | — | Sale del claim `exp` del token, no de la BD. |

---

## Reglas de integridad

| Regla | Dónde se garantiza |
|---|---|
| [RN-MS01-01](1_requirements.md#rf-ms01-01-registrar-usuario-institucional) correo único | BD: UNIQUE + `correo = lower(correo)`. El servicio normaliza el correo antes de guardar. |
| [RN-MS01-17](1_requirements.md#rf-ms01-07-administración-de-usuarios) estados válidos | BD: CHECK. |
| [RN-MS01-14](1_requirements.md#rf-ms01-05-gestión-de-roles-y-permisos) roles del catálogo, sin repetir | BD: CHECK y PK compuesta. Código: enum `UserRole`. |
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
| Login ([RF-02](1_requirements.md#rf-ms01-02-autenticar-usuario-login)) | `usuarios` por `correo` + sus roles | Ninguna |
| Actualizar perfil ([RF-06](1_requirements.md#rf-ms01-06-gestión-del-perfil)) | `usuarios` por id | UPDATE de los 4 campos de nombre + `actualizado_en` |
| Cambio de estado en lote ([RF-07](1_requirements.md#rf-ms01-07-administración-de-usuarios)) | Usuarios objetivo con bloqueo de escritura | UPDATE `estado` de cada usuario |
| Cambio de roles en lote ([RF-05](1_requirements.md#rf-ms01-05-gestión-de-roles-y-permisos)) | Usuarios objetivo con bloqueo de escritura | Actualiza las filas de `usuario_rol` de los usuarios cuyos roles cambian (`agregar` / `retirar`) |
| Solicitar código ([RF-08](1_requirements.md#rf-ms01-08-recuperación-y-cambio-de-contraseña)) | `usuarios` por `correo`; último código del usuario (espera) | Si está ACTIVO y fuera de la espera: marca `usado_en` en los códigos sin usar + INSERT del nuevo |
| Restablecer contraseña | Último código sin usar, con bloqueo de escritura | Código incorrecto: `intentos + 1` (y `usado_en` al llegar al máximo). Correcto: UPDATE `password_hash` + `usado_en` del código y de los demás sin usar |
| Cambiar contraseña autenticado | `usuarios` por id | UPDATE `password_hash` |
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

1. **Primer administrador** ([RN-MS01-12](1_requirements.md#rf-ms01-05-gestión-de-roles-y-permisos)): se crea al arrancar solo si no hay ningún usuario con el rol ADMIN y están definidas `BOOTSTRAP_ADMIN_EMAIL` y `BOOTSTRAP_ADMIN_PASSWORD` (nunca en el repositorio). Queda ACTIVO, con el rol ADMIN. `BOOTSTRAP_ADMIN_FIRST_NAME` y `BOOTSTRAP_ADMIN_LAST_NAME` son opcionales (por defecto "Administrador" e "Inicial"). Si el correo ya pertenece a otro usuario, no se crea ni se le asciende.
2. **Limpieza opcional:** una tarea periódica puede eliminar los códigos usados o vencidos con más de 24 horas de antigüedad (no implementada).
