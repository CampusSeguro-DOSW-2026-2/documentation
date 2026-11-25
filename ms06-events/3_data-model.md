# Especificación del Modelo de Datos

MS06 tiene su propia base de datos PostgreSQL. Ningún otro microservicio la lee ni la modifica: solo se accede a ella mediante la API. Esquema desplegado: `database/schema.sql` del repositorio de MS06.

## Índice de Contenido
1. [Alcance](#alcance)
2. [Convenciones](#convenciones)
3. [Almacenamiento seguro](#almacenamiento-seguro)
4. [Tablas](#tablas)
    - [event_types](#event_types)
    - [events](#events)
    - [event_type_links](#event_type_links)
    - [event_scope_items](#event_scope_items)
    - [event_participants](#event_participants)
    - [event_brigadiers](#event_brigadiers)
    - [event_alerts](#event_alerts)
    - [event_start_failures](#event_start_failures)
    - [event_state_history](#event_state_history)
    - [event_evaluations](#event_evaluations)
    - [pending_audit_records](#pending_audit_records)
5. [Relaciones](#relaciones)
6. [Correspondencia entre API y columnas](#correspondencia-entre-api-y-columnas)
7. [Reglas de integridad](#reglas-de-integridad)
8. [Operaciones transaccionales y consultas](#operaciones-transaccionales-y-consultas)
9. [Configuración](#configuración)
10. [Datos iniciales](#datos-iniciales)

---

## Alcance

**Se almacena:** los eventos y simulacros con su estado, los tipos de evento, el alcance (edificios y zonas), los roles convocados, la copia local de los brigadistas del evento, las alertas que MS06 creó en MS03, las fallas al iniciar un evento, el historial de cambios de estado, la evaluación automática al finalizar y la bandeja de salida de auditoría hacia MS08.

**No se almacena:**
* **Usuarios, roles y permisos:** pertenecen a MS01. MS06 solo guarda el identificador del usuario que creó el evento o hizo un cambio, y el nombre de los roles convocados.
* **Edificios y zonas:** pertenecen a MS02. MS06 guarda el identificador y el nombre que MS02 devolvió al validar el alcance.
* **Alertas, check-ins e incidentes:** pertenecen a MS03, MS04 y MS05. MS06 solo guarda el identificador de la alerta y las cifras de la evaluación.
* **Historial de auditoría consultable e indicadores:** los entrega MS08 ([RF-EVE-03](1_requirements.md)). La bandeja `pending_audit_records` es solo una cola de envío: un registro entregado queda marcado y no se consulta desde MS06.
* **Sesiones y tokens:** los valida el Gateway. MS06 no guarda nada de ellos.

**Fuente de verdad del estado del evento:** la tabla `events`. Los demás módulos (alertas, check-in, brigadistas) reciben el estado desde MS06 y toleran que alguno esté caído: la falla queda en [`event_start_failures`](#event_start_failures) y se reintenta.

---

## Convenciones

* Tablas y columnas en `snake_case` y en inglés. Los campos del contrato OpenAPI usan `camelCase` y su correspondencia con las columnas está en [Correspondencia entre API y columnas](#correspondencia-entre-api-y-columnas). Los valores que se guardan son códigos en inglés (`DRAFT`, `SCHEDULED`, `IN_PROGRESS`, `FINISHED`); los nombres de los tipos de evento y los roles se guardan como texto.
* Fechas y horas en `timestamptz`, guardadas en UTC. La zona horaria de la aplicación (`America/Bogota`) solo se usa para mostrar y para decidir cuándo inicia un evento.
* Los identificadores propios de MS06 son UUID que genera la aplicación. Los identificadores que vienen de otros módulos (`scope_ref_id`, `brigadier_id`, `alert_id`, `created_by`, `changed_by`, `actor`) se guardan como `varchar(64)` sin clave foránea, porque esos datos viven en otra base.
* Todas las tablas tienen `created_at` y `updated_at`, que fija la aplicación en cada inserción y modificación. La excepción es `event_state_history`, que es inmutable y solo tiene `changed_at`.
* El esquema **no usa Flyway**. Spring ejecuta `database/schema.sql` en cada arranque (`spring.sql.init.mode=always`), antes de que Hibernate valide las entidades (`spring.jpa.hibernate.ddl-auto: validate`). Por eso cada sentencia es idempotente (`IF NOT EXISTS`, `ON CONFLICT`, `ADD COLUMN IF NOT EXISTS`) y se puede repetir sin efecto. El archivo siempre muestra el esquema como si se hubiera creado desde cero.
* `CREATE TABLE IF NOT EXISTS` no modifica una tabla que ya existe: un cambio sobre una base existente (borrar una columna, cambiar un tipo) se aplica a mano con `psql` y después se edita `schema.sql` para que coincida. Agregar una columna se hace con `ALTER TABLE ... ADD COLUMN IF NOT EXISTS` dentro del mismo archivo (así se agregaron `event_types.alert_eligible` y `event_scope_items.scope_name`).
* Los eventos no se borran: pasan a `FINISHED`. Las tablas hijas se borran en cascada solo si se borra el evento, algo que la API no ofrece.
* El control de ediciones simultáneas usa `events.version` (bloqueo optimista de JPA). Un conflicto responde 409 `MODIFICACION_CONCURRENTE`.

---

## Almacenamiento seguro

* MS06 **no guarda contraseñas, tokens, correos ni datos personales** de los usuarios: solo identificadores (de usuario y de brigadista) y el nombre del brigadista que MS05 informa para mostrarlo en el monitoreo.
* El campo `payload` de la bandeja de auditoría guarda únicamente identificadores, códigos y conteos, nunca datos personales ni el contenido de los mensajes ([`pending_audit_records`](#pending_audit_records)).
* Los registros de log del servicio no incluyen cuerpos de solicitudes ni cabeceras de identidad.
* La clave de servicio (`GATEWAY_SERVICE_KEY`) y la contraseña de la base se entregan solo por variables de entorno ([Configuración](#configuración)); no están en el repositorio.

---

## Tablas

### event_types
Catálogo de tipos de evento. Una fila por tipo. El código `SIMULACRO` hace que un evento sea un simulacro.

| Columna | Tipo | Nulo | Por defecto | Restricciones | Descripción |
|---|---|---|---|---|---|
| `id` | uuid | No | — | PK | Identificador del tipo. |
| `code` | varchar(40) | No | — | UNIQUE | Código en MAYÚSCULAS y guion bajo. Lo genera el servicio a partir del nombre ([F-05](2_forms-inputs.md#f-05-crear-tipo-de-evento)): sin tildes, solo `A-Z`, `0-9` y `_`, y recortado a 40. |
| `name` | varchar(80) | No | — | UNIQUE | Nombre que ve el usuario. Único sin distinguir mayúsculas (lo verifica el servicio). |
| `active` | boolean | No | — | — | `true` si se puede elegir al crear o editar un evento. |
| `alert_eligible` | boolean | No | `true` | — | `true` si el tipo se ofrece a MS03 como tipo de alerta (`GET /api/v1/tipos-alerta`). `SIMULACRO` queda en `false`: un simulacro no es un tipo de alerta. |
| `created_by` | varchar(64) | No | — | — | Identificador de quien creó el tipo, o `system` para los predefinidos. |
| `created_at` | timestamptz | No | — | — | Fecha de creación. |
| `updated_at` | timestamptz | No | — | — | Última modificación. |

**Índices:** PK `id`; únicos `code` y `name`.

### events
Una fila por evento o simulacro. Es la tabla central.

| Columna | Tipo | Nulo | Por defecto | Restricciones | Descripción |
|---|---|---|---|---|---|
| `id` | uuid | No | — | PK | Identificador del evento (`idEvento` en la API). |
| `name` | varchar(150) | No | — | — | Nombre del evento ([F-01](2_forms-inputs.md#f-01-crear-evento)). |
| `description` | varchar(1000) | Sí | NULL | — | Objetivo y contexto. Opcional. |
| `status` | varchar(20) | No | — | — | `DRAFT`, `SCHEDULED`, `IN_PROGRESS` o `FINISHED`. Lo verifica el servicio ([reglas de integridad](#reglas-de-integridad)). |
| `simulacro` | boolean | No | — | — | `true` si alguno de sus tipos es `SIMULACRO`. Decide qué juego de permisos aplica (`ms06:drill:*` o `ms06:real_event:*`) y no cambia después de crear el evento. |
| `scheduled_start` | timestamptz | Sí | NULL | — | Fecha y hora de inicio programada. NULL en un borrador sin fecha. |
| `started_at` | timestamptz | Sí | NULL | — | Momento real en que pasó a `IN_PROGRESS`. Es el instante cero de los tiempos que calcula MS08. |
| `ended_at` | timestamptz | Sí | NULL | — | Momento en que pasó a `FINISHED`. |
| `created_by` | varchar(64) | No | — | — | Identificador (UUID de MS01) de quien creó el evento. |
| `version` | bigint | No | — | — | Bloqueo optimista. |
| `created_at` | timestamptz | No | — | — | Fecha de creación. |
| `updated_at` | timestamptz | No | — | — | Última modificación. |

**Índices:** PK `id`; `idx_events_status (status)`; `idx_events_status_start (status, scheduled_start)`, que usa el proceso que busca los eventos por iniciar.

### event_type_links
Tipos de cada evento (relación N:M entre `events` y `event_types`). El servicio verifica el máximo de 4 tipos por evento.

| Columna | Tipo | Nulo | Por defecto | Restricciones | Descripción |
|---|---|---|---|---|---|
| `event_id` | uuid | No | — | PK (con `type_id`); FK → `events.id` `ON DELETE CASCADE` | Evento. |
| `type_id` | uuid | No | — | PK (con `event_id`); FK → `event_types.id` | Tipo asignado. Sin cascada: un tipo en uso no se borra. |

**Índices:** la PK compuesta (`event_id`, `type_id`), que impide repetidos.

### event_scope_items
Alcance del evento: los edificios y zonas del campus donde ocurre.

| Columna | Tipo | Nulo | Por defecto | Restricciones | Descripción |
|---|---|---|---|---|---|
| `id` | uuid | No | — | PK | |
| `event_id` | uuid | No | — | FK → `events.id` `ON DELETE CASCADE` | Evento. |
| `scope_type` | varchar(20) | No | — | UNIQUE con `event_id` y `scope_ref_id` | `BUILDING` o `ZONE`. |
| `scope_ref_id` | varchar(64) | No | — | — | Identificador del edificio o zona en MS02. |
| `scope_name` | varchar(100) | Sí | NULL | — | Nombre del edificio o zona que MS02 devolvió al validar el alcance. Se guarda al crear o editar el evento y es lo que MS06 envía a MS04 y a MS05; no se consulta de nuevo al iniciar el evento. NULL en los eventos guardados antes de esta columna: en ese caso se envía el identificador. |
| `created_at` | timestamptz | No | — | — | |
| `updated_at` | timestamptz | No | — | — | |

**Restricción:** `UNIQUE (event_id, scope_type, scope_ref_id)`, que impide el mismo edificio o zona dos veces en un evento.

### event_participants
Roles convocados al evento, por el nombre que MS01 les da.

| Columna | Tipo | Nulo | Por defecto | Restricciones | Descripción |
|---|---|---|---|---|---|
| `id` | uuid | No | — | PK | |
| `event_id` | uuid | No | — | FK → `events.id` `ON DELETE CASCADE` | Evento. |
| `role` | varchar(30) | No | — | UNIQUE con `event_id` | Nombre del rol en MAYÚSCULAS (por ejemplo `ESTUDIANTE`). Tiene el formato de los roles de MS01: 2 a 30 caracteres, letras, dígitos y guion bajo, empezando con una letra. |
| `created_at` | timestamptz | No | — | — | |
| `updated_at` | timestamptz | No | — | — | |

**Restricción:** `UNIQUE (event_id, role)`.

### event_brigadiers
Copia local de los brigadistas del evento, que un proceso periódico sincroniza desde MS05.

| Columna | Tipo | Nulo | Por defecto | Restricciones | Descripción |
|---|---|---|---|---|---|
| `id` | uuid | No | — | PK | |
| `event_id` | uuid | No | — | FK → `events.id` `ON DELETE CASCADE` | Evento. |
| `brigadier_id` | varchar(64) | No | — | UNIQUE con `event_id` | Identificador del brigadista (UUID de MS01). |
| `name` | varchar(150) | Sí | NULL | — | Nombre que MS05 informa, solo para mostrarlo. |
| `zone_id` | varchar(64) | Sí | NULL | — | Zona que atiende, o NULL si no tiene asignación. |
| `status` | varchar(20) | No | — | — | `ACTIVE` o `INACTIVE`. |
| `synced_at` | timestamptz | No | — | — | Última vez que se copió desde MS05. |
| `created_at` | timestamptz | No | — | — | |
| `updated_at` | timestamptz | No | — | — | |

**Restricción:** `UNIQUE (event_id, brigadier_id)`. Un evento puede iniciar sin brigadistas.

### event_alerts
Alertas que MS06 creó en MS03 para el evento, y el propósito de cada una.

| Columna | Tipo | Nulo | Por defecto | Restricciones | Descripción |
|---|---|---|---|---|---|
| `id` | uuid | No | — | PK | |
| `event_id` | uuid | No | — | FK → `events.id` `ON DELETE CASCADE` | Evento. |
| `alert_id` | varchar(64) | No | — | — | Identificador de la alerta en MS03. |
| `purpose` | varchar(20) | No | — | — | `BRIGADISTS` (pide a los brigadistas que se activen), `GENERAL` (avisa el inicio a los participantes), `EDIT` (avisa una edición) o `FINISH` (avisa la finalización). |
| `created_at` | timestamptz | No | — | — | |
| `updated_at` | timestamptz | No | — | — | |

**Índices:** `idx_event_alerts_event (event_id, purpose)`.

### event_start_failures
Llamadas a MS04, MS05 o MS03 que fallaron al iniciar un evento. El evento inicia igual y el proceso periódico repite la llamada hasta que funciona.

| Columna | Tipo | Nulo | Por defecto | Restricciones | Descripción |
|---|---|---|---|---|---|
| `id` | uuid | No | — | PK | |
| `event_id` | uuid | No | — | FK → `events.id` `ON DELETE CASCADE` | Evento. |
| `module` | varchar(10) | No | — | UNIQUE con `event_id` | Módulo que falló: `CHI` (MS04), `BEI` (MS05) o `ALE` (MS03). |
| `last_error` | varchar(500) | Sí | NULL | — | Motivo del último intento fallido, sin datos personales. |
| `attempts` | integer | No | — | — | Intentos realizados. |
| `resolved_at` | timestamptz | Sí | NULL | — | NULL mientras la falla sigue abierta; con valor, el reintento funcionó. |
| `created_at` | timestamptz | No | — | — | |
| `updated_at` | timestamptz | No | — | — | |

**Restricción:** `UNIQUE (event_id, module)`: una sola falla por evento y módulo.
**Índices:** `idx_event_start_failures_open (event_id) WHERE resolved_at IS NULL` (parcial: solo las fallas abiertas, que son pocas).

### event_state_history
Historial inmutable de los cambios de estado. Solo se inserta; no se modifica ni se borra.

| Columna | Tipo | Nulo | Por defecto | Restricciones | Descripción |
|---|---|---|---|---|---|
| `id` | uuid | No | — | PK | |
| `event_id` | uuid | No | — | FK → `events.id` `ON DELETE CASCADE` | Evento. |
| `from_status` | varchar(20) | Sí | NULL | — | Estado anterior. NULL cuando el evento se crea. |
| `to_status` | varchar(20) | No | — | — | Estado nuevo. |
| `changed_by` | varchar(64) | No | — | — | Identificador de quien hizo el cambio, o el del proceso automático que inició el evento. |
| `reason` | varchar(300) | Sí | NULL | — | Motivo o nota del cambio. |
| `changed_at` | timestamptz | No | — | — | Momento del cambio. |

**Índices:** `idx_event_history_event (event_id, changed_at)`.

### event_evaluations
Evaluación automática que se guarda al finalizar el evento, con una instantánea de las cifras de MS04 y MS05.

| Columna | Tipo | Nulo | Por defecto | Restricciones | Descripción |
|---|---|---|---|---|---|
| `id` | uuid | No | — | PK | |
| `event_id` | uuid | No | — | UNIQUE; FK → `events.id` `ON DELETE CASCADE` | Evento. Una evaluación por evento. |
| `evaluated_at` | timestamptz | No | — | — | Cuándo se calculó. |
| `total_checkins` | integer | Sí | NULL | — | Check-ins registrados. NULL si MS04 no respondió. |
| `checkin_participation_pct` | numeric(5,2) | Sí | NULL | — | Porcentaje de participación en el check-in (0 a 100, con dos decimales). |
| `total_incidents` | integer | Sí | NULL | — | Incidentes reportados. NULL si MS05 no respondió. |
| `cases_attended` | integer | Sí | NULL | — | Casos atendidos por los brigadistas. |
| `cases_resolved` | integer | Sí | NULL | — | Casos resueltos. |
| `avg_attention_seconds` | bigint | Sí | NULL | — | Tiempo promedio de atención, en segundos. |
| `details` | text | Sí | NULL | — | Detalle en JSON (desempeño por brigadista y avisos de cifras incompletas). |
| `created_at` | timestamptz | No | — | — | |
| `updated_at` | timestamptz | No | — | — | |

Si faltan cifras de MS04 o MS05 al finalizar, la fila se guarda con esos campos en NULL y un proceso periódico los completa después.

### pending_audit_records
Bandeja de salida hacia MS08. Cada acción auditable se guarda aquí en la misma transacción de la operación, de modo que la operación nunca falla por la auditoría; un proceso periódico entrega los registros pendientes.

| Columna | Tipo | Nulo | Por defecto | Restricciones | Descripción |
|---|---|---|---|---|---|
| `id` | uuid | No | — | PK | Es el `id` del hecho que recibe MS08: reenviar el mismo registro es idempotente. |
| `action` | varchar(60) | No | — | — | Acción en MAYÚSCULAS y guion bajo, según el catálogo de auditoría (`EVENT_CREATED`, `EVENT_UPDATED`, `EVENT_STATUS_CHANGED`, `EVENT_STARTED`, `EVENT_FINISHED`, `EVENT_TYPE_CREATED`). |
| `actor` | varchar(64) | No | — | — | Usuario que hizo la acción, o el del proceso automático. |
| `event_id` | uuid | Sí | NULL | — | Evento relacionado. NULL en las acciones que no son de un evento (crear un tipo de evento). Sin clave foránea. |
| `payload` | text | Sí | NULL | — | JSON con `type` (creación, modificación o finalización), `resource` (`<tipo>:<id>`), `attributes` (mapa plano de identificadores, códigos y conteos), `result`, `correlation_id`, `event_kind` (`SIMULACRO` o `REAL`) y `detail` (código genérico de máximo 100 caracteres). |
| `occurred_at` | timestamptz | No | — | — | Cuándo ocurrió la acción. Es la `fecha_hora` que recibe MS08: `started_at` en `EVENT_STARTED` y `ended_at` en `EVENT_FINISHED`. |
| `attempts` | integer | No | — | — | Intentos de envío. |
| `last_error` | varchar(500) | Sí | NULL | — | Motivo del último fallo. Empieza con `REJECTED:` si MS08 rechazó el registro con 400. |
| `sent_at` | timestamptz | Sí | NULL | — | NULL mientras el registro está pendiente. Con valor, MS08 lo aceptó (202) o lo rechazó con 400; en ese segundo caso el registro se cierra para que no bloquee la cola. |
| `created_at` | timestamptz | No | — | — | |
| `updated_at` | timestamptz | No | — | — | |

**Índices:** `idx_pending_audit_unsent (occurred_at) WHERE sent_at IS NULL` (parcial: solo los pendientes).
Un fallo distinto de 400 (MS08 caído, 5xx, tiempo agotado) detiene el envío en ese registro y se reintenta en el siguiente ciclo, en el mismo orden. Un registro guardado antes del contrato de MS08 (sin `correlation_id` en el `payload`) se traduce al enviarlo.

---

## Relaciones

| Relación | Cardinalidad | Borrado |
|---|---|---|
| `events` → `event_type_links` | 1 a N (de 1 a 4 filas por evento, validado en el servicio) | Cascada |
| `event_types` → `event_type_links` | 1 a N | Restringido: un tipo con eventos no se borra |
| `events` → `event_scope_items` | 1 a N (puede ser 0 en un borrador) | Cascada |
| `events` → `event_participants` | 1 a N (puede ser 0 en un borrador) | Cascada |
| `events` → `event_brigadiers` | 1 a N (puede ser 0: un evento puede iniciar sin brigadistas) | Cascada |
| `events` → `event_alerts` | 1 a N (una por propósito) | Cascada |
| `events` → `event_start_failures` | 1 a N (una por módulo) | Cascada |
| `events` → `event_state_history` | 1 a N | Cascada |
| `events` → `event_evaluations` | 1 a 1 | Cascada |

`pending_audit_records` no tiene clave foránea: `event_id` es opcional y la bandeja debe conservar el registro aunque el evento cambie.

---

## Correspondencia entre API y columnas

Los campos del contrato están en `camelCase` y las columnas en `snake_case`. Los que no figuran aquí no se guardan: se calculan o vienen del encabezado de identidad.

| Campo del contrato | Columna | Notas |
|---|---|---|
| `id` | `events.id` | |
| `name` | `events.name` | |
| `description` | `events.description` | |
| `status` | `events.status` | Los valores son los mismos en la API y en la base. |
| `startDate` | `events.scheduled_start` | Solo en la entrada. En la respuesta se devuelve como fecha de inicio programada. |
| `startedAt`, `endedAt` | `events.started_at`, `events.ended_at` | Los fija el sistema. |
| `typeIds` | `event_type_links.type_id` | Arreglo de UUID. |
| `types` (respuesta) | `event_types.code`, `event_types.name` | |
| `scope[].type` | `event_scope_items.scope_type` | `BUILDING` o `ZONE`. |
| `scope[].id` | `event_scope_items.scope_ref_id` | |
| `scope[].name` (respuesta) | `event_scope_items.scope_name` | |
| `participantRoles` | `event_participants.role` | Arreglo de nombres de rol. |
| `createdBy` | `events.created_by` | Viene de `X-User-Id`. |
| `brigadiers` (monitoreo) | `event_brigadiers.*` | Copia sincronizada desde MS05. |
| `evaluation` | `event_evaluations.*` | Solo en eventos finalizados. |
| `warnings` | — | No se guarda: se calcula en cada respuesta. |
| `version` | `events.version` | |
| `name`, `code`, `active` (tipo de evento) | `event_types.name`, `.code`, `.active` | |

---

## Reglas de integridad

| Regla | Dónde se garantiza |
|---|---|
| Un evento tiene de 1 a 4 tipos activos y existentes | Servicio. BD: PK compuesta y FK de `event_type_links`. |
| Estados válidos (`DRAFT`, `SCHEDULED`, `IN_PROGRESS`, `FINISHED`) y transiciones: `DRAFT` ↔ `SCHEDULED` → `IN_PROGRESS` → `FINISHED`; `FINISHED` es definitivo | Servicio. La base no tiene un CHECK sobre `status`: `schema.sql` no lo declara. |
| Un simulacro no pasa a ser evento real ni al revés | Servicio: compara el tipo antes y después de editar. |
| Un evento finalizado no se edita; uno en ejecución no cambia su fecha de inicio ni queda sin alcance o sin participantes | Servicio. |
| La fecha de inicio es futura al crear o programar | Servicio, con la hora del reloj de la aplicación. |
| Un evento se programa solo si tiene fecha, alcance y participantes; si falta algo o un módulo no responde, queda en borrador con una advertencia | Servicio. |
| Alcance sin repetidos | BD: `UNIQUE (event_id, scope_type, scope_ref_id)`. |
| Roles convocados sin repetidos y con el formato de los roles de MS01 | BD: `UNIQUE (event_id, role)`. Servicio: el formato. |
| Brigadistas sin repetidos por evento | BD: `UNIQUE (event_id, brigadier_id)`. |
| Una sola falla de inicio por evento y módulo | BD: `UNIQUE (event_id, module)`. |
| Una sola evaluación por evento | BD: `UNIQUE (event_id)` en `event_evaluations`. |
| Código y nombre de tipo únicos | BD: `UNIQUE` en `code` y en `name`. Servicio: el nombre sin distinguir mayúsculas. |
| Ediciones simultáneas | Servicio y BD: `events.version`, con 409 `MODIFICACION_CONCURRENTE`. |
| Cada creación, modificación, cambio de estado y finalización queda en la bandeja de auditoría dentro de la misma transacción | Servicio. |
| El historial de estados no se modifica | Servicio: la entidad solo se inserta. |
| La auditoría nunca bloquea la operación | Servicio: la entrega es un proceso aparte con reintentos. |

---

## Operaciones transaccionales y consultas

Cada operación de escritura es una sola transacción: se aplica completa o no se aplica. Las llamadas a otros módulos que no pueden revertirse se hacen según la regla de cada operación (si MS04 o MS05 no responden al finalizar, el evento no finaliza; si MS03 no responde, finaliza con una advertencia).

| Operación | Lecturas | Escrituras |
|---|---|---|
| Crear un evento ([RF-EVE-01](1_requirements.md)) | Tipos pedidos en `event_types`; alcance en MS02 (guarda `id` y nombre) | INSERT `events` (`DRAFT`, `simulacro` según los tipos) + `event_type_links` + `event_scope_items` + `event_participants` + `event_state_history` + `pending_audit_records` (`EVENT_CREATED`). Si trae fecha, alcance y roles, pasa a `SCHEDULED` y se insertan las filas de `event_alerts` |
| Editar un evento ([RF-EVE-02](1_requirements.md)) | `events` por id | UPDATE `events` (con `version`) y reemplazo de tipos, alcance y roles; si cambia de estado (`DRAFT` ↔ `SCHEDULED`), `event_state_history` y `pending_audit_records` (`EVENT_STATUS_CHANGED`, con el estado anterior y el nuevo); si hay cambios, `pending_audit_records` (`EVENT_UPDATED`) |
| Iniciar un evento (proceso periódico, [RF-EVE-05](1_requirements.md)) | `events` en `SCHEDULED` con `scheduled_start` vencido | UPDATE `status = IN_PROGRESS` y `started_at`; INSERT `event_state_history` y `pending_audit_records` (`EVENT_STARTED`). Una llamada fallida a MS04, MS05 o MS03 inserta o actualiza `event_start_failures` |
| Reintentar fallas de inicio (proceso periódico) | `event_start_failures` con `resolved_at IS NULL` | UPDATE `attempts`, `last_error` y, si funciona, `resolved_at` |
| Sincronizar brigadistas (proceso periódico) | MS05 por evento: cada 10 s los `IN_PROGRESS` y cada 30 s los `SCHEDULED` | INSERT o UPDATE de `event_brigadiers` |
| Finalizar un evento ([RF-EVE-04](1_requirements.md)) | `events` por id; cifras de MS04 y MS05 | UPDATE `status = FINISHED` y `ended_at`; INSERT `event_evaluations`, `event_state_history`, `pending_audit_records` (`EVENT_FINISHED`, con la duración en segundos) |
| Completar evaluaciones (proceso periódico) | `event_evaluations` con cifras en NULL | UPDATE de las cifras y `details` |
| Crear un tipo de evento | ¿Existe el nombre o el código? | INSERT `event_types` + `pending_audit_records` (`EVENT_TYPE_CREATED`) |
| Entregar la auditoría (proceso periódico) | `pending_audit_records` con `sent_at IS NULL`, por `occurred_at` | UPDATE `attempts`, `last_error` y `sent_at` |

**Listado `GET /eventos`** (parámetros en [F-03](2_forms-inputs.md#f-03-búsqueda-y-filtros-de-eventos)):
* `title`: coincidencia parcial sin distinguir mayúsculas (`LIKE '%texto%'` sobre el nombre en minúsculas, escapando `%` y `_`).
* `status`: igualdad; sin valor se incluyen todos los estados.
* Solo se devuelven los eventos de los tipos que el usuario puede leer: simulacros con `ms06:drill:read` y eventos reales con `ms06:real_event:read`.
* Orden: `created_at` descendente.

---

## Configuración

Variables de entorno que afectan los datos y los procesos (valores por defecto de `application.yml`):

| Variable | Por defecto | Uso |
|---|---|---|
| `DB_URL` · `DB_USER` · `DB_PASSWORD` | — (el perfil `local` los entrega) | Conexión a PostgreSQL. La contraseña nunca va en el repositorio. |
| `APP_TIMEZONE` | `America/Bogota` | Zona horaria de la aplicación. |
| `SCHEDULING_ENABLED` | `true` | Enciende o apaga los procesos periódicos (las pruebas los apagan). |
| `EVENT_START_CHECK_MS` | `10000` | Cada cuánto se buscan eventos por iniciar. |
| `BEI_SYNC_RUNNING_MS` · `BEI_SYNC_SCHEDULED_MS` | `10000` · `30000` | Cada cuánto se copian los brigadistas de los eventos en ejecución y de los programados. |
| `EVALUATION_RETRY_MS` | `30000` | Cada cuánto se completan las evaluaciones con cifras faltantes. |
| `app.events.audit-audit-ms` (propiedad) | `15000` | Cada cuánto se entregan a MS08 los registros pendientes. El README del repositorio la lista como `AUDIT_DELIVERY_MS`, pero `application.yml` no define esa variable: hoy solo se cambia con la propiedad. |
| `GATEWAY_URL` | `http://localhost:8080` | Dirección del Gateway, por donde pasan todas las llamadas a otros módulos. |
| `GATEWAY_SERVICE_KEY` | — | Clave de servicio de MS06 (mínimo 32 caracteres), obligatoria fuera de los perfiles `local` y de prueba. |
| `GATEWAY_AUDIT_PATH` | `/api/v1/audit/events` | Ruta de la ingesta de MS08. |

---

## Datos iniciales

1. **Tipos de evento predefinidos:** `schema.sql` inserta, con `ON CONFLICT (code) DO NOTHING`, siete tipos con `created_by = 'system'` y `active = true`. Como el `INSERT` se repite en cada arranque, un tipo borrado a mano reaparece, y uno que el administrador desactivó conserva su estado.

   | Código | Nombre | `alert_eligible` |
   |---|---|---|
   | `SIMULACRO` | Simulacro | No |
   | `SISMO` | Sismo | Sí |
   | `INCENDIO` | Incendio | Sí |
   | `DISTURBIOS` | Disturbios | Sí |
   | `INUNDACION` | Inundación | Sí |
   | `FUGA_QUIMICA` | Fuga de Sustancias Químicas | Sí |
   | `OTRO` | Otro | Sí |

2. **Tipos nuevos:** `POST /api/v1/tipos-evento` agrega un tipo con `alert_eligible = true` y `active = true`. Solo lo hace quien tiene `ms06:real_event:manage`.
3. **Sin primer usuario ni roles:** MS06 no crea usuarios ni roles. Los permisos `ms06:*` los siembra MS01 en la [matriz de roles y permisos](../integration/permissions-matrix.md).
4. **Bandeja de auditoría:** los registros entregados no se borran en el servicio; la retención de 180 días la aplica MS08 sobre sus propios registros.
