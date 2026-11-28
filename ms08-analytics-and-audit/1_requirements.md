# Requerimientos — MS08 Analítica y auditoría

Los valores transversales (cabeceras, errores, correlación, paginación, llamadas internas, auditoría) se rigen por los [Estándares de Integración](../integration/standards.md) y los permisos por la [matriz de roles y permisos](../integration/permissions-matrix.md). Este documento solo define el comportamiento de MS08. Los endpoints, con el permiso de cada uno, están en [`api/openapi.yml`](api/openapi.yml); los campos de entrada y sus mensajes, en [`2_forms-inputs.md`](2_forms-inputs.md); y la colección y sus índices, en [`3_data-model.md`](3_data-model.md).

## Índice de Contenido
1. [Alcance](#alcance)
2. [Requerimientos Funcionales](#requerimientos-funcionales)
   - [RF-AYA-01 Registrar evento de auditoría](#rf-aya-01)
   - [RF-AYA-02 Consultar registros de auditoría](#rf-aya-02)
   - [RF-AYA-03 Consultar el detalle de un registro de auditoría](#rf-aya-03)
   - [RF-AYA-04 Consultar indicadores de un evento](#rf-aya-04)
   - [RF-AYA-05 Comparar indicadores entre eventos](#rf-aya-05)
   - [RF-AYA-06 Identificar ubicaciones con mayores tiempos de respuesta](#rf-aya-06)
   - [RF-AYA-07 Conservar y depurar los registros de auditoría](#rf-aya-07)
3. [Reglas de negocio](#reglas-de-negocio)
4. [Requerimientos No Funcionales (RNF)](#requerimientos-no-funcionales-rnf)

---

<a id="alcance"></a>
## Alcance

MS08 reúne los registros de las acciones relevantes de los demás microservicios: Identidad y acceso (MS01), Campus y seguridad (MS02), Alertas y emergencias (MS03), Evacuación y check-in (MS04), Brigadistas e incidentes (MS05), Eventos (MS06) y Formación y asistente de IA (MS07). Con ellos ofrece la consulta del historial de auditoría según los permisos de cada usuario y calcula los indicadores operativos de simulacros y emergencias reales.

| Tipo de registro | `tipo_evento` | Permiso de auditoría que lo muestra |
|---|---|---|
| Acción de un simulacro | `SIMULACRO` (siempre con `evento_id`) | `ms08:audit_drill:read` |
| Acción de una emergencia real | `REAL` (con o sin `evento_id`) | `ms08:audit_real_event:read` |
| Acción administrativa, sin evento (inicio de sesión, gestión de usuarios) | Vacío | Ambos permisos |

---

<a id="requerimientos-funcionales"></a>
## Requerimientos Funcionales

<a id="rf-aya-01"></a>
### RF-AYA-01 Registrar evento de auditoría

| Atributo | Detalle |
|---|---|
| **ID** | RF-AYA-01 |
| **Nombre** | Registrar evento de auditoría |
| **Épica / Grupo funcional** | MS08 - Auditoría |
| **Descripción** | Permitir que los microservicios de CampusSeguro (MS01 a MS08) registren cada acción relevante de su dominio: inicios de sesión, creación o inicio de un evento, envío o confirmación de una alerta, registro o cambio de estado de un check-in, reporte o atención de un incidente. Los registros solo se insertan: no se editan ni se eliminan manualmente. Un reintento del mismo registro no crea duplicados. |
| **Actor(es)** | Microservicios emisores (MS01 a MS08), a través del API Gateway con su clave de servicio. |
| **Precondiciones** | El emisor tiene su clave de servicio configurada en el Gateway, que inyecta `X-Service-Name` con un valor de `ms01` a `ms08`. El nombre de la acción es el de su sección del [catálogo de auditoría](../integration/audit-catalog.md). |
| **Flujo base** | <ol><li>El emisor ejecuta una acción de su dominio y arma el registro ([F-01](2_forms-inputs.md)): `modulo`, `accion`, `resultado`, `fecha_hora` y `correlation_id`, y cuando aplican `id`, `usuario_id`, `recurso_afectado`, `detalle`, `evento_id`, `tipo_evento` y `atributos`.</li><li>Envía `POST /api/v1/audit/events` al Gateway sin bloquear la operación del usuario.</li><li>MS08 verifica que la petición traiga `X-Service-Name` de un microservicio (`ms01` a `ms08`).</li><li>Valida el cuerpo: campos obligatorios, formatos, longitudes, campos desconocidos, vigencia de `fecha_hora`, coherencia de `tipo_evento` y estructura de `atributos`.</li><li>Convierte `fecha_hora` a UTC, pasa a minúsculas `id`, `usuario_id` y `evento_id`, agrega la fecha de recepción (`recibido_en`) y el servicio de origen verificado.</li><li>Inserta el registro sin sobrescribir ninguno anterior.</li><li>Responde `202 Accepted` sin cuerpo.</li></ol> |
| **Flujos alternos / excepciones** | <ul><li>**A1 (paso 3), sin identidad de servicio o con un servicio no reconocido:** `401 NO_AUTENTICADO`; no se guarda nada.</li><li>**A2 (paso 3), la petición la envía un usuario:** `403 PROHIBIDO`.</li><li>**A3 (paso 4), datos inválidos o campos desconocidos:** `400 VALIDACION` con el detalle de cada campo; no se guarda nada. El emisor corrige el registro y no reintenta el mismo cuerpo.</li><li>**A4 (paso 6), registro ya almacenado:** responde `202` sin crear un duplicado ni modificar el original.</li><li>**A5, base de datos no disponible:** `503 SERVICIO_NO_DISPONIBLE`. El emisor conserva el registro y lo reintenta sin bloquear al usuario.</li></ul> |
| **Reglas De Negocio** | [RN-AYA-01](#rn-aya-01), [RN-AYA-03](#rn-aya-03), [RN-AYA-04](#rn-aya-04), [RN-AYA-05](#rn-aya-05), [RN-AYA-06](#rn-aya-06), [RN-AYA-07](#rn-aya-07), [RN-AYA-08](#rn-aya-08), [RN-AYA-09](#rn-aya-09), [RN-AYA-13](#rn-aya-13), [RN-AYA-19](#rn-aya-19) |
| **Postcondiciones** | El registro queda almacenado una sola vez y aparece en las consultas y en los indicadores. |
| **Requerimientos asociados** | [RF-AYA-02](#rf-aya-02), [RF-AYA-04](#rf-aya-04), [RF-AYA-07](#rf-aya-07) |
| **Criterios de aceptación** | <ul><li>Un registro válido enviado por un microservicio devuelve 202 y queda almacenado.</li><li>El mismo registro enviado dos veces devuelve 202 las dos veces y se almacena una sola vez.</li><li>Un campo desconocido, una `fecha_hora` más de 5 minutos en el futuro o anterior a la retención, o un `SIMULACRO` sin `evento_id` devuelven 400 y no guardan nada.</li><li>Una petición con identidad de usuario devuelve 403.</li></ul> |
| **Prioridad** | Alta |

<br>

<a id="rf-aya-02"></a>
### RF-AYA-02 Consultar registros de auditoría

| Atributo | Detalle |
|---|---|
| **ID** | RF-AYA-02 |
| **Nombre** | Consultar registros de auditoría |
| **Épica / Grupo funcional** | MS08 - Auditoría |
| **Descripción** | Permitir que los usuarios autorizados consulten por páginas el historial de auditoría, del registro más reciente al más antiguo, con filtros opcionales. Cada usuario solo ve los registros que sus permisos le autorizan. |
| **Actor(es)** | Usuarios con `ms08:audit_drill:read`, `ms08:audit_real_event:read` o ambos. |
| **Precondiciones** | Usuario autenticado por el Gateway con al menos uno de los dos permisos de lectura de auditoría. |
| **Flujo base** | <ol><li>El usuario indica los filtros que necesite ([F-02](2_forms-inputs.md)): `modulo`, `accion`, `usuario_id`, `evento_id`, `tipo_evento`, `recurso_afectado`, `correlation_id`, `desde`, `hasta`, y la paginación `page` y `size`.</li><li>MS08 verifica la identidad y el permiso.</li><li>Valida los filtros y la paginación.</li><li>Si se pidió un `tipo_evento`, comprueba que el usuario pueda verlo, y limita los resultados a los tipos de registro que sus permisos autorizan.</li><li>Busca los registros, los ordena por `fecha_hora` descendente con `id` como desempate y calcula los totales.</li><li>Responde la página con `page`, `size`, `total_elementos` y `total_paginas`.</li></ol> |
| **Flujos alternos / excepciones** | <ul><li>**A1, sin identidad de usuario válida:** `401 NO_AUTENTICADO`.</li><li>**A2, sin permiso de auditoría, filtro por un tipo de evento no permitido o petición de un microservicio:** `403 PROHIBIDO`.</li><li>**A3, filtro inválido:** UUID mal formado, `desde` posterior a `hasta`, `tipo_evento` desconocido, `page` negativa o `size` fuera de 1 a 100: `400 VALIDACION` con el campo.</li><li>**A4, sin coincidencias:** página vacía con los totales en 0, sin error.</li><li>**A5, base de datos no disponible:** `503 SERVICIO_NO_DISPONIBLE`.</li></ul> |
| **Reglas De Negocio** | [RN-AYA-10](#rn-aya-10), [RN-AYA-12](#rn-aya-12), [RN-AYA-13](#rn-aya-13), [RN-AYA-18](#rn-aya-18) |
| **Postcondiciones** | Ninguna: la consulta no modifica datos. |
| **Requerimientos asociados** | [RF-AYA-01](#rf-aya-01), [RF-AYA-03](#rf-aya-03) |
| **Criterios de aceptación** | <ul><li>Un usuario con solo `ms08:audit_drill:read` solo recibe registros `SIMULACRO`.</li><li>Los registros administrativos solo aparecen a quien tiene los dos permisos.</li><li>`usuario_id` y `evento_id` aceptan uno o varios UUID separados por coma (máximo 100).</li><li>`size` fuera de 1 a 100 devuelve 400.</li></ul> |
| **Prioridad** | Alta |

<br>

<a id="rf-aya-03"></a>
### RF-AYA-03 Consultar el detalle de un registro de auditoría

| Atributo | Detalle |
|---|---|
| **ID** | RF-AYA-03 |
| **Nombre** | Consultar el detalle de un registro de auditoría |
| **Épica / Grupo funcional** | MS08 - Auditoría |
| **Descripción** | Permitir que los usuarios autorizados consulten la información completa de un registro elegido del historial, después de verificar que pueden ver su tipo de evento. |
| **Actor(es)** | Usuarios con `ms08:audit_drill:read`, `ms08:audit_real_event:read` o ambos. |
| **Precondiciones** | Usuario autenticado con el permiso que corresponde al tipo del registro (ambos para un registro administrativo). El registro existe y no venció su retención ([RF-AYA-07](#rf-aya-07)). |
| **Flujo base** | <ol><li>El usuario elige un registro del historial ([F-03](2_forms-inputs.md)).</li><li>MS08 verifica la identidad y que tenga al menos un permiso de auditoría.</li><li>Busca el registro por su `id`.</li><li>Comprueba que el usuario pueda ver su tipo de evento.</li><li>Responde todos sus campos: `id`, `usuario_id`, `modulo`, `accion`, `recurso_afectado`, `resultado`, `fecha_hora`, `recibido_en`, `correlation_id`, `detalle`, `evento_id`, `tipo_evento` y `atributos`. El servicio de origen verificado no se muestra.</li></ol> |
| **Flujos alternos / excepciones** | <ul><li>**A1, sin identidad de usuario válida:** `401 NO_AUTENTICADO`.</li><li>**A2, sin permiso para el tipo del registro o petición de un microservicio:** `403 PROHIBIDO`.</li><li>**A3, el registro no existe o ya venció su retención:** `404 NO_ENCONTRADO`.</li><li>**A4, base de datos no disponible:** `503 SERVICIO_NO_DISPONIBLE`.</li></ul> |
| **Reglas De Negocio** | [RN-AYA-10](#rn-aya-10), [RN-AYA-12](#rn-aya-12) |
| **Postcondiciones** | Ninguna: la consulta no modifica datos. |
| **Requerimientos asociados** | [RF-AYA-02](#rf-aya-02), [RF-AYA-07](#rf-aya-07) |
| **Criterios de aceptación** | <ul><li>El detalle de un registro `REAL` con solo `ms08:audit_drill:read` devuelve 403.</li><li>Un `id` inexistente devuelve 404.</li></ul> |
| **Prioridad** | Media |

<br>

<a id="rf-aya-04"></a>
### RF-AYA-04 Consultar indicadores de un evento

| Atributo | Detalle |
|---|---|
| **ID** | RF-AYA-04 |
| **Nombre** | Consultar indicadores de un evento |
| **Épica / Grupo funcional** | MS08 - Analítica |
| **Descripción** | Calcular en el momento de la consulta los indicadores operativos de un simulacro o una emergencia real a partir de sus registros de auditoría: notificaciones y confirmaciones (MS03), participación, distribución por estado, puntos de encuentro y solicitudes de asistencia (MS04), incidentes por prioridad y tiempos hasta la primera atención (MS05) y la hora real de inicio (MS06). Como no se guardan, incluyen los registros que lleguen con retraso. |
| **Actor(es)** | Usuarios con `ms08:indicators_drill:read` (simulacros) o `ms08:indicators_real_event:read` (emergencias reales). Los permisos de auditoría no dan acceso a los indicadores. |
| **Precondiciones** | Usuario autenticado con el permiso de indicadores del tipo del evento. Existe al menos un registro del evento. |
| **Flujo base** | <ol><li>El usuario elige el evento ([F-04](2_forms-inputs.md)).</li><li>MS08 verifica la identidad y que tenga al menos un permiso de indicadores.</li><li>Valida que `evento_id` sea un UUID y lo pasa a minúsculas.</li><li>Carga los registros del evento y determina su tipo (`SIMULACRO` o `REAL`).</li><li>Comprueba el permiso de indicadores de ese tipo.</li><li>Calcula los indicadores (tabla siguiente).</li><li>Responde los indicadores con `evento_id`, `tipo_evento`, `inicio_evento` y `calculado_en`.</li></ol> |
| **Indicadores** | <ul><li>**`inicio_evento`:** el `EVENT_STARTED` exitoso más temprano de MS06; vacío si no ha llegado.</li><li>**`notificacion`:** destinatarios, envíos exitosos y fallidos (`NOTIFICATION_SENT` y `NOTIFICATION_FAILED` de MS03, sumando los destinatarios de todas las alertas del evento), usuarios distintos que confirmaron (`CONFIRM`) y porcentaje de confirmación = confirmados / destinatarios × 100.</li><li>**`participacion`:** con los `REGISTRO_CHECKIN` y `CAMBIO_ESTADO_CHECKIN` exitosos de MS04: participantes (usuarios distintos), porcentaje de participación = participantes / destinatarios × 100, distribución por el último estado de cada persona (`SIN_ESTADO` si falta), distribución por punto de encuentro de quienes terminaron en `SAFE` (`SIN_PUNTO` si falta), personas que alguna vez pidieron asistencia (`NEEDS_ASSISTANCE`) y porcentaje de asistencia = solicitaron asistencia / participantes × 100. Cada grupo trae clave, cantidad y porcentaje, de mayor a menor cantidad.</li><li>**`incidentes`:** los `CREAR_INCIDENTE` exitosos de MS05, por prioridad (`SIN_PRIORIDAD` si falta), con clave, cantidad y porcentaje.</li><li>**`tiempos`:** tiempo promedio hasta el primer check-in y hasta el primer `SAFE` de cada persona, medidos desde `inicio_evento` y sin contar registros anteriores; tiempo promedio hasta la primera atención (diferencia entre el `INICIAR_ATENCION` y el `CREAR_INCIDENTE` más tempranos de cada incidente, sin diferencias negativas ni incidentes sin reporte); y las 10 ubicaciones con mayor tiempo promedio de primera atención. Cada promedio trae segundos con un decimal y la cantidad de muestras.</li></ul> |
| **Flujos alternos / excepciones** | <ul><li>**A1, sin identidad de usuario válida:** `401 NO_AUTENTICADO`.</li><li>**A2, sin el permiso de indicadores del tipo del evento o petición de un microservicio:** `403 PROHIBIDO`.</li><li>**A3, `evento_id` no es un UUID:** `400 VALIDACION` con el campo `evento_id`.</li><li>**A4, no hay registros del evento:** `404 NO_ENCONTRADO`.</li><li>**A5, faltan datos para algún indicador:** responde sin error, con 0 en conteos y muestras, listas vacías y valores vacíos donde no se puede calcular.</li><li>**A6, base de datos no disponible:** `503 SERVICIO_NO_DISPONIBLE`.</li></ul> |
| **Reglas De Negocio** | [RN-AYA-11](#rn-aya-11), [RN-AYA-12](#rn-aya-12), [RN-AYA-14](#rn-aya-14), [RN-AYA-15](#rn-aya-15), [RN-AYA-16](#rn-aya-16) |
| **Postcondiciones** | Ninguna: los indicadores no se almacenan. |
| **Requerimientos asociados** | [RF-AYA-01](#rf-aya-01), [RF-AYA-05](#rf-aya-05), [RF-AYA-06](#rf-aya-06) |
| **Criterios de aceptación** | <ul><li>Un evento sin registros de MS05 devuelve los indicadores de incidentes en 0 sin error.</li><li>Un usuario con solo permisos de auditoría recibe 403.</li><li>Los porcentajes están entre 0 y 100 y, como los tiempos, se redondean a un decimal.</li></ul> |
| **Prioridad** | Alta |

<br>

<a id="rf-aya-05"></a>
### RF-AYA-05 Comparar indicadores entre eventos

| Atributo | Detalle |
|---|---|
| **ID** | RF-AYA-05 |
| **Nombre** | Comparar indicadores entre eventos |
| **Épica / Grupo funcional** | MS08 - Analítica |
| **Descripción** | Comparar los indicadores principales de 2 a 10 eventos distintos del mismo tipo, ordenados del más antiguo al más reciente, con la variación de cada indicador clave respecto al evento anterior. |
| **Actor(es)** | Usuarios con `ms08:indicators_drill:read` o `ms08:indicators_real_event:read`. |
| **Precondiciones** | Usuario autenticado con el permiso de indicadores del tipo de los eventos. Todos los eventos tienen registros y son del mismo tipo. |
| **Flujo base** | <ol><li>El usuario elige los eventos ([F-05](2_forms-inputs.md)); viajan como parámetros `evento_id` repetidos.</li><li>MS08 verifica la identidad y que tenga al menos un permiso de indicadores.</li><li>Valida que cada `evento_id` sea un UUID, los pasa a minúsculas, unifica los repetidos y comprueba que queden entre 2 y 10.</li><li>Carga los registros de cada evento, comprueba que todos existan, que sean del mismo tipo y que el usuario tenga el permiso de ese tipo.</li><li>Calcula los indicadores de cada evento con las reglas de [RF-AYA-04](#rf-aya-04).</li><li>Ordena las filas por `inicio_evento` (los eventos sin inicio al final, con `evento_id` como desempate) y calcula `cambio_vs_anterior`.</li><li>Responde la lista `eventos`.</li></ol> |
| **Datos de cada fila** | `evento_id`, `tipo_evento`, `inicio_evento`, participantes, porcentaje de participación, porcentaje de confirmación, porcentaje de asistencia, total de incidentes, tiempo promedio hasta el check-in, tiempo promedio de evacuación, tiempo promedio hasta la primera atención y `cambio_vs_anterior`: valor actual menos el anterior, con un decimal, para los porcentajes de participación y confirmación y los tres tiempos. La primera fila no tiene variación, y una métrica queda vacía si falta en alguno de los dos eventos. En los porcentajes un valor positivo es una mejora; en los tiempos, un valor negativo es una respuesta más rápida. |
| **Flujos alternos / excepciones** | <ul><li>**A1, sin identidad de usuario válida:** `401 NO_AUTENTICADO`.</li><li>**A2, sin el permiso de indicadores del tipo de algún evento o petición de un microservicio:** `403 PROHIBIDO`.</li><li>**A3, sin eventos, con un UUID inválido, con menos de 2 o más de 10 eventos distintos, o mezclando simulacros con emergencias reales:** `400 VALIDACION` con el campo `evento_id`.</li><li>**A4, algún evento sin registros:** `404 NO_ENCONTRADO`.</li><li>**A5, base de datos no disponible:** `503 SERVICIO_NO_DISPONIBLE`.</li></ul> |
| **Reglas De Negocio** | [RN-AYA-11](#rn-aya-11), [RN-AYA-14](#rn-aya-14), [RN-AYA-16](#rn-aya-16), [RN-AYA-17](#rn-aya-17) |
| **Postcondiciones** | Ninguna: la comparación no se almacena. |
| **Requerimientos asociados** | [RF-AYA-04](#rf-aya-04) |
| **Criterios de aceptación** | <ul><li>Un solo evento, o el mismo evento repetido, devuelve 400.</li><li>Un simulacro y una emergencia real juntos devuelven 400.</li><li>La primera fila no tiene `cambio_vs_anterior`.</li></ul> |
| **Prioridad** | Media |

<br>

<a id="rf-aya-06"></a>
### RF-AYA-06 Identificar ubicaciones con mayores tiempos de respuesta

| Atributo | Detalle |
|---|---|
| **ID** | RF-AYA-06 |
| **Nombre** | Identificar ubicaciones con mayores tiempos de respuesta |
| **Épica / Grupo funcional** | MS08 - Analítica |
| **Descripción** | Consolidar, para 1 a 10 eventos del mismo tipo, los edificios o zonas del campus con su total de incidentes, solicitudes de asistencia y tiempo promedio hasta la primera atención, con el desglose por evento, ordenados de la ubicación más lenta a la más rápida. |
| **Actor(es)** | Usuarios con `ms08:indicators_drill:read` o `ms08:indicators_real_event:read`. |
| **Precondiciones** | Usuario autenticado con el permiso de indicadores del tipo de los eventos. Todos los eventos tienen registros y son del mismo tipo. Las ubicaciones salen de los atributos `ubicacion_tipo` y `ubicacion_id` que envían MS04 y MS05; un registro sin ambos no se asigna a ninguna ubicación. |
| **Flujo base** | <ol><li>El usuario elige los eventos ([F-06](2_forms-inputs.md)); viajan como parámetros `evento_id` repetidos.</li><li>MS08 verifica la identidad y que tenga al menos un permiso de indicadores.</li><li>Valida los UUID, los pasa a minúsculas, unifica los repetidos y comprueba que queden entre 1 y 10.</li><li>Carga los registros de cada evento, comprueba que existan, que sean del mismo tipo y que el usuario tenga el permiso de ese tipo.</li><li>Agrupa por tipo e identificador de ubicación los incidentes (MS05), las solicitudes de asistencia (usuarios distintos que pidieron asistencia en MS04, sumados por evento) y los tiempos de primera atención, en total y por evento.</li><li>Ordena las ubicaciones de mayor a menor tiempo promedio ponderado (las que no tienen muestras al final; en empate, más incidentes primero y después por tipo e identificador) y responde `ubicaciones`, cada una con su `por_evento`.</li></ol> |
| **Flujos alternos / excepciones** | <ul><li>**A1, sin identidad de usuario válida:** `401 NO_AUTENTICADO`.</li><li>**A2, sin el permiso de indicadores o petición de un microservicio:** `403 PROHIBIDO`.</li><li>**A3, sin eventos, con un UUID inválido, con más de 10 eventos distintos o mezclando tipos:** `400 VALIDACION` con el campo `evento_id`.</li><li>**A4, algún evento sin registros:** `404 NO_ENCONTRADO`.</li><li>**A5, ningún registro trae ubicación:** lista vacía, sin error.</li><li>**A6, base de datos no disponible:** `503 SERVICIO_NO_DISPONIBLE`.</li></ul> |
| **Reglas De Negocio** | [RN-AYA-11](#rn-aya-11), [RN-AYA-14](#rn-aya-14), [RN-AYA-16](#rn-aya-16), [RN-AYA-17](#rn-aya-17) |
| **Postcondiciones** | Ninguna: el resultado no se almacena. |
| **Requerimientos asociados** | [RF-AYA-04](#rf-aya-04) |
| **Criterios de aceptación** | <ul><li>Un solo evento es válido.</li><li>Las ubicaciones sin muestras de atención quedan al final.</li><li>Registros sin `ubicacion_tipo` o `ubicacion_id` no aparecen en ninguna ubicación.</li></ul> |
| **Prioridad** | Media |

<br>

<a id="rf-aya-07"></a>
### RF-AYA-07 Conservar y depurar los registros de auditoría

| Atributo | Detalle |
|---|---|
| **ID** | RF-AYA-07 |
| **Nombre** | Conservar y depurar los registros de auditoría (retención) |
| **Épica / Grupo funcional** | MS08 - Auditoría |
| **Descripción** | Conservar cada registro durante 180 días contados desde su `fecha_hora` y eliminar automáticamente los que superen ese plazo, sin intervención de usuarios ni de otros servicios. Al arrancar, MS08 deja preparados los índices de consulta. |
| **Actor(es)** | MS08 y su base de datos. |
| **Precondiciones** | MS08 conectado a su base de datos propia. |
| **Flujo base** | <ol><li>Al arrancar, `AuditIndexInitializer` asegura los índices de consulta (tipo y fecha, evento y fecha, usuario y fecha, módulo, acción y fecha, y `correlation_id`) y, con `AUDIT_TTL_INDEX_ENABLED=true` (valor por defecto), el índice de expiración (TTL) sobre `fecha_hora` con los días de `AUDIT_RETENTION_DAYS` (180 por defecto).</li><li>La base de datos elimina en segundo plano los registros vencidos.</li><li>Los registros eliminados dejan de aparecer en las consultas ([RF-AYA-02](#rf-aya-02), [RF-AYA-03](#rf-aya-03)) y en los indicadores ([RF-AYA-04](#rf-aya-04), [RF-AYA-05](#rf-aya-05), [RF-AYA-06](#rf-aya-06)).</li></ol> |
| **Flujos alternos / excepciones** | <ul><li>**A1, la base de datos no admite índices TTL:** se configura `AUDIT_TTL_INDEX_ENABLED=false` y `AUDIT_CLEANUP_JOB_ENABLED=true`. La tarea `AuditRetentionJob` corre todos los días a las 3:30 a. m. (zona `America/Bogota`, configurable con `APP_TIMEZONE`), elimina los registros con `fecha_hora` anterior al límite y registra en el log cuántos eliminó.</li><li>**A2, cambio de los días de retención con el índice TTL activo:** el índice anterior (`ttl_fecha_hora`) se elimina en la base de datos antes de reiniciar MS08, para que se cree con el nuevo plazo.</li></ul> |
| **Reglas De Negocio** | [RN-AYA-01](#rn-aya-01), [RN-AYA-02](#rn-aya-02) |
| **Postcondiciones** | Ningún registro supera el periodo de retención. |
| **Requerimientos asociados** | [RF-AYA-01](#rf-aya-01), [RF-AYA-02](#rf-aya-02), [RF-AYA-03](#rf-aya-03) |
| **Criterios de aceptación** | <ul><li>Al arrancar existen los índices de consulta y el índice TTL.</li><li>Con la tarea diaria activa, los registros más antiguos que el límite se eliminan y el log indica cuántos.</li></ul> |
| **Prioridad** | Media |

<br>

---

<a id="reglas-de-negocio"></a>
## Reglas de negocio

* <a id="rn-aya-01"></a>**RN-AYA-01:** Los registros de auditoría son de solo inserción: no existen operaciones para modificarlos ni eliminarlos. Solo se eliminan al cumplir su periodo de retención ([RF-AYA-07](#rf-aya-07)).
* <a id="rn-aya-02"></a>**RN-AYA-02:** Cada registro se conserva 180 días contados desde su `fecha_hora`.
* <a id="rn-aya-03"></a>**RN-AYA-03:** El registro es idempotente: el `id` que envía el emisor identifica el registro, y si no lo envía MS08 lo calcula con un hash SHA-256 de `correlation_id`, `modulo`, `accion`, `recurso_afectado` y `fecha_hora`. Un registro con un `id` ya almacenado se acepta sin crear un duplicado ni modificar el original.
* <a id="rn-aya-04"></a>**RN-AYA-04:** Los identificadores de registro, de usuario y de evento se guardan en minúsculas, y todas las fechas en UTC.
* <a id="rn-aya-05"></a>**RN-AYA-05:** Todo registro lleva un `correlation_id` (máximo 100 caracteres), el mismo de la operación que lo originó, para rastrearla entre microservicios. Cada respuesta de MS08 y cada línea de su log llevan el Correlation ID de la petición.
* <a id="rn-aya-06"></a>**RN-AYA-06:** `fecha_hora` no puede superar en más de 5 minutos la hora de MS08 ni ser anterior al periodo de retención.
* <a id="rn-aya-07"></a>**RN-AYA-07:** Con `evento_id`, `tipo_evento` es obligatorio. `SIMULACRO` siempre exige `evento_id`. Sin evento solo se admite `REAL` o el tipo vacío (acciones administrativas).
* <a id="rn-aya-08"></a>**RN-AYA-08:** El cuerpo de registro es estricto: se rechaza cualquier campo desconocido, y `atributos` es una estructura plana de máximo 20 claves en `snake_case` cuyos valores son texto (máximo 100 caracteres), número o booleano.
* <a id="rn-aya-09"></a>**RN-AYA-09:** `detalle` (máximo 100 caracteres) solo lleva un código o motivo, y `atributos` solo identificadores técnicos, códigos y conteos. Nunca se guardan datos personales ni contenido de mensajes.
* <a id="rn-aya-10"></a>**RN-AYA-10:** La visibilidad del historial se controla con dos permisos independientes: `ms08:audit_drill:read` (simulacros) y `ms08:audit_real_event:read` (emergencias reales). Los registros administrativos exigen ambos.
* <a id="rn-aya-11"></a>**RN-AYA-11:** La visibilidad de los indicadores, comparaciones y ubicaciones depende del tipo del evento y se controla con `ms08:indicators_drill:read` y `ms08:indicators_real_event:read`. Los permisos de auditoría no dan acceso a la analítica, ni al revés.
* <a id="rn-aya-12"></a>**RN-AYA-12:** MS08 autoriza con los permisos que el Gateway envía en `X-User-Permission`, nunca por el nombre del rol. La validación del token y la política de acceso perimetral son del Gateway.
* <a id="rn-aya-13"></a>**RN-AYA-13:** El registro de auditoría es exclusivo de los microservicios (`X-Service-Name`), y las consultas son exclusivas de usuarios con permiso. Un intento cruzado se rechaza con 403.
* <a id="rn-aya-14"></a>**RN-AYA-14:** Los indicadores no se guardan: se calculan en cada consulta con los registros vigentes.
* <a id="rn-aya-15"></a>**RN-AYA-15:** Los indicadores solo cuentan registros con resultado `EXITO`, salvo las notificaciones de MS03, donde se cuentan los envíos exitosos y los fallidos.
* <a id="rn-aya-16"></a>**RN-AYA-16:** Los porcentajes (0 a 100), los tiempos promedio en segundos y las variaciones se redondean a un decimal. Si no hay datos para calcular un valor (sin destinatarios, sin hora de inicio, tiempos negativos por desfase de relojes), el valor queda vacío y las muestras en 0.
* <a id="rn-aya-17"></a>**RN-AYA-17:** Una comparación admite de 2 a 10 eventos distintos, y el análisis de ubicaciones de 1 a 10, siempre del mismo tipo. Los eventos repetidos se unifican antes de validar los límites.
* <a id="rn-aya-18"></a>**RN-AYA-18:** Los listados de auditoría se entregan paginados (20 registros por defecto, máximo 100), del más reciente al más antiguo y con el `id` como desempate, para que no se repitan ni se salten registros entre páginas.
* <a id="rn-aya-19"></a>**RN-AYA-19:** Ningún otro microservicio accede a la base de datos de MS08: toda interacción pasa por el Gateway. El envío de auditoría nunca bloquea la operación del usuario en el emisor; ante fallos temporales el emisor conserva el registro y lo reintenta, y los errores de validación no se reintentan sin corregir el cuerpo.

---

<a id="requerimientos-no-funcionales-rnf"></a>
## Requerimientos No Funcionales (RNF)

| ID | Categoría | Requerimiento |
|---|---|---|
| <a id="rnf-aya-01"></a>RNF-AYA-01 | Seguridad | MS08 solo confía en las cabeceras que inyecta el Gateway (`X-Service-Name`, `X-User-Id`, `X-User-Permission`) y rechaza con 401 una identidad mal formada ([security.md](security.md)). |
| <a id="rnf-aya-02"></a>RNF-AYA-02 | Privacidad | Los logs nunca incluyen cuerpos de petición, `detalle`, `atributos`, identificadores de usuario, cabeceras ni credenciales de conexión. |
| <a id="rnf-aya-03"></a>RNF-AYA-03 | Consistencia | Todos los errores salen como `ErrorResponse` (`status`, `codigo`, `mensaje`, `errores`, `correlation_id`), incluidos los del framework. |
| <a id="rnf-aya-04"></a>RNF-AYA-04 | Disponibilidad | `/actuator/health` responde el estado sin detalles internos. |
| <a id="rnf-aya-05"></a>RNF-AYA-05 | Rendimiento | Las consultas por tipo, evento, usuario, módulo y acción, y por `correlation_id`, usan los índices de [RF-AYA-07](#rf-aya-07). |
| <a id="rnf-aya-06"></a>RNF-AYA-06 | Calidad | Las pruebas, la cobertura y el análisis estático están en [quality-and-tests.md](quality-and-tests.md). |
