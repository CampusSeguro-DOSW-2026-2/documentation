# Especificación de Formularios y Entradas

Define los campos, validaciones y mensajes de cada formulario de MS03 (alertas y emergencias). Las validaciones están en el servicio (MS03) y el frontend las repite solo para dar una respuesta rápida. Los mensajes de validación de campos los devuelve la API en inglés, tal como se listan; los de reglas de negocio (estado, destinatario, simulacro…) vienen en español.

## Índice de Contenido
1. [Reglas generales de entrada](#reglas-generales-de-entrada)
2. [F-01 Crear alerta](#f-01-crear-alerta)
3. [F-02 Editar alerta](#f-02-editar-alerta)
4. [F-03 Programar alerta](#f-03-programar-alerta)
5. [F-04 Activar o archivar alerta](#f-04-activar-o-archivar-alerta)
6. [F-05 Listado y filtros de alertas](#f-05-listado-y-filtros-de-alertas)
7. [F-06 Bandeja y confirmación de recepción](#f-06-bandeja-y-confirmación-de-recepción)
8. [F-07 Auditoría de una alerta](#f-07-auditoría-de-una-alerta)

---

## Reglas generales de entrada

| ID | Regla |
|---|---|
| RG-01 | **Identidad:** el servicio no lee el usuario del cuerpo ni de la URL. El Gateway valida el JWT e inyecta `X-User-Id`, `X-User-Role` y `X-User-Permission`; sin ellos responde 401 `NO_AUTENTICADO`. |
| RG-02 | **Permisos:** `ms03:alert:manage` (crear, editar y gestionar cualquier alerta), `ms03:alert:manage_drill` (lo mismo, solo para alertas de simulacros), `ms03:alert:read_history` (listado, detalle y auditoría) y `ms03:alert:receive` (bandeja y confirmación). Con el permiso equivocado responde 403 `PROHIBIDO`. |
| RG-03 | **Tipos de alerta:** al menos uno, sin repetidos ni vacíos. `OPERATIVA` va sola y no se usa en alertas de un simulacro. |
| RG-04 | **Alcance:** `CAMPUS`, `BUILDING` o `ZONE`; `BUILDING` y `ZONE` exigen identificadores de MS02 (cada uno de hasta 64 caracteres y no vacío). |
| RG-05 | **Origen:** `MANUAL`, o `SIMULATION` con el `simulationId` del simulacro de MS06. Solo hay una alerta en borrador, programada o activa por simulacro y propósito. |
| RG-06 | **Identificadores:** los identificadores de alertas y simulacros son UUID. |
| RG-07 | **Fechas:** `activationAt` en formato ISO 8601 con zona y siempre en el futuro; `startDate` y `endDate`, solo fecha (`AAAA-MM-DD`). |
| RG-08 | **Formato de error:** `{ status, codigo, mensaje, errores: [{ campo, mensaje }], correlation_id }`. Las validaciones de campo devuelven 400 con `codigo = VALIDACION` y un elemento en `errores` por campo inválido. |
| RG-09 | **Paginación:** `page` desde 0 (por defecto 0); `size` entre 1 y 100 (20 por defecto). La respuesta trae `contenido`, `page`, `size`, `total_elementos` y `total_paginas`. |
| RG-10 | **Concurrencia:** si otro usuario modificó la alerta mientras se editaba, el servicio responde 409 `MODIFICACION_CONCURRENTE`. |

---

## F-01 Crear alerta

**Endpoint:** `POST /api/v1/alertas` · **Permiso:** `ms03:alert:manage` o `ms03:alert:manage_drill` · **Respuesta:** 201 con la alerta (en `DRAFT`, `SCHEDULED` o `ACTIVE` según lo pedido).

| Campo | Tipo | Obligatorio | Longitud / formato | Reglas de validación | Mensaje de error | Ejemplo válido |
|---|---|---|---|---|---|---|
| `title` | string | Sí | Hasta 150 caracteres | No puede estar vacío. | `Title is required` / `Title cannot exceed 150 characters` | `Evacuación por sismo` |
| `types` | lista de string | Sí | Códigos del catálogo (`EARTHQUAKE`, `FIRE`, `OTHER`, `OPERATIVA`) | Al menos uno; sin repetidos ni nulos; códigos conocidos; `OPERATIVA` va sola y no se usa con simulacros. | `At least one alert type is required` / `Alert types cannot contain duplicates` / `Alert types cannot contain null values` / `Unknown alert types: <códigos>` / `OPERATIVA cannot be combined with other alert types` / `OPERATIVA cannot be used for a simulation` | `["EARTHQUAKE"]` |
| `instructions` | string | Sí | Hasta 1000 caracteres | No puede estar vacío. | `Instructions are required` / `Instructions cannot exceed 1000 characters` | `Diríjase al punto de encuentro más cercano` |
| `recipients` | lista de objetos | Sí | — | Al menos uno; sin nulos. | `At least one recipient is required` / `Recipient cannot be null` | `[{ "type": "ROLE", "id": "ESTUDIANTE" }]` |
| `recipients[].type` | string | Sí | `USER` o `ROLE` | — | `Recipient type is required` | `ROLE` |
| `recipients[].id` | string | Sí | Hasta 64 caracteres | Identificador del usuario o nombre del rol de MS01. | `Recipient ID is required` / `Recipient ID cannot exceed 64 characters` | `ESTUDIANTE` |
| `scope` o `scopes` | objeto o lista | Sí | — | Al menos un alcance. | `At least one scope is required` | `{ "type": "CAMPUS" }` |
| `scope.type` | string | Sí | `CAMPUS`, `BUILDING` o `ZONE` | — | `Scope type is required` | `BUILDING` |
| `scope.ids` | lista de string | Condicional | Cada uno hasta 64 caracteres | Obligatorio con `BUILDING` o `ZONE`; sin vacíos. Deben existir en MS02. | `At least one identifier must be provided for the selected scope` / `Scope identifiers must not be blank or exceed 64 characters` | `["b7c1e9a4-..."]` |
| `origin.type` | string | Sí | `MANUAL` o `SIMULATION` | — | `Origin type is required` | `SIMULATION` |
| `origin.simulationId` | UUID | Condicional | UUID | Obligatorio con `SIMULATION`; no se permite con `MANUAL`. | `simulationId is required when the origin is SIMULATION` / `simulationId is only allowed when the origin is SIMULATION` | `9a2d4e6f-1b3c-4d5e-8f70-a1b2c3d4e5f6` |
| `purpose` | string | No | Hasta 32 caracteres | Distingue varias alertas de un mismo simulacro. | `Purpose cannot exceed 32 characters` | `participantes` |
| `activationAt` | fecha-hora | No | ISO 8601 con zona | Futura. Programa la alerta (`SCHEDULED`). | `La fecha de activación debe ser futura` | `2026-12-01T09:00:00-05:00` |
| `activateImmediately` | booleano | No | `true` o `false` | No se combina con `activationAt`. Activa la alerta al crearla. | `An immediate activation cannot include activationAt` | `false` |

Si no se envía ni `activationAt` ni `activateImmediately`, la alerta queda en `DRAFT`.

**Mensajes globales de formulario**

| Situación | Código HTTP / `codigo` | Mensaje |
|---|---|---|
| Alerta creada | 201 | Cuerpo con la alerta. |
| Campo inválido | 400 `VALIDACION` | `Los datos enviados no son válidos` (con el detalle en `errores`). |
| Un edificio o zona no existe en MS02 | 400 `REFERENCIA_INVALIDA` | `Referencias inválidas en <campo>: <identificadores>` |
| Fecha de activación pasada | 400 `FECHA_ACTIVACION_INVALIDA` | `La fecha de activación debe ser futura` |
| Ya hay una alerta viva del simulacro y propósito | 409 `SIMULACION_DUPLICADA` | `Ya existe una alerta en borrador, programada o activa para este simulacro` |
| Sin permiso | 403 `PROHIBIDO` | `No tienes permiso para realizar esta operación` |
| ORGANIZADOR con una alerta que no es de un simulacro | 403 `PROHIBIDO` | `El rol ORGANIZADOR solo puede operar alertas asociadas a un simulacro` |
| MS02 no responde | 503 `SERVICIO_EXTERNO_NO_DISPONIBLE` | `Un servicio externo no está disponible. Intenta de nuevo más tarde` |

---

## F-02 Editar alerta

**Endpoint:** `PUT /api/v1/alertas/{id}` · **Permiso:** `ms03:alert:manage` o `ms03:alert:manage_drill`. Solo se edita una alerta en `DRAFT` o `SCHEDULED`.

| Campo | Tipo | Obligatorio | Longitud / formato | Reglas de validación | Mensaje de error | Ejemplo válido |
|---|---|---|---|---|---|---|
| `id` (ruta) | UUID | Sí | UUID | Debe existir. | `La alerta no existe` | `5d1e8a2b-4c3f-4a60-8b91-2c3d4e5f6a7b` |
| `title` | string | No | Hasta 150 caracteres | Si se envía, no puede estar vacío. | `Title cannot be empty` / `Title cannot exceed 150 characters` | `Evacuación por incendio` |
| `types` | lista de string | No | Códigos del catálogo | Si se envía, las mismas reglas que al crear. | `You must specify at least one alert type.` | `["FIRE"]` |
| `instructions` | string | No | Hasta 1000 caracteres | Si se envía, no puede estar vacío. | `Instructions cannot be empty` / `Instructions cannot exceed 1000 characters.` | `Salga por la puerta norte` |
| `recipients` | lista de objetos | No | — | Si se envía, al menos uno; sin nulos. | `You must specify at least one recipient.` / `The recipient cannot be null` | `[{ "type": "USER", "id": "3f6c2a9e-..." }]` |
| `scope` o `scopes` | objeto o lista | No | — | Las mismas reglas que al crear. | `At least one scope is required` | `{ "type": "ZONE", "ids": ["..."] }` |

Debe enviarse al menos un campo. Los campos que no se envían no cambian; los que se envían reemplazan por completo al anterior.

**Mensajes globales de formulario**

| Situación | Código HTTP / `codigo` | Mensaje |
|---|---|---|
| Alerta editada | 200 | Cuerpo con la alerta. |
| No se envió ningún campo | 400 `VALIDACION` | `At least one field must be provided for the update` |
| La alerta no se puede editar en su estado actual | 409 `ESTADO_ALERTA_INVALIDO` | `La alerta está en estado <estado> y no permite esta operación` |
| La alerta no existe | 404 `ALERTA_NO_ENCONTRADA` | `La alerta no existe` |
| Otro usuario la modificó | 409 `MODIFICACION_CONCURRENTE` | `La alerta fue modificada por otro usuario` |

---

## F-03 Programar alerta

**Endpoint:** `PATCH /api/v1/alertas/{id}/programar` · **Permiso:** `ms03:alert:manage` o `ms03:alert:manage_drill`. Solo desde `DRAFT`.

| Campo | Tipo | Obligatorio | Longitud / formato | Reglas de validación | Mensaje de error | Ejemplo válido |
|---|---|---|---|---|---|---|
| `activationAt` | fecha-hora | Sí | ISO 8601 con zona | Futura. | `The activation date is mandatory.` / `La fecha de activación debe ser futura` | `2026-12-01T09:00:00-05:00` |

**Mensajes globales de formulario**

| Situación | Código HTTP / `codigo` | Mensaje |
|---|---|---|
| Alerta programada | 200 | Cuerpo con el cambio de estado. |
| Transición no permitida | 409 `TRANSICION_INVALIDA` | `No se puede cambiar la alerta de <estado actual> a SCHEDULED` |
| La alerta no existe | 404 `ALERTA_NO_ENCONTRADA` | `La alerta no existe` |

---

## F-04 Activar o archivar alerta

**Endpoints:** `PATCH /api/v1/alertas/{id}/activar` y `PATCH /api/v1/alertas/{id}/archivar` · **Permiso:** `ms03:alert:manage` o `ms03:alert:manage_drill`. No llevan cuerpo.

| Campo | Tipo | Obligatorio | Longitud / formato | Reglas de validación | Mensaje de error | Ejemplo válido |
|---|---|---|---|---|---|---|
| `id` (ruta) | UUID | Sí | UUID | Debe existir. | `La alerta no existe` | `5d1e8a2b-4c3f-4a60-8b91-2c3d4e5f6a7b` |

Activar es válido desde `DRAFT` o `SCHEDULED`; archivar, desde `DRAFT`, `SCHEDULED` o `ACTIVE`. Al activar, MS03 resuelve los destinatarios con MS01 y envía los correos.

**Mensajes globales de formulario**

| Situación | Código HTTP / `codigo` | Mensaje |
|---|---|---|
| Cambio aplicado | 200 | Cuerpo con el cambio de estado. |
| Transición no permitida (por ejemplo, activar una archivada) | 409 `TRANSICION_INVALIDA` | `No se puede cambiar la alerta de <estado actual> a <estado pedido>` |
| La alerta no existe | 404 `ALERTA_NO_ENCONTRADA` | `La alerta no existe` |
| ORGANIZADOR con una alerta que no es de un simulacro | 403 `PROHIBIDO` | `El rol ORGANIZADOR solo puede operar alertas asociadas a un simulacro` |

---

## F-05 Listado y filtros de alertas

**Endpoint:** `GET /api/v1/alertas` · **Permiso:** `ms03:alert:read_history` · **Respuesta:** página de alertas, resumidas.

| Campo | Tipo | Obligatorio | Longitud / formato | Reglas de validación | Mensaje de error | Ejemplo válido |
|---|---|---|---|---|---|---|
| `status` | string | No | `DRAFT`, `SCHEDULED`, `ACTIVE` o `ARCHIVED` | Estado conocido. | `Unknown alert status: <valor>` | `ACTIVE` |
| `type` | string (repetible) | No | Código del catálogo | Varios `type` se combinan (la alerta tiene alguno de ellos). | `Unknown alert types: <códigos>` | `FIRE` |
| `origin` | string | No | `MANUAL` o `SIMULATION` | — | `El valor enviado no es válido` | `SIMULATION` |
| `scope` | string | No | `CAMPUS`, `BUILDING` o `ZONE` | — | `El valor enviado no es válido` | `BUILDING` |
| `activeAt` | fecha-hora | No | ISO 8601 | Alertas que ya estaban activadas en ese momento y aún no archivadas. | `El valor enviado no es válido` | `2026-11-25T10:00:00Z` |
| `startDate` | fecha | No | `AAAA-MM-DD` | No puede ser posterior a `endDate`. | `Start date cannot be after end date` | `2026-11-01` |
| `endDate` | fecha | No | `AAAA-MM-DD` | — | `El valor enviado no es válido` | `2026-11-30` |
| `simulationId` | UUID | No | UUID | — | `El valor enviado no es válido` | `9a2d4e6f-1b3c-4d5e-8f70-a1b2c3d4e5f6` |
| `page` | entero | No | ≥ 0 | Por defecto 0. | `Page index cannot be negative` | `0` |
| `size` | entero | No | 1 a 100 | Por defecto 20. | `Page size must be between 1 and 100` | `20` |

**Mensajes globales de formulario**

| Situación | Código HTTP / `codigo` | Mensaje |
|---|---|---|
| Consulta correcta (con o sin resultados) | 200 | Página con `contenido` vacío si no hay coincidencias. |
| Filtro o paginación inválidos | 400 `PAGINACION_INVALIDA` / `VALIDACION` | `Los parámetros de consulta no son válidos` / `Los datos enviados no son válidos` |
| Sin permiso | 403 `PROHIBIDO` | `No tienes permiso para realizar esta operación` |

**Detalle:** `GET /api/v1/alertas/{id}` devuelve la alerta completa a quien tenga `ms03:alert:read_history` o `ms03:alert:receive`; si no existe, responde 404 `ALERTA_NO_ENCONTRADA`.

---

## F-06 Bandeja y confirmación de recepción

**Endpoints:** `GET /api/v1/alertas/mias` y `POST /api/v1/alertas/{id}/confirmaciones` · **Permiso:** `ms03:alert:receive`. La confirmación no lleva cuerpo.

| Campo | Tipo | Obligatorio | Longitud / formato | Reglas de validación | Mensaje de error | Ejemplo válido |
|---|---|---|---|---|---|---|
| `status` (bandeja) | string | No | `ACTIVE`, `ARCHIVED` o `ALL` | Por defecto `ACTIVE`. Nunca devuelve `DRAFT` ni `SCHEDULED`. | `El valor enviado no es válido` | `ALL` |
| `page` (bandeja) | entero | No | ≥ 0 | Por defecto 0. | `Page index cannot be negative` | `0` |
| `size` (bandeja) | entero | No | 1 a 100 | Por defecto 20. | `Page size must be between 1 and 100` | `20` |
| `id` (confirmación, ruta) | UUID | Sí | UUID | La alerta debe existir, estar activa y estar dirigida al usuario. | `La alerta no existe` | `5d1e8a2b-4c3f-4a60-8b91-2c3d4e5f6a7b` |

**Mensajes globales de formulario**

| Situación | Código HTTP / `codigo` | Mensaje |
|---|---|---|
| Bandeja consultada | 200 | Página con las alertas dirigidas al usuario por identificador, por rol o por envío registrado. |
| Primera confirmación | 201 | Cuerpo con la confirmación. |
| Confirmación repetida | 200 | Cuerpo con la confirmación original; no crea otra. |
| La alerta no está activa | 409 `ALERTA_NO_ACTIVA` | `La alerta <id> no esta activa` |
| El usuario no es destinatario | 403 `NO_ES_DESTINATARIO` | `No es destinatario de la alerta <id>` |
| La alerta no existe | 404 `ALERTA_NO_ENCONTRADA` | `La alerta no existe` |

---

## F-07 Auditoría de una alerta

**Endpoint:** `GET /api/v1/alertas/{id}/auditoria` · **Permiso:** `ms03:alert:read_history`.

| Campo | Tipo | Obligatorio | Longitud / formato | Reglas de validación | Mensaje de error | Ejemplo válido |
|---|---|---|---|---|---|---|
| `id` (ruta) | UUID | Sí | UUID | La alerta debe existir. | `La alerta no existe` | `5d1e8a2b-4c3f-4a60-8b91-2c3d4e5f6a7b` |

**Mensajes globales de formulario**

| Situación | Código HTTP / `codigo` | Mensaje |
|---|---|---|
| Historial devuelto | 200 | Lista de hechos de la alerta, del más antiguo al más reciente. |
| La alerta no existe | 404 `ALERTA_NO_ENCONTRADA` | `La alerta no existe` |
| Sin permiso | 403 `PROHIBIDO` | `No tienes permiso para realizar esta operación` |
