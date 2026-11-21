<h1 align="center">🛡️ Campus Seguro</h1>
<h3 align="center">MS06 · Eventos</h3>

<p align="center"><em>Análisis de requerimientos funcionales, reglas de negocio e historial de revisión.</em></p>

<p align="center">
  <img alt="Curso" src="https://img.shields.io/badge/Curso-DOSW%202026--2-1f6feb">
  <img alt="Squad" src="https://img.shields.io/badge/Squad-Zephyros-8957e5">
  <img alt="Documento" src="https://img.shields.io/badge/Documento-Requerimientos-d29922">
  <img alt="Versión" src="https://img.shields.io/badge/Versi%C3%B3n-1.0-2da44e">
</p>

<p align="center">
  <a href="#rf-eve-01">📌 Requerimientos</a> ·
  <a href="#reglas-de-negocio">📜 Reglas de negocio</a> ·
  <a href="#abreviaturas">🔤 Abreviaturas</a> ·
  <a href="#historial-de-revision">🕓 Historial</a> ·
  <a href=https://github.com/CampusSeguro-DOSW-2026-2/ms06-drills.git>📦 Repositorio</a> ·
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

Este documento detalla cada uno de los requerimientos funcionales del microservicio **MS06 - Eventos**, incluyendo su funcionalidad, datos de entrada y salida, flujo básico y flujo alterno. Al final se incluyen las reglas de negocio, abreviaturas e historial de revisión aplicables a todos los requerimientos.

---

<a id="indice"></a>

## 🧭 Índice

