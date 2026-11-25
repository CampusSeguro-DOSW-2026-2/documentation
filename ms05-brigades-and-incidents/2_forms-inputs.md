# Especificación de Formularios y Entradas

Define los campos, validaciones y mensajes de cada formulario de MS05 (brigadistas e incidentes). Las validaciones están en el servicio (MS05) y el frontend las repite solo para dar una respuesta rápida. Los mensajes son los que devuelve la API, en español.

## Índice de Contenido
1. [Reglas generales de entrada](#reglas-generales-de-entrada)
2. [F-01 Asignar brigadista a una ubicación](#f-01-asignar-brigadista-a-una-ubicación)
3. [F-02 Modificar o retirar una asignación](#f-02-modificar-o-retirar-una-asignación)
4. [F-03 Buscar y activar brigadistas](#f-03-buscar-y-activar-brigadistas)
5. [F-04 Registrar incidente](#f-04-registrar-incidente)
6. [F-05 Editar incidente](#f-05-editar-incidente)
7. [F-06 Gestionar la atención de un incidente](#f-06-gestionar-la-atención-de-un-incidente)
8. [F-07 Adjuntar evidencia](#f-07-adjuntar-evidencia)
9. [F-08 Consultas y filtros](#f-08-consultas-y-filtros)

---

## Reglas generales de entrada

| ID | Regla |
|---|---|
| RG-01 | **Identidad:** el servicio no lee el usuario del cuerpo ni de la URL. El Gateway valida el JWT e inyecta `X-User-Id` y `X-User-Role`; con ellos MS05 decide quién actúa. |
| RG-02 | **Autorización por rol:** MS05 autoriza por el rol de la cabecera `X-User-Role` (`COORDINATOR`, `BRIGADISTA`, `AUDITOR` y `SERVICE_MS06`), no por permisos `ms05:*`. Los roles de negocio de MS01 se corresponden así: COORDINADOR/ORGANIZADOR → `COORDINATOR`, BRIGADISTA → `BRIGADISTA`, AUDITOR → `AUDITOR`. |
| RG-03 | **Brigadista activo:** quien atiende incidentes o se asigna debe existir en MS01, estar activo allí y estar `ACTIVO` en MS05. |
| RG-04 | **Evento en ejecución:** los incidentes solo se registran mientras el evento está en ejecución (lo informa MS06). |
| RG-05 | **Ubicación:** `ubicacionId` es el identificador de MS02 y `tipoUbicacion` es `EDIFICIO`, `ZONA_SEGURA` o `PUNTO_ENCUENTRO`. La ubicación debe existir y estar activa. |
| RG-06 | **Identificadores:** los identificadores de incidentes y asignaciones son UUID. |
| RG-07 | **Formato de error:** `{ fecha, codigo, error, mensaje }`, donde `codigo` es el código HTTP numérico. Las validaciones de campo devuelven 400 con el mensaje del primer campo inválido. |
| RG-08 | **Paginación:** `pagina` desde 0; `tamanio` (incidentes) o `tamano` (responsabilidades), por defecto 10 y máximo 100. |
| RG-09 | **Archivos:** solo `image/png`, `application/pdf` y `video/mp4`; máximo 5 por incidente. |

---

## F-01 Asignar brigadista a una ubicación

**Endpoints:** `POST /api/v1/asignaciones` (el coordinador asigna a un brigadista) y `POST /api/v1/asignaciones/mis-asignaciones` (el brigadista se asigna a sí mismo; el servicio ignora `brigadistaId` y usa el usuario del Gateway). · **Rol:** `COORDINATOR` / `BRIGADISTA`.

| Campo | Tipo | Obligatorio | Longitud / formato | Reglas de validación | Mensaje de error | Ejemplo válido |
|---|---|---|---|---|---|---|
| `brigadistaId` | string | Sí (coordinador) | Identificador de usuario de MS01 | Debe ser un brigadista activo. | `El identificador del brigadista es obligatorio` | `3f6c2a9e-8d1b-4c70-a1e5-6b2f9d4c7a10` |
| `eventoId` | string | Sí | Identificador del evento de MS06 | — | `El identificador del evento es obligatorio` | `9a2d4e6f-1b3c-4d5e-8f70-a1b2c3d4e5f6` |
| `ubicacionId` | string | Sí | Identificador de MS02 | La ubicación debe existir y estar activa. | `El identificador de la ubicación es obligatorio` | `b7c1e9a4-2d3f-4a56-9e80-1f2a3b4c5d6e` |
| `tipoUbicacion` | string | Sí | `EDIFICIO`, `ZONA_SEGURA` o `PUNTO_ENCUENTRO` | Debe coincidir con el tipo de la ubicación en MS02. | `El tipo de ubicación es obligatorio` | `ZONA_SEGURA` |
| `observaciones` | string | No | Hasta 5000 caracteres | — | `Las observaciones son demasiado largas` | `Cubre la entrada norte` |

**Mensajes globales de formulario**

| Situación | Código HTTP | Mensaje |
|---|---|---|
| Asignación creada | 201 | Cuerpo con la asignación (estado `VIGENTE`). |
| Ya hay una asignación vigente igual | 409 | `Ya existe una asignación vigente para este brigadista, ubicación y evento.` |
| La ubicación no existe en MS02 | 400 | `No existe una ubicación con identificador <id> y tipo <tipo> en MS02.` |
| La ubicación está inactiva | 400 | `No se puede asignar un brigadista a una ubicación inactiva.` |
| La ubicación no corresponde con los datos enviados | 400 | `La ubicación consultada no corresponde con los datos solicitados.` |
| Falta el usuario del Gateway | 400 | `No se pudo identificar al usuario que realiza la operación.` |
| El brigadista pide asignarse sin ser brigadista | 403 | `Esta operación está disponible únicamente para brigadistas.` |
| Rol sin acceso | 403 | Acceso denegado. |

**Asignaciones predeterminadas (MS06):** `POST /api/v1/asignaciones/predeterminadas` recibe una lista de `{ brigadistaId, eventoId, ubicacionId, tipoUbicacion }` (todos obligatorios: `El brigadista es obligatorio`, `El evento es obligatorio`, `La ubicación es obligatoria`, `El tipo de ubicación es obligatorio`). Solo la puede llamar el rol de servicio `SERVICE_MS06`; con otro rol responde 403 `Esta operación está reservada para el servicio autorizado de MS06.`

---

## F-02 Modificar o retirar una asignación

**Endpoints:** `PUT /api/v1/asignaciones/{idAsignacion}` y `PATCH /api/v1/asignaciones/{idAsignacion}/retiro` · **Rol:** `COORDINATOR`.

| Campo | Tipo | Obligatorio | Longitud / formato | Reglas de validación | Mensaje de error | Ejemplo válido |
|---|---|---|---|---|---|---|
| `idAsignacion` (ruta) | UUID | Sí | UUID | La asignación debe existir. | `No existe la asignación con identificador: <id>` | `5d1e8a2b-4c3f-4a60-8b91-2c3d4e5f6a7b` |
| `ubicacionId` (solo modificar) | string | Sí | Identificador de MS02 | Existente y activa. | `La ubicación es obligatoria` | `b7c1e9a4-2d3f-4a56-9e80-1f2a3b4c5d6e` |
| `tipoUbicacion` (solo modificar) | string | Sí | `EDIFICIO`, `ZONA_SEGURA` o `PUNTO_ENCUENTRO` | — | `El tipo de ubicación es obligatorio` | `EDIFICIO` |
| `observaciones` (solo modificar) | string | No | Hasta 5000 caracteres | — | `Las observaciones son demasiado largas` | `Cambio por cierre de acceso` |

El retiro no lleva cuerpo.

**Mensajes globales de formulario**

| Situación | Código HTTP | Mensaje |
|---|---|---|
| Asignación modificada o retirada | 200 | Cuerpo con la asignación actualizada. |
| La asignación ya no está vigente | 409 | `La asignación no está vigente y no puede modificarse.` |
| Ya existe otra igual | 409 | `Ya existe otra asignación vigente idéntica.` |
| El brigadista es el único de la ubicación y hay incidentes sin resolver | 409 | `No se puede retirar o modificar esta asignación porque existen incidentes sin resolver y el brigadista es el único asignado a esta ubicación.` |

---

## F-03 Buscar y activar brigadistas

**Endpoints:** `GET /api/v1/brigadistas/buscar` y `PATCH /api/v1/brigadistas/{brigadistaId}/estado` (también responde a `PUT`) · **Rol:** `COORDINATOR`.

| Campo | Tipo | Obligatorio | Longitud / formato | Reglas de validación | Mensaje de error | Ejemplo válido |
|---|---|---|---|---|---|---|
| `nombreBrigadista` (parámetro) | string | Sí | Nombre o apellido | No puede estar vacío. | `Debe ingresar un nombre o apellido para buscar` | `Gómez` |
| `brigadistaId` (ruta) | string | Sí | Identificador de usuario de MS01 | El brigadista debe estar registrado en MS05. | `No existe el brigadista indicado` | `3f6c2a9e-8d1b-4c70-a1e5-6b2f9d4c7a10` |
| `estado` | string | Sí | `ACTIVO` o `INACTIVO` | Se acepta en cualquier combinación de mayúsculas y minúsculas. | `El estado debe ser ACTIVO o INACTIVO` | `ACTIVO` |

**Mensajes globales de formulario**

| Situación | Código HTTP | Mensaje |
|---|---|---|
| Búsqueda correcta | 200 | Lista de brigadistas con su estado en MS05 (sin resultados, lista vacía). |
| Estado cambiado (`PATCH`) | 204 | Sin cuerpo. |
| Falta el identificador o el estado | 400 | `El identificador del brigadista es obligatorio` / `Debe indicar el nuevo estado` / `El estado del brigadista es obligatorio` |
| MS01 no responde | 503 | `No fue posible consultar los brigadistas en MS01` |

---

## F-04 Registrar incidente

**Endpoint:** `POST /api/v1/eventos/{eventoId}/incidentes` · **Rol:** cualquier usuario autenticado con un rol autorizado: estudiante, profesor, administrativo, coordinador, organizador, admin o brigadista activo. El auditor no puede.

| Campo | Tipo | Obligatorio | Longitud / formato | Reglas de validación | Mensaje de error | Ejemplo válido |
|---|---|---|---|---|---|---|
| `eventoId` (ruta) | string | Sí | Identificador del evento de MS06 | El evento debe estar en ejecución. | `No se pueden registrar incidentes porque el evento no está en ejecución.` | `9a2d4e6f-1b3c-4d5e-8f70-a1b2c3d4e5f6` |
| `nombreAfectado` | string | Sí | Hasta 150 caracteres | — | `El nombre del afectado no puede superar 150 caracteres` | `Laura Méndez` |
| `prioridadId` | número | Sí | Identificador del catálogo | Debe existir (`EMERGENCIA`, `URGENCIA_MODERADA` o `POCA_URGENCIA`). | `La prioridad es obligatoria` / `La prioridad seleccionada no existe.` | `1` |
| `ubicacionId` | string | Sí | Identificador de MS02 | La ubicación debe existir y estar activa. | `La ubicación es obligatoria` | `b7c1e9a4-2d3f-4a56-9e80-1f2a3b4c5d6e` |
| `tipoUbicacion` | string | Sí | `EDIFICIO`, `ZONA_SEGURA` o `PUNTO_ENCUENTRO` | Debe estar registrado. | `El tipo de ubicación es obligatorio` / `El tipo de ubicación no está registrado.` | `EDIFICIO` |
| `descripcion` | string | No | Hasta 1000 caracteres | — | `La descripción no puede superar 1000 caracteres` | `Persona con mareo en el segundo piso` |

**Mensajes globales de formulario**

| Situación | Código HTTP | Mensaje |
|---|---|---|
| Incidente creado | 201 | Cuerpo con el incidente (estado `ASIGNADO` si hay brigadistas vigentes en la ubicación; si no, `REPORTADO`). |
| La ubicación no existe o está inactiva | 400 | `La ubicación no existe o no se encuentra activa.` |
| Rol auditor | 403 | `Los usuarios con rol auditor no pueden registrar incidentes.` |
| Rol sin autorización | 403 | `El usuario no tiene un rol autorizado para registrar incidentes.` |
| El brigadista no está registrado o no está activo | 403 | `El usuario no está registrado como brigadista en MS05.` / `El brigadista debe estar activo para realizar esta operación.` |
| Usuario inexistente o inactivo en MS01 | 403 | `El usuario no existe en MS01.` / `El usuario se encuentra inactivo.` |

---

## F-05 Editar incidente

**Endpoint:** `PUT /api/v1/eventos/{eventoId}/incidentes/{incidenteId}` · **Rol:** `COORDINATOR` (y brigadista activo según las reglas del servicio).

| Campo | Tipo | Obligatorio | Longitud / formato | Reglas de validación | Mensaje de error | Ejemplo válido |
|---|---|---|---|---|---|---|
| `incidenteId` (ruta) | UUID | Sí | UUID | Debe existir y pertenecer al evento. | `El incidente no existe.` / `El incidente no pertenece al evento indicado.` | `c4d5e6f7-a8b9-4c0d-9e1f-2a3b4c5d6e7f` |
| `descripcion` | string | No | Hasta 500 caracteres | — | `La descripción no puede superar los 500 caracteres.` | `Se confirma esguince de tobillo` |

**Mensajes globales de formulario**

| Situación | Código HTTP | Mensaje |
|---|---|---|
| Incidente editado | 200 | Cuerpo con el incidente. |
| El incidente está cerrado | 409 | `No se puede editar un incidente cerrado.` |
| Rol auditor | 403 | `Los usuarios con rol auditor no pueden editar incidentes.` |
| Sin permiso sobre ese incidente | 403 | `El usuario no tiene permisos para editar este incidente.` |
| Brigadista no activo | 403 | `Solo un brigadista activo puede editar incidentes de otros usuarios.` |

**Eliminar:** `DELETE /api/v1/eventos/{eventoId}/incidentes/{incidenteId}` (`COORDINATOR`) hace una eliminación lógica y no lleva cuerpo.

---

## F-06 Gestionar la atención de un incidente

**Endpoints** (ruta base `/api/v1/eventos/{eventoId}/incidentes/{incidenteId}`):

| Acción | Endpoint | Rol |
|---|---|---|
| Asignar o reasignar responsable | `PATCH /responsable` | `COORDINATOR` |
| Iniciar la atención | `POST /iniciar-atencion` | `BRIGADISTA` |
| Resolver | `POST /resolver` | `BRIGADISTA` |
| Cerrar | `POST /cerrar` | `COORDINATOR` |
| Ver el historial | `GET /historial` | `COORDINATOR`, `BRIGADISTA`, `AUDITOR` |

| Campo | Tipo | Obligatorio | Longitud / formato | Reglas de validación | Mensaje de error | Ejemplo válido |
|---|---|---|---|---|---|---|
| `responsableId` (asignar) | string | Sí | Identificador de usuario de MS01 | Debe ser un brigadista activo con asignación vigente en la ubicación del incidente. | `El identificador del brigadista es obligatorio.` | `3f6c2a9e-8d1b-4c70-a1e5-6b2f9d4c7a10` |
| `observaciones` (asignar) | string | No | Hasta 1000 caracteres | — | — | `Reasignado por cambio de turno` |
| `observaciones` (resolver) | string | Sí | Hasta 1000 caracteres | No puede estar vacío. | `Las observaciones de resolución son obligatorias.` / `Las observaciones no pueden superar 1000 caracteres.` | `Se trasladó al afectado a enfermería` |

Iniciar la atención y cerrar no llevan cuerpo.

**Mensajes globales de formulario**

| Situación | Código HTTP | Mensaje |
|---|---|---|
| Cambio aplicado | 200 | Cuerpo con el incidente actualizado. |
| Transición inválida | 409 | `Transición inválida. El incidente debe estar en estado <esperado> y actualmente está en <actual>` |
| Reasignar un incidente resuelto o cerrado | 409 | `No se puede reasignar un incidente resuelto o cerrado.` |
| Ya es el responsable | 409 | `El brigadista indicado ya es el responsable del incidente.` |
| El brigadista no tiene asignación en la ubicación | 409 | `El brigadista no tiene una asignación vigente en la ubicación del incidente.` |
| El responsable no es brigadista | 403 | `El responsable debe tener el rol BRIGADISTA en MS01.` / `El usuario no está registrado como brigadista en MS05.` / `El brigadista debe estar activo en MS05.` |
| Atiende quien no es el responsable | 403 | `Solo el brigadista responsable puede atender este incidente.` |
| Rol insuficiente | 403 | `Solo un brigadista puede realizar esta operación.` / `La operación requiere el rol COORDINADOR u ORGANIZADOR.` |
| Sin permiso para ver el historial | 403 | `El usuario no tiene permisos para consultar el historial.` |
| Incidente inexistente o de otro evento | 404 | `El incidente no existe.` / `El incidente no pertenece al evento indicado.` |

---

## F-07 Adjuntar evidencia

**Endpoints:** `POST /api/v1/eventos/{eventoId}/incidentes/{incidenteId}/evidencias` (`multipart/form-data`) y `GET` sobre la misma ruta para listar · **Rol:** `COORDINATOR` y `BRIGADISTA` (cargar); más `AUDITOR` (consultar).

| Campo | Tipo | Obligatorio | Longitud / formato | Reglas de validación | Mensaje de error | Ejemplo válido |
|---|---|---|---|---|---|---|
| `archivo` | archivo | Sí | `image/png`, `application/pdf` o `video/mp4`; nombre hasta 255 caracteres | Debe traer contenido y un nombre válido. | `Debe seleccionar un archivo válido.` / `El archivo debe tener un nombre válido.` / `El formato no está permitido. Solo se aceptan PNG, PDF y MP4.` | `foto-lesion.png` |

**Mensajes globales de formulario**

| Situación | Código HTTP | Mensaje |
|---|---|---|
| Evidencia cargada | 201 | Cuerpo con los datos de la evidencia. |
| El incidente ya tiene 5 evidencias | 400 | `Un incidente no puede tener más de 5 evidencias.` |
| El incidente está cerrado | 400 | `No se pueden adjuntar evidencias a un incidente cerrado.` |
| Incidente inexistente o de otro evento | 404 | `El incidente indicado no existe.` / `El incidente no pertenece al evento indicado.` |
| Rol auditor al cargar | 403 | `Los usuarios con rol auditor no pueden cargar evidencias.` |
| Sin permiso al cargar o consultar | 403 | `El usuario no tiene permisos para adjuntar evidencias.` / `No tiene permisos para consultar las evidencias del incidente.` |
| Brigadista no activo | 403 | `Solo un brigadista activo puede adjuntar evidencias a incidentes de otros usuarios.` |
| No se pudo guardar el archivo | 500 | `No fue posible almacenar la evidencia.` |

---

## F-08 Consultas y filtros

**Incidentes del evento:** `GET /api/v1/eventos/{eventoId}/incidentes/consulta` · cualquier usuario autenticado (el resultado depende de su rol). **Responsabilidades:** `GET /api/v1/responsabilidades` (`COORDINATOR`, `AUDITOR`) y `GET /api/v1/responsabilidades/mias` (`BRIGADISTA`).

| Campo | Tipo | Obligatorio | Longitud / formato | Reglas de validación | Mensaje de error | Ejemplo válido |
|---|---|---|---|---|---|---|
| `prioridad` | string | No | `EMERGENCIA`, `URGENCIA_MODERADA` o `POCA_URGENCIA` | — | — | `EMERGENCIA` |
| `estado` | string | No | Estado del incidente o de la asignación | — | — | `EN_ATENCION` |
| `ubicacionId` | string | No | Identificador de MS02 | — | — | `b7c1e9a4-2d3f-4a56-9e80-1f2a3b4c5d6e` |
| `ordenarPor` | string | No | `prioridad`, `estado`, `ubicacion`, `antiguedad` o `edificio` | Cualquier otro valor ordena por fecha de reporte (el valor por defecto). | — | `prioridad` |
| `direccion` | string | No | `asc` o `desc` | Si falta, ordena en ascendente. | — | `desc` |
| `pagina` | entero | No | ≥ 0 | Por defecto 0. | `Los parámetros de paginación no son válidos.` | `0` |
| `tamanio` (incidentes) / `tamano` (responsabilidades) | entero | No | 1 a 100 | Por defecto 10. | `Los parámetros de paginación no son válidos.` | `10` |
| `eventoId`, `nombreBrigadista`, `tipoUbicacion` (solo responsabilidades) | string | No | — | `COORDINATOR` y `AUDITOR` pueden combinarlos. | — | `Gómez` |

**Auditoría:** `GET /api/v1/auditoria`, `/auditoria/eventos/{eventoId}` y `/auditoria/recursos/{recursoTipo}/{recursoId}` (solo `AUDITOR`) no tienen filtros propios más allá de la ruta.

**Mensajes globales de formulario**

| Situación | Código HTTP | Mensaje |
|---|---|---|
| Consulta correcta (con o sin resultados) | 200 | Página con `content` vacío si no hay coincidencias. |
| Falta el identificador del brigadista en responsabilidades | 400 | `El identificador del brigadista es obligatorio.` |
| Rol sin acceso | 403 | Acceso denegado. |
| Error inesperado | 500 | Mensaje genérico de error interno. |
