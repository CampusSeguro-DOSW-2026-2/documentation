# Especificación del Modelo de Datos

Base de datos PostgreSQL propia de MS04 ([RNF-MS04-08](1_requirements.md#requerimientos-no-funcionales-rnf)). El esquema desplegado está en `database/` del repositorio de MS04. Ningún otro microservicio la lee ni la modifica; se accede a ella solo mediante la API.

## Índice de Contenido
1. [Alcance](#alcance)
2. [Convenciones](#convenciones)
3. [Tablas](#tablas)
    - [eventos](#eventos)
    - [checkins](#checkins)
    - [Vista v_checkins_vigentes](#v_checkins_vigentes)
    - [solicitudes_asistencia](#solicitudes_asistencia)
4. [Relaciones](#relaciones)
5. [Correspondencia API ↔ columnas](#correspondencia-api)
6. [Reglas de integridad](#reglas-de-integridad)
7. [Operaciones transaccionales y consultas](#operaciones)
8. [Datos iniciales y tareas periódicas](#datos-iniciales)

---

<a id="alcance"></a>
## Alcance

**Se almacena:** los eventos que MS06 le comunica a MS04 (estado, fecha de inicio, escenario y roles convocados), los check-ins de cada usuario (uno por cada reporte, de modo que el estado vigente es el más reciente y los anteriores son el histórico) y las solicitudes de asistencia generadas desde el check-in con su estado de envío a MS05.

**No se almacena, y por qué:**
* **Un historial aparte de check-ins:** el histórico es la propia tabla `checkins`, donde cada reporte es una fila ([RN-MS04-07](1_requirements.md#rn-ms04-07), [RN-MS04-08](1_requirements.md#rn-ms04-08)). No hay una tabla de historial ni una marca de “vigente”: el estado vigente se obtiene del check-in más reciente.
* **Auditoría:** la delega MS08 ([RNF-MS04-17](1_requirements.md#requerimientos-no-funcionales-rnf)). El histórico de check-ins es un dato del dominio, no auditoría. El historial de cambios de estado de un evento tampoco se guarda aquí: `eventos` solo guarda el estado actual y los cambios quedan auditados en MS08.
* **Datos personales:** del usuario solo se guarda su id (`X-User-Id`), sin nombres ni correo ([RN-MS04-02](1_requirements.md#rn-ms04-02)). Los nombres se piden a MS01 en cada consulta ([RN-MS04-25](1_requirements.md#rn-ms04-25)).
* **Población objetivo:** se consulta a MS01 por los roles convocados en cada cálculo de pendientes ([RN-MS04-15](1_requirements.md#rn-ms04-15)).
* **Datos de otros dominios** (usuarios, puntos de encuentro, edificios, atención de solicitudes, asignaciones de brigadistas, el evento en MS06): pertenecen a MS01, MS02, MS05 y MS06. MS04 solo guarda sus ids como referencia. El escenario del evento (edificios y zonas) es texto informativo que envía MS06.
* **Indicadores:** los calculan MS06 y MS08 ([RN-MS04-18](1_requirements.md#rn-ms04-18)).

---

<a id="convenciones"></a>
## Convenciones

* Nombres de tablas y columnas en `snake_case` y en español, alineados con el contrato OpenAPI.
* Fechas y horas en `TIMESTAMPTZ`, guardadas en UTC.
* Identificadores propios (check-in, solicitud): UUID generado por la base de datos (`gen_random_uuid()`). Además, cada check-in lleva un `numero` secuencial que desempata los reportes con la misma fecha/hora.
* `eventos.id` es el `eventoId` (UUID) que asigna MS06 ([RN-MS04-01](1_requirements.md#rn-ms04-01)): MS04 no lo genera. Dentro de esta base sí es llave foránea de `checkins` y `solicitudes_asistencia`.
* Identificadores externos, que no son llaves foráneas porque viven en otro microservicio:
  * `usuario_id`: UUID del usuario en MS01 (`X-User-Id`).
  * `punto_encuentro_id`, `ubicacion_id`: `BIGINT`, id de MS02.
  * `solicitud_ms05_id`: id devuelto por MS05.
* Los roles convocados y el escenario son listas de texto (`VARCHAR[]` de PostgreSQL), porque MS04 solo los guarda y los devuelve completos; nunca se filtra por un elemento de ellas, salvo los roles convocados al calcular a quién le aplica un evento.
* Los estados del evento, los estados del check-in, el tipo de ubicación y el estado de envío se guardan como texto con `@Enumerated(EnumType.STRING)` (nunca `ORDINAL`) y la base de datos los valida con un `CHECK`. Cambiar un catálogo exige actualizar el enum y el `CHECK` en la misma migración y Pull Request.
* Los check-ins nunca se actualizan ni se eliminan, y no hay borrado físico de ningún registro ([RN-MS04-08](1_requirements.md#rn-ms04-08), [RN-MS04-17](1_requirements.md#rn-ms04-17)).
* El esquema se versiona con migraciones (Flyway o Liquibase); no se usa generación automática de esquema en producción.

---

<a id="tablas"></a>
## Tablas

<a id="eventos"></a>
### eventos
Una fila por evento que MS06 ha comunicado a MS04. Se crea con la primera llamada de estado del evento ([RF-MS04-11](1_requirements.md#rf-ms04-11)), que es también la única que la modifica. Un evento que no existe aquí no admite check-ins (404).

| Columna | Tipo | Nulo | Por defecto | Restricciones | Descripción |
|---|---|---|---|---|---|
| `id` | UUID | No | — | PK | `eventoId` asignado por MS06. No lo genera MS04. |
| `nombre` | VARCHAR(150) | No | — | — | Nombre informativo del evento ([RN-MS04-24](1_requirements.md#rn-ms04-24)). |
| `simulacro` | BOOLEAN | No | — | — | `true` es un simulacro y `false` un evento real; los dos siguen los mismos estados ([RN-MS04-20](1_requirements.md#rn-ms04-20)). No cambia una vez guardado. |
| `estado` | VARCHAR(12) | No | — | CHECK IN (`BORRADOR`, `PROGRAMADO`, `EN_EJECUCION`, `FINALIZADO`) | Estado del evento ([Estados del evento](1_requirements.md#estados-del-evento)). El check-in está habilitado si y solo si es `EN_EJECUCION` ([RN-MS04-19](1_requirements.md#rn-ms04-19)). Solo cambia por las transiciones de [RN-MS04-21](1_requirements.md#rn-ms04-21). |
| `fecha_inicio` | TIMESTAMPTZ | Sí | NULL | CHECK: nula si y solo si `estado = 'BORRADOR'` | Fecha de inicio que informa MS06 ([RN-MS04-24](1_requirements.md#rn-ms04-24)). MS04 no la compara con el reloj: no inicia el evento por sí misma. |
| `fecha_cierre` | TIMESTAMPTZ | Sí | NULL | CHECK: no nula si y solo si `estado = 'FINALIZADO'` | Fecha/hora en que MS04 recibió `FINALIZADO` (MS06 no envía fecha de cierre). |
| `roles_convocados` | VARCHAR(20)[] | No | — | CHECK: al menos 1 elemento y todos dentro de `ESTUDIANTE`, `PROFESOR`, `ADMINISTRATIVO`, `BRIGADISTA`, `COORDINADOR`, `ORGANIZADOR` | Roles a los que les aplica el evento y que forman su población objetivo ([RN-MS04-01](1_requirements.md#rn-ms04-01), [RN-MS04-15](1_requirements.md#rn-ms04-15)). Sin repetidos (lo valida el servicio). |
| `escenario_edificios` | VARCHAR(50)[] | No | `'{}'` | — | Edificios del escenario, tal como los envía MS06. Informativos ([RN-MS04-24](1_requirements.md#rn-ms04-24)). |
| `escenario_zonas` | VARCHAR(50)[] | No | `'{}'` | — | Zonas del escenario, tal como las envía MS06. Informativas. |
| `creado_en` | TIMESTAMPTZ | No | `now()` | — | Primera vez que MS04 recibió el evento. |
| `actualizado_en` | TIMESTAMPTZ | No | `now()` | — | Último cambio recibido de MS06. |

**Índices:** PK `id`; `idx_eventos_estado (estado)` para listar los eventos en ejecución ([RF-MS04-01](1_requirements.md#rf-ms04-01)).

<a id="checkins"></a>
### checkins
Una fila por cada estado reportado por un usuario en un evento. Solo admite INSERT: un cambio de estado es una fila nueva y las anteriores quedan como histórico ([RN-MS04-07](1_requirements.md#rn-ms04-07), [RN-MS04-08](1_requirements.md#rn-ms04-08)). El estado vigente de un usuario en un evento es su fila más reciente ([vista](#v_checkins_vigentes)).

| Columna | Tipo | Nulo | Por defecto | Restricciones | Descripción |
|---|---|---|---|---|---|
| `id` | UUID | No | `gen_random_uuid()` | PK | Id del check-in. |
| `numero` | BIGINT | No | `GENERATED ALWAYS AS IDENTITY` | UNIQUE (`uq_checkins_numero`) | Secuencia global. Desempata dos check-ins con la misma `registrado_en`: el de mayor `numero` es el más reciente. |
| `evento_id` | UUID | No | — | FK → `eventos.id` | Evento al que pertenece el reporte. |
| `usuario_id` | UUID | No | — | — | Usuario que reporta (`X-User-Id`). No es única: un usuario puede tener varios check-ins en el mismo evento. |
| `estado` | VARCHAR(20) | No | — | CHECK IN (`SAFE`, `EVACUATING`, `NEEDS_ASSISTANCE`, `OFF_CAMPUS`) | Estado reportado ([RN-MS04-03](1_requirements.md#rn-ms04-03)). |
| `punto_encuentro_id` | BIGINT | Sí | NULL | CHECK: no nulo si y solo si `estado = 'SAFE'` | Punto de encuentro de MS02 ([RN-MS04-04](1_requirements.md#rn-ms04-04)). |
| `ubicacion_tipo` | VARCHAR(20) | Sí | NULL | CHECK IN (`EDIFICIO`, `ESPACIO`, `PUNTO_ENCUENTRO`); no nulo si y solo si `estado = 'NEEDS_ASSISTANCE'` | [RN-MS04-09](1_requirements.md#rn-ms04-09). |
| `ubicacion_id` | BIGINT | Sí | NULL | No nulo si y solo si `ubicacion_tipo` no es nulo | Id de MS02 del tipo indicado. |
| `registrado_en` | TIMESTAMPTZ | No | `clock_timestamp()` | — | Fecha/hora real del reporte. Se usa `clock_timestamp()` y no `now()`, que devuelve la hora de inicio de la transacción. |

**Índices:** PK `id`; único `numero`; `idx_checkins_vigente (evento_id, usuario_id, registrado_en DESC, numero DESC)` para obtener el check-in más reciente de cada usuario; `idx_checkins_evento_fecha (evento_id, registrado_en)` para el histórico y la lista de asistencia.

*No hay restricción UNIQUE sobre (`evento_id`, `usuario_id`): permitir varios check-ins del mismo usuario en el mismo evento es lo que forma el histórico.*

<a id="v_checkins_vigentes"></a>
### Vista v_checkins_vigentes
Devuelve, por cada usuario y evento, su check-in más reciente: el estado vigente. La usan el consolidado, las personas por estado, los pendientes, la lista de asistencia, el propio estado y el detalle para MS06.

```sql
CREATE VIEW v_checkins_vigentes AS
SELECT DISTINCT ON (evento_id, usuario_id) *
FROM checkins
ORDER BY evento_id, usuario_id, registrado_en DESC, numero DESC;
```

<a id="solicitudes_asistencia"></a>
### solicitudes_asistencia
Solicitudes de asistencia generadas desde el check-in y su estado de envío a MS05 ([RF-MS04-04](1_requirements.md#rf-ms04-04)). Una por usuario y evento ([RN-MS04-11](1_requirements.md#rn-ms04-11)).

| Columna | Tipo | Nulo | Por defecto | Restricciones | Descripción |
|---|---|---|---|---|---|
| `id` | UUID | No | `gen_random_uuid()` | PK | Id propio; se envía a MS05 como llave de idempotencia ([RN-MS04-12](1_requirements.md#rn-ms04-12)). |
| `checkin_id` | UUID | No | — | FK → `checkins.id` | Check-in `NEEDS_ASSISTANCE` que originó la solicitud. |
| `evento_id` | UUID | No | — | FK → `eventos.id`; UNIQUE (con `usuario_id`, `uq_solicitudes_evento_usuario`) | |
| `usuario_id` | UUID | No | — | UNIQUE (con `evento_id`) | |
| `ubicacion_tipo` | VARCHAR(20) | No | — | CHECK IN (`EDIFICIO`, `ESPACIO`, `PUNTO_ENCUENTRO`) | Ubicación enviada a MS05. |
| `ubicacion_id` | BIGINT | No | — | — | |
| `solicitud_ms05_id` | VARCHAR(64) | Sí | NULL | Obligatorio si `estado_envio = 'ENVIADA'` | Id devuelto por MS05 ([RN-MS04-10](1_requirements.md#rn-ms04-10)). |
| `estado_envio` | VARCHAR(16) | No | `'PENDIENTE_ENVIO'` | CHECK IN (`ENVIADA`, `PENDIENTE_ENVIO`, `RECHAZADA`) | [RN-MS04-10](1_requirements.md#rn-ms04-10). |
| `intentos` | INTEGER | No | `0` | ≥ 0 | Intentos de envío a MS05. |
| `ultimo_intento_en` | TIMESTAMPTZ | Sí | NULL | — | |
| `reportada_en` | TIMESTAMPTZ | No | `now()` | — | Fecha/hora del check-in que originó la solicitud. |
| `actualizado_en` | TIMESTAMPTZ | No | `now()` | — | |

**Índices:** PK `id`; único (`evento_id`, `usuario_id`); `idx_solicitudes_pendientes (estado_envio)` para el reenvío; `idx_solicitudes_evento (evento_id, reportada_en)`.

*`solicitud_ms05_id` es VARCHAR porque el contrato de MS05 aún no define su tipo ([decisión abierta n.º 4](1_requirements.md#dependencias-y-decisiones-abiertas)).*

---

<a id="relaciones"></a>
## Relaciones

| Relación | Cardinalidad | Borrado |
|---|---|---|
| `eventos` → `checkins` | 1 a N (varios por usuario; el vigente es el más reciente) | RESTRICT |
| `checkins` → `solicitudes_asistencia` | 1 a 0..1 | RESTRICT |
| `eventos` → `solicitudes_asistencia` | 1 a N (una por usuario) | RESTRICT |

Ninguna tabla tiene llaves foráneas hacia otros microservicios: `usuario_id`, `punto_encuentro_id`, `ubicacion_id` y `solicitud_ms05_id` se validan por API ([RN-MS04-02](1_requirements.md#rn-ms04-02)), y el `eventoId` lo asigna MS06.

---
## Modelo Logico
*(espacio reservado para insertar el diagrama del modelo lógico de MS04: `diagramas/modelo-logico.png`)*

---

<a id="correspondencia-api"></a>
## Correspondencia API ↔ columnas

Donde aparecen dos nombres, el primero es el de las rutas internas acordadas con MS06 (camelCase) y el segundo el del contrato protegido de MS04 (snake_case).

| Campo del contrato | Columna | Notas |
|---|---|---|
| `eventoId` / `evento_id` | `eventos.id`, `checkins.evento_id` | Igual al `eventoId` de la ruta. |
| `nombre`, `simulacro`, `estado` (del evento), `rolesConvocados` / `roles_convocados` | `eventos.nombre`, `eventos.simulacro`, `eventos.estado`, `eventos.roles_convocados` | Los envía MS06 ([RF-MS04-11](1_requirements.md#rf-ms04-11)) y los devuelven el PUT y la consulta de eventos ([RF-MS04-13](1_requirements.md#rf-ms04-13)). |
| `fechaInicio` / `fecha_inicio` | `eventos.fecha_inicio` | Se recibe en ISO-8601 con desfase horario y se guarda en UTC. |
| `escenario.edificios`, `escenario.zonas` (igual en ambos contratos) | `eventos.escenario_edificios`, `eventos.escenario_zonas` | Texto informativo. |
| `checkinHabilitado` / `checkin_habilitado` (respuesta del PUT y de la consulta de eventos) | — | Se deriva de `eventos.estado = 'EN_EJECUCION'`; no se guarda. |
| `id` (check-in) | `checkins.id` | |
| `usuario_id` / `usuarioId` | `checkins.usuario_id` | En el check-in viene de `X-User-Id`; nunca del cuerpo. |
| `estado` (del check-in) | `checkins.estado` | |
| `punto_encuentro_id` | `checkins.punto_encuentro_id` | Solo con `SAFE`. |
| `zonaId` (detalle para MS06) | `checkins.punto_encuentro_id` | Solo con `SAFE`; nulo en los demás estados. |
| `ubicacion.tipo`, `ubicacion.id` | `ubicacion_tipo`, `ubicacion_id` | Solo con `NEEDS_ASSISTANCE`. |
| `fecha_hora` (respuesta del check-in), `registrado_en` (histórico), `registradoEn` (detalle para MS06) | `checkins.registrado_en` | |
| `nombre` (listas de personas) | — | Lo entrega MS01 en cada consulta; no se guarda. |
| `primer_checkin_en` (resultados) | `MIN(checkins.registrado_en)` por usuario y evento | |
| `solicitud_asistencia.id` | `solicitudes_asistencia.solicitud_ms05_id` | NULL mientras no llegue a MS05. |
| `solicitud_asistencia.estado_envio` | `solicitudes_asistencia.estado_envio` | |
| `fecha_reporte` (lista de asistencia) | `checkins.registrado_en` del check-in vigente | |
| `simulacro`, `estado_evento`, `fecha_inicio`, `fecha_cierre`, `finales` (resultados y consulta de eventos) | `eventos.simulacro`, `eventos.estado`, `eventos.fecha_inicio`, `eventos.fecha_cierre`, `eventos.estado = 'FINALIZADO'` (para `finales`) | |
| `poblacion_objetivo`, `total_pendientes` | — | Se calculan con los usuarios de MS01 por rol convocado; no se guardan. |

---

<a id="reglas-de-integridad"></a>
## Reglas de integridad

| Regla | Dónde se garantiza |
|---|---|
| El estado vigente es el check-in más reciente del usuario en el evento ([RN-MS04-07](1_requirements.md#rn-ms04-07)) | BD: vista `v_checkins_vigentes` e índice `idx_checkins_vigente`, con orden (`registrado_en`, `numero`) descendente. No hay una columna de “vigente” que pueda quedar inconsistente. |
| Los check-ins solo se agregan ([RN-MS04-08](1_requirements.md#rn-ms04-08), [RN-MS04-17](1_requirements.md#rn-ms04-17)) | BD: el usuario de la aplicación solo tiene permiso de INSERT y SELECT sobre `checkins`. El código no tiene operaciones de UPDATE ni DELETE sobre ella. |
| Reportes simultáneos del mismo usuario en el mismo evento ([RF-MS04-03](1_requirements.md#rf-ms04-03), A5; [RNF-MS04-04](1_requirements.md#requerimientos-no-funcionales-rnf)) | Servicio + BD: bloqueo asesor transaccional (`pg_advisory_xact_lock`) sobre (`evento_id`, `usuario_id`) al inicio de la transacción. Se atienden uno tras otro, ambos se guardan y no hay 409 ni 500. |
| Repetir el mismo estado no crea un check-in ([RF-MS04-03](1_requirements.md#rf-ms04-03), A3) | Servicio: bajo el bloqueo anterior compara con el check-in más reciente (estado y, con `SAFE`, punto de encuentro); si no cambian, no inserta. |
| Solo los 4 estados de check-in ([RN-MS04-03](1_requirements.md#rn-ms04-03)) | BD: CHECK; en el código, enum `EstadoCheckin`. |
| Punto de encuentro solo y siempre con `SAFE` ([RN-MS04-04](1_requirements.md#rn-ms04-04)) | BD: CHECK; el servicio además valida con MS02 que exista y esté ACTIVO. |
| Ubicación solo y siempre con `NEEDS_ASSISTANCE` ([RN-MS04-09](1_requirements.md#rn-ms04-09)) | BD: CHECK. |
| Solo los 4 estados de evento y fechas coherentes con el estado ([RN-MS04-19](1_requirements.md#rn-ms04-19)) | BD: CHECK sobre `estado`, `fecha_inicio` (nula solo en `BORRADOR`) y `fecha_cierre` (no nula solo en `FINALIZADO`); en el código, enum `EstadoEvento`. |
| Roles convocados válidos y al menos uno ([RN-MS04-24](1_requirements.md#rn-ms04-24)) | BD: CHECK sobre `roles_convocados`; el servicio además rechaza repetidos. |
| Transiciones de estado del evento y `simulacro` inmutable ([RN-MS04-21](1_requirements.md#rn-ms04-21), [RN-MS04-20](1_requirements.md#rn-ms04-20)) | Servicio: lee el evento con `SELECT … FOR UPDATE` y rechaza con 409 las transiciones no permitidas; repetir `EN_EJECUCION` o `FINALIZADO` es idempotente. |
| Solo los eventos que envía MS06 existen ([RN-MS04-01](1_requirements.md#rn-ms04-01)) | Servicio: solo [RF-MS04-11](1_requirements.md#rf-ms04-11) inserta en `eventos`. Un check-in sobre un evento inexistente responde 404 (y la FK `checkins.evento_id` lo impide en la BD). |
| Check-in solo con el evento `EN_EJECUCION` ([RN-MS04-05](1_requirements.md#rn-ms04-05), [RN-MS04-19](1_requirements.md#rn-ms04-19)) | Servicio: lee `eventos.estado` con bloqueo (`SELECT … FOR SHARE`) en la misma transacción del check-in; si no es `EN_EJECUCION`, responde 409. Como el cambio de estado usa `FOR UPDATE`, un check-in en curso termina antes de que `FINALIZADO` se confirme y ninguno se cuela después. |
| Sin escrituras sobre eventos `FINALIZADO` ([RN-MS04-17](1_requirements.md#rn-ms04-17)) | Servicio: `FINALIZADO` no admite ninguna otra transición ni check-ins nuevos; sus filas no se modifican ni se eliminan. |
| Una solicitud de asistencia por usuario y evento ([RN-MS04-11](1_requirements.md#rn-ms04-11)) | BD: UNIQUE (`evento_id`, `usuario_id`) en `solicitudes_asistencia`. |
| Sin solicitudes duplicadas en MS05 al reenviar ([RN-MS04-12](1_requirements.md#rn-ms04-12)) | Servicio: envía `solicitudes_asistencia.id` como llave de idempotencia (contrato pendiente con MS05). |
| Identidad solo por cabeceras ([RN-MS04-06](1_requirements.md#rn-ms04-06)) | Servicio: `usuario_id` se toma de `X-User-Id`. |
| Rutas internas solo para `ms06-drills` ([RN-MS04-23](1_requirements.md#rn-ms04-23)) | Servicio: valida `X-Service-Name`. |

---

<a id="operaciones"></a>
## Operaciones transaccionales y consultas

Cada operación de escritura es una sola transacción: o se aplica completa o no se aplica nada. La llamada a MS02 se hace antes de abrir la transacción; el envío a MS05 se hace después de confirmarla, para no bloquear el check-in ([RN-MS04-12](1_requirements.md#rn-ms04-12)). Las operaciones de un mismo usuario y evento empiezan tomando el bloqueo asesor por (`evento_id`, `usuario_id`). Ninguna operación de MS04 llama a MS06 ni a MS03.

| Operación | Lecturas | Escrituras |
|---|---|---|
| Recibir estado del evento ([RF-11](1_requirements.md#rf-ms04-11)) | `eventos` por id con `SELECT … FOR UPDATE` | Si no existe: INSERT `eventos`. Si existe y la transición es válida: UPDATE `eventos` (`estado`, `fecha_inicio`, `nombre`, escenario, roles, `fecha_cierre` solo con `FINALIZADO`, `actualizado_en`). Repetir `EN_EJECUCION` o `FINALIZADO`: ninguna escritura. |
| Registrar check-in ([RF-02](1_requirements.md#rf-ms04-02)) | MS02: punto de encuentro (si `SAFE`), antes de la transacción. `eventos` por id con `SELECT … FOR SHARE`. | INSERT `checkins`. Si `NEEDS_ASSISTANCE`: INSERT `solicitudes_asistencia` (PENDIENTE_ENVIO). |
| Actualizar estado ([RF-03](1_requirements.md#rf-ms04-03)) | Igual que el registro + check-in más reciente del usuario en el evento (`v_checkins_vigentes`) | Si el estado o el punto de encuentro cambian: INSERT `checkins` (fila nueva). Si no cambian: ninguna escritura. Si pasa a `NEEDS_ASSISTANCE` y no hay solicitud: INSERT `solicitudes_asistencia`. Nunca UPDATE ni DELETE sobre `checkins`. |
| Enviar solicitud a MS05 ([RF-04](1_requirements.md#rf-ms04-04)) | `solicitudes_asistencia` por id | UPDATE `estado_envio`, `solicitud_ms05_id`, `intentos`, `ultimo_intento_en`. |
| Consultar estado vigente ([RF-05](1_requirements.md#rf-ms04-05)) | `v_checkins_vigentes` por (`evento_id`, `X-User-Id`) + su solicitud | Ninguna |
| Eventos activos ([RF-01](1_requirements.md#rf-ms04-01)) | `eventos` con `estado = 'EN_EJECUCION'` y el rol del usuario dentro de `roles_convocados` | Ninguna |
| Consultar eventos ([RF-13](1_requirements.md#rf-ms04-13)) | `eventos` por id, o todos con filtro opcional por `estado`; orden por `fecha_inicio` descendente (los nulos al final) y luego `creado_en` descendente | Ninguna |

**Consolidación ([RF-06](1_requirements.md#rf-ms04-06)):**
* Totales: `COUNT(*)` de `v_checkins_vigentes` del evento agrupado por `estado`; los estados sin filas se devuelven en 0 ([RN-MS04-13](1_requirements.md#rn-ms04-13)).
* Por punto de encuentro: `COUNT(*)` de `v_checkins_vigentes` con `estado = 'SAFE'` agrupado por `punto_encuentro_id`.
* Personas por estado (`/personas`): `v_checkins_vigentes` filtrada por `estado` y `punto_encuentro_id`; orden por `registrado_en` descendente y luego `numero` descendente. Los nombres se piden a MS01 por página ([RN-MS04-25](1_requirements.md#rn-ms04-25)).

**Pendientes ([RF-07](1_requirements.md#rf-ms04-07)):** usuarios ACTIVOS de MS01 por cada rol de `eventos.roles_convocados`, sin repetir a quien tiene varios roles, menos los `usuario_id` de `v_checkins_vigentes` del evento; paginado en el servicio.

**Lista de asistencia ([RF-08](1_requirements.md#rf-ms04-08)):** `v_checkins_vigentes` con `estado = 'NEEDS_ASSISTANCE'` unida con `solicitudes_asistencia` por (`evento_id`, `usuario_id`); orden por `registrado_en` ascendente (más antiguo primero) y luego `numero`.

**Histórico ([RF-09](1_requirements.md#rf-ms04-09), [F-04](2_forms-inputs.md#f-04)):** `checkins` del evento (todas las filas), con filtros opcionales por `usuario_id` y `estado`; orden por `registrado_en` ascendente y luego `numero`.

**Resultados ([RF-10](1_requirements.md#rf-ms04-10)):** totales de la consolidación + `solicitudes_asistencia` del evento + `checkins` con `estado = 'NEEDS_ASSISTANCE'` (personas que lo reportaron al menos una vez) + `MIN(registrado_en)` por usuario (primer check-in) + datos de `eventos`.

**Detalle para MS06 ([RF-12](1_requirements.md#rf-ms04-12)):** `v_checkins_vigentes` del evento, con filtro opcional por `estado`; orden por `registrado_en` ascendente y luego `numero`; `total` es el número de personas del filtro. Los nombres se piden a MS01 por página.

**Filtro del BRIGADISTA ([RN-MS04-14](1_requirements.md#rn-ms04-14)):** las consultas anteriores se restringen a los check-ins vigentes `SAFE` cuyo `punto_encuentro_id` está en sus zonas y `NEEDS_ASSISTANCE` cuya ubicación está en sus zonas, obtenidas de MS05 ([decisión abierta n.º 5](1_requirements.md#dependencias-y-decisiones-abiertas)). Los check-ins `EVACUATING` y `OFF_CAMPUS` y los pendientes no tienen ubicación en esta base, por eso no se devuelven al BRIGADISTA (sus totales salen en 0).

Los listados protegidos usan `page` (desde 0) y `size` (1 a 100, por defecto 20). El detalle para MS06 usa los mismos parámetros con el formato acordado con MS06.

---

<a id="datos-iniciales"></a>
## Datos iniciales y tareas periódicas

1. **Datos iniciales:** ninguno. Las tablas nacen vacías; los eventos se registran a medida que MS06 los comunica ([RF-MS04-11](1_requirements.md#rf-ms04-11)).
2. **Reenvío de solicitudes:** una tarea periódica toma las filas de `solicitudes_asistencia` con `estado_envio = 'PENDIENTE_ENVIO'` y las reenvía a MS05 ([RN-MS04-12](1_requirements.md#rn-ms04-12)). La frecuencia se configura por variable de entorno.
3. **Sin inicio ni cierre automático de eventos:** ninguna tarea cambia el estado de un evento ni lo compara con el reloj; solo MS06 lo cambia ([RN-MS04-19](1_requirements.md#rn-ms04-19)).
4. **Sin limpieza:** no se eliminan datos de eventos, check-ins ni solicitudes ([RN-MS04-17](1_requirements.md#rn-ms04-17)).
