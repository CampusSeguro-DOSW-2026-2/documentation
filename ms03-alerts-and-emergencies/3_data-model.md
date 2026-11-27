# Especificación del Modelo de Datos

MS03 tiene su propia base de datos PostgreSQL (`campusseguro_alerts`). Ningún otro microservicio la lee ni la modifica: solo se accede a ella mediante la API. El esquema lo crea Flyway al arrancar (migraciones `V1` a `V10` en `src/main/resources/db/migration`). Código del acceso: paquete `edu.eci.dosw.campusseguro.alertas` (`entity/` y `repository/`) del repositorio de MS03.

## Índice de Contenido
1. [Alcance](#alcance)
2. [Convenciones](#convenciones)
3. [Almacenamiento seguro](#almacenamiento-seguro)
4. [Tablas](#tablas)
    - [Catálogos](#catálogos)
    - [alerts](#alerts)
    - [alert_types](#alert_types)
    - [alert_scope_items](#alert_scope_items)
    - [alert_recipients](#alert_recipients)
    - [notification_deliveries](#notification_deliveries)
    - [alert_confirmations](#alert_confirmations)
    - [audit_events](#audit_events)
5. [Relaciones](#relaciones)
6. [Correspondencia entre API y columnas](#correspondencia-entre-api-y-columnas)
7. [Reglas de integridad](#reglas-de-integridad)
8. [Operaciones transaccionales y consultas](#operaciones-transaccionales-y-consultas)
9. [Configuración](#configuración)
10. [Datos iniciales](#datos-iniciales)

---

## Alcance

**Se almacena:** las alertas con su ciclo de vida, tipos, alcance y destinatarios ([RF-ALT-01](1_requirements.md) a [RF-ALT-04](1_requirements.md)), el resultado de cada envío de correo ([RF-ALT-05](1_requirements.md)), las confirmaciones de recepción y el historial de auditoría de cada alerta.

**No se almacena:**
* **Usuarios, roles y permisos:** pertenecen a MS01. MS03 solo guarda el identificador del usuario (o el nombre del rol) y el correo al que se envió cada notificación, y consulta a MS01 los usuarios de un rol.
* **Edificios y zonas seguras:** pertenecen a MS02. MS03 solo guarda su identificador en el alcance y lo valida contra MS02.
* **Simulacros y eventos:** pertenecen a MS06. MS03 solo guarda el identificador del simulacro (`simulation_id`) cuando la alerta nace de uno.
* **Contenido de los correos y claves del proveedor:** MS03 no guarda el cuerpo de los mensajes; la clave de Brevo es un secreto del entorno.
* **Sesiones y tokens:** los valida el Gateway. MS03 no ve el JWT.

**Fuente de verdad de la auditoría:** hoy es la tabla local `audit_events`. MS03 todavía no envía sus hechos a MS08 (ver [Operaciones](#operaciones-transaccionales-y-consultas)).

---

## Convenciones

* Nombres de tablas y columnas en inglés y en `snake_case`.
* Identificadores propios en UUID. Los identificadores de otros servicios (`user_id`, `created_by`, `scope_ref_id`, `recipient_ref_id`) se guardan como texto sin clave foránea, porque viven en otra base de datos.
* Fechas y horas en UTC (`TIMESTAMPTZ`); la zona horaria de la aplicación (`America/Bogota`) solo se usa para presentar y programar.
* Los **estados, tipos de alerta y acciones de auditoría viven en tablas de catálogo** con clave foránea: añadir un valor es insertar una fila, sin cambiar el código de la base.
* Control de concurrencia optimista en `alerts` con la columna `version`.
* Las alertas **no se borran**: terminan en `ARCHIVED`.
* El esquema solo se cambia con migraciones Flyway nuevas; nunca se editan las ya aplicadas.

---

## Almacenamiento seguro

* Los únicos datos personales son identificadores de usuario (UUID de MS01) y el correo de la tabla `notification_deliveries` (hasta 254 caracteres). El correo se guarda para poder explicar un envío fallido; no se devuelve en la API ni se copia a la auditoría ni a los registros del servicio.
* No se guardan contraseñas, tokens ni datos de tarjetas.
* Las credenciales (`DB_PASSWORD`, `BREVO_API_KEY`, `MAIL_PASSWORD`, `MS_SERVICE_KEY`) llegan por variable de entorno y nunca están en el repositorio.
* Los registros del servicio no incluyen correos, instrucciones de las alertas, cuerpos de solicitudes ni cadenas de conexión.

---

## Tablas

### Catálogos

| Tabla | Columnas | Valores |
|---|---|---|
| `alert_status` | `code` (PK, VARCHAR 20), `label`, `display_order`, `editable` (BOOLEAN), `final_state` (BOOLEAN) | `DRAFT` (editable), `SCHEDULED` (editable), `ACTIVE`, `ARCHIVED` (final) |
| `alert_status_transition` | `from_code`, `to_code` (PK compuesta; FK a `alert_status`) | `DRAFT`→`SCHEDULED`/`ACTIVE`/`ARCHIVED`; `SCHEDULED`→`ACTIVE`/`ARCHIVED`; `ACTIVE`→`ARCHIVED` |
| `alert_type` | `code` (PK, VARCHAR 20), `label` (80), `display_order`, `active` (BOOLEAN) | `EARTHQUAKE`, `FIRE`, `OTHER`, `OPERATIVA` |
| `audit_action` | `code` (PK, VARCHAR 40), `label` | `CREATE`, `UPDATE`, `SCHEDULE`, `ACTIVATE`, `ARCHIVE`, `CONFIRM`, `NOTIFICATION_SENT`, `NOTIFICATION_FAILED`, `RECIPIENT_RESOLUTION_FAILED` y `END` (histórica: ya no se emite) |
| `audit_status` | `code` (PK), `label` | Los estados de `alert_status` más `ENDED` (histórico) |

Una alerta pasa solo por las transiciones de `alert_status_transition`; solo `DRAFT` y `SCHEDULED` se pueden editar (`editable`).

---

### alerts
Una alerta de emergencia ([RF-ALT-01](1_requirements.md)).

| Columna | Tipo | Nulo | Restricciones | Descripción |
|---|---|---|---|---|
| `id` | UUID | No | PK | Identificador de la alerta. |
| `title` | VARCHAR(150) | No | — | Título. |
| `status` | VARCHAR(20) | No | FK → `alert_status(code)` | Estado actual. |
| `instructions` | VARCHAR(1000) | No | — | Qué debe hacer quien recibe la alerta. |
| `origin` | VARCHAR(20) | No | `MANUAL` o `SIMULATION` | Quién la originó. |
| `simulation_id` | UUID | Sí | — | Simulacro de MS06. Obligatorio si `origin = SIMULATION`; NULL en caso contrario. |
| `purpose` | VARCHAR(32) | Sí | — | Etiqueta para distinguir varias alertas de un mismo simulacro (por ejemplo, a quién van dirigidas). |
| `created_by` | VARCHAR(64) | No | — | Usuario de MS01 que la creó. |
| `activation_at` | TIMESTAMPTZ | Sí | — | Cuándo se activará, si está programada. |
| `activated_at` | TIMESTAMPTZ | Sí | — | Cuándo se activó. |
| `archived_at` | TIMESTAMPTZ | Sí | — | Cuándo se archivó. |
| `version` | BIGINT | No | — | Control de concurrencia optimista. |
| `created_at` | TIMESTAMPTZ | No | — | Cuándo se creó. |
| `updated_at` | TIMESTAMPTZ | No | — | Última modificación. |

**Índices:**

| Índice | Campos | Para qué |
|---|---|---|
| `uq_alerts_simulation_purpose_live` (único, parcial) | `simulation_id`, `COALESCE(purpose, '')` donde `status` es `SCHEDULED` o `ACTIVE` | Impide dos alertas vivas con el mismo simulacro y propósito. |
| `idx_alerts_status_activation_at` | `status`, `activation_at` | Buscar las programadas que ya deben activarse. |
| `idx_alerts_status_activated_at` | `status`, `activated_at` | Bandeja y consultas por alertas activas. |
| `idx_alerts_created_at` | `created_at` | Listado ordenado por fecha. |
| `idx_alerts_simulation_id` | `simulation_id` | Alertas de un simulacro. |
| `idx_alerts_simulation_purpose` | `simulation_id`, `purpose` | Alertas de un simulacro por propósito. |

---

### alert_types
Tipos de una alerta (una alerta puede tener varios).

| Columna | Tipo | Nulo | Restricciones | Descripción |
|---|---|---|---|---|
| `alert_id` | UUID | No | PK compuesta; FK → `alerts(id)` | Alerta. |
| `type` | VARCHAR(20) | No | PK compuesta; FK → `alert_type(code)` | Tipo (`EARTHQUAKE`, `FIRE`, `OTHER` u `OPERATIVA`). |

**Índices:** `idx_alert_types_type` (`type`).

---

### alert_scope_items
Dónde aplica la alerta: todo el campus, edificios o zonas.

| Columna | Tipo | Nulo | Restricciones | Descripción |
|---|---|---|---|---|
| `id` | UUID | No | PK | Identificador del elemento. |
| `alert_id` | UUID | No | FK → `alerts(id)` | Alerta. |
| `scope_type` | VARCHAR(20) | No | `CAMPUS`, `BUILDING` o `ZONE` | Tipo de alcance. |
| `scope_ref_id` | VARCHAR(64) | Sí | — | Identificador del edificio o zona en MS02. NULL cuando es `CAMPUS`. |

**Restricciones:** único (`alert_id`, `scope_type`, `scope_ref_id`) con `NULLS NOT DISTINCT`, de modo que `CAMPUS` tampoco se repite.

**Índices:** `idx_scope_items_alert_id` (`alert_id`).

---

### alert_recipients
A quién va dirigida la alerta: usuarios concretos o todos los de un rol.

| Columna | Tipo | Nulo | Restricciones | Descripción |
|---|---|---|---|---|
| `id` | UUID | No | PK | Identificador del destinatario. |
| `alert_id` | UUID | No | FK → `alerts(id)` | Alerta. |
| `recipient_type` | VARCHAR(20) | No | `USER` o `ROLE` | Tipo de destinatario. |
| `recipient_ref_id` | VARCHAR(64) | No | — | Identificador del usuario o nombre del rol en MS01. |

**Restricciones:** único (`alert_id`, `recipient_type`, `recipient_ref_id`).

**Índices:** `idx_alert_recipients_inbox` (para la bandeja del destinatario).

---

### notification_deliveries
Resultado del envío de correo a cada usuario resuelto ([RF-ALT-05](1_requirements.md)).

| Columna | Tipo | Nulo | Restricciones | Descripción |
|---|---|---|---|---|
| `id` | UUID | No | PK | Identificador del envío. |
| `alert_id` | UUID | No | FK → `alerts(id)` | Alerta. |
| `user_id` | VARCHAR(64) | No | — | Usuario de MS01 destinatario. |
| `email` | VARCHAR(254) | No | — | Correo al que se envió (ver [Almacenamiento seguro](#almacenamiento-seguro)). |
| `status` | VARCHAR(20) | No | `SENT` o `FAILED` | Resultado. |
| `error_reason` | VARCHAR(500) | Sí | — | Motivo del fallo. |
| `sent_at` | TIMESTAMPTZ | Sí | — | Cuándo se envió. |

**Restricciones:** único (`alert_id`, `user_id`): una persona recibe una sola notificación por alerta.

**Índices:** `idx_deliveries_user_alert` (`user_id`, `alert_id`) para la bandeja del destinatario.

---

### alert_confirmations
Confirmaciones de recepción de una alerta activa por parte de sus destinatarios.

| Columna | Tipo | Nulo | Restricciones | Descripción |
|---|---|---|---|---|
| `id` | UUID | No | PK | Identificador de la confirmación. |
| `alert_id` | UUID | No | FK → `alerts(id)` | Alerta. |
| `user_id` | VARCHAR(64) | No | — | Usuario de MS01 que confirmó. |
| `confirmed_at` | TIMESTAMPTZ | No | — | Cuándo confirmó. |

**Restricciones:** único (`alert_id`, `user_id`): confirmar dos veces no crea otra fila.

**Índices:** `idx_confirmations_user_id` (`user_id`), `idx_confirmations_user_alert` (`user_id`, `alert_id`).

---

### audit_events
Historial de lo que le pasa a cada alerta. Solo se inserta.

| Columna | Tipo | Nulo | Restricciones | Descripción |
|---|---|---|---|---|
| `id` | UUID | No | PK | Identificador del hecho. |
| `alert_id` | UUID | No | FK → `alerts(id)` | Alerta. |
| `action` | VARCHAR(40) | No | FK → `audit_action(code)` | Acción. |
| `actor` | VARCHAR(64) | No | — | Usuario de MS01, o `SYSTEM_SCHEDULER` (activación programada) / `SYSTEM_NOTIFIER` (envío de notificaciones) en las acciones automáticas. |
| `from_status` | VARCHAR(20) | Sí | FK → `audit_status(code)` | Estado anterior, si hubo cambio. |
| `to_status` | VARCHAR(20) | Sí | FK → `audit_status(code)` | Estado nuevo, si hubo cambio. |
| `details` | VARCHAR(500) | Sí | — | Texto breve sobre la acción. Sin correos ni contenido de la alerta. |
| `created_at` | TIMESTAMPTZ | No | — | Cuándo ocurrió. |

**Índices:** `idx_audit_alert_created` (`alert_id`, `created_at`).

---

## Relaciones

```
alert_status  1 ── N alerts
alert_status  1 ── N alert_status_transition (origen y destino)
alert_type    1 ── N alert_types
audit_action  1 ── N audit_events
audit_status  1 ── N audit_events (estado anterior y nuevo)
alerts        1 ── N alert_types
alerts        1 ── N alert_scope_items
alerts        1 ── N alert_recipients
alerts        1 ── N notification_deliveries
alerts        1 ── N alert_confirmations
alerts        1 ── N audit_events
```

`alerts.simulation_id` se relaciona con un simulacro de MS06, sin clave foránea porque vive en otra base.

---

## Correspondencia entre API y columnas

| Campo del contrato | Tabla.columna | Notas |
|---|---|---|
| `title` | `alerts.title` | |
| `types[]` | `alert_types.type` | Una fila por tipo. |
| `instructions` | `alerts.instructions` | |
| `recipients[].type` / `.id` | `alert_recipients.recipient_type` / `recipient_ref_id` | `USER` o `ROLE`. |
| `scope` / `scopes[]` (`type`, `ids[]`) | `alert_scope_items.scope_type` / `scope_ref_id` | Una fila por identificador; `CAMPUS` no lleva identificador. |
| `origin.type` | `alerts.origin` | `MANUAL` o `SIMULATION`. |
| `origin.simulationId` | `alerts.simulation_id` | |
| `purpose` | `alerts.purpose` | |
| `activationAt` | `alerts.activation_at` | |
| `activateImmediately` | — | No se guarda: activa la alerta al crearla (`activated_at`). |
| `status` | `alerts.status` | Código de `alert_status`. |
| `createdBy` | `alerts.created_by` | Lo toma el servicio de `X-User-Id`. |
| `confirmedAt` | `alert_confirmations.confirmed_at` | |
| Entrada de auditoría (`GET /{id}/auditoria`) | `audit_events` | `action`, `actor`, `fromStatus`, `toStatus`, `details`, `createdAt`. |

---

## Reglas de integridad

| Regla | Dónde se garantiza |
|---|---|
| Estado, tipo y acción existentes | BD: claves foráneas a los catálogos. |
| Solo transiciones válidas | Servicio: consulta `alert_status_transition`; responde 409 `TRANSICION_INVALIDA`. |
| Solo se edita en `DRAFT` o `SCHEDULED` | Servicio: usa `alert_status.editable`; responde 409 `ESTADO_ALERTA_INVALIDO`. |
| Una alerta viva por simulacro y propósito | BD: índice único parcial `uq_alerts_simulation_purpose_live`. Servicio: responde 409 `SIMULACION_DUPLICADA`. |
| Un tipo, un alcance y un destinatario no se repiten por alerta | BD: claves y restricciones únicas. |
| `OPERATIVA` va sola y no se usa con simulacros | Servicio ([RF-ALT-01](1_requirements.md)). |
| Un envío por usuario y alerta | BD: único (`alert_id`, `user_id`) en `notification_deliveries`. |
| Una confirmación por usuario y alerta | BD: único (`alert_id`, `user_id`). Servicio: la repetición responde 200 sin crear otra fila. |
| Solo confirma un destinatario de una alerta activa | Servicio: 409 `ALERTA_NO_ACTIVA` o 403 `NO_ES_DESTINATARIO`. |
| Edición concurrente | Servicio: columna `version`; responde 409 `MODIFICACION_CONCURRENTE`. |
| El ORGANIZADOR solo opera alertas de simulacros | Servicio, según los permisos del Gateway. |
| La auditoría solo se inserta | Servicio: la API solo expone una consulta sobre `audit_events`. |

---

## Operaciones transaccionales y consultas

| Operación | Escribe | Notas |
|---|---|---|
| Crear alerta ([RF-ALT-01](1_requirements.md)) | `alerts`, `alert_types`, `alert_scope_items`, `alert_recipients`, `audit_events` | Una sola transacción. Valida edificios y zonas en MS02. |
| Editar alerta ([RF-ALT-03](1_requirements.md)) | las mismas, más `audit_events` | Solo en `DRAFT` o `SCHEDULED`; reemplaza tipos, alcance o destinatarios si se envían. |
| Programar, activar, archivar ([RF-ALT-04](1_requirements.md)) | `alerts`, `audit_events` | Cada cambio de estado deja un hecho de auditoría en la misma transacción (la auditoría exige estar dentro de una). |
| Activación programada | `alerts`, `audit_events` | Una tarea revisa cada 10 segundos (`app.alerts.activation-check-ms`) y activa las `SCHEDULED` cuya `activation_at` ya llegó; el actor es `SYSTEM_SCHEDULER`. |
| Notificar ([RF-ALT-05](1_requirements.md)) | `notification_deliveries`, `audit_events` | Cuando termina la transacción de activación, resuelve los destinatarios con MS01 y envía por correo; cada resultado queda en `notification_deliveries`. Un fallo de correo no revierte la activación. |
| Confirmar recepción | `alert_confirmations`, `audit_events` | 201 la primera vez, 200 si ya existía. |

**Consultas:**

* **Listado** (`GET /api/v1/alertas`): filtra por estado, tipo (repetible), origen, alcance, `activeAt`, rango de fechas (`startDate`, `endDate`) y `simulationId`; paginado.
* **Bandeja del usuario** (`GET /api/v1/alertas/mias`): alertas dirigidas al usuario por identificador, por rol o por envío registrado. Por defecto solo `ACTIVE`; admite `ARCHIVED` y `ALL`; nunca devuelve `DRAFT` ni `SCHEDULED`.
* **Detalle** (`GET /api/v1/alertas/{id}`) y **auditoría** (`GET /api/v1/alertas/{id}/auditoria`).

> **Pendiente:** MS03 todavía no envía sus hechos de auditoría a MS08 (`POST /api/v1/audit/events`). Hasta que lo haga, `audit_events` es su única bitácora y MS08 no recibe las acciones de MS03 del [catálogo de auditoría](../integration/audit-catalog.md).

---

## Configuración

Variables de entorno que afectan los datos (valores por defecto de `application.yml`):

| Variable | Por defecto | Uso |
|---|---|---|
| `DB_URL` / `DB_USER` / `DB_PASSWORD` | — (obligatorias) | Conexión a PostgreSQL. La contraseña es un secreto del entorno. |
| `APP_TIMEZONE` | `America/Bogota` | Zona horaria de la aplicación. |
| `MS01_URL` | `http://localhost:8080` | MS01: usuarios por rol y por identificador. |
| `MS02_URL` | `http://localhost:8080` | MS02: validación de edificios y zonas. |
| `MS_SERVICE_KEY` | — | Llave de servicio para llamar a MS01 y MS02 (secreto). |
| `MAIL_PROVIDER` | `brevo` | Proveedor de correo. |
| `BREVO_API_KEY` | — | Clave de Brevo (secreto). |
| `MAIL_FROM` / `MAIL_FROM_NAME` | `no-reply@campusseguro.local` / `CampusSeguro` | Remitente. |
| `MAIL_DAILY_LIMIT` | `300` | Tope de correos por día. |
| `BREVO_MAX_ATTEMPTS` / `BREVO_RETRY_DELAY_MS` | `3` / `200` | Reintentos del envío. |
| `APP_BASE_URL` | `http://localhost:5173` | Dirección del frontend que aparece en el correo. |

---

## Datos iniciales

1. **Catálogos:** los carga Flyway al arrancar (`V8` tipos de alerta, `V9` estados y transiciones, `V10` acciones y estados de auditoría).
2. **Sin alertas ni destinatarios:** se crean en operación, por los coordinadores y organizadores o por MS06 cuando un simulacro lo requiere.
3. **Sin usuarios ni permisos propios:** los permisos de MS03 (`ms03:alert:manage`, `ms03:alert:manage_drill`, `ms03:alert:read_history` y `ms03:alert:receive`) los siembra MS01 en la [matriz de roles y permisos](../integration/permissions-matrix.md).
