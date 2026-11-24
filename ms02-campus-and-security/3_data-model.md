# Especificación del Modelo de Datos

Base de datos PostgreSQL propia de MS02 ([RNF-MS02-01](1_requirements.md#requerimientos-no-funcionales-rnf)). El esquema lo crean las migraciones Flyway de `src/main/resources/db/migration` del repositorio de MS02 (V1 a V17). Ningún otro microservicio la lee ni la modifica; se accede a ella solo por la API.

## Índice de Contenido
1. [Alcance](#alcance)
2. [Convenciones](#convenciones)
3. [Tablas](#tablas)
    - [campus](#campus)
    - [zona_abierta](#zona_abierta)
    - [edificio](#edificio)
    - [piso](#piso)
    - [espacio](#espacio)
    - [salida](#salida)
    - [escalera y escalera_piso](#escalera)
    - [zona_segura](#zona_segura)
    - [ruta_evacuacion](#ruta_evacuacion)
    - [recomendacion_preventiva](#recomendacion_preventiva)
    - [imagen](#imagen)
4. [Relaciones](#relaciones)
5. [Reglas de integridad](#reglas-de-integridad)
6. [Datos iniciales y migraciones](#datos-iniciales)

---

<a id="alcance"></a>
## Alcance

**Se almacena:** la infraestructura del campus (campus, zonas abiertas, edificios, pisos, espacios, salidas, escaleras), las zonas seguras, las rutas de evacuación (de texto y exteriores, con el último cálculo), las recomendaciones preventivas y los datos de las imágenes.

**No se almacena, y por qué:**
* **Los archivos de las imágenes:** van en un volumen de disco o en Azure Blob Storage, según la configuración; la tabla `imagen` guarda su ruta ([RN-MS02-35](1_requirements.md#rn-ms02-35)).
* **Los códigos QR:** se generan al vuelo a partir del UUID ([RN-MS02-37](1_requirements.md#rn-ms02-37)).
* **Usuarios y auditoría:** los usuarios son de MS01 y la auditoría de MS08. MS02 no guarda quién hizo cada cambio; lo deja en el log con el Correlation ID.
* **Estado de las personas durante una evacuación:** es de MS04.

---

<a id="convenciones"></a>
## Convenciones

* Nombres de tablas y columnas en `snake_case` y en español.
* Llaves primarias UUID generadas por la base de datos (`gen_random_uuid()`).
* Estados y catálogos guardados como texto (`@Enumerated(EnumType.STRING)`) y validados con un `CHECK`.
* Coordenadas en `NUMERIC(9,6)` (grados WGS84).
* No hay borrado físico, salvo en `imagen` ([RN-MS02-01](1_requirements.md#rn-ms02-01)).
* Una migración ya publicada no se edita: los cambios van en una versión nueva.

---

<a id="tablas"></a>
## Tablas

<a id="campus"></a>
### campus

| Columna | Tipo | Nulo | Por defecto | Restricciones | Descripción |
|---|---|---|---|---|---|
| `id` | UUID | No | `gen_random_uuid()` | PK | |
| `nombre` | VARCHAR(100) | No | — | — | |
| `descripcion` | TEXT | Sí | — | — | |
| `estado` | VARCHAR(20) | No | `'ACTIVO'` | CHECK IN (`ACTIVO`, `INACTIVO`) | |

<a id="zona_abierta"></a>
### zona_abierta

| Columna | Tipo | Nulo | Por defecto | Restricciones | Descripción |
|---|---|---|---|---|---|
| `id` | UUID | No | `gen_random_uuid()` | PK | |
| `campus_id` | UUID | No | — | FK → `campus.id` | |
| `nombre` | VARCHAR(100) | No | — | — | |
| `tipo` | VARCHAR(20) | No | — | CHECK IN (`ZONA_VERDE`, `PLAZOLETA`, `CANCHA`, `PARQUEADERO`, `OTRA`) | |
| `latitud`, `longitud` | NUMERIC(9,6) | Sí | — | CHECK: las dos nulas o las dos en rango | Opcionales. |
| `estado` | VARCHAR(20) | No | `'ACTIVO'` | CHECK IN (`ACTIVO`, `INACTIVO`) | |

**Índices:** `idx_zona_abierta_campus (campus_id)`.

<a id="edificio"></a>
### edificio

| Columna | Tipo | Nulo | Por defecto | Restricciones | Descripción |
|---|---|---|---|---|---|
| `id` | UUID | No | `gen_random_uuid()` | PK | Único identificador del edificio (no hay código). |
| `nombre` | VARCHAR(100) | No | — | — | |
| `descripcion` | TEXT | Sí | — | — | |
| `estado` | VARCHAR(20) | No | `'ACTIVO'` | — | `ACTIVO` / `INACTIVO`. |
| `campus_id` | UUID | No | — | FK → `campus.id` | Obligatorio desde V16 ([RN-MS02-02](1_requirements.md#rn-ms02-02)). |

**Índices:** `idx_edificio_campus (campus_id)`.

<a id="piso"></a>
### piso

| Columna | Tipo | Nulo | Por defecto | Restricciones | Descripción |
|---|---|---|---|---|---|
| `id` | UUID | No | `gen_random_uuid()` | PK; UNIQUE (`id`, `edificio_id`) | La llave compuesta permite que espacios y salidas validen que su piso es de su edificio. |
| `edificio_id` | UUID | No | — | FK → `edificio.id` | |
| `numero` | INTEGER | No | — | UNIQUE (`edificio_id`, `numero`) | Puede ser negativo ([RN-MS02-05](1_requirements.md#rn-ms02-05)). |
| `nombre` | VARCHAR(100) | Sí | — | — | |
| `estado` | VARCHAR(20) | No | `'ACTIVO'` | CHECK IN (`ACTIVO`, `INACTIVO`) | |

<a id="espacio"></a>
### espacio

| Columna | Tipo | Nulo | Por defecto | Restricciones | Descripción |
|---|---|---|---|---|---|
| `id` | UUID | No | `gen_random_uuid()` | PK | Único identificador (no hay código desde V16). |
| `edificio_id` | UUID | No | — | FK → `edificio.id` | |
| `piso_id` | UUID | No | — | FK (`piso_id`, `edificio_id`) → `piso (id, edificio_id)` | El piso es del mismo edificio. |
| `nombre` | VARCHAR(100) | No | — | — | |
| `tipo` | VARCHAR(20) | No | — | CHECK IN (`SALON`, `LABORATORIO`, `AUDITORIO`, `OFICINA`, `ESPACIO_COMUN`) | |
| `estado` | VARCHAR(20) | No | `'ACTIVO'` | CHECK IN (`ACTIVO`, `INACTIVO`) | |

**Índices:** `idx_espacio_piso (piso_id)`.

<a id="salida"></a>
### salida

| Columna | Tipo | Nulo | Por defecto | Restricciones | Descripción |
|---|---|---|---|---|---|
| `id` | UUID | No | `gen_random_uuid()` | PK | Único identificador (no hay código desde V16). |
| `edificio_id` | UUID | No | — | FK → `edificio.id` | |
| `piso_id` | UUID | No | — | FK (`piso_id`, `edificio_id`) → `piso (id, edificio_id)` | Piso desde el que se sale. |
| `nombre` | VARCHAR(100) | No | — | — | |
| `latitud`, `longitud` | NUMERIC(9,6) | No | — | CHECK de rango | Obligatorias ([RN-MS02-29](1_requirements.md#rn-ms02-29)). |
| `estado` | VARCHAR(20) | No | `'ACTIVO'` | CHECK IN (`ACTIVO`, `INACTIVO`) | |

**Índices:** `idx_salida_piso (piso_id)`.

<a id="escalera"></a>
### escalera y escalera_piso

**escalera**

| Columna | Tipo | Nulo | Por defecto | Restricciones | Descripción |
|---|---|---|---|---|---|
| `id` | UUID | No | `gen_random_uuid()` | PK | |
| `edificio_id` | UUID | No | — | FK → `edificio.id` | |
| `codigo` | VARCHAR(30) | No | — | UNIQUE | Lo genera el sistema (`ESC-…`). |
| `nombre` | VARCHAR(100) | No | — | — | |
| `disponibilidad` | VARCHAR(25) | No | `'DISPONIBLE'` | CHECK IN (`DISPONIBLE`, `BLOQUEADA`, `FUERA_DE_SERVICIO`) | |
| `motivo` | VARCHAR(255) | Sí | — | CHECK: obligatorio si no está `DISPONIBLE` | |
| `estado` | VARCHAR(20) | No | `'ACTIVO'` | CHECK IN (`ACTIVO`, `INACTIVO`) | |
| `prioridad` | INTEGER | No | `1` | CHECK ≥ 1 | 1 es la principal del edificio (V17). |

**escalera_piso** (pisos que conecta cada escalera)

| Columna | Tipo | Nulo | Restricciones |
|---|---|---|---|
| `escalera_id` | UUID | No | PK compuesta; FK → `escalera.id` ON DELETE CASCADE |
| `piso_id` | UUID | No | PK compuesta; FK → `piso.id` |

**Índices:** `idx_escalera_edificio (edificio_id)`, `idx_escalera_piso_piso (piso_id)`.

<a id="zona_segura"></a>
### zona_segura

| Columna | Tipo | Nulo | Por defecto | Restricciones | Descripción |
|---|---|---|---|---|---|
| `id` | UUID | No | `gen_random_uuid()` | PK | |
| `codigo` | VARCHAR(50) | No | — | UNIQUE | Fijo ([RN-MS02-09](1_requirements.md#rn-ms02-09)). |
| `nombre` | VARCHAR(100) | No | — | — | |
| `ubicacion` | VARCHAR(255) | No | — | — | Descripción del lugar. |
| `descripcion` | TEXT | No | — | — | |
| `capacidad` | INTEGER | No | — | CHECK > 0 | Aproximada. |
| `estado` | VARCHAR(20) | No | `'ACTIVO'` | — | |
| `latitud`, `longitud` | NUMERIC(9,6) | Sí | — | CHECK: las dos nulas o las dos en rango | Necesarias para ser destino de una ruta exterior. |
| `zona_abierta_id` | UUID | Sí | — | FK → `zona_abierta.id` | |
| `es_punto_encuentro` | BOOLEAN | No | `TRUE` | — | [RN-MS02-12](1_requirements.md#rn-ms02-12). |

**Índices:** `idx_zona_segura_zona_abierta (zona_abierta_id)`.

<a id="ruta_evacuacion"></a>
### ruta_evacuacion

Guarda los dos tipos de ruta ([RN-MS02-13](1_requirements.md#rn-ms02-13)): la de texto (origen en un espacio) y la exterior (origen en una salida, con el último cálculo del proveedor).

| Columna | Tipo | Nulo | Por defecto | Restricciones | Descripción |
|---|---|---|---|---|---|
| `id` | UUID | No | `gen_random_uuid()` | PK | |
| `espacio_origen_id` | UUID | Sí | — | FK → `espacio.id`; UNIQUE | Origen de una ruta de texto. |
| `salida_origen_id` | UUID | Sí | — | FK → `salida.id`; UNIQUE (`salida_origen_id`, `zona_segura_destino_id`) | Origen de una ruta exterior. |
| `zona_segura_destino_id` | UUID | No | — | FK → `zona_segura.id` | |
| `descripcion` | VARCHAR(1000) | No | — | — | Recorrido en texto. En las exteriores lo arma el sistema. |
| `estado` | VARCHAR(20) | No | `'DISPONIBLE'` | CHECK IN (`DISPONIBLE`, `BLOQUEADA`, `RESTRINGIDA`) | |
| `motivo` | VARCHAR(255) | Sí | — | CHECK: nulo si y solo si `DISPONIBLE` | [RN-MS02-14](1_requirements.md#rn-ms02-14). |
| `distancia_metros` | NUMERIC(10,2) | Sí | — | — | Solo rutas exteriores. |
| `duracion_segundos` | NUMERIC(10,2) | Sí | — | — | Solo rutas exteriores. |
| `geometria` | TEXT | Sí | — | — | Puntos del recorrido como `lat,lon` separados por `;`. |
| `pasos` | TEXT | Sí | — | — | Una indicación por línea, con su distancia. |
| `calculada_en` | TIMESTAMPTZ | Sí | — | — | Último cálculo. |
| `created_at`, `updated_at` | TIMESTAMPTZ | No | — | — | |

**Restricciones:** `ck_ruta_un_origen`: exactamente uno de `espacio_origen_id` y `salida_origen_id`.
**Índices:** `idx_ruta_zona_destino (zona_segura_destino_id)`, `idx_ruta_salida_origen (salida_origen_id)`.

<a id="recomendacion_preventiva"></a>
### recomendacion_preventiva

| Columna | Tipo | Nulo | Por defecto | Restricciones | Descripción |
|---|---|---|---|---|---|
| `id` | UUID | No | `gen_random_uuid()` | PK | |
| `ambito` | VARCHAR(20) | No | — | CHECK IN (`EDIFICIO`, `TIPO_ESPACIO`) | |
| `edificio_id` | UUID | Sí | — | FK → `edificio.id` | Solo con ámbito `EDIFICIO`. |
| `tipo_espacio` | VARCHAR(20) | Sí | — | CHECK del catálogo de tipos | Solo con ámbito `TIPO_ESPACIO`. |
| `titulo` | VARCHAR(150) | No | — | — | |
| `recomendacion` | TEXT | No | — | — | |
| `estado` | VARCHAR(20) | No | `'ACTIVO'` | CHECK IN (`ACTIVO`, `INACTIVO`) | |
| `created_at`, `updated_at` | TIMESTAMP | No | `CURRENT_TIMESTAMP` | — | |

**Restricciones:** `ck_recomendacion_ambito_datos`: con `EDIFICIO` hay edificio y no tipo; con `TIPO_ESPACIO`, al revés ([RN-MS02-23](1_requirements.md#rn-ms02-23)).
**Índices:** `idx_recomendacion_edificio`, `idx_recomendacion_tipo_espacio`.

<a id="imagen"></a>
### imagen

| Columna | Tipo | Nulo | Por defecto | Restricciones | Descripción |
|---|---|---|---|---|---|
| `id` | UUID | No | `gen_random_uuid()` | PK | |
| `tipo_elemento` | VARCHAR(30) | No | — | CHECK IN (`EDIFICIO`, `ESPACIO`, `SALIDA`, `ZONA_SEGURA`) | |
| `elemento_id` | UUID | No | — | — | Id del elemento; no es FK porque apunta a tablas distintas según el tipo. |
| `nombre_original` | VARCHAR(255) | No | — | — | |
| `tipo_contenido` | VARCHAR(50) | No | — | CHECK IN (`image/jpeg`, `image/png`, `image/webp`) | |
| `tamanio_bytes` | BIGINT | No | — | CHECK > 0 | Máximo 5 MB, validado en la aplicación. |
| `ruta_archivo` | VARCHAR(500) | No | — | — | Ruta dentro del volumen de imágenes o nombre del blob en el contenedor de Azure (es la misma en ambos). |
| `created_at` | TIMESTAMP | No | `CURRENT_TIMESTAMP` | — | |

**Índices:** `idx_imagen_elemento (tipo_elemento, elemento_id)`.

---

<a id="relaciones"></a>
## Relaciones

| Relación | Cardinalidad | Cómo se garantiza |
|---|---|---|
| campus → edificio | 1 a N | FK `edificio.campus_id` NOT NULL |
| campus → zona_abierta | 1 a N | FK `zona_abierta.campus_id` |
| zona_abierta → zona_segura | 1 a N (opcional) | FK `zona_segura.zona_abierta_id` |
| edificio → piso | 1 a N | FK `piso.edificio_id` |
| piso → espacio, piso → salida | 1 a N | FK compuesta (`piso_id`, `edificio_id`): el piso es del mismo edificio |
| escalera ↔ piso | N a N | `escalera_piso` |
| espacio → ruta_evacuacion | 1 a 0..1 | UNIQUE `espacio_origen_id` |
| salida → ruta_evacuacion | 1 a N (una por zona) | UNIQUE (`salida_origen_id`, `zona_segura_destino_id`) |
| zona_segura → ruta_evacuacion | 1 a N | FK `zona_segura_destino_id` |
| edificio → recomendacion_preventiva | 1 a N | FK `edificio_id` |
| elemento → imagen | 1 a N | `tipo_elemento` + `elemento_id` (sin FK) |

El diagrama está en [`diagrams/ms02-er-physical.puml`](diagrams/ms02-er-physical.puml).

---

<a id="reglas-de-integridad"></a>
## Reglas de integridad

Las reglas que dependen del estado (no inactivar con dependientes activos, no registrar en contenedores inactivos, rutas activas) se validan en la aplicación y responden 422 ([`1_requirements.md`](1_requirements.md)). La base de datos garantiza las que no dependen del estado:

* Unicidad: número de piso por edificio, código de zona segura, código de escalera, una ruta por espacio y una por par salida–zona.
* Pertenencia: espacios y salidas en un piso de su mismo edificio (FK compuesta).
* Catálogos y coordenadas válidas (`CHECK`).
* Motivo obligatorio en rutas no disponibles y escaleras no disponibles.
* Un solo origen por ruta.

---

<a id="datos-iniciales"></a>
## Datos iniciales y migraciones

* V10 crea el campus «Campus principal» y le asigna los edificios existentes. Un despliegue nuevo empieza solo con ese campus.
* V16 asigna «Campus principal» a los edificios sin campus, vuelve obligatorio `edificio.campus_id` y elimina la columna `codigo` de `edificio`, `espacio` y `salida`.

| Versión | Cambio |
|---|---|
| V1 | `edificio` y `zona_segura` |
| V2 | `ruta_evacuacion` |
| V3 | `piso` y `espacio` |
| V4 | FK de la ruta al espacio de origen |
| V5 | Tipo de espacio `ESPACIO_COMUN` |
| V6 | `motivo` de la ruta |
| V7 | `codigo` de la zona segura |
| V8 | Se quita la ruta alternativa |
| V9 | `recomendacion_preventiva` |
| V10 | `campus` y `zona_abierta`; `edificio.campus_id` |
| V11 | `escalera` y `escalera_piso` |
| V12 | `salida` |
| V13 | Coordenadas, zona abierta y punto de encuentro en `zona_segura` |
| V14 | `imagen` |
| V15 | Rutas exteriores en `ruta_evacuacion` |
| V16 | Sin código en edificio, espacio y salida; campus obligatorio |
| V17 | `prioridad` de la escalera |
