# Especificación del Modelo de Datos

MS08 tiene su propia base de datos MongoDB. Ningún otro microservicio la lee ni la modifica: solo se accede a ella mediante la API. Código del acceso: `entity/AuditEvent.java` y `config/AuditIndexInitializer.java` del repositorio de MS08.

## Índice de Contenido
1. [Alcance](#alcance)
2. [Convenciones](#convenciones)
3. [Almacenamiento seguro](#almacenamiento-seguro)
4. [Colecciones](#colecciones)
    - [audit_events](#audit_events)
5. [Relaciones](#relaciones)
6. [Correspondencia entre API y campos](#correspondencia-entre-api-y-campos)
7. [Reglas de integridad](#reglas-de-integridad)
8. [Operaciones y consultas](#operaciones-y-consultas)
9. [Configuración](#configuración)
10. [Datos iniciales](#datos-iniciales)

---

## Alcance

**Se almacena:** los hechos de auditoría que los demás microservicios envían ([RF-AYA-01](1_requirements.md)): quién hizo qué, sobre qué recurso, cuándo y con qué resultado, más los identificadores y códigos con los que se calculan los indicadores.

**No se almacena:**
* **Indicadores:** no se guardan. Se calculan en cada consulta con los hechos disponibles, así que incluyen los que lleguen con retraso.
* **Usuarios, roles y permisos:** pertenecen a MS01. MS08 solo guarda el identificador del usuario (`usuario_id`) y decide el acceso con los permisos que el Gateway envía en cada petición.
* **Eventos, alertas, check-ins, incidentes y ubicaciones:** pertenecen a MS06, MS03, MS04, MS05 y MS02. MS08 solo guarda su identificador dentro de un hecho.
* **Datos personales y contenido:** `detalle` y `atributos` solo llevan códigos, identificadores y conteos: nunca nombres, correos, mensajes ni cuerpos de solicitudes.
* **Sesiones y tokens:** los valida el Gateway. MS08 no ve el JWT.

**Fuente de verdad de la auditoría:** la colección `audit_events`. Cada servicio conserva su propia copia operativa, pero la consulta y los indicadores salen de MS08.

---

## Convenciones

* Una sola colección: `audit_events`, en la base `campusseguro_audit`. MongoDB crea la colección con el primer registro: no hay migraciones ni scripts de esquema.
* Los nombres de los campos del documento están en `snake_case` y en español, los mismos que el contrato OpenAPI y el cuerpo de `POST /api/v1/audit/events` (`usuario_id`, `recurso_afectado`, `fecha_hora`…). La excepción es el identificador del documento, que MongoDB guarda como `_id` y la API devuelve como `id`.
* Fechas y horas en UTC (`Date` de MongoDB). La zona horaria de la aplicación (`America/Bogota`) solo se usa para la hora de la limpieza programada ([Configuración](#configuración)).
* Los identificadores de usuario y de evento se guardan en minúsculas, aunque lleguen en mayúsculas, para que la búsqueda por igualdad funcione sin importar cómo se escriban.
* Los registros son de **solo inserción**: la API no ofrece operaciones para modificarlos ni borrarlos. El único borrado es el de retención ([RF-AYA-07](1_requirements.md)).
* El esquema lo declara el código (un `record` inmutable), no la base. Los índices los crea `AuditIndexInitializer` al arrancar; crear un índice que ya existe no tiene efecto.
* Cambiar los días de retención después de creado el índice TTL exige borrar antes el índice anterior (`ttl_fecha_hora`): MongoDB no admite cambiar el tiempo de expiración de un índice con otro valor sin recrearlo.

---

## Almacenamiento seguro

* La **cadena de conexión** (`MONGODB_URI`) incluye la contraseña de la base y es un secreto: llega solo por variable de entorno (en producción, un secreto del entorno de despliegue) y nunca está en el repositorio ni en los registros del servicio.
* La base corre en una máquina privada (`ms08-db`), accesible solo desde la máquina de la aplicación; no tiene IP pública.
* El único dato personal es `usuario_id`, un UUID de MS01. MS08 no guarda correos, nombres ni contraseñas.
* Los registros del servicio **no incluyen** cuerpos de solicitudes, `detalle`, `atributos`, identificadores de usuario, cabeceras ni cadenas de conexión.
* Un hecho recibido no se puede alterar: aunque se reenvíe con el mismo `id`, el segundo envío se ignora y el original queda intacto ([`audit_events`](#audit_events)).

---

## Colecciones

### audit_events
Un documento por hecho auditado.

| Campo | Tipo | Nulo | Restricciones | Descripción |
|---|---|---|---|---|
| `_id` | string | No | Único (índice primario) | Identificador del hecho (`id` en la API). Lo envía el servicio emisor (UUID en minúsculas). Si no lo envía, MS08 lo calcula como el SHA-256, en hexadecimal, de `correlation_id`, `modulo`, `accion`, `recurso_afectado` y `fecha_hora` unidos con `\|`, de modo que el mismo hecho enviado dos veces reciba el mismo `_id`. |
| `usuario_id` | string | Sí | UUID en minúsculas | Usuario que hizo la acción. NULL en las acciones del sistema (por ejemplo, un inicio automático). |
| `modulo` | string | No | `MS01` a `MS08` | Servicio que envió el hecho. |
| `accion` | string | No | MAYÚSCULAS y guion bajo, hasta 100 caracteres | Código de la acción (por ejemplo `EVENT_STARTED`). El catálogo de cada servicio está en el [catálogo de auditoría](../integration/audit-catalog.md). |
| `recurso_afectado` | string | Sí | Formato `<tipo>:<id>`, `<id>` de 1 a 100 caracteres sin espacios | Recurso sobre el que se actuó (por ejemplo `evento:3f6c2a9e-...`). NULL si no hay recurso. |
| `resultado` | string | No | `EXITO` o `FALLO` | Resultado de la operación. Los indicadores solo cuentan `EXITO`, salvo en las notificaciones. |
| `fecha_hora` | date | No | No más de 5 minutos en el futuro ni anterior a la retención | Cuándo ocurrió la acción, según el emisor. MS08 calcula los tiempos con ella y el índice TTL se basa en ella. |
| `recibido_en` | date | No | — | Cuándo lo recibió MS08. Lo fija el servicio. |
| `correlation_id` | string | No | Hasta 100 caracteres | Identificador de correlación de la petición original, para seguirla entre servicios. |
| `detalle` | string | Sí | Hasta 100 caracteres | Código genérico sobre el resultado (por ejemplo `RESPONDIDA`). Nunca datos personales. |
| `evento_id` | string | Sí | UUID en minúsculas | Simulacro o evento real al que pertenece el hecho. NULL en los hechos administrativos (de MS01, MS02 o MS07). |
| `tipo_evento` | string | Sí | `SIMULACRO` o `REAL` | Tipo del evento. Obligatorio con `evento_id`; `SIMULACRO` exige `evento_id`. Decide qué permiso de lectura se exige. |
| `atributos` | objeto | Sí | Plano, hasta 20 claves en `snake_case`; valores de texto (máx. 100), número o booleano | Identificadores, códigos y conteos con los que se calculan los indicadores (por ejemplo `estado`, `punto_encuentro_id`, `prioridad`). |
| `servicio_origen` | string | Sí | `ms01` a `ms08` | Nombre del servicio que envió la petición, tomado de la cabecera `X-Service-Name` que inyecta el Gateway. No se devuelve en la API. |

**Índices:**

| Índice | Campos | Para qué |
|---|---|---|
| `_id` (primario, único) | `_id` | Idempotencia por `id` y consulta del detalle de un registro. |
| `ttl_fecha_hora` | `fecha_hora` ascendente, con expiración de 180 días | MongoDB borra solo los registros que cumplen la retención. Solo existe si `app.audit.ttl-index-enabled` es `true` ([Configuración](#configuración)). |
| `tipo_fecha` | `tipo_evento`, `fecha_hora` descendente | Listados filtrados por tipo de evento, del más reciente al más antiguo. |
| `evento_fecha` | `evento_id`, `fecha_hora` descendente | Hechos de un evento: es la consulta base de los indicadores, la comparación y las ubicaciones. |
| `usuario_fecha` | `usuario_id`, `fecha_hora` descendente | Historial de un usuario. |
| `modulo_accion_fecha` | `modulo`, `accion`, `fecha_hora` descendente | Filtro por servicio y acción. |
| `correlation` | `correlation_id` | Seguir una petición entre servicios. |

Los hechos de un mismo `evento_id` que llegan fuera de orden no se reordenan al guardar: el orden lo da `fecha_hora`.

---

## Relaciones

`audit_events` es la única colección y no tiene referencias formales. Sus identificadores apuntan a datos de otros servicios, sin clave foránea porque viven en otra base:

| Campo | Apunta a | Servicio |
|---|---|---|
| `usuario_id` | Usuario | MS01 |
| `evento_id` | Evento o simulacro | MS06 |
| `recurso_afectado` (`<tipo>:<id>`) | Alerta, check-in, incidente, asignación, edificio, zona, etc. | El servicio que lo envió |
| `atributos.ubicacion_id`, `atributos.punto_encuentro_id` | Edificio, espacio o zona segura | MS02 |

Si el recurso o el evento se borra en su servicio, el hecho se conserva hasta que cumpla la retención.

---

## Correspondencia entre API y campos

Los nombres del contrato y los del documento son los mismos, con estas excepciones:

| Campo del contrato | Campo del documento | Notas |
|---|---|---|
| `id` | `_id` | Si falta en la entrada, se calcula (ver [`audit_events`](#audit_events)). |
| `fecha_hora` (entrada, con zona) | `fecha_hora` (UTC) | Se convierte a UTC al guardar. |
| `recibido_en` | `recibido_en` | Solo en la salida. Lo fija el servicio. |
| — | `servicio_origen` | Solo se guarda: no sale en la API. |
| `evento_id` (parámetro, repetible) | `evento_id` | En las consultas de comparación y ubicaciones se repite el parámetro, una vez por evento. |
| `tipo_evento` (parámetro) | `tipo_evento` | Filtra el listado de auditoría. |
| `desde`, `hasta` (parámetros) | `fecha_hora` | Rango incluyente, en formato ISO 8601. |
| `page`, `size` | — | Paginación del listado; no se guardan. |

Los indicadores (`GET /api/v1/analytics/events/{eventoId}`), la comparación (`/comparisons`) y las ubicaciones (`/locations`) no tienen campos propios: se calculan a partir de `accion`, `resultado`, `fecha_hora`, `usuario_id` y `atributos`.

---

## Reglas de integridad

| Regla | Dónde se garantiza |
|---|---|
| Registros de solo inserción | Servicio: la API solo expone `POST` y `GET`; la entidad es un `record` inmutable. |
| Retención de 180 días, `fecha_hora` dentro del rango | BD: índice TTL `ttl_fecha_hora` (o la tarea diaria de respaldo). Servicio: rechaza una `fecha_hora` a más de 5 minutos en el futuro o más antigua que la retención. |
| Idempotencia por `id` | BD: `_id` único. Servicio: captura la clave duplicada y responde 202 sin guardar de nuevo. |
| `correlation_id` obligatorio, hasta 100 caracteres | Servicio (validación del cuerpo). |
| `tipo_evento` con `evento_id` | Servicio. |
| Contrato estricto: campos desconocidos y `atributos` planos | Servicio: Jackson rechaza los campos desconocidos (`fail-on-unknown-properties`); el servicio valida hasta 20 claves en `snake_case` y valores de texto (máx. 100), número o booleano. |
| Sin datos personales en `detalle` ni `atributos` | Contrato con los emisores y revisión de cada squad: MS08 no puede detectar un dato personal, solo limita el tamaño y el tipo. |
| Visibilidad por permiso | Servicio: cada consulta filtra por `tipo_evento` según los permisos `ms08:*` del Gateway. |
| Comparación de 2 a 10 eventos y ubicaciones de 1 a 10, del mismo tipo | Servicio. |
| Paginación de 20 por defecto y máximo 100 | Servicio. |
| Ningún servicio accede a la base | Red: la base no es pública y solo acepta la máquina de la aplicación. |

---

## Operaciones y consultas

Cada escritura es un solo `insert`: se guarda completo o no se guarda. No hay transacciones multi-documento.

| Operación | Lecturas | Escrituras |
|---|---|---|
| Registrar un hecho ([RF-AYA-01](1_requirements.md)) | — | `insert` en `audit_events`. Si el `_id` ya existe, no escribe y responde 202. |
| Consultar registros ([RF-AYA-02](1_requirements.md)) | `find` con los filtros recibidos, paginado | Ninguna |
| Consultar un registro ([RF-AYA-03](1_requirements.md)) | `findById` | Ninguna |
| Indicadores de un evento ([RF-AYA-04](1_requirements.md)) | Todos los hechos del `evento_id` (índice `evento_fecha`) | Ninguna: se calculan en memoria y no se guardan |
| Comparar eventos ([RF-AYA-05](1_requirements.md)) | Los hechos de cada `evento_id` pedido | Ninguna |
| Ubicaciones más lentas ([RF-AYA-06](1_requirements.md)) | Los hechos de cada `evento_id` pedido | Ninguna |
| Retención ([RF-AYA-07](1_requirements.md)) | — | Índice TTL: MongoDB borra los vencidos. De respaldo, una tarea diaria (`delete` de los que tienen `fecha_hora` anterior al límite) |

**Listado `GET /api/v1/audit/events`** (parámetros en [F-02](2_forms-inputs.md#f-02-búsqueda-y-filtros-de-auditoría)):
* Filtros por igualdad: `modulo`, `accion`, `usuario_id`, `evento_id`, `recurso_afectado`, `correlation_id`. Todos opcionales y combinables.
* `tipo_evento`: igualdad. Si no se envía y el usuario solo tiene uno de los dos permisos de lectura, se aplica el único tipo que puede ver. Los registros sin tipo (administrativos) solo los ven quienes tienen ambos permisos.
* `desde` y `hasta`: rango sobre `fecha_hora`, ambos incluidos.
* Orden: `fecha_hora` descendente y luego `id`, para que las páginas no repitan ni omitan registros.

---

## Configuración

Variables de entorno que afectan los datos (valores por defecto de `application.yml`):

| Variable | Por defecto | Uso |
|---|---|---|
| `MONGODB_URI` | `mongodb://localhost:27017/campusseguro_audit` | Cadena de conexión. En producción es un secreto del entorno y nunca va en el repositorio. |
| `SERVER_PORT` | `8088` | Puerto del servicio. |
| `APP_TIMEZONE` | `America/Bogota` | Zona horaria de la aplicación. |
| `AUDIT_RETENTION_DAYS` | `180` | Días que se conserva un registro (6 meses). |
| `AUDIT_TTL_INDEX_ENABLED` | `true` | Crea el índice TTL para que MongoDB borre solo los registros vencidos. |
| `AUDIT_CLEANUP_JOB_ENABLED` | `false` | Enciende la tarea diaria de borrado, solo para una base sin soporte de TTL (apague también el índice TTL). |
| `app.audit.cleanup-cron` (propiedad) | `0 30 3 * * *` | Hora de la tarea diaria de limpieza, en la zona `APP_TIMEZONE`. |
| `LOG_LEVEL` | `INFO` | Nivel de los registros del propio servicio. |

---

## Datos iniciales

1. **Sin siembra:** MS08 no carga datos al arrancar. La colección `audit_events` aparece con el primer hecho que llega y los índices los crea `AuditIndexInitializer` en cada arranque.
2. **Sin usuarios ni permisos propios:** los cuatro permisos de MS08 (`ms08:audit_drill:read`, `ms08:audit_real_event:read`, `ms08:indicators_drill:read` y `ms08:indicators_real_event:read`) los siembra MS01 en la [matriz de roles y permisos](../integration/permissions-matrix.md).
3. **Quién escribe:** solo los microservicios, por `POST /api/v1/audit/events` a través del Gateway, con la cabecera `X-Service-Name`. Ningún usuario escribe en MS08.
4. **Datos de prueba:** en local, `infra/docker-compose.yml` levanta MongoDB vacío; los hechos de prueba se envían por la propia API.
