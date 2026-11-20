# Matriz de roles y permisos

**Fuente única de permisos de CampusSeguro.** Origen: *Propuesta DOSW · CampusSeguro* (actores y funcionalidades). Los roles iniciales y las reglas de autorización transversales están en [Estándares](standards.md). Esta página es el **registro oficial de los códigos de permiso** y de la **asignación inicial** de permisos a los 8 roles iniciales. El catálogo de permisos vive en la base de datos de MS01, que es su fuente de verdad, y es de solo lectura para el administrador. Cada squad mantiene aquí los permisos que su microservicio realmente exige; el responsable de la base de datos agrega al catálogo los que faltan y retira los que ya no figuran aquí ([MR-10](#reglas-de-aplicación)). MS01 carga la asignación inicial como datos iniciales y, desde entonces, los roles y sus permisos se administran en ejecución con el permiso `ms01:role:manage` ([RF-MS01-10](../ms01-identity-and-access/1_requirements.md#rf-ms01-10-administración-de-roles-y-catálogo-de-permisos), [RN-MS01-14](../ms01-identity-and-access/1_requirements.md#rf-ms01-05-gestión-de-roles-y-permisos)): las columnas de cada rol son indicativas, porque muestran lo que tiene al cargar los datos iniciales, no necesariamente lo que tiene hoy. Cada cambio a un código de permiso o a la asignación inicial se acuerda con el squad dueño del módulo antes de editar la página y exige actualizar el `x-permiso` de su OpenAPI.

## Índice de Contenido
1. [Convenciones](#convenciones)
2. [Matriz por módulo](#matriz-por-módulo)
3. [Reglas de aplicación](#reglas-de-aplicación)

---

## Convenciones

* **Código de permiso:** `<ms>:<recurso>:<acción>` (p. ej. `ms02:building:write`). Estos mismos códigos van en el `x-permiso` del OpenAPI de cada MS y son los que devuelve `GET /api/v1/roles` de MS01 en `permisos`. El catálogo completo, con la descripción de cada permiso, lo devuelve `GET /api/v1/permissions`.
* **Roles iniciales (8):** ESTUDIANTE · PROFESOR · ADMINISTRATIVO · BRIGADISTA · COORDINADOR · ORGANIZADOR · ADMIN · AUDITOR. Son los que MS01 crea al desplegar; después el administrador puede crear otros roles y editar los permisos de cualquiera ([RF-MS01-10](../ms01-identity-and-access/1_requirements.md#rf-ms01-10-administración-de-roles-y-catálogo-de-permisos)). **ESTUDIANTE y ADMIN son roles de sistema:** no se pueden borrar ni renombrar ([MR-09](#reglas-de-aplicación)). Columnas: **EST** · **PROF** · **ADMTV** · **BRIG** · **COORD** · **ORG** · **ADMIN** · **AUD** (ADMTV = ADMINISTRATIVO).
* **✅** = permitido. Celda vacía = denegado (denegado por defecto).
* **COM** (comunidad en campus) = EST, PROF, ADMTV, BRIG, COORD, ORG.
* **Escaleras:** no tienen un permiso propio. Se consultan con `ms02:campus:read`, se gestionan con `ms02:building:write` y su disponibilidad (bloquearlas o habilitarlas) se cambia con `ms02:route_status:write`.
* **Recomendaciones preventivas:** no tienen un permiso de escritura propio. Se crean y modifican con `ms02:building:write` y se consultan con `ms02:preventive_info:read`.
* **Asignación inicial:** las columnas describen lo que cada rol inicial tiene al cargar los datos iniciales y son indicativas, porque los roles se editan después desde la interfaz de administración. Lo que un rol tiene en ejecución (incluidos los roles creados después) se consulta en `GET /api/v1/roles/details` de MS01 (`ms01:role:read_all`), no en esta página.

---

## Matriz por módulo

### MS01 · Identidad y acceso

| Permiso | Descripción | EST | PROF | ADMTV | BRIG | COORD | ORG | ADMIN | AUD |
|---|---|:-:|:-:|:-:|:-:|:-:|:-:|:-:|:-:|
| `ms01:profile:read` | Consultar el propio perfil | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| `ms01:profile:write` | Actualizar el propio perfil (nombres y apellidos) | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| `ms01:password:change` | Cambiar la propia contraseña | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| `ms01:session:logout` | Cerrar sesión | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| `ms01:role:read_own` | Consultar los propios roles y sus permisos | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| `ms01:role:read_all` | Consultar el catálogo completo de roles, el detalle de cada rol y el catálogo de permisos |  |  |  |  |  |  | ✅ |  |
| `ms01:role:manage` | Crear, renombrar, actualizar y borrar roles y asignarles permisos |  |  |  |  |  |  | ✅ |  |
| `ms01:user:read` | Listar, buscar y consultar usuarios |  |  |  |  |  |  | ✅ |✅ |
| `ms01:user:status` | Activar o inactivar usuarios |  |  |  |  |  |  | ✅ |  |
| `ms01:user:roles` | Agregar o retirar roles a usuarios |  |  |  |  |  |  | ✅ |  |

### MS02 · Campus y seguridad

| Permiso | Descripción | EST | PROF | ADMTV | BRIG | COORD | ORG | ADMIN | AUD |
|---|---|:-:|:-:|:-:|:-:|:-:|:-:|:-:|:-:|
| `ms02:campus:read` | Consultar campus, zonas abiertas, edificios, pisos, espacios, salidas, escaleras, zonas seguras (puntos de encuentro) e imágenes | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| `ms02:route:read` | Consultar la ruta de evacuación recomendada y listar o ver las rutas de evacuación | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| `ms02:preventive_info:read` | Consultar información preventiva por edificio o espacio | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| `ms02:building:write` | Crear, modificar e inactivar campus, zonas abiertas, edificios, pisos, salidas, escaleras (sin su disponibilidad), imágenes y recomendaciones preventivas |  |  |  |  |  |  | ✅ |  |
| `ms02:space:write` | Gestionar espacios |  |  |  |  |  |  | ✅ |  |
| `ms02:safe_zone:write` | Gestionar zonas seguras y puntos de encuentro (un mismo recurso) |  |  |  |  |  |  | ✅ |  |
| `ms02:route:write` | Asociar espacios con rutas de evacuación y modificar su descripción o destino |  |  |  |  |  |  | ✅ |  |
| `ms02:route_status:write` | Cambiar el estado de una ruta (disponible, bloqueada, restringida) y la disponibilidad de una escalera (bloquearla o habilitarla) |  |  |  |  | ✅ |  | ✅ |  |
| `ms02:qr:manage` | Generar el código QR de espacios, salidas y zonas seguras |  |  |  |  |  |  | ✅ |  |

### MS03 · Alertas y emergencias

| Permiso | Descripción | EST | PROF | ADMTV | BRIG | COORD | ORG | ADMIN | AUD |
|---|---|:-:|:-:|:-:|:-:|:-:|:-:|:-:|:-:|
| `ms03:alert:receive` | Recibir alertas y confirmar su recepción | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |  |  |
| `ms03:alert:manage` | Crear, programar, activar, segmentar, modificar y finalizar alertas de cualquier tipo |  | ✅ |  |  | ✅ | ✅ |  |  |
| `ms03:alert:manage_drill` | Igual que el anterior, solo para alertas de tipo SIMULACRO (originadas desde MS06) |  |  |  |  |  | ✅ |  |  |
| `ms03:alert:read_history` | Consultar el historial de alertas | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |

### MS04 · Evacuación y check-in

| Permiso | Descripción | EST | PROF | ADMTV | BRIG | COORD | ORG | ADMIN | AUD |
|---|---|:-:|:-:|:-:|:-:|:-:|:-:|:-:|:-:|
| `ms04:checkin:write` | Reportar y actualizar el propio estado y punto de encuentro | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |  |  |
| `ms04:checkin:read_own` | Consultar el propio estado vigente | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |  |  |
| `ms04:assistance:request` | Solicitar asistencia desde el check-in | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |  |  |
| `ms04:dashboard:read` | Ver consolidación por punto de encuentro, personas pendientes y dashboard |  |  |  | ✅ | ✅ | ✅ |  | ✅ |
| `ms04:history:read` | Consultar el histórico de check-ins de un evento |  |  |  |  | ✅ | ✅ |  | ✅ |

### MS05 · Brigadistas e incidentes

| Permiso | Descripción | EST | PROF | ADMTV | BRIG | COORD | ORG | ADMIN | AUD |
|---|---|:-:|:-:|:-:|:-:|:-:|:-:|:-:|:-:|
| `ms05:brigadier:assign_to_drill` | Asignar brigadistas a edificios, zonas o puntos de encuentro para simulacros |  |  |  |  |  | ✅ |  |  |
| `ms05:brigadier:assign_to_real` | Asignar brigadistas a edificios, zonas o puntos de encuentro para eventos reales|  |  |  |  | ✅ |  |  |  |
| `ms05:panel:read_own` | Consultar las propias zonas y tareas asignadas |  |  |  | ✅ |  |  |  |  |
| `ms05:assistance:read` | Consultar solicitudes de asistencia |  |  |  | ✅ | ✅ | ✅ |  |  |
| `ms05:assistance:register` | Registrar una solicitud de asistencia para alguien diferente a si mismo |  |  |  | ✅ |  |  |  |  |
| `ms05:assistance:assign_to_drill` | Asignar una solicitud a un brigadista y cerrarla para un simulacro|  |  |  |  |  | ✅ |  |  |
| `ms05:assistance:assign_to_real` | Asignar una solicitud a un brigadista y cerrarla para un evento real|  |  |  |  | ✅ |  |  |  |
| `ms05:assistance:update_status` | Cambiar el estado de atención (en atención, resuelta) |  |  |  | ✅ | ✅ | ✅ |  |  |
| `ms05:incident:report` | Registrar un incidente propio con descripción y evidencia | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |  |
| `ms05:incident:manage_own` | Consultar, editar y eliminar incidentes propios| ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| `ms05:incident:read` | Consultar y dar seguimiento a incidentes |  |  |  | ✅ | ✅ | ✅ |  | ✅ |
| `ms05:incident:update` | Actualizar un incidente |  |  |  | ✅ | ✅ |  |  |  |


### MS06 · Eventos

| Permiso | Descripción | EST | PROF | ADMTV | BRIG | COORD | ORG | ADMIN | AUD |
|---|---|:-:|:-:|:-:|:-:|:-:|:-:|:-:|:-:|
| `ms06:drill:manage` | Crear y editar evento con tipo `SIMULACRO`, escenario y participantes |  |  |  |  |  | ✅ |  |  |
| `ms06:drill:execute` | Iniciar y terminar un evento con tipo `SIMULACRO`|  |  |  |  |  | ✅ |  |  |
| `ms06:drill:evaluate` | Evaluar un evento con tipo `SIMULACRO`|  |  |  |  |  | ✅ |  |  |
| `ms06:drill:read` | Consultar eventos con tipo `SIMULACRO` y su seguimiento |  |  |  |  |  | ✅ |  | ✅ |
| `ms06:drill:compare_report` | Comparar eventos con tipo `SIMULACRO` y consultar el informe final |  |  |  |  |  | ✅ |  | ✅ |
| `ms06:real_event:manage` | Crear y editar evento real (sin tipo `SIMULACRO`), escenario y participantes |  |  |  |  | ✅ |  |  |  |
| `ms06:real_event:execute` | Iniciar y terminar un evento real (sin tipo `SIMULACRO`)|  |  |  |  | ✅ |  |  |  |
| `ms06:real_event:evaluate` | Evaluar un evento real (sin tipo `SIMULACRO`)|  |  |  |  | ✅ |  |  |  |
| `ms06:real_event:read` | Consultar eventos reales (sin tipo `SIMULACRO`) y su seguimiento |  |  |  |  | ✅ |  |  | ✅ |
| `ms06:real_event:compare_report` | Comparar eventos reales (sin tipo `SIMULACRO`) y consultar el informe final |  |  |  |  | ✅ |  |  | ✅ |

### MS07 · Formación y asistente IA

| Permiso | Descripción | EST | PROF | ADMTV | BRIG | COORD | ORG | ADMIN | AUD |
|---|---|:-:|:-:|:-:|:-:|:-:|:-:|:-:|:-:|
| `ms07:content:read` | Consultar contenidos del centro de aprendizaje | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |  |  |
| `ms07:quiz:take` | Realizar evaluaciones y consultar el propio progreso | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |  |  |
| `ms07:tutor:use` | Usar el tutor CampusSeguro | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |  |  |
| `ms07:content:manage` | Gestionar contenidos y la información que alimenta al tutor |  |  |  |  |  |  | ✅ |  |

### MS08 · Analítica y auditoría

| Permiso | Descripción | EST | PROF | ADMTV | BRIG | COORD | ORG | ADMIN | AUD |
|---|---|:-:|:-:|:-:|:-:|:-:|:-:|:-:|:-:|
| `ms08:audit_drill:read` | Consultar auditoría de simulacros |   |  |  |  |  | ✅ | ✅ | ✅ |
| `ms08:audit_real_event:read` | Consultar auditoría de eventos reales (con o sin `evento_id`) |  |  |  |  | ✅ |  | ✅ | ✅ |
| `ms08:indicators_drill:read` | Ver indicadores y comparaciones de simulacros |  |  |  |  |  | ✅ | ✅ | ✅ |
| `ms08:indicators_real_event:read` | Ver indicadores de eventos reales |  |  |  |  | ✅ |  | ✅ | ✅ |

---

## Reglas de aplicación

* **MR-01 · Multi-rol:** un usuario con varios roles tiene la unión de sus permisos (RN-MS01-13). Ningún rol hereda a otro.
* **MR-02 · Alcance propio:** los permisos sobre datos propios (`*_own`, `ms01:profile:*`, `ms01:password:change`, `ms01:session:logout`, `ms04:checkin:*`, `ms04:assistance:request`, `ms07:quiz:take`) solo aplican a recursos del usuario identificado por `X-User-Id`.
* **MR-03 · Alcance de brigadista:** `dashboard:read`, `assistance:read` y `assistance:update_status` del BRIG se limitan a sus zonas o solicitudes asignadas (lo valida MS04/MS05).
* **MR-04 · Llamadas internas:** las llamadas entre servicios pasan por el Gateway y propagan la identidad original ([Estándares](standards.md)). Si el destino es una ruta **protegida**, se autoriza con los permisos de la identidad propagada (`X-User-Permission`, que para los roles iniciales parte de esta matriz): por eso el ORG necesita `ms03:alert:manage_drill` (MS06 → MS03), que solo opera sobre alertas de tipo SIMULACRO. Si el destino es una ruta **interna**, se autoriza por servicio (`X-Service-Name`), no por permisos: por ejemplo, la consulta de usuarios por rol que MS03, MS04 y MS05 hacen a MS01. La solicitud de asistencia que MS04 envía a MS05 va a una ruta **protegida**: MS05 la autoriza con la identidad del usuario que hace el check-in y su permiso `ms05:incident:report` (`ms05:assistance:register` es solo para el registro manual de BRIG).
* **MR-05 · Sin autoescalamiento:** nadie modifica sus propios roles ni, siendo ADMIN, su propio estado (RN-MS01-12, RN-MS01-19).
* **MR-06 · Vigencia:** un cambio de los roles de un usuario, de los permisos de un rol o del estado de una cuenta corta las sesiones afectadas en pocos segundos (7 s como máximo) y el siguiente token emitido ya lo refleja ([RF-MS01-11](../ms01-identity-and-access/1_requirements.md#rf-ms01-11-corte-de-sesión-por-cambios-de-acceso), RN-MS01-09). Los demás cambios (crear un rol, editar solo su descripción) no cortan sesiones.
* **MR-07 · Verificación:** cada microservicio autoriza sus endpoints leyendo `X-User-Permission` (permisos separados por coma, ya con la unión de todos los roles del usuario) contra el `x-permiso` de la operación y, si falta, responde `403 PROHIBIDO`. No autoriza por nombre de rol ni por `x-roles`, porque los roles ya no son un conjunto fijo. El Gateway no autoriza por rol ni por permiso.
* **MR-08 · Gestión de roles exclusiva de ADMIN:** `ms01:role:manage` pertenece solo a ADMIN y ADMIN siempre lo conserva; ningún otro rol puede tenerlo ([RN-MS01-30](../ms01-identity-and-access/1_requirements.md#rf-ms01-10-administración-de-roles-y-catálogo-de-permisos)).
* **MR-09 · Roles de sistema:** ESTUDIANTE y ADMIN no se pueden borrar ni renombrar, y su condición de sistema no cambia por la API ([RN-MS01-29](../ms01-identity-and-access/1_requirements.md#rf-ms01-10-administración-de-roles-y-catálogo-de-permisos)). Los otros seis roles iniciales se pueden editar, renombrar y borrar.
* **MR-10 · Altas y retiros de permisos:** esta Matriz es el registro oficial de los permisos. Cada squad agrega en su módulo los permisos que su microservicio exige y quita los que ya no usa, y mantiene su `x-permiso` igual a la Matriz. El responsable de la base de datos lleva cada cambio de esta Matriz a MS01 con el script de cambio (`database/maintenance/cambio_permisos.sql` del repositorio de MS01): agrega al catálogo los permisos de `ms02` a `ms08` que aparecen aquí y aún no están, retira los que ya no aparecen y da o quita a los ocho roles iniciales lo que cambió en sus columnas, sin tocar los roles creados por el administrador ni sus demás cambios. Mientras un permiso siga en la Matriz se mantiene en la base de datos, aunque ningún rol lo tenga: si un squad olvida quitar un permiso que ya no usa, la base lo conserva. Los permisos de MS01 (`ms01:*`) los define MS01 en su código y los sincroniza con la base de datos al arrancar. Un permiso nuevo no queda asignado a ningún rol por sí solo: en una base nueva lo asigna la siembra según esta Matriz y en una existente lo asigna el script de cambio a los ocho roles iniciales o lo concede el administrador con `PUT /api/v1/roles/{rol}/permissions`. Retirar un permiso lo quita de todos los roles que lo tenían y corta sus sesiones; si dejara a un rol sin `ms01:profile:read` o `ms01:session:logout`, a ADMIN sin `ms01:role:manage`, o a ESTUDIANTE o ADMIN sin `ms01:profile:write`, el retiro no se aplica y el error nombra el rol ([RN-MS01-42](../ms01-identity-and-access/1_requirements.md#rf-ms01-10-administración-de-roles-y-catálogo-de-permisos)).
* **MR-11 · Permisos obligatorios de los roles:** todo rol, incluidos los que crea el administrador, conserva siempre `ms01:profile:read` y `ms01:session:logout`, el mínimo para que cualquier cuenta consulte su perfil y cierre sesión; ESTUDIANTE y ADMIN conservan además `ms01:profile:write`. Al crear un rol, MS01 agrega los dos primeros aunque no se envíen; al reemplazar los permisos de un rol, la lista debe incluirlos o MS01 responde 422 ([RN-MS01-41](../ms01-identity-and-access/1_requirements.md#rf-ms01-10-administración-de-roles-y-catálogo-de-permisos)).
