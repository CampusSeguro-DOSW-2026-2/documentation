# Diccionario de Datos, Formularios y Entradas

## Índice de Contenido
1. [Reglas generales de entrada](#reglas-generales-de-entrada)
2. [Formularios y Entradas](#formularios-y-entradas)
   - [F-01 · Reporte de estado (check-in)](#f-01)
   - [F-02 · Filtros de personas por estado](#f-02)
   - [F-03 · Listados de pendientes y de asistencia](#f-03)
   - [F-04 · Filtros del histórico del evento](#f-04)
   - [F-05 · Estado del evento (MS06)](#f-05)
   - [F-06 · Detalle de check-ins del evento (MS06)](#f-06)
   - [F-07 · Filtros de eventos](#f-07)
3. [Pantallas sin formulario](#pantallas-sin-formulario)

---

<a id="reglas-generales-de-entrada"></a>
## Reglas generales de entrada

* **RG-01**: Todo texto se recorta (trim) de espacios al inicio y al final antes de validar.
* **RG-02**: Un campo obligatorio vacío o con solo espacios se considera faltante.
* **RG-03**: El id del evento (`eventoId` en la ruta) es un UUID: el que asigna MS06 al evento (**RN-MS04-01**). Un valor con formato inválido se rechaza (HTTP 400): "Evento no válido."
* **RG-04**: Se rechazan (HTTP 400) los campos no definidos en el contrato (en especial un id de usuario enviado en el cuerpo del check-in, **RN-MS04-06**).
* **RG-05**: Se rechazan (HTTP 400) los campos que no corresponden al estado reportado: el punto de encuentro solo se envía con Estoy a salvo y la ubicación solo con Requiero asistencia (ver [Estados del check-in](1_requirements.md#estados-del-check-in)).
* **RG-06**: Los errores de validación son HTTP 400 `VALIDACION`, siempre con el `ErrorResponse` de los Estándares; el campo y su mensaje van en `errores`.
* **RG-07**: La identidad del solicitante (id y roles) nunca viaja en el cuerpo: llega en las cabeceras X-User-Id y X-User-Role (**RN-MS04-06**). En las rutas internas (F-05 y F-06) no hay usuario: llega el servicio en X-Service-Name (**RN-MS04-23**).
* **RG-08**: Los ids de MS02 (punto de encuentro, edificio, espacio) son enteros positivos, igual que en el contrato de MS02.
* **RG-09**: MS04 no tiene formularios de auditoría: su consulta pertenece a MS08 (**RNF-MS04-17**).
* **RG-10**: F-05 y F-06 no son pantallas: son el contrato de entrada de las rutas internas que usa MS06. Sus mensajes de error son para quien integra, no para el usuario final, y se entregan en el `ErrorResponse` de los Estándares.

---

<a id="formularios-y-entradas"></a>
## Formularios y Entradas

<a id="f-01"></a>
### F-01 · Reporte de estado (check-in)
Referencia: **RF-MS04-02**, **RF-MS04-03**, **RF-MS04-04**

El mismo formulario registra el primer check-in y lo actualiza. Solo se puede enviar mientras el check-in del evento esté habilitado (evento en ejecución, **RN-MS04-19**). Se completa en máximo 3 interacciones desde la pantalla inicial (**RNF-MS04-07**).

| Campo | Tipo | Obligatorio | Longitud / formato | Reglas de validación | Mensaje de error | Ejemplo válido |
|---|---|---|---|---|---|---|
| Estado | Selección única | Sí | `SAFE` / `EVACUATING` / `NEEDS_ASSISTANCE` / `OFF_CAMPUS` | Se muestran siempre las 4 opciones con su texto: Estoy a salvo, Me estoy evacuando, Requiero asistencia, No estoy en el campus. Se envía el código (**RN-MS04-03**). | "Seleccione un estado válido." | SAFE |
| Punto de encuentro | Selección (puntos de encuentro de MS02) | Solo si el estado es Estoy a salvo | Entero positivo | Debe existir y estar ACTIVO en MS02 (**RN-MS04-04**). No se envía con otros estados (RG-05). | Falta (400): "Seleccione el punto de encuentro donde está." Inexistente (404): "Punto de encuentro no encontrado." Inactivo (422): "El punto de encuentro no está activo." Con otro estado (400): "El punto de encuentro solo aplica a Estoy a salvo." | 3 |
| Tipo de ubicación | Selección única | Solo si el estado es Requiero asistencia | `EDIFICIO` / `ESPACIO` / `PUNTO_ENCUENTRO` | Solo valores del catálogo (**RN-MS04-09**). No se envía con otros estados (RG-05). | Falta (400): "Indique dónde necesita la asistencia." Con otro estado (400): "La ubicación solo aplica a Requiero asistencia." | EDIFICIO |
| Ubicación | Selección (elementos de MS02 del tipo elegido) | Solo si el estado es Requiero asistencia | Entero positivo | Debe corresponder al tipo elegido. Su existencia la valida MS05 al crear la solicitud (**RF-MS04-04**, A4). | Falta (400): "Seleccione el lugar donde necesita la asistencia." | 12 |

**Mensajes globales de formulario:**
* Evento que no está en ejecución, ya sea programado, en borrador o finalizado (409): "El check-in de este evento no está habilitado."
* El evento no le aplica al usuario, porque su rol no está convocado (422): "Este evento no le aplica."
* Evento inexistente (404): "Evento no encontrado."
* MS02 no disponible (503): "El servicio no está disponible en este momento. Intente de nuevo."
* Éxito, solicitud de asistencia ENVIADA: "Su estado quedó registrado y su solicitud de asistencia fue enviada."
* Éxito, solicitud PENDIENTE_ENVIO: "Su estado quedó registrado. La solicitud de asistencia se enviará en cuanto el servicio esté disponible."
* Éxito, solicitud RECHAZADA: "Su estado quedó registrado, pero la solicitud de asistencia no pudo crearse. Un coordinador la revisará."

*Mockup F-01: (espacio reservado para insertar imagen del reporte de estado)*

---

<a id="f-02"></a>
### F-02 · Filtros de personas por estado
Referencia: **RF-MS04-06** (`GET /{eventoId}/personas`)

| Campo | Tipo | Obligatorio | Longitud / formato | Reglas de validación | Mensaje de error | Ejemplo válido |
|---|---|---|---|---|---|---|
| Estado | Lista | No | Uno de los 4 estados / Todos | Solo valores del catálogo (**RN-MS04-03**). Omitir equivale a Todos. | "Estado no válido." | SAFE |
| Punto de encuentro | Lista (puntos de encuentro de MS02) | No | Entero positivo | Solo con el estado Estoy a salvo o sin estado (en ese caso se aplica Estoy a salvo). | Formato: "Punto de encuentro no válido." Con otro estado (400): "El punto de encuentro solo aplica a Estoy a salvo." | 3 |
| Página | Número entero | No | ≥ 0 (por defecto 0) | Entero no negativo. | "Página no válida." | 0 |
| Tamaño de página | Número entero | No | 1 a 100 (por defecto 20) | Entero en el rango. | "Tamaño de página no válido (1 a 100)." | 20 |

*Mockup F-02: (espacio reservado para insertar imagen del dashboard con el listado por estado)*

---

<a id="f-03"></a>
### F-03 · Listados de pendientes y de asistencia
Referencia: **RF-MS04-07**, **RF-MS04-08**

| Campo | Tipo | Obligatorio | Longitud / formato | Reglas de validación | Mensaje de error | Ejemplo válido |
|---|---|---|---|---|---|---|
| Página | Número entero | No | ≥ 0 (por defecto 0) | Entero no negativo. | "Página no válida." | 0 |
| Tamaño de página | Número entero | No | 1 a 100 (por defecto 20) | Entero en el rango. | "Tamaño de página no válido (1 a 100)." | 20 |

**Mensajes globales de formulario:**
* Pendientes sin población objetivo, porque MS01 no responde (503): "La lista de pendientes no está disponible en este momento."
* Lista de asistencia vacía: "Nadie ha reportado Requiero asistencia en este evento."
* Solicitud PENDIENTE_ENVIO o RECHAZADA (solo COORDINADOR): "Esta solicitud no llegó a Brigadistas e Incidentes. Regístrela manualmente."
* Pendientes para un BRIGADISTA: la lista llega siempre vacía, porque los pendientes no tienen ubicación conocida (**RN-MS04-14**).
* Persona sin nombre porque MS01 no respondió: se muestra solo su id de usuario (**RN-MS04-25**).

*Mockup F-03: (espacio reservado para insertar imagen de los listados)*

---

<a id="f-04"></a>
### F-04 · Filtros del histórico del evento
Referencia: **RF-MS04-09**

Muestra todos los check-ins del evento, no solo el más reciente de cada usuario (**RN-MS04-08**).

| Campo | Tipo | Obligatorio | Longitud / formato | Reglas de validación | Mensaje de error | Ejemplo válido |
|---|---|---|---|---|---|---|
| Usuario | Texto | No | UUID | Formato UUID. | "Usuario no válido." | 3f6c2a9e-8b1d-4c7a-9e2f-1a5b7c9d0e42 |
| Estado | Lista | No | Uno de los 4 estados / Todos | Solo valores del catálogo. Omitir equivale a Todos. | "Estado no válido." | NEEDS_ASSISTANCE |
| Página | Número entero | No | ≥ 0 (por defecto 0) | Entero no negativo. | "Página no válida." | 0 |
| Tamaño de página | Número entero | No | 1 a 100 (por defecto 20) | Entero en el rango. | "Tamaño de página no válido (1 a 100)." | 20 |

*Mockup F-04: (espacio reservado para insertar imagen del histórico)*

---

<a id="f-05"></a>
### F-05 · Estado del evento (MS06)
Referencia: **RF-MS04-11** (`PUT /{eventoId}/estado`, ruta interna)

Cuerpo que envía MS06 cada vez que programa, reprograma, cancela la programación, inicia o finaliza un evento. Los nombres de los campos son los acordados con MS06 (camelCase). No es una pantalla (RG-10).

| Campo | Tipo | Obligatorio | Longitud / formato | Reglas de validación | Mensaje de error | Ejemplo válido |
|---|---|---|---|---|---|---|
| `eventoId` | Texto | Sí | UUID | Igual al `eventoId` de la ruta (RG-03). | Formato (400): "Evento no válido." Distinto de la ruta (400): "El id del evento del cuerpo no coincide con la ruta." | 7c1d5b2e-4f3a-4e8b-9a61-0d2f8c3b5a17 |
| `nombre` | Texto | Sí | 1 a 150 caracteres | Se recorta (RG-01). Es informativo (**RN-MS04-24**). | "El nombre del evento es obligatorio (1 a 150 caracteres)." | Simulacro de evacuación bloque A |
| `simulacro` | Booleano | Sí | `true` / `false` | `false` es un evento real; sigue los mismos estados que un simulacro (**RN-MS04-20**). No puede cambiar en un evento ya guardado (409). | Falta (400): "Indique si el evento es un simulacro." Cambio (409): "El tipo de evento no puede cambiar." | true |
| `estado` | Selección única | Sí | `BORRADOR` / `PROGRAMADO` / `EN_EJECUCION` / `FINALIZADO` | Solo valores del catálogo. Debe ser una transición permitida (**RN-MS04-21**). | Catálogo (400): "Estado de evento no válido." Transición (409): "La transición de estado no está permitida." | PROGRAMADO |
| `fechaInicio` | Fecha y hora | Obligatoria con `PROGRAMADO`. No se envía con `BORRADOR`. Opcional con `EN_EJECUCION` y `FINALIZADO`. | ISO-8601 con desfase horario (p. ej. `-05:00`) | Fecha válida con desfase. Se guarda en UTC. No se compara con el reloj (**RN-MS04-24**). | Falta (400): "La fecha de inicio es obligatoria para programar el evento." Con `BORRADOR` (400): "Un evento en borrador no lleva fecha de inicio." Formato (400): "Fecha de inicio no válida (ISO-8601 con desfase horario)." | 2026-11-20T10:00:00-05:00 |
| `escenario.edificios` | Lista de textos | No (por defecto vacía) | Cada texto de 1 a 50 caracteres | Se guarda tal cual, sin validar contra MS02. No restringe el check-in (**RN-MS04-24**). | "Edificio no válido (1 a 50 caracteres)." | ["Bloque A", "Bloque B"] |
| `escenario.zonas` | Lista de textos | No (por defecto vacía) | Cada texto de 1 a 50 caracteres | Igual que `escenario.edificios`. | "Zona no válida (1 a 50 caracteres)." | ["Zona norte"] |
| `rolesConvocados` | Lista de selección | Sí | 1 o más de `ESTUDIANTE`, `PROFESOR`, `ADMINISTRATIVO`, `BRIGADISTA`, `COORDINADOR`, `ORGANIZADOR` | Sin repetidos. Otros roles (p. ej. `ADMIN`, `AUDITOR`) se rechazan (**RN-MS04-24**). | Vacía (400): "Indique al menos un rol convocado." Rol (400): "Rol convocado no válido." | ["ESTUDIANTE", "PROFESOR"] |

**Mensajes de error de la ruta (para MS06):**
* Sin `X-Service-Name` (401 `NO_AUTENTICADO`) o servicio distinto de `ms06-drills` (403 `PROHIBIDO`): el mensaje es el del catálogo de errores de los Estándares.
* Éxito: 201 si el evento es nuevo, 200 si ya existía o se repitió. Devuelve el evento almacenado con su estado efectivo y si el check-in quedó habilitado.

---

<a id="f-06"></a>
### F-06 · Detalle de check-ins del evento (MS06)
Referencia: **RF-MS04-12** (`GET /{eventoId}/checkins`, ruta interna)

Parámetros que envía MS06 en la consulta. No es una pantalla (RG-10).

| Campo | Tipo | Obligatorio | Longitud / formato | Reglas de validación | Mensaje de error | Ejemplo válido |
|---|---|---|---|---|---|---|
| `estado` | Lista | No | Uno de los 4 estados / Todos | Solo valores del catálogo (**RN-MS04-03**). Omitir equivale a Todos. | "Estado no válido." | SAFE |
| `page` | Número entero | No | ≥ 0 (por defecto 0) | Entero no negativo. | "Página no válida." | 0 |
| `size` | Número entero | No | 1 a 100 (por defecto 20) | Entero en el rango. | "Tamaño de página no válido (1 a 100)." | 20 |

**Respuesta:** `page`, `size`, `total` (personas que cumplen el filtro) e `items`, con un elemento por persona: `usuarioId`, `nombre` (nulo si MS01 no respondió, **RN-MS04-25**), `estado`, `zonaId` (id del punto de encuentro, solo con `SAFE`) y `registradoEn`.

**Mensajes de error de la ruta (para MS06):**
* Sin `X-Service-Name` (401 `NO_AUTENTICADO`) o servicio distinto de `ms06-drills` (403 `PROHIBIDO`): el mensaje es el del catálogo de errores de los Estándares.
* Evento inexistente (404): "Evento no encontrado."

---

<a id="f-07"></a>
### F-07 · Filtros de eventos
Referencia: **RF-MS04-13** (`GET /` con la lista de eventos)

Solo la usan los roles con `ms04:dashboard:read`. Muestra los eventos en cualquier estado, tengan o no el check-in habilitado (**RN-MS04-22**).

| Campo | Tipo | Obligatorio | Longitud / formato | Reglas de validación | Mensaje de error | Ejemplo válido |
|---|---|---|---|---|---|---|
| Estado | Lista | No | `BORRADOR` / `PROGRAMADO` / `EN_EJECUCION` / `FINALIZADO` / Todos | Solo valores del catálogo de estados del evento. Omitir equivale a Todos. | "Estado de evento no válido." | PROGRAMADO |
| Página | Número entero | No | ≥ 0 (por defecto 0) | Entero no negativo. | "Página no válida." | 0 |
| Tamaño de página | Número entero | No | 1 a 100 (por defecto 20) | Entero en el rango. | "Tamaño de página no válido (1 a 100)." | 20 |

**Mensajes globales de formulario:**
* Sin eventos (o ninguno con ese estado): "No hay eventos."

*Mockup F-07: (espacio reservado para insertar imagen de la lista de eventos)*

---

<a id="pantallas-sin-formulario"></a>
## Pantallas sin formulario

Estas consultas solo reciben el id del evento (RG-03) y las cabeceras de identidad:

| Pantalla | Referencia | Mensaje cuando no hay datos |
|---|---|---|
| Eventos activos | **RF-MS04-01** | "No hay ningún evento activo." |
| Mi estado vigente | **RF-MS04-05** | (404) "No tiene check-in en este evento." |
| Consolidado (dashboard) | **RF-MS04-06** | Los 4 estados en 0. |
| Detalle de un evento | **RF-MS04-13** | (404) "Evento no encontrado." |
| Resultados del evento | **RF-MS04-10** | Si el evento no está finalizado: "Resultados parciales: el evento aún no ha finalizado." |
