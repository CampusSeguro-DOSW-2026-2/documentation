<h1 align="center">🛡️ Campus Seguro</h1>
<h3 align="center">MS03 · Alertas y Emergencias</h3>

<p align="center"><em>Análisis de requerimientos funcionales, reglas de negocio e historial de revisión.</em></p>

<p align="center">
  <img alt="Curso" src="https://img.shields.io/badge/Curso-DOSW%202026--2-1f6feb">
  <img alt="Squad" src="https://img.shields.io/badge/Squad-Gaia%20%C2%B7%20Kairos%20%C2%B7%20Zephyros-8957e5">
  <img alt="Documento" src="https://img.shields.io/badge/Documento-Requerimientos-d29922">
  <img alt="Versión" src="https://img.shields.io/badge/Versi%C3%B3n-1.0-2da44e">
</p>

<p align="center">
  <a href="#rf-alt-01">📌 Requerimientos</a> ·
  <a href="#reglas-de-negocio">📜 Reglas de negocio</a> ·
  <a href="#abreviaturas">🔤 Abreviaturas</a> ·
  <a href="#historial-de-revision">🕓 Historial</a> ·
  <a href=https://github.com/CampusSeguro-DOSW-2026-2/ms03-alerts-and-emergencies.git>📦 Repositorio</a> ·
  <a href=https://dosw-team.atlassian.net/jira/software/projects/CS/boards/35/backlog?epics=visible&atlOrigin=eyJpIjoiYjcxZjVlODBmNjBjNGE4ZmI4MTg0NGUzNDk3NmE1OGIiLCJwIjoiaiJ9>📋 Jira</a>
</p>

---

## 📚 Ficha del documento

| | |
|:--|:--|
| **Curso** | Desarrollo y Operaciones de Software 2026-2 |
| **Fecha** | 06/11/2026 |
| **Versión** | 3.0 |

## 👥 Equipo

