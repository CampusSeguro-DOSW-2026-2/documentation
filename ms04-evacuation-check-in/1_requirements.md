# Requerimientos — MS04 Evacuación y Check-in

Los valores transversales (cabeceras, errores, correlación, paginación, llamadas internas, auditoría) se rigen por los [Estándares de Integración](https://github.com/CampusSeguro-DOSW-2026-2/documentation/wiki/Est%C3%A1ndares-de-Integraci%C3%B3n-y-Contratos-Transversales) y los permisos por la [Matriz Rol‐Permiso](https://github.com/CampusSeguro-DOSW-2026-2/documentation/wiki/Matriz-Rol%E2%80%90Permiso). Este documento solo define el comportamiento de MS04.

## Índice de Contenido
1. [Criterios de Separación](#criterios-de-separacion)
2. [Estados del check-in](#estados-del-check-in)
3. [Requerimientos Funcionales](#requerimientos-funcionales)
   - [RF-MS04-01 Consultar evento activo](#rf-ms04-01)
   - [RF-MS04-02 Registrar check-in](#rf-ms04-02)
   - [RF-MS04-03 Mantener y actualizar el estado vigente](#rf-ms04-03)
   - [RF-MS04-04 Generar solicitud de asistencia en MS05](#rf-ms04-04)
   - [RF-MS04-05 Consultar el propio estado vigente](#rf-ms04-05)
   - [RF-MS04-06 Consolidar el estado del evento (dashboard)](#rf-ms04-06)
   - [RF-MS04-07 Identificar personas pendientes](#rf-ms04-07)
   - [RF-MS04-08 Consultar personas que requieren asistencia](#rf-ms04-08)
   - [RF-MS04-09 Conservar eventos finalizados y consultar su histórico](#rf-ms04-09)
   - [RF-MS04-10 Entregar resultados del evento](#rf-ms04-10)
4. [Requerimientos No Funcionales (RNF)](#requerimientos-no-funcionales-rnf)
5. [Dependencias y decisiones abiertas](#dependencias-y-decisiones-abiertas)
6. [API: ms04-evacuation-and-check-in](#api-ms04-evacuation-and-check-in)

---

<a id="criterios-de-separacion"></a>
## Criterios de Separación

**¿Cuándo un requerimiento merece existir por separado?**
* Se reutiliza en varios flujos (auth, notificaciones, uploads).
* Tiene reglas de negocio propias y no triviales.
* Pertenece a otro microservicio/squad con su propio contrato de API o evento.
* Tiene su propio ciclo de pruebas independiente.
* Si un paso no cumple ninguna, va dentro del flujo base o como flujo alterno, no como requerimiento aparte.

---

<a id="estados-del-check-in"></a>
## Estados del check-in

Durante un evento activo, el usuario reporta exactamente uno de estos cuatro estados ([RN-MS04-03](#rn-ms04-03)). El texto es el que ve el usuario. El código es el valor del contrato.

| Estado (texto al usuario) | Código | Datos obligatorios | Efecto en MS04 |
|---|---|---|---|
| **Estoy a salvo** | `SAFE` | Punto de encuentro (existente y ACTIVO en MS02) | Se guarda el punto de encuentro y la persona cuenta en la consolidación de ese punto. |
| **Me estoy evacuando** | `EVACUATING` | Ninguno | La persona cuenta como reportada y en evacuación. |
| **Requiero asistencia** | `NEEDS_ASSISTANCE` | Ubicación (edificio, espacio o punto de encuentro de MS02) | Se genera una solicitud de asistencia en MS05 ([RF-MS04-04](#rf-ms04-04)) y la persona aparece en la lista de asistencia ([RF-MS04-08](#rf-ms04-08)). |
| **No estoy en el campus** | `OFF_CAMPUS` | Ninguno | La persona cuenta como reportada y deja de ser pendiente, pero no se asocia a ningún punto de encuentro. |

* Los cuatro estados cuentan como “reportado”: ninguno deja a la persona en la lista de pendientes.
* Mientras el evento esté activo, el usuario puede pasar de cualquier estado a cualquier otro ([RF-MS04-03](#rf-ms04-03)).
* La consolidación siempre muestra los cuatro estados, con 0 cuando no hay personas en alguno.

---

<a id="requerimientos-funcionales"></a>
## Requerimientos Funcionales

<a id="rf-ms04-01"></a>
### RF-MS04-01 Consultar evento activo

| Atributo | Detalle |
|---|---|
| **ID** | RF-MS04-01 |
| **Nombre** | Consultar evento activo |
| **Épica / Grupo funcional** | MS04 - Evento activo |
| **Descripción** | Permitir que el usuario autenticado sepa si existe un evento activo (simulacro o emergencia) que le aplique, para poder reportar su estado. |
| **Actor(es)** | ESTUDIANTE, PROFESOR, ADMINISTRATIVO, BRIGADISTA, COORDINADOR, ORGANIZADOR |
| **Precondiciones** | Usuario autenticado. Las peticiones llegan con las cabeceras de identidad (X-User-Id, X-User-Role, X-Correlation-ID). |
| **Flujo base** | <ol><li>El cliente solicita los eventos activos.</li><li>El sistema identifica al usuario (X-User-Id) y valida su permiso (X-User-Role).</li><li>Consulta a MS03, a través del API Gateway, las alertas en estado Activa de las que el usuario es destinatario ([RN-MS04-01](#rn-ms04-01)).</li><li>Responde con los eventos activos que le aplican: id del evento, tipo (simulacro o emergencia), alcance y fecha de inicio.</li></ol> |
| **Flujos alternos / excepciones** | <ul><li>**A1 (paso 3), no hay evento activo que le aplique:** el sistema responde 200 con una lista vacía y el cliente muestra “No hay ningún evento activo”.</li><li>**A2 (paso 3), varios eventos activos le aplican:** el sistema los devuelve todos, ordenados por fecha de inicio.</li><li>**A3 (paso 3), MS03 no responde:** el sistema responde 503 con un mensaje controlado y sin detalles internos.</li></ul> |
| **Reglas De Negocio** | <ul><li><a id="rn-ms04-01"></a>**RN-MS04-01:** Un evento es una alerta de MS03 en estado Activa, sea de emergencia o de simulacro (generada por MS06). Su id de evento es el id de esa alerta. Un evento le aplica a un usuario solo si es destinatario de la alerta según MS03, que resuelve la segmentación (campus completo, edificios o zonas).</li><li><a id="rn-ms04-02"></a>**RN-MS04-02:** MS04 obtiene información de otros microservicios (MS02, MS03 y MS05) únicamente por sus API REST a través del API Gateway, nunca accediendo a sus bases de datos. Del usuario solo almacena su id (X-User-Id), sin duplicar datos personales.</li></ul> |
| **Postcondiciones** | El usuario conoce el evento activo o sabe que no hay ninguno. No se modifica ningún dato. |
| **Requerimientos asociados** | [RF-MS04-02](#rf-ms04-02), [RF-MS04-05](#rf-ms04-05). Depende de MS03. |
| **Criterios de aceptación** | <ul><li>Con una alerta Activa de la que el usuario es destinatario, la consulta devuelve 200 con el id y los datos del evento.</li><li>Sin alertas Activas que le apliquen, la consulta devuelve 200 con una lista vacía.</li><li>Una alerta Activa de la que el usuario no es destinatario no aparece en su consulta.</li><li>Si MS03 no responde, la consulta devuelve 503 sin exponer detalles internos.</li></ul> |
| **Prioridad** | Alta |
| **Estado / Squad responsable** | Pendiente / kAiRoS |

<br>

<a id="rf-ms04-02"></a>
### RF-MS04-02 Registrar check-in

| Atributo | Detalle |
|---|---|
| **ID** | RF-MS04-02 |
| **Nombre** | Registrar check-in |
| **Épica / Grupo funcional** | MS04 - Check-in |
| **Descripción** | Permitir que el usuario reporte, durante un evento activo, uno de los cuatro estados: Estoy a salvo, Me estoy evacuando, Requiero asistencia o No estoy en el campus. Si está a salvo, se guarda el punto de encuentro donde está. |
| **Actor(es)** | ESTUDIANTE, PROFESOR, ADMINISTRATIVO, BRIGADISTA, COORDINADOR, ORGANIZADOR |
| **Precondiciones** | Usuario autenticado. Existe un evento activo que le aplica ([RN-MS04-01](#rn-ms04-01)). Para Estoy a salvo, el punto de encuentro existe en MS02. |
| **Flujo base** | <ol><li>El cliente envía el estado y, según el estado, el punto de encuentro (Estoy a salvo) o la ubicación (Requiero asistencia).</li><li>El sistema identifica al usuario (X-User-Id) y valida su permiso (X-User-Role).</li><li>Valida que el estado sea uno de los cuatro del catálogo ([RN-MS04-03](#rn-ms04-03)) y que vengan los datos obligatorios para ese estado.</li><li>Valida con MS03 que el evento exista, esté Activo y le aplique al usuario.</li><li>Si el estado es Estoy a salvo, valida con MS02 que el punto de encuentro exista y esté ACTIVO.</li><li>Verifica que el usuario no tenga un check-in en el evento.</li><li>Guarda el check-in como estado vigente y lo agrega al histórico.</li><li>Emite a MS08 el evento de auditoría (acción REGISTRO_CHECKIN, resultado EXITO).</li><li>Si el estado es Requiero asistencia, ejecuta [RF-MS04-04](#rf-ms04-04).</li><li>Responde 201 con el id del check-in, el estado aceptado y la fecha/hora de registro.</li></ol> |
| **Flujos alternos / excepciones** | <ul><li>**A1 (paso 3), datos inválidos:** si el estado no es uno de los cuatro, falta el punto de encuentro en Estoy a salvo o falta la ubicación en Requiero asistencia, el sistema responde 400 con el detalle de los campos, no registra el check-in y emite a MS08 el evento de auditoría.</li><li>**A2 (paso 1), el cuerpo incluye un identificador de usuario:** el sistema rechaza la petición (400). La identidad solo llega por X-User-Id.</li><li>**A3 (paso 4), evento inexistente:** el sistema responde 404 y emite a MS08 el evento de auditoría.</li><li>**A4 (paso 4), evento finalizado o no activo:** el sistema responde 409, indica que no existe un evento activo, no crea el check-in y emite a MS08 el evento de auditoría.</li><li>**A5 (paso 4), el evento no le aplica al usuario:** el sistema responde 422, no crea el check-in y emite a MS08 el evento de auditoría.</li><li>**A6 (paso 5), punto de encuentro inexistente o inactivo:** el sistema responde 404 si no existe o 422 si está inactivo, no crea el check-in y emite a MS08 el evento de auditoría.</li><li>**A7 (pasos 4 y 5), MS03 o MS02 no responden:** el sistema responde 503 con un mensaje controlado, no crea el check-in y emite a MS08 el evento de auditoría.</li><li>**A8 (paso 6), el usuario ya tiene check-in en el evento:** la petición se procesa como actualización ([RF-MS04-03](#rf-ms04-03)).</li></ul> |
| **Reglas De Negocio** | <ul><li><a id="rn-ms04-03"></a>**RN-MS04-03:** Los únicos estados permitidos para un check-in son Estoy a salvo (`SAFE`), Me estoy evacuando (`EVACUATING`), Requiero asistencia (`NEEDS_ASSISTANCE`) y No estoy en el campus (`OFF_CAMPUS`), con los datos obligatorios de la tabla [Estados del check-in](#estados-del-check-in).</li><li><a id="rn-ms04-04"></a>**RN-MS04-04:** Estoy a salvo exige un punto de encuentro existente y ACTIVO en MS02, que se almacena con el check-in. Los demás estados no se asocian a un punto de encuentro.</li><li><a id="rn-ms04-05"></a>**RN-MS04-05:** No se puede registrar ni actualizar un check-in si el evento no existe o no está Activo.</li><li><a id="rn-ms04-06"></a>**RN-MS04-06:** El usuario solo reporta y consulta su propio estado vigente. Se identifica únicamente por X-User-Id, nunca por el cuerpo de la petición.</li><li><a id="rn-ms04-07"></a>**RN-MS04-07:** Solo puede existir un estado vigente por usuario y evento, y esta unicidad se garantiza también en la base de datos.</li></ul> |
| **Postcondiciones** | Existe un check-in vigente para el usuario en el evento. Si fue Estoy a salvo, el punto de encuentro queda almacenado. |
| **Requerimientos asociados** | [RF-MS04-01](#rf-ms04-01), [RF-MS04-03](#rf-ms04-03), [RF-MS04-04](#rf-ms04-04), [RF-MS04-05](#rf-ms04-05). Depende de MS02 y MS03. |
| **Criterios de aceptación** | <ul><li>Cada uno de los cuatro estados, con sus datos obligatorios, en un evento activo que le aplica al usuario, devuelve 201 con el id del check-in, el estado aceptado y la fecha/hora.</li><li>Un estado distinto de los cuatro devuelve 400 y no crea el check-in.</li><li>Estoy a salvo sin punto de encuentro devuelve 400 y no crea el check-in.</li><li>Un check-in en un evento finalizado devuelve 409 y no crea el check-in.</li><li>Un punto de encuentro inactivo devuelve 422 y no crea el check-in.</li><li>Cada intento, exitoso o fallido, genera un evento de auditoría hacia MS08.</li></ul> |
| **Prioridad** | Alta |
| **Estado / Squad responsable** | Pendiente / kAiRoS |

<br>

<a id="rf-ms04-03"></a>
### RF-MS04-03 Mantener y actualizar el estado vigente

| Atributo | Detalle |
|---|---|
| **ID** | RF-MS04-03 |
| **Nombre** | Mantener y actualizar el estado vigente |
| **Épica / Grupo funcional** | MS04 - Check-in |
| **Descripción** | Garantizar un único estado vigente por usuario y evento, y permitir cambiarlo entre los cuatro estados mientras el evento siga activo, conservando cada cambio en el histórico. |
| **Actor(es)** | ESTUDIANTE, PROFESOR, ADMINISTRATIVO, BRIGADISTA, COORDINADOR, ORGANIZADOR |
| **Precondiciones** | El usuario ya tiene un check-in en el evento. El evento sigue activo. |
| **Flujo base** | <ol><li>El cliente envía el nuevo estado (y el punto de encuentro o la ubicación, según el estado).</li><li>El sistema valida el estado, los datos obligatorios, que el evento siga Activo y, si es Estoy a salvo, el punto de encuentro (pasos 3 a 5 de [RF-MS04-02](#rf-ms04-02)).</li><li>En una sola operación, el nuevo estado reemplaza al vigente y se agrega al histórico.</li><li>Emite a MS08 el evento de auditoría (acción CAMBIO_ESTADO_CHECKIN, `recurso_afectado` = check-in, `detalle` con el estado anterior y el nuevo).</li><li>Si el nuevo estado es Requiero asistencia, ejecuta [RF-MS04-04](#rf-ms04-04).</li><li>Responde 200 con el id del check-in, el nuevo estado y la fecha/hora del cambio.</li></ol> |
| **Flujos alternos / excepciones** | <ul><li>**A1 (paso 2), evento finalizado:** el sistema responde 409, no modifica nada y emite a MS08 el evento de auditoría.</li><li>**A2 (paso 2), datos inválidos:** el sistema responde 400 con el detalle de los campos, no modifica nada y emite a MS08 el evento de auditoría. Aplica también al cambio a Estoy a salvo sin punto de encuentro.</li><li>**A3 (paso 3), mismo estado que el vigente (y, si es Estoy a salvo, mismo punto de encuentro):** el sistema responde 200 con el estado vigente sin agregar una entrada al histórico, y emite a MS08 el evento de auditoría del intento.</li><li>**A4 (paso 3), Estoy a salvo con otro punto de encuentro:** se trata como un cambio y se agrega al histórico.</li><li>**A5 (paso 3), dos cambios simultáneos del mismo usuario en el mismo evento:** la restricción de unicidad de la base de datos deja un solo estado vigente. La petición rechazada recibe 409, no modifica nada y emite a MS08 el evento de auditoría. El cliente puede reintentar.</li></ul> |
| **Reglas De Negocio** | <ul><li><a id="rn-ms04-08"></a>**RN-MS04-08:** El histórico contiene todos los estados reportados por el usuario en el evento (estado, punto de encuentro si aplica y fecha/hora), en orden cronológico. No se modifica ni se elimina, y solo lo consultan los roles con `ms04:history:read` ([RF-MS04-09](#rf-ms04-09)).</li><li>Aplican [RN-MS04-03](#rn-ms04-03), [RN-MS04-04](#rn-ms04-04), [RN-MS04-05](#rn-ms04-05) y [RN-MS04-07](#rn-ms04-07).</li></ul> |
| **Postcondiciones** | El usuario tiene exactamente un estado vigente en el evento y el cambio queda en el histórico. |
| **Requerimientos asociados** | [RF-MS04-02](#rf-ms04-02), [RF-MS04-04](#rf-ms04-04), [RF-MS04-05](#rf-ms04-05), [RF-MS04-09](#rf-ms04-09) |
| **Criterios de aceptación** | <ul><li>Un usuario en Me estoy evacuando que actualiza a Estoy a salvo con un punto de encuentro activo queda con Estoy a salvo como único estado vigente, y ambos estados quedan en el histórico en orden cronológico.</li><li>Un cambio de estado en un evento finalizado devuelve 409 y no tiene efecto.</li><li>Dos peticiones simultáneas del mismo usuario dejan un solo estado vigente y la rechazada recibe 409, no 500.</li><li>Cada cambio, exitoso o fallido, genera un evento de auditoría hacia MS08.</li></ul> |
| **Prioridad** | Alta |
| **Estado / Squad responsable** | Pendiente / kAiRoS |

<br>

<a id="rf-ms04-04"></a>
### RF-MS04-04 Generar solicitud de asistencia en MS05

| Atributo | Detalle |
|---|---|
| **ID** | RF-MS04-04 |
| **Nombre** | Generar solicitud de asistencia en MS05 |
| **Épica / Grupo funcional** | MS04 - Asistencia |
| **Descripción** | Cuando un usuario reporta Requiero asistencia, MS04 crea una solicitud de asistencia en MS05 con su ubicación, para que el coordinador la asigne a un brigadista. La asignación y la atención de la solicitud ocurren en MS05. |
| **Actor(es)** | Disparan la acción: ESTUDIANTE, PROFESOR, ADMINISTRATIVO, BRIGADISTA, COORDINADOR, ORGANIZADOR (`ms04:assistance:request`).<br>Ejecuta la integración: sistema MS04. |
| **Precondiciones** | Check-in Requiero asistencia aceptado ([RF-MS04-02](#rf-ms04-02) o [RF-MS04-03](#rf-ms04-03)). La llamada interna MS04 → MS05 está registrada en los Estándares. |
| **Flujo base** | <ol><li>El usuario reporta Requiero asistencia con su ubicación y MS04 registra el check-in.</li><li>MS04 verifica que el usuario no tenga ya una solicitud de asistencia en el evento ([RN-MS04-11](#rn-ms04-11)).</li><li>MS04 envía a MS05, a través del API Gateway (ruta interna, con su clave de servicio y la identidad del usuario propagada), el id del evento, el id del usuario, la ubicación y el id del check-in.</li><li>MS05 crea la solicitud en estado Reportada y devuelve su id.</li><li>MS04 guarda el id de la solicitud asociado al check-in y marca su envío como ENVIADA.</li><li>Emite a MS08 el evento de auditoría (acción SOLICITUD_ASISTENCIA, resultado EXITO).</li><li>La respuesta del check-in indica que la solicitud fue enviada.</li></ol> |
| **Flujos alternos / excepciones** | <ul><li>**A1 (paso 1), sin ubicación:** se aplica el flujo A1 de [RF-MS04-02](#rf-ms04-02) (400, no se registra el check-in).</li><li>**A2 (paso 3), MS05 no disponible o sin respuesta dentro del timeout:** el check-in se conserva y la solicitud queda PENDIENTE_ENVIO. MS04 registra el fallo en su log, emite a MS08 el evento de auditoría (FALLO) y la respuesta del check-in indica que la solicitud está pendiente. MS04 reenvía las solicitudes pendientes cuando MS05 se recupera ([RN-MS04-12](#rn-ms04-12)).</li><li>**A3 (paso 2), el usuario ya tiene una solicitud en el evento (enviada o pendiente de envío):** no se crea otra y se reutiliza la existente.</li><li>**A4 (paso 4), MS05 rechaza la solicitud por datos inválidos:** el check-in se conserva, la solicitud queda RECHAZADA y no se reenvía. El error queda en el log, se emite a MS08 el evento de auditoría (FALLO) y la respuesta del check-in indica que la solicitud no pudo crearse. El coordinador la ve en [RF-MS04-08](#rf-ms04-08).</li><li>**A5, el usuario cambia de estado mientras la solicitud está pendiente de envío:** el cambio queda en el histórico y la solicitud pendiente se envía igual. Su atención o cierre se decide en MS05.</li></ul> |
| **Reglas De Negocio** | <ul><li><a id="rn-ms04-09"></a>**RN-MS04-09:** Toda solicitud de asistencia incluye la ubicación del usuario, expresada como un edificio, espacio o punto de encuentro de MS02 (tipo e id), según lo exige MS05.</li><li><a id="rn-ms04-10"></a>**RN-MS04-10:** MS04 conserva el id de la solicitud devuelto por MS05, asociado al check-in, y el estado de su envío: ENVIADA, PENDIENTE_ENVIO o RECHAZADA.</li><li><a id="rn-ms04-11"></a>**RN-MS04-11:** Un usuario no puede tener más de una solicitud de asistencia abierta por evento originada en MS04.</li><li><a id="rn-ms04-12"></a>**RN-MS04-12:** La indisponibilidad de MS05 no impide registrar el check-in. Las solicitudes que no se pudieron enviar se reintentan cuando MS05 se recupera, sin generar solicitudes duplicadas para el mismo check-in.</li></ul> |
| **Postcondiciones** | Existe una solicitud en MS05 y MS04 conserva su id vinculado al check-in, o la solicitud queda pendiente de envío o rechazada. |
| **Requerimientos asociados** | [RF-MS04-02](#rf-ms04-02), [RF-MS04-03](#rf-ms04-03), [RF-MS04-08](#rf-ms04-08). Depende de MS05. |
| **Criterios de aceptación** | <ul><li>Un Requiero asistencia con ubicación, en un evento activo, crea la solicitud en MS05 y MS04 conserva su id asociado al check-in.</li><li>Un Requiero asistencia sin ubicación devuelve 400 y no registra el check-in.</li><li>Con MS05 detenido, el check-in se registra con éxito y la solicitud queda pendiente. Al recuperarse MS05, se envía una sola vez y se guarda su id.</li><li>Un segundo Requiero asistencia del mismo usuario en el mismo evento no crea otra solicitud.</li><li>Cada intento de envío, exitoso o fallido, genera un evento de auditoría hacia MS08.</li></ul> |
| **Prioridad** | Alta |
| **Estado / Squad responsable** | Pendiente / kAiRoS (coordinar contrato con MS05) |

<br>

<a id="rf-ms04-05"></a>
### RF-MS04-05 Consultar el propio estado vigente

| Atributo | Detalle |
|---|---|
| **ID** | RF-MS04-05 |
| **Nombre** | Consultar el propio estado vigente |
| **Épica / Grupo funcional** | MS04 - Check-in |
| **Descripción** | Permitir que el usuario consulte su estado vigente en un evento, para verificar lo último que reportó. |
| **Actor(es)** | ESTUDIANTE, PROFESOR, ADMINISTRATIVO, BRIGADISTA, COORDINADOR, ORGANIZADOR (`ms04:checkin:read_own`) |
| **Precondiciones** | Usuario autenticado. El evento existe (activo o finalizado). |
| **Flujo base** | <ol><li>El usuario selecciona el evento.</li><li>El sistema identifica al usuario (X-User-Id) y obtiene su estado vigente en ese evento.</li><li>Responde con el estado vigente, su fecha/hora, el punto de encuentro (si es Estoy a salvo) y el estado de envío de la solicitud de asistencia (si es Requiero asistencia).</li></ol> |
| **Flujos alternos / excepciones** | <ul><li>**A1 (paso 2), el usuario no ha reportado estado en el evento:** el sistema responde 404 e indica que no tiene check-in en ese evento.</li><li>**A2 (paso 2), evento inexistente:** el sistema responde 404.</li></ul> |
| **Reglas De Negocio** | <ul><li>Aplican [RN-MS04-06](#rn-ms04-06), [RN-MS04-07](#rn-ms04-07) y [RN-MS04-08](#rn-ms04-08): el usuario ve solo su estado vigente, no el histórico de cambios.</li></ul> |
| **Postcondiciones** | El usuario visualiza su estado vigente. No se modifica ningún dato. |
| **Requerimientos asociados** | [RF-MS04-01](#rf-ms04-01), [RF-MS04-02](#rf-ms04-02), [RF-MS04-03](#rf-ms04-03) |
| **Criterios de aceptación** | <ul><li>Un usuario con varios cambios en un evento recibe solo su último estado vigente.</li><li>La respuesta no incluye el histórico de cambios.</li><li>Un usuario sin check-in en el evento recibe 404.</li><li>Un usuario no puede consultar el estado de otro mediante este flujo.</li></ul> |
| **Prioridad** | Media |
| **Estado / Squad responsable** | Pendiente / kAiRoS |

<br>

<a id="rf-ms04-06"></a>
### RF-MS04-06 Consolidar el estado del evento (dashboard)

| Atributo | Detalle |
|---|---|
| **ID** | RF-MS04-06 |
| **Nombre** | Consolidar el estado del evento (dashboard) |
| **Épica / Grupo funcional** | MS04 - Consolidación |
| **Descripción** | Permitir consultar, durante y después del evento, el número de personas en cada uno de los cuatro estados y el número de personas a salvo por punto de encuentro, para supervisar la evacuación o hacer seguimiento al simulacro. |
| **Actor(es)** | COORDINADOR (supervisa la emergencia o el simulacro), ORGANIZADOR (seguimiento del simulacro), AUDITOR (solo lectura).<br>BRIGADISTA: limitado a sus zonas asignadas. |
| **Precondiciones** | Usuario autenticado con `ms04:dashboard:read`. El evento existe (activo o finalizado). |
| **Flujo base** | <ol><li>El usuario autorizado selecciona un evento.</li><li>El sistema valida su permiso (X-User-Role).</li><li>Consolida los estados vigentes: total por cada uno de los cuatro estados, total de personas a salvo por punto de encuentro y total general de personas que reportaron.</li><li>Responde con el consolidado y la fecha/hora en que se calculó.</li><li>El usuario puede consultar, con paginación, las personas de un estado o de un punto de encuentro (p. ej. las personas a salvo en un punto).</li></ol> |
| **Flujos alternos / excepciones** | <ul><li>**A1 (paso 3), evento sin check-ins:** los cuatro totales son 0.</li><li>**A2 (paso 2), rol sin permiso:** el sistema responde 403.</li><li>**A3 (paso 1), evento inexistente:** el sistema responde 404.</li><li>**A4 (paso 3), el solicitante es BRIGADISTA:** el consolidado y las listas se limitan a sus zonas asignadas ([RN-MS04-14](#rn-ms04-14)).</li></ul> |
| **Reglas De Negocio** | <ul><li><a id="rn-ms04-13"></a>**RN-MS04-13:** El consolidado siempre incluye los cuatro estados (0 si no hay personas). La suma de los cuatro es igual al número de usuarios con estado vigente en el evento. Solo las personas en Estoy a salvo se cuentan por punto de encuentro.</li><li><a id="rn-ms04-14"></a>**RN-MS04-14:** La consolidación, los pendientes y la lista de asistencia requieren `ms04:dashboard:read` (BRIGADISTA, COORDINADOR, ORGANIZADOR, AUDITOR) y son de solo lectura. El BRIGADISTA ve solo la información de sus zonas asignadas en MS05 (edificios, zonas o puntos de encuentro). El ADMIN no tiene permisos en MS04.</li><li>Aplica [RN-MS04-07](#rn-ms04-07).</li></ul> |
| **Postcondiciones** | El usuario dispone del consolidado. No se modifica ningún dato. |
| **Requerimientos asociados** | [RF-MS04-02](#rf-ms04-02), [RF-MS04-03](#rf-ms04-03), [RF-MS04-07](#rf-ms04-07), [RF-MS04-08](#rf-ms04-08), [RF-MS04-10](#rf-ms04-10). Depende de MS05 (zonas del brigadista). |
| **Criterios de aceptación** | <ul><li>El consolidado muestra los cuatro estados y su suma coincide con el número de usuarios con estado vigente.</li><li>Las personas a salvo aparecen agrupadas por su punto de encuentro.</li><li>Un cambio de estado mueve a la persona de un total a otro sin contarla dos veces.</li><li>Un rol sin `ms04:dashboard:read` recibe 403.</li><li>Un BRIGADISTA solo ve información de sus zonas asignadas.</li></ul> |
| **Prioridad** | Alta |
| **Estado / Squad responsable** | Pendiente / kAiRoS |

<br>

<a id="rf-ms04-07"></a>
### RF-MS04-07 Identificar personas pendientes

| Atributo | Detalle |
|---|---|
| **ID** | RF-MS04-07 |
| **Nombre** | Identificar personas pendientes |
| **Épica / Grupo funcional** | MS04 - Consolidación |
| **Descripción** | Permitir identificar a las personas de la población objetivo que todavía no han reportado ninguno de los cuatro estados. |
| **Actor(es)** | COORDINADOR, ORGANIZADOR, AUDITOR (solo lectura).<br>BRIGADISTA: limitado a sus zonas asignadas. |
| **Precondiciones** | Usuario autenticado con `ms04:dashboard:read`. El evento existe (activo o finalizado). |
| **Flujo base** | <ol><li>El usuario autorizado solicita los pendientes de un evento.</li><li>El sistema valida su permiso (X-User-Role).</li><li>Obtiene de MS03 la población objetivo: los destinatarios de la alerta ([RN-MS04-15](#rn-ms04-15)).</li><li>Determina los pendientes: población objetivo sin estado vigente.</li><li>Responde, con paginación, el total de pendientes y la lista de sus ids de usuario.</li></ol> |
| **Flujos alternos / excepciones** | <ul><li>**A1 (paso 4), evento sin check-ins:** toda la población objetivo aparece como pendiente.</li><li>**A2 (paso 4), todos reportaron:** el sistema responde con total 0 y lista vacía.</li><li>**A3 (paso 3), MS03 no entrega la población objetivo:** el sistema responde 503 e indica que la lista de pendientes no está disponible. El consolidado de [RF-MS04-06](#rf-ms04-06) sigue disponible.</li><li>**A4 (paso 2), rol sin permiso:** el sistema responde 403.</li><li>**A5 (paso 1), evento inexistente:** el sistema responde 404.</li></ul> |
| **Reglas De Negocio** | <ul><li><a id="rn-ms04-15"></a>**RN-MS04-15:** La población objetivo son los destinatarios de la alerta según MS03. En un simulacro corresponde a los participantes que define MS06. Las personas pendientes son la población objetivo menos los usuarios con estado vigente. Quien reportó cualquiera de los cuatro estados, incluido No estoy en el campus, deja de ser pendiente.</li><li>Aplica [RN-MS04-14](#rn-ms04-14).</li></ul> |
| **Postcondiciones** | El usuario conoce quiénes no han reportado. No se modifica ningún dato. |
| **Requerimientos asociados** | [RF-MS04-06](#rf-ms04-06), [RF-MS04-10](#rf-ms04-10). Depende de MS03 (y de MS06 para los participantes de un simulacro). |
| **Criterios de aceptación** | <ul><li>Un destinatario sin check-in aparece como pendiente. Al reportar cualquiera de los cuatro estados, deja de aparecer.</li><li>El total de pendientes más el total de reportados de la población objetivo es igual a la población objetivo.</li><li>Si MS03 no responde, la consulta devuelve 503 y la consolidación sigue funcionando.</li><li>Un rol sin `ms04:dashboard:read` recibe 403.</li></ul> |
| **Prioridad** | Alta |
| **Estado / Squad responsable** | Pendiente / kAiRoS |

<br>

<a id="rf-ms04-08"></a>
### RF-MS04-08 Consultar personas que requieren asistencia

| Atributo | Detalle |
|---|---|
| **ID** | RF-MS04-08 |
| **Nombre** | Consultar personas que requieren asistencia |
| **Épica / Grupo funcional** | MS04 - Asistencia |
| **Descripción** | Permitir que el brigadista y el coordinador identifiquen a las personas que reportaron Requiero asistencia, con su ubicación y la solicitud generada en MS05, para que el coordinador la asigne y el brigadista la atienda en MS05. |
| **Actor(es)** | BRIGADISTA (sus zonas asignadas), COORDINADOR (todas las solicitudes del evento).<br>Solo lectura: ORGANIZADOR (seguimiento del simulacro), AUDITOR. |
| **Precondiciones** | Usuario autenticado con `ms04:dashboard:read`. El evento existe (activo o finalizado). |
| **Flujo base** | <ol><li>El usuario autorizado solicita las personas que requieren asistencia en un evento.</li><li>El sistema valida su permiso (X-User-Role).</li><li>Obtiene los check-ins cuyo estado vigente es Requiero asistencia.</li><li>Responde, con paginación y ordenado del reporte más antiguo al más reciente, el id del usuario, la ubicación, la fecha/hora del reporte, el id de la solicitud en MS05 y el estado de su envío (ENVIADA, PENDIENTE_ENVIO o RECHAZADA).</li></ol> |
| **Flujos alternos / excepciones** | <ul><li>**A1 (paso 3), nadie requiere asistencia:** el sistema responde con lista vacía.</li><li>**A2 (paso 3), el solicitante es BRIGADISTA:** solo recibe las personas cuya ubicación está en sus zonas asignadas.</li><li>**A3 (paso 4), una solicitud está PENDIENTE_ENVIO o RECHAZADA:** se muestra así, para que el coordinador la registre manualmente en MS05 (`ms05:assistance:register`).</li><li>**A4 (paso 2), rol sin permiso:** el sistema responde 403.</li><li>**A5 (paso 1), evento inexistente:** el sistema responde 404.</li></ul> |
| **Reglas De Negocio** | <ul><li><a id="rn-ms04-16"></a>**RN-MS04-16:** MS04 muestra quién requiere asistencia y dónde, pero no asigna ni cambia el estado de atención de las solicitudes: eso lo hacen el coordinador y el brigadista en MS05. Si la persona cambia a otro estado, deja de aparecer en esta lista y su solicitud sigue su curso en MS05.</li><li>Aplican [RN-MS04-10](#rn-ms04-10) y [RN-MS04-14](#rn-ms04-14).</li></ul> |
| **Postcondiciones** | El brigadista y el coordinador conocen a las personas que requieren asistencia. No se modifica ningún dato. |
| **Requerimientos asociados** | [RF-MS04-04](#rf-ms04-04), [RF-MS04-06](#rf-ms04-06). Depende de MS05. |
| **Criterios de aceptación** | <ul><li>Una persona en Requiero asistencia aparece con su ubicación, la fecha/hora del reporte y el id de su solicitud en MS05.</li><li>Una solicitud que no llegó a MS05 aparece como PENDIENTE_ENVIO o RECHAZADA.</li><li>Un BRIGADISTA solo ve las personas de sus zonas asignadas. El COORDINADOR ve todas las del evento.</li><li>Una persona que cambia a Estoy a salvo deja de aparecer en la lista.</li><li>Un rol sin `ms04:dashboard:read` recibe 403.</li></ul> |
| **Prioridad** | Alta |
| **Estado / Squad responsable** | Pendiente / kAiRoS |

<br>

<a id="rf-ms04-09"></a>
### RF-MS04-09 Conservar eventos finalizados y consultar su histórico

| Atributo | Detalle |
|---|---|
| **ID** | RF-MS04-09 |
| **Nombre** | Conservar eventos finalizados y consultar su histórico |
| **Épica / Grupo funcional** | MS04 - Histórico |
| **Descripción** | Cuando un evento termina, MS04 lo marca como no vigente, bloquea nuevos check-ins y conserva sus datos. Los roles autorizados consultan el histórico de check-ins del evento. |
| **Actor(es)** | Finalizan el evento (en MS03 o desde MS06): COORDINADOR u ORGANIZADOR.<br>Procesa el cierre: sistema MS04.<br>Consultan el histórico: COORDINADOR, ORGANIZADOR, AUDITOR (`ms04:history:read`). |
| **Precondiciones** | Para el cierre: el evento estaba activo y MS03 registra su finalización. Para la consulta: el evento existe (activo o finalizado). |
| **Flujo base** | <ol><li>El coordinador u organizador finaliza la alerta en MS03 (directamente o al terminar el simulacro en MS06).</li><li>En la siguiente operación sobre el evento, MS04 consulta su estado a MS03 y detecta que está Finalizada o Archivada.</li><li>MS04 marca el evento como no vigente y guarda la fecha/hora de cierre.</li><li>MS04 rechaza desde ese momento todo registro o actualización de check-in en el evento ([RN-MS04-05](#rn-ms04-05)).</li><li>Se conservan los estados vigentes, el histórico y las solicitudes asociadas.</li><li>El usuario autorizado consulta, con paginación y filtro opcional por usuario o estado, el histórico de check-ins del evento en orden cronológico.</li></ol> |
| **Flujos alternos / excepciones** | <ul><li>**A1 (paso 3), el evento ya está marcado como no vigente:** no se repite la operación.</li><li>**A2 (paso 3), falla el marcado en MS04:** el error queda en el log, la operación en curso responde 500 sin detalles internos y no aplica ningún cambio. El marcado se reintenta en la siguiente operación sobre el evento.</li><li>**A3 (paso 6), rol sin permiso:** el sistema responde 403.</li><li>**A4 (paso 6), evento inexistente:** el sistema responde 404.</li></ul> |
| **Reglas De Negocio** | <ul><li><a id="rn-ms04-17"></a>**RN-MS04-17:** Los datos de un evento no vigente (estados vigentes, histórico y solicitudes asociadas) no se eliminan ni se modifican, y permanecen disponibles para consulta y análisis posterior.</li><li>Aplican [RN-MS04-05](#rn-ms04-05) y [RN-MS04-08](#rn-ms04-08).</li></ul> |
| **Postcondiciones** | El evento queda no vigente con todos sus datos almacenados y disponibles. |
| **Requerimientos asociados** | [RF-MS04-02](#rf-ms04-02), [RF-MS04-03](#rf-ms04-03), [RF-MS04-06](#rf-ms04-06), [RF-MS04-10](#rf-ms04-10). Depende de MS03 (y de MS06 para simulacros). |
| **Criterios de aceptación** | <ul><li>Tras finalizar la alerta en MS03, el evento aparece como no vigente en MS04.</li><li>Un registro o actualización de check-in en un evento no vigente devuelve 409.</li><li>El histórico de un evento no vigente se entrega completo e idéntico al del cierre.</li><li>Un rol sin `ms04:history:read` (p. ej. ESTUDIANTE o BRIGADISTA) recibe 403 al consultar el histórico.</li></ul> |
| **Prioridad** | Media |
| **Estado / Squad responsable** | Pendiente / kAiRoS |

<br>

<a id="rf-ms04-10"></a>
### RF-MS04-10 Entregar resultados del evento

| Atributo | Detalle |
|---|---|
| **ID** | RF-MS04-10 |
| **Nombre** | Entregar resultados del evento |
| **Épica / Grupo funcional** | MS04 - Histórico |
| **Descripción** | Entregar los resultados de check-in de un evento para que el organizador consulte los resultados del simulacro, el auditor revise los históricos, MS06 evalúe el simulacro y genere su informe, y MS08 calcule los indicadores. |
| **Actor(es)** | ORGANIZADOR (resultados del simulacro), COORDINADOR, AUDITOR (`ms04:history:read`).<br>Sistemas consumidores: MS06 (evaluación e informe del simulacro) y MS08 (indicadores). |
| **Precondiciones** | El evento existe. Para resultados finales, el evento está no vigente ([RF-MS04-09](#rf-ms04-09)). |
| **Flujo base** | <ol><li>El usuario autorizado o el servicio consumidor solicita los resultados de un evento.</li><li>El sistema valida el permiso (X-User-Role).</li><li>Reúne los resultados: población objetivo, total que reportó y total de pendientes ([RF-MS04-07](#rf-ms04-07)), total por cada uno de los cuatro estados y por punto de encuentro, personas que reportaron Requiero asistencia al menos una vez (con el id de su solicitud) y, por cada usuario, la fecha/hora de su primer check-in.</li><li>Responde con los resultados, la fecha de inicio y de cierre del evento, e indica si son finales (evento no vigente) o parciales (evento activo).</li></ol> |
| **Flujos alternos / excepciones** | <ul><li>**A1 (paso 4), evento activo:** los resultados se entregan marcados como parciales.</li><li>**A2 (paso 3), MS03 no entrega la población objetivo:** se entregan los demás resultados e indica que la población objetivo y los pendientes no están disponibles.</li><li>**A3 (paso 2), rol sin permiso:** el sistema responde 403.</li><li>**A4 (paso 1), evento inexistente:** el sistema responde 404.</li></ul> |
| **Reglas De Negocio** | <ul><li><a id="rn-ms04-18"></a>**RN-MS04-18:** MS04 entrega los datos de check-in del evento, pero no calcula indicadores ni compara simulacros: la evaluación y el informe del simulacro son de MS06, y los indicadores (p. ej. tiempo promedio hasta el check-in) son de MS08.</li><li>Aplican [RN-MS04-13](#rn-ms04-13), [RN-MS04-15](#rn-ms04-15) y [RN-MS04-17](#rn-ms04-17).</li></ul> |
| **Postcondiciones** | El organizador, el auditor, MS06 y MS08 disponen de los resultados del evento. No se modifica ningún dato. |
| **Requerimientos asociados** | [RF-MS04-06](#rf-ms04-06), [RF-MS04-07](#rf-ms04-07), [RF-MS04-09](#rf-ms04-09). Consumido por MS06 y MS08. |
| **Criterios de aceptación** | <ul><li>Los resultados de un evento finalizado incluyen los cuatro estados, los puntos de encuentro, los pendientes, las personas que pidieron asistencia y la fecha/hora del primer check-in de cada usuario.</li><li>Los totales coinciden con los del consolidado al cierre.</li><li>Los resultados de un evento activo se marcan como parciales.</li><li>Un rol sin `ms04:history:read` recibe 403.</li></ul> |
| **Prioridad** | Media |
| **Estado / Squad responsable** | Pendiente / kAiRoS (coordinar contrato con MS06 y MS08) |

---

<a id="requerimientos-no-funcionales-rnf"></a>
## Requerimientos No Funcionales (RNF)

| ID | Requerimiento |
|---|---|
| **RNF-MS04-01** | Crear o actualizar un check-in (incluidas las validaciones con MS02 y MS03) responde en menos de 5 segundos para al menos el 80 % de las peticiones con carga normal, para que reportar el estado sea rápido durante una emergencia. Las peticiones que superan 5 segundos quedan en el log con su Correlation ID. Se verifica midiendo el tiempo entre el envío y la respuesta con usuarios de distintos roles. |
| **RNF-MS04-02** | MS04 soporta 500 check-ins simultáneos (la institución tiene alrededor de 5000 personas) sin pérdida de datos, sin duplicados y sin errores por saturación, y cada usuario conserva un único estado vigente. Por encima de 500 sigue respondiendo, aunque más lento. Se verifica con una prueba de carga de 500 usuarios simulados. |
| **RNF-MS04-03** | El registro de check-ins no depende de MS05: con MS05 detenido, un check-in Requiero asistencia se guarda y responde con éxito, la solicitud queda pendiente y se envía cuando MS05 se recupera ([RN-MS04-12](#rn-ms04-12)). Cada intento fallido queda en el log. |
| **RNF-MS04-04** | La unicidad del estado vigente por usuario y evento ([RN-MS04-07](#rn-ms04-07)) se garantiza con una restricción en la base de datos, no solo en la lógica de la aplicación. Una escritura concurrente rechazada por la restricción responde 409 `CONFLICTO`, nunca 500. |
| **RNF-MS04-05** | Tras marcar un evento como no vigente, ningún rol puede crear, modificar ni eliminar sus check-ins ni su histórico ([RN-MS04-17](#rn-ms04-17)). Los intentos responden 409 `CONFLICTO` y quedan auditados. MS04 no expone operaciones de eliminación. |
| **RNF-MS04-06** | Un check-in registrado se refleja en la consolidación, los pendientes y la lista de asistencia ([RF-MS04-06](#rf-ms04-06), [RF-MS04-07](#rf-ms04-07), [RF-MS04-08](#rf-ms04-08)) en menos de 5 segundos, con totales que cumplen [RN-MS04-13](#rn-ms04-13). |
| **RNF-MS04-07** | El contrato de MS04 permite que el usuario complete el reporte de su estado en máximo 3 interacciones desde la pantalla inicial, incluida la selección del punto de encuentro en Estoy a salvo. Los cuatro estados se muestran siempre con su texto (Estoy a salvo, Me estoy evacuando, Requiero asistencia, No estoy en el campus). Si no hay evento activo, la pantalla inicial lo indica sin pasos adicionales. Se verifica con una prueba de usabilidad con usuarios de distintos roles. |
| **RNF-MS04-08** | Aplicación Spring Boot independiente (Java, Maven) con Spring Data JPA y base de datos PostgreSQL propia. Ningún otro servicio accede a ella. |
| **RNF-MS04-09** | Todos los endpoints bajo `/api/v1/`, documentados con OpenAPI/Swagger y publicados antes de que otros squads dependan de ellos. Cada operación declara `x-permiso` y `x-roles` iguales a la Matriz Rol‐Permiso y documenta las cabeceras de identidad como parámetros, sin `bearerAuth`. |
| **RNF-MS04-10** | Todo acceso a MS04 llega a través del API Gateway. MS04 recibe la identidad en X-User-Id, X-User-Role y X-Correlation-ID y, si faltan las de identidad, responde 401 `NO_AUTENTICADO`. Sus llamadas salientes (MS02, MS03, MS05 y MS08) también pasan por el Gateway con su clave de servicio, el mismo Correlation ID y la identidad del usuario propagada, con timeout de cliente de 2000 ms. MS04 no configura CORS. Puerto interno 8084. |
| **RNF-MS04-11** | Todo error, incluidos los del framework (404, 405, 415, cabecera faltante), responde con el `ErrorResponse` y el catálogo de códigos de los Estándares, sin exponer trazas internas. La indisponibilidad de MS02, MS03 o MS05 responde 503 con un código propio declarado en el OpenAPI de MS04. Nunca se reenvía tal cual el cuerpo de error de otro servicio. |
| **RNF-MS04-12** | Logs útiles para diagnóstico que incluyen el Correlation ID en cada línea, sin tokens ni datos personales. Spring Boot Actuator habilitado. |
| **RNF-MS04-13** | Secretos (credenciales de BD y clave de servicio del Gateway) fuera del repositorio, mediante variables de entorno. |
| **RNF-MS04-14** | Pruebas unitarias y de integración de los flujos de evento activo, registro y actualización de check-in en los cuatro estados, unicidad del estado vigente, solicitud de asistencia (incluida la caída de MS05), consolidación, pendientes, lista de asistencia, cierre del evento, resultados, lectura de cabeceras de identidad y emisión de eventos de auditoría a MS08. Análisis de calidad con SonarCloud. |
| **RNF-MS04-15** | Ejecución en contenedor Docker, con pipeline en GitHub Actions. Código integrado por Pull Request con revisión. |
| **RNF-MS04-16** | Entradas validadas en el servidor contra datos inválidos. Los listados usan la paginación de los Estándares. |
| **RNF-MS04-17** | La auditoría se delega en [ms08-analytics-and-audit](https://github.com/CampusSeguro-DOSW-2026-2/ms08-analytics-and-audit) (MS08), que almacena y consulta los eventos (el AUDITOR consulta allí los registros). MS04 los emite de forma asíncrona con el formato de los Estándares (`modulo` = MS04; acciones REGISTRO_CHECKIN, CAMBIO_ESTADO_CHECKIN, SOLICITUD_ASISTENCIA; resultado EXITO o FALLO; `recurso_afectado` = `checkin:<id>`, o `evento:<id>` si el check-in no llegó a crearse) con el Correlation ID y sin datos personales distintos de los ids. Si MS08 falla, la respuesta al cliente no cambia y el error se registra en logs. MS04 no tiene tablas ni endpoints de auditoría: el histórico de check-ins es un dato del dominio, no auditoría. |

---

<a id="dependencias-y-decisiones-abiertas"></a>
## Dependencias y decisiones abiertas

MS04 depende de contratos que aún no están publicados. Se acuerdan con cada squad y se registran en los Estándares antes de implementarlos.

| # | Tema | Responsables | Estado |
|---|---|---|---|
| 1 | Endpoint de MS03 para consultar las alertas Activas de las que un usuario es destinatario y el estado de una alerta. Registrar la llamada interna MS04 → MS03. | MS03 + MS04 | Pendiente |
| 2 | Endpoint de MS03 que entregue los destinatarios de una alerta (población objetivo). En simulacros, confirmar con MS06 que los destinatarios corresponden a sus participantes. | MS03 + MS06 + MS04 | Pendiente |
| 3 | Registrar la llamada MS04 → MS02 para validar puntos de encuentro (`ms02:campus:read`, identidad propagada). | MS02 + MS04 | Pendiente |
| 4 | Endpoint interno de MS05 para las solicitudes que vienen de MS04 (decisión abierta n.º 3 de los Estándares): campos, idempotencia por check-in y cómo sabe MS04 si una solicitud sigue abierta. | MS04 + MS05 | Pendiente |
| 5 | Endpoint de MS05 con las zonas asignadas a un brigadista, para limitar su consolidación y su lista de asistencia (MR-03 de la matriz). | MS04 + MS05 | Pendiente |
| 6 | Nombres de usuarios en las listas de pendientes, de personas por estado y de asistencia. Hoy MS04 solo devuelve ids (decisión abierta n.º 2 de los Estándares). | MS01 + MS04 | Pendiente |
| 7 | Consumo de los resultados del evento ([RF-MS04-10](#rf-ms04-10)) por MS06 y MS08: registrar las llamadas MS06 → MS04 y MS08 → MS04 y definir si la ruta es protegida (con la identidad del usuario propagada) o interna (por `X-Service-Name`). | MS04 + MS06 + MS08 | Pendiente |

---

<a id="api-ms04-evacuation-and-check-in"></a>
## API: ms04-evacuation-and-check-in
Reporte del estado de las personas y consolidación por punto de encuentro.

Contrato OpenAPI: [`api/swagger.yml`](api/swagger.yml). Todos los endpoints son protegidos y reciben la identidad por cabeceras ([RN-MS04-06](#rn-ms04-06)). La columna *Permiso* coincide con el `x-permiso` del contrato y con la [Matriz Rol‐Permiso](https://github.com/CampusSeguro-DOSW-2026-2/documentation/wiki/Matriz-Rol%E2%80%90Permiso). COM = ESTUDIANTE, PROFESOR, ADMINISTRATIVO, BRIGADISTA, COORDINADOR, ORGANIZADOR.

| Categoría | Método | Ruta (/api/v1) | Acceso | Permiso | RF asociado |
|---|---|---|---|---|---|
| **Evento activo** | GET | `/events/active` | COM | `ms04:checkin:read_own` | [01](#rf-ms04-01) |
| **Check-in propio** | POST | `/events/{eventId}/checkins` | COM | `ms04:checkin:write`, `ms04:assistance:request` (si es Requiero asistencia) | [02](#rf-ms04-02), [03](#rf-ms04-03), [04](#rf-ms04-04) |
| | GET | `/events/{eventId}/checkins/me` | COM | `ms04:checkin:read_own` | [05](#rf-ms04-05) |
| **Consolidación** | GET | `/events/{eventId}/consolidation` | BRIGADISTA (sus zonas), COORDINADOR, ORGANIZADOR, AUDITOR | `ms04:dashboard:read` | [06](#rf-ms04-06) |
| | GET | `/events/{eventId}/checkins` | BRIGADISTA (sus zonas), COORDINADOR, ORGANIZADOR, AUDITOR | `ms04:dashboard:read` | [06](#rf-ms04-06) |
| | GET | `/events/{eventId}/pending` | BRIGADISTA (sus zonas), COORDINADOR, ORGANIZADOR, AUDITOR | `ms04:dashboard:read` | [07](#rf-ms04-07) |
| **Asistencia** | GET | `/events/{eventId}/assistance` | BRIGADISTA (sus zonas), COORDINADOR, ORGANIZADOR, AUDITOR | `ms04:dashboard:read` | [08](#rf-ms04-08) |
| **Histórico y resultados** | GET | `/events/{eventId}/history` | COORDINADOR, ORGANIZADOR, AUDITOR | `ms04:history:read` | [09](#rf-ms04-09) |
| | GET | `/events/{eventId}/results` | COORDINADOR, ORGANIZADOR, AUDITOR (y MS06, MS08) | `ms04:history:read` | [10](#rf-ms04-10) |

`GET /events/{eventId}/checkins` devuelve los estados vigentes del evento con paginación y filtros opcionales por estado y punto de encuentro (p. ej. las personas en Estoy a salvo de un punto). La consulta de auditoría no pertenece a MS04: la expone ms08-analytics-and-audit (RNF-MS04-17).
