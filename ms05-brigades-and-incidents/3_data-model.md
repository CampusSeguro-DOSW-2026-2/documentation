# Especificación del Modelo de Datos

Base de datos PostgreSQL propia de MS05. El esquema desplegado está en `database/` del repositorio de MS05. Ningún otro microservicio la lee ni la modifica; se accede a ella solo mediante la API.

## Índice de Contenido
1. [Alcance](#alcance)
2. [Convenciones](#convenciones)
3. [Catálogos](#catalogos)
4. [Tablas](#tablas)
    - [brigadista](#brigadista)
    - [configuracion_operacion_evento](#configuracion_operacion_evento)
    - [evento_zona](#evento_zona)
    - [asignacion_brigadista](#asignacion_brigadista)
    - [incidente](#incidente)
    - [evidencia_incidente](#evidencia_incidente)
    - [historial_incidente](#historial_incidente)
    - [registro_auditoria](#registro_auditoria)
5. [Relaciones](#relaciones)
6. [Correspondencia requerimientos ↔ tablas](#correspondencia)
7. [Reglas de integridad](#reglas-de-integridad)
8. [Datos iniciales](#datos-iniciales)

---

<a id="alcance"></a>
## Alcance

**Se almacena:** los brigadistas con su estado de habilitación, la configuración de operación de cada evento (auto asignación y panel de gestión) con los edificios o zonas de su alcance, las asignaciones de brigadistas a edificios, zonas seguras o puntos de encuentro, los incidentes con su ciclo de atención, su historial de transiciones y sus evidencias, y el registro de auditoría propio de MS05.

**No se almacena, y por qué:**
* **Datos personales:** del usuario solo se guarda su id en MS01 (`usuario_id_externo`, `reportante_id`, `actor_id`, `usuario_origen`), sin nombres ni correo. Los datos del usuario se consultan a MS01 cuando se necesitan.
* **Datos de otros dominios:** los edificios, zonas seguras y puntos de encuentro pertenecen a MS02 y los eventos a MS06. De cada uno solo se guarda su id como referencia. Del evento, MS05 conserva únicamente lo que recibe de MS06 para operar: nombre, tipo (simulacro o real), estado y los edificios o zonas del alcance ([evento_zona](#evento_zona)).
* **Los archivos de las evidencias:** se guardan en el almacenamiento de archivos del servicio y la base solo guarda su nombre, su tipo y la referencia al archivo ([evidencia_incidente](#evidencia_incidente)).
* **Solicitudes de asistencia como entidad aparte:** una solicitud de asistencia que llega desde el check-in se registra como un incidente (`incidente`), de modo que comparte su ciclo de vida, su historial y sus evidencias.
* **Auditoría centralizada:** la consolida MS08 ([catálogo de auditoría](../integration/audit-catalog.md#ms05--brigadistas-e-incidentes)). `registro_auditoria` es el registro propio de MS05 que atiende las consultas de [RF-BEI-09](1_requirements.md#rf-bei-09).

---

<a id="convenciones"></a>
## Convenciones

* Nombres de tablas y columnas en `snake_case` y en español.
* Identificadores propios de las tablas de negocio (brigadista, asignación, incidente, evidencia, historial, registro de auditoría): UUID generado por el servicio. Los catálogos usan un `BIGSERIAL`.
* Fechas y horas en `TIMESTAMP WITH TIME ZONE`, con la excepción de `asignacion_brigadista.fecha_creacion` y `fecha_actualizacion`, que son `TIMESTAMP` sin zona horaria.
* Identificadores externos, que no son llaves foráneas porque viven en otro microservicio:
  * `usuario_id_externo`, `brigadista_id`, `brigadista_responsable_id`, `responsable_anterior_id`, `responsable_nuevo_id`, `reportante_id`, `actor_id`, `usuario_origen`: id del usuario en MS01. El `brigadista_id` de una asignación es el mismo valor que `brigadista.usuario_id_externo`.
  * `evento_id`: id del evento en MS06.
  * `ubicacion_id`, `zona_id`: id de MS02 del tipo de ubicación que indica `tipo_ubicacion_id` (edificio, zona segura o punto de encuentro).
* Los estados, las prioridades y los tipos son tablas de catálogo con valores fijos ([Catálogos](#catalogos)). Agregar o cambiar un valor exige una migración nueva y la actualización del código en el mismo Pull Request.
* Un incidente nunca se elimina físicamente: la eliminación es lógica (`incidente.eliminado`) y las consultas excluyen los incidentes eliminados.
* El esquema se versiona con los scripts SQL de `database/`: `init/01-create-tables.sql` y `init/02-insert-catalogs.sql` crean las tablas y los catálogos iniciales, y las migraciones `migrations/V1` a `V5` los llevan al modelo actual. En una base nueva, el script de arranque `infra/db/01-initialize-ms05.sh` ejecuta `init/01`, `init/02` y las migraciones `V2` a `V5` en ese orden; `V1` actualiza a mano una base creada con un esquema anterior y no corre en una base nueva. En una base que ya existe, cada migración nueva se aplica una vez a mano. La aplicación solo valida el esquema (`ddl-auto=validate`) y nunca lo genera.

---

<a id="catalogos"></a>
## Catálogos

Cada catálogo tiene la misma forma: `id` (`BIGSERIAL`, PK), `nombre` (único) y `descripcion`.

| Tabla | Longitud de `nombre` | Valores |
|---|---|---|
| `estado_brigadista` | 30 | `INACTIVO`, `ACTIVO` |
| `estado_asignacion` | 30 | `VIGENTE`, `RETIRADA` |
| `tipo_ubicacion` | 40 | `EDIFICIO`, `ZONA_SEGURA`, `PUNTO_ENCUENTRO` |
| `estado_incidente` | 50 | `REPORTADO`, `ASIGNADO`, `EN_ATENCION`, `RESUELTO`, `CERRADO` |
| `prioridad_incidente` | 50 | `EMERGENCIA`, `URGENCIA_MODERADA`, `POCA_URGENCIA` |
| `tipo_registro_auditoria` | 30 | `ASIGNACION`, `SOLICITUD`, `INCIDENTE`, `EVIDENCIA` |

`descripcion` es `VARCHAR(150)` en `estado_brigadista`, `estado_asignacion`, `tipo_ubicacion` y `tipo_registro_auditoria`, y `VARCHAR(255)` en `estado_incidente` y `prioridad_incidente`. Todas admiten nulo.

---

<a id="tablas"></a>
## Tablas

<a id="brigadista"></a>
### brigadista
Una fila por usuario con rol de brigadista que MS05 ya conoce. MS05 la crea con estado `INACTIVO` la primera vez que lo ve (al buscarlo en MS01 o al listar los brigadistas de un evento); [RF-BEI-03](1_requirements.md#rf-bei-03) cambia el estado a `ACTIVO` o `INACTIVO`. Un brigadista solo puede asignarse a un edificio mientras su estado es `ACTIVO`.

| Columna | Tipo | Nulo | Restricciones | Descripción |
|---|---|---|---|---|
| `id` | UUID | No | PK | Id del brigadista en MS05. |
| `usuario_id_externo` | VARCHAR(255) | No | UNIQUE | Id del usuario en MS01. Un usuario tiene como máximo un registro de brigadista. |
| `estado_id` | BIGINT | No | FK → `estado_brigadista.id` (`fk_brigadista_estado`) | `INACTIVO` o `ACTIVO`. |

**Índices:** PK `id`; único `usuario_id_externo`; `idx_brigadista_estado (estado_id)`.

<a id="configuracion_operacion_evento"></a>
### configuracion_operacion_evento
Una fila por evento que MS06 ha comunicado a MS05. Define si el evento admite auto asignación de brigadistas y si su panel de gestión está habilitado, y guarda los datos del evento que MS05 necesita para operar.

| Columna | Tipo | Nulo | Por defecto | Restricciones | Descripción |
|---|---|---|---|---|---|
| `evento_id` | VARCHAR(100) | No | — | PK | Id del evento en MS06. |
| `autoasignacion_habilitada` | BOOLEAN | No | — | — | `true` mientras el evento está `PROGRAMADO`: los brigadistas pueden asignarse por sí mismos a un edificio del evento. |
| `panel_habilitado` | BOOLEAN | No | — | — | `true` mientras el evento está `EN_EJECUCION`: el coordinador gestiona a los brigadistas desde el panel y se pueden registrar incidentes. Nunca es `true` a la vez que `autoasignacion_habilitada`; con el evento en `BORRADOR` o `FINALIZADO` ambas son `false`. |
| `actualizado_en` | TIMESTAMPTZ | No | — | — | Último cambio de la configuración. |
| `nombre` | VARCHAR(200) | Sí | NULL | — | Nombre del evento, tal como lo envía MS06. |
| `simulacro` | BOOLEAN | No | `FALSE` | — | `true` si el evento es un simulacro y `false` si es real. Determina cuál de los dos permisos de la [matriz](../integration/permissions-matrix.md) (simulacro o evento real) exige cada operación del evento. |
| `estado_evento` | VARCHAR(30) | Sí | NULL | — | Último estado del evento que envió MS06 (`BORRADOR`, `PROGRAMADO`, `EN_EJECUCION` o `FINALIZADO`). |

**Índices:** PK `evento_id`.

<a id="evento_zona"></a>
### evento_zona
Edificios o zonas del alcance de un evento, tal como los envía MS06 al cambiar el estado del evento. Los usan la auto asignación y el panel de gestión de brigadistas. El alcance guardado se reemplaza solo cuando MS06 envía al menos una ubicación.

| Columna | Tipo | Nulo | Por defecto | Restricciones | Descripción |
|---|---|---|---|---|---|
| `evento_id` | VARCHAR(100) | No | — | PK compuesta; FK → `configuracion_operacion_evento.evento_id` (`fk_evento_zona_configuracion`), `ON DELETE CASCADE` | Evento al que pertenece el edificio. |
| `zona_id` | VARCHAR(100) | No | — | PK compuesta | Id del edificio o la zona en MS02. |
| `nombre` | VARCHAR(200) | Sí | NULL | — | Nombre del edificio o la zona, tal como lo envía MS06. |
| `tipo_ubicacion` | VARCHAR(40) | No | `'EDIFICIO'` | — | Tipo de ubicación en texto: `EDIFICIO`, `ZONA_SEGURA` o `PUNTO_ENCUENTRO`. MS05 convierte a estos valores los tipos que usa MS06. Los edificios que MS06 envía solo con su id se guardan como `EDIFICIO`. |

**Índices:** PK (`evento_id`, `zona_id`).

<a id="asignacion_brigadista"></a>
### asignacion_brigadista
Una fila por cada asignación de un brigadista a una ubicación en un evento ([RF-BEI-01](1_requirements.md#rf-bei-01)). Retirar una asignación no la elimina: cambia su estado a `RETIRADA` y la fila queda como histórico.

| Columna | Tipo | Nulo | Por defecto | Restricciones | Descripción |
|---|---|---|---|---|---|
| `id_asignacion` | UUID | No | — | PK | Id de la asignación. |
| `brigadista_id` | VARCHAR(100) | No | — | — | Id del usuario brigadista en MS01. |
| `evento_id` | VARCHAR(100) | No | — | — | Evento de la asignación. |
| `ubicacion_id` | VARCHAR(100) | No | — | — | Id de MS02 de la ubicación asignada. |
| `tipo_ubicacion_id` | BIGINT | No | — | FK → `tipo_ubicacion.id` (`fk_asignacion_tipo_ubicacion`) | Tipo de la ubicación: edificio, zona segura o punto de encuentro. |
| `estado_id` | BIGINT | No | — | FK → `estado_asignacion.id` (`fk_asignacion_estado`) | `VIGENTE` o `RETIRADA`. |
| `observaciones` | TEXT | Sí | NULL | — | Observaciones de la asignación. |
| `fecha_creacion` | TIMESTAMP | No | — | — | Fecha y hora de creación. |
| `fecha_actualizacion` | TIMESTAMP | Sí | NULL | — | Fecha y hora del último cambio. |
| `version` | BIGINT | No | `0` | — | Versión para el control de concurrencia optimista: dos cambios simultáneos sobre la misma asignación no se pisan. |

**Índices:** PK `id_asignacion`; `idx_asignacion_brigadista (brigadista_id)`; `idx_asignacion_evento (evento_id)`; `idx_asignacion_ubicacion (ubicacion_id)`; `idx_asignacion_estado (estado_id)`; `idx_asignacion_evento_estado (evento_id, estado_id)` para listar las asignaciones vigentes de un evento.

<a id="incidente"></a>
### incidente
Una fila por incidente, ya sea reportado directamente ([RF-BEI-04](1_requirements.md#rf-bei-04)) o generado por una solicitud de asistencia del check-in. Recorre el flujo `REPORTADO` → `ASIGNADO` → `EN_ATENCION` → `RESUELTO` → `CERRADO` ([RF-BEI-07](1_requirements.md#rf-bei-07), [RF-BEI-08](1_requirements.md#rf-bei-08)); cada transición queda en [historial_incidente](#historial_incidente).

| Columna | Tipo | Nulo | Por defecto | Restricciones | Descripción |
|---|---|---|---|---|---|
| `id` | UUID | No | — | PK | Id del incidente. |
| `evento_id` | VARCHAR(100) | No | — | — | Evento en el que se reportó. |
| `nombre_afectado` | VARCHAR(150) | No | — | — | Nombre de la persona afectada. |
| `reportante_id` | VARCHAR(100) | No | — | — | Id del usuario que reportó el incidente en MS01. |
| `ubicacion_id` | VARCHAR(100) | No | — | — | Id de MS02 de la ubicación del incidente. |
| `tipo_ubicacion_id` | BIGINT | No | — | FK → `tipo_ubicacion.id` (`fk_incidente_tipo_ubicacion`) | Tipo de la ubicación. |
| `prioridad_id` | BIGINT | No | — | FK → `prioridad_incidente.id` (`fk_incidente_prioridad`) | `EMERGENCIA`, `URGENCIA_MODERADA` o `POCA_URGENCIA`. |
| `estado_id` | BIGINT | No | — | FK → `estado_incidente.id` (`fk_incidente_estado`) | Estado actual del incidente. |
| `descripcion` | VARCHAR(1000) | Sí | NULL | — | Descripción del incidente. |
| `fecha_reporte` | TIMESTAMPTZ | No | — | — | Fecha y hora en que se reportó. |
| `brigadista_responsable_id` | VARCHAR(100) | Sí | NULL | — | Id en MS01 del brigadista que atiende el incidente. Se fija al iniciar la atención o al reasignarlo. |
| `fecha_primera_atencion` | TIMESTAMPTZ | Sí | NULL | — | Momento en que se inició la atención. De ella se calcula el tiempo hasta la primera atención. |
| `fecha_resolucion` | TIMESTAMPTZ | Sí | NULL | — | Momento en que el brigadista resolvió el incidente. |
| `fecha_cierre` | TIMESTAMPTZ | Sí | NULL | — | Momento en que el coordinador cerró el incidente. |
| `observaciones_resolucion` | VARCHAR(1000) | Sí | NULL | — | Observaciones con las que se resolvió el incidente. |
| `eliminado` | BOOLEAN | No | `FALSE` | — | Marca de eliminación lógica. Solo se puede marcar un incidente en estado `RESUELTO`. |

**Índices:** PK `id`; `idx_incidente_evento (evento_id)`; `idx_incidente_reportante (reportante_id)`; `idx_incidente_ubicacion (ubicacion_id)`; `idx_incidente_estado (estado_id)`; `idx_incidente_prioridad (prioridad_id)`; `idx_incidente_responsable (brigadista_responsable_id)`; `idx_incidente_eliminado (eliminado)`.

<a id="evidencia_incidente"></a>
### evidencia_incidente
Una fila por cada archivo de evidencia que se adjunta a un incidente. Solo admite INSERT.

| Columna | Tipo | Nulo | Restricciones | Descripción |
|---|---|---|---|---|
| `id` | UUID | No | PK | Id de la evidencia. |
| `nombre_archivo` | VARCHAR(255) | No | — | Nombre original del archivo. |
| `tipo_contenido` | VARCHAR(100) | No | — | Tipo de contenido (MIME) del archivo. |
| `referencia_archivo` | VARCHAR(500) | No | — | Ubicación del archivo en el almacenamiento del servicio. |
| `incidente_id` | UUID | No | FK → `incidente.id` (`fk_evidencia_incidente`) | Incidente al que pertenece. |
| `fecha_carga` | TIMESTAMPTZ | No | — | Fecha y hora de la carga. |

**Índices:** PK `id`; `idx_evidencia_incidente (incidente_id)`.

<a id="historial_incidente"></a>
### historial_incidente
Una fila por cada transición de estado de un incidente, incluida su creación. Solo admite INSERT; es el registro de los tiempos en que ocurre cada transición.

| Columna | Tipo | Nulo | Restricciones | Descripción |
|---|---|---|---|---|
| `id` | UUID | No | PK | Id del registro. |
| `incidente_id` | UUID | No | FK → `incidente.id` (`fk_historial_incidente`) | Incidente al que pertenece. |
| `estado_anterior_id` | BIGINT | Sí | FK → `estado_incidente.id` (`fk_historial_estado_anterior`) | Estado antes del cambio. Nulo en el registro de creación. |
| `estado_nuevo_id` | BIGINT | No | FK → `estado_incidente.id` (`fk_historial_estado_nuevo`) | Estado después del cambio. |
| `responsable_anterior_id` | VARCHAR(100) | Sí | — | Brigadista responsable antes del cambio. |
| `responsable_nuevo_id` | VARCHAR(100) | Sí | — | Brigadista responsable después del cambio. |
| `actor_id` | VARCHAR(100) | No | — | Usuario o servicio que realizó el cambio. |
| `observaciones` | VARCHAR(1000) | Sí | — | Observaciones del cambio. |
| `fecha_cambio` | TIMESTAMPTZ | No | — | Fecha y hora del cambio. |

**Índices:** PK `id`; `idx_historial_incidente (incidente_id)`; `idx_historial_fecha (fecha_cambio)`.

<a id="registro_auditoria"></a>
### registro_auditoria
Una fila por cada acción auditable realizada en MS05 ([RF-BEI-09](1_requirements.md#rf-bei-09)). Solo admite INSERT.

| Columna | Tipo | Nulo | Restricciones | Descripción |
|---|---|---|---|---|
| `id` | UUID | No | PK | Id del registro (`idRegistro`). |
| `tipo_registro_id` | BIGINT | No | FK → `tipo_registro_auditoria.id` (`fk_auditoria_tipo_registro`) | `ASIGNACION`, `SOLICITUD`, `INCIDENTE` o `EVIDENCIA`. |
| `usuario_origen` | VARCHAR(100) | No | — | Usuario o servicio que realizó la acción. |
| `accion` | VARCHAR(100) | No | — | Acción realizada. |
| `recurso_tipo` | VARCHAR(50) | No | — | Tipo del recurso afectado. |
| `recurso_id` | VARCHAR(100) | No | — | Id del recurso afectado. |
| `fecha_hora_registro` | TIMESTAMPTZ | No | — | Fecha y hora en que se realizó la acción. |
| `evento_id` | VARCHAR(100) | Sí | — | Evento durante el que se realizó. Nulo si no había un evento. |
| `origen` | VARCHAR(100) | No | — | Módulo que solicita guardar el registro. |
| `resultado` | VARCHAR(50) | No | — | Resultado de la operación. |
| `detalle` | VARCHAR(1000) | Sí | — | Detalle adicional de la acción. |

**Índices:** PK `id`; `idx_auditoria_evento (evento_id)`; `idx_auditoria_recurso (recurso_tipo, recurso_id)`; `idx_auditoria_fecha (fecha_hora_registro)`; `idx_auditoria_tipo (tipo_registro_id)`.

---

<a id="relaciones"></a>
## Relaciones

| Origen | Destino | Cardinalidad | Llave |
|---|---|---|---|
| `brigadista` | `estado_brigadista` | N:1 | `estado_id` |
| `asignacion_brigadista` | `tipo_ubicacion` | N:1 | `tipo_ubicacion_id` |
| `asignacion_brigadista` | `estado_asignacion` | N:1 | `estado_id` |
| `incidente` | `tipo_ubicacion` | N:1 | `tipo_ubicacion_id` |
| `incidente` | `prioridad_incidente` | N:1 | `prioridad_id` |
| `incidente` | `estado_incidente` | N:1 | `estado_id` |
| `evidencia_incidente` | `incidente` | N:1 | `incidente_id` |
| `historial_incidente` | `incidente` | N:1 | `incidente_id` |
| `historial_incidente` | `estado_incidente` | N:1 (dos llaves) | `estado_anterior_id`, `estado_nuevo_id` |
| `evento_zona` | `configuracion_operacion_evento` | N:1 | `evento_id` (`ON DELETE CASCADE`) |
| `registro_auditoria` | `tipo_registro_auditoria` | N:1 | `tipo_registro_id` |

`asignacion_brigadista.brigadista_id` y `incidente.brigadista_responsable_id` se relacionan con `brigadista.usuario_id_externo` por valor, sin llave foránea, porque en el mismo valor se guarda el id del usuario en MS01. Del mismo modo, `asignacion_brigadista.evento_id`, `incidente.evento_id` y `registro_auditoria.evento_id` no tienen llave foránea hacia `configuracion_operacion_evento`, porque el evento pertenece a MS06.

---

<a id="correspondencia"></a>
## Correspondencia requerimientos ↔ tablas

| Requerimiento | Tablas |
|---|---|
| [RF-BEI-01](1_requirements.md#rf-bei-01) Asignar espacios a brigadistas | `asignacion_brigadista`, `configuracion_operacion_evento`, `evento_zona` |
| [RF-BEI-02](1_requirements.md#rf-bei-02) Buscar brigadista | `brigadista` |
| [RF-BEI-03](1_requirements.md#rf-bei-03) Activar brigadista | `brigadista` |
| [RF-BEI-04](1_requirements.md#rf-bei-04) Registrar incidente | `incidente`, `historial_incidente`, `evidencia_incidente` |
| [RF-BEI-05](1_requirements.md#rf-bei-05) Editar incidente | `incidente`, `historial_incidente`, `evidencia_incidente` |
| [RF-BEI-06](1_requirements.md#rf-bei-06) Consultar responsabilidades | `asignacion_brigadista`, `incidente` |
| [RF-BEI-07](1_requirements.md#rf-bei-07) Gestionar atención de incidentes | `incidente`, `historial_incidente` |
| [RF-BEI-08](1_requirements.md#rf-bei-08) Seguimiento y cierre de incidentes | `incidente`, `historial_incidente`, `evidencia_incidente` |
| [RF-BEI-09](1_requirements.md#rf-bei-09) Auditoría | `registro_auditoria` |

---

<a id="reglas-de-integridad"></a>
## Reglas de integridad

La base de datos garantiza las llaves primarias, las llaves foráneas, la unicidad de `brigadista.usuario_id_externo` y de los nombres de los catálogos, y los campos obligatorios. El resto de las [reglas de negocio](1_requirements.md#reglas-de-negocio) las valida el servicio antes de escribir:

* El flujo de estados del incidente no admite saltos ni retrocesos, y un incidente `CERRADO` no se modifica (reglas 1 y 2).
* No se repite una asignación `VIGENTE` del mismo brigadista, ubicación y evento; en la auto asignación y el panel, un brigadista tiene un solo edificio por evento (regla 14).
* Un incidente admite como máximo 5 evidencias (regla 10).
* Un incidente solo se registra mientras el evento está en ejecución, es decir, con `panel_habilitado = true` (regla 7).
* Solo se elimina lógicamente un incidente `RESUELTO`.
* La concurrencia sobre una asignación se controla con `asignacion_brigadista.version`.

---

<a id="datos-iniciales"></a>
## Datos iniciales

Los valores de los [catálogos](#catalogos) los inserta `init/02-insert-catalogs.sql` (`estado_asignacion`, `tipo_ubicacion`, `estado_brigadista`, `estado_incidente` con `REPORTADO` y `ASIGNADO`, y `prioridad_incidente`). Los estados `EN_ATENCION`, `RESUELTO` y `CERRADO` los agrega la migración `V2`, y `tipo_registro_auditoria` se crea con sus cuatro valores en la migración `V4`. Los inserts de `init/02` y de `V2` usan `ON CONFLICT (nombre) DO NOTHING`. El esquema no incluye datos de ejemplo.