| Requerimiento | Nombre |
|:--|:--|
| [RF-EVE-01](#rf-eve-01) | Crear y configurar evento. |
| [RF-EVE-02](#rf-eve-02) | Editar evento |
| [RF-EVE-03](#rf-eve-03) | Consultar y monitorear evento |
| [RF-EVE-04](#rf-eve-04) | Finalizar evento |
| [RF-EVE-05](#rf-eve-05) | Habilitar panel del evento |
| [RF-BEI-06](#rf-bei-06) | Auditoría |
| [Reglas de negocio](#reglas-de-negocio) | Aplican a todos los requerimientos |
| [Abreviaturas](#abreviaturas) | Glosario de siglas |
| [Historial de revisión](#historial-de-revision) | Cambios del documento |

---

<!-- ══════════════════════ COPIA ESTE BLOQUE POR CADA REQUERIMIENTO ══════════════════════ -->

## 📌 ANÁLISIS DE REQUERIMIENTOS · RF-EVE-01 <a id="rf-eve-01"></a>

### 1. Funcionalidad

| | |
|:--|:--|
| **Código** | RF-EVE-01 |
| **Nombre** | Crear y configurar evento. |
| **Descripción** | El sistema debe permitir al coordinador de emergencias (eventos reales) y al organizador de simulacros (simulaciones de prueba) crear un evento registrando su información básica (nombre, tipo, descripción y fecha y hora de inicio) y configurando, en el mismo flujo, su alcance (los edificios y zonas del campus donde se realizará el evento), sus participantes (los tipos de usuario convocados: estudiantes, profesores o personal administrativo) y sus responsables (brigadistas o coordinadores de emergencia que lo dirijan). |
| **Actor principal** | Coordinador de emergencias y organizador de simulacros. |
| **Precondiciones** | El usuario debe estar autenticado y contar con permisos de coordinador de emergencia u organizador de simulacros, validados AUT. Los edificios y zonas a registrar deben existir y estar activos según CAM. ALE debe estar disponible para enviar las alertas correspondientes. Los usuarios que se registren como brigadistas deben estar activos y tener dicho rol. |

**Cómo se ejecutará**

- **Automáticamente:** Al darle crear se generará automáticamente una alerta administrativa solicitando a los brigadistas que, de ser posible, se activen y seleccionen una zona que atenderán. Una vez llegada la fecha y hora del evento se emitirá una alerta general notificando a los participantes el inicio del evento.
- **Manualmente:** El coordinador de emergencias o el organizador de simulacros ingresa a la aplicación, se autentica, despliega el menú hamburguesa y selecciona la opción de eventos. Luego, accede al panel de gestión de eventos y da click en el botón de crear, seguidamente será redireccionado al entorno para la creación de un evento donde deberá diligenciar nombre, tipo, descripción, fecha de inicio, edificios o zonas y participantes.

### 2. Datos de entrada

| Nombre | Descripción | Tipo de campo | Reglas / Aplicación | Obligatorio |
|:--|:--|:--|:--|:--:|
| Nombre del evento | Nombre con el que se identifica el evento. | Texto | No puede estar vacío. | Sí |
| Tipo | Categorías del evento (puede ser: Simulacro, Sismo, Incendio, Disturbios, Inundación, Fuga de Sustancias Químicas, Otro) | Texto | Pertenecen al catálogo predefinido de tipos y se pueden añadir nuevos tipos. Un simulacro puede tener varios tipos. | Sí |
| Descripción | Explicación del objetivo y contexto del evento. | Texto | Puede estar vacío. | No |
| Fecha y hora de inicio | Momento en que el evento debe comenzar. | Fecha y hora | Debe ser posterior al momento actual y anterior a la finalización. Si este campo está vacío el evento se guarda en estado Borrador, de lo contrario se guarda como Programado. | No |
| Alcance (edificios y zonas) | Edificios y zonas del campus donde se realizará el evento. | Selección múltiple | Solo edificios y zonas activos en CAM. | No |
| Participantes | Tipos de usuario convocados al evento. | Selección múltiple | Roles de usuario definidos en AUT. | No |

### 3. Datos de salida

| Nombre | Descripción | Tipo de campo | Reglas / Aplicación | Obligatorio |
|:--|:--|:--|:--|:--:|
| idEvento | Identificador único generado para el evento. | Texto (UUID) | Generado automáticamente por el sistema. | Sí |
| estado | Estado inicial de la alerta. | Texto | "Borrador" al momento de crearse y si los campos de entrada no están completos. "Programado" al estar diligenciados todos los campos. | Sí |
| fechaCreacion | Fecha y hora de creación del registro. | Fecha/hora | Generada automáticamente por el sistema. | Sí |
| brigadistas | Listado de los brigadistas por zona que confirmaron su participación. | Lista de objetos | Se va completando a medida que los brigadistas se van activando y seleccionando su zona de participación. | No |
| alertaBrigadistas | Alerta que insta a los brigadistas a activarse y participar en una zona asignada. | Notificación. | Se activa una vez el evento esté en estado Programado. | Sí |
| alertaParticipantes | Alerta que notifica a los participantes el inicio de un evento. | Notificación | Se activa a la misma hora de inicio del evento. | Sí |

### 4. Flujo básico

| Paso | Actor | Descripción | Excepciones |
|:--:|:--|:--|:--|
| 1 | Coordinador u Organizador | Ingresa a la aplicación, se autentica, ingresa a la sección de eventos e inicia la creación de uno nuevo. Registra el nombre, el tipo y opcionalmente la descripción. | Si el módulo AUT está disponible, el sistema no puede validar los permisos del usuario, no permite crear ni configurar el evento e informa que intente nuevamente más tarde. |
| 2 | Coordinador u Organizador | Selecciona, entre los edificios y zonas registrados en CAM, los que conformarán el alcance del evento. | Revisar A1 |
| 3 | Coordinador u Organizador | Selecciona los tipos de usuario que participarán. | Revisar A2 |
| 4 | Sistema | Valida que la información obligatoria esté completa y sea consistente. | FA-6 |
| 5 | Sistema | Registra el evento en estado Borrador si los datos no están completos, de lo contrario lo guarda en estado Programado. | - |
| 6 | Sistema | Una vez el evento pase a estado Programado, emite una alerta administrativa a los brigadistas para que se activen y seleccionen una zona donde apoyar. | Si ALE o BEI no están disponibles el evento se guarda en estado Borrador. |
| 7 | Brigadista | Recibe la alerta, activa su estado y selecciona una zona donde apoyar. | - |
| 8 | Sistema | Recibe las autoasignaciones de los brigadistas y los guarda por zona, asociándolos al evento programado. | Si el módulo BEI no está disponible, el sistema permite guardar la información básica y el escenario, y deja pendiente la asignación de responsables para completarla posteriormente. |
| 9 | Sistema | Revisa si la fecha de programación llegó y una vez concuerde con la fecha actual envía una alerta general a los participantes. | - |

### 5. Flujo alterno

| Paso | Actor | Descripción | Excepciones |
|:--:|:--|:--|:--|
| A1 | Sistema | Si CAM no está disponible, el sistema permite guardar la información básica del evento y deja pendiente la configuración del escenario (alcance) para completarla posteriormente. | El evento permanece en Borrador y no puede programarse hasta completar el escenario. |
| A2 | Sistema | Si AUT no está disponible, el sistema permite guardar la información básica y el escenario, y deja pendiente la asignación de participantes. | El evento permanece en Borrador y no puede programarse hasta asignar los responsables. |
| A3 | Sistema | Si la fecha y hora de inicio es anterior a la actual, el sistema rechaza el registro de las fechas y solicita su corrección. | Las fechas no se registran correctamente. |

<p align="right"><a href="#indice">⬆ Volver al índice</a></p>

---

<!-- ══════════════════════ FIN DEL BLOQUE DEL REQUERIMIENTO ══════════════════════ -->

## 📌 ANÁLISIS DE REQUERIMIENTOS · RF-EVE-02 <a id="rf-eve-02"></a>

### 1. Funcionalidad

| | |
|:--|:--|
| **Código** | RF-EVE-02 |
| **Nombre** | Editar evento |
| **Descripción** | El sistema debe permitir al coordinador u organizador editar un evento para programarlo (añadir fecha de inicio) o modificar sus especificaciones (título, descripción, tipo, alcance, participantes). Solo se podrán editar eventos en estado Borrador, Programado y En ejecución. Cuando se edite un simulacro que esté en ejecución se deberán enviar nuevas alertas a los brigadistas y a los participantes del evento con la información actualizada. |
| **Actor principal** | Coordinador de emergencias y organizador de simulacros. |
| **Precondiciones** | El usuario debe autenticarse y tener el rol de coordinador u organizador. Los módulos ALE y CHI deben estar disponibles para la reprogramación de las alertas y habilitación del check-in respectivamente. El módulo BEI debe estar disponible para confirmar la autoasignación a la zona inicialmente indicada. |

**Cómo se ejecutará**

- **Automáticamente:** Una vez se guarden los cambios se va a activar una alerta automática dirigida a los brigadistas pidiendo que verifiquen su asignación tras los cambios (en esa alerta deben poder cambiar de asignación, desactivarse o retirar la asignación). Así mismo se editará la alerta previamente creada con la creación de la alerta, modificando su fecha de programación.
- **Manualmente:** El coordinador u organizador ingresa a la aplicación, se autentica, despliega el menú hamburguesa y selecciona la opción de eventos. Luego, accede al panel de gestión de eventos y da click sobre el icono de edición del evento de interés. Luego será redirigido a un entorno de edición donde podrá modificar cualquiera de los campos del evento.

### 2. Datos de entrada

| Nombre | Descripción | Tipo de campo | Reglas / Aplicación | Obligatorio |
|:--|:--|:--|:--|:--:|
| Nombre del evento | Nombre con el que se identifica el evento. | Texto | No puede estar vacío. | Sí |
| Tipo | Categorías del evento (puede ser: Simulacro, Sismo, Incendio, Disturbios, Inundación, Fuga de Sustancias Químicas, Otro) | Texto | Pertenecen al catálogo predefinido de tipos y se pueden añadir nuevos tipos. Un simulacro puede tener varios tipos. | Sí |
| Descripción | Explicación del objetivo y contexto del evento. | Texto | Puede estar vacío. | No |
| Fecha y hora de inicio | Momento en que el evento debe comenzar. | Fecha y hora | Debe ser posterior al momento actual y anterior a la finalización. Si este campo está vacío el evento se guarda en estado Borrador, de lo contrario se guarda como Programado. | No |
| Alcance (edificios y zonas) | Edificios y zonas del campus donde se realizará el evento. | Selección múltiple | Solo edificios y zonas activos en CAM. | No |
| Participantes | Tipos de usuario convocados al evento. | Selección múltiple | Roles de usuario definidos en AUT. | No |

### 3. Datos de salida

| Nombre | Descripción | Tipo de campo | Reglas / Aplicación | Obligatorio |
|:--|:--|:--|:--|:--:|
| estado | Estado en que queda el evento después de editarlo, puede ser Borrador o Programado. | Texto | Queda en Programado si incluye una fecha de programación, de lo contrario queda en Borrador. | Sí |
| mensaje_confirmación | Aviso de que el evento quedó programado, con la alerta asociada y la programación comunicada a los demás servicios. | Texto | Enviarlo si los módulos ALE y BEI estén disponibles. | Sí |
| Mensaje_error | Motivo por el cual no se pudo completar la programación. | Texto | Se muestra solo cuando la programación falla. | No |

### 4. Flujo básico

| Paso | Actor | Descripción | Excepciones |
|:--:|:--|:--|:--|
| 1 | Coordinador, Organizador | Ingresa al aplicativo, se autentica, accede al entorno de edición de un evento en específico. | - |
| 2 | Sistema | Verifica que el evento esté en estado Borrador, Programado o En ejecución. | Si el evento está en estado Finalizado o Evaluado notifica que el evento no se pude editar. |
| 3 | Coordinador, Organizador | Modifica los campos de su interés | - |
| 4 | Sistema | Verifica que el evento esté en estado borrador. | Si está en estado Programado o En Ejecución, revisar A1. Si está en estado Finalizado o Evaluado indicar que la acción no está permitida. |
| 5 | Sistema | Verifica que todos los campos solicitados para la programación estén diligenciados y sean válidos. | Si hay un campo invalido, indicar al usuario. Si hay campos obligatorios para la activación vacíos, revisar A2. |
| 6 | Sistema | Envía al módulo CHI la fecha y hora de inicio y el escenario del evento, para que conozca cuándo y dónde se habilitará el check-in. | Si el módulo CHI no está disponible o no confirma la recepción de la programación, no se completa la programación y se informa el error. |
| 7 | Sistema | Solicita a ALE que envíe una alerta a los brigadistas confirmando su asignación tras los cambios. | Si el módulo ALE no está disponible o no confirma la solicitud, no se completa la programación y se informa el error. |
| 8 | Sistema | Solicita a ALE que cree una alerta y la programa para la misma fecha y hora del evento. | |
| 9 | Sistema | Cambia el estado del evento a Programado. | - |
| 10 | Sistema | Al llegar la fecha y hora de inicio, cambia automáticamente el estado del evento a En ejecución. | - |
| 11 | Sistema | ALE activa la alerta y CHI habilita el check-in para los participantes del escenario, hasta la finalización. | Si CHI no está disponible al iniciar el evento revisar A3.<br><br>Si ALE no está disponible al iniciar el evento revisar A4. |

### 5. Flujo alterno

| Paso | Actor | Descripción | Excepciones |
|:--:|:--|:--|:--|
| A1 | Sistema | Identifica que se modificó la fecha de programación del evento. | La fecha de programación siempre debe ser superior a las fechas de creación y modificación |
| A1 | Sistema | Guarda los cambios realizados al evento. | |
| A1 | Sistema | Se comunica con el módulo ALE para enviar una nueva alerta a los brigadistas confirmando sus asignaciones tras los cambios y para reprogramar la alerta general. | Si el módulo ALE no está disponible no se guardan los cambios y se notifica al usuario. |
| A1 | Sistema | Se comunica con el módulo CHI para reprogramar la hora de inicio del check-in asociado al evento. | Si el módulo CHI no está disponible no se guardan los cambios y se notifica al usuario. |
| A1 | Sistema | Retoma el flujo básico desde el paso. | - |
| A2 | Sistema | Identifica que faltan campos por diligenciar que son necesarios para que el evento pueda pasar a estado Programado. | - |
| A2 | Sistema | Actualiza la información del evento. | - |
| A2 | Sistema | Guarda el evento en estado Borrador. | - |
| A2 | Sistema | Indica al usuario que una vez diligencie todos los campos con información válida, el evento se programará | - |
| A3 | Sistema | Si al llegar la fecha y hora de inicio el microservicio MS04 – Evacuación y check-in ya no está disponible (aunque sí lo estaba al programar), el sistema inicia el evento de todos modos y registra que no fue posible habilitar el check-in, para que esa falla se revise en la evaluación del evento. | El evento pasa igualmente a En ejecución. |
| A4 | Sistema | Si al llegar la fecha y hora de inicio el microservicio MS03 – Alertas y emergencias no logra activar la alerta, el sistema inicia el evento de todos modos, registra la falla y avisa al coordinador para que la gestione. | El evento pasa igualmente a En ejecución. |

<p align="right"><a href="#indice">⬆ Volver al índice</a></p>

---

## 📌 ANÁLISIS DE REQUERIMIENTOS · RF-EVE-03 <a id="rf-eve-03"></a>

### 1. Funcionalidad

| | |
|:--|:--|
| **Código** | RF-EVE-03 |
| **Nombre** | Consultar y monitorear evento |
| **Descripción** | El sistema debe permitir consultar por título y estado los eventos registrados, visualizando como mínimo su nombre, fecha, estado, alcance y responsables. |
| **Actor principal** | Coordinador de emergencia u organizador de simulacros. |
| **Precondiciones** | El usuario debe autenticarse y tener el rol de coordinador u organizador. El módulo CHI debe estar reportando información de check-ins en tiempo real. El módulo BEI debe estar disponible para reportar solicitudes de asistencia e incidentes asociados al evento. |

**Cómo se ejecutará**

- **Automáticamente:** -
- **Manualmente:** El coordinador u organizador ingresa a la aplicación, se autentica, despliega el menú hamburguesa y selecciona la opción de eventos. Luego, accede al panel de gestión de eventos y da click en la barra de búsqueda, diligencia el título del evento de interés y obtiene los resultados que concuerdan con su búsqueda. Da click sobre el evento que desea consultar y se despliega la información del evento. Si el evento llega a estar en ejecución le aparecerán los indicadores de los brigadistas por zona, junto con los incidentes reportados y sus estados. Así mismo podrá ver el progreso del check-in de los participantes. Si el evento ya está terminado podrá ver la información general y los índices de tiempo de atención de un incidente, porcentaje de participación en el check-in y desempeño de los brigadistas.

### 2. Datos de entrada

| Nombre | Descripción | Tipo de campo | Reglas / Aplicación | Obligatorio |
|:--|:--|:--|:--|:--:|
| Nombre del evento | Nombre con el que se identifica el evento. | Texto | No puede estar vacío. | Sí |
| estado | Estado en que queda el evento después de editarlo, puede ser Borrador o Programado. | Texto | Queda en Programado si incluye una fecha de programación, de lo contrario queda en Borrador. | No |

### 3. Datos de salida

| Nombre | Descripción | Tipo de campo | Reglas / Aplicación | Obligatorio |
|:--|:--|:--|:--|:--:|
| Resultados de consulta | Eventos que cumplen con los criterios de búsqueda aplicados: su título contiene el texto buscado o su estado corresponde al filtrado. | Lista | Cada resultado debe mostrar como mínimo nombre, fecha, estado, alcance y responsables. | Sí |
| Detalle del evento | Información del evento seleccionado para consulta individual. | Lista de datos | Debe mostrar la información registrada del evento y sus resultados disponibles. | Sí |

### 4. Flujo básico

| Paso | Actor | Descripción | Excepciones |
|:--:|:--|:--|:--|
| 1 | Coordinador, Organizador | Ingresa al aplicativo, se autentica, accede al panel de eventos, consulta el listado de eventos y digita el título del evento de interés, si lo requiere, aplica filtros por estado. | Si no existen eventos que cumplan con los criterios, informa al usuario que no se encontraron resultados. |
| 2 | Sistema | Muestra los eventos que cumplen con los criterios de consulta, incluyendo nombre, fecha, estado, alcance y participantes. | - |
| 3 | Coordinador, Organizador | Selecciona un evento para consultar su información detallada. | Si ningún evento cumple con lo requerido, informa que no hay coincidencias. |
| 4 | Sistema | Muestra la información del evento seleccionado (titulo, tipos, fecha, estado, alcance y participantes) y, en caso de que esté En ejecución, muestra los indicadores en tiempo real para monitorear el desempeño del mismo. | Revisar A2 y A3 |

### 5. Flujo alterno

| Paso | Actor | Descripción | Excepciones |
|:--:|:--|:--|:--|
| A1 | Sistema | Si MS08 — Analítica y Auditoría no está disponible, el sistema permite consultar la información básica de los eventos, pero no presenta los indicadores en tiempo real. | Solo se habilita la consulta de información básica. |
| A2 | Sistema | No se puede conectar con los módulos BEI o CHI. | - |
| A2 | Sistema | Informa que en el momento no es posible monitorear (o ver los resultados, en caso de haber finalizado) el evento de interés | - |
| A3 | Sistema | Recibe una lista de datos vacía de los módulos CHI o BEI. | - |
| A3 | Sistema | Informa que no se han registrado check-ins, incidentes o brigadistas en el momento | - |

<p align="right"><a href="#indice">⬆ Volver al índice</a></p>

---

## 📌 ANÁLISIS DE REQUERIMIENTOS · RF-EVE-04 <a id="rf-eve-04"></a>

### 1. Funcionalidad

| | |
|:--|:--|
| **Código** | RF-EVE-04 |
| **Nombre** | Finalizar evento |
| **Descripción** | El sistema debe permitir al coordinador u organizador terminar un evento y con ello deshabilitar el check-in, congelar la atención que hacen los brigadistas a los incidentes y notificar a los participantes la finalización del mismo. |
| **Actor principal** | Coordinador de emergencia u organizador de simulacros. |
| **Precondiciones** | El usuario debe estar autenticado y tener el rol de coordinador u organizador. El evento debe estar en estado En ejecución. El módulo CHI debe estar reportando información de check-ins en tiempo real. El módulo BEI debe estar disponible para reportar solicitudes de asistencia e incidentes asociados al evento. |

**Cómo se ejecutará**

- **Automáticamente:** En ese momento el check-in se deshabilitará y la gestión de los brigadistas quedará congelada en el estado actual del momento en el que se finalizó el evento. Así mismo, se emitirá una alerta notificando a los participantes que el evento ha finalizado.
- **Manualmente:** El coordinador u organizador ingresa a la aplicación, se autentica, despliega el menú hamburguesa y selecciona la opción de eventos. Luego, accede al panel de gestión de eventos identifica el evento de interés (lo visualiza en la lista general o lo filtra por eventos activos) y da click en el botón "Finalizar" asociado al mismo.

### 2. Datos de entrada

| Nombre | Descripción | Tipo de campo | Reglas / Aplicación | Obligatorio |
|:--|:--|:--|:--|:--:|
| Evento en ejecución | Evento cuyo seguimiento desea consultar el coordinador en tiempo real. | Selección (identificador) | Debe estar en estado En ejecución. | Sí |
| estado | Estado Finalizado para terminar el evento. | Texto | El estado anterior debe ser En ejecución, de otra manera no se permitirá el cambio. | No |

### 3. Datos de salida

| Nombre | Descripción | Tipo de campo | Reglas / Aplicación | Obligatorio |
|:--|:--|:--|:--|:--:|
| Check-ins | Cantidad y detalle de check-ins reportados por los participantes, por estado. | Lista / contadores | Información obtenida del módulo CHI/AYA | Sí |
| brigadistas | Casos atendidos y resueltos por los brigadistas durante el evento. | Lista | Información obtenida del módulo BEI/AYA | Sí |
| Incidentes | Incidentes reportados durante el evento. | Lista | Información obtenida del módulo BEI/AYA | Sí |
| estado | Estado del evento tras la finalización. | Lista | Pasa a Finalizado una vez se oprima el botón para finalizar el evento. | Sí |
| alerta | Alerta que informa a los participantes que el evento finalizó | Texto | Si el módulo ALE no está disponible igual permite terminar el evento e informa que la finalización no se pudo notificar a los participantes. | No |

### 4. Flujo básico

| Paso | Actor | Descripción | Excepciones |
|:--:|:--|:--|:--|
| 1 | Coordinador, Organizador | Ingresa al aplicativo, se autentica, accede al panel de eventos, consulta el evento de interés y oprime el botón "Finalizar" asociado a este. | FA-1, FA-2 |
| 2 | Sistema | Verifica que el estado anterior del evento sea En ejecución | Si no lo es, informa que un evento que no esté En ejecución no se puede finalizar. |
| 3 | Sistema | Informa al módulo CHI que deshabilite el check-in. | Si el módulo CHI no está disponible, informa que no se puede finalizar el evento. |
| 4 | Sistema | Informa al módulo BEI que congele el seguimiento de incidentes de los brigadistas y que deshabilite la solicitud de ayuda. | Si el módulo BEI no está disponible, informa que no se puede finalizar el evento. |
| 5 | Sistema | Informa al módulo ALE que envíe una alerta a los participantes indicando que el evento finalizó. | Si el módulo ALE no está disponible, finaliza el evento e informa que no fue posible enviar la alerta. |
| 6 | Sistema | Actualiza el estado del evento a Finalizado. | - |
| 7 | Sistema | Presenta las métricas generales del evento, incluyendo: índices de tiempo de atención de un incidente, porcentaje de participación en el check-in y desempeño de los brigadistas. | Si no hay datos, revisar A1 |

### 5. Flujo alterno

| Paso | Actor | Descripción | Excepciones |
|:--:|:--|:--|:--|
| A1 | Sistema | Si aún no se han reportado check-ins, solicitudes de asistencia o incidentes, el sistema muestra un aviso de que todavía no hay registros. | La consulta continúa disponible. |

<p align="right"><a href="#indice">⬆ Volver al índice</a></p>

---

## 📌 ANÁLISIS DE REQUERIMIENTOS · RF-EVE-05 <a id="rf-eve-05"></a>

### 1. Funcionalidad

| | |
|:--|:--|
| **Código** | RF-EVE-05 |
| **Nombre** | Habilitar panel del evento |
| **Descripción** | El sistema debe activar automáticamente el panel asociado al evento, una vez llegue la fecha de inicio y desactivarlo una vez se finalice el evento; y notificar a los módulos CHI y BEI que detengan las actividades asociadas a dicho evento. |
| **Actor principal** | Sistema (habilitación) / todos los usuarios (uso según los requerimientos de cada módulo). |
| **Precondiciones** | El evento debe encontrarse en estado Finalizado. Los resultados obtenidos durante la ejecución del evento deben estar disponibles para su consulta. MS08 — Analítica y Auditoría debe estar disponible para procesar los resultados del evento y proporcionar los indicadores requeridos para apoyar la evaluación. |

**Cómo se ejecutará**

- **Automáticamente:** **Habilitar**: El sistema detecta la fecha de inicio del evento y habilita el panel del evento (debe contener el nombre del evento, las rutas de las zonas y espacios del evento, el botón de solicitar ayuda y el botón del check-in) en la pantalla principal. Luego le solicita al módulo CAM que le envíe los mapas de las rutas del alcance del evento, al módulo CHI que inicie el registro del check-in y al módulo BEI que habilite los paneles de responsabilidades de los brigadistas asignados y el registro de incidentes. **Deshabilitar**: El sistema detecta que se finalizó el evento y deshabilita el panel del evento, quitándolo de la pantalla principal. Luego le solicita al módulo CHI que detenga el registro del check-in y al módulo BEI que deshabilite los paneles de responsabilidades de los brigadistas asignados y el registro de incidentes.
- **Manualmente:** -

### 2. Datos de entrada

| Nombre | Descripción | Tipo de campo | Reglas / Aplicación | Obligatorio |
|:--|:--|:--|:--|:--:|
| Fecha y hora de inicio | Momento en que el evento debe comenzar. | Fecha y hora | - | Sí |
| Fecha y hora de fin | Momento en que se registra la finalización manual del evento. | Fecha y hora | Debe ser posterior al momento de activación del evento. Si está vacío, todo se mantendrá habilitado. | No |

### 3. Datos de salida

| Nombre | Descripción | Tipo de campo | Reglas / Aplicación | Obligatorio |
|:--|:--|:--|:--|:--:|
| estado | Estado del evento tras la finalización. | Lista | Pasa a Finalizado una vez se oprima el botón para finalizar el evento. | Sí |

### 4. Flujo básico

| Paso | Actor | Descripción | Excepciones |
|:--:|:--|:--|:--|
| 1 | Sistema | Identifica que la fecha y hora de inicio han llegado y habilita el panel asociado al evento. | - |
| 2 | Sistema | Muestra el panel asociado al evento de la página principal de todos los usuarios. | - |
| 3 | Sistema | Solicita al módulo CAM que le envía las rutas de las zonas y espacios del alcance del evento. | Si el módulo CAM no está disponible no habilita el botón de rutas. |
| 4 | Sistema | Solicita al módulo CHI que habilite el registro de check-in. | Si el módulo CHI no está disponible revisar RF-EVE 01 y 02 |
| 5 | Sistema | Solicita al módulo BEI que habilite el registro de incidentes. | Si el módulo BEI no está disponible revisar RF-EVE 01 y 02 |
| 6 | Sistema | Solicita al módulo BEI que habilite los paneles de responsabilidades de los brigadistas. | Si el módulo BEI no está disponible revisar RF-EVE 01 y 02 |

### 5. Flujo alterno

| Paso | Actor | Descripción | Excepciones |
|:--:|:--|:--|:--|
| 1 | Sistema | Identifica que el evento fue finalizado de forma manual | - |
| 2 | Sistema | Solicita al módulo CHI que deshabilite el registro de check-in. | Si el módulo CHI no está disponible revisar RF-EVE 01 y 02 |
| 3 | Sistema | Solicita al módulo BEI que deshabilite el registro de incidentes. | Si el módulo BEI no está disponible revisar RF-EVE 01 y 02 |
| 4 | Sistema | Solicita al módulo BEI que deshabilite los paneles de responsabilidades de los brigadistas. | Si el módulo BEI no está disponible revisar RF-EVE 01 y 02 |
| 5 | Sistema | Quita el panel asociado al evento de la página principal de todos los usuarios. | - |

<p align="right"><a href="#indice">⬆ Volver al índice</a></p>

---

## 📌 ANÁLISIS DE REQUERIMIENTOS · RF-BEI-06 <a id="rf-bei-06"></a>

### 1. Funcionalidad

| | |
|:--|:--|
| **Código** | RF-BEI-06 |
| **Nombre** | Auditoría |
| **Descripción** | Registra las acciones relevantes realizadas de los eventos (creación, modificación, finalización), indicando usuario o servicio, acción, recurso afectado, fecha y hora, y resultado de la operación, y permite su consulta. |
| **Actor principal** | Sistema (registro) / Auditor - Coordinador de emergencia (consulta) |
| **Precondiciones** | Para consultar, el coordinador o auditor debe estar autenticado y el módulo AYA debe estar disponible. |

**Cómo se ejecutará**

- **Automáticamente:** El sistema detecta un cambio de estado relevante o una acción destacable durante el flujo de su funcionamiento y guarda los datos más importantes asociados a esta.
- **Manualmente:** -

### 2. Datos de entrada

| Nombre | Descripción | Tipo de campo | Reglas / Aplicación | Obligatorio |
|:--|:--|:--|:--|:--:|
| tipoRegistro | Describe la acción efectuada sobre el módulo. | Texto | El tipo debe ser Creación, Modificación o Finalización. | Sí |
| usuarioOrigen | Usuario que realizó la acción en el sistema | Texto | Generado automáticamente al ocurrir cada acción auditable. | Sí |
| accion | Acción que se realizó en el sistema | Texto | - | Sí |
| fechaHoraRegistro | Fecha y hora en la que se realizó la acción en el sistema | Fecha y hora | - | Sí |
| evento | Evento durante el que se realizó la acción. | Objeto | Si la acción se realizó cuando no había eventos activos, guardad este campo como vacío | No |
| origen | Origen del módulo que solicita guardar el registro de auditoría | Texto | - | Sí |

### 3. Datos de salida

| Nombre | Descripción | Tipo de campo | Reglas / Aplicación | Obligatorio |
|:--|:--|:--|:--|:--:|
| idRegistro | Identificador único generado para el registro de auditoría. | Texto (UUID) | Generado automáticamente por el sistema. | Sí |
| listaEventos | Lista de eventos de auditoría, cada uno con usuario/servicio, acción, recurso afectado, fecha/hora y resultado, ordenados cronológicamente. | Lista de objetos | - | Sí |

### 4. Flujo básico

| Paso | Actor | Descripción | Excepciones |
|:--:|:--|:--|:--|
| 1 | Sistema | Detecta que ocurrió una acción auditable (por ejemplo, una creación o una modificación). | - |
| 2 | Sistema | Registra el evento con el usuario o servicio responsable, la acción, el recurso afectado, la fecha y hora y el resultado. | - |
| 3 | Auditor | Solicita el historial de auditoría indicando el recurso o el evento. | Si el actor no tiene el rol requerido, ver Flujo alterno A2. |
| 4 | Sistema | Retorna la lista de eventos registrados. | - |

### 5. Flujo alterno

| Paso | Actor | Descripción | Excepciones |
|:--:|:--|:--|:--|
| A1 | Sistema | Detecta que el recurso consultado no tiene eventos registrados. | Retorna una lista vacía sin error. |
| A2 | Sistema | Detecta que el actor no cuenta con el rol requerido para consultar la auditoría. | Rechaza la solicitud con un error de autorización. |

<p align="right"><a href="#indice">⬆ Volver al índice</a></p>

---

## 📜 REGLAS DE NEGOCIO <a id="reglas-de-negocio"></a>

| No. | Descripción |
|:--:|:--|
| 1 | Todo evento nuevo se registra en estado Borrador. |
| 2 | El evento debe transitar de forma unidireccional por los estados Borrador → Programado → En ejecución → Finalizado → Evaluado. No se permite retroceder de estado ni omitir un estado intermedio. |
| 3 | La fecha de inicio debe ser posterior a la fecha actual. |
| 4 | Solo un usuario con rol de coordinador de emergencias puede crear un evento real. |
| 5 | Un organizador de simulacros solo podrá gestionar eventos que necesariamente tengan el tipo "Simulacro." |
| 6 | Un evento con el escenario o la fecha pendientes se guarda en estado borrador y al diligenciar estos datos pasa a estado programado. |
| 7 | Un evento puede iniciar sin brigadistas asignados. |
| 8 | La fecha y hora de programación debe ser superior a la de creación (la identifica el sistema). |
| 9 | Solo puede existir un evento activo al tiempo. |
| 10 | Al llegar la fecha y hora de inicio, un evento Programado pasa a En ejecución automáticamente y se envía una alerta general a los participantes. |
| 11 | El seguimiento en tiempo real solo está disponible mientras el evento está En ejecución. Una vez Finalizado, esa información se consulta como resultados en la evaluación (RF-EVE-04). |
| 12 | Al llegar la fecha y hora de finalización, un evento En ejecución pasa a Finalizado automáticamente. |
| 13 | Solo un evento en estado Finalizado puede evaluarse. |
| 14 | Un evento se puede editar en los estados Borrador, Programado y En ejecución. |
| 15 | La evaluación y las observaciones deben registrarse antes de cambiar el estado del evento de Finalizado a Evaluado. |
| 16 | Solo un brigadista activo podrá registrarse a un evento. |
| 17 | La alerta de activación/asignación de brigadistas será enviada solo cuando el evento esté en estado Programado y en caso de que se edite en ese mismo estado. |

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
| RG | Regla de negocio |
| FA | Flujo alterno |
| MS01 | Microservicio Identidad y acceso |
| MS02 | Microservicio Campus y seguridad |
| MS03 | Microservicio Alertas y emergencias |
| MS04 | Microservicio Evacuación y check-in |
| MS05 | Microservicio Brigadistas e incidentes |
| MS06 | Microservicio Eventos |
| MS08 | Microservicio Analítica y auditoría |

<p align="right"><a href="#indice">⬆ Volver al índice</a></p>

---

## 🕓 HISTORIAL DE REVISIÓN <a id="historial-de-revision"></a>

| Elaborado por | Revisado por | Fecha | Descripción y justificación de cambios |
|:--|:--|:--:|:--|
| Equipo Zephyros | Pendiente | 28/09/2026 | Versión inicial del documento de análisis de requerimientos de MS06 - Eventos. |

<p align="right"><a href="#indice">⬆ Volver al índice</a></p>
