# Formularios y entradas

## Índice de Contenido
1. [Reglas generales de entrada](#reglas-generales-de-entrada)
2. [Formularios y Entradas](#formularios-y-entradas)
   - [F-01 · Crear evento](#f-01-crear-evento)
   - [F-02 · Editar evento](#f-02-editar-evento)
   - [F-03 · Búsqueda y filtros de eventos](#f-03-búsqueda-y-filtros-de-eventos)
   - [F-04 · Finalizar evento](#f-04-finalizar-evento)
   - [F-05 · Crear tipo de evento](#f-05-crear-tipo-de-evento)
   - [F-06 · Detalle y panel de un evento](#f-06-detalle-y-panel-de-un-evento)

---

## Reglas generales de entrada

* **RG-01**: El nombre del evento se recorta (espacios al inicio y al final) antes de guardarse. El título de la búsqueda se recorta y se pasa a minúsculas para comparar, y el nombre del tipo de evento se recorta para generar su código. Los nombres de rol se pasan a MAYÚSCULAS antes de validarlos. Los demás textos se guardan tal como llegan.
* **RG-02**: Un campo obligatorio ausente, vacío o con solo espacios se considera faltante y se rechaza con 400 `VALIDACION`.
* **RG-03**: Se rechazan (HTTP 400) los cuerpos que no son JSON válido o que traen un tipo de dato equivocado (por ejemplo un `typeIds` que no sea una lista de UUID). Los campos de identidad (usuario, rol, permisos) no van en ningún formulario: los inyecta el Gateway ([RG-06](#reglas-generales-de-entrada)).
* **RG-04**: Los errores de validación responden 400 `VALIDACION` con el `ErrorResponse` de los [Estándares](../integration/standards.md): `status`, `codigo`, `mensaje`, `errores` y `correlation_id`. Cada campo inválido va en `errores` con su `campo` y su `mensaje`. Los mensajes de las reglas de negocio de MS06 están en inglés, tal como los devuelve la API, y la interfaz muestra su propio texto.
* **RG-05**: Los errores de reglas de negocio usan estos códigos: 404 `NO_ENCONTRADO` (el evento no existe), 409 `TRANSICION_INVALIDA` (el cambio de estado no está permitido), 409 `EVENTO_NO_EDITABLE` (el evento está finalizado), 409 `EVENTO_NO_ACTIVO` (el panel solo existe para un evento en ejecución), 409 `MODIFICACION_CONCURRENTE` (alguien lo editó al mismo tiempo), 409 `CONFLICTO` (nombre de tipo repetido) y 503 `SERVICIO_EXTERNO_NO_DISPONIBLE` (un módulo requerido no respondió).
* **RG-06**: La identidad del solicitante (`X-User-Id`, `X-User-Role`, `X-User-Permission`) la inyecta el Gateway; ningún formulario la envía ni la acepta como dato. La autorización se decide por permiso ([RF-EVE-01](1_requirements.md) a [RF-EVE-05](1_requirements.md)): `ms06:drill:*` para simulacros y `ms06:real_event:*` para eventos reales, cada uno con `manage`, `execute` y `read`. El tipo del evento decide qué permiso se exige.
* **RG-07**: Los estados son `DRAFT`, `SCHEDULED`, `IN_PROGRESS` y `FINISHED`. Se escriben en MAYÚSCULAS, tal como aparecen en el contrato.
* **RG-08**: Las fechas viajan en formato ISO 8601 con zona (por ejemplo `2026-12-01T15:00:00Z`). La API las compara en UTC con el reloj de la aplicación.
* **RG-09**: Si un módulo de apoyo no responde al guardar, el evento se guarda igual cuando es posible y la respuesta incluye una advertencia (`warnings`) con un código (`SCOPE_PENDING`, `BRIGADIERS_PENDING`, `SAVED_AS_DRAFT`, `FINISH_ALERT_NOT_SENT`, entre otros). Las excepciones están descritas en cada formulario.

---

## Formularios y Entradas

### F-01: Crear evento
Referencia: [RF-EVE-01](1_requirements.md) · `POST /api/v1/eventos` · Permiso: `ms06:drill:manage` (simulacro) o `ms06:real_event:manage` (evento real)

| Campo | Tipo | Obligatorio | Longitud / formato | Reglas de validación | Mensaje de error | Ejemplo válido |
|---|---|---|---|---|---|---|
| Nombre (`name`) | Texto | Sí | Hasta 150 caracteres | No vacío ni solo espacios. | Campo `name` (400 `VALIDACION`): no puede estar vacío ni superar 150 caracteres. | Simulacro de evacuación bloque A |
| Descripción (`description`) | Texto largo | No | Hasta 1000 caracteres | Objetivo y contexto del evento. Puede ir vacía. | Campo `description` (400): no puede superar 1000 caracteres. | Simulacro semestral de evacuación |
| Tipos (`typeIds`) | Lista de UUID | Sí | De 1 a 4 elementos | Cada UUID debe ser de un tipo existente y activo del catálogo ([F-05](#f-05-crear-tipo-de-evento)). Si uno de los tipos es `SIMULACRO`, el evento es un simulacro. | Vacía o con más de 4 (400): campo `typeIds`. Tipo inexistente o inactivo (400): campo `typeIds` con los identificadores inválidos. | `["<id-del-tipo-SIMULACRO>"]` |
| Fecha de inicio (`startDate`) | Fecha y hora | No | ISO 8601 con zona | Debe ser posterior al momento actual. Sin fecha, el evento queda en borrador. | "The start date must be in the future" (400, campo `startDate`). | `2026-12-01T15:00:00Z` |
| Alcance (`scope`) | Lista de objetos | No | Cada elemento: `type` y `id` | Ver las dos filas siguientes. Los elementos repetidos se descartan. Los edificios y zonas deben existir y estar activos en MS02. | Elemento inexistente o inactivo en MS02 (400): campo `scope`. MS02 no responde: no se guarda el alcance y se devuelve la advertencia `SCOPE_PENDING`. | `[{"type":"ZONE","id":"z1"}]` |
| ↳ Tipo (`scope[].type`) | Lista | Sí, en cada elemento | `BUILDING` o `ZONE` | Solo valores del catálogo, en MAYÚSCULAS. | Campo `scope[n].type` (400): obligatorio y del catálogo. | ZONE |
| ↳ Identificador (`scope[].id`) | Texto | Sí, en cada elemento | Hasta 64 caracteres | No vacío. Es el identificador del edificio o la zona en MS02. | Campo `scope[n].id` (400): no puede estar vacío ni superar 64 caracteres. | 3f6c2a9e-8b1d-4c7a-9e2f-1a5b7c9d0e42 |
| Participantes (`participantRoles`) | Lista de textos | No | Nombres de rol | Cada nombre se pasa a MAYÚSCULAS y debe tener el formato de los roles de MS01: 2 a 30 caracteres, letras sin tilde, dígitos o guion bajo, empezando con una letra. Los repetidos se descartan. | Formato inválido (400): campo `participantRoles` con los nombres rechazados. | `["ESTUDIANTE","PROFESOR"]` |

**Reglas globales de formulario:**
* El evento se guarda como `SCHEDULED` solo si tiene fecha de inicio, alcance y participantes. Si falta algo, queda en `DRAFT` y se pasa a `SCHEDULED` cuando se completen los datos ([F-02](#f-02-editar-evento)).
* Al programarse se crean en MS03 la alerta a los brigadistas y la alerta general (que se activa a la hora de inicio) y se informa el escenario a MS04 y MS05. Si MS03, MS04 o MS05 no responden, el evento queda en `DRAFT` con la advertencia `SAVED_AS_DRAFT`.
* Un simulacro lo gestiona quien tenga `ms06:drill:manage` (el organizador) y un evento real quien tenga `ms06:real_event:manage` (el coordinador). Sin el permiso del tipo del evento responde 403 `PROHIBIDO`.
* Respuesta 201 con el evento: `id`, `status`, fechas, tipos, alcance (con el nombre de cada edificio o zona), participantes, alertas y `warnings`.

---

### F-02: Editar evento
Referencia: [RF-EVE-02](1_requirements.md) · `PUT /api/v1/eventos/{id}` · Permiso: `manage` del tipo del evento

El cuerpo es el mismo de [F-01](#f-01-crear-evento) y reemplaza todos los campos editables.

| Campo | Tipo | Obligatorio | Longitud / formato | Reglas de validación | Mensaje de error | Ejemplo válido |
|---|---|---|---|---|---|---|
| Evento a editar (en la ruta, `id`) | UUID | Sí | UUID | Debe existir y no estar finalizado. | No existe (404 `NO_ENCONTRADO`): "Event not found: {id}". Finalizado (409 `EVENTO_NO_EDITABLE`): "A finished event cannot be edited". | 3f6c2a9e-8b1d-4c7a-9e2f-1a5b7c9d0e42 |
| Nombre, descripción, tipos | — | Nombre y tipos, sí | Igual que [F-01](#f-01-crear-evento) | Un simulacro no puede pasar a ser evento real ni al revés: la lista de tipos debe conservar o no tener `SIMULACRO`, igual que antes. | Cambio de naturaleza (400): "A drill cannot become a real event or the other way around" (campo `typeIds`). | — |
| Fecha de inicio (`startDate`) | Fecha y hora | No | ISO 8601 con zona | En un evento en `DRAFT`, posterior al momento actual. En uno en `SCHEDULED`, posterior al momento actual, salvo que se conserve la misma fecha que ya tenía. En uno en `IN_PROGRESS` no se puede cambiar. | Pasada (400): "The start date must be in the future". En ejecución (400): "The start date of a started event cannot change". | `2026-12-02T15:00:00Z` |
| Alcance (`scope`) | Lista de objetos | No | Igual que [F-01](#f-01-crear-evento) | Si no se envía, se conserva el actual. Si se envía vacía, se borra. En un evento en ejecución no puede quedar vacío. | Evento en ejecución sin alcance (400): "A started event must keep its scope and its participants" (campo `scope`). | — |
| Participantes (`participantRoles`) | Lista de textos | No | Igual que [F-01](#f-01-crear-evento) | Igual que el alcance: si no se envía se conserva, y en un evento en ejecución no puede quedar vacío. | Evento en ejecución sin participantes (400): mismo mensaje, campo `participantRoles`. | — |

**Reglas globales de formulario:**
* Solo se editan eventos en `DRAFT`, `SCHEDULED` e `IN_PROGRESS`.
* Un evento en `DRAFT` que queda completo (fecha, alcance y participantes) pasa a `SCHEDULED` y se crean sus alertas.
* Un evento en `SCHEDULED` cuyos datos siguen completos se re-sincroniza: MS04 y MS05 reciben el escenario nuevo, la alerta general sigue la nueva fecha y los brigadistas reciben una alerta para que confirmen su asignación. Si los datos quedan incompletos, vuelve a `DRAFT` y se archiva la alerta general.
* Un evento en `IN_PROGRESS` con cambios reales envía el escenario actualizado a MS04 y MS05 y crea en MS03 dos alertas nuevas con la información actualizada: una para los brigadistas y otra para los participantes.
* Si un módulo requerido no responde, la edición se revierte y responde 503 `SERVICIO_EXTERNO_NO_DISPONIBLE`.
* Dos ediciones simultáneas del mismo evento: la segunda responde 409 `MODIFICACION_CONCURRENTE`.
* Sin cambios reales, la edición no genera auditoría ni llamadas a otros módulos.

---

### F-03: Búsqueda y filtros de eventos
Referencia: [RF-EVE-03](1_requirements.md) · `GET /api/v1/eventos` · Permiso: `ms06:drill:read` o `ms06:real_event:read`

| Campo | Tipo | Obligatorio | Longitud / formato | Reglas de validación | Mensaje de error | Ejemplo válido |
|---|---|---|---|---|---|---|
| Título (`title`) | Texto | No | Sin límite propio | Búsqueda parcial sin distinguir mayúsculas. Se recorta; `%` y `_` se buscan como texto. Vacío equivale a todos. | — (nunca falla) | simulacro |
| Estado (`status`) | Lista | No | `DRAFT` / `SCHEDULED` / `IN_PROGRESS` / `FINISHED` | Solo valores del catálogo, en MAYÚSCULAS. Sin valor incluye todos. | 400 `VALIDACION`: valor de `status` no válido. | SCHEDULED |
| Página (`page`) | Número entero | No | ≥ 0 (por defecto 0) | Entero no negativo. | "The page number cannot be negative" (400, campo `page`). | 0 |
| Tamaño de página (`size`) | Número entero | No | 1 a 100 (por defecto 20) | Entero en el rango. | "The page size must be between 1 and 100" (400, campo `size`). | 20 |

**Mensajes globales de formulario:**
* Solo se devuelven los eventos de los tipos que el usuario puede leer: simulacros con `ms06:drill:read` y eventos reales con `ms06:real_event:read`. Sin ninguno de los dos permisos responde 403 `PROHIBIDO`.
* Sin coincidencias, responde 200 con una página vacía, no un error.
* Orden: del más reciente al más antiguo por fecha de creación.
* `GET /api/v1/eventos/activos` no lleva filtros: devuelve los eventos en ejecución que conciernen al usuario (los que gestiona y los que convocan alguno de sus roles) y lo puede llamar cualquier usuario autenticado.

---

### F-04: Finalizar evento
Referencia: [RF-EVE-04](1_requirements.md) · `PATCH /api/v1/eventos/{id}/finalizar` · Permiso: `ms06:drill:execute` o `ms06:real_event:execute`

| Campo | Tipo | Obligatorio | Longitud / formato | Reglas de validación | Mensaje de error | Ejemplo válido |
|---|---|---|---|---|---|---|
| Evento a finalizar (en la ruta, `id`) | UUID | Sí | UUID | Debe existir y estar en `IN_PROGRESS`. | Formato inválido (400). No existe (404 `NO_ENCONTRADO`). En otro estado (409 `TRANSICION_INVALIDA`): un evento que no está en ejecución no se puede finalizar. | 3f6c2a9e-8b1d-4c7a-9e2f-1a5b7c9d0e42 |

**Mensajes globales de formulario:**
* No lleva cuerpo.
* Si MS04 (check-in) o MS05 (brigadistas) no responden, el evento **no** finaliza: responde 503 `SERVICIO_EXTERNO_NO_DISPONIBLE` y sigue en ejecución.
* Si MS03 (alertas) no responde, el evento finaliza igual y la respuesta incluye la advertencia `FINISH_ALERT_NOT_SENT`.
* La respuesta 200 incluye las métricas generales: check-ins y porcentaje de participación, incidentes, casos atendidos y resueltos, y tiempo promedio de atención. Si faltan cifras (`EVALUATION_INCOMPLETE`), un proceso periódico las completa después.

---

### F-05: Crear tipo de evento
Referencia: [RF-EVE-01](1_requirements.md) · `POST /api/v1/tipos-evento` · Permiso: `ms06:real_event:manage`

| Campo | Tipo | Obligatorio | Longitud / formato | Reglas de validación | Mensaje de error | Ejemplo válido |
|---|---|---|---|---|---|---|
| Nombre (`name`) | Texto | Sí | Hasta 80 caracteres | No vacío. Debe tener al menos una letra o un dígito. Único sin distinguir mayúsculas ni tildes. El código se genera solo: sin tildes, en MAYÚSCULAS, con guion bajo en lugar de cualquier otro símbolo y recortado a 40 caracteres. | Vacío o más de 80 (400, campo `name`). Sin letras ni dígitos (400): "The name must have letters or digits". Repetido (409 `CONFLICTO`): "An event type like '{nombre}' already exists". | Tormenta eléctrica |

**Mensajes globales de formulario:**
* El tipo nuevo queda activo y elegible como tipo de alerta. El código (`TORMENTA_ELECTRICA` en el ejemplo) no se envía: es de solo lectura.
* `GET /api/v1/tipos-evento` lista los tipos activos para el selector del formulario ([F-01](#f-01-crear-evento)) y lo puede llamar quien tenga `read` o `manage` de cualquier tipo. `GET /api/v1/tipos-alerta` es el catálogo que consume MS03 y no incluye `SIMULACRO`.

---

### F-06: Detalle y panel de un evento
Referencia: [RF-EVE-03](1_requirements.md), [RF-EVE-05](1_requirements.md) · `GET /api/v1/eventos/{id}` y `GET /api/v1/eventos/{id}/panel`

| Campo | Tipo | Obligatorio | Longitud / formato | Reglas de validación | Mensaje de error | Ejemplo válido |
|---|---|---|---|---|---|---|
| Evento (en la ruta, `id`) | UUID | Sí | UUID | Debe existir. El detalle exige `ms06:drill:read` o `ms06:real_event:read` del tipo del evento; el panel lo puede pedir cualquier usuario autenticado. El panel solo existe para un evento en ejecución. | Formato inválido (400). No existe (404 `NO_ENCONTRADO`). Panel de un evento que no está en ejecución (409 `EVENTO_NO_ACTIVO`). | 3f6c2a9e-8b1d-4c7a-9e2f-1a5b7c9d0e42 |

**Mensajes globales de formulario:**
* El detalle de un evento en ejecución incluye el monitoreo (brigadistas por zona, check-ins e incidentes en tiempo real) y el de un evento finalizado incluye su evaluación guardada.
* Si MS04 o MS05 no responden, el detalle se devuelve igual con una advertencia (`CHECKIN_UNAVAILABLE`, `INCIDENTS_UNAVAILABLE` o `BRIGADIERS_NOT_SYNCED`); una lista vacía produce `NO_CHECKINS`, `NO_INCIDENTS` o `NO_BRIGADIERS`.
* El panel entrega el nombre del evento, las rutas de su alcance (de MS02) y lo necesario para los botones de check-in y de pedir ayuda.
