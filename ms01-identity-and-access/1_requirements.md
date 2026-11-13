# Requerimientos — MS01 Identidad y Acceso

Este documento define solo el comportamiento de MS01. Los valores transversales (cabeceras, errores, correlación, paginación, auditoría) están en los [Estándares de Integración](https://github.com/CampusSeguro-DOSW-2026-2/documentation/wiki/Est%C3%A1ndares-de-Integraci%C3%B3n-y-Contratos-Transversales) y los permisos en la [Matriz Rol‐Permiso](https://github.com/CampusSeguro-DOSW-2026-2/documentation/wiki/Matriz-Rol%E2%80%90Permiso). Los detalles técnicos están en [`3_jwt-spec.md`](3_jwt-spec.md) (sesión) y [`4_modelo-datos.md`](4_modelo-datos.md) (almacenamiento). Los campos y mensajes, en [`2_forms-inputs.md`](2_forms-inputs.md).

## Índice de Contenido
1. [Criterios de Separación](#criterios-de-separación)
2. [Requerimientos Funcionales](#requerimientos-funcionales)
   - [RF-MS01-01 Registrar usuario institucional](#rf-ms01-01-registrar-usuario-institucional)
   - [RF-MS01-02 Autenticar usuario (login)](#rf-ms01-02-autenticar-usuario-login)
   - [RF-MS01-03 Gestión de sesión](#rf-ms01-03-gestión-de-sesión)
   - [RF-MS01-04 Cerrar sesión (logout)](#rf-ms01-04-cerrar-sesión-logout)
   - [RF-MS01-05 Gestión de roles y permisos](#rf-ms01-05-gestión-de-roles-y-permisos)
   - [RF-MS01-06 Gestión del perfil](#rf-ms01-06-gestión-del-perfil)
   - [RF-MS01-07 Administración de usuarios: consulta y activación/inactivación](#rf-ms01-07-administración-de-usuarios)
   - [RF-MS01-08 Recuperación y cambio de contraseña](#rf-ms01-08-recuperación-y-cambio-de-contraseña)
   - [RF-MS01-09 Resolución interna de usuarios](#rf-ms01-09-resolución-interna-de-usuarios)
   - [RF-MS01-10 Administración de roles y catálogo de permisos](#rf-ms01-10-administración-de-roles-y-catálogo-de-permisos)
   - [RF-MS01-11 Corte de sesión por cambios de acceso](#rf-ms01-11-corte-de-sesión-por-cambios-de-acceso)
3. [Requerimientos No Funcionales (RNF)](#requerimientos-no-funcionales-rnf)
4. [API: ms01-identity-and-access](#api-ms01-identity-and-access)

---

## Criterios de Separación

**¿Cuándo un requerimiento merece existir por separado?**
* Se reutiliza en varios flujos (auth, notificaciones, uploads).
* Tiene reglas de negocio propias y no triviales.
* Pertenece a otro microservicio/squad con su propio contrato de API o evento.
* Tiene su propio ciclo de pruebas independiente.
* Si un paso no cumple ninguna, va dentro del flujo base o como flujo alterno, no como requerimiento aparte.

**Flujo alterno:** solo un camino distinto que no corresponde a ningún paso del flujo base. Si un paso ya verifica una condición, su rechazo se describe en ese mismo paso. Toda verificación que falla no modifica datos.

---

## Requerimientos Funcionales

### RF-MS01-01 Registrar usuario institucional

| Atributo | Detalle |
|---|---|
| **ID** | RF-MS01-01 |
| **Nombre** | Registrar usuario institucional |
| **Épica / Grupo funcional** | MS01 - Registro |
| **Descripción** | Permitir que una persona se registre con su correo institucional, primer y segundo nombre, primer y segundo apellido (segundo nombre y segundo apellido opcionales) y contraseña. |
| **Actor(es)** | Usuario no registrado |
| **Precondiciones** | Dominio institucional configurado. Existe al menos un administrador inicial (ver [RN-MS01-12](#rf-ms01-05-gestión-de-roles-y-permisos)). |
| **Flujo base** | <ol><li>El cliente envía correo, primer nombre, segundo nombre (opcional), primer apellido, segundo apellido (opcional), contraseña y confirmación de contraseña ([F-01](2_forms-inputs.md#f-01-registro-de-usuario)).</li><li>El sistema valida que estén los campos obligatorios, que el correo tenga formato válido y pertenezca al dominio institucional, y que ambas contraseñas coincidan y cumplan la política ([RN-MS01-03](#rf-ms01-01-registrar-usuario-institucional)). Si algo falla, responde 400 indicando cada campo y su regla (para el correo, el mensaje es genérico: “Correo no válido.”) y emite el evento REGISTRO con resultado FALLO.</li><li>Verifica que el correo no esté registrado ([RN-MS01-01](#rf-ms01-01-registrar-usuario-institucional)). Si ya existe, responde 409 (“El correo ya está registrado.”) y emite el evento REGISTRO con resultado FALLO.</li><li>Guarda la contraseña según [RN-MS01-04](#rf-ms01-01-registrar-usuario-institucional) y crea el usuario en estado ACTIVO con el rol ESTUDIANTE.</li><li>Emite el evento REGISTRO con resultado EXITO.</li><li>Responde 201 con el id, correo, roles y estado (nunca la contraseña). No inicia sesión automáticamente.</li></ol> |
| **Flujos alternos** | — |
| **Reglas De Negocio** | <ul><li>**RN-MS01-01:** Un correo no puede asociarse a más de un usuario (sin distinguir mayúsculas/minúsculas).</li><li>**RN-MS01-02:** El rol inicial siempre es ESTUDIANTE. El solicitante no puede elegir ni enviar un rol.</li><li>**RN-MS01-03:** La contraseña debe tener entre 8 y 64 caracteres, sin espacios, con al menos una mayúscula, una minúscula, un número y un carácter especial ([F-01](2_forms-inputs.md#f-01-registro-de-usuario)).</li><li>**RN-MS01-04:** Las contraseñas se guardan de forma segura e irreversible, nunca en texto plano, y nunca se devuelven ([almacenamiento](4_modelo-datos.md#almacenamiento-seguro)).</li></ul> |
| **Postcondiciones** | El usuario existe, está ACTIVO y puede autenticarse con [RF-MS01-02](#rf-ms01-02-autenticar-usuario-login). |
| **Requerimientos asociados** | [RF-MS01-02](#rf-ms01-02-autenticar-usuario-login), [RF-MS01-05](#rf-ms01-05-gestión-de-roles-y-permisos) |
| **Criterios de aceptación** | <ul><li>Un registro válido devuelve 201 y el usuario queda ACTIVO con rol ESTUDIANTE.</li><li>Un correo repetido devuelve 409 y no crea un segundo usuario.</li><li>Un correo externo o una contraseña débil devuelven 400 y no crean usuario.</li><li>La respuesta no contiene la contraseña.</li><li>Cada intento, exitoso o fallido, genera un evento de auditoría.</li></ul> |
| **Prioridad** | Alta |
| **Estado / Squad responsable** | Implementado / kAiRoS |

<br>

### RF-MS01-02 Autenticar usuario (login)

| Atributo | Detalle |
|---|---|
| **ID** | RF-MS01-02 |
| **Nombre** | Autenticar usuario (login) |
| **Épica / Grupo funcional** | MS01 - Autenticación |
| **Descripción** | Permitir autenticarse con correo institucional y contraseña para acceder a los servicios protegidos. |
| **Actor(es)** | Todos los roles, ya registrados |
| **Precondiciones** | El usuario existe en MS01. |
| **Flujo base** | <ol><li>El cliente envía correo y contraseña ([F-02](2_forms-inputs.md#f-02-inicio-de-sesión)).</li><li>El sistema valida el formato de los datos. Si es inválido, responde 400 indicando el campo.</li><li>Busca el usuario por correo y verifica la contraseña. Si la cuenta no existe o la contraseña no coincide, responde 401 con un mensaje genérico (“Credenciales incorrectas.”) y emite el evento LOGIN con resultado FALLO.</li><li>Verifica que el usuario esté ACTIVO. Si no, responde 403 indicando que debe contactar a un administrador y emite el evento LOGIN con resultado FALLO.</li><li>Inicia la sesión del usuario ([RF-MS01-03](#rf-ms01-03-gestión-de-sesión)).</li><li>Emite el evento LOGIN con resultado EXITO.</li><li>Responde 200 con la credencial de acceso, su vencimiento y los datos básicos (id, correo, roles).</li></ol> |
| **Flujos alternos** | <ul><li>**A1 (paso 5), error al emitir la credencial:** el sistema responde 500 `ERROR_INTERNO` sin detalles internos y emite el evento LOGIN con resultado FALLO.</li></ul> |
| **Reglas De Negocio** | <ul><li>**RN-MS01-05:** Ante credenciales inválidas no se revela si falló el correo o la contraseña.</li><li>**RN-MS01-06:** Una cuenta INACTIVA nunca inicia sesión, aunque las credenciales sean correctas.</li></ul> |
| **Postcondiciones** | El usuario tiene una sesión vigente hasta su vencimiento ([RF-MS01-03](#rf-ms01-03-gestión-de-sesión)). |
| **Requerimientos asociados** | [RF-MS01-01](#rf-ms01-01-registrar-usuario-institucional), [RF-MS01-03](#rf-ms01-03-gestión-de-sesión), [RF-MS01-07](#rf-ms01-07-administración-de-usuarios) |
| **Criterios de aceptación** | <ul><li>Las credenciales correctas de una cuenta activa devuelven 200 y la credencial de acceso.</li><li>Un correo inexistente y una contraseña incorrecta devuelven exactamente la misma respuesta.</li><li>Una cuenta inactiva devuelve 403 aunque las credenciales sean correctas.</li><li>Todo intento que pasa la validación de formato (pasos 3 a 6) genera su evento de auditoría.</li></ul> |
| **Prioridad** | Alta |
| **Estado / Squad responsable** | Implementado / kAiRoS |

<br>

### RF-MS01-03 Gestión de sesión

| Atributo | Detalle |
|---|---|
| **ID** | RF-MS01-03 |
| **Nombre** | Gestión de sesión |
| **Épica / Grupo funcional** | MS01 - Sesiones |
| **Descripción** | Mantener la sesión del usuario autenticado durante un tiempo limitado y permitir que el ecosistema lo identifique, con sus roles y permisos, en cada petición. |
| **Actor(es)** | Usuarios autenticados. Sistema consumidor: API Gateway (único verificador de la credencial). |
| **Precondiciones** | Autenticación exitosa ([RF-MS01-02](#rf-ms01-02-autenticar-usuario-login)). |
| **Flujo base** | <ol><li>Al iniciar sesión, MS01 emite una credencial de acceso firmada con la identidad del usuario, sus roles vigentes, los permisos efectivos de esos roles y su vencimiento ([`3_jwt-spec.md`](3_jwt-spec.md)).</li><li>El cliente presenta la credencial en cada petición.</li><li>El API Gateway la verifica. Si es inválida, venció o es anterior a un corte de sesión del usuario o de alguno de sus roles ([RN-MS01-36](#rf-ms01-11-corte-de-sesión-por-cambios-de-acceso)), responde 401 sin llegar a MS01 y el cliente debe iniciar sesión de nuevo. Si es válida, entrega a MS01 la identidad del solicitante ([RN-MS01-23](#rf-ms01-03-gestión-de-sesión)).</li><li>MS01 identifica al usuario y autoriza la operación con los permisos que recibe del Gateway. Si en un endpoint protegido la identidad falta o no es válida, responde 401 `NO_AUTENTICADO`. Si los permisos no incluyen el requerido, responde 403 `PROHIBIDO`.</li></ol> |
| **Flujos alternos** | — |
| **Reglas De Negocio** | <ul><li>**RN-MS01-07:** La credencial de acceso la firma MS01 y el Gateway la verifica sin consultar a MS01 en cada petición; solo consulta de forma periódica los cortes de sesión ([RN-MS01-38](#rf-ms01-11-corte-de-sesión-por-cambios-de-acceso)) ([formato y firma](3_jwt-spec.md)).</li><li>**RN-MS01-08:** La credencial no contiene contraseñas ni datos sensibles.</li><li>**RN-MS01-09:** Un cambio de los roles de un usuario, de los permisos de un rol o del estado de una cuenta corta las credenciales ya emitidas a las que afecta ([RF-MS01-11](#rf-ms01-11-corte-de-sesión-por-cambios-de-acceso)); la siguiente credencial emitida ya refleja el cambio. Las credenciales que ningún corte afecta conservan sus datos hasta vencer, por eso su vigencia es corta.</li><li>**RN-MS01-23:** El registro, el login y la recuperación de contraseña son públicos. Las consultas internas ([RF-MS01-09](#rf-ms01-09-resolución-interna-de-usuarios)) reciben la identidad del servicio que llama. Los demás endpoints reciben la identidad del solicitante (id, roles y permisos) solo en las cabeceras que inyecta el Gateway, nunca en el cuerpo ([cabeceras](3_jwt-spec.md#identidad-en-ms01)).</li><li>**RN-MS01-27:** La credencial lleva el claim `permissions`: la unión de los permisos de todos los roles del usuario al emitirla, sin repetidos, en orden alfabético y con al menos uno. Si en la base un rol guarda un código que ya no existe en el catálogo de permisos, MS01 lo ignora al emitir y deja un aviso en el log ([`3_jwt-spec.md`](3_jwt-spec.md#token-emitido)).</li></ul> |
| **Postcondiciones** | Solo las peticiones con una credencial válida y vigente llegan a los recursos protegidos. |
| **Requerimientos asociados** | [RF-MS01-02](#rf-ms01-02-autenticar-usuario-login), [RF-MS01-04](#rf-ms01-04-cerrar-sesión-logout), [RF-MS01-05](#rf-ms01-05-gestión-de-roles-y-permisos), [RF-MS01-11](#rf-ms01-11-corte-de-sesión-por-cambios-de-acceso), RF-GW-02 (verificación en el Gateway) |
| **Criterios de aceptación** | <ul><li>El Gateway puede verificar la credencial emitida, que cumple [`3_jwt-spec.md`](3_jwt-spec.md).</li><li>Un endpoint protegido sin identidad responde 401.</li><li>La credencial muestra los roles y los permisos vigentes al momento del login.</li><li>Tras cambiar el rol de un usuario, su siguiente login emite una credencial con el nuevo rol y los permisos que le corresponden.</li><li>Una credencial emitida antes de un cambio de roles, de permisos de un rol o de estado deja de aceptarse en pocos segundos.</li></ul> |
| **Prioridad** | Alta |
| **Estado / Squad responsable** | Implementado / kAiRoS |

<br>

### RF-MS01-04 Cerrar sesión (logout)

| Atributo | Detalle |
|---|---|
| **ID** | RF-MS01-04 |
| **Nombre** | Cerrar sesión (logout) |
| **Épica / Grupo funcional** | MS01 - Sesiones |
| **Descripción** | Permitir que el usuario autenticado cierre su sesión y quede registrado. |
| **Actor(es)** | Todos los roles autenticados |
| **Precondiciones** | El usuario tiene una sesión vigente ([RF-MS01-03](#rf-ms01-03-gestión-de-sesión)). |
| **Flujo base** | <ol><li>El usuario solicita cerrar sesión.</li><li>El sistema identifica al usuario autenticado y verifica que tenga el permiso de cerrar sesión. Si no lo tiene, responde 403 y emite el evento LOGOUT con resultado FALLO.</li><li>Emite el evento LOGOUT con resultado EXITO.</li><li>Responde 200. El cliente descarta su credencial y regresa al login.</li></ol> |
| **Flujos alternos** | — |
| **Reglas De Negocio** | <ul><li>**RN-MS01-10:** El logout cierra la sesión en el cliente y queda auditado. No corta la sesión: la credencial sigue siendo válida hasta su vencimiento y la vigencia corta acota este límite ([RN-MS01-09](#rf-ms01-03-gestión-de-sesión)). Solo los cambios de acceso de [RF-MS01-11](#rf-ms01-11-corte-de-sesión-por-cambios-de-acceso) cortan sesiones.</li></ul> |
| **Postcondiciones** | El cierre de sesión queda registrado. |
| **Requerimientos asociados** | [RF-MS01-03](#rf-ms01-03-gestión-de-sesión) |
| **Criterios de aceptación** | <ul><li>Un logout con sesión vigente devuelve 200 y genera un evento de auditoría con el usuario y la fecha y hora.</li></ul> |
| **Prioridad** | Alta |
| **Estado / Squad responsable** | Implementado / kAiRoS |

<br>

### RF-MS01-05 Gestión de roles y permisos

| Atributo | Detalle |
|---|---|
| **ID** | RF-MS01-05 |
| **Nombre** | Gestión de roles y permisos |
| **Épica / Grupo funcional** | MS01 - Roles y permisos |
| **Descripción** | Permitir al administrador agregar y retirar roles a uno o varios usuarios, y a cualquier usuario consultar los permisos de sus roles. Los roles mismos y sus permisos se administran en [RF-MS01-10](#rf-ms01-10-administración-de-roles-y-catálogo-de-permisos). |
| **Actor(es)** | Administrador<br>Secundarios: los demás roles, que consultan solo sus propios roles y permisos. |
| **Precondiciones** | El administrador está autenticado. |
| **Flujo base** | <ol><li>El administrador busca usuarios ([RF-MS01-07](#rf-ms01-07-administración-de-usuarios)) y consulta sus roles actuales.</li><li>Selecciona uno o varios usuarios y solicita agregar o retirar uno o varios roles ([F-09](2_forms-inputs.md#f-09-asignación-de-roles)).</li><li>El sistema verifica, en este orden, y si algo falla no aplica ningún cambio:<ul><li>que el solicitante sea administrador; si no, responde 403 y emite el evento CAMBIO_ROL con resultado FALLO;</li><li>que la solicitud sea válida (usuarios, roles existentes, al menos una lista, ningún rol en ambas); si no, responde 400;</li><li>que los usuarios objetivo no incluyan al solicitante; si lo incluyen, responde 403 y emite el evento FALLO;</li><li>que todos los usuarios existan; si alguno no existe, responde 404 y emite un evento FALLO por cada uno;</li><li>que cada usuario conserve al menos un rol; si alguno quedaría sin roles, responde 400 y emite un evento FALLO por cada usuario que quedaría sin roles;</li><li>que ningún usuario quede con más de 50 roles ([RN-MS01-40](#rf-ms01-05-gestión-de-roles-y-permisos)); si alguno quedaría con más, responde 422 y emite un evento FALLO por cada usuario que superaría el tope.</li></ul></li><li>Actualiza los roles de los usuarios y registra el corte de sesión de cada uno ([RN-MS01-35](#rf-ms01-11-corte-de-sesión-por-cambios-de-acceso)).</li><li>Emite el evento CAMBIO_ROL con resultado EXITO por cada usuario (`recurso_afectado` = usuario objetivo, `detalle` con los roles agregados y retirados).</li><li>Responde 200 con los roles resultantes de cada usuario.</li></ol> |
| **Flujos alternos** | <ul><li>**A1, consulta de roles y permisos:** cualquier usuario autenticado consulta los roles con sus permisos. El administrador recibe el catálogo completo; los demás, solo sus propios roles.</li></ul> |
| **Reglas De Negocio** | <ul><li>**RN-MS01-11:** Solo el Administrador asigna o retira roles.</li><li>**RN-MS01-12:** Nadie puede asignarse privilegios por sí mismo, incluido el administrador. El primer administrador se crea mediante un mecanismo de inicialización configurado fuera del repositorio ([datos iniciales](4_modelo-datos.md#datos-iniciales)).</li><li>**RN-MS01-13:** Un usuario puede tener más de un rol, siempre con autorización del administrador.</li><li>**RN-MS01-40:** Un usuario no puede tener más de 50 roles. Las listas de roles a agregar y a retirar de una petición no tienen un límite propio de tamaño (solo admiten roles que existen y sin repetir): lo que se verifica es el total de roles que cada usuario tendría después del cambio. Si el resultado dejaría a algún usuario con más de 50, se rechaza toda la petición y no se aplica ningún cambio. El tope mantiene dentro del tamaño admitido por las cabeceras HTTP al claim `roles` de la credencial y a la cabecera `X-User-Role`.</li><li>**RN-MS01-14:** Los roles y los permisos de cada rol son datos de MS01 que el administrador edita en tiempo de ejecución ([RF-MS01-10](#rf-ms01-10-administración-de-roles-y-catálogo-de-permisos)). Los ocho roles iniciales (ESTUDIANTE, PROFESOR, ADMINISTRATIVO, BRIGADISTA, COORDINADOR, ORGANIZADOR, ADMIN, AUDITOR) y su asignación de permisos son la siembra que define la [Matriz Rol‐Permiso](https://github.com/CampusSeguro-DOSW-2026-2/documentation/wiki/Matriz-Rol%E2%80%90Permiso) ([datos iniciales](4_modelo-datos.md#datos-iniciales)). El catálogo de permisos (los códigos y su descripción) sigue versionado en el código y es de solo lectura.</li></ul> |
| **Postcondiciones** | Los nuevos roles se reflejan en la siguiente credencial; las anteriores del usuario dejan de aceptarse ([RN-MS01-09](#rf-ms01-03-gestión-de-sesión)). |
| **Requerimientos asociados** | [RF-MS01-03](#rf-ms01-03-gestión-de-sesión), [RF-MS01-07](#rf-ms01-07-administración-de-usuarios), [RF-MS01-10](#rf-ms01-10-administración-de-roles-y-catálogo-de-permisos), [RF-MS01-11](#rf-ms01-11-corte-de-sesión-por-cambios-de-acceso) |
| **Criterios de aceptación** | <ul><li>Un administrador agrega o retira roles a uno o varios usuarios en una sola operación y la consulta posterior lo muestra.</li><li>Tras el cambio, las credenciales anteriores del usuario dejan de aceptarse y su siguiente login refleja el nuevo rol.</li><li>Un administrador no puede modificar sus propios roles.</li><li>Las listas de roles a agregar y a retirar no tienen un límite propio: una petición con más de 8 roles se acepta mientras ningún usuario quede con más de 50.</li><li>Una petición que dejaría a algún usuario con más de 50 roles responde 422 y no cambia a ningún usuario.</li><li>Un no administrador recibe 403.</li><li>Cada cambio queda auditado.</li></ul> |
| **Prioridad** | Media |
| **Estado / Squad responsable** | Implementado / kAiRoS |

<br>

### RF-MS01-06 Gestión del perfil

| Atributo | Detalle |
|---|---|
| **ID** | RF-MS01-06 |
| **Nombre** | Gestión del perfil |
| **Épica / Grupo funcional** | MS01 - Perfil |
| **Descripción** | Permitir al usuario consultar y actualizar su información básica de perfil. |
| **Actor(es)** | Todos los roles autenticados |
| **Precondiciones** | Usuario autenticado. |
| **Flujo base** | <ol><li>El usuario consulta su perfil (primer nombre, segundo nombre, primer apellido, segundo apellido, correo, roles, estado). El sistema lo identifica como usuario autenticado ([RN-MS01-16](#rf-ms01-06-gestión-del-perfil)) y verifica que tenga el permiso de consultar su perfil; si no lo tiene, responde 403, y si la cuenta no existe, 404.</li><li>Envía los datos a modificar (primer nombre, segundo nombre, primer apellido, segundo apellido) ([F-03](2_forms-inputs.md#f-03-perfil-de-usuario)).</li><li>El sistema verifica, en este orden: que tenga el permiso de modificar su perfil (si no, 403); que los campos obligatorios no estén vacíos, que todos respeten longitud y formato, y que no se envíen correo, roles, estado ni otros campos no editables (si algo falla, 400 con el detalle de los campos), y que la cuenta exista (si no, 404). Si algo falla, no modifica nada. La actualización reemplaza el perfil completo: omitir el segundo nombre o el segundo apellido los deja vacíos.</li><li>Actualiza el perfil.</li><li>Responde 200 con el perfil actualizado.</li></ol> |
| **Flujos alternos** | — |
| **Reglas De Negocio** | <ul><li>**RN-MS01-15:** El usuario nunca puede modificar el correo institucional.</li><li>**RN-MS01-16:** Un usuario solo consulta y modifica su propio perfil. Los roles y el estado no se editan aquí.</li></ul> |
| **Postcondiciones** | Los datos modificados persisten y se devuelven en las consultas posteriores. |
| **Requerimientos asociados** | [RF-MS01-01](#rf-ms01-01-registrar-usuario-institucional), [RF-MS01-03](#rf-ms01-03-gestión-de-sesión) |
| **Criterios de aceptación** | <ul><li>Una modificación válida se refleja en la siguiente consulta.</li><li>Un intento de cambiar el correo se rechaza con 400 y no tiene efecto.</li><li>Un usuario no puede ver ni editar el perfil de otro mediante este flujo.</li></ul> |
| **Prioridad** | Alta |
| **Estado / Squad responsable** | Implementado / kAiRoS |

<br>

### RF-MS01-07 Administración de usuarios

| Atributo | Detalle |
|---|---|
| **ID** | RF-MS01-07 |
| **Nombre** | Administración de usuarios: consulta y activación/inactivación |
| **Épica / Grupo funcional** | MS01 - Estado de usuarios |
| **Descripción** | Permitir al administrador y al auditor listar, buscar y consultar usuarios, y al administrador cambiar su estado entre ACTIVO e INACTIVO. |
| **Actor(es)** | Administrador (consulta y cambio de estado)<br>Auditor (solo consulta) |
| **Precondiciones** | El administrador o el auditor está autenticado. |
| **Flujo base** | <ol><li>El administrador o el auditor lista o busca usuarios (por correo, con paginación y filtro opcional por estado o rol, [F-07](2_forms-inputs.md#f-07-búsqueda-y-filtros-de-usuarios)) o consulta uno por su id. Si no tiene permiso de consulta, el sistema responde 403. Si los parámetros son inválidos, responde 400. Si el usuario consultado no existe, responde 404.</li><li>El administrador selecciona uno o varios usuarios y solicita el nuevo estado ([F-08](2_forms-inputs.md#f-08-cambio-de-estado-de-usuario)).</li><li>El sistema verifica, en este orden, y si algo falla no aplica ningún cambio:<ul><li>que el solicitante sea administrador; si no, responde 403 y emite el evento CAMBIO_ESTADO con resultado FALLO;</li><li>que la solicitud sea válida; si no, responde 400;</li><li>que no esté cambiando su propia cuenta; si lo hace, responde 403 y emite el evento FALLO;</li><li>que todos los usuarios existan; si alguno no existe, responde 404 y emite un evento FALLO por cada uno.</li></ul></li><li>Actualiza el estado y registra el corte de sesión de cada usuario ([RN-MS01-35](#rf-ms01-11-corte-de-sesión-por-cambios-de-acceso)).</li><li>Emite el evento CAMBIO_ESTADO con resultado EXITO por cada usuario (`detalle` con el estado anterior y el nuevo).</li><li>Responde 200 con el estado anterior y el nuevo de cada usuario.</li></ol> |
| **Flujos alternos** | <ul><li>**A1 (paso 4), error al guardar:** el sistema responde 500 `ERROR_INTERNO` sin detalles internos, no aplica ningún cambio y emite el evento CAMBIO_ESTADO con resultado FALLO por cada usuario.</li></ul> |
| **Reglas De Negocio** | <ul><li>**RN-MS01-17:** Los únicos estados son ACTIVO e INACTIVO. Toda cuenta nueva nace ACTIVA.</li><li>**RN-MS01-18:** Una cuenta INACTIVA no puede iniciar sesión ([RN-MS01-06](#rf-ms01-02-autenticar-usuario-login)). Al cambiar el estado se corta la sesión de la cuenta ([RN-MS01-35](#rf-ms01-11-corte-de-sesión-por-cambios-de-acceso)): las credenciales emitidas antes dejan de aceptarse ([RN-MS01-09](#rf-ms01-03-gestión-de-sesión)).</li><li>**RN-MS01-19:** Solo el administrador cambia estados, y no puede cambiar el propio. El auditor solo consulta.</li></ul> |
| **Postcondiciones** | El estado queda actualizado, condiciona el siguiente login y corta las credenciales ya emitidas de la cuenta. |
| **Requerimientos asociados** | [RF-MS01-02](#rf-ms01-02-autenticar-usuario-login), [RF-MS01-05](#rf-ms01-05-gestión-de-roles-y-permisos) |
| **Criterios de aceptación** | <ul><li>Tras inactivar un usuario, su login falla con 403 aunque las credenciales sean correctas.</li><li>Tras inactivar un usuario, sus credenciales ya emitidas dejan de aceptarse en pocos segundos.</li><li>Tras reactivarlo, su login funciona.</li><li>Un administrador no puede inactivarse a sí mismo.</li><li>Cada cambio de estado queda auditado.</li></ul> |
| **Prioridad** | Alta |
| **Estado / Squad responsable** | Implementado / kAiRoS |

<br>

### RF-MS01-08 Recuperación y cambio de contraseña

| Atributo | Detalle |
|---|---|
| **ID** | RF-MS01-08 |
| **Nombre** | Recuperación y cambio de contraseña |
| **Épica / Grupo funcional** | MS01 - Contraseña |
| **Descripción** | Permitir restablecer la contraseña olvidada mediante un proceso controlado y cambiarla estando autenticado. |
| **Actor(es)** | Todos los roles (restablecer: usuario no autenticado; cambiar: usuario autenticado). |
| **Precondiciones** | El usuario existe. El servicio de envío de correo está configurado. |
| **Flujo base** | <ul><li><b>Flujo base A, restablecer contraseña olvidada:</b><ol><li>El usuario ingresa su correo institucional ([F-04](2_forms-inputs.md#f-04-solicitud-de-recuperación-de-contraseña)). Si el formato es inválido, el sistema responde 400.</li><li>El sistema responde siempre con el mismo mensaje (“Si el correo está registrado, enviamos un código de recuperación.”).</li><li>Solo si el correo pertenece a una cuenta ACTIVA y no se pidió otro código durante la espera mínima ([RN-MS01-21](#rf-ms01-08-recuperación-y-cambio-de-contraseña)): genera un código de 6 dígitos, invalida los códigos anteriores, lo guarda según [RN-MS01-04](#rf-ms01-01-registrar-usuario-institucional) y lo envía al correo. En cualquier otro caso no envía nada.</li><li>El usuario envía su correo, el código y la nueva contraseña con su confirmación ([F-05](2_forms-inputs.md#f-05-restablecer-contraseña-con-código)).</li><li>El sistema verifica, y si algo falla responde 400 y emite el evento RECUPERACION con resultado FALLO:<ul><li>que los datos sean válidos (formato, política de contraseña, confirmación);</li><li>que la cuenta esté ACTIVA y el código sea correcto, vigente y no usado (mensaje genérico: “Código incorrecto o vencido.”); cada código incorrecto cuenta como un intento fallido;</li><li>que la nueva contraseña sea distinta de la actual.</li></ul></li><li>Actualiza la contraseña, invalida el código y emite el evento RECUPERACION con resultado EXITO.</li><li>Responde 200. El usuario debe iniciar sesión de nuevo.</li></ol></li><li><b>Flujo base B, cambiar contraseña autenticado:</b><ol><li>El usuario envía su contraseña actual, la nueva y la confirmación ([F-06](2_forms-inputs.md#f-06-cambio-de-contraseña-autenticado)).</li><li>El sistema verifica, en este orden: que tenga el permiso de cambiar su contraseña (si no, 403); que los datos sean válidos (política y confirmación); que la contraseña actual sea correcta, y que la nueva sea distinta de la actual (si falla alguna de estas tres, 400). En cada caso emite el evento CAMBIO_CONTRASENA con resultado FALLO. Si la cuenta no existe, responde 404.</li><li>Actualiza la contraseña y emite el evento CAMBIO_CONTRASENA con resultado EXITO.</li><li>Responde 200.</li></ol></li></ul> |
| **Flujos alternos** | — |
| **Reglas De Negocio** | <ul><li>**RN-MS01-20:** La respuesta de la solicitud de código no revela si el correo existe.</li><li>**RN-MS01-21:** El código es de un solo uso, tiene vigencia corta, se invalida al alcanzar el máximo de intentos fallidos y entre dos solicitudes del mismo usuario hay una espera mínima. Los valores son configurables ([configuración](4_modelo-datos.md#configuración)).</li><li>**RN-MS01-22:** La nueva contraseña cumple [RN-MS01-03](#rf-ms01-01-registrar-usuario-institucional), es distinta de la actual y se guarda según [RN-MS01-04](#rf-ms01-01-registrar-usuario-institucional).</li></ul> |
| **Postcondiciones** | La contraseña anterior deja de funcionar. La nueva permite iniciar sesión. |
| **Requerimientos asociados** | [RF-MS01-01](#rf-ms01-01-registrar-usuario-institucional), [RF-MS01-02](#rf-ms01-02-autenticar-usuario-login) |
| **Criterios de aceptación** | <ul><li>Con la contraseña anterior el login falla. Con la nueva funciona.</li><li>Un código usado, vencido o que agotó sus intentos es rechazado.</li><li>La solicitud de código tiene la misma respuesta para un correo existente y uno inexistente.</li><li>El restablecimiento y el cambio de contraseña, exitosos o fallidos, quedan auditados. La solicitud de código no genera evento.</li></ul> |
| **Prioridad** | Alta |
| **Estado / Squad responsable** | Implementado / kAiRoS |

<br>

### RF-MS01-09 Resolución interna de usuarios

| Atributo | Detalle |
|---|---|
| **ID** | RF-MS01-09 |
| **Nombre** | Resolución interna de usuarios |
| **Épica / Grupo funcional** | MS01 - Integración |
| **Descripción** | Permitir que otros microservicios autorizados consulten los datos de varios usuarios por su id o de todos los usuarios con un rol, sin usar una cuenta de usuario. |
| **Actor(es)** | Microservicios autorizados, a través del API Gateway: MS03 (destinatarios de alertas) y MS05 (brigadistas). |
| **Precondiciones** | El Gateway identificó al servicio que llama ([identidad del servicio](3_jwt-spec.md#identidad-en-ms01)). |
| **Flujo base** | <ul><li><b>Flujo base A, consulta por ids:</b><ol><li>El servicio envía de 1 a 100 ids de usuario.</li><li>El sistema verifica que el servicio esté identificado y autorizado ([RN-MS01-24](#rf-ms01-09-resolución-interna-de-usuarios)). Si no está identificado, responde 401; si no está autorizado, responde 403.</li><li>Valida la lista: entre 1 y 100 ids, cada uno con formato válido y sin campos adicionales. Si algo falla, responde 400 indicando el campo.</li><li>Responde 200 con los usuarios encontrados, sin repetidos y en el orden pedido, y con la lista de ids que no existen.</li></ol></li><li><b>Flujo base B, consulta por rol:</b><ol><li>El servicio envía el rol, la página y, opcionalmente, una fecha de corte.</li><li>El sistema verifica el servicio como en el paso A2.</li><li>Valida que el rol exista y que la fecha de corte, la página y el tamaño sean válidos ([F-07](2_forms-inputs.md#f-07-búsqueda-y-filtros-de-usuarios) para la paginación). Si algo falla, responde 400 indicando el campo.</li><li>Responde 200 con la página de usuarios que tienen el rol y, si llegó la fecha de corte, que se registraron hasta ese momento, en un orden fijo.</li></ol></li></ul> |
| **Flujos alternos** | — |
| **Reglas De Negocio** | <ul><li>**RN-MS01-24:** Solo los servicios configurados como autorizados usan estas consultas. No se autorizan con la Matriz Rol‐Permiso ni con los roles de un usuario. La consulta de cortes de sesión del Gateway ([RF-MS01-11](#rf-ms01-11-corte-de-sesión-por-cambios-de-acceso)) tiene su propia regla ([RN-MS01-37](#rf-ms01-11-corte-de-sesión-por-cambios-de-acceso)).</li><li>**RN-MS01-25:** Las respuestas incluyen usuarios activos e inactivos, con su estado. El servicio que consulta decide a cuáles usar.</li><li>**RN-MS01-26:** Estas consultas no modifican datos ni generan eventos de auditoría.</li></ul> |
| **Postcondiciones** | Ninguna: solo lectura. |
| **Requerimientos asociados** | [RF-MS01-03](#rf-ms01-03-gestión-de-sesión), [RF-MS01-07](#rf-ms01-07-administración-de-usuarios) |
| **Criterios de aceptación** | <ul><li>Una consulta con ids existentes e inexistentes devuelve 200 con los usuarios encontrados y los ids que no existen.</li><li>Más de 100 ids, o un id con formato inválido, devuelve 400.</li><li>Un servicio no autorizado recibe 403, y una llamada sin servicio identificado, 401.</li><li>Recorriendo todas las páginas con la misma fecha de corte se obtienen todos los usuarios con el rol, sin repetidos ni faltantes.</li></ul> |
| **Prioridad** | Alta |
| **Estado / Squad responsable** | Implementado / kAiRoS |

<br>

### RF-MS01-10 Administración de roles y catálogo de permisos

| Atributo | Detalle |
|---|---|
| **ID** | RF-MS01-10 |
| **Nombre** | Administración de roles y catálogo de permisos |
| **Épica / Grupo funcional** | MS01 - Roles y permisos |
| **Descripción** | Permitir al administrador consultar, crear, renombrar, describir y borrar roles, cambiar los permisos de cada rol y consultar el catálogo de permisos que puede asignar. |
| **Actor(es)** | Administrador |
| **Precondiciones** | El administrador está autenticado y su credencial incluye `ms01:role:manage` (escritura) o `ms01:role:read_all` (consulta). Existen los roles de sistema ESTUDIANTE y ADMIN ([datos iniciales](4_modelo-datos.md#datos-iniciales)). |
| **Flujo base** | <ul><li><b>Flujo base A, consultar roles y catálogo:</b><ol><li>El administrador solicita la lista de roles con detalle, un rol por su nombre o el catálogo de permisos.</li><li>El sistema verifica que tenga el permiso de consulta; si no, responde 403.</li><li>En la consulta de un rol normaliza el nombre ([RG-08](2_forms-inputs.md#reglas-generales-de-entrada)), valida su formato (si es inválido, 400) y que el rol exista (si no, 404).</li><li>Responde 200 con cada rol (nombre, descripción, permisos y si es de sistema) o con el catálogo (código y descripción de cada permiso).</li></ol></li><li><b>Flujo base B, crear un rol:</b><ol><li>El administrador envía el nombre, la descripción y los permisos del rol ([F-10](2_forms-inputs.md#f-10-crear-rol)).</li><li>El sistema normaliza el nombre ([RG-08](2_forms-inputs.md#reglas-generales-de-entrada)) y verifica, en este orden, y si algo falla no aplica ningún cambio: que tenga el permiso de gestión (si no, 403 y evento CREACION_ROL con resultado FALLO); que los datos sean válidos, es decir, nombre con formato y distinto de `DETAILS` ([RN-MS01-28](#rf-ms01-10-administración-de-roles-y-catálogo-de-permisos)), descripción con longitud, al menos un permiso y todos del catálogo (si no, 400 y evento FALLO); que el rol no incluya `ms01:role:manage` ([RN-MS01-30](#rf-ms01-10-administración-de-roles-y-catálogo-de-permisos)); si lo incluye, 422 y evento FALLO; que el nombre no exista (si existe, 409 y evento FALLO).</li><li>Crea el rol como rol que no es de sistema y sin corte de sesión.</li><li>Emite el evento CREACION_ROL con resultado EXITO (`recurso_afectado` = nombre del rol, `detalle` con la descripción y los permisos).</li><li>Responde 201 con el rol creado.</li></ol></li><li><b>Flujo base C, renombrar o describir un rol:</b><ol><li>El administrador envía el nombre (igual o nuevo) y la descripción ([F-11](2_forms-inputs.md#f-11-editar-rol)).</li><li>El sistema normaliza el nombre de la ruta y el nombre nuevo ([RG-08](2_forms-inputs.md#reglas-generales-de-entrada)) y verifica, en este orden, y si algo falla no aplica ningún cambio: el permiso de gestión (403); que el rol exista (404); que los datos sean válidos, incluido que el nombre nuevo no sea `DETAILS` (400); que un rol de sistema no cambie de nombre (409); que el nombre nuevo no pertenezca a otro rol (409). Cada rechazo emite el evento EDICION_ROL con resultado FALLO.</li><li>Actualiza el nombre y la descripción. Si el nombre cambió, los usuarios que tenían el rol pasan a tenerlo con el nombre nuevo y se registra el corte de sesión de cada uno, incluido el propio administrador si tiene ese rol ([RN-MS01-35](#rf-ms01-11-corte-de-sesión-por-cambios-de-acceso)).</li><li>Emite el evento EDICION_ROL con resultado EXITO (`detalle` con el nombre y la descripción anteriores y los nuevos).</li><li>Responde 200 con el rol actualizado.</li></ol></li><li><b>Flujo base D, cambiar los permisos de un rol:</b><ol><li>El administrador envía la lista completa de permisos del rol ([F-12](2_forms-inputs.md#f-12-permisos-de-un-rol)).</li><li>El sistema normaliza el nombre de la ruta ([RG-08](2_forms-inputs.md#reglas-generales-de-entrada)) y verifica, en este orden, y si algo falla no aplica ningún cambio: el permiso de gestión (403); que el rol exista (404); que la lista sea válida, es decir, con al menos un permiso y todos del catálogo (400); que se cumpla [RN-MS01-30](#rf-ms01-10-administración-de-roles-y-catálogo-de-permisos), esto es, que `ms01:role:manage` solo esté en ADMIN y que ADMIN lo conserve (422). Cada rechazo emite el evento CAMBIO_PERMISOS_ROL con resultado FALLO.</li><li>Reemplaza la lista de permisos del rol y registra el corte de sesión del rol, que cubre a todos sus usuarios, incluido el propio administrador si tiene ese rol ([RN-MS01-35](#rf-ms01-11-corte-de-sesión-por-cambios-de-acceso)).</li><li>Emite el evento CAMBIO_PERMISOS_ROL con resultado EXITO (`detalle` con los permisos agregados y retirados).</li><li>Responde 200 con el rol actualizado.</li></ol></li><li><b>Flujo base E, borrar un rol:</b><ol><li>El administrador solicita borrar el rol ([F-13](2_forms-inputs.md#f-13-borrar-rol)).</li><li>El sistema normaliza el nombre de la ruta ([RG-08](2_forms-inputs.md#reglas-generales-de-entrada)) y verifica, en este orden: el permiso de gestión (403); que el rol exista (404); que no sea de sistema (409). Cada rechazo emite el evento ELIMINACION_ROL con resultado FALLO.</li><li>En una sola transacción: a cada usuario que tiene solo ese rol le asigna ESTUDIANTE; a cada usuario que tiene varios roles le retira ese rol; registra el corte de sesión de cada usuario afectado, incluido el propio administrador si tiene ese rol ([RN-MS01-35](#rf-ms01-11-corte-de-sesión-por-cambios-de-acceso)); y borra el rol.</li><li>Emite el evento ELIMINACION_ROL con resultado EXITO (`recurso_afectado` = nombre del rol, `detalle` con la cantidad de usuarios que pasaron a ESTUDIANTE y de los que solo perdieron el rol).</li><li>Responde 204 sin cuerpo.</li></ol></li></ul> |
| **Flujos alternos** | <ul><li>**A1 (flujo base E, paso 3), error al guardar:** el sistema responde 500 `ERROR_INTERNO` sin detalles internos, no aplica ningún cambio (ni a los usuarios ni al rol) y emite el evento ELIMINACION_ROL con resultado FALLO. Lo mismo aplica a un error al guardar en los flujos base B, C y D, con su evento correspondiente.</li></ul> |
| **Reglas De Negocio** | <ul><li>**RN-MS01-28:** El nombre de un rol tiene entre 2 y 30 caracteres: letras sin tilde, dígitos y guion bajo, y empieza con una letra. Antes de validarlo se recortan los espacios y se pasa a mayúsculas, y así se guarda y se devuelve; es único y es la clave del rol. Los códigos de permiso no se normalizan ([RG-08](2_forms-inputs.md#reglas-generales-de-entrada)). La descripción es obligatoria y tiene entre 3 y 200 caracteres. Los permisos son al menos uno, sin repetidos, y cada uno debe existir en el catálogo; la base de datos no valida los valores de los permisos, solo la API. El nombre `DETAILS` está reservado: es el segmento fijo de la ruta `GET /roles/details` y ningún rol puede llamarse así, en cualquier combinación de mayúsculas y minúsculas. La API lo rechaza con 400 al crear o renombrar y la base de datos lo impide con una restricción de la tabla `roles` ([modelo de datos](4_modelo-datos.md#tablas)).</li><li>**RN-MS01-29:** ESTUDIANTE y ADMIN son los roles de sistema: no se pueden borrar ni renombrar, y su condición de sistema no cambia por la API. El indicador `es_sistema` nunca se recibe en un cuerpo; solo se devuelve en las lecturas. Los demás roles, incluidos los seis restantes de la siembra, se pueden editar, renombrar y borrar. ESTUDIANTE es además el rol que reciben los usuarios que quedan sin ninguno al borrar un rol.</li><li>**RN-MS01-30:** `ms01:role:manage` pertenece solo a ADMIN y ADMIN siempre lo tiene; ningún otro rol puede incluirlo. Salvo ese permiso, el administrador puede asignar a cualquier rol cualquier permiso del catálogo, incluso uno que ADMIN no tenga. No hay otra restricción de escalamiento.</li><li>**RN-MS01-31:** Borrar un rol reasigna o retira el rol a sus usuarios y lo borra en una sola transacción: si algo falla, no queda ningún cambio. Un rol con usuarios vinculados no se puede borrar en la base de datos; por eso el sistema reasigna antes de borrar.</li><li>**RN-MS01-32:** Renombrar un rol actualiza su nombre en todos los usuarios que lo tienen y corta sus sesiones, porque la credencial anterior lleva el nombre viejo.</li><li>**RN-MS01-33:** Cambiar los permisos de un rol reemplaza la lista completa y corta las sesiones de todos los usuarios que lo tienen.</li><li>**RN-MS01-34:** Los ocho roles iniciales y su asignación de permisos los carga el script de datos iniciales con la [Matriz Rol‐Permiso](https://github.com/CampusSeguro-DOSW-2026-2/documentation/wiki/Matriz-Rol%E2%80%90Permiso). A partir de ahí la fuente es la base de datos. Un código guardado que ya no existe en el catálogo de permisos se ignora al emitir la credencial y al responder las consultas, con un aviso en el log.</li></ul> |
| **Postcondiciones** | Los cambios quedan guardados y auditados. Las sesiones afectadas se cortan ([RF-MS01-11](#rf-ms01-11-corte-de-sesión-por-cambios-de-acceso)). Un rol borrado ya no puede asignarse. |
| **Requerimientos asociados** | [RF-MS01-05](#rf-ms01-05-gestión-de-roles-y-permisos), [RF-MS01-03](#rf-ms01-03-gestión-de-sesión), [RF-MS01-11](#rf-ms01-11-corte-de-sesión-por-cambios-de-acceso) |
| **Criterios de aceptación** | <ul><li>Un administrador crea un rol con permisos del catálogo y la consulta posterior lo muestra, con `es_sistema` falso.</li><li>Crear un rol con un nombre repetido responde 409; con un permiso fuera del catálogo o sin permisos, 400; con `ms01:role:manage`, 422.</li><li>Crear un rol llamado `DETAILS`, o renombrar uno a `DETAILS`, responde 400 y no cambia nada; la base de datos también lo rechaza.</li><li>Cambiar los permisos de ADMIN de modo que pierda `ms01:role:manage` responde 422.</li><li>Renombrar ESTUDIANTE o ADMIN, o borrarlos, responde 409 y no cambia nada.</li><li>Renombrar un rol normal actualiza a sus usuarios y corta sus sesiones.</li><li>Si el administrador cambia los permisos de un rol, lo renombra o lo borra y él también lo tiene, su propia sesión se corta y debe iniciar sesión de nuevo.</li><li>Borrar un rol pasa a ESTUDIANTE a quienes solo tenían ese rol y retira el rol a quienes tenían varios, en una sola operación.</li><li>Un no administrador recibe 403 en cualquier operación de gestión.</li><li>El catálogo de permisos es solo de lectura: no existe ninguna ruta para crear, editar o borrar permisos.</li><li>Cada cambio, exitoso o fallido, queda auditado.</li></ul> |
| **Prioridad** | Alta |
| **Estado / Squad responsable** | Por implementar / kAiRoS |

<br>

### RF-MS01-11 Corte de sesión por cambios de acceso

| Atributo | Detalle |
|---|---|
| **ID** | RF-MS01-11 |
| **Nombre** | Corte de sesión por cambios de acceso |
| **Épica / Grupo funcional** | MS01 - Sesiones |
| **Descripción** | Hacer que un cambio de roles, de permisos de un rol o de estado de una cuenta surta efecto en pocos segundos, sin esperar a que venza la credencial. MS01 registra un corte de sesión y el API Gateway lo consulta y lo aplica. |
| **Actor(es)** | Sistema. Consumidor: API Gateway (única identidad autorizada para la consulta). |
| **Precondiciones** | Se produjo uno de los eventos de [RN-MS01-35](#rf-ms01-11-corte-de-sesión-por-cambios-de-acceso) o el Gateway está en línea. |
| **Flujo base** | <ul><li><b>Flujo base A, registrar el corte (en MS01):</b><ol><li>Al ejecutarse uno de los eventos de [RN-MS01-35](#rf-ms01-11-corte-de-sesión-por-cambios-de-acceso), MS01 escribe en la misma transacción del cambio el instante actual de la base de datos, sin fracción de segundo, como corte de cada usuario afectado o del rol cuyos permisos cambiaron.</li></ol></li><li><b>Flujo base B, publicar los cortes:</b><ol><li>Cada 4 segundos, el Gateway consulta la ruta interna de MS01 con su identidad de servicio.</li><li>El sistema verifica la identidad: si falta, responde 401; si no es el Gateway, responde 403 ([RN-MS01-37](#rf-ms01-11-corte-de-sesión-por-cambios-de-acceso)).</li><li>Responde 200 con la instantánea completa de los cortes de los últimos 3700 segundos: los de usuarios y los de roles.</li><li>El Gateway reemplaza por completo sus cortes en memoria con la instantánea recibida.</li></ol></li><li><b>Flujo base C, aplicar el corte (en el Gateway):</b><ol><li>En cada petición con una credencial válida, el Gateway compara la hora de emisión de la credencial con el corte del usuario y con el corte de cada uno de sus roles.</li><li>Si la hora de emisión es menor que alguno, responde 401 `NO_AUTENTICADO` con el mensaje genérico y el cliente debe iniciar sesión de nuevo. Si no, deja pasar la petición.</li></ol></li></ul> |
| **Flujos alternos** | <ul><li>**A1 (flujo base B), MS01 no responde o la respuesta es inválida:** el Gateway conserva la última instantánea, deja un aviso en el log y no rechaza ninguna petición por ello.</li><li>**A2, arranque del Gateway:** hace una consulta inicial antes de depender de la instantánea.</li></ul> |
| **Reglas De Negocio** | <ul><li>**RN-MS01-35:** MS01 registra un corte cuando: se agregan o retiran roles a un usuario (corte del usuario); cambia el estado de una cuenta (corte del usuario); se renombra un rol (corte de cada usuario que lo tiene); se borra un rol (corte de cada usuario afectado); cambian los permisos de un rol (corte del rol, que cubre a todos sus usuarios). No cortan sesiones: crear un rol, cambiar solo su descripción, cerrar sesión ([RN-MS01-10](#rf-ms01-04-cerrar-sesión-logout)) y cambiar o restablecer la contraseña. El instante es la hora de la base de datos, truncada a segundos, tanto para el corte como para la hora de emisión de la credencial: todas las instancias de MS01 usan el mismo reloj, y la comparación del Gateway no depende de que sus relojes estén sincronizados. El corte no distingue quién hizo el cambio: si el administrador que renombra, borra o cambia los permisos de un rol también tiene ese rol, su propia sesión se corta y debe iniciar sesión de nuevo.</li><li>**RN-MS01-36:** Una credencial se rechaza solo si su hora de emisión es estrictamente menor que el corte del usuario o que el de alguno de sus roles; si es igual o mayor, se acepta. Los cortes se identifican por el nombre del rol tal como viaja en la credencial.</li><li>**RN-MS01-37:** La ruta interna de cortes solo acepta al servicio `gateway`. No se publica a través del Gateway, no se audita y no usa la lista de servicios autorizados de [RF-MS01-09](#rf-ms01-09-resolución-interna-de-usuarios) (MS03 y MS05), que no cambia. El servicio llega en la cabecera `X-Service-Name` y MS01 no la comprueba con ninguna credencial: la ruta es segura solo mientras la red impida llegar a MS01 sin pasar por el Gateway ([aislamiento de red](3_jwt-spec.md#identidad-en-ms01)).</li><li>**RN-MS01-38:** El Gateway consulta cada 4 segundos y guarda los cortes solo en memoria (no usa base de datos). Con MS01 respondiendo, un corte se aplica como máximo 7 segundos después del cambio: 4 del intervalo más hasta 3 de espera de la consulta. Ante un fallo de MS01 conserva lo último que recibió y sigue aceptando credenciales válidas. La instantánea cubre 3700 segundos (3600 de vigencia, 30 de tolerancia de reloj y un margen); un corte más antiguo ya no afecta a ninguna credencial vigente y deja de publicarse.</li><li>**RN-MS01-39:** Al cliente se le informa con el mismo 401 genérico que cualquier credencial inválida, sin indicar que fue un corte.</li></ul> |
| **Postcondiciones** | Las credenciales emitidas antes del cambio y afectadas por él dejan de aceptarse; las emitidas después se aceptan. |
| **Requerimientos asociados** | [RF-MS01-03](#rf-ms01-03-gestión-de-sesión), [RF-MS01-05](#rf-ms01-05-gestión-de-roles-y-permisos), [RF-MS01-07](#rf-ms01-07-administración-de-usuarios), [RF-MS01-10](#rf-ms01-10-administración-de-roles-y-catálogo-de-permisos), RF-GW-02 (verificación en el Gateway) |
| **Criterios de aceptación** | <ul><li>Tras cambiar los permisos de un rol, la credencial anterior de un usuario con ese rol se rechaza con 401 como máximo 7 segundos después del cambio (4 de intervalo y hasta 3 de espera de la consulta), y su nuevo login trae los permisos nuevos.</li><li>Tras cambiar los roles de un usuario, inactivarlo, o renombrar o borrar uno de sus roles, sus credenciales anteriores se rechazan con 401.</li><li>Crear un rol, cambiar solo su descripción, cerrar sesión o cambiar la contraseña no rechaza ninguna credencial.</li><li>Una credencial emitida después del corte (misma hora o posterior) se acepta.</li><li>La ruta de cortes responde 403 a cualquier servicio distinto del Gateway y 401 si no hay servicio identificado.</li><li>Si MS01 no responde, el Gateway sigue aceptando las credenciales válidas.</li></ul> |
| **Prioridad** | Alta |
| **Estado / Squad responsable** | Por implementar / kAiRoS |

---

## Requerimientos No Funcionales (RNF)

| ID | Requerimiento |
|---|---|
| **RNF-MS01-01** | Aplicación Spring Boot independiente (Java, Maven) con Spring Security, Spring Data JPA y base de datos PostgreSQL propia. Ningún otro servicio accede a ella. |
| **RNF-MS01-02** | Todos los endpoints bajo `/api/v1/`, documentados con OpenAPI/Swagger y publicados antes de que otros squads dependan de ellos. |
| **RNF-MS01-03** | Todo acceso a MS01, externo o de otro MS, llega a través del API Gateway, y las llamadas salientes de MS01 (auditoría a MS08) también pasan por él, según los Estándares. MS01 no configura CORS. |
| **RNF-MS01-04** | Todo error, incluidos los del framework, responde con el `ErrorResponse` y el catálogo de códigos de los Estándares, sin exponer trazas internas. |
| **RNF-MS01-05** | Logs con el Correlation ID y sin datos sensibles, según los Estándares. Spring Boot Actuator expone solo `/actuator/health`. |
| **RNF-MS01-06** | Los secretos (llave de firma, credenciales de BD y de correo, clave de servicio) quedan fuera del repositorio, en variables de entorno. |
| **RNF-MS01-07** | Pruebas unitarias y de integración de los flujos de registro, login, sesión, roles, estado, contraseña, lectura de la identidad y emisión de eventos de auditoría. Análisis de calidad con SonarCloud. |
| **RNF-MS01-08** | Ejecución en contenedor Docker, con CI en GitHub Actions y despliegue continuo en dos entornos: prueba (automático) y producción (manual, con la misma imagen). El código se integra por Pull Request con revisión. |
| **RNF-MS01-09** | Las entradas se validan en el servidor ([`2_forms-inputs.md`](2_forms-inputs.md)). Las contraseñas se guardan según [RN-MS01-04](#rf-ms01-01-registrar-usuario-institucional). |
| **RNF-MS01-10** | La auditoría se delega en [ms08-analytics-and-audit](https://github.com/CampusSeguro-DOSW-2026-2/ms08-analytics-and-audit) (MS08), que almacena y consulta los eventos. MS01 los emite de forma asíncrona con el formato de los Estándares (`modulo` = MS01). Acciones de MS01: REGISTRO, LOGIN, LOGOUT, CAMBIO_ROL, CREACION_ROL, EDICION_ROL, CAMBIO_PERMISOS_ROL, ELIMINACION_ROL, CAMBIO_ESTADO, CAMBIO_CONTRASENA, RECUPERACION; resultado EXITO o FALLO. Si MS08 falla, la respuesta al cliente no cambia y el error queda en el log. Las consultas internas ([RF-MS01-09](#rf-ms01-09-resolución-interna-de-usuarios)) y la consulta de cortes de sesión del Gateway ([RF-MS01-11](#rf-ms01-11-corte-de-sesión-por-cambios-de-acceso)) no se auditan. MS01 no tiene tablas ni endpoints de auditoría. |

---

## API: ms01-identity-and-access

Contrato completo: [`api/swagger.yml`](api/swagger.yml). Los endpoints **protegidos** reciben la identidad según [RN-MS01-23](#rf-ms01-03-gestión-de-sesión); los **internos**, la del servicio que llama ([RN-MS01-24](#rf-ms01-09-resolución-interna-de-usuarios)). El *Permiso* coincide con el `x-permiso` del contrato. La asignación inicial de permisos a roles está en la [Matriz Rol‐Permiso](https://github.com/CampusSeguro-DOSW-2026-2/documentation/wiki/Matriz-Rol%E2%80%90Permiso); la vigente se consulta con `GET /roles/details`.

| Categoría | Método | Ruta (/api/v1) | Acceso | Permiso | RF |
|---|---|---|---|---|---|
| **Autenticación y sesión** | POST | `/auth/register` | Público | — | [01](#rf-ms01-01-registrar-usuario-institucional) |
| | POST | `/auth/login` | Público | — | [02](#rf-ms01-02-autenticar-usuario-login) |
| | POST | `/auth/logout` | Protegido | `ms01:session:logout` | [04](#rf-ms01-04-cerrar-sesión-logout) |
| **Recuperación y cambio de clave** | POST | `/auth/password/forgot` | Público | — | [08](#rf-ms01-08-recuperación-y-cambio-de-contraseña) |
| | POST | `/auth/password/reset` | Público | — | [08](#rf-ms01-08-recuperación-y-cambio-de-contraseña) |
| | PUT | `/users/me/password` | Protegido | `ms01:password:change` | [08](#rf-ms01-08-recuperación-y-cambio-de-contraseña) |
| **Perfil propio** | GET | `/users/me` | Protegido | `ms01:profile:read` | [06](#rf-ms01-06-gestión-del-perfil) |
| | PUT | `/users/me` | Protegido | `ms01:profile:write` | [06](#rf-ms01-06-gestión-del-perfil) |
| **Administración de usuarios** | GET | `/users` | Protegido | `ms01:user:read` | [07](#rf-ms01-07-administración-de-usuarios) |
| | GET | `/users/{id}` | Protegido | `ms01:user:read` | [05](#rf-ms01-05-gestión-de-roles-y-permisos), [07](#rf-ms01-07-administración-de-usuarios) |
| | PATCH | `/users/status` | Protegido | `ms01:user:status` | [07](#rf-ms01-07-administración-de-usuarios) |
| **Roles y permisos** | GET | `/roles` | Protegido | `ms01:role:read_own`, `ms01:role:read_all` (catálogo completo) | [05](#rf-ms01-05-gestión-de-roles-y-permisos) |
| | PATCH | `/users/roles` | Protegido | `ms01:user:roles` | [05](#rf-ms01-05-gestión-de-roles-y-permisos) |
| | GET | `/roles/details` | Protegido | `ms01:role:read_all` | [10](#rf-ms01-10-administración-de-roles-y-catálogo-de-permisos) |
| | GET | `/roles/{rol}` | Protegido | `ms01:role:read_all` | [10](#rf-ms01-10-administración-de-roles-y-catálogo-de-permisos) |
| | POST | `/roles` | Protegido | `ms01:role:manage` | [10](#rf-ms01-10-administración-de-roles-y-catálogo-de-permisos) |
| | PUT | `/roles/{rol}` | Protegido | `ms01:role:manage` | [10](#rf-ms01-10-administración-de-roles-y-catálogo-de-permisos) |
| | PUT | `/roles/{rol}/permissions` | Protegido | `ms01:role:manage` | [10](#rf-ms01-10-administración-de-roles-y-catálogo-de-permisos) |
| | DELETE | `/roles/{rol}` | Protegido | `ms01:role:manage` | [10](#rf-ms01-10-administración-de-roles-y-catálogo-de-permisos) |
| | GET | `/permissions` | Protegido | `ms01:role:read_all` | [10](#rf-ms01-10-administración-de-roles-y-catálogo-de-permisos) |
| **Interno (servicios)** | POST | `/users/batch` | Interno (MS03, MS05) | — | [09](#rf-ms01-09-resolución-interna-de-usuarios) |
| | GET | `/users/by-role` | Interno (MS03, MS05) | — | [09](#rf-ms01-09-resolución-interna-de-usuarios) |
| | GET | `/internal/session-cuts` | Interno (solo el Gateway) | — | [11](#rf-ms01-11-corte-de-sesión-por-cambios-de-acceso) |
