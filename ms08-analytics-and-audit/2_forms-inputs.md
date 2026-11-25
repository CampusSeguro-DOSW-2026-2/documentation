# Especificación de Formularios y Entradas

MS08 no tiene pantallas de captura: lo que recibe son (a) los hechos que envían los demás microservicios y (b) los filtros de consulta del panel de auditoría y de indicadores. Cada "formulario" de este documento describe el cuerpo o los parámetros de un endpoint: campos, validaciones y mensajes. Las validaciones están en el servicio (MS08) y el frontend las repite solo para dar una respuesta rápida.

## Índice de Contenido
1. [Reglas generales de entrada](#reglas-generales-de-entrada)
2. [F-01 Registrar un hecho de auditoría](#f-01-registrar-un-hecho-de-auditoría)
3. [F-02 Búsqueda y filtros de auditoría](#f-02-búsqueda-y-filtros-de-auditoría)
4. [F-03 Detalle de un registro](#f-03-detalle-de-un-registro)
5. [F-04 Indicadores de un evento](#f-04-indicadores-de-un-evento)
6. [F-05 Comparación de eventos](#f-05-comparación-de-eventos)
7. [F-06 Ubicaciones de un evento](#f-06-ubicaciones-de-un-evento)

---

## Reglas generales de entrada

| ID | Regla |
|---|---|
| RG-01 | **Identidad:** el servicio no lee el usuario del cuerpo ni de la URL. El Gateway valida el JWT e inyecta `X-User-Id`, `X-User-Role`, `X-User-Permission` y `X-Service-Name`. |
| RG-02 | **Quién escribe y quién lee:** `POST /api/v1/audit/events` solo lo puede llamar un microservicio (cabecera `X-Service-Name` con un valor entre `ms01` y `ms08`). Los `GET` los pide un usuario con alguno de los permisos `ms08:audit_drill:read`, `ms08:audit_real_event:read`, `ms08:indicators_drill:read` o `ms08:indicators_real_event:read`. |
| RG-03 | **Contrato estricto:** el cuerpo del `POST` solo admite los campos definidos en [F-01](#f-01-registrar-un-hecho-de-auditoría). Un campo desconocido se rechaza con 400 y no se ignora en silencio. |
| RG-04 | **Sin datos personales:** `detalle` y `atributos` solo llevan códigos, identificadores y conteos: nunca nombres, correos, mensajes ni cuerpos de solicitudes. |
| RG-05 | **Identificadores:** `usuario_id` y `evento_id` son UUID. Se aceptan en mayúsculas y se guardan en minúsculas. |
| RG-06 | **Fechas:** formato ISO 8601 con zona (por ejemplo `2026-11-25T14:30:00-05:00`). Se guardan en UTC. |
| RG-07 | **Idempotencia:** reenviar un hecho con el mismo `id` (o, sin `id`, con la misma combinación de correlación, módulo, acción, recurso y fecha) responde 202 igual que la primera vez y no crea otro registro. |
| RG-08 | **Formato de error:** `{ status, codigo, mensaje, errores: [{ campo, mensaje }], correlation_id }`. Las validaciones de campo devuelven 400 con `codigo = VALIDACION` y un elemento en `errores` por cada campo inválido. |
| RG-09 | **Paginación:** `page` desde 0; `size` de 1 a 100 (20 por defecto). La respuesta trae `contenido`, `page`, `size`, `total_elementos` y `total_paginas`. |
| RG-10 | **Mensajes:** la API devuelve los mensajes en español, tal como se listan; el frontend puede mostrarlos o reemplazarlos con su propio texto. |

---

## F-01 Registrar un hecho de auditoría

**Endpoint:** `POST /api/v1/audit/events` · **Quién:** un microservicio (RG-02) · **Respuesta:** 202 Accepted sin cuerpo.

| Campo | Tipo | Obligatorio | Longitud / formato | Reglas de validación | Mensaje de error | Ejemplo válido |
|---|---|---|---|---|---|---|
| `id` | string (UUID) | No | UUID | Si se envía, debe ser un UUID. Si no, MS08 lo calcula. Reenviar el mismo `id` no duplica el registro. | `Debe ser un UUID` | `7b0f6c1e-2a4d-4a55-9f0b-3d5b8a1c9e21` |
| `usuario_id` | string (UUID) | No | UUID | Quien hizo la acción. Se omite en acciones del sistema (por ejemplo, un inicio automático). | `Debe ser un UUID` | `3f6c2a9e-8d1b-4c70-a1e5-6b2f9d4c7a10` |
| `modulo` | string | Sí | `MS01` a `MS08` | Servicio que envía el hecho. | `Debe ser MS01..MS08` | `MS06` |
| `accion` | string | Sí | Hasta 100 caracteres, MAYÚSCULAS, dígitos y `_`; empieza con letra | Código de la acción, del [catálogo de auditoría](../integration/audit-catalog.md). | `Debe estar en MAYÚSCULAS con guion bajo` | `EVENT_STARTED` |
| `recurso_afectado` | string | No | `<tipo>:<id>`; `<tipo>` empieza con letra; `<id>` de 1 a 100 caracteres sin espacios | Recurso sobre el que se actuó. | `Debe tener el formato <tipo>:<id>` | `evento:3f6c2a9e-8d1b-4c70-a1e5-6b2f9d4c7a10` |
| `resultado` | string | Sí | `EXITO` o `FALLO` | Resultado de la operación. | `Es obligatorio` (un valor distinto de `EXITO`/`FALLO` da el error general de cuerpo) | `EXITO` |
| `fecha_hora` | string (fecha-hora) | Sí | ISO 8601 con zona | No puede estar más de 5 minutos en el futuro ni ser anterior a la retención (180 días). | `No puede estar más de 5 minutos en el futuro` / `Es más antigua que el periodo de retención` | `2026-11-25T14:30:00-05:00` |
| `correlation_id` | string | Sí | Hasta 100 caracteres | Identificador de correlación de la petición original. | `Es obligatorio` / `Máximo 100 caracteres` | `c0a8f2e4-91b7-4f0d-8e26-1c3d5a7b9f02` |
| `detalle` | string | No | Hasta 100 caracteres | Código genérico sobre el resultado. Sin datos personales (RG-04). | `Máximo 100 caracteres` | `RESPONDIDA` |
| `evento_id` | string (UUID) | No | UUID | Simulacro o evento real al que pertenece el hecho. Se omite en los hechos administrativos (MS01, MS02, MS07). | `Debe ser un UUID` | `3f6c2a9e-8d1b-4c70-a1e5-6b2f9d4c7a10` |
| `tipo_evento` | string | Condicional | `SIMULACRO` o `REAL` | Obligatorio si se envía `evento_id`. `SIMULACRO` exige `evento_id`. | `Es obligatorio cuando se envía evento_id` / `SIMULACRO requiere evento_id` | `SIMULACRO` |
| `atributos` | objeto | No | Plano; hasta 20 claves en `snake_case` | Cada valor es texto (hasta 100 caracteres), número o booleano. No se admiten objetos ni listas anidados. | `Máximo 20 claves` / `Las claves deben estar en snake_case: <clave>` / `Solo texto (máx. 100), número o booleano` | `{ "estado": "SEGURO", "punto_encuentro_id": "pe-04" }` |

**Mensajes globales de formulario**

| Situación | Código HTTP / `codigo` | Mensaje |
|---|---|---|
| Hecho aceptado (o repetido) | 202 | Sin cuerpo. |
| Campo desconocido, JSON mal formado o regla de campo incumplida | 400 `VALIDACION` | `Los datos enviados no son válidos` (con el detalle en `errores`). |
| Sin identidad del Gateway | 401 `NO_AUTENTICADO` | `Se requiere autenticación o la identidad no es válida`. |
| Quien llama no es un microservicio | 403 `PROHIBIDO` | `No tienes permiso para realizar esta operación`. |
| Tipo de contenido distinto de `application/json` | 415 `TIPO_NO_SOPORTADO` | `El tipo de contenido no está soportado`. |
| Base de datos no disponible | 503 `SERVICIO_NO_DISPONIBLE` | `El servicio no está disponible. Intenta de nuevo más tarde`. |
| Error inesperado | 500 `ERROR_INTERNO` | `Ocurrió un error inesperado`. |

---

## F-02 Búsqueda y filtros de auditoría

**Endpoint:** `GET /api/v1/audit/events` · **Permiso:** `ms08:audit_drill:read` y/o `ms08:audit_real_event:read` · **Respuesta:** página de registros, del más reciente al más antiguo.

| Campo | Tipo | Obligatorio | Longitud / formato | Reglas de validación | Mensaje de error | Ejemplo válido |
|---|---|---|---|---|---|---|
| `modulo` | string | No | Hasta 200 caracteres | Igualdad. | `Máximo 200 caracteres` | `MS06` |
| `accion` | string | No | Hasta 200 caracteres | Igualdad. | `Máximo 200 caracteres` | `EVENT_STARTED` |
| `usuario_id` | string (UUID) | No | UUID | Igualdad. | `Debe ser un UUID` | `3f6c2a9e-8d1b-4c70-a1e5-6b2f9d4c7a10` |
| `evento_id` | string (UUID) | No | UUID | Igualdad. | `Debe ser un UUID` | `9a2d4e6f-1b3c-4d5e-8f70-a1b2c3d4e5f6` |
| `tipo_evento` | string | No | `SIMULACRO` o `REAL` | Si no se envía y el usuario solo tiene uno de los dos permisos de lectura, se usa el único tipo que puede ver. Pedir un tipo sin su permiso da 403. | `El valor enviado no es válido` | `SIMULACRO` |
| `recurso_afectado` | string | No | Hasta 200 caracteres | Igualdad. | `Máximo 200 caracteres` | `alerta:5d1e8a2b-...` |
| `correlation_id` | string | No | Hasta 200 caracteres | Igualdad. | `Máximo 200 caracteres` | `c0a8f2e4-91b7-4f0d-8e26-1c3d5a7b9f02` |
| `desde` | string (fecha-hora) | No | ISO 8601 | Incluyente. | `El valor enviado no es válido` | `2026-11-01T00:00:00-05:00` |
| `hasta` | string (fecha-hora) | No | ISO 8601 | Incluyente. `desde` no puede ser posterior a `hasta`. | `No puede ser posterior a hasta` | `2026-11-25T23:59:59-05:00` |
| `page` | entero | No | ≥ 0 | Por defecto 0. | `Debe ser 0 o mayor` | `0` |
| `size` | entero | No | 1 a 100 | Por defecto 20. | `Debe estar entre 1 y 100` | `20` |

**Mensajes globales de formulario**

| Situación | Código HTTP / `codigo` | Mensaje |
|---|---|---|
| Consulta correcta (con o sin resultados) | 200 | Página con `contenido` vacío si no hay coincidencias. |
| Parámetro inválido | 400 `VALIDACION` | `Los datos enviados no son válidos` (con el detalle en `errores`). |
| Sin identidad del Gateway | 401 `NO_AUTENTICADO` | `Se requiere autenticación o la identidad no es válida`. |
| Sin permiso de lectura de auditoría, o tipo de evento no permitido | 403 `PROHIBIDO` | `No tienes permiso para realizar esta operación`. |
| Método distinto de GET/POST | 405 `METODO_NO_PERMITIDO` | `El método HTTP no está permitido`. |

---

## F-03 Detalle de un registro

**Endpoint:** `GET /api/v1/audit/events/{id}` · **Permiso:** el de lectura de auditoría que corresponda al `tipo_evento` del registro (ambos si es administrativo, sin tipo).

| Campo | Tipo | Obligatorio | Longitud / formato | Reglas de validación | Mensaje de error | Ejemplo válido |
|---|---|---|---|---|---|---|
| `id` (ruta) | string | Sí | Identificador del registro | Debe existir. | `El recurso solicitado no existe` | `7b0f6c1e-2a4d-4a55-9f0b-3d5b8a1c9e21` |

**Mensajes globales de formulario**

| Situación | Código HTTP / `codigo` | Mensaje |
|---|---|---|
| Registro encontrado | 200 | Registro completo. |
| El registro no existe | 404 `NO_ENCONTRADO` | `El recurso solicitado no existe`. |
| El registro existe pero el usuario no tiene el permiso de su tipo | 403 `PROHIBIDO` | `No tienes permiso para realizar esta operación`. |
| Sin identidad del Gateway | 401 `NO_AUTENTICADO` | `Se requiere autenticación o la identidad no es válida`. |

---

## F-04 Indicadores de un evento

**Endpoint:** `GET /api/v1/analytics/events/{eventoId}` · **Permiso:** `ms08:indicators_drill:read` para simulacros y `ms08:indicators_real_event:read` para eventos reales. Los indicadores no se guardan: se calculan en cada consulta con los hechos `EXITO` disponibles.

| Campo | Tipo | Obligatorio | Longitud / formato | Reglas de validación | Mensaje de error | Ejemplo válido |
|---|---|---|---|---|---|---|
| `eventoId` (ruta) | string (UUID) | Sí | UUID | Debe ser un UUID y tener hechos registrados. | `Debe ser un UUID` (campo `evento_id`) | `9a2d4e6f-1b3c-4d5e-8f70-a1b2c3d4e5f6` |

**Mensajes globales de formulario**

| Situación | Código HTTP / `codigo` | Mensaje |
|---|---|---|
| Indicadores calculados | 200 | Indicadores del evento. Los que no tengan datos suficientes salen vacíos, no en cero. |
| `eventoId` inválido | 400 `VALIDACION` | `Los datos enviados no son válidos`. |
| Sin permiso para el tipo del evento | 403 `PROHIBIDO` | `No tienes permiso para realizar esta operación`. |
| El evento no tiene hechos | 404 `NO_ENCONTRADO` | `El recurso solicitado no existe`. |

---

## F-05 Comparación de eventos

**Endpoint:** `GET /api/v1/analytics/comparisons` · **Permiso:** el de indicadores del tipo de los eventos comparados.

| Campo | Tipo | Obligatorio | Longitud / formato | Reglas de validación | Mensaje de error | Ejemplo válido |
|---|---|---|---|---|---|---|
| `evento_id` (repetible) | lista de UUID | Sí | De 2 a 10 UUID | Sin repetidos. Todos del mismo tipo (simulacros con simulacros, o reales con reales). | `Indica entre 2 y 10 eventos distintos` | `?evento_id=9a2d...&evento_id=3f6c...` |

**Mensajes globales de formulario**

| Situación | Código HTTP / `codigo` | Mensaje |
|---|---|---|
| Comparación calculada | 200 | Un bloque de indicadores por evento. |
| Menos de 2, más de 10, repetidos o UUID inválido | 400 `VALIDACION` | `Los datos enviados no son válidos`. |
| Eventos de tipos distintos | 400 `VALIDACION` | `Los eventos deben ser del mismo tipo`. |
| Sin permiso para el tipo | 403 `PROHIBIDO` | `No tienes permiso para realizar esta operación`. |
| Algún evento no tiene hechos | 404 `NO_ENCONTRADO` | `El recurso solicitado no existe`. |

---

## F-06 Ubicaciones de un evento

**Endpoint:** `GET /api/v1/analytics/locations` · **Permiso:** el de indicadores del tipo de los eventos pedidos. Devuelve las ubicaciones más lentas en llegar a un lugar seguro.

| Campo | Tipo | Obligatorio | Longitud / formato | Reglas de validación | Mensaje de error | Ejemplo válido |
|---|---|---|---|---|---|---|
| `evento_id` (repetible) | lista de UUID | Sí | De 1 a 10 UUID | Sin repetidos. Todos del mismo tipo. | `Indica entre 1 y 10 eventos distintos` | `?evento_id=9a2d...` |

**Mensajes globales de formulario**

| Situación | Código HTTP / `codigo` | Mensaje |
|---|---|---|
| Ubicaciones calculadas | 200 | Lista de ubicaciones, de la más lenta a la más rápida. |
| Sin eventos, más de 10, repetidos o UUID inválido | 400 `VALIDACION` | `Los datos enviados no son válidos`. |
| Eventos de tipos distintos | 400 `VALIDACION` | `Los eventos deben ser del mismo tipo`. |
| Sin permiso para el tipo | 403 `PROHIBIDO` | `No tienes permiso para realizar esta operación`. |
| Algún evento no tiene hechos | 404 `NO_ENCONTRADO` | `El recurso solicitado no existe`. |
