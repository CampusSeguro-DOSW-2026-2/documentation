# Diccionario de Datos, Formularios y Entradas

## Índice de Contenido
1. [Reglas generales de entrada](#reglas-generales-de-entrada)
2. [Formularios y Entradas](#formularios-y-entradas)
   - [F-01 · Reporte de estado (check-in)](#f-01)
   - [F-02 · Filtros de personas por estado](#f-02)
   - [F-03 · Listados de pendientes y de asistencia](#f-03)
   - [F-04 · Filtros del histórico del evento](#f-04)
3. [Pantallas sin formulario](#pantallas-sin-formulario)

---

<a id="reglas-generales-de-entrada"></a>
## Reglas generales de entrada

* **RG-01**: Todo texto se recorta (trim) de espacios al inicio y al final antes de validar.
* **RG-02**: Un campo obligatorio vacío o con solo espacios se considera faltante.
* **RG-03**: El id del evento (`eventId` en la ruta) es un UUID: el id de la alerta en MS03 (**RN-MS04-01**). Un valor con formato inválido se rechaza (HTTP 400): "Evento no válido."
* **RG-04**: Se rechazan (HTTP 400) los campos no definidos en el contrato (en especial un id de usuario enviado en el cuerpo del check-in, **RN-MS04-06**).
* **RG-05**: Se rechazan (HTTP 400) los campos que no corresponden al estado reportado: el punto de encuentro solo se envía con Estoy a salvo y la ubicación solo con Requiero asistencia (ver [Estados del check-in](1_requirements.md#estados-del-check-in)).
* **RG-06**: Los errores de validación son HTTP 400 `VALIDACION`, siempre con el `ErrorResponse` de los Estándares; el campo y su mensaje van en `errores`.
* **RG-07**: La identidad del solicitante (id y roles) nunca viaja en el cuerpo: llega en las cabeceras X-User-Id y X-User-Role (**RN-MS04-06**).
* **RG-08**: Los ids de MS02 (punto de encuentro, edificio, espacio) son enteros positivos, igual que en el contrato de MS02.
* **RG-09**: MS04 no tiene formularios de auditoría: su consulta pertenece a MS08 (**RNF-MS04-17**).

---

<a id="formularios-y-entradas"></a>
## Formularios y Entradas

<a id="f-01"></a>
### F-01 · Reporte de estado (check-in)
Referencia: **RF-MS04-02**, **RF-MS04-03**, **RF-MS04-04**

El mismo formulario registra el primer check-in y lo actualiza. Se completa en máximo 3 interacciones desde la pantalla inicial (**RNF-MS04-07**).

| Campo | Tipo | Obligatorio | Longitud / formato | Reglas de validación | Mensaje de error | Ejemplo válido |
|---|---|---|---|---|---|---|
| Estado | Selección única | Sí | `SAFE` / `EVACUATING` / `NEEDS_ASSISTANCE` / `OFF_CAMPUS` | Se muestran siempre las 4 opciones con su texto: Estoy a salvo, Me estoy evacuando, Requiero asistencia, No estoy en el campus. Se envía el código (**RN-MS04-03**). | "Seleccione un estado válido." | SAFE |
| Punto de encuentro | Selección (puntos de encuentro de MS02) | Solo si el estado es Estoy a salvo | Entero positivo | Debe existir y estar ACTIVO en MS02 (**RN-MS04-04**). No se envía con otros estados (RG-05). | Falta (400): "Seleccione el punto de encuentro donde está." Inexistente (404): "Punto de encuentro no encontrado." Inactivo (422): "El punto de encuentro no está activo." Con otro estado (400): "El punto de encuentro solo aplica a Estoy a salvo." | 3 |
| Tipo de ubicación | Selección única | Solo si el estado es Requiero asistencia | `EDIFICIO` / `ESPACIO` / `PUNTO_ENCUENTRO` | Solo valores del catálogo (**RN-MS04-09**). No se envía con otros estados (RG-05). | Falta (400): "Indique dónde necesita la asistencia." Con otro estado (400): "La ubicación solo aplica a Requiero asistencia." | EDIFICIO |
| Ubicación | Selección (elementos de MS02 del tipo elegido) | Solo si el estado es Requiero asistencia | Entero positivo | Debe corresponder al tipo elegido. Su existencia la valida MS05 al crear la solicitud (**RF-MS04-04**, A4). | Falta (400): "Seleccione el lugar donde necesita la asistencia." | 12 |

**Mensajes globales de formulario:**
* Evento finalizado o no activo (409): "No existe un evento activo."
* El evento no le aplica al usuario (422): "Este evento no le aplica."
* Evento inexistente (404): "Evento no encontrado."
* Cambio simultáneo (409): "Su estado cambió mientras enviaba el reporte. Intente de nuevo."
* MS02 o MS03 no disponibles (503): "El servicio no está disponible en este momento. Intente de nuevo."
* Éxito, solicitud de asistencia ENVIADA: "Su estado quedó registrado y su solicitud de asistencia fue enviada."
* Éxito, solicitud PENDIENTE_ENVIO: "Su estado quedó registrado. La solicitud de asistencia se enviará en cuanto el servicio esté disponible."
* Éxito, solicitud RECHAZADA: "Su estado quedó registrado, pero la solicitud de asistencia no pudo crearse. Un coordinador la revisará."

*Mockup F-01: (espacio reservado para insertar imagen del reporte de estado)*

---

<a id="f-02"></a>
### F-02 · Filtros de personas por estado
Referencia: **RF-MS04-06**

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
* Pendientes sin población objetivo (503): "La lista de pendientes no está disponible en este momento."
* Lista de asistencia vacía: "Nadie ha reportado Requiero asistencia en este evento."
* Solicitud PENDIENTE_ENVIO o RECHAZADA (solo COORDINADOR): "Esta solicitud no llegó a Brigadistas e Incidentes. Regístrela manualmente."

*Mockup F-03: (espacio reservado para insertar imagen de los listados)*

---

<a id="f-04"></a>
### F-04 · Filtros del histórico del evento
Referencia: **RF-MS04-09**

| Campo | Tipo | Obligatorio | Longitud / formato | Reglas de validación | Mensaje de error | Ejemplo válido |
|---|---|---|---|---|---|---|
| Usuario | Texto | No | UUID | Formato UUID. | "Usuario no válido." | 3f6c2a9e-8b1d-4c7a-9e2f-1a5b7c9d0e42 |
| Estado | Lista | No | Uno de los 4 estados / Todos | Solo valores del catálogo. Omitir equivale a Todos. | "Estado no válido." | NEEDS_ASSISTANCE |
| Página | Número entero | No | ≥ 0 (por defecto 0) | Entero no negativo. | "Página no válida." | 0 |
| Tamaño de página | Número entero | No | 1 a 100 (por defecto 20) | Entero en el rango. | "Tamaño de página no válido (1 a 100)." | 20 |

*Mockup F-04: (espacio reservado para insertar imagen del histórico)*

---

<a id="pantallas-sin-formulario"></a>
## Pantallas sin formulario

Estas consultas solo reciben el id del evento (RG-03) y las cabeceras de identidad:

| Pantalla | Referencia | Mensaje cuando no hay datos |
|---|---|---|
| Evento activo | **RF-MS04-01** | "No hay ningún evento activo." |
| Mi estado vigente | **RF-MS04-05** | (404) "No tiene check-in en este evento." |
| Consolidado (dashboard) | **RF-MS04-06** | Los 4 estados en 0. |
| Resultados del evento | **RF-MS04-10** | Si el evento está activo: "Resultados parciales: el evento sigue activo." |
