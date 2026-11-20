# Requerimientos — MS02 Campus y seguridad

Los valores transversales (cabeceras, errores, correlación, llamadas internas) se rigen por los [Estándares de Integración](../integration/standards.md) y los permisos por la [matriz de roles y permisos](../integration/permissions-matrix.md). Este documento solo define el comportamiento de MS02. Los endpoints, con el permiso de cada uno, están en [`api/openapi.yml`](api/openapi.yml).

## Índice de Contenido
1. [Alcance](#alcance)
2. [Conceptos y estados](#conceptos-y-estados)
3. [Requerimientos Funcionales](#requerimientos-funcionales)
   - [RF-MS02-01 Gestionar edificios](#rf-ms02-01)
   - [RF-MS02-02 Gestionar pisos](#rf-ms02-02)
   - [RF-MS02-03 Gestionar espacios](#rf-ms02-03)
   - [RF-MS02-04 Gestionar zonas seguras y puntos de encuentro](#rf-ms02-04)
   - [RF-MS02-05 Gestionar rutas de evacuación](#rf-ms02-05)
   - [RF-MS02-06 Consultar la ruta de evacuación](#rf-ms02-06)
   - [RF-MS02-07 Reactivar elementos del campus](#rf-ms02-07)
   - [RF-MS02-08 Gestionar información preventiva](#rf-ms02-08)
   - [RF-MS02-09 Consultar el catálogo del campus](#rf-ms02-09)
   - [RF-MS02-10 Gestionar campus y zonas abiertas](#rf-ms02-10)
   - [RF-MS02-11 Gestionar salidas](#rf-ms02-11)
   - [RF-MS02-12 Gestionar escaleras](#rf-ms02-12)
   - [RF-MS02-13 Calcular y generar rutas exteriores](#rf-ms02-13)
   - [RF-MS02-14 Gestionar imágenes](#rf-ms02-14)
   - [RF-MS02-15 Generar códigos QR](#rf-ms02-15)
   - [RF-MS02-16 Consultar información por QR sin sesión](#rf-ms02-16)
4. [Requerimientos No Funcionales (RNF)](#requerimientos-no-funcionales-rnf)

---

<a id="alcance"></a>
## Alcance

MS02 administra la infraestructura física del campus y la información de evacuación: campus, zonas abiertas, edificios, pisos, espacios, salidas, escaleras, zonas seguras (puntos de encuentro), rutas de evacuación, recomendaciones preventivas, imágenes y códigos QR.

La evacuación tiene dos partes independientes:

1. **Dentro del edificio:** desde el piso del espacio hasta el piso de una salida, por escaleras disponibles. No hay coordenadas interiores ni rutas salón → escalera.
2. **Fuera del edificio:** desde la salida hasta una zona segura, a pie, calculada con OpenStreetMap ([RF-MS02-13](#rf-ms02-13)).

**Fuera del alcance:** el reporte del estado de las personas y la consolidación por punto de encuentro (MS04), las alertas (MS03), las brigadas e incidentes (MS05), los simulacros (MS06), los contenidos de capacitación (MS07) y la auditoría (MS08).

**Dependencias:** MS02 no llama a otros microservicios del proyecto. Su única dependencia externa es el servicio de ruteo peatonal de OpenStreetMap (OSRM), que solo se usa al generar rutas exteriores ([RNF-MS02-06](#requerimientos-no-funcionales-rnf)).

---

<a id="conceptos-y-estados"></a>
## Conceptos y estados

| Concepto | Definición |
|---|---|
| **Estado** (`ACTIVO` / `INACTIVO`) | Lo tienen campus, zonas abiertas, edificios, pisos, espacios, salidas, escaleras, zonas seguras y recomendaciones. Se asigna `ACTIVO` al crear. Las entidades que cuentan con operaciones de reactivación pueden cambiar entre `ACTIVO` e `INACTIVO`; las recomendaciones preventivas solo cuentan con operación de inactivación. Nada se elimina, salvo las imágenes ([RN-MS02-01](#rn-ms02-01)). |
| **Disponible** | Un espacio está disponible si él, su piso y su edificio están activos. Una salida está disponible si ella, su piso y su edificio están activos. Las respuestas lo exponen como `disponible`. |
| **Estado de una ruta** (`DISPONIBLE` / `BLOQUEADA` / `RESTRINGIDA`) | Una ruta es **activa** si está `DISPONIBLE` o `RESTRINGIDA`. Una ruta `BLOQUEADA` no impide inactivar nada. |
| **Disponibilidad de una escalera** (`DISPONIBLE` / `BLOQUEADA` / `FUERA_DE_SERVICIO`) | Es independiente de su estado. Solo una escalera `ACTIVO` y `DISPONIBLE` se recomienda en la guía. |
| **Punto de encuentro** | No es una entidad aparte: es una zona segura con `puntoEncuentro = true` ([RN-MS02-12](#rn-ms02-12)). |
| **Identificadores** | Todos son UUID y no cambian. Edificios, espacios y salidas no tienen código. Las zonas seguras tienen un código fijo que escribe el administrador (p. ej. `MP-H-01`) y las escaleras uno que genera el sistema (`ESC-…`). |

---

<a id="requerimientos-funcionales"></a>
## Requerimientos Funcionales

<a id="rf-ms02-01"></a>
### RF-MS02-01 Gestionar edificios

| Atributo | Detalle |
|---|---|
| **ID** | RF-MS02-01 |
| **Nombre** | Gestionar edificios |
| **Épica / Grupo funcional** | MS02 - Infraestructura |
| **Descripción** | Permitir al administrador crear, modificar e inactivar los edificios de un campus. |
| **Actor(es)** | ADMIN (`ms02:building:write`) |
| **Precondiciones** | Usuario autenticado con el permiso. Para crear, existe al menos un campus activo. Para modificar o inactivar, el edificio existe. |
| **Flujo base** | **Crear:** <ol><li>El administrador envía nombre, campus y, opcionalmente, descripción.</li><li>El sistema valida los campos ([F-03](2_forms-inputs.md#f-03)) y que el campus exista y esté activo.</li><li>Guarda el edificio `ACTIVO` y responde 201.</li></ol>**Modificar:** cambia nombre, descripción y campus. Si no se envía campus, conserva el actual.<br>**Inactivar:** <ol><li>El sistema verifica que ningún espacio del edificio sea origen de una ruta activa.</li><li>Cambia el estado a `INACTIVO` sin tocar sus pisos ni espacios.</li></ol> |
| **Flujos alternos / excepciones** | <ul><li>**A1, datos inválidos o sin campus:** 400 `VALIDACION`.</li><li>**A2, campus inexistente:** 404 `NO_ENCONTRADO`.</li><li>**A3, campus inactivo:** 422 `REGLA_NEGOCIO` («No se pueden asignar edificios a un campus inactivo»).</li><li>**A4, el edificio tiene rutas activas al inactivar:** 422 con los ids de las rutas, que deben bloquearse antes.</li></ul> |
| **Reglas De Negocio** | <ul><li><a id="rn-ms02-01"></a>**RN-MS02-01:** Los elementos del campus no se eliminan, solo se inactivan. La única excepción son las imágenes ([RF-MS02-14](#rf-ms02-14)).</li><li><a id="rn-ms02-02"></a>**RN-MS02-02:** Todo edificio pertenece a un campus. El campus es obligatorio al crear.</li><li><a id="rn-ms02-03"></a>**RN-MS02-03:** Un edificio no se inactiva mientras alguno de sus espacios sea origen de una ruta activa.</li><li><a id="rn-ms02-04"></a>**RN-MS02-04:** Inactivar un elemento no inactiva lo que contiene. Mientras un edificio esté inactivo no se le registran pisos, espacios, salidas ni escaleras, y no aparece en la consulta de rutas ([RF-MS02-06](#rf-ms02-06)). Lo que ya tiene se puede seguir modificando e inactivando.</li></ul> |
| **Postcondiciones** | El edificio queda creado, modificado o inactivo, sin perder sus datos. |
| **Requerimientos asociados** | [RF-MS02-02](#rf-ms02-02), [RF-MS02-07](#rf-ms02-07), [RF-MS02-10](#rf-ms02-10) |
| **Criterios de aceptación** | <ul><li>Crear con nombre y campus activo devuelve 201 con el edificio `ACTIVO`.</li><li>Crear sin campus devuelve 400 con el campo `campusId`.</li><li>Inactivar un edificio con un espacio origen de una ruta `DISPONIBLE` devuelve 422 y no cambia su estado.</li><li>Inactivar un edificio no cambia el estado de sus pisos ni espacios.</li></ul> |
| **Prioridad** | Alta |
| **Estado / Squad responsable** | Implementado / Gaia |

<br>

<a id="rf-ms02-02"></a>
### RF-MS02-02 Gestionar pisos

| Atributo | Detalle |
|---|---|
| **ID** | RF-MS02-02 |
| **Nombre** | Gestionar pisos |
| **Épica / Grupo funcional** | MS02 - Infraestructura |
| **Descripción** | Permitir al administrador registrar, modificar e inactivar los pisos de un edificio. |
| **Actor(es)** | ADMIN (`ms02:building:write`) |
| **Precondiciones** | El edificio existe. Para registrar, el edificio está activo. |
| **Flujo base** | **Registrar:** el administrador envía número y, opcionalmente, nombre. El sistema valida que el número no exista en el edificio y guarda el piso `ACTIVO`.<br>**Modificar:** cambia número y nombre.<br>**Inactivar:** el sistema verifica que el piso no tenga espacios, escaleras ni salidas activas y lo inactiva. |
| **Flujos alternos / excepciones** | <ul><li>**A1, número repetido en el edificio:** 409 `CONFLICTO` con el campo `numero`.</li><li>**A2, edificio inactivo al registrar:** 422.</li><li>**A3, el piso tiene espacios, escaleras o salidas activas:** 422 con sus nombres (o códigos, para las escaleras), que deben inactivarse antes.</li></ul> |
| **Reglas De Negocio** | <ul><li><a id="rn-ms02-05"></a>**RN-MS02-05:** El número del piso es único dentro de su edificio, incluso entre pisos inactivos. Puede ser negativo (sótanos). El nombre es opcional.</li><li><a id="rn-ms02-06"></a>**RN-MS02-06:** Un piso con espacios, escaleras o salidas activas no se inactiva.</li></ul> |
| **Postcondiciones** | El piso queda registrado, modificado o inactivo. |
| **Requerimientos asociados** | [RF-MS02-01](#rf-ms02-01), [RF-MS02-03](#rf-ms02-03), [RF-MS02-11](#rf-ms02-11), [RF-MS02-12](#rf-ms02-12) |
| **Criterios de aceptación** | <ul><li>Registrar el piso 2 dos veces en el mismo edificio devuelve 409 la segunda vez.</li><li>Inactivar un piso con una salida activa devuelve 422 y nombra la salida.</li><li>Los pisos se listan ordenados por número.</li></ul> |
| **Prioridad** | Alta |
| **Estado / Squad responsable** | Implementado / Gaia |

<br>

<a id="rf-ms02-03"></a>
### RF-MS02-03 Gestionar espacios

| Atributo | Detalle |
|---|---|
| **ID** | RF-MS02-03 |
| **Nombre** | Gestionar espacios |
| **Épica / Grupo funcional** | MS02 - Infraestructura |
| **Descripción** | Permitir al administrador registrar, modificar e inactivar los espacios de un piso: salones, laboratorios, auditorios, oficinas y espacios comunes. |
| **Actor(es)** | ADMIN (`ms02:space:write`) |
| **Precondiciones** | El piso existe. Para registrar, el piso y su edificio están activos. |
| **Flujo base** | **Registrar:** el administrador elige el piso y envía nombre y tipo. El sistema guarda el espacio `ACTIVO` en ese piso y en su edificio.<br>**Modificar:** cambia nombre y tipo. El piso no cambia.<br>**Inactivar:** el sistema verifica que el espacio no sea origen de una ruta activa y lo inactiva. |
| **Flujos alternos / excepciones** | <ul><li>**A1, datos inválidos o tipo fuera del catálogo:** 400.</li><li>**A2, piso o edificio inactivo al registrar:** 422.</li><li>**A3, el espacio es origen de una ruta activa:** 422.</li></ul> |
| **Reglas De Negocio** | <ul><li><a id="rn-ms02-07"></a>**RN-MS02-07:** Tipos permitidos: `SALON`, `LABORATORIO`, `AUDITORIO`, `OFICINA`, `ESPACIO_COMUN`.</li><li><a id="rn-ms02-08"></a>**RN-MS02-08:** Un espacio está disponible cuando él, su piso y su edificio están activos. Solo un espacio disponible puede ser origen de una ruta activa y aparecer en la consulta de rutas.</li></ul> |
| **Postcondiciones** | El espacio queda registrado, modificado o inactivo. |
| **Requerimientos asociados** | [RF-MS02-02](#rf-ms02-02), [RF-MS02-05](#rf-ms02-05), [RF-MS02-06](#rf-ms02-06) |
| **Criterios de aceptación** | <ul><li>Registrar un espacio en un piso inactivo devuelve 422.</li><li>Un espacio activo de un piso inactivo responde `disponible: false`.</li><li>Por defecto los listados solo devuelven espacios disponibles; `incluirInactivos=true` devuelve todos.</li></ul> |
| **Prioridad** | Alta |
| **Estado / Squad responsable** | Implementado / Gaia |

<br>

<a id="rf-ms02-04"></a>
### RF-MS02-04 Gestionar zonas seguras y puntos de encuentro

| Atributo | Detalle |
|---|---|
| **ID** | RF-MS02-04 |
| **Nombre** | Gestionar zonas seguras y puntos de encuentro |
| **Épica / Grupo funcional** | MS02 - Evacuación |
| **Descripción** | Permitir al administrador registrar, modificar e inactivar zonas seguras, con código, nombre, ubicación, descripción, capacidad aproximada y, opcionalmente, coordenadas y zona abierta. Una zona segura puede marcarse como punto de encuentro. |
| **Actor(es)** | ADMIN para gestionar (`ms02:safe_zone:write`); todos los roles para consultar (`ms02:campus:read`) |
| **Precondiciones** | Para modificar o inactivar, la zona existe. |
| **Flujo base** | **Registrar:** el administrador envía los datos ([F-08](2_forms-inputs.md#f-08)). El sistema valida que el código sea único, que la capacidad sea mayor que cero y, si se indica, que la zona abierta esté activa. Guarda la zona `ACTIVO` con `puntoEncuentro = true` si no se envía.<br>**Modificar:** cambia todo menos el código. Coordenadas, zona abierta y `puntoEncuentro` solo cambian si se envían. Si cambian las coordenadas se recalculan sus rutas exteriores ([RF-MS02-13](#rf-ms02-13)).<br>**Inactivar:** el sistema verifica que la zona no sea destino de rutas activas y la inactiva.<br>**Consultar:** los usuarios con `ms02:campus:read` ven las zonas seguras activas, con los puntos de encuentro primero, y de cada una su ubicación, capacidad, fotografías y un enlace al mapa cuando tiene coordenadas (pantalla `#/zonas-seguras`). Las zonas inactivas no se muestran. |
| **Flujos alternos / excepciones** | <ul><li>**A1, código repetido:** 409 `CONFLICTO` con el campo `codigo`.</li><li>**A2, solo latitud o solo longitud:** 422 («La latitud y la longitud deben enviarse juntas»).</li><li>**A3, zona abierta inactiva:** 422.</li><li>**A4, la zona es destino de rutas activas:** 422; deben bloquearse o cambiar de destino antes.</li></ul> |
| **Reglas De Negocio** | <ul><li><a id="rn-ms02-09"></a>**RN-MS02-09:** El código de una zona segura es único, incluso entre zonas inactivas, y no cambia después de creado. MS04 lo muestra como identificador del punto de encuentro.</li><li><a id="rn-ms02-10"></a>**RN-MS02-10:** Una zona segura que es destino de una ruta activa no se inactiva.</li><li><a id="rn-ms02-11"></a>**RN-MS02-11:** Latitud y longitud se envían juntas o no se envían. Una zona sin coordenadas no puede ser destino de una ruta exterior.</li><li><a id="rn-ms02-12"></a>**RN-MS02-12:** No existe la entidad punto de encuentro: es una zona segura con `puntoEncuentro = true`. Una zona abierta no es zona segura salvo que el administrador la registre como tal.</li></ul> |
| **Postcondiciones** | La zona queda registrada, modificada o inactiva. |
| **Requerimientos asociados** | [RF-MS02-05](#rf-ms02-05), [RF-MS02-10](#rf-ms02-10), [RF-MS02-13](#rf-ms02-13) |
| **Criterios de aceptación** | <ul><li>Registrar dos zonas con el código `MP-H-01` devuelve 409 la segunda vez.</li><li>`GET /safe-zones?puntoEncuentro=true` solo devuelve puntos de encuentro; `sinCoordenadas=true` lista las que aún no tienen coordenadas.</li><li>Inactivar una zona destino de una ruta `RESTRINGIDA` devuelve 422.</li></ul> |
| **Prioridad** | Alta |
| **Estado / Squad responsable** | Implementado / Gaia |

<br>

<a id="rf-ms02-05"></a>
### RF-MS02-05 Gestionar rutas de evacuación

| Atributo | Detalle |
|---|---|
| **ID** | RF-MS02-05 |
| **Nombre** | Gestionar rutas de evacuación |
| **Épica / Grupo funcional** | MS02 - Evacuación |
| **Descripción** | Permitir al administrador asociar un espacio con una zona segura mediante una ruta descrita en texto y definir su estado. El administrador y el coordinador de emergencia cambian el estado de cualquier ruta, incluidas las exteriores. |
| **Actor(es)** | ADMIN (`ms02:route:write`); ADMIN y COORDINADOR para el estado (`ms02:route_status:write`) |
| **Precondiciones** | El espacio está disponible y la zona segura activa. |
| **Flujo base** | **Crear:** el administrador envía espacio de origen, zona de destino, descripción, estado y, si no es `DISPONIBLE`, motivo. El sistema valida y guarda la ruta.<br>**Actualizar:** cambia descripción y zona de destino; la nueva zona debe estar activa.<br>**Cambiar estado:** el sistema exige motivo para `BLOQUEADA` y `RESTRINGIDA` y lo borra al pasar a `DISPONIBLE`. |
| **Flujos alternos / excepciones** | <ul><li>**A1, el espacio ya tiene ruta:** 409 `CONFLICTO`.</li><li>**A2, falta el motivo:** 422.</li><li>**A3, origen no disponible o destino inactivo al dejar la ruta activa:** 422; la ruta solo puede quedar `BLOQUEADA`.</li><li>**A4, `PUT` sobre una ruta exterior:** 422; las rutas exteriores se recalculan ([RF-MS02-13](#rf-ms02-13)).</li></ul> |
| **Reglas De Negocio** | <ul><li><a id="rn-ms02-13"></a>**RN-MS02-13:** Una ruta tiene un solo origen: un espacio (ruta de texto) o una salida (ruta exterior). Un espacio es origen de como máximo una ruta.</li><li><a id="rn-ms02-14"></a>**RN-MS02-14:** Una ruta `BLOQUEADA` o `RESTRINGIDA` siempre tiene motivo. Al pasar a `DISPONIBLE` el motivo se borra.</li><li><a id="rn-ms02-15"></a>**RN-MS02-15:** Una ruta no pasa a `DISPONIBLE` ni a `RESTRINGIDA` si su origen no está disponible o su destino está inactivo.</li><li><a id="rn-ms02-16"></a>**RN-MS02-16:** Las rutas no se eliminan. Para reemplazar una ruta se cambian su destino y su descripción; para retirarla se bloquea.</li></ul> |
| **Postcondiciones** | La ruta queda creada o actualizada con su estado. |
| **Requerimientos asociados** | [RF-MS02-03](#rf-ms02-03), [RF-MS02-04](#rf-ms02-04), [RF-MS02-06](#rf-ms02-06), [RF-MS02-13](#rf-ms02-13) |
| **Criterios de aceptación** | <ul><li>Crear una segunda ruta para el mismo espacio devuelve 409.</li><li>Cambiar a `BLOQUEADA` sin motivo devuelve 422.</li><li>Un COORDINADOR puede cambiar el estado pero no crear rutas (403).</li></ul> |
| **Prioridad** | Alta |
| **Estado / Squad responsable** | Implementado / Gaia |

<br>

<a id="rf-ms02-06"></a>
### RF-MS02-06 Consultar la ruta de evacuación

| Atributo | Detalle |
|---|---|
| **ID** | RF-MS02-06 |
| **Nombre** | Consultar la ruta de evacuación |
| **Épica / Grupo funcional** | MS02 - Evacuación |
| **Descripción** | Permitir a cualquier usuario consultar cómo evacuar desde un espacio o desde los espacios de un edificio: la ruta con su estado y motivo, las recomendaciones preventivas y una guía con escaleras, salida y recorrido exterior hasta la zona segura. |
| **Actor(es)** | Todos los roles (`ms02:route:read`) |
| **Precondiciones** | El espacio está disponible o el edificio está activo. |
| **Flujo base** | <ol><li>El usuario elige un espacio (o un edificio).</li><li>El sistema devuelve la ruta del espacio con su estado, motivo y recomendaciones, o el aviso de que no tiene ruta.</li><li>Arma la guía: busca las salidas disponibles del edificio, las escaleras `DISPONIBLE` que llevan del piso del espacio al piso de cada salida y la ruta exterior guardada de cada salida.</li><li>Recomienda la salida alcanzable con menos tramos de escalera y, a igual número, la de recorrido exterior más corto, y responde una indicación en texto.</li></ol> |
| **Flujos alternos / excepciones** | <ul><li>**A1, espacio no disponible o edificio inactivo:** 422.</li><li>**A2, ninguna escalera disponible o ninguna salida con ruta:** 200 con `guia.mensaje` explicándolo y `guia.salida = null`.</li><li>**A3, la salida está en el mismo piso:** la guía no incluye escaleras.</li></ul> |
| **Reglas De Negocio** | <ul><li><a id="rn-ms02-17"></a>**RN-MS02-17:** Siempre se muestra el estado actual de la ruta.</li><li><a id="rn-ms02-18"></a>**RN-MS02-18:** Una escalera bloqueada, fuera de servicio o inactiva nunca se recomienda. Si ninguna escalera cubre todo el trayecto se encadenan varias. En cada tramo se recomienda la de menor prioridad y las demás se nombran como alternas.</li><li><a id="rn-ms02-19"></a>**RN-MS02-19:** La consulta usa la ruta exterior guardada y no llama al proveedor de navegación, para no depender de internet en una emergencia.</li></ul> |
| **Postcondiciones** | Solo lectura. |
| **Requerimientos asociados** | [RF-MS02-05](#rf-ms02-05), [RF-MS02-08](#rf-ms02-08), [RF-MS02-12](#rf-ms02-12), [RF-MS02-13](#rf-ms02-13) |
| **Criterios de aceptación** | <ul><li>Un espacio del piso 3 con una salida en el piso 1 y una escalera disponible 3→1 recibe una guía con esa escalera, la salida y el recorrido exterior.</li><li>Si la única escalera está `BLOQUEADA`, la guía no recomienda salida y explica por qué.</li><li>Consultar un espacio inactivo devuelve 422.</li></ul> |
| **Prioridad** | Alta |
| **Estado / Squad responsable** | Implementado / Gaia |

<br>

<a id="rf-ms02-07"></a>
### RF-MS02-07 Reactivar elementos del campus

| Atributo | Detalle |
|---|---|
| **ID** | RF-MS02-07 |
| **Nombre** | Reactivar elementos del campus |
| **Épica / Grupo funcional** | MS02 - Infraestructura |
| **Descripción** | Permitir al administrador reactivar un campus, zona abierta, edificio, piso, espacio, salida, escalera o zona segura inactivo, conservando todos sus datos. |
| **Actor(es)** | ADMIN (el permiso de escritura de cada elemento) |
| **Precondiciones** | El elemento existe y está inactivo. |
| **Flujo base** | <ol><li>El administrador elige «Reactivar».</li><li>El sistema verifica que lo que lo contiene esté activo.</li><li>Cambia el estado a `ACTIVO`.</li></ol> |
| **Flujos alternos / excepciones** | <ul><li>**A1, el elemento ya está activo:** 422.</li><li>**A2, lo que lo contiene está inactivo:** 422 (p. ej. «El edificio está inactivo: debe reactivarse primero»).</li></ul> |
| **Reglas De Negocio** | <ul><li><a id="rn-ms02-20"></a>**RN-MS02-20:** La reactivación no es en cascada.</li><li><a id="rn-ms02-21"></a>**RN-MS02-21:** Un piso solo se reactiva si su edificio está activo; un espacio o una salida, si su piso y su edificio lo están; una escalera, si sus pisos lo están; una zona segura, si su zona abierta lo está; una zona abierta, si su campus lo está.</li><li><a id="rn-ms02-22"></a>**RN-MS02-22:** Reactivar no cambia el estado de las rutas. Una ruta `BLOQUEADA` sigue así hasta que alguien cambie su estado.</li></ul> |
| **Postcondiciones** | El elemento queda activo con sus datos originales. |
| **Requerimientos asociados** | [RF-MS02-01](#rf-ms02-01) a [RF-MS02-04](#rf-ms02-04), [RF-MS02-10](#rf-ms02-10) a [RF-MS02-12](#rf-ms02-12) |
| **Criterios de aceptación** | <ul><li>Reactivar un edificio no reactiva sus pisos.</li><li>Reactivar un espacio de un piso inactivo devuelve 422.</li></ul> |
| **Prioridad** | Media |
| **Estado / Squad responsable** | Implementado / Gaia |

<br>

<a id="rf-ms02-08"></a>
### RF-MS02-08 Gestionar información preventiva

| Atributo | Detalle |
|---|---|
| **ID** | RF-MS02-08 |
| **Nombre** | Gestionar información preventiva |
| **Épica / Grupo funcional** | MS02 - Prevención |
| **Descripción** | Permitir al administrador registrar, modificar e inactivar recomendaciones preventivas asociadas a un edificio o a un tipo de espacio, y a los usuarios consultarlas. |
| **Actor(es)** | ADMIN para gestionar (`ms02:building:write`); todos los roles para consultar (`ms02:preventive_info:read`) |
| **Precondiciones** | Para el ámbito `EDIFICIO`, el edificio existe. |
| **Flujo base** | <ol><li>El administrador envía ámbito, edificio o tipo de espacio, título y recomendación.</li><li>El sistema valida que el ámbito tenga su dato y solo ese.</li><li>Guarda la recomendación `ACTIVO`.</li><li>El usuario las consulta por edificio o por tipo de espacio (pantalla `#/preventiva`, con `ms02:preventive_info:read`); también llegan con la consulta de rutas ([RF-MS02-06](#rf-ms02-06)).</li></ol> |
| **Flujos alternos / excepciones** | <ul><li>**A1, ámbito `EDIFICIO` sin edificio o con tipo de espacio (o al revés):** 400.</li><li>**A2, recomendación inexistente:** 404.</li></ul> |
| **Reglas De Negocio** | <ul><li><a id="rn-ms02-23"></a>**RN-MS02-23:** Cada recomendación aplica a un edificio o a un tipo de espacio, nunca a ambos.</li><li><a id="rn-ms02-24"></a>**RN-MS02-24:** Las recomendaciones no se eliminan, solo se inactivan. Una inactiva no se muestra.</li></ul> |
| **Postcondiciones** | La recomendación queda registrada, modificada o inactiva. |
| **Requerimientos asociados** | [RF-MS02-06](#rf-ms02-06) |
| **Criterios de aceptación** | <ul><li>Una recomendación de tipo `LABORATORIO` aparece al consultar la ruta de cualquier laboratorio.</li><li>Una recomendación inactiva no aparece en las consultas.</li></ul> |
| **Prioridad** | Media |
| **Estado / Squad responsable** | Implementado / Gaia |

<br>

<a id="rf-ms02-09"></a>
### RF-MS02-09 Consultar el catálogo del campus

| Atributo | Detalle |
|---|---|
| **ID** | RF-MS02-09 |
| **Nombre** | Consultar el catálogo del campus |
| **Épica / Grupo funcional** | MS02 - Catálogo |
| **Descripción** | Permitir a los usuarios y a los demás microservicios consultar campus, zonas abiertas, edificios, pisos, espacios, salidas, escaleras, zonas seguras e imágenes con su id y su estado. |
| **Actor(es)** | Todos los roles (`ms02:campus:read`); otros MS por el Gateway con la identidad propagada |
| **Precondiciones** | Usuario autenticado. |
| **Flujo base** | <ol><li>El cliente pide un listado o un elemento por id.</li><li>El sistema responde solo los activos (para espacios y salidas, solo los disponibles), salvo que se pida `incluirInactivos=true`. En escaleras, el parámetro equivalente es `incluirInactivas`.</li></ol> |
| **Flujos alternos / excepciones** | <ul><li>**A1, id con formato inválido:** 400.</li><li>**A2, elemento inexistente:** 404.</li></ul> |
| **Reglas De Negocio** | <ul><li><a id="rn-ms02-25"></a>**RN-MS02-25:** La consulta es de solo lectura. Los demás microservicios no acceden a la base de datos de MS02: usan esta API a través del Gateway.</li><li><a id="rn-ms02-26"></a>**RN-MS02-26:** Las respuestas incluyen `tieneFotografia` para edificios, espacios, salidas y zonas seguras.</li></ul> |
| **Postcondiciones** | Solo lectura. |
| **Requerimientos asociados** | Todos los de gestión. Lo usan MS04 (puntos de encuentro y ubicaciones) y MS06 (escenarios). |
| **Criterios de aceptación** | <ul><li>Un edificio inactivo no aparece en `GET /buildings` y sí con `incluirInactivos=true`.</li><li>Un id inexistente devuelve 404 `NO_ENCONTRADO`.</li></ul> |
| **Prioridad** | Alta |
| **Estado / Squad responsable** | Implementado / Gaia |

<br>

<a id="rf-ms02-10"></a>
### RF-MS02-10 Gestionar campus y zonas abiertas

| Atributo | Detalle |
|---|---|
| **ID** | RF-MS02-10 |
| **Nombre** | Gestionar campus y zonas abiertas |
| **Épica / Grupo funcional** | MS02 - Infraestructura |
| **Descripción** | Permitir al administrador crear, modificar e inactivar campus y sus zonas abiertas (zonas verdes, plazoletas, canchas, parqueaderos), que agrupan edificios y ubican las zonas seguras. |
| **Actor(es)** | ADMIN (`ms02:building:write`) |
| **Precondiciones** | Para registrar una zona abierta, el campus está activo. |
| **Flujo base** | **Campus:** el administrador envía nombre y descripción; el sistema lo guarda `ACTIVO`.<br>**Zona abierta:** envía nombre, tipo y, opcionalmente, coordenadas; el sistema la guarda `ACTIVO` en el campus.<br>**Inactivar:** un campus sin edificios ni zonas abiertas activas; una zona abierta sin zonas seguras activas. |
| **Flujos alternos / excepciones** | <ul><li>**A1, campus inactivo al registrar una zona abierta:** 422.</li><li>**A2, el campus tiene edificios o zonas abiertas activas:** 422.</li><li>**A3, la zona abierta tiene zonas seguras activas:** 422.</li></ul> |
| **Reglas De Negocio** | <ul><li><a id="rn-ms02-27"></a>**RN-MS02-27:** El despliegue crea un campus por defecto, «Campus principal».</li><li><a id="rn-ms02-28"></a>**RN-MS02-28:** Tipos de zona abierta: `ZONA_VERDE`, `PLAZOLETA`, `CANCHA`, `PARQUEADERO`, `OTRA`.</li></ul> |
| **Postcondiciones** | El campus o la zona abierta queda registrado, modificado o inactivo. |
| **Requerimientos asociados** | [RF-MS02-01](#rf-ms02-01), [RF-MS02-04](#rf-ms02-04), [RF-MS02-07](#rf-ms02-07) |
| **Criterios de aceptación** | <ul><li>Inactivar un campus con un edificio activo devuelve 422.</li><li>Una zona abierta con solo latitud devuelve 422.</li></ul> |
| **Prioridad** | Media |
| **Estado / Squad responsable** | Implementado / Gaia |

<br>

<a id="rf-ms02-11"></a>
### RF-MS02-11 Gestionar salidas

| Atributo | Detalle |
|---|---|
| **ID** | RF-MS02-11 |
| **Nombre** | Gestionar salidas |
| **Épica / Grupo funcional** | MS02 - Evacuación |
| **Descripción** | Permitir al administrador registrar, modificar e inactivar las salidas de un edificio al exterior, cada una con el piso desde el que se sale y sus coordenadas. |
| **Actor(es)** | ADMIN (`ms02:building:write`) |
| **Precondiciones** | El edificio y el piso existen y están activos, y el piso es del edificio. |
| **Flujo base** | <ol><li>El administrador envía piso, nombre, latitud y longitud.</li><li>El sistema valida los datos y que el piso sea del edificio y esté activo.</li><li>Guarda la salida `ACTIVO`.</li><li>Después de guardar, en segundo plano, genera sus rutas exteriores hacia las zonas seguras más cercanas ([RF-MS02-13](#rf-ms02-13)).</li></ol>**Modificar:** cambia piso, nombre y coordenadas; si cambian piso o coordenadas, se recalculan sus rutas.<br>**Inactivar:** la salida deja de usarse como origen de rutas exteriores. |
| **Flujos alternos / excepciones** | <ul><li>**A1, coordenadas fuera de rango o faltantes:** 400.</li><li>**A2, piso de otro edificio:** 422.</li><li>**A3, edificio o piso inactivo:** 422.</li></ul> |
| **Reglas De Negocio** | <ul><li><a id="rn-ms02-29"></a>**RN-MS02-29:** Solo los elementos exteriores tienen coordenadas: salidas (obligatorias), zonas seguras y zonas abiertas (opcionales). Las escaleras y los espacios no.</li><li><a id="rn-ms02-30"></a>**RN-MS02-30:** Una salida cuyo piso o edificio está inactivo responde `disponible: false` y no es origen de rutas exteriores ni aparece en la guía.</li></ul> |
| **Postcondiciones** | La salida queda registrada, modificada o inactiva. |
| **Requerimientos asociados** | [RF-MS02-02](#rf-ms02-02), [RF-MS02-06](#rf-ms02-06), [RF-MS02-13](#rf-ms02-13) |
| **Criterios de aceptación** | <ul><li>Registrar una salida con latitud 95 devuelve 400 con el campo `latitud`.</li><li>Al registrar una salida aparecen sus rutas exteriores sin que el administrador las cree.</li></ul> |
| **Prioridad** | Alta |
| **Estado / Squad responsable** | Implementado / Gaia |

<br>

<a id="rf-ms02-12"></a>
### RF-MS02-12 Gestionar escaleras

| Atributo | Detalle |
|---|---|
| **ID** | RF-MS02-12 |
| **Nombre** | Gestionar escaleras |
| **Épica / Grupo funcional** | MS02 - Evacuación |
| **Descripción** | Permitir al administrador registrar escaleras que conectan pisos de un edificio, ordenarlas por prioridad e inactivarlas, y al administrador o al coordinador bloquearlas o habilitarlas durante una emergencia. |
| **Actor(es)** | ADMIN (`ms02:building:write`); ADMIN y COORDINADOR para la disponibilidad (`ms02:route_status:write`) |
| **Precondiciones** | El edificio está activo y los pisos conectados existen, son del edificio y están activos. |
| **Flujo base** | <ol><li>El administrador envía nombre, pisos que conecta (al menos dos) y, opcionalmente, prioridad.</li><li>El sistema valida los pisos, genera el código `ESC-…` y, si no se envió prioridad, la deja después de las existentes.</li><li>Guarda la escalera `ACTIVO` y `DISPONIBLE`.</li></ol>**Cambiar disponibilidad:** `BLOQUEADA` y `FUERA_DE_SERVICIO` exigen motivo.<br>**Cambiar prioridad:** 1 es la principal del edificio. |
| **Flujos alternos / excepciones** | <ul><li>**A1, menos de dos pisos o pisos repetidos:** 400 o 422.</li><li>**A2, pisos de otro edificio o inactivos:** 422.</li><li>**A3, bloquear sin motivo:** 422.</li></ul> |
| **Reglas De Negocio** | <ul><li><a id="rn-ms02-31"></a>**RN-MS02-31:** Una escalera es una conexión vertical, nunca una salida, y no tiene coordenadas. No se modelan ascensores.</li><li><a id="rn-ms02-32"></a>**RN-MS02-32:** La disponibilidad es independiente del estado: el coordinador puede bloquear una escalera activa sin inactivarla.</li></ul> |
| **Postcondiciones** | La escalera queda registrada, con su disponibilidad y prioridad. |
| **Requerimientos asociados** | [RF-MS02-02](#rf-ms02-02), [RF-MS02-06](#rf-ms02-06) |
| **Criterios de aceptación** | <ul><li>Registrar una escalera con un solo piso devuelve 400.</li><li>Una escalera `BLOQUEADA` no aparece en la guía de evacuación.</li></ul> |
| **Prioridad** | Alta |
| **Estado / Squad responsable** | Implementado / Gaia |

<br>

<a id="rf-ms02-13"></a>
### RF-MS02-13 Calcular y generar rutas exteriores

| Atributo | Detalle |
|---|---|
| **ID** | RF-MS02-13 |
| **Nombre** | Calcular y generar rutas exteriores |
| **Épica / Grupo funcional** | MS02 - Evacuación |
| **Descripción** | Calcular el recorrido a pie de cada salida a sus zonas seguras más cercanas con OpenStreetMap y guardar el último resultado (distancia, duración, geometría y pasos). |
| **Actor(es)** | Sistema (automático); ADMIN para forzar o completar (`ms02:route:write`); todos los roles para consultar (`ms02:route:read`) |
| **Precondiciones** | La salida está disponible y la zona segura está activa y tiene coordenadas. |
| **Flujo base** | <ol><li>Al crear, mover o reactivar una salida, o al crear, mover o reactivar una zona segura con coordenadas, el sistema elige las N zonas más cercanas a cada salida (en línea recta, N = 3 por defecto).</li><li>Para cada par llama al proveedor de navegación peatonal.</li><li>Guarda el resultado; si el par ya tenía ruta, la recalcula.</li></ol>El administrador puede calcular un par puntual (`POST /evacuation-routes/exterior`) o generar las que falten (`POST /evacuation-routes/exterior/generate`, con `recalcular=true` para rehacerlas todas). |
| **Flujos alternos / excepciones** | <ul><li>**A1, salida no disponible, zona inactiva o sin coordenadas:** 422.</li><li>**A2, no hay recorrido a pie entre los puntos:** 422 `RUTA_NO_ENCONTRADA`.</li><li>**A3, el proveedor no responde:** 503 `NAVEGACION_NO_DISPONIBLE`. En la generación automática el error solo queda en el log y no afecta la operación del administrador; la generación manual responde `interrumpida: true`.</li></ul> |
| **Reglas De Negocio** | <ul><li><a id="rn-ms02-33"></a>**RN-MS02-33:** Hay a lo sumo una ruta por par salida–zona. Una ruta exterior no se edita a mano: se recalcula.</li><li><a id="rn-ms02-34"></a>**RN-MS02-34:** El proveedor de navegación está detrás de una interfaz y se configura por variables de entorno; la consulta de rutas no depende de él ([RN-MS02-19](#rn-ms02-19)).</li></ul> |
| **Postcondiciones** | Las rutas exteriores quedan guardadas y disponibles para la guía. |
| **Requerimientos asociados** | [RF-MS02-04](#rf-ms02-04), [RF-MS02-06](#rf-ms02-06), [RF-MS02-11](#rf-ms02-11) |
| **Criterios de aceptación** | <ul><li>Registrar una salida con coordenadas genera sus rutas sin otra acción.</li><li>`POST /evacuation-routes/exterior/generate` responde cuántas generó y cuántas omitió.</li><li>Con el proveedor caído, `GET /exits/{id}/evacuation-route` sigue devolviendo las rutas guardadas.</li></ul> |
| **Prioridad** | Alta |
| **Estado / Squad responsable** | Implementado / Gaia |

<br>

<a id="rf-ms02-14"></a>
### RF-MS02-14 Gestionar imágenes

| Atributo | Detalle |
|---|---|
| **ID** | RF-MS02-14 |
| **Nombre** | Gestionar imágenes |
| **Épica / Grupo funcional** | MS02 - Infraestructura |
| **Descripción** | Permitir al administrador subir y eliminar fotografías de edificios, espacios, salidas y zonas seguras, y a los usuarios verlas. |
| **Actor(es)** | ADMIN (`ms02:building:write`); todos los roles para ver (`ms02:campus:read`) |
| **Precondiciones** | El elemento existe y está activo. |
| **Flujo base** | <ol><li>El administrador sube el archivo con el tipo y el id del elemento.</li><li>El sistema valida el formato y el tamaño y que el elemento esté activo.</li><li>Guarda el archivo en el almacenamiento configurado ([RN-MS02-35](#rn-ms02-35)) y sus datos en la base de datos.</li></ol> |
| **Flujos alternos / excepciones** | <ul><li>**A1, formato distinto de JPEG, PNG o WEBP o archivo vacío:** 422 `REGLA_NEGOCIO`. Si se supera el límite de carga multipart de 5 MB, Spring puede rechazar la solicitud antes de llegar al controlador y responder 500 `ERROR_INTERNO`.</li><li>**A2, faltan `tipoElemento` o `elementoId`:** 400 `VALIDACION`.</li><li>**A3, elemento inexistente:** 404 `NO_ENCONTRADO`.</li><li>**A4, elemento inactivo:** 422 `REGLA_NEGOCIO`.</li><li>**A5, imagen inexistente:** 404.</li><li>**A6, falla el almacenamiento de archivos:** 500. Al subir, no se crea el registro de la imagen.</li></ul> |
| **Reglas De Negocio** | <ul><li><a id="rn-ms02-35"></a>**RN-MS02-35:** Los archivos se guardan fuera de PostgreSQL, en un volumen del contenedor o en Azure Blob Storage según la configuración; la base guarda nombre, tipo, tamaño, ruta y elemento.</li><li><a id="rn-ms02-36"></a>**RN-MS02-36:** Las imágenes sí se eliminan (archivo y registro).</li><li><a id="rn-ms02-41"></a>**RN-MS02-41:** El almacenamiento se elige con `IMAGEN_STORAGE_TYPE`: `local` (por defecto) o `azure`. Con `azure`, cada entorno usa su propio contenedor (`AZURE_STORAGE_CONTAINER`) y la cadena de conexión es un secreto del despliegue. La ruta guardada en la base (`imagenes/{uuid}.{ext}`) es también el nombre del blob, y `GET /images/{id}` no cambia.</li></ul> |
| **Postcondiciones** | La imagen queda asociada al elemento, o eliminada. |
| **Requerimientos asociados** | [RF-MS02-09](#rf-ms02-09) |
| **Criterios de aceptación** | <ul><li>Subir un PDF devuelve 422.</li><li>Tras subir una foto de un espacio, el espacio responde `tieneFotografia: true`.</li><li>Con `IMAGEN_STORAGE_TYPE=azure`, la foto subida queda como blob del contenedor del entorno y `GET /images/{id}` la devuelve igual que con `local`.</li></ul> |
| **Prioridad** | Media |
| **Estado / Squad responsable** | Implementado / Gaia |

<br>

<a id="rf-ms02-15"></a>
### RF-MS02-15 Generar códigos QR

| Atributo | Detalle |
|---|---|
| **ID** | RF-MS02-15 |
| **Nombre** | Generar códigos QR |
| **Épica / Grupo funcional** | MS02 - QR |
| **Descripción** | Permitir al administrador obtener el código QR (imagen PNG) de un espacio, una salida o una zona segura para imprimirlo. |
| **Actor(es)** | ADMIN (`ms02:qr:manage`) |
| **Precondiciones** | El elemento existe. |
| **Flujo base** | <ol><li>El administrador pide el QR del elemento.</li><li>El sistema genera al vuelo un PNG con el enlace `{QR_BASE_URL}/#/qr/{espacios\|salidas\|zonas}/{id}`.</li></ol> |
| **Flujos alternos / excepciones** | <ul><li>**A1, elemento inexistente:** 404.</li></ul> |
| **Reglas De Negocio** | <ul><li><a id="rn-ms02-37"></a>**RN-MS02-37:** El QR solo contiene el enlace a la pantalla pública del frontend, construido con el UUID; no lleva datos del elemento ni otro código.</li><li><a id="rn-ms02-38"></a>**RN-MS02-38:** `QR_BASE_URL` debe ser la dirección pública definitiva del frontend antes de imprimir: si cambia, los QR impresos dejan de funcionar.</li></ul> |
| **Postcondiciones** | Solo lectura; los QR no se guardan. |
| **Requerimientos asociados** | [RF-MS02-16](#rf-ms02-16) |
| **Criterios de aceptación** | <ul><li>El PNG de una salida, leído, contiene `…/#/qr/salidas/{id}`.</li><li>Un usuario sin `ms02:qr:manage` recibe 403.</li></ul> |
| **Prioridad** | Media |
| **Estado / Squad responsable** | Implementado / Gaia |

<br>

<a id="rf-ms02-16"></a>
### RF-MS02-16 Consultar información por QR sin sesión

| Atributo | Detalle |
|---|---|
| **ID** | RF-MS02-16 |
| **Nombre** | Consultar información por QR sin sesión |
| **Épica / Grupo funcional** | MS02 - QR |
| **Descripción** | Permitir que cualquier persona que escanee un QR vea, sin iniciar sesión, la información mínima del lugar y cómo evacuar. |
| **Actor(es)** | Cualquier persona, sin sesión |
| **Precondiciones** | Ninguna. |
| **Flujo base** | <ol><li>La persona escanea el QR y abre la pantalla pública del frontend.</li><li>El frontend llama a `GET /api/v1/public/{spaces\|exits\|safe-zones}?id={uuid}`.</li><li>El sistema responde los datos mínimos: para un espacio, nombre, tipo, edificio, piso, evacuación recomendada y guía; para una salida, nombre, edificio, piso y coordenadas; para una zona segura, código, nombre, ubicación, coordenadas y si es punto de encuentro.</li></ol> |
| **Flujos alternos / excepciones** | <ul><li>**A1, elemento, piso o edificio inactivo:** 200 con `disponible: false` y un aviso para seguir las indicaciones del personal.</li><li>**A2, id inexistente:** 404 `NO_ENCONTRADO`.</li><li>**A3, método distinto de GET:** se rechaza.</li></ul> |
| **Reglas De Negocio** | <ul><li><a id="rn-ms02-39"></a>**RN-MS02-39:** La consulta pública es de solo lectura y no devuelve ids internos, capacidad ni datos de administración.</li><li><a id="rn-ms02-40"></a>**RN-MS02-40:** El id va como parámetro (`?id=`) porque el Gateway solo acepta rutas públicas exactas.</li></ul> |
| **Postcondiciones** | Solo lectura. |
| **Requerimientos asociados** | [RF-MS02-06](#rf-ms02-06), [RF-MS02-15](#rf-ms02-15) |
| **Criterios de aceptación** | <ul><li>Sin cabeceras de identidad, `GET /public/spaces?id=` de un espacio disponible responde 200 con su evacuación.</li><li>`POST /public/spaces` se rechaza.</li></ul> |
| **Prioridad** | Alta |
| **Estado / Squad responsable** | Implementado / Gaia |

---

<a id="requerimientos-no-funcionales-rnf"></a>
## Requerimientos No Funcionales (RNF)

| ID | Requerimiento |
|---|---|
| **RNF-MS02-01** | Aplicación Spring Boot independiente (Java 21, Maven) con Spring Data JPA y base de datos PostgreSQL propia. El esquema se versiona con Flyway (`ddl-auto=validate`). Ningún otro servicio accede a la base. |
| **RNF-MS02-02** | Todos los endpoints están bajo `/api/v1/` y documentados con OpenAPI 3 (`/v3/api-docs`, publicado en [`api/openapi.yml`](api/openapi.yml)). Cada operación protegida declara `x-permiso` igual a la matriz de roles y permisos; una prueba automática (`ContratoPermisosTest`) lo verifica. |
| **RNF-MS02-03** | Todo acceso llega por el API Gateway. En las rutas protegidas MS02 recibe `X-User-Id`, `X-User-Role` y `X-User-Permission` y autoriza con `X-User-Permission`; si faltan responde 401 `NO_AUTENTICADO` y si falta el permiso, 403 `PROHIBIDO`. Las tres rutas de `/api/v1/public` no exigen identidad. |
| **RNF-MS02-04** | Todo error, incluidos los del framework (404, 405 y 400 para tipos de contenido no compatibles), responde con el `ErrorResponse` de los Estándares, sin trazas internas. |
| **RNF-MS02-05** | Cada línea de log lleva el `X-Correlation-ID`. Las operaciones de escritura quedan en el log, sin tokens ni datos personales. Actuator expone `health`. |
| **RNF-MS02-06** | La consulta de rutas en una emergencia no depende de servicios externos: usa las rutas exteriores guardadas ([RN-MS02-19](#rn-ms02-19)). El proveedor de navegación es el servicio público de OpenStreetMap con perfil a pie, configurable con `NAVIGATION_BASE_URL`, y tiene timeouts configurables (3 s de conexión y 10 s de lectura por defecto). |
| **RNF-MS02-07** | Las imágenes se guardan fuera de la base de datos, en un volumen del contenedor o en Azure Blob Storage según la configuración, con un máximo de 5 MB por archivo y solo JPEG, PNG o WEBP. |
| **RNF-MS02-08** | Secretos y configuración (base de datos, URL del frontend para los QR, proveedor de navegación, almacenamiento de imágenes) fuera del repositorio, en variables de entorno. |
| **RNF-MS02-09** | Pruebas unitarias y de integración de los servicios, controladores, seguridad y contrato OpenAPI, ejecutadas en GitHub Actions en cada Pull Request. |
| **RNF-MS02-10** | Ejecución en contenedor Docker. La imagen se publica en GHCR y se despliega en una VM de Azure detrás del Gateway, con firewall que solo acepta la IP del Gateway y mTLS en un Caddy de borde, con stacks separados de prueba y producción ([despliegue](diagrams/ms02-deployment-azure.puml)). |
