# Requerimientos — MS04 Evacuación y Check-in

Los valores transversales (cabeceras, errores, correlación, paginación, llamadas internas, auditoría) se rigen por los [Estándares de Integración](https://github.com/CampusSeguro-DOSW-2026-2/documentation/wiki/Est%C3%A1ndares-de-Integraci%C3%B3n-y-Contratos-Transversales) y los permisos por la [Matriz Rol‐Permiso](https://github.com/CampusSeguro-DOSW-2026-2/documentation/wiki/Matriz-Rol%E2%80%90Permiso). Este documento solo define el comportamiento de MS04.

## Índice de Contenido
1. [Criterios de Separación](#criterios-de-separacion)
2. [Estados del evento](#estados-del-evento)
3. [Estados del check-in](#estados-del-check-in)
4. [Requerimientos Funcionales](#requerimientos-funcionales)
   - [RF-MS04-01 Consultar eventos activos](#rf-ms04-01)
   - [RF-MS04-02 Registrar check-in](#rf-ms04-02)
   - [RF-MS04-03 Mantener y actualizar el estado vigente](#rf-ms04-03)
   - [RF-MS04-04 Generar solicitud de asistencia en MS05](#rf-ms04-04)
   - [RF-MS04-05 Consultar el propio estado vigente](#rf-ms04-05)
   - [RF-MS04-06 Consolidar el estado del evento (dashboard)](#rf-ms04-06)
   - [RF-MS04-07 Identificar personas pendientes](#rf-ms04-07)
   - [RF-MS04-08 Consultar personas que requieren asistencia](#rf-ms04-08)
   - [RF-MS04-09 Conservar eventos finalizados y consultar su histórico](#rf-ms04-09)
   - [RF-MS04-10 Entregar resultados del evento](#rf-ms04-10)
   - [RF-MS04-11 Recibir el estado del evento y habilitar o deshabilitar el check-in](#rf-ms04-11)
   - [RF-MS04-12 Entregar a MS06 el detalle de check-ins del evento](#rf-ms04-12)
   - [RF-MS04-13 Consultar eventos](#rf-ms04-13)
5. [Requerimientos No Funcionales (RNF)](#requerimientos-no-funcionales-rnf)
6. [Dependencias y decisiones abiertas](#dependencias-y-decisiones-abiertas)
7. [API: ms04-evacuation-and-check-in](#api-ms04-evacuation-and-check-in)

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

<a id="estados-del-evento"></a>
## Estados del evento

Un evento es un simulacro o una emergencia que define MS06. MS04 no crea eventos por su cuenta: los guarda cuando MS06 le informa su estado ([RF-MS04-11](#rf-ms04-11)) y es MS06 quien lo cambia. MS04 espera esas llamadas: nunca inicia ni termina un evento por reloj ni consulta a MS06 por su cuenta.

| Estado | Significado | Check-in | Lo que MS04 hace |
|---|---|---|---|
| `BORRADOR` | El evento no tiene fecha de inicio (nunca se programó o perdió la fecha). | Deshabilitado | Guarda el evento sin fecha. Si estaba `PROGRAMADO`, cancela esa programación. |
| `PROGRAMADO` | El evento tiene fecha de inicio. | Deshabilitado | Guarda cuándo (fecha de inicio) y dónde (escenario) se habilitará el check-in, y a quiénes (roles convocados). MS06 lo envía al programar y cada vez que reprograma. |
| `EN_EJECUCION` | El evento está en curso. | **Habilitado** | Acepta check-ins de los usuarios a quienes el evento les aplica. |
| `FINALIZADO` | El evento terminó. | Deshabilitado | Rechaza nuevos check-ins y conserva todos los datos del evento. |

* El check-in está habilitado si y solo si el evento está `EN_EJECUCION` ([RN-MS04-19](#rn-ms04-19)).
* Simulacros y eventos reales (`simulacro = false`) siguen los mismos estados: MS04 guarda el estado que envía MS06, dueño del ciclo de vida del evento, y el check-in solo se habilita con `EN_EJECUCION` ([RN-MS04-20](#rn-ms04-20)).
* Las consultas de un evento (consolidado, personas, pendientes, asistencia, histórico, resultados y el propio estado) funcionan en cualquier estado, esté o no habilitado el check-in ([RN-MS04-22](#rn-ms04-22)). Solo el registro y la actualización de check-ins exigen `EN_EJECUCION`. Los roles con `ms04:dashboard:read` también pueden consultar los eventos en sí, en cualquier estado ([RF-MS04-13](#rf-ms04-13)); el resto de roles solo ve los eventos en ejecución que le aplican ([RF-MS04-01](#rf-ms04-01)).
* Las transiciones permitidas están en [RN-MS04-21](#rn-ms04-21).

---

<a id="estados-del-check-in"></a>
## Estados del check-in

Con el check-in habilitado, el usuario reporta exactamente uno de estos cuatro estados ([RN-MS04-03](#rn-ms04-03)). El texto es el que ve el usuario. El código es el valor del contrato.

| Estado (texto al usuario) | Código | Datos obligatorios | Efecto en MS04 |
|---|---|---|---|
| **Estoy a salvo** | `SAFE` | Punto de encuentro (existente y ACTIVO en MS02) | Se guarda el punto de encuentro y la persona cuenta en la consolidación de ese punto. |
| **Me estoy evacuando** | `EVACUATING` | Ninguno | La persona cuenta como reportada y en evacuación. |
| **Requiero asistencia** | `NEEDS_ASSISTANCE` | Ubicación (edificio, espacio o punto de encuentro de MS02) | Se genera una solicitud de asistencia en MS05 ([RF-MS04-04](#rf-ms04-04)) y la persona aparece en la lista de asistencia ([RF-MS04-08](#rf-ms04-08)). |
| **No estoy en el campus** | `OFF_CAMPUS` | Ninguno | La persona cuenta como reportada y deja de ser pendiente, pero no se asocia a ningún punto de encuentro. |

* Los cuatro estados cuentan como “reportado”: ninguno deja a la persona en la lista de pendientes.
* Mientras el check-in esté habilitado, el usuario puede pasar de cualquier estado a cualquier otro ([RF-MS04-03](#rf-ms04-03)). Cada reporte es un check-in nuevo: el estado vigente del usuario es su check-in más reciente en el evento y los anteriores forman su histórico ([RN-MS04-07](#rn-ms04-07), [RN-MS04-08](#rn-ms04-08)).
* La consolidación siempre muestra los cuatro estados, con 0 cuando no hay personas en alguno.

---

<a id="requerimientos-funcionales"></a>
## Requerimientos Funcionales

<a id="rf-ms04-01"></a>
### RF-MS04-01 Consultar eventos activos

| Atributo | Detalle |
|---|---|
| **ID** | RF-MS04-01 |
| **Nombre** | Consultar eventos activos |
| **Épica / Grupo funcional** | MS04 - Evento activo |
| **Descripción** | Permitir que el usuario autenticado sepa si hay un evento activo (simulacro o emergencia con el check-in habilitado) que le aplique, para poder reportar su estado. |
| **Actor(es)** | ESTUDIANTE, PROFESOR, ADMINISTRATIVO, BRIGADISTA, COORDINADOR, ORGANIZADOR |
| **Precondiciones** | Usuario autenticado. Las peticiones llegan con las cabeceras de identidad (X-User-Id, X-User-Role, X-Correlation-ID). |
| **Flujo base** | <ol><li>El cliente solicita los eventos activos.</li><li>El sistema identifica al usuario (X-User-Id) y valida su permiso (X-User-Role).</li><li>Busca en la base de datos de MS04 los eventos `EN_EJECUCION` cuyos roles convocados incluyen el rol del usuario ([RN-MS04-01](#rn-ms04-01)).</li><li>Responde con los eventos activos que le aplican: id del evento, nombre, tipo (simulacro o emergencia), fecha de inicio y escenario.</li></ol> |
| **Flujos alternos / excepciones** | <ul><li>**A1 (paso 3), no hay evento activo que le aplique:** el sistema responde 200 con una lista vacía y el cliente muestra “No hay ningún evento activo”.</li><li>**A2 (paso 3), varios eventos activos le aplican:** el sistema los devuelve todos, ordenados por fecha de inicio.</li></ul> |
| **Reglas De Negocio** | <ul><li><a id="rn-ms04-01"></a>**RN-MS04-01:** Un evento es un simulacro o una emergencia que define MS06 y que MS04 conoce solo por [RF-MS04-11](#rf-ms04-11). Su id es el `eventoId` (UUID) que asigna MS06. Un evento le aplica a un usuario solo si el rol del usuario (X-User-Role) está entre los `rolesConvocados` del evento. El escenario (edificios y zonas) no restringe a quién le aplica ([RN-MS04-24](#rn-ms04-24)).</li><li><a id="rn-ms04-02"></a>**RN-MS04-02:** MS04 obtiene información de otros microservicios (MS01, MS02 y MS05) únicamente por sus API REST a través del API Gateway, nunca accediendo a sus bases de datos. Del usuario solo almacena su id (X-User-Id), sin duplicar datos personales.</li></ul> |
| **Postcondiciones** | El usuario conoce el evento activo o sabe que no hay ninguno. No se modifica ningún dato. |
| **Requerimientos asociados** | [RF-MS04-02](#rf-ms04-02), [RF-MS04-05](#rf-ms04-05), [RF-MS04-11](#rf-ms04-11), [RF-MS04-13](#rf-ms04-13). |
| **Criterios de aceptación** | <ul><li>Con un evento `EN_EJECUCION` cuyos roles convocados incluyen el rol del usuario, la consulta devuelve 200 con el id y los datos del evento.</li><li>Sin eventos `EN_EJECUCION` que le apliquen, la consulta devuelve 200 con una lista vacía.</li><li>Un evento `PROGRAMADO`, `BORRADOR` o `FINALIZADO`, o uno `EN_EJECUCION` que no convoca el rol del usuario, no aparece en su consulta.</li></ul> |
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
| **Descripción** | Permitir que el usuario reporte, durante un evento con el check-in habilitado, uno de los cuatro estados: Estoy a salvo, Me estoy evacuando, Requiero asistencia o No estoy en el campus. Si está a salvo, se guarda el punto de encuentro donde está. |
| **Actor(es)** | ESTUDIANTE, PROFESOR, ADMINISTRATIVO, BRIGADISTA, COORDINADOR, ORGANIZADOR |
| **Precondiciones** | Usuario autenticado. El evento existe, está `EN_EJECUCION` y le aplica ([RN-MS04-01](#rn-ms04-01)). Para Estoy a salvo, el punto de encuentro existe en MS02. |
| **Flujo base** | <ol><li>El cliente envía el estado y, según el estado, el punto de encuentro (Estoy a salvo) o la ubicación (Requiero asistencia).</li><li>El sistema identifica al usuario (X-User-Id) y valida su permiso (X-User-Role).</li><li>Valida que el estado sea uno de los cuatro del catálogo ([RN-MS04-03](#rn-ms04-03)) y que vengan los datos obligatorios para ese estado.</li><li>Busca el evento en la base de datos de MS04 y valida que exista, que el check-in esté habilitado (`EN_EJECUCION`) y que le aplique al rol del usuario.</li><li>Si el estado es Estoy a salvo, valida con MS02 que el punto de encuentro exista y esté ACTIVO.</li><li>En una sola operación, serializada por usuario y evento, busca el check-in más reciente del usuario en el evento ([RN-MS04-07](#rn-ms04-07)). Si no tiene ninguno, agrega el check-in con su fecha/hora de registro.</li><li>Emite a MS08 el evento de auditoría (acción REGISTRO_CHECKIN, resultado EXITO).</li><li>Si el estado es Requiero asistencia, ejecuta [RF-MS04-04](#rf-ms04-04).</li><li>Responde 201 con el id del check-in, el estado aceptado y la fecha/hora de registro.</li></ol> |
| **Flujos alternos / excepciones** | <ul><li>**A1 (paso 3), datos inválidos:** si el estado no es uno de los cuatro, falta el punto de encuentro en Estoy a salvo o falta la ubicación en Requiero asistencia, el sistema responde 400 con el detalle de los campos, no registra el check-in y emite a MS08 el evento de auditoría.</li><li>**A2 (paso 1), el cuerpo incluye un identificador de usuario:** el sistema rechaza la petición (400). La identidad solo llega por X-User-Id.</li><li>**A3 (paso 4), evento inexistente:** el sistema responde 404 y emite a MS08 el evento de auditoría.</li><li>**A4 (paso 4), el check-in del evento no está habilitado (evento `BORRADOR`, `PROGRAMADO` o `FINALIZADO`):** el sistema responde 409, indica que el check-in de este evento no está habilitado, no crea el check-in y emite a MS08 el evento de auditoría.</li><li>**A5 (paso 4), el evento no le aplica al usuario (su rol no está convocado):** el sistema responde 422, no crea el check-in y emite a MS08 el evento de auditoría.</li><li>**A6 (paso 5), punto de encuentro inexistente o inactivo:** el sistema responde 404 si no existe o 422 si está inactivo, no crea el check-in y emite a MS08 el evento de auditoría.</li><li>**A7 (paso 5), MS02 no responde:** el sistema responde 503 con un mensaje controlado, no crea el check-in y emite a MS08 el evento de auditoría. Solo aplica a Estoy a salvo, único estado que consulta a MS02.</li><li>**A8 (paso 6), el usuario ya tiene un check-in en el evento:** la petición se procesa como actualización ([RF-MS04-03](#rf-ms04-03)).</li></ul> |
| **Reglas De Negocio** | <ul><li><a id="rn-ms04-03"></a>**RN-MS04-03:** Los únicos estados permitidos para un check-in son Estoy a salvo (`SAFE`), Me estoy evacuando (`EVACUATING`), Requiero asistencia (`NEEDS_ASSISTANCE`) y No estoy en el campus (`OFF_CAMPUS`), con los datos obligatorios de la tabla [Estados del check-in](#estados-del-check-in).</li><li><a id="rn-ms04-04"></a>**RN-MS04-04:** Estoy a salvo exige un punto de encuentro existente y ACTIVO en MS02, que se almacena con el check-in. Los demás estados no se asocian a un punto de encuentro.</li><li><a id="rn-ms04-05"></a>**RN-MS04-05:** No se puede registrar ni actualizar un check-in si el evento no existe o si su check-in no está habilitado, es decir, si no está `EN_EJECUCION` ([RN-MS04-19](#rn-ms04-19)).</li><li><a id="rn-ms04-06"></a>**RN-MS04-06:** El usuario solo reporta y consulta su propio estado vigente. Se identifica únicamente por X-User-Id, nunca por el cuerpo de la petición.</li><li><a id="rn-ms04-07"></a>**RN-MS04-07:** Cada reporte de un usuario en un evento es un check-in nuevo: los check-ins solo se agregan, nunca se reemplazan. El estado vigente del usuario en el evento es su check-in más reciente. No existe ninguna otra tabla ni marca de “vigente”.</li></ul> |
| **Postcondiciones** | El check-in registrado es el estado vigente del usuario en el evento. Si fue Estoy a salvo, el punto de encuentro queda almacenado. |
| **Requerimientos asociados** | [RF-MS04-01](#rf-ms04-01), [RF-MS04-03](#rf-ms04-03), [RF-MS04-04](#rf-ms04-04), [RF-MS04-05](#rf-ms04-05), [RF-MS04-11](#rf-ms04-11). Depende de MS02. |
| **Criterios de aceptación** | <ul><li>Cada uno de los cuatro estados, con sus datos obligatorios, en un evento `EN_EJECUCION` que le aplica al usuario, devuelve 201 con el id del check-in, el estado aceptado y la fecha/hora.</li><li>Un estado distinto de los cuatro devuelve 400 y no crea el check-in.</li><li>Estoy a salvo sin punto de encuentro devuelve 400 y no crea el check-in.</li><li>Un check-in en un evento `PROGRAMADO`, `BORRADOR` o `FINALIZADO` devuelve 409 y no crea el check-in.</li><li>Un check-in en un evento que no existe devuelve 404.</li><li>Un check-in de un usuario cuyo rol no está convocado devuelve 422.</li><li>Un punto de encuentro inactivo devuelve 422 y no crea el check-in.</li><li>Cada intento, exitoso o fallido, genera un evento de auditoría hacia MS08.</li></ul> |
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
| **Descripción** | Permitir que el usuario cambie su estado entre los cuatro mientras el check-in del evento siga habilitado. Cada cambio se guarda como un check-in nuevo: el más reciente es el estado vigente y los anteriores forman el histórico. |
| **Actor(es)** | ESTUDIANTE, PROFESOR, ADMINISTRATIVO, BRIGADISTA, COORDINADOR, ORGANIZADOR |
| **Precondiciones** | El usuario ya tiene al menos un check-in en el evento. El evento sigue `EN_EJECUCION`. |
| **Flujo base** | <ol><li>El cliente envía el nuevo estado (y el punto de encuentro o la ubicación, según el estado).</li><li>El sistema valida el estado, los datos obligatorios, que el check-in del evento siga habilitado y, si es Estoy a salvo, el punto de encuentro (pasos 3 a 5 de [RF-MS04-02](#rf-ms04-02)).</li><li>En una sola operación, serializada por usuario y evento, compara con el check-in más reciente del usuario y, si hay un cambio, agrega un check-in nuevo. Los check-ins anteriores no se modifican.</li><li>Emite a MS08 el evento de auditoría (acción CAMBIO_ESTADO_CHECKIN, `recurso_afectado` = check-in, `detalle` con el estado anterior y el nuevo).</li><li>Si el nuevo estado es Requiero asistencia, ejecuta [RF-MS04-04](#rf-ms04-04).</li><li>Responde 200 con el id del nuevo check-in, el nuevo estado y su fecha/hora de registro.</li></ol> |
| **Flujos alternos / excepciones** | <ul><li>**A1 (paso 2), el check-in del evento no está habilitado (evento `FINALIZADO`, `PROGRAMADO` o `BORRADOR`):** el sistema responde 409, no agrega nada y emite a MS08 el evento de auditoría.</li><li>**A2 (paso 2), datos inválidos:** el sistema responde 400 con el detalle de los campos, no agrega nada y emite a MS08 el evento de auditoría. Aplica también al cambio a Estoy a salvo sin punto de encuentro.</li><li>**A3 (paso 3), mismo estado que el vigente (y, si es Estoy a salvo, mismo punto de encuentro):** el sistema responde 200 con el check-in vigente sin agregar uno nuevo, y emite a MS08 el evento de auditoría del intento.</li><li>**A4 (paso 3), Estoy a salvo con otro punto de encuentro:** se trata como un cambio y se agrega un check-in nuevo.</li><li>**A5 (paso 3), dos reportes simultáneos del mismo usuario en el mismo evento:** el sistema los atiende uno tras otro, en el orden en que llegan a la base de datos. Ambos se procesan y el estado vigente es el del último. Ninguno responde 409 ni 500.</li></ul> |
| **Reglas De Negocio** | <ul><li><a id="rn-ms04-08"></a>**RN-MS04-08:** El histórico del usuario en un evento son todos sus check-ins (estado, punto de encuentro o ubicación si aplica, y fecha/hora), en orden cronológico. Los check-ins no se modifican ni se eliminan, y el histórico solo lo consultan los roles con `ms04:history:read` ([RF-MS04-09](#rf-ms04-09)).</li><li>Aplican [RN-MS04-03](#rn-ms04-03), [RN-MS04-04](#rn-ms04-04), [RN-MS04-05](#rn-ms04-05) y [RN-MS04-07](#rn-ms04-07).</li></ul> |
| **Postcondiciones** | El check-in más reciente del usuario es su estado vigente y todos los anteriores siguen en el histórico. |
| **Requerimientos asociados** | [RF-MS04-02](#rf-ms04-02), [RF-MS04-04](#rf-ms04-04), [RF-MS04-05](#rf-ms04-05), [RF-MS04-09](#rf-ms04-09) |
| **Criterios de aceptación** | <ul><li>Un usuario en Me estoy evacuando que actualiza a Estoy a salvo con un punto de encuentro activo queda con Estoy a salvo como estado vigente (su check-in más reciente), y ambos check-ins quedan en el histórico en orden cronológico.</li><li>Un cambio de estado en un evento `FINALIZADO` devuelve 409 y no tiene efecto.</li><li>Repetir el mismo estado no crea un check-in nuevo.</li><li>Dos peticiones simultáneas del mismo usuario se procesan ambas, dejan como vigente la última y ninguna devuelve 409 ni 500.</li><li>Cada cambio, exitoso o fallido, genera un evento de auditoría hacia MS08.</li></ul> |
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
| **Flujo base** | <ol><li>El usuario reporta Requiero asistencia con su ubicación y MS04 agrega el check-in.</li><li>MS04 verifica que el usuario no tenga ya una solicitud de asistencia en el evento ([RN-MS04-11](#rn-ms04-11)).</li><li>MS04 envía a MS05, a través del API Gateway (ruta interna, con su clave de servicio y la identidad del usuario propagada), el id del evento, el id del usuario, la ubicación y el id del check-in Requiero asistencia que originó la solicitud.</li><li>MS05 crea la solicitud en estado Reportada y devuelve su id.</li><li>MS04 guarda el id de la solicitud asociado a ese check-in y marca su envío como ENVIADA.</li><li>Emite a MS08 el evento de auditoría (acción SOLICITUD_ASISTENCIA, resultado EXITO).</li><li>La respuesta del check-in indica que la solicitud fue enviada.</li></ol> |
| **Flujos alternos / excepciones** | <ul><li>**A1 (paso 1), sin ubicación:** se aplica el flujo A1 de [RF-MS04-02](#rf-ms04-02) (400, no se registra el check-in).</li><li>**A2 (paso 3), MS05 no disponible o sin respuesta dentro del timeout:** el check-in se conserva y la solicitud queda PENDIENTE_ENVIO. MS04 registra el fallo en su log, emite a MS08 el evento de auditoría (FALLO) y la respuesta del check-in indica que la solicitud está pendiente. MS04 reenvía las solicitudes pendientes cuando MS05 se recupera ([RN-MS04-12](#rn-ms04-12)).</li><li>**A3 (paso 2), el usuario ya tiene una solicitud en el evento (enviada o pendiente de envío):** no se crea otra y se reutiliza la existente.</li><li>**A4 (paso 4), MS05 rechaza la solicitud por datos inválidos:** el check-in se conserva, la solicitud queda RECHAZADA y no se reenvía. El error queda en el log, se emite a MS08 el evento de auditoría (FALLO) y la respuesta del check-in indica que la solicitud no pudo crearse. El coordinador la ve en [RF-MS04-08](#rf-ms04-08).</li><li>**A5, el usuario cambia de estado mientras la solicitud está pendiente de envío:** el nuevo check-in queda en el histórico y la solicitud pendiente se envía igual. Su atención o cierre se decide en MS05.</li></ul> |
| **Reglas De Negocio** | <ul><li><a id="rn-ms04-09"></a>**RN-MS04-09:** Toda solicitud de asistencia incluye la ubicación del usuario, expresada como un edificio, espacio o punto de encuentro de MS02 (tipo e id), según lo exige MS05.</li><li><a id="rn-ms04-10"></a>**RN-MS04-10:** MS04 conserva el id de la solicitud devuelto por MS05, asociado al check-in que la originó, y el estado de su envío: ENVIADA, PENDIENTE_ENVIO o RECHAZADA.</li><li><a id="rn-ms04-11"></a>**RN-MS04-11:** Un usuario no puede tener más de una solicitud de asistencia abierta por evento originada en MS04.</li><li><a id="rn-ms04-12"></a>**RN-MS04-12:** La indisponibilidad de MS05 no impide registrar el check-in. Las solicitudes que no se pudieron enviar se reintentan cuando MS05 se recupera, sin generar solicitudes duplicadas para el mismo check-in.</li></ul> |
| **Postcondiciones** | Existe una solicitud en MS05 y MS04 conserva su id vinculado al check-in, o la solicitud queda pendiente de envío o rechazada. |
| **Requerimientos asociados** | [RF-MS04-02](#rf-ms04-02), [RF-MS04-03](#rf-ms04-03), [RF-MS04-08](#rf-ms04-08). Depende de MS05. |
| **Criterios de aceptación** | <ul><li>Un Requiero asistencia con ubicación, en un evento `EN_EJECUCION`, crea la solicitud en MS05 y MS04 conserva su id asociado al check-in.</li><li>Un Requiero asistencia sin ubicación devuelve 400 y no registra el check-in.</li><li>Con MS05 detenido, el check-in se registra con éxito y la solicitud queda pendiente. Al recuperarse MS05, se envía una sola vez y se guarda su id.</li><li>Un segundo Requiero asistencia del mismo usuario en el mismo evento no crea otra solicitud.</li><li>Cada intento de envío, exitoso o fallido, genera un evento de auditoría hacia MS08.</li></ul> |
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
| **Precondiciones** | Usuario autenticado. El evento existe (en cualquier estado). |
| **Flujo base** | <ol><li>El usuario selecciona el evento.</li><li>El sistema identifica al usuario (X-User-Id) y obtiene su check-in más reciente en ese evento: su estado vigente.</li><li>Responde con el estado vigente, su fecha/hora de registro, el punto de encuentro (si es Estoy a salvo) y el estado de envío de la solicitud de asistencia (si es Requiero asistencia).</li></ol> |
| **Flujos alternos / excepciones** | <ul><li>**A1 (paso 2), el usuario no ha reportado estado en el evento:** el sistema responde 404 e indica que no tiene check-in en ese evento.</li><li>**A2 (paso 2), evento inexistente:** el sistema responde 404.</li></ul> |
| **Reglas De Negocio** | <ul><li>Aplican [RN-MS04-06](#rn-ms04-06), [RN-MS04-07](#rn-ms04-07) y [RN-MS04-08](#rn-ms04-08): el usuario ve solo su estado vigente (su check-in más reciente), no sus check-ins anteriores.</li></ul> |
| **Postcondiciones** | El usuario visualiza su estado vigente. No se modifica ningún dato. |
| **Requerimientos asociados** | [RF-MS04-01](#rf-ms04-01), [RF-MS04-02](#rf-ms04-02), [RF-MS04-03](#rf-ms04-03) |
| **Criterios de aceptación** | <ul><li>Un usuario con varios cambios en un evento recibe solo su último check-in como estado vigente.</li><li>La respuesta no incluye los check-ins anteriores.</li><li>Un usuario sin check-in en el evento recibe 404.</li><li>La consulta funciona también cuando el evento está `FINALIZADO`.</li><li>Un usuario no puede consultar el estado de otro mediante este flujo.</li></ul> |
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
| **Precondiciones** | Usuario autenticado con `ms04:dashboard:read`. El evento existe (en cualquier estado). |
| **Flujo base** | <ol><li>El usuario autorizado selecciona un evento.</li><li>El sistema valida su permiso (X-User-Role).</li><li>Consolida los estados vigentes (el check-in más reciente de cada usuario): total por cada uno de los cuatro estados, total de personas a salvo por punto de encuentro y total general de personas que reportaron.</li><li>Responde con el consolidado, el estado del evento y la fecha/hora en que se calculó.</li><li>El usuario puede consultar, con paginación, las personas de un estado o de un punto de encuentro (p. ej. las personas a salvo en un punto), con su nombre ([RN-MS04-25](#rn-ms04-25)).</li></ol> |
| **Flujos alternos / excepciones** | <ul><li>**A1 (paso 3), evento sin check-ins (p. ej. `PROGRAMADO`):** los cuatro totales son 0.</li><li>**A2 (paso 2), rol sin permiso:** el sistema responde 403.</li><li>**A3 (paso 1), evento inexistente:** el sistema responde 404.</li><li>**A4 (paso 3), el solicitante es BRIGADISTA:** el consolidado y las listas se limitan a sus zonas asignadas ([RN-MS04-14](#rn-ms04-14)). Me estoy evacuando y No estoy en el campus se muestran en 0, porque esas personas no tienen ubicación.</li></ul> |
| **Reglas De Negocio** | <ul><li><a id="rn-ms04-13"></a>**RN-MS04-13:** El consolidado siempre incluye los cuatro estados (0 si no hay personas). La suma de los cuatro es igual al número de usuarios con estado vigente en el evento (para el BRIGADISTA, al número de usuarios con estado vigente que puede ver). Solo las personas en Estoy a salvo se cuentan por punto de encuentro.</li><li><a id="rn-ms04-14"></a>**RN-MS04-14:** La consolidación, los pendientes y la lista de asistencia requieren `ms04:dashboard:read` (BRIGADISTA, COORDINADOR, ORGANIZADOR, AUDITOR) y son de solo lectura. El BRIGADISTA ve solo la información de sus zonas asignadas en MS05 (edificios, zonas o puntos de encuentro). La ubicación solo se conoce en dos casos: Estoy a salvo (punto de encuentro) y Requiero asistencia (ubicación). Por eso al BRIGADISTA solo se le muestran las personas en esos dos estados cuya ubicación está en sus zonas: las personas sin ubicación conocida (los pendientes y quienes están en Me estoy evacuando o No estoy en el campus) no pertenecen a ninguna zona y no se le muestran. El ADMIN no tiene permisos en MS04.</li><li>Aplica [RN-MS04-07](#rn-ms04-07).</li></ul> |
| **Postcondiciones** | El usuario dispone del consolidado. No se modifica ningún dato. |
| **Requerimientos asociados** | [RF-MS04-02](#rf-ms04-02), [RF-MS04-03](#rf-ms04-03), [RF-MS04-07](#rf-ms04-07), [RF-MS04-08](#rf-ms04-08), [RF-MS04-10](#rf-ms04-10). Depende de MS05 (zonas del brigadista) y de MS01 (nombres). |
| **Criterios de aceptación** | <ul><li>El consolidado muestra los cuatro estados y su suma coincide con el número de usuarios con estado vigente.</li><li>Las personas a salvo aparecen agrupadas por su punto de encuentro.</li><li>Un cambio de estado mueve a la persona de un total a otro sin contarla dos veces.</li><li>El consolidado de un evento `PROGRAMADO` devuelve los cuatro totales en 0, y el de uno `FINALIZADO` sigue disponible.</li><li>Un rol sin `ms04:dashboard:read` recibe 403.</li><li>Un BRIGADISTA solo ve información de sus zonas asignadas: las personas en Estoy a salvo y Requiero asistencia de esas zonas, con Me estoy evacuando y No estoy en el campus en 0.</li></ul> |
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
| **Actor(es)** | COORDINADOR, ORGANIZADOR, AUDITOR (solo lectura).<br>BRIGADISTA: no recibe pendientes ([RN-MS04-14](#rn-ms04-14)). |
| **Precondiciones** | Usuario autenticado con `ms04:dashboard:read`. El evento existe (en cualquier estado). |
| **Flujo base** | <ol><li>El usuario autorizado solicita los pendientes de un evento.</li><li>El sistema valida su permiso (X-User-Role).</li><li>Obtiene de MS01 la población objetivo: los usuarios ACTIVOS cuyo rol está entre los roles convocados del evento ([RN-MS04-15](#rn-ms04-15)).</li><li>Determina los pendientes: población objetivo sin estado vigente (sin ningún check-in en el evento).</li><li>Responde, con paginación, el total de pendientes y la lista de sus ids de usuario con su nombre ([RN-MS04-25](#rn-ms04-25)).</li></ol> |
| **Flujos alternos / excepciones** | <ul><li>**A1 (paso 4), evento sin check-ins:** toda la población objetivo aparece como pendiente.</li><li>**A2 (paso 4), todos reportaron:** el sistema responde con total 0 y lista vacía.</li><li>**A3 (paso 3), MS01 no entrega la población objetivo:** el sistema responde 503 e indica que la lista de pendientes no está disponible. El consolidado de [RF-MS04-06](#rf-ms04-06) sigue disponible.</li><li>**A4 (paso 2), rol sin permiso:** el sistema responde 403.</li><li>**A5 (paso 1), evento inexistente:** el sistema responde 404.</li><li>**A6 (paso 4), el solicitante es BRIGADISTA:** los pendientes no tienen ubicación conocida, así que ninguno pertenece a sus zonas: el sistema responde 200 con total 0 y lista vacía ([RN-MS04-14](#rn-ms04-14)).</li></ul> |
| **Reglas De Negocio** | <ul><li><a id="rn-ms04-15"></a>**RN-MS04-15:** La población objetivo son los usuarios ACTIVOS de MS01 cuyo rol está entre los `rolesConvocados` del evento (una persona con varios roles cuenta una sola vez). MS04 no filtra por escenario porque no conoce la ubicación de las personas. Las personas pendientes son la población objetivo menos los usuarios con estado vigente. Quien reportó cualquiera de los cuatro estados, incluido No estoy en el campus, deja de ser pendiente. La población se calcula con los usuarios creados antes de la fecha de inicio del evento (o antes del momento de la consulta, si es anterior), y con su estado actual en MS01.</li><li>Aplica [RN-MS04-14](#rn-ms04-14).</li></ul> |
| **Postcondiciones** | El usuario conoce quiénes no han reportado. No se modifica ningún dato. |
| **Requerimientos asociados** | [RF-MS04-06](#rf-ms04-06), [RF-MS04-10](#rf-ms04-10), [RF-MS04-11](#rf-ms04-11). Depende de MS01. |
| **Criterios de aceptación** | <ul><li>Un usuario ACTIVO con un rol convocado y sin check-in aparece como pendiente. Al reportar cualquiera de los cuatro estados, deja de aparecer.</li><li>Un usuario con un rol no convocado no aparece como pendiente.</li><li>El total de pendientes más el total de reportados de la población objetivo es igual a la población objetivo.</li><li>Si MS01 no responde, la consulta devuelve 503 y la consolidación sigue funcionando.</li><li>Un BRIGADISTA recibe 200 con total 0 y lista vacía.</li><li>Un rol sin `ms04:dashboard:read` recibe 403.</li></ul> |
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
| **Precondiciones** | Usuario autenticado con `ms04:dashboard:read`. El evento existe (en cualquier estado). |
| **Flujo base** | <ol><li>El usuario autorizado solicita las personas que requieren asistencia en un evento.</li><li>El sistema valida su permiso (X-User-Role).</li><li>Obtiene las personas cuyo estado vigente es Requiero asistencia (su check-in más reciente).</li><li>Responde, con paginación y ordenado del reporte más antiguo al más reciente, el id del usuario, su nombre ([RN-MS04-25](#rn-ms04-25)), la ubicación, la fecha/hora de su check-in vigente, el id de la solicitud en MS05 y el estado de su envío (ENVIADA, PENDIENTE_ENVIO o RECHAZADA).</li></ol> |
| **Flujos alternos / excepciones** | <ul><li>**A1 (paso 3), nadie requiere asistencia:** el sistema responde con lista vacía.</li><li>**A2 (paso 3), el solicitante es BRIGADISTA:** solo recibe las personas cuya ubicación está en sus zonas asignadas.</li><li>**A3 (paso 4), una solicitud está PENDIENTE_ENVIO o RECHAZADA:** se muestra así, para que el coordinador la registre manualmente en MS05 (`ms05:assistance:register`).</li><li>**A4 (paso 2), rol sin permiso:** el sistema responde 403.</li><li>**A5 (paso 1), evento inexistente:** el sistema responde 404.</li></ul> |
| **Reglas De Negocio** | <ul><li><a id="rn-ms04-16"></a>**RN-MS04-16:** MS04 muestra quién requiere asistencia y dónde, pero no asigna ni cambia el estado de atención de las solicitudes: eso lo hacen el coordinador y el brigadista en MS05. Si la persona cambia a otro estado, deja de aparecer en esta lista y su solicitud sigue su curso en MS05.</li><li>Aplican [RN-MS04-10](#rn-ms04-10) y [RN-MS04-14](#rn-ms04-14).</li></ul> |
| **Postcondiciones** | El brigadista y el coordinador conocen a las personas que requieren asistencia. No se modifica ningún dato. |
| **Requerimientos asociados** | [RF-MS04-04](#rf-ms04-04), [RF-MS04-06](#rf-ms04-06). Depende de MS05 y de MS01 (nombres). |
| **Criterios de aceptación** | <ul><li>Una persona en Requiero asistencia aparece con su ubicación, la fecha/hora de su check-in vigente y el id de su solicitud en MS05.</li><li>Una solicitud que no llegó a MS05 aparece como PENDIENTE_ENVIO o RECHAZADA.</li><li>Un BRIGADISTA solo ve las personas de sus zonas asignadas. El COORDINADOR ve todas las del evento.</li><li>Una persona que cambia a Estoy a salvo deja de aparecer en la lista.</li><li>Un rol sin `ms04:dashboard:read` recibe 403.</li></ul> |
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
| **Descripción** | Cuando MS06 informa que un evento terminó, MS04 lo marca `FINALIZADO`, deshabilita el check-in y conserva sus datos. Los roles autorizados consultan el histórico de check-ins del evento: todos los reportes de cada usuario, en orden cronológico. |
| **Actor(es)** | Finalizan el evento en MS06: COORDINADOR u ORGANIZADOR.<br>Procesa el cierre: sistema MS04 ([RF-MS04-11](#rf-ms04-11)).<br>Consultan el histórico: COORDINADOR, ORGANIZADOR, AUDITOR (`ms04:history:read`). |
| **Precondiciones** | Para el cierre: el evento estaba `EN_EJECUCION` y MS06 informa su finalización. Para la consulta: el evento existe (en cualquier estado). |
| **Flujo base** | <ol><li>El coordinador u organizador finaliza el evento en MS06.</li><li>MS06 informa a MS04 el estado `FINALIZADO` ([RF-MS04-11](#rf-ms04-11)).</li><li>MS04 marca el evento `FINALIZADO` y guarda la fecha/hora de cierre (la de recepción).</li><li>MS04 rechaza desde ese momento todo registro o actualización de check-in en el evento ([RN-MS04-05](#rn-ms04-05)).</li><li>Se conservan todos los check-ins del evento y las solicitudes asociadas.</li><li>El usuario autorizado consulta, con paginación y filtro opcional por usuario o estado, el histórico de check-ins del evento en orden cronológico.</li></ol> |
| **Flujos alternos / excepciones** | <ul><li>**A1 (paso 3), el evento ya está `FINALIZADO`:** no se repite la operación ni cambia la fecha de cierre.</li><li>**A2 (paso 6), rol sin permiso:** el sistema responde 403.</li><li>**A3 (paso 6), evento inexistente:** el sistema responde 404.</li><li>**A4 (paso 6), el evento aún no tiene check-ins (o sigue en curso):** el sistema responde con lo que exista hasta el momento, que puede ser una lista vacía.</li></ul> |
| **Reglas De Negocio** | <ul><li><a id="rn-ms04-17"></a>**RN-MS04-17:** Los datos de un evento `FINALIZADO` (check-ins, histórico y solicitudes asociadas) no se eliminan ni se modifican, y permanecen disponibles para consulta y análisis posterior.</li><li>Aplican [RN-MS04-05](#rn-ms04-05) y [RN-MS04-08](#rn-ms04-08).</li></ul> |
| **Postcondiciones** | El evento queda `FINALIZADO` con todos sus datos almacenados y disponibles. |
| **Requerimientos asociados** | [RF-MS04-02](#rf-ms04-02), [RF-MS04-03](#rf-ms04-03), [RF-MS04-06](#rf-ms04-06), [RF-MS04-10](#rf-ms04-10), [RF-MS04-11](#rf-ms04-11). Depende de MS06. |
| **Criterios de aceptación** | <ul><li>Tras informar MS06 el estado `FINALIZADO`, el evento figura `FINALIZADO` en MS04.</li><li>Un registro o actualización de check-in en un evento `FINALIZADO` devuelve 409.</li><li>El histórico de un evento `FINALIZADO` se entrega completo e idéntico al del cierre.</li><li>Un rol sin `ms04:history:read` (p. ej. ESTUDIANTE o BRIGADISTA) recibe 403 al consultar el histórico.</li></ul> |
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
| **Precondiciones** | El evento existe. Para resultados finales, el evento está `FINALIZADO` ([RF-MS04-09](#rf-ms04-09)). |
| **Flujo base** | <ol><li>El usuario autorizado o el servicio consumidor solicita los resultados de un evento.</li><li>El sistema valida el permiso (X-User-Role).</li><li>Reúne los resultados: población objetivo, total que reportó y total de pendientes ([RF-MS04-07](#rf-ms04-07)), total por cada uno de los cuatro estados y por punto de encuentro (sobre los estados vigentes), personas que reportaron Requiero asistencia al menos una vez (con el id de su solicitud) y, por cada usuario, la fecha/hora de su primer check-in.</li><li>Responde con los resultados, la fecha de inicio y de cierre del evento, e indica si son finales (evento `FINALIZADO`) o parciales (cualquier otro estado).</li></ol> |
| **Flujos alternos / excepciones** | <ul><li>**A1 (paso 4), evento que no está `FINALIZADO`:** los resultados se entregan marcados como parciales.</li><li>**A2 (paso 3), MS01 no entrega la población objetivo:** se entregan los demás resultados e indica que la población objetivo y los pendientes no están disponibles.</li><li>**A3 (paso 2), rol sin permiso:** el sistema responde 403.</li><li>**A4 (paso 1), evento inexistente:** el sistema responde 404.</li></ul> |
| **Reglas De Negocio** | <ul><li><a id="rn-ms04-18"></a>**RN-MS04-18:** MS04 entrega los datos de check-in del evento, pero no calcula indicadores ni compara simulacros: la evaluación y el informe del simulacro son de MS06, y los indicadores (p. ej. tiempo promedio hasta el check-in) son de MS08.</li><li>Aplican [RN-MS04-13](#rn-ms04-13), [RN-MS04-15](#rn-ms04-15) y [RN-MS04-17](#rn-ms04-17).</li></ul> |
| **Postcondiciones** | El organizador, el auditor, MS06 y MS08 disponen de los resultados del evento. No se modifica ningún dato. |
| **Requerimientos asociados** | [RF-MS04-06](#rf-ms04-06), [RF-MS04-07](#rf-ms04-07), [RF-MS04-09](#rf-ms04-09), [RF-MS04-12](#rf-ms04-12). Consumido por MS06 y MS08. |
| **Criterios de aceptación** | <ul><li>Los resultados de un evento `FINALIZADO` incluyen los cuatro estados, los puntos de encuentro, los pendientes, las personas que pidieron asistencia y la fecha/hora del primer check-in de cada usuario.</li><li>Los totales coinciden con los del consolidado al cierre.</li><li>Los resultados de un evento que no está `FINALIZADO` se marcan como parciales.</li><li>Un rol sin `ms04:history:read` recibe 403.</li></ul> |
| **Prioridad** | Media |
| **Estado / Squad responsable** | Pendiente / kAiRoS (coordinar contrato con MS06 y MS08) |

<br>

<a id="rf-ms04-11"></a>
### RF-MS04-11 Recibir el estado del evento y habilitar o deshabilitar el check-in

| Atributo | Detalle |
|---|---|
| **ID** | RF-MS04-11 |
| **Nombre** | Recibir el estado del evento y habilitar o deshabilitar el check-in |
| **Épica / Grupo funcional** | MS04 - Evento |
| **Descripción** | Permitir que MS06 informe a MS04 el estado de un evento (programarlo, reprogramarlo, cancelar su programación, iniciarlo o finalizarlo) para que MS04 guarde cuándo, dónde y a quiénes aplica, y habilite o deshabilite el check-in. MS04 solo espera estas llamadas: no consulta a MS06 ni cambia el estado de un evento por su cuenta. |
| **Actor(es)** | Sistema MS06 (`ms06-drills`), por ruta interna, cuando el ORGANIZADOR o el COORDINADOR programa, reprograma, inicia o finaliza el evento en MS06. |
| **Precondiciones** | La llamada interna MS06 → MS04 está registrada en los Estándares y el API Gateway la expone como ruta interna con la clave de servicio de MS06. |
| **Flujo base** | <ol><li>MS06 envía el estado del evento (`PUT /api/v1/chi/eventos/{eventoId}/estado`) con `eventoId`, `nombre`, `simulacro`, `estado`, `fechaInicio`, `escenario` (`edificios` y `zonas`) y `rolesConvocados`.</li><li>El sistema identifica al servicio por `X-Service-Name` y verifica que sea `ms06-drills` ([RN-MS04-23](#rn-ms04-23)).</li><li>Valida el cuerpo: que `eventoId` sea igual al de la ruta, el nombre, el tipo (`simulacro`), el estado del catálogo, la fecha de inicio según el estado, el escenario y los roles convocados ([RN-MS04-24](#rn-ms04-24)).</li><li>Determina el estado efectivo: si `simulacro` es `false`, cualquier estado distinto de `FINALIZADO` deja el evento `EN_EJECUCION` ([RN-MS04-20](#rn-ms04-20)).</li><li>En una sola transacción, bloquea el evento (si ya existe), valida la transición ([RN-MS04-21](#rn-ms04-21)) y crea o actualiza el evento. Si el estado es `FINALIZADO`, guarda la fecha/hora de cierre.</li><li>El check-in queda habilitado si el estado efectivo es `EN_EJECUCION` y deshabilitado en cualquier otro ([RN-MS04-19](#rn-ms04-19)).</li><li>Emite a MS08 el evento de auditoría (acción CAMBIO_ESTADO_EVENTO, `recurso_afectado` = `evento:<id>`, `detalle` con el servicio de origen y los estados anterior y nuevo).</li><li>Responde 201 si el evento era nuevo o 200 si ya existía, con el evento almacenado: su estado efectivo y si el check-in quedó habilitado.</li></ol> |
| **Flujos alternos / excepciones** | <ul><li>**A1 (paso 3), cuerpo inválido:** `eventoId` distinto del de la ruta, estado fuera del catálogo, `PROGRAMADO` sin `fechaInicio`, `BORRADOR` con `fechaInicio`, fecha sin desfase horario, lista de roles vacía o con roles no permitidos. El sistema responde 400 con el detalle de los campos, no modifica nada y emite a MS08 el evento de auditoría.</li><li>**A2 (paso 2), no llega `X-Service-Name`:** el sistema responde 401 `NO_AUTENTICADO`.</li><li>**A3 (paso 2), la llamada viene de otro servicio:** el sistema responde 403 `PROHIBIDO`.</li><li>**A4 (paso 5), transición no permitida, o `simulacro` distinto del guardado:** el sistema responde 409 `CONFLICTO`, no modifica nada y emite a MS08 el evento de auditoría.</li><li>**A5 (paso 5), el evento ya está en ese estado:** si es `PROGRAMADO` (reprogramación) se actualizan la fecha de inicio, el nombre, el escenario y los roles convocados. Si es `BORRADOR` se actualizan el nombre, el escenario y los roles. Si es `EN_EJECUCION` o `FINALIZADO`, el sistema responde 200 con el evento almacenado sin cambiar nada (idempotente).</li><li>**A6 (paso 5), un evento `PROGRAMADO` pasa a `BORRADOR` porque MS06 le quitó la fecha:** se cancela la programación: se elimina la fecha de inicio guardada y el check-in sigue deshabilitado.</li><li>**A7, llega la fecha de inicio de un evento `PROGRAMADO`:** MS04 no hace nada. El evento sigue `PROGRAMADO` y el check-in deshabilitado hasta que MS06 envíe `EN_EJECUCION`.</li></ul> |
| **Reglas De Negocio** | <ul><li><a id="rn-ms04-19"></a>**RN-MS04-19:** El check-in de un evento está habilitado si y solo si su estado es `EN_EJECUCION`. `BORRADOR`, `PROGRAMADO` y `FINALIZADO` lo deshabilitan. Solo el registro y la actualización de check-ins dependen de esa habilitación ([RN-MS04-05](#rn-ms04-05)).</li><li><a id="rn-ms04-20"></a>**RN-MS04-20:** Un evento real (`simulacro = false`) sigue los mismos estados que un simulacro: MS04 guarda el estado que envía MS06, que es el dueño del ciclo de vida, y no lo cambia. Una emergencia queda `EN_EJECUCION` cuando MS06 la inicia. El valor de `simulacro` no cambia mientras el evento exista.</li><li><a id="rn-ms04-21"></a>**RN-MS04-21:** Un evento nuevo puede llegar en cualquier estado. Con el evento ya guardado, MS04 acepta solo estas transiciones (ver la tabla bajo este requerimiento): `BORRADOR` → `PROGRAMADO` o `EN_EJECUCION`; `PROGRAMADO` → `PROGRAMADO` (reprogramación), `BORRADOR` o `EN_EJECUCION`; `EN_EJECUCION` → `FINALIZADO`. Repetir el estado guardado es idempotente (200). Cualquier otra transición responde 409 `CONFLICTO`. `FINALIZADO` es definitivo.</li><li><a id="rn-ms04-22"></a>**RN-MS04-22:** MS04 guarda los eventos y permite consultarlos, junto con los check-ins de cada uno, esté o no habilitado el check-in: las consultas de [RF-MS04-05](#rf-ms04-05) a [RF-MS04-10](#rf-ms04-10), [RF-MS04-12](#rf-ms04-12) y [RF-MS04-13](#rf-ms04-13) funcionan en cualquier estado del evento. Un evento que MS04 no conoce responde 404.</li><li><a id="rn-ms04-23"></a>**RN-MS04-23:** Las rutas de este requerimiento y de [RF-MS04-12](#rf-ms04-12) son internas: solo las puede usar el servicio `ms06-drills`, identificado por `X-Service-Name` que el Gateway deriva de su clave de servicio. Se autorizan por servicio y no por la Matriz Rol‐Permiso. Ningún usuario las usa directamente.</li><li><a id="rn-ms04-24"></a>**RN-MS04-24:** `nombre` (1 a 150 caracteres) y `escenario` (`edificios` y `zonas`, listas de textos de hasta 50 caracteres) son informativos: MS04 los guarda y los devuelve tal cual, sin validarlos contra MS02 y sin usarlos para limitar quién puede reportar. `rolesConvocados` tiene al menos un rol, sin repetidos, solo entre ESTUDIANTE, PROFESOR, ADMINISTRATIVO, BRIGADISTA, COORDINADOR y ORGANIZADOR. `fechaInicio` llega en ISO-8601 con desfase horario (p. ej. `-05:00`), es obligatoria con `PROGRAMADO`, no se envía con `BORRADOR` y es opcional en los demás estados (si falta y el evento es nuevo, se toma la fecha/hora de recepción). Se guarda en UTC y se aplica al crear el evento, mientras esté `BORRADOR` o `PROGRAMADO` y al pasar a `EN_EJECUCION`.</li></ul> |
| **Postcondiciones** | El evento queda guardado con el estado informado por MS06 y el check-in habilitado solo si está `EN_EJECUCION`. |
| **Requerimientos asociados** | [RF-MS04-01](#rf-ms04-01), [RF-MS04-02](#rf-ms04-02), [RF-MS04-09](#rf-ms04-09), [RF-MS04-12](#rf-ms04-12). Depende de MS06. |
| **Criterios de aceptación** | <ul><li>Un `PROGRAMADO` con `fechaInicio` y roles guarda el evento (201) sin habilitar el check-in. Repetirlo con otra fecha lo reprograma (200).</li><li>Un `EN_EJECUCION` habilita el check-in: un usuario con rol convocado ya puede registrar su estado. Un `FINALIZADO` lo deshabilita y deja de aceptar check-ins.</li><li>Un `BORRADOR` sobre un evento `PROGRAMADO` elimina su fecha y mantiene el check-in deshabilitado.</li><li>Un evento con `simulacro = false` queda en el estado enviado (por ejemplo `BORRADOR` o `PROGRAMADO`, con el check-in deshabilitado) y lo habilita al recibir `EN_EJECUCION`.</li><li>Un `PROGRAMADO` después de `FINALIZADO`, o un `simulacro` distinto del guardado, devuelve 409 y no modifica nada.</li><li>Repetir el mismo `EN_EJECUCION` o `FINALIZADO` devuelve 200 sin cambios.</li><li>Una llamada sin `X-Service-Name` devuelve 401 y una de otro servicio devuelve 403.</li><li>Cada cambio, exitoso o fallido, genera un evento de auditoría hacia MS08.</li></ul> |
| **Prioridad** | Alta |
| **Estado / Squad responsable** | Pendiente / kAiRoS (coordinar contrato con MS06) |

**Transiciones de estado aceptadas ([RN-MS04-21](#rn-ms04-21)):**

| Estado guardado | Estado recibido | Resultado |
|---|---|---|
| (evento nuevo) | `BORRADOR`, `PROGRAMADO`, `EN_EJECUCION` o `FINALIZADO` | 201. Se crea el evento en el estado recibido, sea o no simulacro ([RN-MS04-20](#rn-ms04-20)). |
| `BORRADOR` | `BORRADOR` | 200. Actualiza nombre, escenario y roles. |
| `BORRADOR` | `PROGRAMADO` | 200. Guarda la fecha de inicio. |
| `BORRADOR` | `EN_EJECUCION` | 200. Habilita el check-in (inicio sin programación previa). |
| `PROGRAMADO` | `PROGRAMADO` | 200. Reprogramación: actualiza fecha, nombre, escenario y roles. |
| `PROGRAMADO` | `BORRADOR` | 200. Cancela la programación y elimina la fecha. |
| `PROGRAMADO` | `EN_EJECUCION` | 200. Habilita el check-in. |
| `EN_EJECUCION` | `EN_EJECUCION` | 200. Idempotente, sin cambios. |
| `EN_EJECUCION` | `FINALIZADO` | 200. Deshabilita el check-in y guarda la fecha de cierre. |
| `FINALIZADO` | `FINALIZADO` | 200. Idempotente, sin cambios. |
| Cualquier otra combinación | — | 409 `CONFLICTO`. |

<br>

<a id="rf-ms04-12"></a>
### RF-MS04-12 Entregar a MS06 el detalle de check-ins del evento

| Atributo | Detalle |
|---|---|
| **ID** | RF-MS04-12 |
| **Nombre** | Entregar a MS06 el detalle de check-ins del evento |
| **Épica / Grupo funcional** | MS04 - Evento |
| **Descripción** | Permitir que MS06 consulte, por ruta interna, el estado vigente de cada persona en un evento (con su nombre y su punto de encuentro), para dar seguimiento al simulacro y evaluarlo. |
| **Actor(es)** | Sistema MS06 (`ms06-drills`), por ruta interna. |
| **Precondiciones** | El evento existe (en cualquier estado). La llamada interna MS06 → MS04 está registrada en los Estándares. |
| **Flujo base** | <ol><li>MS06 solicita los check-ins del evento (`GET /api/v1/chi/eventos/{eventoId}/checkins`), con filtro opcional por `estado` y paginación (`page`, `size`).</li><li>El sistema identifica al servicio por `X-Service-Name` y verifica que sea `ms06-drills` ([RN-MS04-23](#rn-ms04-23)).</li><li>Valida los parámetros: `estado` del catálogo ([RN-MS04-03](#rn-ms04-03)), `page` desde 0 y `size` de 1 a 100 (20 por defecto).</li><li>Obtiene los estados vigentes del evento (el check-in más reciente de cada usuario), filtrados por estado si se indicó, del más antiguo al más reciente.</li><li>Obtiene el nombre de cada persona en MS01 ([RN-MS04-25](#rn-ms04-25)).</li><li>Responde `page`, `size`, `total` e `items`, con un elemento por persona: `usuarioId`, `nombre`, `estado`, `zonaId` y `registradoEn`. `zonaId` es el id del punto de encuentro (solo con `SAFE`) y es nulo en los demás estados. El formato es el acordado con MS06 y no el de los listados protegidos.</li></ol> |
| **Flujos alternos / excepciones** | <ul><li>**A1 (paso 3), parámetros inválidos:** el sistema responde 400 con el detalle de los campos.</li><li>**A2 (paso 2), no llega `X-Service-Name`:** el sistema responde 401 `NO_AUTENTICADO`.</li><li>**A3 (paso 2), la llamada viene de otro servicio:** el sistema responde 403 `PROHIBIDO`.</li><li>**A4 (paso 1), evento inexistente:** el sistema responde 404.</li><li>**A5 (paso 4), el evento no tiene check-ins:** el sistema responde 200 con `total` 0 e `items` vacío.</li><li>**A6 (paso 5), MS01 no responde:** el sistema responde 200 con `nombre` nulo y el fallo queda en el log.</li></ul> |
| **Reglas De Negocio** | <ul><li><a id="rn-ms04-25"></a>**RN-MS04-25:** Las listas de personas ([RF-MS04-06](#rf-ms04-06)), pendientes ([RF-MS04-07](#rf-ms04-07)), asistencia ([RF-MS04-08](#rf-ms04-08)) y el detalle de esta ruta muestran el nombre de cada persona, que MS04 obtiene de MS01 (`POST /users/batch`, hasta 100 ids por página) en cada consulta y no guarda ([RN-MS04-02](#rn-ms04-02)). Si MS01 no responde, la lista se entrega con `nombre` nulo y el fallo queda en el log. Solo la lista de pendientes falla (503) si MS01 no entrega la población objetivo ([RF-MS04-07](#rf-ms04-07)).</li><li>Aplican [RN-MS04-07](#rn-ms04-07) y [RN-MS04-22](#rn-ms04-22).</li></ul> |
| **Postcondiciones** | MS06 dispone del detalle de check-ins del evento. No se modifica ningún dato. |
| **Requerimientos asociados** | [RF-MS04-03](#rf-ms04-03), [RF-MS04-10](#rf-ms04-10), [RF-MS04-11](#rf-ms04-11). Depende de MS01 (nombres) y de MS06. |
| **Criterios de aceptación** | <ul><li>Con varios check-ins de una misma persona, la respuesta trae un solo elemento por persona, con su check-in más reciente.</li><li>El filtro `estado` devuelve solo las personas cuyo estado vigente es ese. `total` cuenta las personas del filtro, no los elementos de la página.</li><li>Un evento inexistente devuelve 404, una llamada sin `X-Service-Name` devuelve 401 y una de otro servicio devuelve 403.</li><li>La consulta funciona con el evento `PROGRAMADO`, `EN_EJECUCION` o `FINALIZADO`.</li><li>Con MS01 detenido, la respuesta es 200 con `nombre` nulo.</li></ul> |
| **Prioridad** | Alta |
| **Estado / Squad responsable** | Pendiente / kAiRoS (coordinar contrato con MS06) |

<br>

<a id="rf-ms04-13"></a>
### RF-MS04-13 Consultar eventos

| Atributo | Detalle |
|---|---|
| **ID** | RF-MS04-13 |
| **Nombre** | Consultar eventos |
| **Épica / Grupo funcional** | MS04 - Evento |
| **Descripción** | Permitir que los roles de supervisión consulten los eventos que MS04 conoce, en cualquier estado (borrador, programado, en ejecución o finalizado), tengan o no el check-in habilitado, para saber cuándo, dónde y a quiénes aplica cada uno. Es de solo lectura: un evento solo lo cambia MS06 ([RF-MS04-11](#rf-ms04-11)). |
| **Actor(es)** | BRIGADISTA, COORDINADOR, ORGANIZADOR, AUDITOR (`ms04:dashboard:read`).<br>Los demás roles solo consultan los eventos en ejecución que les aplican ([RF-MS04-01](#rf-ms04-01)). |
| **Precondiciones** | Usuario autenticado con `ms04:dashboard:read`. |
| **Flujo base** | <ol><li>El usuario autorizado solicita la lista de eventos (con filtro opcional por estado y paginación) o un evento por su id.</li><li>El sistema valida su permiso (X-User-Role).</li><li>Busca en la base de datos de MS04 los eventos del estado pedido, o todos si no se filtra, ordenados por fecha de inicio del más reciente al más antiguo, con los `BORRADOR` (sin fecha) al final.</li><li>Responde con cada evento: id, nombre, tipo (simulacro o emergencia), estado, fecha de inicio, fecha de cierre (solo `FINALIZADO`), escenario, roles convocados y si el check-in está habilitado.</li></ol> |
| **Flujos alternos / excepciones** | <ul><li>**A1 (paso 3), no hay eventos (o ninguno con ese estado):** el sistema responde 200 con una lista vacía.</li><li>**A2 (paso 1), estado fuera del catálogo, página o tamaño inválidos:** el sistema responde 400 con el detalle de los campos.</li><li>**A3 (paso 2), rol sin permiso:** el sistema responde 403.</li><li>**A4 (paso 1), se pide un evento por su id y no existe:** el sistema responde 404.</li></ul> |
| **Reglas De Negocio** | <ul><li><a id="rn-ms04-26"></a>**RN-MS04-26:** Los eventos se consultan por rutas de solo lectura: ningún usuario crea, modifica ni elimina un evento ni su estado. Solo MS06 lo hace ([RF-MS04-11](#rf-ms04-11)) y un evento `FINALIZADO` no vuelve a cambiar ([RN-MS04-21](#rn-ms04-21)). Los eventos no son datos de personas ni tienen ubicación, por eso el BRIGADISTA los ve todos, sin filtro de zona.</li><li>Aplica [RN-MS04-22](#rn-ms04-22).</li></ul> |
| **Postcondiciones** | El usuario conoce los eventos. No se modifica ningún dato. |
| **Requerimientos asociados** | [RF-MS04-01](#rf-ms04-01), [RF-MS04-11](#rf-ms04-11). |
| **Criterios de aceptación** | <ul><li>Con eventos en distintos estados, la lista los devuelve todos. Con el filtro `estado` devuelve solo los de ese estado.</li><li>Un evento `BORRADOR`, `PROGRAMADO` o `FINALIZADO` se puede consultar por su id aunque su check-in esté deshabilitado.</li><li>Un evento inexistente devuelve 404 y un rol sin `ms04:dashboard:read` (p. ej. ESTUDIANTE) devuelve 403.</li><li>Ninguna ruta de usuario permite crear, modificar ni eliminar un evento.</li></ul> |
| **Prioridad** | Media |
| **Estado / Squad responsable** | Pendiente / kAiRoS |

---

<a id="requerimientos-no-funcionales-rnf"></a>
## Requerimientos No Funcionales (RNF)

| ID | Requerimiento |
|---|---|
| **RNF-MS04-01** | Crear o actualizar un check-in (incluida la validación del punto de encuentro con MS02) responde en menos de 5 segundos para al menos el 80 % de las peticiones con carga normal, para que reportar el estado sea rápido durante una emergencia. Las peticiones que superan 5 segundos quedan en el log con su Correlation ID. Se verifica midiendo el tiempo entre el envío y la respuesta con usuarios de distintos roles. |
| **RNF-MS04-02** | MS04 soporta 500 check-ins simultáneos (la institución tiene alrededor de 5000 personas) sin pérdida de datos, sin duplicados y sin errores por saturación, y cada usuario conserva un único estado vigente: su check-in más reciente. Por encima de 500 sigue respondiendo, aunque más lento. Se verifica con una prueba de carga de 500 usuarios simulados. |
| **RNF-MS04-03** | El registro de check-ins no depende de MS05: con MS05 detenido, un check-in Requiero asistencia se guarda y responde con éxito, la solicitud queda pendiente y se envía cuando MS05 se recupera ([RN-MS04-12](#rn-ms04-12)). Cada intento fallido queda en el log. |
| **RNF-MS04-04** | Los check-ins solo se agregan: ninguna operación actualiza ni elimina uno existente ([RN-MS04-07](#rn-ms04-07), [RN-MS04-08](#rn-ms04-08)), y la base de datos lo refuerza con permisos de solo INSERT y SELECT sobre ellos. Los reportes simultáneos del mismo usuario en el mismo evento se serializan en la base de datos (bloqueo por usuario y evento), de modo que el estado vigente sea siempre el último confirmado y ninguno responda 409 ni 500 por la concurrencia. |
| **RNF-MS04-05** | Tras marcar un evento como `FINALIZADO`, ningún rol puede crear, modificar ni eliminar sus check-ins ni su histórico ([RN-MS04-17](#rn-ms04-17)). Los intentos de escritura responden 409 `CONFLICTO` y quedan auditados. MS04 no expone operaciones de modificación ni de eliminación de check-ins. |
| **RNF-MS04-06** | Un check-in registrado se refleja en la consolidación, los pendientes y la lista de asistencia ([RF-MS04-06](#rf-ms04-06), [RF-MS04-07](#rf-ms04-07), [RF-MS04-08](#rf-ms04-08)) en menos de 5 segundos, con totales que cumplen [RN-MS04-13](#rn-ms04-13). |
| **RNF-MS04-07** | El contrato de MS04 permite que el usuario complete el reporte de su estado en máximo 3 interacciones desde la pantalla inicial, incluida la selección del punto de encuentro en Estoy a salvo. Los cuatro estados se muestran siempre con su texto (Estoy a salvo, Me estoy evacuando, Requiero asistencia, No estoy en el campus). Si no hay evento activo, la pantalla inicial lo indica sin pasos adicionales. Se verifica con una prueba de usabilidad con usuarios de distintos roles. |
| **RNF-MS04-08** | Aplicación Spring Boot independiente (Java, Maven) con Spring Data JPA y base de datos PostgreSQL propia. Ningún otro servicio accede a ella. |
| **RNF-MS04-09** | Todos los endpoints bajo `/api/v1/` (los de eventos, en `/api/v1/chi/eventos`), documentados con OpenAPI/Swagger y publicados antes de que otros squads dependan de ellos. Cada operación protegida declara `x-permiso` y `x-roles` iguales a la Matriz Rol‐Permiso y documenta las cabeceras de identidad como parámetros, sin `bearerAuth`. Las operaciones internas ([RF-MS04-11](#rf-ms04-11), [RF-MS04-12](#rf-ms04-12)) declaran en su lugar el servicio autorizado (`X-Service-Name`) y no usan `x-permiso`. |
| **RNF-MS04-10** | Todo acceso a MS04 llega a través del API Gateway. En las rutas protegidas MS04 recibe la identidad en X-User-Id, X-User-Role y X-Correlation-ID y, si faltan las de identidad, responde 401 `NO_AUTENTICADO`. En las rutas internas recibe X-Service-Name y X-Correlation-ID y, si falta el servicio, responde 401 `NO_AUTENTICADO`. Sus llamadas salientes (MS01, MS02, MS05 y MS08) también pasan por el Gateway con su clave de servicio, el mismo Correlation ID y, cuando existe, la identidad del usuario propagada, con timeout de cliente de 2000 ms. MS04 no configura CORS. Puerto interno 8084. |
| **RNF-MS04-11** | Todo error, incluidos los del framework (404, 405, 415, cabecera faltante), responde con el `ErrorResponse` y el catálogo de códigos de los Estándares, sin exponer trazas internas. La indisponibilidad de MS01 (población objetivo de los pendientes), MS02 (punto de encuentro) o MS05 (zonas del brigadista) responde 503 con un código propio declarado en el OpenAPI de MS04. Nunca se reenvía tal cual el cuerpo de error de otro servicio. |
| **RNF-MS04-12** | Logs útiles para diagnóstico que incluyen el Correlation ID en cada línea, sin tokens ni datos personales. Spring Boot Actuator habilitado. |
| **RNF-MS04-13** | Secretos (credenciales de BD y clave de servicio del Gateway) fuera del repositorio, mediante variables de entorno. |
| **RNF-MS04-14** | Pruebas unitarias y de integración de los flujos de recepción del estado del evento (programar, reprogramar, cancelar la programación, iniciar, finalizar, evento que no es simulacro y transiciones no permitidas), habilitación del check-in según el estado del evento, registro y actualización de check-in en los cuatro estados, estado vigente como check-in más reciente, reportes simultáneos del mismo usuario, solicitud de asistencia (incluida la caída de MS05), consolidación, pendientes, lista de asistencia, histórico, resultados, detalle de check-ins para MS06, consulta de eventos en cualquier estado, autorización de las rutas internas por `X-Service-Name`, lectura de cabeceras de identidad y emisión de eventos de auditoría a MS08. Análisis de calidad con SonarCloud. |
| **RNF-MS04-15** | Ejecución en contenedor Docker, con pipeline en GitHub Actions. Código integrado por Pull Request con revisión. |
| **RNF-MS04-16** | Entradas validadas en el servidor contra datos inválidos. Los listados usan la paginación de los Estándares, salvo la ruta interna de [RF-MS04-12](#rf-ms04-12), cuyo formato está acordado con MS06. |
| **RNF-MS04-17** | La auditoría se delega en [ms08-analytics-and-audit](https://github.com/CampusSeguro-DOSW-2026-2/ms08-analytics-and-audit) (MS08), que almacena y consulta los eventos (el AUDITOR consulta allí los registros). MS04 los emite de forma asíncrona con el formato de los Estándares (`modulo` = MS04; acciones REGISTRO_CHECKIN, CAMBIO_ESTADO_CHECKIN, SOLICITUD_ASISTENCIA y CAMBIO_ESTADO_EVENTO; resultado EXITO o FALLO; `recurso_afectado` = `checkin:<id>`, o `evento:<id>` si el check-in no llegó a crearse o el cambio es del evento) con el Correlation ID y sin datos personales distintos de los ids. En los cambios de estado del evento, el servicio de origen (`X-Service-Name`) va en `detalle`. Si MS08 falla, la respuesta al cliente no cambia y el error se registra en logs. MS04 no tiene tablas ni endpoints de auditoría: el histórico de check-ins es un dato del dominio, no auditoría, y el historial de cambios de estado de un evento se consulta en MS08. |
| **RNF-MS04-18** | La recepción del estado del evento ([RF-MS04-11](#rf-ms04-11)) responde en menos de 2000 ms, el timeout de cliente de los Estándares con el que la llama MS06, y es idempotente: repetir la misma petición deja el mismo evento. Dos llamadas simultáneas sobre el mismo evento se atienden una tras otra, sin errores 500. |

---

<a id="dependencias-y-decisiones-abiertas"></a>
## Dependencias y decisiones abiertas

MS04 depende de contratos que aún no están publicados. Se acuerdan con cada squad y se registran en los Estándares antes de implementarlos.

| # | Tema | Responsables | Estado |
|---|---|---|---|
| 1 | Contrato de las rutas internas MS06 → MS04 ([RF-MS04-11](#rf-ms04-11) y [RF-MS04-12](#rf-ms04-12)): registrar las llamadas en los Estándares (`X-Service-Name` = `ms06-drills`) y configurar en el Gateway la política interna por método (la ruta `/{eventoId}/checkins` también tiene un POST protegido para los usuarios). Confirmar con MS06: `eventoId` como UUID, valores de `estado` y `fechaInicio`, la respuesta del PUT (la propone MS04: el evento almacenado y si el check-in quedó habilitado), `zonaId` como id del punto de encuentro (solo con `SAFE`), los códigos de estado del check-in en el filtro `estado`, y que `nombre` y `escenario` son solo informativos. MS06 usa camelCase en estas rutas y el resto de MS04 usa snake_case. | MS06 + MS04 | Pendiente |
| 2 | Llamadas MS04 → MS01 para la población objetivo (`GET /users/by-role`, [RF-MS04-07](#rf-ms04-07)) y los nombres (`POST /users/batch`, [RN-MS04-25](#rn-ms04-25)), definidas en RF-MS01-09: autorizar a MS04 (hoy solo están configurados MS03 y MS05) y registrar la llamada en los Estándares. Volumen: con unas 5000 personas, calcular los pendientes recorre unas 50 páginas de 100; acordar un tamaño de página mayor o una caché. | MS01 + MS04 | Pendiente |
| 3 | Registrar la llamada MS04 → MS02 para validar puntos de encuentro (`ms02:campus:read`, identidad propagada). | MS02 + MS04 | Pendiente |
| 4 | Endpoint interno de MS05 para las solicitudes que vienen de MS04 (decisión abierta n.º 3 de los Estándares): campos, idempotencia por check-in y cómo sabe MS04 si una solicitud sigue abierta. | MS04 + MS05 | Pendiente |
| 5 | Endpoint de MS05 con las zonas asignadas a un brigadista, para limitar su consolidación y su lista de asistencia (MR-03 de la matriz). | MS04 + MS05 | Pendiente |
| 6 | Consumo de los resultados del evento ([RF-MS04-10](#rf-ms04-10)) por MS06 y MS08 (MS06 ya obtiene el detalle de check-ins por [RF-MS04-12](#rf-ms04-12)): registrar las llamadas MS06 → MS04 y MS08 → MS04 y definir si la ruta es protegida (con la identidad del usuario propagada) o interna (por `X-Service-Name`). | MS04 + MS06 + MS08 | Pendiente |

**Decisiones tomadas en este planteamiento:**
* MS06 es la única fuente de los eventos: MS04 ya no consulta alertas a MS03 y por eso desaparece su dependencia de MS03.
* La población objetivo sale de MS01, por los roles convocados del evento ([RN-MS04-15](#rn-ms04-15)).
* Los nombres de las personas salen de MS01 en cada consulta y no se guardan ([RN-MS04-25](#rn-ms04-25)).
* Las personas sin ubicación conocida (los pendientes y quienes están en Me estoy evacuando o No estoy en el campus) no se muestran al BRIGADISTA, porque la ubicación solo se conoce con Estoy a salvo y Requiero asistencia ([RN-MS04-14](#rn-ms04-14)). Sin cambios en la Matriz.
* Los eventos en cualquier estado los consultan solo los roles con `ms04:dashboard:read` ([RF-MS04-13](#rf-ms04-13)). El resto de roles ve solo los eventos en ejecución que le aplican ([RF-MS04-01](#rf-ms04-01)).
* Las rutas de MS04 pasan a `/api/v1/chi/eventos`. Las de [RF-MS04-11](#rf-ms04-11) y [RF-MS04-12](#rf-ms04-12) son internas de MS06 y el resto sigue protegido por la Matriz, sin cambios en ella.

---

<a id="api-ms04-evacuation-and-check-in"></a>
## API: ms04-evacuation-and-check-in
Reporte del estado de las personas, consolidación por punto de encuentro y habilitación del check-in según el estado del evento.

Contrato OpenAPI: [`api/swagger.yml`](api/swagger.yml) Las rutas protegidas reciben la identidad por cabeceras ([RN-MS04-06](#rn-ms04-06)) y las internas se autorizan por servicio ([RN-MS04-23](#rn-ms04-23)). La columna *Permiso* coincide con el `x-permiso` del contrato y con la [Matriz Rol‐Permiso](https://github.com/CampusSeguro-DOSW-2026-2/documentation/wiki/Matriz-Rol%E2%80%90Permiso). COM = ESTUDIANTE, PROFESOR, ADMINISTRATIVO, BRIGADISTA, COORDINADOR, ORGANIZADOR.

| Categoría | Método | Ruta (/api/v1/chi/eventos) | Acceso | Permiso | RF asociado |
|---|---|---|---|---|---|
| **Estado del evento** | PUT | `/{eventoId}/estado` | Interna: `ms06-drills` | — (se autoriza por `X-Service-Name`) | [11](#rf-ms04-11) |
| | GET | `/{eventoId}/checkins` | Interna: `ms06-drills` | — (se autoriza por `X-Service-Name`) | [12](#rf-ms04-12) |
| **Evento activo** | GET | `/activos` | COM | `ms04:checkin:read_own` | [01](#rf-ms04-01) |
| **Consulta de eventos** | GET | `/` | BRIGADISTA, COORDINADOR, ORGANIZADOR, AUDITOR | `ms04:dashboard:read` | [13](#rf-ms04-13) |
| | GET | `/{eventoId}` | BRIGADISTA, COORDINADOR, ORGANIZADOR, AUDITOR | `ms04:dashboard:read` | [13](#rf-ms04-13) |
| **Check-in propio** | POST | `/{eventoId}/checkins` | COM | `ms04:checkin:write`, `ms04:assistance:request` (si es Requiero asistencia) | [02](#rf-ms04-02), [03](#rf-ms04-03), [04](#rf-ms04-04) |
| | GET | `/{eventoId}/checkins/propio` | COM | `ms04:checkin:read_own` | [05](#rf-ms04-05) |
| **Consolidación** | GET | `/{eventoId}/consolidado` | BRIGADISTA (sus zonas), COORDINADOR, ORGANIZADOR, AUDITOR | `ms04:dashboard:read` | [06](#rf-ms04-06) |
| | GET | `/{eventoId}/personas` | BRIGADISTA (sus zonas), COORDINADOR, ORGANIZADOR, AUDITOR | `ms04:dashboard:read` | [06](#rf-ms04-06) |
| | GET | `/{eventoId}/pendientes` | BRIGADISTA (sin personas, [RN-MS04-14](#rn-ms04-14)), COORDINADOR, ORGANIZADOR, AUDITOR | `ms04:dashboard:read` | [07](#rf-ms04-07) |
| **Asistencia** | GET | `/{eventoId}/asistencia` | BRIGADISTA (sus zonas), COORDINADOR, ORGANIZADOR, AUDITOR | `ms04:dashboard:read` | [08](#rf-ms04-08) |
| **Histórico y resultados** | GET | `/{eventoId}/historial` | COORDINADOR, ORGANIZADOR, AUDITOR | `ms04:history:read` | [09](#rf-ms04-09) |
| | GET | `/{eventoId}/resultados` | COORDINADOR, ORGANIZADOR, AUDITOR (y MS06, MS08) | `ms04:history:read` | [10](#rf-ms04-10) |

* `GET /{eventoId}/personas` devuelve los estados vigentes del evento con paginación y filtros opcionales por estado y punto de encuentro (p. ej. las personas en Estoy a salvo de un punto). `GET /{eventoId}/checkins` es la consulta interna de MS06.
* `POST /{eventoId}/checkins` y `GET /{eventoId}/checkins` comparten ruta pero no política: el POST es protegido (usuarios) y el GET es interno (MS06).
* `GET /` es `GET /api/v1/chi/eventos`: lista los eventos en cualquier estado, con filtro opcional por `estado` y paginación. `GET /{eventoId}` devuelve un evento. Ambas son de solo lectura ([RN-MS04-26](#rn-ms04-26)).
* `PUT /{eventoId}/estado` responde 201 si crea el evento y 200 si lo actualiza o repite.
* La consulta de auditoría no pertenece a MS04: la expone ms08-analytics-and-audit (RNF-MS04-17).
