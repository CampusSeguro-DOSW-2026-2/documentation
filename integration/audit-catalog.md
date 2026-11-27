# Catálogo de auditoría por servicio

Registro de las acciones que cada microservicio emite a MS08 y de los hechos con los que MS08 calcula los indicadores. El contrato del evento (campos, cabeceras, respuestas, reintentos y retención) está en los [Estándares §12](standards.md#12-auditoría-ms08) y cómo enviarlo en la [guía de integración de MS08](../ms08-analytics-and-audit/integration-guide.md); aquí solo se listan las acciones.

Cada servicio tiene su bloque, que mantiene su squad: **Evento** (cómo identifica el evento de MS06), la tabla de **acciones** (`accion`, cuándo se emite, `recurso_afectado` y `atributos`) y **Alimenta** (qué indicadores usan esos hechos). Reglas para todos:

* Un hecho se envía **una sola vez** por cada cosa que ocurre.
* `fecha_hora` es cuándo ocurrió; MS08 calcula los tiempos con ella.
* Las acciones nuevas las confirma el squad dueño del servicio y se agregan a su bloque.
* Los permisos de consulta de la auditoría están en la [matriz de roles y permisos](permissions-matrix.md) (`ms08:audit_drill:read` y `ms08:audit_real_event:read`).
* Un servicio nuevo agrega un bloque con la misma forma, por ejemplo:

| `accion` | Cuándo se emite | `recurso_afectado` | Evento | `atributos` |
|---|---|---|---|---|
| `REGISTRO_CHECKIN` | La persona registra su primer estado en el evento | `checkin:<id>` | Sí | `estado`, `punto_encuentro_id` |
| `LOGIN_FALLIDO` | Credenciales incorrectas | `null` | No | — |

## MS01 – Identidad y acceso

**Evento:** ninguno (`evento_id` y `tipo_evento` = `null`). Solo los ve quien tenga ambos permisos de lectura de auditoría.

| `accion` | Cuándo | `recurso_afectado` | `atributos` |
|---|---|---|---|
| `REGISTRO` | Se confirma el registro de un usuario o se rechaza (datos inválidos, código inválido o vencido, correo en conflicto) | `usuario:<id>` (`null` si el registro falla) | — |
| `LOGIN` | Inicio de sesión, exitoso o fallido | `usuario:<id>` (`null` si el correo no existe) | — |
| `LOGOUT` | Cierre de sesión | `usuario:<id>` | — |
| `CAMBIO_ROL` | Se agregan o retiran roles a un usuario | `usuario:<id>` | — |
| `CAMBIO_ESTADO` | Se activa o inactiva un usuario | `usuario:<id>` | — |
| `CAMBIO_CONTRASENA` | Un usuario autenticado cambia su contraseña | `usuario:<id>` | — |
| `RECUPERACION` | Se solicita o se restablece la contraseña con un código | `usuario:<id>` (`null` si no hay usuario) | — |
| `CREACION_ROL`, `EDICION_ROL`, `CAMBIO_PERMISOS_ROL`, `ELIMINACION_ROL` | El administrador crea, edita, cambia los permisos o elimina un rol | `rol:<NOMBRE>` | — |

**Alimenta:** solo historial y trazabilidad.

## MS02 – Campus y seguridad

**Evento:** ninguno (`evento_id` y `tipo_evento` = `null`): son cambios de la configuración del campus. Solo los ve quien tenga ambos permisos de lectura de auditoría.

Se audita **solo lo que cambia datos** y se envía cuando el cambio ya quedó guardado (`resultado` `EXITO`). Las consultas (`GET`) no se auditan.

| `accion` | Cuándo | `recurso_afectado` | `atributos` |
|---|---|---|---|
| `EDIFICIO_CREADO`, `EDIFICIO_ACTUALIZADO`, `EDIFICIO_DESACTIVADO`, `EDIFICIO_REACTIVADO` | Se crea, edita, desactiva o reactiva un edificio | `edificio:<id>` | — |
| `PISO_CREADO`, `PISO_ACTUALIZADO`, `PISO_DESACTIVADO`, `PISO_REACTIVADO` | Ídem, para un piso | `piso:<id>` | `edificio_id` |
| `ESPACIO_CREADO`, `ESPACIO_ACTUALIZADO`, `ESPACIO_DESACTIVADO`, `ESPACIO_REACTIVADO` | Ídem, para un espacio | `espacio:<id>` | `edificio_id`, `piso_id`, `tipo_espacio` |
| `ZONA_SEGURA_CREADA`, `ZONA_SEGURA_ACTUALIZADA`, `ZONA_SEGURA_DESACTIVADA`, `ZONA_SEGURA_REACTIVADA` | Ídem, para una zona segura o punto de encuentro | `zona_segura:<id>` | `codigo` |
| `RUTA_CREADA`, `RUTA_ACTUALIZADA` | Se crea o edita una ruta de evacuación | `ruta:<id>` | `espacio_origen_id`, `zona_segura_destino_id` |
| `RUTA_ESTADO_CAMBIADO` | Cambia el estado de una ruta (`DISPONIBLE`, `BLOQUEADA`, `RESTRINGIDA`) | `ruta:<id>` | `estado_anterior`, `estado_nuevo`, `espacio_origen_id`, `zona_segura_destino_id` |
| `RECOMENDACION_CREADA`, `RECOMENDACION_ACTUALIZADA`, `RECOMENDACION_DESACTIVADA` | Cambia una recomendación preventiva | `recomendacion:<id>` | `ambito` (`EDIFICIO` o `TIPO_ESPACIO`) |

No envíe nombres, descripciones ni el `motivo` de una ruta (texto libre): solo ids y códigos.

**Alimenta:** ningún indicador directamente; deja el rastro de quién cambió el campus y cuándo (por ejemplo, quién bloqueó una ruta). Aun así, MS02 es la **fuente de los ids de ubicación** que usan los demás servicios (`ubicacion_id`, `punto_encuentro_id`): edificio, espacio o zona segura. Para mostrar nombres, el front consulta a MS02 por id (`GET /api/v1/buildings/{id}`, `/spaces/{id}`, `/safe-zones/{id}`).

## MS03 – Alertas

**Evento:** `evento_id` = simulacro o evento al que pertenece la alerta (MS03 ya sabe si es simulación). Alerta sin evento: regla de la sección 3.

| `accion` | Cuándo | `recurso_afectado` | `atributos` |
|---|---|---|---|
| `CREATE`, `UPDATE`, `SCHEDULE`, `ACTIVATE`, `ARCHIVE` | Ciclo de vida de la alerta | `alerta:<id>` | — |
| `NOTIFICATION_SENT`, `NOTIFICATION_FAILED` | Termina el envío a los destinatarios | `alerta:<id>` | `destinatarios`, `enviados`, `fallidos` |
| `CONFIRM` | Un usuario confirma la recepción (una vez por usuario; `usuario_id` = quien confirma) | `alerta:<id>` | — |
| `RECIPIENT_RESOLUTION_FAILED` | No se pudieron resolver los destinatarios | `alerta:<id>` | — |

**Alimenta:** personas notificadas y % de confirmación (= usuarios distintos con `CONFIRM` ÷ `destinatarios`).

## MS04 – Evacuación y check-in

**Evento:** `evento_id` = id del evento; `tipo_evento` según el campo `simulacro` del evento (MS04 ya lo guarda).

| `accion` | Cuándo | `recurso_afectado` | `atributos` |
|---|---|---|---|
| `REGISTRO_CHECKIN` | Primer check-in del usuario en el evento (`usuario_id` = quien reporta) | `checkin:<id>` (si se rechaza: `evento:<id>`) | `estado` (`SAFE`, `EVACUATING`, `NEEDS_ASSISTANCE`, `OFF_CAMPUS`); `punto_encuentro_id` (solo `SAFE`); `ubicacion_tipo` y `ubicacion_id` (solo `NEEDS_ASSISTANCE`) |
| `CAMBIO_ESTADO_CHECKIN` | El usuario cambia su estado (o repite el vigente: `SIN_CAMBIO`) | `checkin:<id>` | los mismos que arriba, más `estado_anterior` |
| `SOLICITUD_ASISTENCIA` | Se envía la solicitud de asistencia a MS05 (`detalle` = estado del envío) | `checkin:<id>` | `ubicacion_tipo`, `ubicacion_id` |
| `CAMBIO_ESTADO_EVENTO` | MS06 le informa que el evento cambia de estado (sin usuario) | `evento:<id>` | `estado_anterior`, `estado_nuevo`, `servicio_origen` |

Solo los hechos con `resultado` `EXITO` cuentan para los indicadores; los `FALLO` (check-in rechazado) quedan como traza.

**Alimenta:** participantes, check-ins por estado y punto de encuentro (último estado de cada usuario), personas que pidieron asistencia (usuarios distintos con `estado` `NEEDS_ASSISTANCE`), tiempo hasta el check-in y tiempo de evacuación (hasta el primer `SAFE`).

## MS05 – Brigadistas e incidentes

**Evento:** `evento_id` = evento del incidente o de la asignación; `tipo_evento` según el evento (MS05 ya lo resuelve con `TipoEventoResolver`).

| `accion` | Cuándo | `recurso_afectado` | `atributos` |
|---|---|---|---|
| `CREAR_INCIDENTE` | Se reporta (`fecha_hora` = fecha de reporte) | `incidente:<id>` | `prioridad` (`ALTA`, `MEDIA`, `BAJA`), `ubicacion_tipo`, `ubicacion_id` |
| `EDITAR_INCIDENTE` | Se edita la descripción o la prioridad | `incidente:<id>` | — |
| `ASIGNAR_RESPONSABLE` | Se asigna o reasigna brigadista | `incidente:<id>` | `responsable_id` |
| `INICIAR_ATENCION` | Primera atención (`fecha_hora` = fecha de primera atención) | `incidente:<id>` | `ubicacion_tipo`, `ubicacion_id` |
| `RESOLVER_INCIDENTE` | Se resuelve | `incidente:<id>` | — |
| `CERRAR_INCIDENTE` | El coordinador cierra | `incidente:<id>` | — |
| `CARGAR_EVIDENCIA` | Se sube una evidencia | `evidencia:<id>` | `incidente_id` |
| `CREACION`, `MODIFICACION`, `RETIRO` | Cambios en la asignación de brigadistas | `asignacion:<id>` | `brigadista_id` |

La «categoría» de un incidente es su **prioridad**.

**Alimenta:** incidentes por categoría, tiempo hasta la primera atención y edificios o zonas con mayor tiempo de respuesta.

## MS06 – Simulacros y eventos

**Evento:** `evento_id` = id del evento; `tipo_evento` según `simulacro` del evento.

| `accion` | Cuándo | `recurso_afectado` | `atributos` |
|---|---|---|---|
| `EVENT_CREATED`, `EVENT_UPDATED` | Se crea o edita el evento | `evento:<id>` | — |
| `EVENT_STATUS_CHANGED` | Cambia de estado (distinto de iniciar o finalizar) | `evento:<id>` | `estado_anterior`, `estado_nuevo` |
| `EVENT_STARTED` | Pasa a en curso (`fecha_hora` = `startedAt`) | `evento:<id>` | `automatico` (boolean) |
| `EVENT_FINISHED` | Pasa a finalizado (`fecha_hora` = `endedAt`) | `evento:<id>` | `duracion_segundos` |
| `EVENT_TYPE_CREATED` | Se crea un tipo de evento (sin evento) | `tipo_evento:<id>` | — |

**Alimenta:** `EVENT_STARTED` es el punto cero de los tiempos medios de check-in y de evacuación; `EVENT_FINISHED`, la duración y la comparación entre simulacros.

## MS07 – Formación y asistente IA

**Evento:** ninguno (`evento_id` y `tipo_evento` = `null`): la formación no está ligada a un simulacro. Solo los ve quien tenga ambos permisos de lectura de auditoría.

MS07 envía a MS08 por el Gateway, de forma asíncrona, con `id` y después de guardar. No se audita el progreso ni las vistas de contenidos (mucho volumen y sin valor de auditoría).

| `accion` | Cuándo | `recurso_afectado` | `atributos` |
|---|---|---|---|
| `TUTOR_QUERY` | Un usuario consulta al tutor. `detalle` = `RESPONDIDA` o `FUERA_DE_ALCANCE` | `null` | — |
| `EVALUACION_CREADA`, `EVALUACION_ACTUALIZADA` | Se crea o edita una evaluación | `evaluacion:<id>` | — |
| `INTENTO_REGISTRADO` | Un usuario registra un intento de evaluación | `intento:<id>` | `evaluacion_id` |
| `EVALUACION_DESACTIVADA`, `EVALUACION_REACTIVADA` | Se desactiva o reactiva una evaluación | `evaluacion:<id>` | — |
| `CONTENIDO_CREADO`, `CONTENIDO_ACTUALIZADO`, `CONTENIDO_DESACTIVADO`, `CONTENIDO_REACTIVADO` | Administración de contenidos | `contenido:<id>` | `categoria_id` |
| `CATEGORIA_CREADA`, `CATEGORIA_ACTUALIZADA`, `CATEGORIA_DESACTIVADA`, `CATEGORIA_REACTIVADA` | Administración de categorías | `categoria:<id>` | — |

Nunca envíe la pregunta del usuario, la respuesta del tutor ni el texto de los contenidos: solo ids y el código del resultado.

**Alimenta:** ningún indicador de eventos; sirve para trazabilidad (quién consultó al tutor y con qué resultado, quién administra el material).

## Indicadores y de dónde salen

| Indicador | Qué hecho lo alimenta |
|---|---|
| Participantes, notificados y % de confirmación | MS03 `NOTIFICATION_*` y `CONFIRM`; MS04 `REGISTRO_CHECKIN` y `CAMBIO_ESTADO_CHECKIN` |
| Check-ins por estado y punto de encuentro; personas que pidieron asistencia | MS04 `REGISTRO_CHECKIN` y `CAMBIO_ESTADO_CHECKIN` (`estado`, `punto_encuentro_id`) |
| Incidentes por categoría | MS05 `CREAR_INCIDENTE` (`prioridad`) |
| Tiempo hasta el check-in y de evacuación | MS06 `EVENT_STARTED` → primer check-in / primer `SAFE` |
| Tiempo hasta la primera atención; edificios o zonas más lentos | MS05 `CREAR_INCIDENTE` → `INICIAR_ATENCION` (`ubicacion_*`) |
| Comparación entre simulacros | Los mismos indicadores de varios eventos `SIMULACRO` |

