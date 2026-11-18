# Diccionario de Datos, Formularios y Entradas

## Índice de Contenido
1. [Reglas generales de entrada](#reglas-generales-de-entrada)
2. [Formularios y Entradas](#formularios-y-entradas)
   - [F-01 · Campus](#f-01)
   - [F-02 · Zona abierta](#f-02)
   - [F-03 · Edificio](#f-03)
   - [F-04 · Piso](#f-04)
   - [F-05 · Espacio](#f-05)
   - [F-06 · Salida](#f-06)
   - [F-07 · Escalera](#f-07)
   - [F-08 · Zona segura](#f-08)
   - [F-09 · Ruta de evacuación](#f-09)
   - [F-10 · Ruta exterior](#f-10)
   - [F-11 · Recomendación preventiva](#f-11)
   - [F-12 · Imagen](#f-12)
   - [F-13 · Filtros de los listados](#f-13)
3. [Pantallas sin formulario](#pantallas-sin-formulario)

---

<a id="reglas-generales-de-entrada"></a>
## Reglas generales de entrada

* **RG-01**: Todo texto se recorta (trim) de espacios al inicio y al final antes de guardar.
* **RG-02**: Un campo obligatorio vacío o con solo espacios se considera faltante.
* **RG-03**: Todos los ids (de la ruta o del cuerpo) son UUID. Un valor con formato inválido se rechaza con HTTP 400 `VALIDACION`.
* **RG-04**: Los errores de validación de campos son HTTP 400 `VALIDACION`, con el `ErrorResponse` de los Estándares y el campo y su mensaje en `errores`. Los errores de reglas de negocio son HTTP 422 `REGLA_NEGOCIO`, con el mensaje en `mensaje`.
* **RG-05**: Los valores de catálogo (tipos, estados, ámbitos) se envían en MAYÚSCULAS, tal como aparecen en cada tabla. Un valor fuera del catálogo es 400.
* **RG-06**: Latitud y longitud son decimales en grados (WGS84), con hasta 6 decimales. La latitud va de -90 a 90 y la longitud de -180 a 180. Donde son opcionales, se envían juntas o no se envían; si llega solo una, la respuesta es 422 «La latitud y la longitud deben enviarse juntas».
* **RG-07**: El estado (`ACTIVO`/`INACTIVO`) nunca se envía en un formulario: lo asigna el sistema al crear y cambia con «Inactivar» y «Reactivar».
* **RG-08**: La identidad del usuario nunca viaja en el cuerpo: llega del Gateway en `X-User-Id`, `X-User-Role` y `X-User-Permission`.

---

<a id="formularios-y-entradas"></a>
## Formularios y Entradas

<a id="f-01"></a>
### F-01 · Campus
Referencia: **RF-MS02-10** · `POST /campuses`, `PUT /campuses/{id}`

| Campo | Tipo | Obligatorio | Longitud / formato | Reglas de validación | Mensaje de error | Ejemplo válido |
|---|---|---|---|---|---|---|
| Nombre (`nombre`) | Texto | Sí | Máx. 100 | — | "El nombre es obligatorio" / "El nombre no puede superar 100 caracteres" | Campus principal |
| Descripción (`descripcion`) | Texto largo | No | — | — | — | Sede norte, Autopista Norte km 13 |

<a id="f-02"></a>
### F-02 · Zona abierta
Referencia: **RF-MS02-10** · `POST /campuses/{campusId}/open-zones`, `PUT /open-zones/{id}`

| Campo | Tipo | Obligatorio | Longitud / formato | Reglas de validación | Mensaje de error | Ejemplo válido |
|---|---|---|---|---|---|---|
| Nombre (`nombre`) | Texto | Sí | Máx. 100 | — | "El nombre es obligatorio" / "El nombre no puede superar 100 caracteres" | Plazoleta central |
| Tipo (`tipo`) | Selección | Sí | `ZONA_VERDE` / `PLAZOLETA` / `CANCHA` / `PARQUEADERO` / `OTRA` | — | "El tipo es obligatorio" | PLAZOLETA |
| Latitud (`latitud`) | Decimal | No | -90 a 90 | Junto con la longitud (RG-06) | "La latitud debe estar entre -90 y 90" | 4.782710 |
| Longitud (`longitud`) | Decimal | No | -180 a 180 | Junto con la latitud (RG-06) | "La longitud debe estar entre -180 y 180" | -74.042630 |

**Mensajes globales:** campus inactivo (422): "No se pueden registrar zonas abiertas en un campus inactivo".

<a id="f-03"></a>
### F-03 · Edificio
Referencia: **RF-MS02-01** · `POST /buildings`, `PUT /buildings/{id}`

| Campo | Tipo | Obligatorio | Longitud / formato | Reglas de validación | Mensaje de error | Ejemplo válido |
|---|---|---|---|---|---|---|
| Nombre (`nombre`) | Texto | Sí | Máx. 100 | — | "El nombre es obligatorio" / "El nombre no puede superar 100 caracteres" | Bloque G |
| Campus (`campusId`) | Selección (campus activos) | Sí al crear; al modificar, si no se envía se conserva el actual | UUID | Debe existir (404) y estar activo (422) | "El campus es obligatorio" / "No se pueden asignar edificios a un campus inactivo" | Campus principal |
| Descripción (`descripcion`) | Texto largo | No | — | — | — | Laboratorios y aulas de ingeniería |

El edificio no tiene código: se identifica por su UUID.

<a id="f-04"></a>
### F-04 · Piso
Referencia: **RF-MS02-02** · `POST /buildings/{edificioId}/floors`, `PUT /floors/{id}`

| Campo | Tipo | Obligatorio | Longitud / formato | Reglas de validación | Mensaje de error | Ejemplo válido |
|---|---|---|---|---|---|---|
| Número (`numero`) | Entero | Sí | Puede ser negativo (sótanos) | Único en el edificio, incluso entre pisos inactivos (409, campo `numero`) | "El número del piso es obligatorio" / "Ya existe un piso con ese número en el edificio." | 2 |
| Nombre (`nombre`) | Texto | No | Máx. 100 | — | "El nombre no puede superar 100 caracteres" | Segundo piso |

**Mensajes globales:** edificio inactivo (422): "No se pueden registrar pisos en un edificio inactivo".

<a id="f-05"></a>
### F-05 · Espacio
Referencia: **RF-MS02-03** · `POST /floors/{pisoId}/spaces`, `PUT /spaces/{id}`

| Campo | Tipo | Obligatorio | Longitud / formato | Reglas de validación | Mensaje de error | Ejemplo válido |
|---|---|---|---|---|---|---|
| Nombre (`nombre`) | Texto | Sí | Máx. 100 | — | "El nombre es obligatorio" / "El nombre no puede superar 100 caracteres" | Salón 201 |
| Tipo (`tipo`) | Selección | Sí | `SALON` / `LABORATORIO` / `AUDITORIO` / `OFICINA` / `ESPACIO_COMUN` | — | "El tipo de espacio es obligatorio" | SALON |

El piso se elige en la ruta (`{pisoId}`) y no cambia al modificar. El espacio no tiene código.

**Mensajes globales:** "No se pueden registrar espacios en un piso inactivo" (422) · "No se pueden registrar espacios en un edificio inactivo" (422).

<a id="f-06"></a>
### F-06 · Salida
Referencia: **RF-MS02-11** · `POST /buildings/{edificioId}/exits`, `PUT /exits/{id}`

| Campo | Tipo | Obligatorio | Longitud / formato | Reglas de validación | Mensaje de error | Ejemplo válido |
|---|---|---|---|---|---|---|
| Piso (`pisoId`) | Selección (pisos activos del edificio) | Sí | UUID | Debe ser del edificio (422) y estar activo (422) | "El piso es obligatorio" / "No se puede asociar una salida a un piso inactivo" | Piso 1 |
| Nombre (`nombre`) | Texto | Sí | Máx. 100 | — | "El nombre es obligatorio" / "El nombre no puede superar 100 caracteres" | Salida principal |
| Latitud (`latitud`) | Decimal | Sí | -90 a 90 | — | "La latitud es obligatoria" / "La latitud debe estar entre -90 y 90" | 4.782710 |
| Longitud (`longitud`) | Decimal | Sí | -180 a 180 | — | "La longitud es obligatoria" / "La longitud debe estar entre -180 y 180" | -74.042630 |

La salida no tiene código. Al guardarla se generan sus rutas exteriores (RF-MS02-13).

<a id="f-07"></a>
### F-07 · Escalera
Referencia: **RF-MS02-12**

**Registro** · `POST /buildings/{edificioId}/stairs`

| Campo | Tipo | Obligatorio | Longitud / formato | Reglas de validación | Mensaje de error | Ejemplo válido |
|---|---|---|---|---|---|---|
| Nombre (`nombre`) | Texto | Sí | Máx. 100 | — | "El nombre de la escalera es obligatorio" / "El nombre no puede superar 100 caracteres" | Escalera norte |
| Pisos (`pisoIds`) | Selección múltiple (pisos activos del edificio) | Sí | Al menos 2 UUID, sin repetir | Todos del mismo edificio y activos (422) | "Debe seleccionar los pisos que conecta la escalera" / "Una escalera debe conectar al menos dos pisos" / "La lista de pisos contiene identificadores duplicados" / "No se pueden conectar pisos inactivos" | Pisos 1, 2 y 3 |
| Prioridad (`prioridad`) | Entero | No | ≥ 1 | Si no se envía, queda después de las existentes | "La prioridad debe ser 1 o mayor" | 1 |

El código (`ESC-…`) lo genera el sistema.

**Disponibilidad** · `PATCH /stairs/{id}/status`

| Campo | Tipo | Obligatorio | Longitud / formato | Reglas de validación | Mensaje de error | Ejemplo válido |
|---|---|---|---|---|---|---|
| Disponibilidad (`disponibilidad`) | Selección | Sí | `DISPONIBLE` / `BLOQUEADA` / `FUERA_DE_SERVICIO` | — | "La disponibilidad es obligatoria" | BLOQUEADA |
| Motivo (`motivo`) | Texto | Si no es `DISPONIBLE` | Máx. 255 | Obligatorio para bloquear o poner fuera de servicio (422) | "Debe indicar el motivo para bloquear o poner fuera de servicio la escalera" | Humo en el tercer piso |

**Prioridad** · `PATCH /stairs/{id}/priority`: `prioridad` entero obligatorio ≥ 1 ("La prioridad es obligatoria" / "La prioridad debe ser 1 o mayor").

<a id="f-08"></a>
### F-08 · Zona segura
Referencia: **RF-MS02-04** · `POST /safe-zones`, `PUT /safe-zones/{id}`

| Campo | Tipo | Obligatorio | Longitud / formato | Reglas de validación | Mensaje de error | Ejemplo válido |
|---|---|---|---|---|---|---|
| Código (`codigo`) | Texto | Sí, solo al crear | Máx. 50 | Único, incluso entre zonas inactivas (409, campo `codigo`). No se modifica | "El código es obligatorio" / "El código no puede superar 50 caracteres" / "Ya existe una zona segura con ese código." | MP-H-01 |
| Nombre (`nombre`) | Texto | Sí | Máx. 100 | — | "El nombre es obligatorio" / "El nombre no puede superar 100 caracteres" | Plazoleta H |
| Ubicación (`ubicacion`) | Texto | Sí | Máx. 255 | — | "La ubicación es obligatoria" / "La ubicación no puede superar 255 caracteres" | Frente al bloque H |
| Descripción (`descripcion`) | Texto largo | Sí | — | — | "La descripción es obligatoria" | Punto de encuentro de los bloques G y H |
| Capacidad (`capacidad`) | Entero | Sí | > 0 | — | "La capacidad es obligatoria" / "La capacidad debe ser mayor que 0" | 300 |
| Latitud y longitud | Decimal | No | RG-06 | Sin coordenadas la zona no puede ser destino de una ruta exterior | Ver RG-06 | 4.783100, -74.043100 |
| Zona abierta (`zonaAbiertaId`) | Selección (zonas abiertas activas) | No | UUID | Debe estar activa (422) | "No se pueden ubicar zonas seguras en una zona abierta inactiva" | Plazoleta central |
| Punto de encuentro (`puntoEncuentro`) | Sí/No | No | — | `true` por defecto | — | true |

Al modificar, coordenadas, zona abierta y punto de encuentro solo cambian si se envían.

<a id="f-09"></a>
### F-09 · Ruta de evacuación
Referencia: **RF-MS02-05**

**Crear** · `POST /evacuation-routes`

| Campo | Tipo | Obligatorio | Longitud / formato | Reglas de validación | Mensaje de error | Ejemplo válido |
|---|---|---|---|---|---|---|
| Espacio de origen (`espacioOrigenId`) | Selección (espacios disponibles sin ruta) | Sí | UUID | Disponible (422); sin otra ruta (409) | "El espacio de origen es obligatorio" | Salón 201 |
| Zona segura de destino (`zonaSeguraDestinoId`) | Selección (zonas activas) | Sí | UUID | Activa (422) | "La zona segura de destino es obligatoria" | Plazoleta H |
| Descripción (`descripcion`) | Texto largo | Sí | Máx. 1000 | — | "La descripción de la ruta es obligatoria" / "La descripción no puede superar 1000 caracteres" | Baje por la escalera norte y salga por la puerta principal. |
| Estado (`estado`) | Selección | Sí | `DISPONIBLE` / `BLOQUEADA` / `RESTRINGIDA` | — | "El estado es obligatorio" | DISPONIBLE |
| Motivo (`motivo`) | Texto | Si no es `DISPONIBLE` | Máx. 255 | 422 si falta | "El motivo es obligatorio cuando la ruta queda bloqueada o restringida" | Obra en el corredor |

**Actualizar** · `PUT /evacuation-routes/{id}`: `zonaSeguraDestinoId` y `descripcion`, con las mismas reglas. No aplica a rutas exteriores (422).

**Cambiar estado** · `PATCH /evacuation-routes/{id}/status`: `estado` y `motivo`, con las mismas reglas. Al pasar a `DISPONIBLE` el motivo se borra.

<a id="f-10"></a>
### F-10 · Ruta exterior
Referencia: **RF-MS02-13**

**Calcular un par** · `POST /evacuation-routes/exterior`

| Campo | Tipo | Obligatorio | Longitud / formato | Reglas de validación | Mensaje de error | Ejemplo válido |
|---|---|---|---|---|---|---|
| Salida (`salidaId`) | Selección (salidas disponibles) | Sí | UUID | Disponible (422) | "La salida de origen es obligatoria" | Salida principal |
| Zona segura (`zonaSeguraId`) | Selección (zonas activas con coordenadas) | Sí | UUID | Activa y con coordenadas (422) | "La zona segura de destino es obligatoria" | Plazoleta H |

**Generar las faltantes** · `POST /evacuation-routes/exterior/generate?recalcular=false`: sin cuerpo. `recalcular=true` rehace también las existentes. Responde `{ "generadas", "omitidas", "interrumpida" }`.

<a id="f-11"></a>
### F-11 · Recomendación preventiva
Referencia: **RF-MS02-08** · `POST /preventive-recommendations`, `PUT /preventive-recommendations/{id}`

| Campo | Tipo | Obligatorio | Longitud / formato | Reglas de validación | Mensaje de error | Ejemplo válido |
|---|---|---|---|---|---|---|
| Ámbito (`ambito`) | Selección | Sí | `EDIFICIO` / `TIPO_ESPACIO` | — | "El ámbito es obligatorio" | TIPO_ESPACIO |
| Edificio (`edificioId`) | Selección | Solo con `EDIFICIO` | UUID | No se envía con `TIPO_ESPACIO` (400) | "Para el ámbito EDIFICIO se requiere edificioId y no tipoEspacio" | Bloque G |
| Tipo de espacio (`tipoEspacio`) | Selección | Solo con `TIPO_ESPACIO` | Catálogo de F-05 | No se envía con `EDIFICIO` (400) | — | LABORATORIO |
| Título (`titulo`) | Texto | Sí | Máx. 150 | — | "El título es obligatorio" / "El título no puede superar 150 caracteres" | Cierre de gases |
| Recomendación (`recomendacion`) | Texto largo | Sí | — | — | "La recomendación es obligatoria" | Cierre las llaves de gas antes de salir. |

<a id="f-12"></a>
### F-12 · Imagen
Referencia: **RF-MS02-14** · `POST /images` (`multipart/form-data`)

| Campo | Tipo | Obligatorio | Longitud / formato | Reglas de validación | Mensaje de error | Ejemplo válido |
|---|---|---|---|---|---|---|
| Tipo de elemento (`tipoElemento`) | Selección | Sí | `EDIFICIO` / `ESPACIO` / `SALIDA` / `ZONA_SEGURA` | — | "El tipo de elemento es obligatorio." | SALIDA |
| Elemento (`elementoId`) | UUID | Sí | — | Debe existir y estar activo (422) | "El identificador del elemento es obligatorio." / "No se pueden asociar imágenes a un elemento inactivo." | — |
| Archivo (`archivo`) | Archivo | Sí | JPEG, PNG o WEBP; máx. 5 MB; nombre máx. 255 | 422 si no cumple | "Formato de imagen no permitido. Use JPEG, PNG o WEBP." / "La imagen no puede estar vacía." / "La imagen supera el tamaño máximo permitido de 5242880 bytes." | salida-principal.jpg |

<a id="f-13"></a>
### F-13 · Filtros de los listados
Referencia: **RF-MS02-09**

| Parámetro | Dónde | Valores | Por defecto | Efecto |
|---|---|---|---|---|
| `incluirInactivos` | Listados de campus, zonas abiertas, edificios, pisos, espacios, salidas y zonas seguras | `true` / `false` | `false` | `false` devuelve solo los activos (espacios y salidas: solo los disponibles). |
| `puntoEncuentro` | `GET /safe-zones` | `true` / `false` | — | Filtra por la marca de punto de encuentro. |
| `sinCoordenadas` | `GET /safe-zones` | `true` | — | Lista las zonas que aún no tienen coordenadas. |
| `tipoElemento`, `elementoId` | `GET /images` | Como en F-12 | — | Imágenes de un elemento. |

---

<a id="pantallas-sin-formulario"></a>
## Pantallas sin formulario

* **Consulta de rutas** (RF-MS02-06): el usuario elige edificio y espacio en listas; no hay campos de texto.
* **Inactivar y reactivar** (RF-MS02-07): botones con confirmación. Si el sistema responde 422, la pantalla muestra el `mensaje` tal cual (por ejemplo, las rutas que deben bloquearse antes).
* **Códigos QR** (RF-MS02-15): botón que descarga el PNG.
* **Consulta pública por QR** (RF-MS02-16): la abre el QR; el id viene en el enlace (`#/qr/{espacios|salidas|zonas}/{id}`) y no hay formulario.