| Integrante | Rol | GitHub |
|:--|:--|:--|
| Rodrigo Humberto Gualtero Martínez | Product Owner | [ginnko2019](https://github.com/ginnko2019) |
| Tomás Olaya Díaz | Scrum Master | [iAxstral](https://github.com/iAxstral) |
| José Daniel García Pineda | Equipo de desarrollo | [KenjiMaster](https://github.com/KenjiMaster) |
| Jerónimo Moreno Herrera | Equipo de desarrollo | [Dracodec113](https://github.com/Dracodec113) |
| Paula Alejandra Novoa Castellanos | Equipo de desarrollo | [Aleja15-31](https://github.com/Aleja15-31) |
| Derly Valeria Pachón Pinzón | Equipo de desarrollo | [itsValepP](https://github.com/itsValepP) |

## 📝 Propósito del documento

Este documento detalla cada uno de los requerimientos funcionales del microservicio **MS03 - Alertas y Emergencias**, incluyendo su funcionalidad, datos de entrada y salida, flujo básico y flujo alterno. Al final se incluyen las reglas de negocio, abreviaturas e historial de revisión aplicables a todos los requerimientos.

---

<a id="indice"></a>

## 🧭 Índice

| Requerimiento | Nombre |
|:--|:--|
| [RF-ALT-01](#rf-alt-01) | Crear alerta |
| [RF-ALT-02](#rf-alt-02) | Consultar alertas |
| [RF-ALT-03](#rf-alt-03) | Editar alerta |
| [RF-ALT-04](#rf-alt-04) | Gestionar alerta |
| [RF-ALT-05](#rf-alt-05) | Notificar alerta |
| [RF-ALT-06](#rf-alt-06) | Terminar alerta |
| [Reglas de negocio](#reglas-de-negocio) | Aplican a todos los requerimientos |
| [Abreviaturas](#abreviaturas) | Glosario de siglas |
| [Historial de revisión](#historial-de-revision) | Cambios del documento |

---

<!-- ══════════════════════ COPIA ESTE BLOQUE POR CADA REQUERIMIENTO ══════════════════════ -->

## 📌 ANÁLISIS DE REQUERIMIENTOS · RF-ALT-01 <a id="rf-alt-01"></a>

### 1. Funcionalidad

| | |
|:--|:--|
| **Código** | RF-ALT-01 |
| **Nombre** | Crear alerta |
| **Descripción** | Permite registrar una nueva alerta con la información necesaria para su gestión posterior (tipo, alcance, instrucciones y destinatarios). La alerta puede ser creada directamente por el coordinador de emergencia desde la interfaz web, o solicitada como resultado secundario asociado al requerimiento funcional RF-SIM-01, al programar un evento. |
| **Actor principal** | Coordinador de emergencia, organizador de eventos, profesor, evento |
| **Precondiciones** | El coordinador de emergencias, el organizador de eventos o un profesor deben estar autenticados. Para la creación manual, debe tener el rol Coordinador de emergencia organizador de eventos o profesor. Para creación automática, la solicitud debe derivar del ciclo completo del requerimiento funcional RF-SIM-01, con un origen válido. Los edificios o zonas referenciados en el alcance deben existir según lo dispuesto en el requerimiento funcional RF-CAMP-01, y los usuarios o roles referenciados en los destinatarios deben existir según el requerimiento funcional RF-AUT-01. |

**Cómo se ejecutará**

- **Automáticamente:** El coordinador, organizador de eventos o profesor ingresa a la aplicación, se autentica, da clic en el botón de *Crear Evento* y diligencia el formulario de creación siguiendo el requerimiento RF-SIM-01. Una vez oprima el botón de guardar el evento, se emitirá una alerta automática que llegará a los usuarios participantes seleccionados inmediatamente a través del correo electrónico asociado, con la información del evento.
- **Manualmente:** El coordinador de emergencias ingresa a la aplicación, se autentica, despliega el menú hamburguesa, da click en el botón de alertas y al hacerlo se redirige a la pestaña de alertas, posteriormente da click en el botón *Crear Alerta* en la parte superior. Luego, será redirigido a la pestaña de creación de alertas donde encontrará un formulario que deberá diligenciar (título, fecha y hora, descripción, zonas y destinatarios). Durante la creación de la alerta, esta permanecerá en estado *borrador*, una vez creada pasará a estado programada y al llegar a la fecha para la cual se programó, pasará a estado activada.

### 2. Datos de entrada

| Nombre | Descripción | Tipo de campo | Reglas / Aplicación | Obligatorio |
|:--|:--|:--|:--|:--:|
| título | Nombre representativo de la alerta | Texto | No puede estar vacío. | Sí |
| fecha y hora de inicio | Fecha y hora en la que la alerta se activará y los destinatarios serán notificados. | Fecha y hora | Si no se asigna fecha queda en estado borrador, de lo contrario queda en estado programada. Si se desea que se active de inmediato, el sistema le asignará la hora de creación + 20 segundos (se creará como programada). | No |
| Fecha y hora de fin | Fecha y hora en la que la alerta se terminará y pasará a estado finalizado. | Fecha y hora | La fecha y hora de fin deben superiores a la fecha y hora de incio. | No |
| alcance | Define si la alerta se dirige a todo el campus o a edificios/zonas específicas. | Objeto (enum + lista de IDs) | Si el tipo es "Edificios específicos", la lista no puede estar vacía y los IDs deben existir según los requerimientos de campus RF-CAMP. | Sí |
| descripción | Texto con las recomendaciones o pasos a seguir que se mostrarán a los destinatarios. | Texto largo | Longitud máxima definida (por ejemplo, 1000 caracteres). | No |
| destinatarios | Lista de usuarios o roles que deben recibir la alerta (por ejemplo: todos los estudiantes, personal de un edificio). | Lista de IDs de usuario o de roles | Debe contener al menos un destinatario o rol válido, existente en MS01. | Sí |

### 3. Datos de salida

| Nombre | Descripción | Tipo de campo | Reglas / Aplicación | Obligatorio |
|:--|:--|:--|:--|:--:|
| idAlerta | Identificador único generado para la alerta. | Texto (UUID) | Generado automáticamente por el sistema. | Sí |
| estado | Estado inicial de la alerta. | Texto (enum) | Siempre "Borrador" al momento de crearse. Al confirmarse su creación | Sí |
| fechaCreacion | Fecha y hora de creación del registro. | Fecha/hora | Generada automáticamente por el sistema. | Sí |
| origen | Indica si la alerta fue creada manualmente o solicitada por el RF-EVE-01, y en este último caso el ID del evento asociado. | Objeto (enum + ID) | Si origen es "Evento", debe incluir el ID del evento correspondiente. | No (por defecto: manual) |

### 4. Flujo básico

| Paso | Actor | Descripción | Excepciones |
|:--:|:--|:--|:--|
| 1 | Coordinador \| Organizador \| Profesor | Ingresa a la aplicación, ingresa a la interfaz de creación de alertas y diligencia título, alcance, descripción y destinatarios. | - |
| 2 | Sistema | Valida que los campos obligatorios estén completos y que los IDs referenciados existan. | Si un campo obligatorio falta o un ID no existe, ver Flujo alterno A1. |
| 3 | Sistema | Genera el identificador de la alerta y la guarda en estado Borrador o en estado programada. | Si se desea una alerta instantánea, elegir una fecha y hora muy cercanas a la actual. |
| 4 | Sistema | Registra la acción de creación en el módulo de auditoría. | - |
| 5 | Sistema | Retorna al coordinador, organizador o profesor la confirmación con el título y el estado de la alerta creada, mediante un mensaje de notificación emergente. | - |

### 5. Flujo alterno

| Paso | Actor | Descripción | Excepciones |
|:--:|:--|:--|:--|
| A1 | Sistema | Detecta que uno o más campos obligatorios no fueron diligenciados o que un ID referenciado no existe. | Se rechaza la operación y se retorna un mensaje indicando el campo o campos inválidos. |
| A1 | Coordinador | Corrige la información señalada y reintenta el envío. | - |
| A2 | Requerimiento RF-EVE-01 | Envía una solicitud de creación de alerta indicando título, descripción, zonas y destinatarios. | - |
| A2 | Sistema | Aplica las mismas validaciones del flujo básico y crea la alerta asociándola al evento de origen. | Si los datos son incompletos, se rechaza y se informa el motivo al flujo del RF-EVE-01. |
| A2 | Sistema | Retorna a RF-EVE-01 el identificador de la alerta creada. | - |
| A2 | Sistema | Retorna al coordinador, organizador o profesor la confirmación con el título y el estado de la alerta creada. | |

<p align="right"><a href="#indice">⬆ Volver al índice</a></p>

---

<!-- ══════════════════════ FIN DEL BLOQUE DEL REQUERIMIENTO ══════════════════════ -->

## 📌 ANÁLISIS DE REQUERIMIENTOS · RF-ALT-02 <a id="rf-alt-02"></a>

### 1. Funcionalidad

| | |
|:--|:--|
| **Código** | RF-ALT-02 |
| **Nombre** | Consultar alertas |
| **Descripción** | Permite consultar la información completa de una alerta específica mediante su identificador, así como obtener un listado de las alertas registradas con filtros por estado y por rango de fechas, con paginación. |
| **Actor principal** | Estudiante, profesor, brigadista, administrador, personal administrativo, coordinación de emergencia, organizador de eventos o auditor |
| **Precondiciones** | El estudiante, profesor, brigadista, administrador, personal administrativo, coordinador de emergencia, organizador de eventos o auditor debe estar autenticado. Para el listado general se requiere el rol Coordinador de emergencia o Auditor/Observador. Para la consulta puntual de estado, cualquier módulo autenticado internamente puede consultarla. |

**Cómo se ejecutará**

- **Automáticamente:** -
- **Manualmente:** Cualquier usuario del sistema (estudiante, profesor, brigadista, administrador, personal administrativo, coordinación de emergencia, organizador de eventos o auditor) ingresa al sistema, se autentica, despliega el menú hamburguesa, da click en el botón de alertas y al hacerlo se redirige a la pestaña de alertas. Allí visualizará inicialmente las alertas activas y podrá filtrar por estado de alerta (Borrador, Programada, Activa, Finalizada, Archivada), según su rol (ver RN-10 y RN-11), oprimiendo el botón de filtro correspondiente cuyo nombre será el del estado de las alertas que desea visualizar. Así mismo, podrá escribir una palabra en la barra de búsqueda para filtrar por nombre (título de la alerta).

### 2. Datos de entrada

| Nombre | Descripción | Tipo de campo | Reglas / Aplicación | Obligatorio |
|:--|:--|:--|:--|:--:|
| estado | Filtro por estado de la alerta. | Texto (enum) | Valor perteneciente al catálogo de estados: Borrador, Programada, Activa, Finalizada, Archivada. | No |
| titulo | Nombre representativo de la alerta | Texto | No puede estar vacío | Sí |

### 3. Datos de salida

| Nombre | Descripción | Tipo de campo | Reglas / Aplicación | Obligatorio |
|:--|:--|:--|:--|:--:|
| listado (consulta general) | Lista de alertas con sus datos principales: título, fecha y hora, descripción, alcance, destinatarios. | Lista de objetos | Filtrada según los parámetros recibidos (estados o por título). | Sí |
| metadatosPaginacion | Información de paginación: total de registros y página actual. | Objeto | Se retorna junto con el listado. | Sí |

### 4. Flujo básico

| Paso | Actor | Descripción | Excepciones |
|:--:|:--|:--|:--|
| 1 | Estudiante, profesor, brigadista, administrador, personal administrativo, coordinación de emergencia, organizador de eventos o auditor | Ingresa a la aplicación, ingresa a la página de las alertas, y filtra las alertas por alguno de los estados a los que tenga acceso, oprimiendo un botón. | - |
| 2 | Sistema | Verifica cuáles son las alertas cuyo estado actual es el seleccionado. | Si no hay ninguna en el estado elegido muestra un mensaje indicando que no hay hay tareas en dicho estado. |
| 3 | Sistema | Retorna una lista de alertas con el estado predilecto. | - |

### 5. Flujo alterno

| Paso | Actor | Descripción | Excepciones |
|:--:|:--|:--|:--|
| A1 | Estudiante, profesor, brigadista, administrador, personal administrativo, coordinación de emergencia, organizador de eventos o auditor | Ingresa a la aplicación, ingresa a la página de las alertas, y escribe el título de la alerta en la barra de búsqueda y oprime en la lupa. | |
| A1 | Sistema | Busca las alertas cuyo titulo coincida con la consulta. | Si no existe una alerta con dicho nombre, retorna un mensaje indicando que no hay alertas que coincidan con la búsqueda. |
| A1 | Sistema | Retorna el listado de alertas con el título de consulta. | - |

<p align="right"><a href="#indice">⬆ Volver al índice</a></p>

---

## 📌 ANÁLISIS DE REQUERIMIENTOS · RF-ALT-03 <a id="rf-alt-03"></a>

### 1. Funcionalidad

| | |
|:--|:--|
| **Código** | RF-ALT-03 |
| **Nombre** | Editar alerta |
| **Descripción** | Permite modificar los datos de una alerta (tipo, alcance, instrucciones, destinatarios) mientras esta se encuentre en un estado que lo permita: Borrador o Programada. |
| **Actor principal** | Coordinador de emergencia, organizador de eventos, profesor. |
| **Precondiciones** | El actor debe estar autenticado con el rol Coordinador de emergencia, organizador de eventos o profesor. La alerta debe existir y encontrarse en estado Borrador o Programada. |

**Cómo se ejecutará**

- **Automáticamente:** El sistema recibe una solicitud de modificar la alerta asociada a un evento que está en edición, según el RF-EVE-3. Cuando se confirma la edición, se edita la alerta, modificando los datos de la misma que están asociados con el evento (título, descripción, alcance y destinatarios). Posteriormente se informa al usuario que la alerta asociada también fue modificada.
- **Manualmente:** El coordinador de emergencia, organizador de eventos o profesor se autentica, despliega el menú hamburguesa, oprime el botón de alertas y es redireccionado a la pantalla de las alertas. Filtra por los estados definidos como editables (borrador o programada, según RN-2) y oprime el ícono de edición. Luego accederá a un espacio de edición para modificar los valores que requiera. Al guardar las modificaciones se le informa al usuario que la alerta fue modificada.

### 2. Datos de entrada

| Nombre | Descripción | Tipo de campo | Reglas / Aplicación | Obligatorio |
|:--|:--|:--|:--|:--:|
| título | Nombre representativo de la alerta | Texto | No puede estar vacío. | Sí |
| fecha y hora de inicio | Fecha en la que la alerta se activará y los destinatarios serán notificados. | Fecha y hora | Si la alerta ya tiene una fecha, no se podrá guardar con una fecha vacía o anterior a la fecha-hora de edición. Si está vacía en el momento de la edición, sí se podrá guardar vacía, si se desea activar en el momento el sistema asigna como fecha la actual + 20 segundos. | No |
| fecha y hora de fin | Fecha y hora en la que la alerta se terminará y pasará a estado finalizado. | Fecha y hora | La fecha y hora de fin deben superiores a la fecha y hora de incio. | No |
| alcance | Define si la alerta se dirige a todo el campus o a edificios/zonas específicas. | Objeto (enum + lista de IDs) | Si el tipo es "Edificios específicos", la lista no puede estar vacía y los IDs deben existir según los requerimientos de campus RF-CAMP. | Sí |
| descripción | Texto con las recomendaciones o pasos a seguir que se mostrarán a los destinatarios. | Texto largo | Longitud máxima definida (por ejemplo, 1000 caracteres). | No |
| destinatarios | Lista de usuarios o roles que deben recibir la alerta (por ejemplo: todos los estudiantes, personal de un edificio). | Lista de IDs de usuario o de roles | Debe contener al menos un destinatario o rol válido, existente en MS01. | Sí |

### 3. Datos de salida

| Nombre | Descripción | Tipo de campo | Reglas / Aplicación | Obligatorio |
|:--|:--|:--|:--|:--:|
| estado | Estado vigente de la alerta luego de la edición. | Texto (enum) | No cambia como resultado de esta operación. | Sí |
| fechaActualizacion | Fecha y hora en que se aplicó el cambio. | Fecha/hora | Generada automáticamente por el sistema para registro auditable. | Sí |

### 4. Flujo básico

| Paso | Actor | Descripción | Excepciones |
|:--:|:--|:--|:--|
| 1 | Coordinador, organizador, profesor | Selecciona la alerta a modificar y edita los campos requeridos (todos los campos visibles son editables). | - |
| 2 | Sistema | Verifica que la alerta se encuentre en un estado editable (Borrador o Programada). | Si el estado no lo permite, ver Flujo alterno A1. |
| 3 | Sistema | Valida los datos ingresados con las mismas reglas de la creación. | Si los datos son inválidos, ver Flujo alterno A2. |
| 4 | Sistema | Guarda los cambios y registra la fecha de actualización. | - |
| 5 | Sistema | Registra la acción de edición en el módulo de auditoría. | - |

### 5. Flujo alterno

| Paso | Actor | Descripción | Excepciones |
|:--:|:--|:--|:--|
| A1 | Sistema | Detecta que la alerta se encuentra en un estado distinto de Borrador o Programada (Activa, Finalizada o Archivada). | Se rechaza la operación informando que la alerta no puede modificarse en su estado actual. |
| A2 | Sistema | Detecta datos inválidos en los campos enviados | Se rechaza la operación indicando el campo o campos inválidos. |

<p align="right"><a href="#indice">⬆ Volver al índice</a></p>

---

## 📌 ANÁLISIS DE REQUERIMIENTOS · RF-ALT-04 <a id="rf-alt-04"></a>

### 1. Funcionalidad

| | |
|:--|:--|
| **Código** | RF-ALT-04 |
| **Nombre** | Gestionar alerta |
| **Descripción** | Administra los estados por los que atraviesa una alerta (Borrador, Programada, Activa, Finalizada, Archivada) y las acciones que producen cada transición: programar, activar (manual o automáticamente al llegar la fecha y hora establecidas), finalizar (manual o automáticamente) y archivar (generar el reporte de auditoría). |
| **Actor principal** | Coordinador de emergencia, organizador de eventos o profesor (para alertas creadas o terminadas manualmente) / Sistema - Scheduler interno (para flujo entre estados Activo, Terminada y Archivada, según programación). |
| **Precondiciones** | Para las acciones manuales, el actor debe estar autenticado con el rol Coordinador de emergencia, organizador de eventos o profesor. La alerta debe existir y encontrarse en el estado previo válido para la transición solicitada según RN-1. |

**Cómo se ejecutará**

- **Automáticamente:** Una vez llegue la hora de programación, la alerta pasará a Activa. Una vez lleguen la fecha y hora de terminación de la alerta, el sistema cambiará la alerta a estado Terminada (un coordinador, organizador o profesor puede terminar la alerta antes de que llegue la fecha de terminación mediante el botón correspondiente de Terminado), después se hará el registro de la alerta y esta pasará a estado archivada, siendo este último el estado final.
- **Manualmente:** El coordinador de emergencias, organizador de eventos o profesor crean la alerta (RF-ALT-01), si la alerta tiene una fecha de programación se creará en estado programada, de lo contrario, se creará en estado borrador y pasará a estado programada una vez se edite (RF-ALT-03) y se añada una fecha de programación.

### 2. Datos de entrada

| Nombre | Descripción | Tipo de campo | Reglas / Aplicación | Obligatorio |
|:--|:--|:--|:--|:--:|
| título | Nombre representativo de la alerta | Texto | No puede estar vacío. | Sí |
| estado | Filtro por estado de la alerta. | Texto (enum) | Valor perteneciente al catálogo de estados: Borrador, Programada, Activa, Finalizada, Archivada. | No |
| fecha y hora inicio | Fecha y hora en que debe activarse la alerta. | Fecha/hora | Debe ser posterior a la fecha y hora actuales. | Sí (solo para la acción "programar") |
| fecha y hora fin | Fecha y hora en que debe finalizarse la alerta. | Fecha/Hora | Debe ser posterior a la fecha de activación | Sí |

### 3. Datos de salida

| Nombre | Descripción | Tipo de campo | Reglas / Aplicación | Obligatorio |
|:--|:--|:--|:--|:--:|
| idAlerta | Identificador de la alerta. | Texto (UUID) | - | Sí |
| estadoAnterior | Estados antes de aplicar la transición. | Texto (enum) | - | Sí |
| estadoNuevo | Estados después de aplicar la transición. | Texto (enum) | Deben corresponder a una transición válida con respecto al estadoAnterior | Sí |
| fechaCambio | Fecha y hora en que ocurrió la transición. | Fecha/hora | Generada automáticamente por el sistema. | Sí |

### 4. Flujo básico

| Paso | Actor | Descripción | Excepciones |
|:--:|:--|:--|:--|
| 1 | Coordinador, organizador, profesor | Crea la alerta siguiendo RF-ALT-01. | - |
| 2 | Sistema | Verifica si la alerta creada tiene fecha-hora futura. Si la tiene, la crea en estado programada, de lo contrario la crea en estado borrador. | Si la fecha no es futura, ver Flujo alterno A4. |
| 3 | Sistema | Cambia el estado de la alerta a Programada. | - |
| 4 | Coordinador, organizador, profesor | Edita la alerta | - |
| | Sistema | Verifica que la alerta esté en estado borrador o programado y que las modificaciones sigan la descripción del RF-ALT-03; de hacerlo, guarda los cambios. | Si la alerta ya existe con una fecha asignada y se edita a una fecha vacía, se envía un error solicitando no dejar el campo de fecha vacío. |
| 5 | Sistema | Al llegar la fecha de programación, cambia el estado de la alerta a Activa y dispara el envío de notificaciones (RF-ALT-05). | - |
| 7 | Sistema | Al llegar la fecha de finalización cambia la alerta a Finalizada. | Si un coordinador, organizador o profesor finalizan la alerta antes, la fecha de finalización se pone como nula y se pasa a Finalizada. |
| 6 | Sistema | Registra la transición en el módulo de auditoría y pasa la alerta a estado Archivada. | - |

### 5. Flujo alterno

| Paso | Actor | Descripción | Excepciones |
|:--:|:--|:--|:--|
| A1 | Sistema (Scheduler) | Revisa periódicamente las alertas en estado Programada y detecta que la fecha y hora de activación de una de ellas ya se cumplió. | - |
| A1 | Sistema | Cambia el estado de la alerta a Activa y dispara el envío de notificaciones. | - |
| A2 | Coordinador o MS06 | Solicita finalizar una alerta que se encuentra en estado Activa. | - |
| A2 | Sistema | Verifica que la alerta esté en estado Activa y cambia su estado a Finalizada. | Si la alerta ya está finalizada, se confirma el estado sin generar error. |
| A3 | Coordinador | Solicita archivar una alerta que se encuentra en estado Finalizada. | - |
| A3 | Sistema | Verifica que la alerta esté en estado Finalizada, cambia su estado a Archivada e impide futuras modificaciones. | - |
| A4 | Sistema | Detecta que la transición solicitada no corresponde al estado actual de la alerta (por ejemplo, activar una alerta ya finalizada). | Se rechaza la operación con un error controlado. |

<p align="right"><a href="#indice">⬆ Volver al índice</a></p>

---

## 📌 ANÁLISIS DE REQUERIMIENTOS · RF-ALT-05 <a id="rf-alt-05"></a>

### 1. Funcionalidad

| | |
|:--|:--|
| **Código** | RF-ALT-05 |
| **Nombre** | Notificar alerta |
| **Descripción** | Envía notificaciones vía correo electrónico mediante **Resend** a los destinatarios definidos cuando una alerta pasa a estado Activa, y permite que cada destinatario confirme que la recibió. |
| **Actor principal** | Sistema (envío de notificación) / Usuario destinatario con cualquier rol del sistema (confirmación) |
| **Precondiciones** | La alerta debe estar en estado Activa. Para confirmar la recepción, el usuario debe estar autenticado y ser parte de los destinatarios definidos para esa alerta. |

**Cómo se ejecutará**

- **Automáticamente:** Una vez la alerta pasa a estado activo (llega la fecha y hora de programación), se retoma el listado de usuarios definidos como destinatarios y sus respectivos tokens, y, mediante Resend, se envía una notificación al correo de cada uno; luego se registra el resultado del envío.
- **Manualmente:** Una vez el destinatario recibe el correo, podrá acceder al aplicativo, y en la pantalla principal podrá notificar que recibió la alerta. La confirmación se guardará con fecha y hora.

### 2. Datos de entrada

| Nombre | Descripción | Tipo de campo | Reglas / Aplicación | Obligatorio |
|:--|:--|:--|:--|:--:|
| destinatarios | Información necesaria para el envío, tomada de la alerta y de los tokens de dispositivo registrados por cada usuario. | Texto (UUID) | Se obtiene de datos ya almacenados; no es un dato ingresado manualmente. | Sí (generado internamente) |
| tokens de dispositivo | Identificador del usuario que confirma, obtenido del token JWT. | Interno del sistema | El usuario debe corresponder a un destinatario asociado a la alerta. | Sí |
| confirmacion | Identificador de la alerta que se confirma, tomado de la URL. | Booleano | Debe corresponder a una alerta en estado Activa. | Sí |

### 3. Datos de salida

| Nombre | Descripción | Tipo de campo | Reglas / Aplicación | Obligatorio |
|:--|:--|:--|:--|:--:|
| resultadoEnvio | Resultado del envío por destinatario: enviado o fallido, usado para el seguimiento interno de entregas. | Lista de objetos | Generado internamente por el sistema. | Sí |
| confirmacion | Registro de la confirmación de recepción con fecha y hora. | Objeto | Único por usuario y alerta. | Sí |

### 4. Flujo básico

| Paso | Actor | Descripción | Excepciones |
|:--:|:--|:--|:--|
| 1 | Sistema | Al activarse la alerta, obtiene la lista de destinatarios y sus tokens de dispositivo. | - |
| 2 | Sistema | Envía la notificación a cada destinatario mediante correo electrónico. | Si el envío a un destinatario falla, ver Flujo alterno A1. |
| 3 | Sistema | Registra el resultado de cada envío. | - |
| 4 | Usuario destinatario | Recibe la notificación en su correo y consulta la alerta. | - |
| 5 | Usuario destinatario | Confirma que recibió la alerta desde la aplicación. | - |
| 6 | Sistema | Registra la confirmación de recepción con fecha y hora. | Si ya existía una confirmación previa, ver Flujo alterno A2. |

### 5. Flujo alterno

| Paso | Actor | Descripción | Excepciones |
|:--:|:--|:--|:--|
| A1 | Sistema | Detecta que el envío a un destinatario específico falló (por ejemplo, token de dispositivo inválido o servicio de Resend). | Registra el fallo sin afectar el estado de la alerta ni el envío a los demás destinatarios. |
| A2 | Sistema | Detecta que el usuario ya había confirmado previamente la recepción de esa alerta. | No duplica el registro y retorna la confirmación ya existente. |
| A3 | Sistema | Detecta que la alerta indicada no se encuentra en estado Activa. | Rechaza la confirmación indicando que la alerta no está vigente. |

<p align="right"><a href="#indice">⬆ Volver al índice</a></p>

---

## 📌 ANÁLISIS DE REQUERIMIENTOS · RF-ALT-06 <a id="rf-alt-06"></a>

### 1. Funcionalidad

| | |
|:--|:--|
| **Código** | RF-ALT-06 |
| **Nombre** | Terminar alerta |
| **Descripción** | Permite pasar una alerta a estado terminado, una vez finaliza la situación de contingencia. |
| **Actor principal** | Coordinador de emergencia, organizador de eventos o profesor |
| **Precondiciones** | La alerta debe existir y estar en estado activa, mientras que el coordinador, organizador o profesor deben estar autenticados. |

**Cómo se ejecutará**

- **Automáticamente:** **Automáticamente 1:** El coordinador, organizador de eventos o profesor ingresa a la aplicación, se autentica, ingresa ala sección de eventos, filtra por activos y oprime el botón de cancelar el evento, la alerta asociada se terminará automáticamente, pasando a estado Finalizada. **Automáticamente 2:** El sistema identifica que ha llega la fecha-hora que se definió para que se termine una alerta, por lo que la pasa de Activa a Finalizada.
- **Manualmente:** El coordinador de emergencia, organizador de eventos o profesor se autentica, despliega el menú hamburguesa, oprime el botón de alertas y es redireccionado a la pantalla de las alertas. Filtra por los estados definidos para pasar a Terminado (Activas, según RN-1) y oprime el ícono de Terminar. Luego la alerta cambiará a la sección que filtra por terminadas e internamente, una vez se archive su finalización, pasará a estado Archivada.

### 2. Datos de entrada

| Nombre | Descripción | Tipo de campo | Reglas / Aplicación | Obligatorio |
|:--|:--|:--|:--|:--:|
| estado | Finalizada, una vez se notifica su terminación, luego a Archivada. | Texto (enum) | Valor perteneciente al catálogo de estados: Finalizada, Archivada. | No |

### 3. Datos de salida

| Nombre | Descripción | Tipo de campo | Reglas / Aplicación | Obligatorio |
|:--|:--|:--|:--|:--:|
| confirmación de eliminación | Confirmación del cambio de estado de la alerta a Finalizada. | Objeto | - | Sí (solo en la solicitud de finalización) |

### 4. Flujo básico

| Paso | Actor | Descripción | Excepciones |
|:--:|:--|:--|:--|
| 1 | Coordinador, organizador, profesor | Selecciona la alerta a modificar y da click en el botón de Finalizar. | - |
| 2 | Sistema | Verifica que la alerta se encuentre en un estado Activo; de estarlo, cambia el estado de la alerta a Finalizada. | Si el estado no lo permite, ver Flujo alterno A1. |
| 4 | Sistema | Guarda los cambios y registra la fecha de finalización. | - |
| 5 | Sistema | Registra la acción de edición en el módulo de auditoría. | - |

### 5. Flujo alterno

| Paso | Actor | Descripción | Excepciones |
|:--:|:--|:--|:--|
| A1 | Sistema | Detecta que la alerta asociada ya se encuentra en estado Finalizada o está en otro estado no permitido. | Informa que la alerta ya fue finalizada. |
| A2 | Sistema | Recibe del flujo de RF-EVE-03 la solicitud de terminar la alerta asociada al evento. | Si las alertas no están disponibles en el momento, deberá pasar a estado Finalizado de forma manual una vez el servicio se reestablezca. |
| A2 | Sistema | Pasa la alerta de Activa a Finalizada. | Si ya está Finalizada o Archivada ver flujo alterno A1 |
| A3 | Sistema | Detecta que la fecha-hora actual corresponde a la fecha-hora de fin de la alerta y pone la alerta como Finalzada. | - |

<p align="right"><a href="#indice">⬆ Volver al índice</a></p>

---

## 📜 REGLAS DE NEGOCIO <a id="reglas-de-negocio"></a>

| No. | Descripción |
|:--:|:--|
| 1 | Una alerta solo puede transitar entre estados siguiendo el flujo: Borrador → Programada → Activa → Finalizada → Archivada. No se permiten saltos ni retrocesos. |
| 2 | Una alerta solo puede editarse (RF-ALT-03) mientras se encuentra en estado Borrador o Programada. |
| 3 | Una alerta en estado Archivada no puede modificarse en ninguna circunstancia. |
| 4 | La activación automática de una alerta Programada debe ejecutarse dentro de una tolerancia definida respecto a la fecha y hora establecidas. |
| 5 | Activar una alerta que ya se encuentra activa no debe generar un nuevo envío de notificaciones (idempotencia). |
| 6 | Un usuario destinatario solo puede registrar una confirmación de recepción por alerta. |
| 7 | Toda acción relevante sobre una alerta (creación, edición, cambio de estado) debe quedar registrada en el módulo de auditoría con el usuario o servicio responsable, la acción y la fecha/hora. |
| 8 | Ningún microservicio distinto de MS03 puede acceder directamente a la base de datos de alertas; toda interacción debe realizarse mediante los endpoints expuestos. |
| 9 | Las solicitudes provenientes de MS06 deben incluir el identificador del evento de origen, para mantener la trazabilidad entre ambos servicios. |
| 10 | Los siguientes roles podrán filtrar las alertas por todos los estados definidos en la RN-1: administrador, coordinador de emergencias, organizador de eventos, profesor. |
| 11 | Los siguientes roles solo podrán filtrar las alertas por los estados Activa, Finalizada y Activada: estudiante, brigadista, personal administrativo y auditor. |

<p align="right"><a href="#indice">⬆ Volver al índice</a></p>

---

## 🔤 ABREVIATURAS <a id="abreviaturas"></a>

| Abreviatura | Significado |
|:--|:--|
| RF | Requerimiento Funcional |
| RN | Regla de negocio |
| IDE | Módulo asociado a la autenticación |
| CAM | Módulo asociado al campus y seguridad |
| ALE | Módulo asociado a las alertas|
| CHI | Módulo asociado al check-in|
| BEI | Módulo asociado a los brigadistas e incidentes|
| EVE | Módulo asociado a Eventos |
| FIA | Módulo asociado a la formación y asistente de IA|
| AYA | Módulo asociado a la analítica y auditoría|
| JWT | JSON Web Token |
| API | Interfaz de Programación de Aplicaciones (Application Programming Interface) |
| REST | Representational State Transfer |
| UUID | Identificador Único Universal (Universally Unique Identifier) |
| ALT | Módulo asociado a Alertas y emergencias |
| FCM | Firebase Cloud Messaging |

<p align="right"><a href="#indice">⬆ Volver al índice</a></p>

---

## 🕓 HISTORIAL DE REVISIÓN <a id="historial-de-revision"></a>

| Elaborado por | Revisado por | Fecha | Descripción y justificación de cambios |
|:--|:--|:--:|:--|
| Equipo Zephyros | Rodrigo Gualtero | 27/09/2026 | Versión inicial del documento de análisis de requerimientos de MS03 - Alertas y Emergencias. |
| Equipo Zephyros | Pendiente | 05/10/2026 | Agrupación de los requerimientos de front y análisis de nuevos flujos alternos. |
| Equipo Zephyros | Rodrigo Gualtero | 15/10/2026 | Ajustes en comentarios |

<p align="right"><a href="#indice">⬆ Volver al índice</a></p>
