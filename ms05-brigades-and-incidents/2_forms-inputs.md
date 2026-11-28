# Especificación de formularios y entradas

Este documento describe los formularios y las entradas que utiliza el microservicio **MS05 – Brigadistas e Incidentes**. Su objetivo es especificar qué información recibe cada operación, cuáles son los campos obligatorios y opcionales, qué validaciones se aplican, qué permisos se requieren y qué mensajes devuelve la API cuando ocurre algún error.

Los mensajes de error incluidos en este documento corresponden a las respuestas que devuelve actualmente la API de MS05 y se presentan en español.

## Índice de contenido

1. [Reglas generales de entrada](#reglas-generales-de-entrada)
2. [F-01 Asignar brigadista a una ubicación](#f-01-asignar-brigadista-a-una-ubicación)
3. [F-02 Modificar o retirar una asignación](#f-02-modificar-o-retirar-una-asignación)
4. [F-03 Panel de gestión y autoasignación por evento](#f-03-panel-de-gestión-y-autoasignación-por-evento)
5. [F-04 Buscar y activar brigadistas](#f-04-buscar-y-activar-brigadistas)
6. [F-05 Registrar incidente](#f-05-registrar-incidente)
7. [F-06 Editar y eliminar incidente](#f-06-editar-y-eliminar-incidente)
8. [F-07 Gestionar la atención de un incidente](#f-07-gestionar-la-atención-de-un-incidente)
9. [F-08 Adjuntar y consultar evidencias](#f-08-adjuntar-y-consultar-evidencias)
10. [F-09 Consultas, filtros y auditoría](#f-09-consultas-filtros-y-auditoría)

---

## Reglas generales de entrada

Antes de describir cada formulario, es importante tener en cuenta las reglas que se aplican de manera general a las operaciones de MS05.

| ID    | Regla                     | Descripción                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                   |
| ----- | ------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| RG-01 | Identidad del usuario     | MS05 no confía en la identidad enviada por el cuerpo de la solicitud ni por la URL. El API Gateway valida el JWT y proporciona las cabeceras `X-User-Id` y `X-User-Permission`, que contienen la identidad y los permisos efectivos del usuario, separados por comas. Para las solicitudes internas de MS06, el Gateway valida la clave de servicio e inyecta `X-Service-Name: ms06`.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                         |
| RG-02 | Autorización              | MS05 verifica los permisos recibidos en `X-User-Permission`. Entre ellos se encuentran `ms05:brigadier:assign_to_drill`, `ms05:brigadier:assign_to_real`, `ms05:panel:read_own`, `ms05:assistance:assign_to_drill`, `ms05:assistance:assign_to_real`, `ms05:assistance:update_status`, `ms05:incident:report`, `ms05:incident:manage_own`, `ms05:incident:read`, `ms05:incident:update`, `ms08:audit_drill:read` y `ms08:audit_real_event:read`. Las rutas internas utilizadas por MS06 requieren la autoridad `SERVICE_MS06`. La separación entre simulacros (`*_to_drill`) y eventos reales (`*_to_real`) se comprueba según la configuración del evento.                                                                                                                                                                                                                                                   |
| RG-03 | Estado del brigadista     | Para recibir asignaciones o atender incidentes, el usuario debe existir en MS01, estar activo allí y tener el rol `BRIGADISTA`. Además, debe encontrarse en estado `ACTIVO` dentro de MS05.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                   |
| RG-04 | Estado del evento         | Los incidentes únicamente pueden registrarse cuando el evento está en estado `EN_EJECUCION`. Por su parte, el panel de gestión y la autoasignación funcionan con eventos abiertos, es decir, en estado `PROGRAMADO` o `EN_EJECUCION`. MS06 informa estos cambios mediante `PUT /api/v1/bei/eventos/{eventoId}/estado`.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                        |
| RG-05 | Validación de ubicaciones | El campo `ubicacionId` corresponde al identificador de una ubicación registrada en MS02. El campo `tipoUbicacion` puede tomar los valores `EDIFICIO`, `ZONA_SEGURA` o `PUNTO_ENCUENTRO`. Antes de realizar una asignación o registrar un incidente, MS05 comprueba que la ubicación exista y esté activa en MS02.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                             |
| RG-06 | Identificadores           | Los identificadores de incidentes, evidencias y asignaciones utilizan el formato UUID.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                        |
| RG-07 | Formato de errores        | De acuerdo con el estándar de respuestas de la sección 7.1, los errores utilizan la estructura `{ status, codigo, mensaje, errores, correlation_id }`. El campo `status` contiene el código HTTP; `codigo` corresponde a una categoría de error (`VALIDACION`, `NO_AUTENTICADO`, `PROHIBIDO`, `NO_ENCONTRADO`, `CONFLICTO`, `SERVICIO_NO_DISPONIBLE` o `ERROR_INTERNO`); `errores` contiene una lista de campos inválidos con su respectivo mensaje, y `correlation_id` permite dar seguimiento a la solicitud mediante `X-Correlation-ID`. Cuando los datos enviados no son válidos, se utiliza el mensaje general `"Los datos enviados no son válidos."`. Si falta la cabecera `X-User-*`, se responde con HTTP 401 y el mensaje `"Se requiere autenticación."`. Si el usuario no tiene los permisos necesarios, se responde con HTTP 403 y el mensaje `"No tiene permisos para realizar esta operación."`. |
| RG-08 | Paginación                | Las consultas de incidentes utilizan los parámetros `page` (desde 0) y `size` (por defecto 10 y máximo 100). Por compatibilidad, también aceptan `pagina` y `tamanio`. La respuesta incluye `contenido`, `page`, `size`, `total_elementos` y `total_paginas`. Las consultas de responsabilidades utilizan `pagina` (desde 0) y `tamano` (por defecto 10 y máximo 100).                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                        |
| RG-09 | Evidencias                | Los archivos adjuntos permitidos son imágenes JPG, PNG, WEBP, GIF, BMP, TIFF o HEIC, documentos PDF (`application/pdf`) y videos MP4 (`video/mp4`), según el tipo de contenido declarado. Cada incidente admite un máximo de cinco evidencias.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                          |

---

## F-01. Asignar brigadista a una ubicación

Este formulario permite asignar un brigadista a una ubicación específica dentro de un evento. La asignación puede realizarla un usuario autorizado desde el panel de gestión o el propio brigadista mediante la funcionalidad de autoasignación.

**Endpoints:**

- `POST /api/v1/asignaciones`: permite realizar asignaciones desde el panel de gestión.
- `POST /api/v1/asignaciones/mis-asignaciones`: permite que el brigadista se autoasigne. En esta operación, MS05 ignora el valor de `brigadistaId` enviado y utiliza la identidad obtenida mediante `X-User-Id`.

**Permisos requeridos:**

- Para la asignación desde el panel: `ms05:brigadier:assign_to_drill` o `ms05:brigadier:assign_to_real`, según el tipo de evento.
- Para la autoasignación: `ms05:panel:read_own`.

### Campos del formulario

| Campo           | Tipo   | Obligatorio    | Longitud o formato                            | Validaciones                                   | Mensaje de error                                  | Ejemplo válido                         |
| --------------- | ------ | -------------- | --------------------------------------------- | ---------------------------------------------- | ------------------------------------------------- | -------------------------------------- |
| `brigadistaId`  | String | Sí, en gestión | Identificador de usuario de MS01              | Debe corresponder a un brigadista activo.      | `El identificador del brigadista es obligatorio`  | `3f6c2a9e-8d1b-4c70-a1e5-6b2f9d4c7a10` |
| `eventoId`      | String | Sí             | Identificador del evento de MS06              | Debe corresponder a un evento válido.          | `El identificador del evento es obligatorio`      | `9a2d4e6f-1b3c-4d5e-8f70-a1b2c3d4e5f6` |
| `ubicacionId`   | String | Sí             | Identificador de MS02                         | La ubicación debe existir y estar activa.      | `El identificador de la ubicación es obligatorio` | `b7c1e9a4-2d3f-4a56-9e80-1f2a3b4c5d6e` |
| `tipoUbicacion` | String | Sí             | `EDIFICIO`, `ZONA_SEGURA` o `PUNTO_ENCUENTRO` | Debe coincidir con el tipo registrado en MS02. | `El tipo de ubicación es obligatorio`             | `ZONA_SEGURA`                          |
| `observaciones` | String | No             | Máximo 5000 caracteres                        | No debe superar la longitud permitida.         | `Las observaciones son demasiado largas`          | `Cubre la entrada norte`               |

### Respuestas y mensajes del formulario

| Situación                                                                      | Código HTTP | Respuesta o mensaje                                                          |
| ------------------------------------------------------------------------------ | ----------: | ---------------------------------------------------------------------------- |
| Asignación creada correctamente                                                |         201 | Devuelve la información de la asignación creada en estado `VIGENTE`.         |
| Ya existe una asignación vigente con los mismos datos                          |         409 | `Ya existe una asignación vigente para este brigadista, ubicación y evento.` |
| La ubicación no existe en MS02                                                 |         400 | `No existe una ubicación con identificador <id> y tipo <tipo> en MS02.`      |
| La ubicación está inactiva                                                     |         400 | `No se puede asignar un brigadista a una ubicación inactiva.`                |
| La ubicación no coincide con los datos enviados                                |         400 | `La ubicación consultada no corresponde con los datos solicitados.`          |
| Falta la identidad del Gateway (`X-User-Id`)                                   |         401 | `Se requiere autenticación.`                                                 |
| El usuario no tiene el permiso requerido o el tipo de evento no está permitido |         403 | `No tiene permisos para realizar esta operación.`                            |

### Asignaciones predeterminadas para MS06

MS06 también puede registrar asignaciones predeterminadas mediante el endpoint `POST /api/v1/asignaciones/predeterminadas`.

Este endpoint recibe una lista de objetos con los siguientes campos:

`{ brigadistaId, eventoId, ubicacionId, tipoUbicacion, observaciones }`

Los cuatro primeros campos son obligatorios. Los mensajes de validación correspondientes son:

- `El brigadista es obligatorio`.
- `El evento es obligatorio`.
- `La ubicación es obligatoria`.
- `El tipo de ubicación es obligatorio`.

El campo `observaciones` es opcional y admite hasta 5000 caracteres. Esta operación requiere la autoridad de servicio `SERVICE_MS06`.

---

## F-02. Modificar o retirar una asignación

Este formulario permite actualizar la ubicación y las observaciones de una asignación existente, o retirarla cuando ya no se necesita. Estas operaciones están sujetas al estado de la asignación y a las reglas de integridad de MS05.

**Endpoints:**

- `PUT /api/v1/asignaciones/{idAsignacion}`: modifica una asignación existente.
- `PATCH /api/v1/asignaciones/{idAsignacion}/retiro`: retira una asignación vigente. No requiere cuerpo en la solicitud.

**Permisos requeridos:** `ms05:brigadier:assign_to_drill` o `ms05:brigadier:assign_to_real`, según el tipo de evento.

### Campos de entrada

| Campo                               | Tipo   | Obligatorio | Longitud o formato                            | Validaciones                              | Mensaje de error                                  | Ejemplo válido                         |
| ----------------------------------- | ------ | ----------- | --------------------------------------------- | ----------------------------------------- | ------------------------------------------------- | -------------------------------------- |
| `idAsignacion` (ruta)               | UUID   | Sí          | UUID                                          | La asignación debe existir.               | `No existe la asignación con identificador: <id>` | `5d1e8a2b-4c3f-4a60-8b91-2c3d4e5f6a7b` |
| `ubicacionId` (solo modificación)   | String | Sí          | Identificador de MS02                         | La ubicación debe existir y estar activa. | `La ubicación es obligatoria`                     | `b7c1e9a4-2d3f-4a56-9e80-1f2a3b4c5d6e` |
| `tipoUbicacion` (solo modificación) | String | Sí          | `EDIFICIO`, `ZONA_SEGURA` o `PUNTO_ENCUENTRO` | Debe corresponder a un tipo válido.       | `El tipo de ubicación es obligatorio`             | `EDIFICIO`                             |
| `observaciones` (solo modificación) | String | No          | Máximo 5000 caracteres                        | No debe superar la longitud permitida.    | `Las observaciones son demasiado largas`          | `Cambio por cierre de acceso`          |

La operación de retiro mediante `PATCH /api/v1/asignaciones/{idAsignacion}/retiro` no requiere cuerpo.

### Respuestas y mensajes del formulario

| Situación                                                                           | Código HTTP | Respuesta o mensaje                                                                                                                             |
| ----------------------------------------------------------------------------------- | ----------: | ----------------------------------------------------------------------------------------------------------------------------------------------- |
| Asignación modificada o retirada                                                    |         200 | Devuelve la información actualizada de la asignación.                                                                                           |
| La asignación no existe                                                             |         404 | `No existe la asignación con identificador: <id>`                                                                                               |
| La asignación ya no está vigente                                                    |         409 | `La asignación no está vigente y no puede modificarse.`                                                                                         |
| Ya existe otra asignación idéntica                                                  |         409 | `Ya existe otra asignación vigente idéntica.`                                                                                                   |
| El brigadista es el único asignado a la ubicación y existen incidentes sin resolver |         409 | `No se puede retirar o modificar esta asignación porque existen incidentes sin resolver y el brigadista es el único asignado a esta ubicación.` |

---

## F-03. Panel de gestión y autoasignación por evento

Esta funcionalidad permite consultar y administrar las asignaciones de los brigadistas dentro de un evento. También permite que cada brigadista consulte los eventos disponibles y seleccione el edificio al que desea asignarse.

Las operaciones se realizan sobre eventos abiertos, es decir, aquellos que se encuentran en estado `PROGRAMADO` o `EN_EJECUCION`.

### Endpoints y permisos

**Panel de gestión:**

- `GET /api/v1/brigadistas/eventos`
- `GET /api/v1/brigadistas/eventos/{eventoId}/gestion`
- `PUT /api/v1/brigadistas/eventos/{eventoId}/brigadistas/{brigadistaId}/asignacion`

Permisos requeridos:

- `ms05:brigadier:assign_to_real` para eventos reales.
- `ms05:brigadier:assign_to_drill` para simulacros.

**Autoasignación del brigadista:**

- `GET /api/v1/brigadistas/autoasignacion`
- `PUT /api/v1/brigadistas/autoasignacion/eventos/{eventoId}`
- `DELETE /api/v1/brigadistas/autoasignacion/eventos/{eventoId}`

Permiso requerido: `ms05:panel:read_own`.

### Campos de entrada

| Campo                             | Tipo   | Obligatorio | Longitud o formato       | Validaciones                                                            | Mensaje de error                                                                 | Ejemplo válido                         |
| --------------------------------- | ------ | ----------- | ------------------------ | ----------------------------------------------------------------------- | -------------------------------------------------------------------------------- | -------------------------------------- |
| `eventoId` (ruta)                 | String | Sí          | Identificador del evento | El evento debe existir y estar en estado `PROGRAMADO` o `EN_EJECUCION`. | `El evento no está programado ni en ejecución; no admite cambios de asignación.` | `9a2d4e6f-1b3c-4d5e-8f70-a1b2c3d4e5f6` |
| `brigadistaId` (ruta, solo panel) | String | Sí          | Identificador de MS01    | El brigadista debe existir y estar en estado `ACTIVO` en MS05.          | `El brigadista es obligatorio.` / `No existe el brigadista indicado.`            | `3f6c2a9e-8d1b-4c70-a1e5-6b2f9d4c7a10` |
| `ubicacionId` (cuerpo del PUT)    | String | Sí          | Identificador de MS02    | La ubicación debe pertenecer al alcance del evento.                     | `El edificio es obligatorio.` / `El edificio indicado no hace parte del evento.` | `b7c1e9a4-2d3f-4a56-9e80-1f2a3b4c5d6e` |

La operación para retirar una autoasignación mediante `DELETE /api/v1/brigadistas/autoasignacion/eventos/{eventoId}` no requiere cuerpo.

### Respuestas y mensajes del formulario

| Situación                                             | Código HTTP | Respuesta o mensaje                                                                                           |
| ----------------------------------------------------- | ----------: | ------------------------------------------------------------------------------------------------------------- |
| Consulta o modificación realizada correctamente       |         200 | Devuelve la información del panel, los eventos o la asignación resultante.                                    |
| El edificio no pertenece al alcance del evento        |         400 | `El edificio indicado no hace parte del evento.`                                                              |
| El usuario no tiene permiso para el tipo de evento    |         403 | `No tiene permisos para realizar esta operación.`                                                             |
| El brigadista no existe                               |         404 | `No existe el brigadista indicado.`                                                                           |
| El evento está cerrado o en borrador                  |         409 | `El evento no está programado ni en ejecución; no admite cambios de asignación.`                              |
| El brigadista está inactivo al gestionar asignaciones |         409 | `No se pueden hacer asignaciones a brigadistas inactivos.`                                                    |
| El brigadista intenta autoasignarse sin estar activo  |         409 | `Para asignarte a un edificio primero debes activarte.`                                                       |
| El brigadista ya está asignado al mismo edificio      |         409 | `El brigadista ya está asignado a <nombre>.`                                                                  |
| Se intenta retirar una asignación que no existe       |         409 | `No tienes un edificio asignado en este evento.`                                                              |
| El brigadista es el único asignado al edificio        |         409 | `No es posible: el brigadista es el único asignado a <nombre> y el edificio no puede quedar sin brigadistas.` |
| MS01 no responde al consultar el panel                |         503 | Servicio de identidad no disponible.                                                                          |

---

## F-04. Buscar y activar brigadistas

Esta funcionalidad permite consultar los brigadistas registrados y actualizar su estado operativo. De esta manera, MS05 puede identificar quiénes están disponibles para recibir asignaciones y atender incidentes.

### Endpoints y permisos

| Endpoint                                          | Descripción                                                         | Permisos                                                                                                                                  |
| ------------------------------------------------- | ------------------------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------- |
| `GET /api/v1/brigadistas/buscar`                  | Busca brigadistas por nombre o apellido.                            | `ms05:brigadier:assign_to_drill` o `ms05:brigadier:assign_to_real`.                                                                       |
| `PATCH /api/v1/brigadistas/{brigadistaId}/estado` | Actualiza el estado de un brigadista. Responde HTTP 204 sin cuerpo. | `ms05:brigadier:assign_to_drill` o `ms05:brigadier:assign_to_real`.                                                                       |
| `PUT /api/v1/brigadistas/{brigadistaId}/estado`   | Actualiza el estado y devuelve la información resultante.           | Los permisos anteriores o `ms05:panel:read_own` cuando el brigadista modifica su propio estado y `brigadistaId` coincide con `X-User-Id`. |

### Campos de entrada

| Campo                                | Tipo   | Obligatorio | Longitud o formato               | Validaciones                                                                   | Mensaje de error                       | Ejemplo válido                         |
| ------------------------------------ | ------ | ----------- | -------------------------------- | ------------------------------------------------------------------------------ | -------------------------------------- | -------------------------------------- |
| `nombreBrigadista` (parámetro query) | String | No          | Nombre o apellido                | Si se omite o está vacío, devuelve todos los brigadistas, activos e inactivos. | No aplica.                             | `Gómez`                                |
| `brigadistaId` (ruta)                | String | Sí          | Identificador de usuario de MS01 | Debe corresponder a un brigadista registrado.                                  | `No existe el brigadista indicado`     | `3f6c2a9e-8d1b-4c70-a1e5-6b2f9d4c7a10` |
| `estado`                             | String | Sí          | `ACTIVO` o `INACTIVO`            | No distingue entre mayúsculas y minúsculas.                                    | `El estado debe ser ACTIVO o INACTIVO` | `ACTIVO`                               |

### Respuestas y mensajes del formulario

| Situación                              | Código HTTP | Respuesta o mensaje                                                                                                           |
| -------------------------------------- | ----------: | ----------------------------------------------------------------------------------------------------------------------------- |
| Búsqueda realizada correctamente       |         200 | Devuelve una lista de brigadistas con su estado en MS05. Si no existen coincidencias, devuelve una lista vacía.               |
| Estado actualizado mediante PUT        |         200 | Devuelve la información del brigadista con el nuevo estado.                                                                   |
| Estado actualizado mediante PATCH      |         204 | Operación realizada sin cuerpo de respuesta.                                                                                  |
| Falta el identificador o el estado     |         400 | `El identificador del brigadista es obligatorio` / `Debe indicar el nuevo estado` / `El estado del brigadista es obligatorio` |
| No se recibió la identidad del usuario |         401 | `Se requiere autenticación.` / `El usuario no está autenticado`                                                               |
| MS01 no responde                       |   502 / 503 | `No fue posible completar la operación con el servicio de identidad.`                                                         |

---

## F-05. Registrar incidente

Este formulario permite reportar un incidente asociado a un evento que se encuentra en ejecución. El registro incluye información sobre la prioridad, la ubicación y una descripción de lo ocurrido. Cuando corresponde, MS05 también identifica automáticamente al usuario que realiza el reporte.

**Endpoint:** `POST /api/v1/incidentes/eventos/{eventoId}`.

**Permiso requerido:** `ms05:incident:report`.

Este permiso está disponible para estudiantes, profesores, personal administrativo, brigadistas, coordinadores y organizadores. El rol auditor no cuenta con este permiso.

### Campos del formulario

| Campo             | Tipo   | Obligatorio | Longitud o formato                            | Validaciones                                                                                                                                                                                                  | Mensaje de error                                                                                                           | Ejemplo válido                         |
| ----------------- | ------ | ----------- | --------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------- | -------------------------------------- |
| `eventoId` (ruta) | String | Sí          | Identificador del evento de MS06              | El evento debe existir y estar en estado `EN_EJECUCION`.                                                                                                                                                      | `El identificador del evento es obligatorio.` / `No se pueden registrar incidentes porque el evento no está en ejecución.` | `9a2d4e6f-1b3c-4d5e-8f70-a1b2c3d4e5f6` |
| `nombreAfectado`  | String | No          | Máximo 150 caracteres                         | Solo se utiliza cuando un brigadista activo con `ms05:panel:read_own` registra un incidente en nombre de otra persona. En los demás casos, MS05 utiliza automáticamente el `nombreCompleto` obtenido de MS01. | `El nombre del afectado no puede superar 150 caracteres`                                                                   | `Laura Méndez`                         |
| `prioridadId`     | Número | Sí          | Identificador del catálogo                    | Debe corresponder a una prioridad registrada: `EMERGENCIA`, `URGENCIA_MODERADA` o `POCA_URGENCIA`.                                                                                                            | `La prioridad es obligatoria` / `La prioridad seleccionada no existe.`                                                     | `1`                                    |
| `ubicacionId`     | String | Sí          | Identificador de MS02                         | La ubicación debe existir y estar activa.                                                                                                                                                                     | `La ubicación es obligatoria`                                                                                              | `b7c1e9a4-2d3f-4a56-9e80-1f2a3b4c5d6e` |
| `tipoUbicacion`   | String | Sí          | `EDIFICIO`, `ZONA_SEGURA` o `PUNTO_ENCUENTRO` | Debe existir en el catálogo de MS05 y coincidir con el tipo registrado en MS02.                                                                                                                               | `El tipo de ubicación es obligatorio` / `El tipo de ubicación no está registrado.`                                         | `EDIFICIO`                             |
| `descripcion`     | String | No          | Máximo 1000 caracteres                        | No debe superar la longitud permitida.                                                                                                                                                                        | `La descripción no puede superar 1000 caracteres`                                                                          | `Persona con mareo en el segundo piso` |

### Respuestas y mensajes del formulario

| Situación                                                      | Código HTTP | Respuesta o mensaje                                                                                                                                                                |
| -------------------------------------------------------------- | ----------: | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Incidente registrado correctamente                             |         200 | Devuelve la información del incidente. Su estado inicial será `ASIGNADO` si existe un brigadista activo con asignación vigente en la ubicación; de lo contrario, será `REPORTADO`. |
| La ubicación no existe o está inactiva                         |         400 | `La ubicación no existe o no se encuentra activa.`                                                                                                                                 |
| No se pudo identificar al usuario                              |         401 | `No se pudo identificar al usuario autenticado.`                                                                                                                                   |
| El usuario no tiene permiso para reportar                      |         403 | `El usuario no tiene permiso para registrar incidentes (ms05:incident:report).`                                                                                                    |
| Brigadista no registrado o inactivo al reportar por un tercero |         403 | `El usuario no está registrado como brigadista en MS05.` / `El brigadista debe estar activo para realizar esta operación.`                                                         |
| Usuario inexistente o inactivo en MS01                         |         403 | `El usuario no existe en MS01.` / `El usuario se encuentra inactivo.`                                                                                                              |
| MS01 no devuelve el nombre completo del usuario                |         502 | `No fue posible completar la operación con el servicio de identidad.`                                                                                                              |

---

## F-06. Editar y eliminar incidente

Esta funcionalidad permite actualizar la descripción de un incidente existente o eliminarlo de manera lógica. Las operaciones están condicionadas por el estado del incidente, la identidad del usuario y los permisos que tenga asignados.

### Endpoints y permisos

| Endpoint                                                     | Descripción                                                      | Permisos                                                                                                                                            |
| ------------------------------------------------------------ | ---------------------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------- |
| `PUT /api/v1/incidentes/eventos/{eventoId}/{incidenteId}`    | Actualiza la información de un incidente.                        | `ms05:incident:update` o `ms05:incident:report`. El usuario debe ser el propietario del incidente o un brigadista activo con `ms05:panel:read_own`. |
| `DELETE /api/v1/incidentes/eventos/{eventoId}/{incidenteId}` | Realiza la eliminación lógica del incidente. No requiere cuerpo. | `ms05:assistance:assign_to_drill` para simulacros o `ms05:assistance:assign_to_real` para eventos reales.                                           |

### Campos de entrada

| Campo                    | Tipo   | Obligatorio | Longitud o formato    | Validaciones                                               | Mensaje de error                                                            | Ejemplo válido                         |
| ------------------------ | ------ | ----------- | --------------------- | ---------------------------------------------------------- | --------------------------------------------------------------------------- | -------------------------------------- |
| `incidenteId` (ruta)     | UUID   | Sí          | UUID                  | El incidente debe existir y pertenecer al evento indicado. | `El incidente no existe.` / `El incidente no pertenece al evento indicado.` | `c4d5e6f7-a8b9-4c0d-9e1f-2a3b4c5d6e7f` |
| `descripcion` (solo PUT) | String | No          | Máximo 500 caracteres | No debe superar la longitud permitida.                     | `La descripción no puede superar los 500 caracteres.`                       | `Se confirma esguince de tobillo`      |

### Respuestas y mensajes del formulario

| Situación                                             | Código HTTP | Respuesta o mensaje                                                                                                                                                                           |
| ----------------------------------------------------- | ----------: | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Incidente actualizado                                 |         200 | Devuelve la información del incidente actualizado.                                                                                                                                            |
| Incidente eliminado lógicamente                       |         204 | Operación realizada sin cuerpo de respuesta.                                                                                                                                                  |
| Incidente inexistente o perteneciente a otro evento   |         404 | `El incidente no existe.` / `El incidente no pertenece al evento indicado.` / `No se encontró el incidente o ya fue eliminado.`                                                               |
| Se intenta editar un incidente cerrado                |         409 | `No se puede editar un incidente cerrado.`                                                                                                                                                    |
| Se intenta eliminar un incidente que no está resuelto |         409 | `Solo se pueden eliminar incidentes en estado RESUELTO.`                                                                                                                                      |
| El usuario no tiene permiso para editar o eliminar    |         403 | `El usuario no tiene permisos para editar incidentes.` / `Solo un brigadista activo puede editar incidentes de otros usuarios.` / `No tiene permiso para eliminar incidentes de este evento.` |

---

## F-07. Gestionar la atención de un incidente

Esta funcionalidad permite realizar el seguimiento de un incidente desde que se asigna un responsable hasta que se resuelve y se cierra. También permite consultar el historial de cambios para conocer las acciones realizadas durante su atención.

**Ruta base:** `/api/v1/incidentes/eventos/{eventoId}/{incidenteId}`.

### Acciones, endpoints y permisos

| Acción                                                                         | Endpoint                 | Permiso requerido                                                                                         |
| ------------------------------------------------------------------------------ | ------------------------ | --------------------------------------------------------------------------------------------------------- |
| Asignar o reasignar responsable y enviar notificación por correo mediante MS03 | `PATCH /responsable`     | `ms05:assistance:assign_to_drill` para simulacros o `ms05:assistance:assign_to_real` para eventos reales. |
| Iniciar la atención (`ASIGNADO` → `EN_ATENCION`)                               | `POST /iniciar-atencion` | `ms05:assistance:update_status`. Solo puede realizarla el brigadista responsable.                         |
| Resolver el incidente (`EN_ATENCION` → `RESUELTO`)                             | `POST /resolver`         | `ms05:assistance:update_status`. Solo puede realizarla el brigadista responsable.                         |
| Cerrar el incidente (`RESUELTO` → `CERRADO`)                                   | `POST /cerrar`           | `ms05:assistance:assign_to_drill` para simulacros o `ms05:assistance:assign_to_real` para eventos reales. |
| Consultar el historial cronológico                                             | `GET /historial`         | `ms05:incident:read`.                                                                                     |

### Campos de entrada

| Campo                        | Tipo   | Obligatorio | Longitud o formato               | Validaciones                                                                                                                                   | Mensaje de error                                                                                             | Ejemplo válido                         |
| ---------------------------- | ------ | ----------- | -------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------ | -------------------------------------- |
| `responsableId` (asignación) | String | Sí          | Identificador de usuario de MS01 | El responsable debe tener el rol `BRIGADISTA` en MS01, estar activo en MS05 y contar con una asignación vigente en la ubicación del incidente. | `El identificador del brigadista es obligatorio.` / `El identificador del responsable es obligatorio.`       | `3f6c2a9e-8d1b-4c70-a1e5-6b2f9d4c7a10` |
| `observaciones` (asignación) | String | No          | Texto libre                      | Se almacena en el historial de cambios.                                                                                                        | No aplica.                                                                                                   | `Reasignado por cambio de turno`       |
| `observaciones` (resolución) | String | Sí          | Máximo 1000 caracteres           | No puede estar vacío ni superar la longitud permitida.                                                                                         | `Las observaciones de resolución son obligatorias.` / `Las observaciones no pueden superar 1000 caracteres.` | `Se trasladó al afectado a enfermería` |

Las operaciones para iniciar la atención (`POST /iniciar-atencion`) y cerrar el incidente (`POST /cerrar`) no requieren cuerpo en la solicitud.

### Respuestas y mensajes del formulario

| Situación                                                               | Código HTTP | Respuesta o mensaje                                                                                                                                                                                                                                             |
| ----------------------------------------------------------------------- | ----------: | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Cambio de estado realizado correctamente                                |         200 | Devuelve el estado anterior, el nuevo estado, el responsable y la fecha del cambio.                                                                                                                                                                             |
| Transición de estado no permitida                                       |         409 | `Transición inválida. El incidente debe estar en estado <esperado> y actualmente está en <actual>.`                                                                                                                                                             |
| Se intenta reasignar un incidente resuelto o cerrado                    |         409 | `No se puede reasignar un incidente resuelto o cerrado.`                                                                                                                                                                                                        |
| El brigadista ya es el responsable del incidente                        |         409 | `El brigadista indicado ya es el responsable del incidente.`                                                                                                                                                                                                    |
| El brigadista no tiene asignación vigente en la ubicación del incidente |         409 | `El brigadista no tiene una asignación vigente en la ubicación del incidente.`                                                                                                                                                                                  |
| El responsable no tiene el rol requerido o está inactivo                |         403 | `El responsable debe tener el rol BRIGADISTA en MS01.` / `El usuario no está registrado como brigadista en MS05.` / `El brigadista debe estar activo en MS05.`                                                                                                  |
| Un usuario diferente al responsable intenta atender el incidente        |         403 | `Solo el brigadista responsable puede atender este incidente.`                                                                                                                                                                                                  |
| El usuario no tiene el permiso requerido                                |         403 | `No tiene permiso para cambiar el estado de atención.` / `La operación requiere el permiso ms05:assistance:assign_to_drill.` / `La operación requiere el permiso ms05:assistance:assign_to_real.` / `El usuario no tiene permisos para consultar el historial.` |
| El incidente no existe o no pertenece al evento                         |         404 | `El incidente no existe.` / `El incidente no pertenece al evento indicado.`                                                                                                                                                                                     |

---

## F-08. Adjuntar y consultar evidencias

Esta funcionalidad permite adjuntar archivos que sirven como soporte del incidente y consultar las evidencias que ya se encuentran registradas. Los archivos pueden ser imágenes, documentos PDF o videos MP4.

### Endpoints y permisos

- `POST /api/v1/incidentes/eventos/{eventoId}/{incidenteId}/evidencias`: permite cargar una evidencia mediante `multipart/form-data`.
- `GET /api/v1/incidentes/eventos/{eventoId}/{incidenteId}/evidencias`: permite consultar las evidencias asociadas al incidente.

**Permisos requeridos:**

- Para cargar evidencias: `ms05:incident:report`. El servicio también exige que el usuario sea el propietario del incidente o un brigadista activo.
- Para consultar evidencias: `ms05:incident:read`.

### Campos del formulario

| Campo     | Tipo                      | Obligatorio | Longitud o formato                                                            | Validaciones                                                   | Mensaje de error                                                                                                                                    | Ejemplo válido    |
| --------- | ------------------------- | ----------- | ----------------------------------------------------------------------------- | -------------------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------- | ----------------- |
| `archivo` | Archivo (`MultipartFile`) | Sí          | Imagen (JPG, PNG, WEBP, GIF, BMP, TIFF o HEIC), `application/pdf` o `video/mp4`. Nombre de hasta 255 caracteres. | El archivo debe contener información y tener un nombre válido. | `Debe seleccionar un archivo válido.` / `El archivo debe tener un nombre válido.` / `El formato no está permitido. Solo se aceptan PNG, PDF y MP4.` | `foto-lesion.png` |

### Respuestas y mensajes del formulario

| Situación                                                 | Código HTTP | Respuesta o mensaje                                                                                                                       |
| --------------------------------------------------------- | ----------: | ----------------------------------------------------------------------------------------------------------------------------------------- |
| Evidencia cargada correctamente                           |         201 | Devuelve los metadatos de la evidencia creada.                                                                                            |
| Consulta de evidencias realizada                          |         200 | Devuelve la lista de evidencias asociadas al incidente.                                                                                   |
| El incidente ya tiene cinco evidencias                    |         400 | `Un incidente no puede tener más de 5 evidencias.`                                                                                        |
| El incidente está cerrado                                 |         400 | `No se pueden adjuntar evidencias a un incidente cerrado.`                                                                                |
| El incidente no existe o no pertenece al evento           |         404 | `El incidente indicado no existe.` / `El incidente no pertenece al evento indicado.`                                                      |
| El usuario no tiene permiso o el brigadista está inactivo |         403 | `No tiene permisos para realizar esta operación.` / `Solo un brigadista activo puede adjuntar evidencias a incidentes de otros usuarios.` |
| Error al guardar el archivo                               |         500 | `No fue posible almacenar la evidencia.`                                                                                                  |

---

## F-09. Consultas, filtros y auditoría

Esta funcionalidad permite consultar incidentes, asignaciones, responsabilidades y registros de auditoría. También permite aplicar filtros para encontrar información específica y utilizar paginación para organizar los resultados.

### Endpoints y permisos

**Consulta de incidentes:**

- `GET /api/v1/incidentes/eventos/{eventoId}/consulta`: permite consultar los incidentes de un evento. Requiere `ms05:incident:read` o `ms05:incident:manage_own` cuando se utiliza el filtro `soloPropios=true`.
- `GET /api/v1/incidentes/eventos/{eventoId}/{incidenteId}`: permite consultar el detalle de un incidente. Requiere `ms05:incident:read`.

**Consulta de responsabilidades:**

- `GET /api/v1/responsabilidades`: permite consultar responsabilidades con los permisos `ms05:brigadier:assign_to_drill`, `ms05:brigadier:assign_to_real` o `ms05:incident:read`.
- `GET /api/v1/responsabilidades/mias`: permite que el brigadista consulte sus propias responsabilidades. Requiere `ms05:panel:read_own`.

**Consulta de asignaciones:**

- `GET /api/v1/asignaciones/{idAsignacion}`: consulta una asignación específica.
- `GET /api/v1/asignaciones/evento/{eventoId}`: consulta las asignaciones de un evento.
- `GET /api/v1/asignaciones/brigadista/{brigadistaId}`: consulta las asignaciones de un brigadista.

**Consulta de auditoría local de MS05:**

- `GET /api/v1/bei/auditoria`: consulta los registros de auditoría.
- `GET /api/v1/bei/auditoria/eventos/{eventoId}`: consulta la auditoría asociada a un evento.
- `GET /api/v1/bei/auditoria/recursos/{recursoTipo}/{recursoId}`: consulta la auditoría asociada a un recurso específico.

Las consultas de auditoría requieren `ms08:audit_drill:read` o `ms08:audit_real_event:read`, según el tipo de evento.

### Parámetros de consulta y filtros

| Campo                                                               | Tipo     | Obligatorio | Longitud o formato                                                            | Validaciones                                                                                                                                                                                               | Mensaje de error                               | Ejemplo válido                         |
| ------------------------------------------------------------------- | -------- | ----------- | ----------------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ---------------------------------------------- | -------------------------------------- |
| `prioridad` (incidentes)                                            | String   | No          | `EMERGENCIA`, `URGENCIA_MODERADA` o `POCA_URGENCIA`                           | Si el valor no existe en el catálogo, el filtro se ignora.                                                                                                                                                 | No aplica.                                     | `EMERGENCIA`                           |
| `estado` (incidentes / responsabilidades)                           | String   | No          | Estado del incidente o de la asignación                                       | Se utiliza para filtrar los resultados según el estado indicado.                                                                                                                                           | No aplica.                                     | `EN_ATENCION`                          |
| `ubicacionId`                                                       | String   | No          | Identificador de MS02                                                         | Permite filtrar los resultados por ubicación.                                                                                                                                                              | No aplica.                                     | `b7c1e9a4-2d3f-4a56-9e80-1f2a3b4c5d6e` |
| `ordenarPor` (incidentes)                                           | String   | No          | `prioridad`, `estado`, `ubicacion`, `antiguedad`, `fechaReporte` o `edificio` | Si se envía un valor diferente, se utiliza `fechaReporte` como criterio predeterminado. Cuando se utiliza `edificio`, MS05 resuelve los nombres de las ubicaciones en MS02 antes de aplicar la paginación. | No aplica.                                     | `prioridad`                            |
| `direccion` (incidentes)                                            | String   | No          | `asc` o `desc`                                                                | Por defecto utiliza `asc`.                                                                                                                                                                                 | No aplica.                                     | `desc`                                 |
| `soloPropios` (incidentes)                                          | Booleano | No          | `true` o `false`                                                              | Por defecto es `false`. Cuando se establece en `true`, devuelve únicamente los incidentes reportados por el usuario autenticado.                                                                           | No aplica.                                     | `true`                                 |
| `page` / `pagina`                                                   | Entero   | No          | Mayor o igual a 0                                                             | Por defecto es `0`.                                                                                                                                                                                        | `Los parámetros de paginación no son válidos.` | `0`                                    |
| `size` / `tamanio` (incidentes) / `tamano` (responsabilidades)      | Entero   | No          | Entre 1 y 100                                                                 | Por defecto es `10`, con un máximo de `100`.                                                                                                                                                               | `Los parámetros de paginación no son válidos.` | `10`                                   |
| `eventoId`, `nombreBrigadista`, `tipoUbicacion` (responsabilidades) | String   | No          | Según el identificador o valor correspondiente                                | Se pueden combinar en la consulta `GET /api/v1/responsabilidades`.                                                                                                                                         | No aplica.                                     | `Gómez`                                |

### Respuestas y mensajes del formulario

| Situación                                                  | Código HTTP | Respuesta o mensaje                                                                                                                                                  |
| ---------------------------------------------------------- | ----------: | -------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Consulta realizada correctamente, con o sin resultados     |         200 | Devuelve una página de resultados. Si no existen coincidencias, se devuelve un contenido vacío para incidentes o una lista vacía para responsabilidades y auditoría. |
| Parámetros inválidos o faltantes                           |         400 | `El identificador del evento es obligatorio.` / `El identificador del brigadista es obligatorio.` / `La solicitud tiene un formato inválido.`                        |
| No se recibió la identidad del usuario                     |         401 | `Se requiere autenticación.` / `No se pudo identificar al usuario autenticado.`                                                                                      |
| El usuario no tiene permiso para consultar la información  |         403 | `No tiene permisos para realizar esta operación.` / `El usuario no tiene permisos para consultar incidentes.` / `No tiene permisos para consultar este incidente.`   |
| MS02 no está disponible al ordenar incidentes por edificio |         503 | `El servicio de ubicaciones no está disponible.`                                                                                                                     |
| Ocurre un error inesperado durante la consulta             |         500 | `Ocurrió un error interno al procesar la solicitud.`                                                                                                                 |

---

## Consideraciones finales

Los formularios y operaciones descritos en este documento representan las entradas que recibe MS05 para gestionar brigadistas, asignaciones, incidentes, evidencias y consultas.

Es importante tener en cuenta que:

- El backend es responsable de validar los datos y autorizar cada operación, independientemente de las validaciones que realice el frontend.
- La identidad y los permisos del usuario se obtienen mediante las cabeceras proporcionadas por el API Gateway.
- Las operaciones que dependen de información de otros microservicios realizan las validaciones correspondientes con MS01, MS02, MS03 o MS06.
- Los incidentes siguen un flujo de atención controlado por sus estados: `REPORTADO`, `ASIGNADO`, `EN_ATENCION`, `RESUELTO` y `CERRADO`.
- Las evidencias deben cumplir los formatos permitidos y el límite máximo establecido por incidente.
- Los mensajes de respuesta documentados permiten que el frontend informe al usuario sobre el resultado de sus operaciones de una manera clara y consistente.
