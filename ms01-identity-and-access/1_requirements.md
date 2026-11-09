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
| **Descripción** | Mantener la sesión del usuario autenticado durante un tiempo limitado y permitir que el ecosistema lo identifique, con sus roles, en cada petición. |
| **Actor(es)** | Usuarios autenticados. Sistema consumidor: API Gateway (único verificador de la credencial). |
| **Precondiciones** | Autenticación exitosa ([RF-MS01-02](#rf-ms01-02-autenticar-usuario-login)). |
| **Flujo base** | <ol><li>Al iniciar sesión, MS01 emite una credencial de acceso firmada con la identidad del usuario, sus roles vigentes y su vencimiento ([`3_jwt-spec.md`](3_jwt-spec.md)).</li><li>El cliente presenta la credencial en cada petición.</li><li>El API Gateway la verifica. Si es inválida o venció, responde 401 sin llegar a MS01 y el cliente debe iniciar sesión de nuevo. Si es válida, entrega a MS01 la identidad del solicitante ([RN-MS01-23](#rf-ms01-03-gestión-de-sesión)).</li><li>MS01 identifica al usuario y autoriza la operación con sus roles y la Matriz Rol‐Permiso. Si en un endpoint protegido la identidad falta o no es válida, responde 401 `NO_AUTENTICADO`. Si los roles no tienen el permiso requerido, responde 403 `PROHIBIDO`.</li></ol> |
| **Flujos alternos** | — |
| **Reglas De Negocio** | <ul><li>**RN-MS01-07:** La credencial de acceso la firma MS01 y el Gateway la verifica sin consultar a MS01 ([formato y firma](3_jwt-spec.md)).</li><li>**RN-MS01-08:** La credencial no contiene contraseñas ni datos sensibles.</li><li>**RN-MS01-09:** Un cambio de rol o de estado se refleja en la siguiente credencial emitida. Las ya emitidas conservan sus datos hasta vencer, por eso su vigencia es corta.</li><li>**RN-MS01-23:** El registro, el login y la recuperación de contraseña son públicos. Las consultas internas ([RF-MS01-09](#rf-ms01-09-resolución-interna-de-usuarios)) reciben la identidad del servicio que llama. Los demás endpoints reciben la identidad del solicitante (id y roles) solo en las cabeceras que inyecta el Gateway, nunca en el cuerpo ([cabeceras](3_jwt-spec.md#identidad-en-ms01)).</li></ul> |
| **Postcondiciones** | Solo las peticiones con una credencial válida y vigente llegan a los recursos protegidos. |
| **Requerimientos asociados** | [RF-MS01-02](#rf-ms01-02-autenticar-usuario-login), [RF-MS01-04](#rf-ms01-04-cerrar-sesión-logout), [RF-MS01-05](#rf-ms01-05-gestión-de-roles-y-permisos), RF-GW-02 (verificación en el Gateway) |
| **Criterios de aceptación** | <ul><li>El Gateway puede verificar la credencial emitida, que cumple [`3_jwt-spec.md`](3_jwt-spec.md).</li><li>Un endpoint protegido sin identidad responde 401.</li><li>La credencial muestra los roles vigentes al momento del login.</li><li>Tras cambiar el rol de un usuario, su siguiente login emite una credencial con el nuevo rol.</li></ul> |
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
| **Reglas De Negocio** | <ul><li>**RN-MS01-10:** El logout cierra la sesión en el cliente y queda auditado. La credencial sigue siendo válida hasta su vencimiento; la vigencia corta acota este límite ([RN-MS01-09](#rf-ms01-03-gestión-de-sesión)).</li></ul> |
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
| **Descripción** | Permitir al administrador agregar y retirar roles a uno o varios usuarios, y a cualquier usuario consultar los permisos de sus roles. |
| **Actor(es)** | Administrador<br>Secundarios: los demás roles, que consultan solo sus propios roles y permisos. |
| **Precondiciones** | El administrador está autenticado. |
| **Flujo base** | <ol><li>El administrador busca usuarios ([RF-MS01-07](#rf-ms01-07-administración-de-usuarios)) y consulta sus roles actuales.</li><li>Selecciona uno o varios usuarios y solicita agregar o retirar uno o varios roles ([F-09](2_forms-inputs.md#f-09-asignación-de-roles)).</li><li>El sistema verifica, en este orden, y si algo falla no aplica ningún cambio:<ul><li>que el solicitante sea administrador; si no, responde 403 y emite el evento CAMBIO_ROL con resultado FALLO;</li><li>que la solicitud sea válida (usuarios, roles del catálogo, al menos una lista, ningún rol en ambas); si no, responde 400;</li><li>que los usuarios objetivo no incluyan al solicitante; si lo incluyen, responde 403 y emite el evento FALLO;</li><li>que todos los usuarios existan; si alguno no existe, responde 404 y emite un evento FALLO por cada uno;</li><li>que cada usuario conserve al menos un rol; si alguno quedaría sin roles, responde 400 y emite un evento FALLO por cada usuario que quedaría sin roles.</li></ul></li><li>Actualiza los roles de los usuarios.</li><li>Emite el evento CAMBIO_ROL con resultado EXITO por cada usuario (`recurso_afectado` = usuario objetivo, `detalle` con los roles agregados y retirados).</li><li>Responde 200 con los roles resultantes de cada usuario.</li></ol> |
| **Flujos alternos** | <ul><li>**A1, consulta de roles y permisos:** cualquier usuario autenticado consulta los roles con sus permisos. El administrador recibe el catálogo completo; los demás, solo sus propios roles.</li></ul> |
| **Reglas De Negocio** | <ul><li>**RN-MS01-11:** Solo el Administrador asigna o retira roles.</li><li>**RN-MS01-12:** Nadie puede asignarse privilegios por sí mismo, incluido el administrador. El primer administrador se crea mediante un mecanismo de inicialización configurado fuera del repositorio ([datos iniciales](4_modelo-datos.md#datos-iniciales)).</li><li>**RN-MS01-13:** Un usuario puede tener más de un rol, siempre con autorización del administrador.</li><li>**RN-MS01-14:** El catálogo de roles (ESTUDIANTE, PROFESOR, ADMINISTRATIVO, BRIGADISTA, COORDINADOR, ORGANIZADOR, ADMIN, AUDITOR) y la [Matriz Rol‐Permiso](https://github.com/CampusSeguro-DOSW-2026-2/documentation/wiki/Matriz-Rol%E2%80%90Permiso) los define el sistema (versionados en el código) y no se editan en tiempo de ejecución.</li></ul> |
| **Postcondiciones** | Los nuevos roles se reflejan en la siguiente sesión ([RN-MS01-09](#rf-ms01-03-gestión-de-sesión)). |
| **Requerimientos asociados** | [RF-MS01-03](#rf-ms01-03-gestión-de-sesión), [RF-MS01-07](#rf-ms01-07-administración-de-usuarios) |
| **Criterios de aceptación** | <ul><li>Un administrador agrega o retira roles a uno o varios usuarios en una sola operación y la consulta posterior lo muestra.</li><li>Tras el cambio, el siguiente login del usuario refleja el nuevo rol.</li><li>Un administrador no puede modificar sus propios roles.</li><li>Un no administrador recibe 403.</li><li>Cada cambio queda auditado.</li></ul> |
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
| **Flujo base** | <ol><li>El administrador o el auditor lista o busca usuarios (por correo, con paginación y filtro opcional por estado o rol, [F-07](2_forms-inputs.md#f-07-búsqueda-y-filtros-de-usuarios)) o consulta uno por su id. Si no tiene permiso de consulta, el sistema responde 403. Si los parámetros son inválidos, responde 400. Si el usuario consultado no existe, responde 404.</li><li>El administrador selecciona uno o varios usuarios y solicita el nuevo estado ([F-08](2_forms-inputs.md#f-08-cambio-de-estado-de-usuario)).</li><li>El sistema verifica, en este orden, y si algo falla no aplica ningún cambio:<ul><li>que el solicitante sea administrador; si no, responde 403 y emite el evento CAMBIO_ESTADO con resultado FALLO;</li><li>que la solicitud sea válida; si no, responde 400;</li><li>que no esté cambiando su propia cuenta; si lo hace, responde 403 y emite el evento FALLO;</li><li>que todos los usuarios existan; si alguno no existe, responde 404 y emite un evento FALLO por cada uno.</li></ul></li><li>Actualiza el estado.</li><li>Emite el evento CAMBIO_ESTADO con resultado EXITO por cada usuario (`detalle` con el estado anterior y el nuevo).</li><li>Responde 200 con el estado anterior y el nuevo de cada usuario.</li></ol> |
| **Flujos alternos** | <ul><li>**A1 (paso 4), error al guardar:** el sistema responde 500 `ERROR_INTERNO` sin detalles internos, no aplica ningún cambio y emite el evento CAMBIO_ESTADO con resultado FALLO por cada usuario.</li></ul> |
| **Reglas De Negocio** | <ul><li>**RN-MS01-17:** Los únicos estados son ACTIVO e INACTIVO. Toda cuenta nueva nace ACTIVA.</li><li>**RN-MS01-18:** Una cuenta INACTIVA no puede iniciar sesión ([RN-MS01-06](#rf-ms01-02-autenticar-usuario-login)). Las credenciales ya emitidas vencen por su vigencia ([RN-MS01-09](#rf-ms01-03-gestión-de-sesión)).</li><li>**RN-MS01-19:** Solo el administrador cambia estados, y no puede cambiar el propio. El auditor solo consulta.</li></ul> |
| **Postcondiciones** | El estado queda actualizado y condiciona el siguiente login. |
| **Requerimientos asociados** | [RF-MS01-02](#rf-ms01-02-autenticar-usuario-login), [RF-MS01-05](#rf-ms01-05-gestión-de-roles-y-permisos) |
| **Criterios de aceptación** | <ul><li>Tras inactivar un usuario, su login falla con 403 aunque las credenciales sean correctas.</li><li>Tras reactivarlo, su login funciona.</li><li>Un administrador no puede inactivarse a sí mismo.</li><li>Cada cambio de estado queda auditado.</li></ul> |
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
| **Flujo base** | <ul><li><b>Flujo base A, consulta por ids:</b><ol><li>El servicio envía de 1 a 100 ids de usuario.</li><li>El sistema verifica que el servicio esté identificado y autorizado ([RN-MS01-24](#rf-ms01-09-resolución-interna-de-usuarios)). Si no está identificado, responde 401; si no está autorizado, responde 403.</li><li>Valida la lista: entre 1 y 100 ids, cada uno con formato válido y sin campos adicionales. Si algo falla, responde 400 indicando el campo.</li><li>Responde 200 con los usuarios encontrados, sin repetidos y en el orden pedido, y con la lista de ids que no existen.</li></ol></li><li><b>Flujo base B, consulta por rol:</b><ol><li>El servicio envía el rol, la página y, opcionalmente, una fecha de corte.</li><li>El sistema verifica el servicio como en el paso A2.</li><li>Valida que el rol sea del catálogo y que la fecha de corte, la página y el tamaño sean válidos ([F-07](2_forms-inputs.md#f-07-búsqueda-y-filtros-de-usuarios) para la paginación). Si algo falla, responde 400 indicando el campo.</li><li>Responde 200 con la página de usuarios que tienen el rol y, si llegó la fecha de corte, que se registraron hasta ese momento, en un orden fijo.</li></ol></li></ul> |
| **Flujos alternos** | — |
| **Reglas De Negocio** | <ul><li>**RN-MS01-24:** Solo los servicios configurados como autorizados usan estas consultas. No se autorizan con la Matriz Rol‐Permiso ni con los roles de un usuario.</li><li>**RN-MS01-25:** Las respuestas incluyen usuarios activos e inactivos, con su estado. El servicio que consulta decide a cuáles usar.</li><li>**RN-MS01-26:** Estas consultas no modifican datos ni generan eventos de auditoría.</li></ul> |
| **Postcondiciones** | Ninguna: solo lectura. |
| **Requerimientos asociados** | [RF-MS01-03](#rf-ms01-03-gestión-de-sesión), [RF-MS01-07](#rf-ms01-07-administración-de-usuarios) |
| **Criterios de aceptación** | <ul><li>Una consulta con ids existentes e inexistentes devuelve 200 con los usuarios encontrados y los ids que no existen.</li><li>Más de 100 ids, o un id con formato inválido, devuelve 400.</li><li>Un servicio no autorizado recibe 403, y una llamada sin servicio identificado, 401.</li><li>Recorriendo todas las páginas con la misma fecha de corte se obtienen todos los usuarios con el rol, sin repetidos ni faltantes.</li></ul> |
| **Prioridad** | Alta |
| **Estado / Squad responsable** | Implementado / kAiRoS |

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
| **RNF-MS01-10** | La auditoría se delega en [ms08-analytics-and-audit](https://github.com/CampusSeguro-DOSW-2026-2/ms08-analytics-and-audit) (MS08), que almacena y consulta los eventos. MS01 los emite de forma asíncrona con el formato de los Estándares (`modulo` = MS01). Acciones de MS01: REGISTRO, LOGIN, LOGOUT, CAMBIO_ROL, CAMBIO_ESTADO, CAMBIO_CONTRASENA, RECUPERACION; resultado EXITO o FALLO. Si MS08 falla, la respuesta al cliente no cambia y el error queda en el log. Las consultas internas ([RF-MS01-09](#rf-ms01-09-resolución-interna-de-usuarios)) no se auditan. MS01 no tiene tablas ni endpoints de auditoría. |

---

## API: ms01-identity-and-access

Contrato completo: [`api/swagger.yml`](api/swagger.yml). Los endpoints **protegidos** reciben la identidad según [RN-MS01-23](#rf-ms01-03-gestión-de-sesión); los **internos**, la del servicio que llama ([RN-MS01-24](#rf-ms01-09-resolución-interna-de-usuarios)). El *Permiso* coincide con el `x-permiso` del contrato. Los roles que lo tienen están en la [Matriz Rol‐Permiso](https://github.com/CampusSeguro-DOSW-2026-2/documentation/wiki/Matriz-Rol%E2%80%90Permiso).

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
| **Interno (servicios)** | POST | `/users/batch` | Interno (MS03, MS05) | — | [09](#rf-ms01-09-resolución-interna-de-usuarios) |
| | GET | `/users/by-role` | Interno (MS03, MS05) | — | [09](#rf-ms01-09-resolución-interna-de-usuarios) |
