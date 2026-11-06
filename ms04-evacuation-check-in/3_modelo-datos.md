# Especificación del Modelo de Datos

Base de datos PostgreSQL propia de MS04 ([RNF-MS04-08](1_requirements.md#requerimientos-no-funcionales-rnf)). El esquema desplegado está en `database/` del repositorio de MS04. Ningún otro microservicio la lee ni la modifica; se accede a ella solo mediante la API.

## Índice de Contenido
1. [Alcance](#alcance)
2. [Convenciones](#convenciones)
3. [Tablas](#tablas)
    - [eventos](#eventos)
    - [checkins](#checkins)
    - [historial_checkins](#historial_checkins)
    - [solicitudes_asistencia](#solicitudes_asistencia)
4. [Relaciones](#relaciones)
5. [Correspondencia API ↔ columnas](#correspondencia-api)
6. [Reglas de integridad](#reglas-de-integridad)
7. [Operaciones transaccionales y consultas](#operaciones)
8. [Datos iniciales y tareas periódicas](#datos-iniciales)

---

<a id="alcance"></a>
## Alcance

**Se almacena:** los eventos sobre los que MS04 ha operado, el estado vigente de cada usuario por evento, el histórico de sus cambios y las solicitudes de asistencia generadas desde el check-in con su estado de envío a MS05.

**No se almacena, y por qué:**
* **Auditoría:** la delega MS08 ([RNF-MS04-17](1_requirements.md#requerimientos-no-funcionales-rnf)). El histórico de check-ins es un dato del dominio, no auditoría.
* **Datos personales:** del usuario solo se guarda su id (`X-User-Id`), sin nombres ni correo ([RN-MS04-02](1_requirements.md#rn-ms04-02)).
* **Población objetivo y destinatarios:** se consultan a MS03 en cada cálculo de pendientes ([RN-MS04-15](1_requirements.md#rn-ms04-15)).
* **Datos de otros dominios** (alertas, puntos de encuentro, edificios, atención de solicitudes, asignaciones de brigadistas): pertenecen a MS02, MS03 y MS05. MS04 solo guarda sus ids como referencia.
* **Indicadores:** los calculan MS06 y MS08 ([RN-MS04-18](1_requirements.md#rn-ms04-18)).

---

<a id="convenciones"></a>
## Convenciones

* Nombres de tablas y columnas en `snake_case` y en español, alineados con el contrato OpenAPI.
* Fechas y horas en `TIMESTAMPTZ`, guardadas en UTC.
* Identificadores propios (check-in, histórico, solicitud): UUID generado por la base de datos (`gen_random_uuid()`).
* Identificadores externos, que no son llaves foráneas porque viven en otro microservicio:
  * `evento_id`: UUID de la alerta en MS03 ([RN-MS04-01](1_requirements.md#rn-ms04-01)).
  * `usuario_id`: UUID del usuario en MS01 (`X-User-Id`).
  * `punto_encuentro_id`, `ubicacion_id`: `BIGINT`, id de MS02.
  * `solicitud_ms05_id`: id devuelto por MS05.
* Los estados del check-in, el tipo de ubicación y el estado de envío se guardan como texto con `@Enumerated(EnumType.STRING)` (nunca `ORDINAL`) y la base de datos los valida con un `CHECK`. Cambiar un catálogo exige actualizar el enum y el `CHECK` en la misma migración y Pull Request.
* No hay borrado físico de ningún registro ([RN-MS04-08](1_requirements.md#rn-ms04-08), [RN-MS04-17](1_requirements.md#rn-ms04-17)).
* El esquema se versiona con migraciones (Flyway o Liquibase); no se usa generación automática de esquema en producción.

---

<a id="tablas"></a>
## Tablas

<a id="eventos"></a>
### eventos
Una fila por alerta de MS03 sobre la que MS04 ha operado. Se crea la primera vez que MS04 la confirma como Activa con MS03.

| Columna | Tipo | Nulo | Por defecto | Restricciones | Descripción |
|---|---|---|---|---|---|
| `id` | UUID | No | — | PK | Id de la alerta en MS03. No lo genera MS04. |
| `tipo` | VARCHAR(20) | No | — | — | Tipo de alerta según MS03 (p. ej. `SIMULATION`), copiado al registrar el evento. |
| `estado` | VARCHAR(12) | No | `'VIGENTE'` | CHECK IN (`VIGENTE`, `NO_VIGENTE`) | [RF-MS04-09](1_requirements.md#rf-ms04-09). Solo pasa de `VIGENTE` a `NO_VIGENTE`. |
| `fecha_inicio` | TIMESTAMPTZ | No | — | — | Fecha de activación de la alerta según MS03 (`activatedAt`). |
| `fecha_cierre` | TIMESTAMPTZ | Sí | NULL | Obligatoria si `estado = 'NO_VIGENTE'` | Fecha de finalización según MS03 (`endedAt`). |
| `creado_en` | TIMESTAMPTZ | No | `now()` | — | Primera vez que MS04 registró el evento. |

**Índices:** PK `id`; `idx_eventos_estado (estado)`.

<a id="checkins"></a>
### checkins
Estado **vigente** de cada usuario en cada evento. Una sola fila por usuario y evento ([RN-MS04-07](1_requirements.md#rn-ms04-07)).

| Columna | Tipo | Nulo | Por defecto | Restricciones | Descripción |
|---|---|---|---|---|---|
| `id` | UUID | No | `gen_random_uuid()` | PK | Id del check-in. |
| `evento_id` | UUID | No | — | FK → `eventos.id`; UNIQUE (con `usuario_id`, `uq_checkins_evento_usuario`) | |
| `usuario_id` | UUID | No | — | UNIQUE (con `evento_id`) | Usuario que reporta (`X-User-Id`). |
| `estado` | VARCHAR(20) | No | — | CHECK IN (`SAFE`, `EVACUATING`, `NEEDS_ASSISTANCE`, `OFF_CAMPUS`) | Estado vigente ([RN-MS04-03](1_requirements.md#rn-ms04-03)). |
| `punto_encuentro_id` | BIGINT | Sí | NULL | CHECK: no nulo si y solo si `estado = 'SAFE'` | Punto de encuentro de MS02 ([RN-MS04-04](1_requirements.md#rn-ms04-04)). |
| `ubicacion_tipo` | VARCHAR(20) | Sí | NULL | CHECK IN (`EDIFICIO`, `ESPACIO`, `PUNTO_ENCUENTRO`); no nulo si y solo si `estado = 'NEEDS_ASSISTANCE'` | [RN-MS04-09](1_requirements.md#rn-ms04-09). |
| `ubicacion_id` | BIGINT | Sí | NULL | No nulo si y solo si `ubicacion_tipo` no es nulo | Id de MS02 del tipo indicado. |
| `version` | INTEGER | No | `0` | — | Bloqueo optimista (`@Version`): un cambio simultáneo responde 409 ([RNF-MS04-04](1_requirements.md#requerimientos-no-funcionales-rnf)). |
| `creado_en` | TIMESTAMPTZ | No | `now()` | — | Fecha/hora del primer check-in del usuario en el evento. |
| `actualizado_en` | TIMESTAMPTZ | No | `now()` | — | Fecha/hora del último cambio de estado. |

**Índices:** PK `id`; único (`evento_id`, `usuario_id`); `idx_checkins_evento_estado (evento_id, estado)` para la consolidación; `idx_checkins_evento_punto (evento_id, punto_encuentro_id)` para el conteo por punto de encuentro.

<a id="historial_checkins"></a>
### historial_checkins
Una fila por cada estado reportado, incluido el vigente ([RN-MS04-08](1_requirements.md#rn-ms04-08)). Solo admite INSERT.

| Columna | Tipo | Nulo | Por defecto | Restricciones | Descripción |
|---|---|---|---|---|---|
| `id` | UUID | No | `gen_random_uuid()` | PK | |
| `checkin_id` | UUID | No | — | FK → `checkins.id` | Check-in al que pertenece el cambio. |
| `evento_id` | UUID | No | — | FK → `eventos.id` | Repetido para filtrar sin unir tablas. |
| `usuario_id` | UUID | No | — | — | Repetido para filtrar por usuario ([F-04](2_forms-inputs.md#f-04)). |
| `estado` | VARCHAR(20) | No | — | Mismo CHECK que `checkins.estado` | Estado reportado. |
| `punto_encuentro_id` | BIGINT | Sí | NULL | Mismo CHECK que `checkins` | |
| `ubicacion_tipo` | VARCHAR(20) | Sí | NULL | Mismo CHECK que `checkins` | |
| `ubicacion_id` | BIGINT | Sí | NULL | Mismo CHECK que `checkins` | |
| `registrado_en` | TIMESTAMPTZ | No | `now()` | — | Fecha/hora del reporte. |

**Índices:** PK `id`; `idx_historial_evento (evento_id, registrado_en)`; `idx_historial_evento_usuario (evento_id, usuario_id, registrado_en)`.

<a id="solicitudes_asistencia"></a>
### solicitudes_asistencia
Solicitudes de asistencia generadas desde el check-in y su estado de envío a MS05 ([RF-MS04-04](1_requirements.md#rf-ms04-04)). Una por usuario y evento ([RN-MS04-11](1_requirements.md#rn-ms04-11)).

| Columna | Tipo | Nulo | Por defecto | Restricciones | Descripción |
|---|---|---|---|---|---|
| `id` | UUID | No | `gen_random_uuid()` | PK | Id propio; se envía a MS05 como llave de idempotencia ([RN-MS04-12](1_requirements.md#rn-ms04-12)). |
| `checkin_id` | UUID | No | — | FK → `checkins.id` | Check-in que la originó. |
| `evento_id` | UUID | No | — | FK → `eventos.id`; UNIQUE (con `usuario_id`, `uq_solicitudes_evento_usuario`) | |
| `usuario_id` | UUID | No | — | UNIQUE (con `evento_id`) | |
| `ubicacion_tipo` | VARCHAR(20) | No | — | CHECK IN (`EDIFICIO`, `ESPACIO`, `PUNTO_ENCUENTRO`) | Ubicación enviada a MS05. |
| `ubicacion_id` | BIGINT | No | — | — | |
| `solicitud_ms05_id` | VARCHAR(64) | Sí | NULL | Obligatorio si `estado_envio = 'ENVIADA'` | Id devuelto por MS05 ([RN-MS04-10](1_requirements.md#rn-ms04-10)). |
| `estado_envio` | VARCHAR(16) | No | `'PENDIENTE_ENVIO'` | CHECK IN (`ENVIADA`, `PENDIENTE_ENVIO`, `RECHAZADA`) | [RN-MS04-10](1_requirements.md#rn-ms04-10). |
| `intentos` | INTEGER | No | `0` | ≥ 0 | Intentos de envío a MS05. |
| `ultimo_intento_en` | TIMESTAMPTZ | Sí | NULL | — | |
| `reportada_en` | TIMESTAMPTZ | No | `now()` | — | Fecha/hora en que el usuario reportó Requiero asistencia. |
| `actualizado_en` | TIMESTAMPTZ | No | `now()` | — | |

**Índices:** PK `id`; único (`evento_id`, `usuario_id`); `idx_solicitudes_pendientes (estado_envio)` para el reenvío; `idx_solicitudes_evento (evento_id, reportada_en)` para la lista de asistencia.

*`solicitud_ms05_id` es VARCHAR porque el contrato de MS05 aún no define su tipo ([decisión abierta n.º 4](1_requirements.md#dependencias-y-decisiones-abiertas)).*

---

<a id="relaciones"></a>
## Relaciones

| Relación | Cardinalidad | Borrado |
|---|---|---|
| `eventos` → `checkins` | 1 a N (uno por usuario) | RESTRICT |
| `checkins` → `historial_checkins` | 1 a N (mínimo 1 fila por check-in) | RESTRICT |
| `checkins` → `solicitudes_asistencia` | 1 a 0..1 | RESTRICT |
| `eventos` → `solicitudes_asistencia` | 1 a N (una por usuario) | RESTRICT |

Ninguna tabla tiene llaves foráneas hacia otros microservicios: `evento_id`, `usuario_id`, `punto_encuentro_id`, `ubicacion_id` y `solicitud_ms05_id` se validan por API ([RN-MS04-02](1_requirements.md#rn-ms04-02)).

---
## Modelo Logico
*(espacio reservado para insertar el diagrama del modelo lógico de MS04: `diagramas/modelo-logico.png`)*

---

<a id="correspondencia-api"></a>
## Correspondencia API ↔ columnas

| Campo del contrato | Columna | Notas |
|---|---|---|
| `id` (check-in) | `checkins.id` | |
| `evento_id` | `eventos.id`, `checkins.evento_id` | Igual al `eventId` de la ruta. |
| `usuario_id` | `checkins.usuario_id` | Viene de `X-User-Id`; nunca del cuerpo. |
| `estado` | `checkins.estado`, `historial_checkins.estado` | |
| `punto_encuentro_id` | `checkins.punto_encuentro_id` | Solo con `SAFE`. |
| `ubicacion.tipo`, `ubicacion.id` | `ubicacion_tipo`, `ubicacion_id` | Solo con `NEEDS_ASSISTANCE`. |
| `fecha_hora` (respuesta del check-in) | `checkins.actualizado_en` | |
| `primer_checkin_en` (resultados) | `checkins.creado_en` | |
| `registrado_en` (histórico) | `historial_checkins.registrado_en` | |
| `solicitud_asistencia.id` | `solicitudes_asistencia.solicitud_ms05_id` | NULL mientras no llegue a MS05. |
| `solicitud_asistencia.estado_envio` | `solicitudes_asistencia.estado_envio` | |
| `fecha_reporte` (lista de asistencia) | `solicitudes_asistencia.reportada_en` | |
| `tipo`, `fecha_inicio`, `fecha_cierre`, `finales` (resultados) | `eventos.tipo`, `eventos.fecha_inicio`, `eventos.fecha_cierre`, `eventos.estado = 'NO_VIGENTE'` | |
| `poblacion_objetivo`, `total_pendientes` | — | Se calculan con los destinatarios de MS03; no se guardan. |

---

<a id="reglas-de-integridad"></a>
## Reglas de integridad

| Regla | Dónde se garantiza |
|---|---|
| Un solo estado vigente por usuario y evento ([RN-MS04-07](1_requirements.md#rn-ms04-07)) | BD: UNIQUE (`evento_id`, `usuario_id`) en `checkins`. Un INSERT concurrente que la viola responde 409 `CONFLICTO`. |
| Cambios simultáneos sobre el mismo check-in ([RF-MS04-03](1_requirements.md#rf-ms04-03), A5) | BD + JPA: columna `version` (bloqueo optimista); el perdedor responde 409 `CONFLICTO`. |
| Solo los 4 estados ([RN-MS04-03](1_requirements.md#rn-ms04-03)) | BD: CHECK; en el código, enum `EstadoCheckin`. |
| Punto de encuentro solo y siempre con `SAFE` ([RN-MS04-04](1_requirements.md#rn-ms04-04)) | BD: CHECK; el servicio además valida con MS02 que exista y esté ACTIVO. |
| Ubicación solo y siempre con `NEEDS_ASSISTANCE` ([RN-MS04-09](1_requirements.md#rn-ms04-09)) | BD: CHECK. |
| Histórico inmutable ([RN-MS04-08](1_requirements.md#rn-ms04-08)) | BD: el usuario de la aplicación solo tiene permiso de INSERT y SELECT sobre `historial_checkins`. |
| Sin escrituras en eventos no vigentes ([RN-MS04-05](1_requirements.md#rn-ms04-05), [RN-MS04-17](1_requirements.md#rn-ms04-17)) | Servicio: lee `eventos.estado` con bloqueo (`SELECT … FOR SHARE`) en la misma transacción del check-in; si es `NO_VIGENTE`, responde 409. |
| `eventos.estado` solo avanza a `NO_VIGENTE` | Servicio; la operación es idempotente ([RF-MS04-09](1_requirements.md#rf-ms04-09), A1). |
| Una solicitud de asistencia por usuario y evento ([RN-MS04-11](1_requirements.md#rn-ms04-11)) | BD: UNIQUE (`evento_id`, `usuario_id`) en `solicitudes_asistencia`. |
| Sin solicitudes duplicadas en MS05 al reenviar ([RN-MS04-12](1_requirements.md#rn-ms04-12)) | Servicio: envía `solicitudes_asistencia.id` como llave de idempotencia (contrato pendiente con MS05). |
| Identidad solo por cabeceras ([RN-MS04-06](1_requirements.md#rn-ms04-06)) | Servicio: `usuario_id` se toma de `X-User-Id`. |

---

<a id="operaciones"></a>
## Operaciones transaccionales y consultas

Cada operación de escritura es una sola transacción: o se aplica completa o no se aplica nada. Las llamadas a MS02 y MS03 se hacen antes de abrir la transacción; el envío a MS05 se hace después de confirmarla, para no bloquear el check-in ([RN-MS04-12](1_requirements.md#rn-ms04-12)).

| Operación | Lecturas | Escrituras |
|---|---|---|
| Registrar check-in ([RF-02](1_requirements.md#rf-ms04-02)) | MS03: evento Activo y aplicable. MS02: punto de encuentro (si `SAFE`). `eventos` por id. | Si el evento no existe: INSERT `eventos` (VIGENTE). INSERT `checkins` + INSERT `historial_checkins`. Si `NEEDS_ASSISTANCE`: INSERT `solicitudes_asistencia` (PENDIENTE_ENVIO). |
| Actualizar estado ([RF-03](1_requirements.md#rf-ms04-03)) | Igual que el registro + `checkins` por (`evento_id`, `usuario_id`) | UPDATE `checkins` (estado, punto, ubicación, `actualizado_en`, `version`) + INSERT `historial_checkins`. Si el estado y el punto no cambian: ninguna escritura. Si pasa a `NEEDS_ASSISTANCE` y no hay solicitud: INSERT `solicitudes_asistencia`. |
| Enviar solicitud a MS05 ([RF-04](1_requirements.md#rf-ms04-04)) | `solicitudes_asistencia` por id | UPDATE `estado_envio`, `solicitud_ms05_id`, `intentos`, `ultimo_intento_en`. |
| Marcar evento no vigente ([RF-09](1_requirements.md#rf-ms04-09)) | MS03: estado de la alerta | UPDATE `eventos` (`estado = 'NO_VIGENTE'`, `fecha_cierre`) solo si estaba `VIGENTE`. |
| Consultar estado vigente ([RF-05](1_requirements.md#rf-ms04-05)) | `checkins` por (`evento_id`, `X-User-Id`) + su solicitud | Ninguna |

**Consolidación ([RF-06](1_requirements.md#rf-ms04-06)):**
* Totales: `COUNT(*)` de `checkins` del evento agrupado por `estado`; los estados sin filas se devuelven en 0 ([RN-MS04-13](1_requirements.md#rn-ms04-13)).
* Por punto de encuentro: `COUNT(*)` de `checkins` con `estado = 'SAFE'` agrupado por `punto_encuentro_id`.
* Personas por estado: filtro por `estado` y `punto_encuentro_id`; orden por `actualizado_en` descendente y luego `id`.

**Pendientes ([RF-07](1_requirements.md#rf-ms04-07)):** destinatarios de MS03 menos los `usuario_id` de `checkins` del evento; paginado en el servicio.

**Lista de asistencia ([RF-08](1_requirements.md#rf-ms04-08)):** `checkins` con `estado = 'NEEDS_ASSISTANCE'` unidos con `solicitudes_asistencia`; orden por `reportada_en` ascendente (más antiguo primero).

**Histórico ([RF-09](1_requirements.md#rf-ms04-09), [F-04](2_forms-inputs.md#f-04)):** `historial_checkins` del evento, con filtros opcionales por `usuario_id` y `estado`; orden por `registrado_en` ascendente y luego `id`.

**Resultados ([RF-10](1_requirements.md#rf-ms04-10)):** totales de la consolidación + `solicitudes_asistencia` del evento + `checkins.creado_en` de cada usuario (primer check-in) + datos de `eventos`.

**Filtro del BRIGADISTA ([RN-MS04-14](1_requirements.md#rn-ms04-14)):** las consultas anteriores se restringen a los puntos de encuentro y ubicaciones de sus zonas asignadas, obtenidas de MS05 ([decisión abierta n.º 5](1_requirements.md#dependencias-y-decisiones-abiertas)).

Todos los listados usan `page` (desde 0) y `size` (1 a 100, por defecto 20).

---

<a id="datos-iniciales"></a>
## Datos iniciales y tareas periódicas

1. **Datos iniciales:** ninguno. Las tablas nacen vacías; los eventos se registran a medida que MS03 los confirma.
2. **Reenvío de solicitudes:** una tarea periódica toma las filas de `solicitudes_asistencia` con `estado_envio = 'PENDIENTE_ENVIO'` y las reenvía a MS05 ([RN-MS04-12](1_requirements.md#rn-ms04-12)). La frecuencia se configura por variable de entorno.
3. **Sin limpieza:** no se eliminan datos de eventos, check-ins, histórico ni solicitudes ([RN-MS04-17](1_requirements.md#rn-ms04-17)).
