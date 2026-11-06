<h1 align="center">🛡️ Campus Seguro</h1>
<h3 align="center">MS05 · Brigadistas e Incidentes</h3>

<p align="center"><em>Análisis de requerimientos funcionales, reglas de negocio e historial de revisión.</em></p>

<p align="center">
  <img alt="Curso" src="https://img.shields.io/badge/Curso-DOSW%202026--2-1f6feb">
  <img alt="Squad" src="https://img.shields.io/badge/Squad-Gaia%20%C2%B7%20Kairos%20%C2%B7%20Zephyros-8957e5">
  <img alt="Documento" src="https://img.shields.io/badge/Documento-Requerimientos-d29922">
  <img alt="Versión" src="https://img.shields.io/badge/Versi%C3%B3n-1.0-2da44e">
</p>

<p align="center">
  <a href="#rf-bei-01">📌 Requerimientos</a> ·
  <a href="#reglas-de-negocio">📜 Reglas de negocio</a> ·
  <a href="#abreviaturas">🔤 Abreviaturas</a> ·
  <a href="#historial-de-revision">🕓 Historial</a> ·
  <a href="TU_URL">📦 Repositorio</a> ·
  <a href="TU_URL">📋 Jira</a>
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

Este documento detalla cada uno de los requerimientos funcionales del microservicio **MS05 - Brigadistas e Incidentes**, incluyendo su funcionalidad, datos de entrada y salida, flujo básico y flujo alterno. Al final se incluyen las reglas de negocio, abreviaturas e historial de revisión aplicables a todos los requerimientos.

---

<a id="indice"></a>

## 🧭 Índice

| Requerimiento | Nombre |
|:--|:--|
| [RF-BEI-01](#rf-bei-01) | Asignar espacios a brigadistas |
| [RF-BEI-02](#rf-bei-02) | Buscar brigadista |
| [RF-BEI-03](#rf-bei-03) | Activar brigadista |
| [RF-BEI-04](#rf-bei-04) | Registrar incidente |
| [RF-BEI-05](#rf-bei-05) | Editar incidente |
| [RF-BEI-06](#rf-bei-06) | Consultar responsabilidades. |
| [RF-BEI-07](#rf-bei-07) | Gestionar atención de incidentes |
| [RF-BEI-08](#rf-bei-08) | Seguimiento y cierre de incidentes |
| [RF-BEI-09](#rf-bei-09) | Auditoría |
| [Reglas de negocio](#reglas-de-negocio) | Aplican a todos los requerimientos |
| [Abreviaturas](#abreviaturas) | Glosario de siglas |
| [Historial de revisión](#historial-de-revision) | Cambios del documento |

---

<!-- ══════════════════════ COPIA ESTE BLOQUE POR CADA REQUERIMIENTO ══════════════════════ -->

## 📌 ANÁLISIS DE REQUERIMIENTOS · RF-BEI-01 <a id="rf-bei-01"></a>

### 1. Funcionalidad

| | |
|:--|:--|
| **Código** | RF-BEI-01 |
| **Nombre** | Asignar espacios a brigadistas |
| **Descripción** | Permite al coordinador de emergencia o al organizador de simulacros asignar brigadistas a edificios, zonas o puntos de encuentro para un evento, así como modificar o retirar dichas asignaciones. Una ubicación puede tener varios brigadistas y un brigadista puede tener varias ubicaciones. |
| **Actor principal** | Coordinador de emergencia, organizador de simulacros o brigadista. |
| **Precondiciones** | El coordinador u organizador debe estar autenticado. El evento debe existir según RF-EVE-01. El usuario debe tener el rol de Brigadista y debe estar activo. Los edificios, zonas o puntos de encuentro deben existir según el RF-CAMP-01. |

**Cómo se ejecutará**

- **Automáticamente:** -
- **Manualmente:** **Opción 1:** El coordinador u organizador ingresa a la aplicación, se autentica, despliega el menú hamburguesa, da click en el botón de Gestión de brigadistas y accede a la pantalla con el listado de brigadistas ordenados por orden alfabético de sus apellidos. Durante el evento, podrá reasignar brigadistas (temporalmente) a las zonas que requieran más apoyo, mediante el botón desplegable correspondiente a la zona asignada, que se encuentra en frente de cada brigadista (en el mapa de seguimiento de incidentes). **Opción 2**: El brigadista recibe una alerta a su correo electrónico invitándole a activarse y hacer parte de un evento. El brigadista ingresa a la aplicación, se autentica y verifica si está activo (de no estarlo se activa). Luego, revisa el mapa de zonas incluidas en el evento y selecciona un punto donde va a brindar su apoyo. Una vez asignada la zona, al brigadista se le habilitan funciones relacionadas con la asignación y el evento (ver RF-BEI del 02 al 09).

### 2. Datos de entrada

| Nombre | Descripción | Tipo de campo | Reglas / Aplicación | Obligatorio |
|:--|:--|:--|:--|:--:|
| nombre | Nombre del usuario brigadista que será asignado. | Texto | Debe existir según RF-AUT-01, estar activo y tener el rol Brigadista. | Sí |
| alcance | Edificio, zona o punto de encuentro del que será responsable el brigadista. Incluye el tipo de ubicación y su ID. | Lista de Objetos | El tipo debe ser Edificio, Zona segura o Punto de encuentro y el objeto debe existir. | Sí |
| observaciones | Información relevante sobre el estado de la ubicación asignada o medidas de seguridad a tener en cuenta. | Texto largo | - | No |

### 3. Datos de salida

| Nombre | Descripción | Tipo de campo | Reglas / Aplicación | Obligatorio |
|:--|:--|:--|:--|:--:|
| idAsignacion | Identificador único generado para la asignación. | Texto (UUID) | Generado automáticamente por el sistema. | Sí |
| estado | Estado de la asignación. | Texto (enum) | Vigente al crearse la asignación; Retirada al terminar la asignación o finalizar el evento. | Sí |
| fechaCreacion | Fecha y hora de creación del registro. | Fecha/hora | Generadas automáticamente por el sistema. | Sí |

### 4. Flujo básico

| Paso | Actor | Descripción | Excepciones |
|:--:|:--|:--|:--|
| 1 | Coordinador u organizador | Ingresa a la aplicación, se autentica, ingresa a la sección de gestión de brigadistas, identifica un brigadista y le asigna un alcance (edificio, zona o punto de encuentro) para cubrir. | - |
| 2 | Sistema | Valida los campos obligatorios estén completos, que el evento esté vigente y que el brigadista y la ubicación existan. | Si un campo falta o un ID no existe, ver Flujo alterno A1. |
| | | Valida que el brigadista esté activo. | Si el brigadista no está activo, ver Flujo alterno A2 |
| 3 | Sistema | Verifica que no exista una asignación vigente idéntica (mismo brigadista, ubicación y evento). | Si ya existe, ver Flujo alterno A3. |
| 4 | Sistema | Genera el identificador de la asignación y la guarda en estado Vigente. | - |
| 5 | Sistema | Registra la acción en el módulo de auditoría. | - |
| 6 | Sistema | Retorna al coordinador, organizador la confirmación con el nombre y el estado de la asignación. | - |

### 5. Flujo alterno

| Paso | Actor | Descripción | Excepciones |
|:--:|:--|:--|:--|
| A1 | Sistema | Detecta que uno o más campos obligatorios no fueron diligenciados, que el evento no está vigente o que un ID referenciado no existe o no es válido. | Se rechaza la operación y se retorna un mensaje indicando el campo o campos inválidos. |
| A2 | Sistema | Detecta que el brigadista no está activo, y rechaza la asignación. | Se rechaza la operación y se retorna un mensaje indicando que no se pueden hacer asignaciones a brigadistas inactivos. |
| A3 | Sistema | Detecta que ya existe una asignación vigente idéntica. | Se rechaza la operación indicando que la asignación ya existe. |
| A4 | Coordinador | Solicita modificar la ubicación u observaciones de una asignación vigente. | - |
| A4 | Sistema | Aplica las mismas validaciones del flujo básico, actualiza la asignación, registra la fecha de actualización y la acción en auditoría. | Si la asignación está Retirada, se rechaza la operación. |
| A5 | Coordinador | Solicita retirar una asignación vigente, cambiando la etiqueta de Zona asignada a "Sin asignación". | - |
| A5 | Sistema | Cambia el estado a Retirada conservando el histórico y registra la acción en auditoría. El brigadista deja de ver los incidentes reportados en esa zona. | Se informa al coordinador si no hay incidentes sin resolver y no hay más brigadistas. |
| A6 | Sistema | Recibe solicitud del RF-EVE-01 para realizar las asignaciones a los brigadistas por default tras iniciar un evento. | |
| A6 | Sistema | Asigna las zonas predefinidas al brigadista indicado según la BD | Si un brigadista no está activo en ese proceso, rechaza la asignación e informa que no se pueden asignar zonas a brigadistas inactivos. |
| A7 | Brigadista | Recibe una alerta administrativa a su correo electrónico y es redireccionado al aplicativo donde debe autenticarse y elegir una zona donde apoyar. | El brigadista debe estar activo para que se le asigne una zona. |
| A7 | Sistema | Verifica que el usuario tenga el rol de brigadista y esté activo. | |
| A7 | Sistema | Verifica que no exista una asignación vigente idéntica (mismo brigadista, ubicación y evento). | Si ya existe, ver Flujo alterno A3. |
| A7 | Sistema | Genera el identificador de la asignación y la guarda en estado Vigente. | - |
| A7 | Sistema | Registra la acción en el módulo de auditoría. | - |
| A7 | Sistema | Retorna al coordinador, organizador la confirmación con el nombre y el estado de la asignación. | - |

<p align="right"><a href="#indice">⬆ Volver al índice</a></p>

---

<!-- ══════════════════════ FIN DEL BLOQUE DEL REQUERIMIENTO ══════════════════════ -->

## 📌 ANÁLISIS DE REQUERIMIENTOS · RF-BEI-02 <a id="rf-bei-02"></a>

### 1. Funcionalidad

| | |
|:--|:--|
| **Código** | RF-BEI-02 |
| **Nombre** | Buscar brigadista |
| **Descripción** | Permite al coordinador de emergencia o al organizador de simulacros buscar a un brigadista específicamente por nombre para realizar modificaciones a sus asignaciones. |
| **Actor principal** | Coordinador de emergencia u organizador de simulacros |
| **Precondiciones** | El coordinador u organizador debe estar autenticado. El evento debe existir según RF-EVE-01. El brigadista debe existir según RF-AUT-01, estar activo y tener el rol de Brigadista. Los edificios, zonas o puntos de encuentro deben existir según el RF-CAMP-01. |

**Cómo se ejecutará**

- **Automáticamente:** -
- **Manualmente:** El coordinador u organizador ingresa a la aplicación, se autentica, despliega el menú hamburguesa, da click en el botón de Gestión de brigadistas y accede a la pantalla con el listado de brigadistas. Allí, busca el símbolo de lupa y diligencia en el campo correspondiente los nombres o apellidos del brigadista deseado. Al darle buscar, el sistema retorna los brigadistas de la lista cuyos nombres/apellidos concuerde con la búsqueda. Desde ese resultado de búsqueda también podrá realizar asignaciones (ver RF-BEI-01).

### 2. Datos de entrada

| Nombre | Descripción | Tipo de campo | Reglas / Aplicación | Obligatorio |
|:--|:--|:--|:--|:--:|
| nombreBrigadista | Nombre del usuario brigadista que se desea identificar. | Texto | Debe existir según RF-AUT-01, estar activo y tener el rol Brigadista. | Sí |

### 3. Datos de salida

| Nombre | Descripción | Tipo de campo | Reglas / Aplicación | Obligatorio |
|:--|:--|:--|:--|:--:|
| nombreBrigadista | Brigadistas cuyos nombre/apellidos concuerdan con la búsqueda | Texto | Generado automáticamente por el sistema. | Sí |
| estado | Indica si el brigadista esta activo o no | Booleano | Puede ser Activo o Inactivo | Sí |
| asignaciones | Zonas a las que el brigadista está asignado en el momento de la consulta | Lista de objetos | Si no tiene ninguna asignación, mostrará un mensaje de "Sin asignaciones" | Sí |

### 4. Flujo básico

| Paso | Actor | Descripción | Excepciones |
|:--:|:--|:--|:--|
| 1 | Coordinador u organizador | Ingresa a la aplicación, se autentica, ingresa a la sección de gestión de brigadistas, identifica la lupa, diligencia el nombre del brigadista en el campo relacionado y le da buscar. | - |
| 2 | Sistema | Valida que el campo del nombre del brigadista no esté vacío | Si está vacío, retorna un mensaje al usuario indicando que por favor ingrese un nombre o apellido. |
| 3 | Sistema | Busca en la base de datos brigadistas cuyos nombres contengan el texto buscado. | Si no encuentra ninguno, ver Flujo alterno A1 |
| 3 | Sistema | Retorna al coordinador, organizador la información relacionada con el brigadista. | - |

### 5. Flujo alterno

| Paso | Actor | Descripción | Excepciones |
|:--:|:--|:--|:--|
| A1 | Sistema | Detecta que uno no hay nombre compatibles y retorna una lista vacía. | Se informa que no hay resultados compatibles con la búsqueda. |

<p align="right"><a href="#indice">⬆ Volver al índice</a></p>

---

## 📌 ANÁLISIS DE REQUERIMIENTOS · RF-BEI-03 <a id="rf-bei-03"></a>

### 1. Funcionalidad

| | |
|:--|:--|
| **Código** | RF-BEI-03 |
| **Nombre** | Activar brigadista |
| **Descripción** | Permite al brigadista activar sus funcionalidades como brigadista para recibir asignaciones y gestionar incidentes. |
| **Actor principal** | Brigadista |
| **Precondiciones** | El brigadista debe estar autenticado. |

**Cómo se ejecutará**

- **Automáticamente:** -
- **Manualmente:** El brigadista ingresa a la aplicación, se autentica, despliega el menú hamburguesa y accede al panel de responsabilidades. Una vez en el panel podrá ver su estado actual. En el botón correspondiente al estado podrá dar click: si está en estado Activo pasa a estado Inactivo, si estaba en Inactivo pasa a estado Activo.

### 2. Datos de entrada

| Nombre | Descripción | Tipo de campo | Reglas / Aplicación | Obligatorio |
|:--|:--|:--|:--|:--:|
| estado | Estado del brigadista. | Texto (enum) | Vigente al crearse la asignación; Retirada al terminar la asignación o finalizar el evento. | estado |

### 3. Datos de salida

| Nombre | Descripción | Tipo de campo | Reglas / Aplicación | Obligatorio |
|:--|:--|:--|:--|:--:|
| idActivacion | Identificador único generado para la activación. | Texto (UUID) | Generado automáticamente por el sistema. | Sí |
| estado | Estado del brigadista. | Booleano | Puede ser Activo o Inactivo. | Sí |
| fechaCambioEstado | Fecha y hora del cambio de estado | Fecha/hora | Generadas automáticamente por el sistema. | Sí |

### 4. Flujo básico

| Paso | Actor | Descripción | Excepciones |
|:--:|:--|:--|:--|
| 1 | Coordinador u organizador | Ingresa a la app y se autentica, accede al panel de responsabilidades y da click sobre su estado. | - |
| 2 | Sistema | Identifica el estado anterior: si era Activo, lo pasa a Inactivo, si estaba Inactivo lo pasa a Activo | - |
| | | Si se va a inactivar durante un evento, verifica que no tenga incidentes En_Atención. | Si los tiene, ver Flujo alterno A1. |
| 5 | Sistema | Registra el cambio de estado. Si pasa a Activo habilita la visualización de incidentes asignados a su zona, si pasa a Inactivo inhibe la visualización de incidentes. | Un brigadista con incidentes sin resolver no se puede desactivar durante un evento vigente asociado. |
| 6 | Sistema | Actualiza el estado actual en la interfaz para que el usuario vea su nuevo estado. | - |

### 5. Flujo alterno

| Paso | Actor | Descripción | Excepciones |
|:--:|:--|:--|:--|
| A1 | Sistema | Detecta que el brigadista que desea pasar a estado Inactivo está en un evento y tiene incidentes en atención. | Rechaza la operación y le informa al brigadista que debe resolver los incidentes asignados antes de desactivarse. |

<p align="right"><a href="#indice">⬆ Volver al índice</a></p>

---

## 📌 ANÁLISIS DE REQUERIMIENTOS · RF-BEI-04 <a id="rf-bei-04"></a>

### 1. Funcionalidad

| | |
|:--|:--|
| **Código** | RF-BEI-04 |
| **Nombre** | Registrar incidente |
| **Descripción** | Permite a todos los usuarios del sistema (exceptuando el observador) registrar una solicitud de ayuda durante un evento. Dependiendo de la gravedad de la situación puede indicar la urgencia de la atención: Emergencia, Urgencia moderada, Poca urgencia.<br>Nota: los incidentes relacionados con infraestructura se gestionarán desde el modulo de campus y seguridad (RF-CAMP). |
| **Actor principal** | **Usuario general** (estudiantes, profesores, personal administrativo, coordinadores, organizadores, administradores, brigadista): para una solicitud propia<br>**Brigadistas**: para solicitudes ajenas. |
| **Precondiciones** | Debe existir un evento activo, al cual se relacione el incidente. Cualquier actor debe estar autenticado para crear un incidente propio, para crear un incidente de un tercero debe tener el rol de brigadista y debe estar activo. Los edificios, espacios, puntos de encuentro o rutas referenciados deben existir en el módulo de campus y seguridad. |

**Cómo se ejecutará**

- **Automáticamente:** -
- **Manualmente:** **Usuario general**: Ingresa a la aplicación, se autentica, accede inmediatamente a la pagina principal donde podrá ver los eventos activos en el momento. Da click en el botón "Necesito Ayuda", relacionado con el evento sobre el cual desea hacer la solicitud. Accederá al panel de creación de incidente donde deberá seleccionar obligatoriamente la ubicación donde necesita la ayuda y la gravedad de la situación para definir la prioridad. Opcionalmente podrá añadir descripción y evidencia (máximo 5 archivos). **Brigadista**: Ingresa a la aplicación, se autentica, despliega el menú hamburguesa, accede al panel de responsabilidades. Una vez allí, si hay un evento activo tendrá habilitado un botón llamado "Reportar Incidente", da click en dicho botón y diligencia el nombre del usuario afectado, ubicación, la prioridad y opcionalmente una descripción y adjuntar evidencia.

### 2. Datos de entrada

| Nombre | Descripción | Tipo de campo | Reglas / Aplicación | Obligatorio |
|:--|:--|:--|:--|:--:|
| nombreAfectado | Nombre del usuario afectado que necesita la asistencia de un brigadista. | Texto | En un incidente creado por un usurario general se guardará automáticamente el nombre de quien reporta. Para incidentes creados por brigadistas, el brigadista debe ingresar el nombre del afectado. | Sí |
| prioridad | Gravedad del incidente reportado. | Objeto (enum) | Gravedad del incidente ocurrido. Puede ser Emergencia (prioridad alta, emergencias vitales), Urgencia (prioridad media, emergencias que no se puede solucionar sola) moderada, Poca urgencia (prioridad baja, no hay riesgo en la vida del usuario) | Sí |
| ubicacion | Lugar donde ocurre el incidente: edificio, espacio o punto de encuentro. | Objeto (enum + ID) | El objeto debe existir según los tipos definidos por el módulo de campus y seguridad (RF-CAMP). | Sí |
| descripcion | Descripción de lo ocurrido. | Texto largo | Longitud máxima definida (por ejemplo, 1000 caracteres). | No |
| evidencia | Archivos de apoyo para atender la emergencia. | Lista de objetos | Pueden ser archivos imágenes (png), videos (mp4) o documentos (pdf). | No |

### 3. Datos de salida

| Nombre | Descripción | Tipo de campo | Reglas / Aplicación | Obligatorio |
|:--|:--|:--|:--|:--:|
| idIncidente | Identificador único generado para el incidente. | Texto (UUID) | Generado automáticamente por el sistema. | Sí |
| estado | Estado inicial del incidente. | Texto (enum) | Reportado al momento de crearse, si no hay brigadistas asignados a la zona reportada. Si los hay estará en estado Asignado y el brigadista lo verá como Pendiente. | Sí |
| idReportante | Usuario que registró el incidente. | Texto (ID de usuario) | Obtenido del token JWT. | Sí |
| fechaReporte | Fecha y hora de registro del incidente. | Fecha/hora | Generada automáticamente por el sistema. | Sí |

### 4. Flujo básico

| Paso | Actor | Descripción | Excepciones |
|:--:|:--|:--|:--|
| 1 | Usuario general | Ingresa al aplicativo, identifica el evento de interés y solicita ayuda. Luego, accede al formulario de incidentes y diligencia obligatoriamente la zona y el nivel de prioridad. | - |
| 2 | Sistema | Valida que los campos obligatorios estén completos, que el evento esté activo y que los IDs referenciados de la zona elegida existan. | Si los datos son inválidos, ver Flujo alterno A1. |
| 3 | Sistema | Genera el identificador, guarda el nombre del usuario que hizo la petición como el afectado, y revisa si hay brigadistas en la zona: si los hay guarda el incidente en estado Asignado, de lo contrario los guarda en estado Reportado. | - |
| 4 | Sistema | Registra la acción de creación en el módulo de auditoría. | - |
| 5 | Sistema | Retorna al actor la confirmación con el estado del incidente. | - |

### 5. Flujo alterno

| Paso | Actor | Descripción | Excepciones |
|:--:|:--|:--|:--|
| A1 | Sistema | Detecta campos obligatorios faltantes, una categoría fuera del catálogo, un evento que no está activo o un ID referenciado inexistente. | Se rechaza la operación y se retorna un mensaje indicando el campo o campos inválidos o servicio no disponible en el momento. |
| A1 | Usuario general | Corrige la información señalada y reintenta el envío. | - |
| A2 | Brigadista | Ingresa al aplicativo, accede al menú y revisa el panel de responsabilidades, da click en el botón de reportar incidente, diligencia obligatoriamente el nombre del afectado, la zona del incidente y la prioridad del mismo. | |
| A2 | Sistema | Valida que los campos obligatorios estén completos, que el evento esté activo y que los IDs referenciados de la zona elegida existan. | Si los datos son inválidos, ver Flujo alterno A1. |
| A2 | Sistema | Genera el identificador y revisa si hay brigadistas en la zona: si los hay guarda el incidente en estado Asignado, de lo contrario los guarda en estado Reportado. | - |
| A2 | Sistema | Continúa con el flujo básico | Ver Flujo básico desde el paso 4. |

<p align="right"><a href="#indice">⬆ Volver al índice</a></p>

---

## 📌 ANÁLISIS DE REQUERIMIENTOS · RF-BEI-05 <a id="rf-bei-05"></a>

### 1. Funcionalidad

| | |
|:--|:--|
| **Código** | RF-BEI-05 |
| **Nombre** | Editar incidente |
| **Descripción** | Permite adjuntar a un incidente una descripción y, cuando corresponda, máximo cinco archivos de evidencia (pueden ser fotos (png), videos (mp4) o documentos (pdf)), así como consultar las evidencias registradas. |
| **Actor principal** | **Usuario general** (estudiantes, profesores, personal administrativo, coordinadores, organizadores, administradores, brigadista): para una solicitud propia<br>**Brigadistas**: para solicitudes ajenas. |
| **Precondiciones** | Debe existir el incidente y debe estar en un estado diferente a cerrado. Cualquier actor debe estar autenticado para editar un incidente propio, para editar un incidente de un tercero debe tener el rol de brigadista y debe estar activo. |

**Cómo se ejecutará**

- **Automáticamente:** -
- **Manualmente:** **Usuario general \| Brigadista**: Ingresa a la aplicación, se autentica, accede inmediatamente a la página principal donde podrá ver los eventos activos en el momento y da click en el botón "Mis incidentes". Accederá a una pantalla con todos los incidentes creados al momento. Podrá elegir un incidente en particular, dar click sobre el botón de editar y acceder a un entorno de donde podrá modificar únicamente la descripción del incidente y adjuntar nueva evidencia (máximo 5 archivos.

### 2. Datos de entrada

| Nombre | Descripción | Tipo de campo | Reglas / Aplicación | Obligatorio |
|:--|:--|:--|:--|:--:|
| descripcion | Descripción de la evidencia. | Texto | Longitud máxima definida (por ejemplo, 500 caracteres). | No |
| evidencia | Archivo de evidencia que apoyan la atención del incidente. | Lista de Objetos | Formatos permitidos: fotos (png), videos (mp4) o documentos (pdf) | No |

### 3. Datos de salida

| Nombre | Descripción | Tipo de campo | Reglas / Aplicación | Obligatorio |
|:--|:--|:--|:--|:--:|
| idEvidencia | Identificador único generado para la evidencia. | Texto (UUID) | Generado automáticamente por el sistema. | Sí |
| idUsuario | Usuario que adjuntó la evidencia. | Texto (ID de usuario) | Obtenido del token JWT. | Sí |
| fechaRegistro | Fecha y hora en que se adjuntó la evidencia. | Fecha/hora | Generada automáticamente por el sistema. | Sí |
| listaEvidencias | Evidencias registradas del incidente, con descripción, autor y fecha. | Lista de objetos | Solo en la consulta de evidencias. | Sí |

### 4. Flujo básico

| Paso | Actor | Descripción | Excepciones |
|:--:|:--|:--|:--|
| 1 | Usuario general | Ingresa al aplicativo, identifica el incidente a modificar. Luego, accede a la zona de edición de este y añade una descripción o evidencia. | Los únicos campos modificables son los de descripción y evidencia. |
| 2 | Sistema | Valida que el incidente exista y no esté cerrado, que el actor esté autenticado, que el archivo cumpla el formato y la descripción la extensión y que la cantidad de archivos no supere el máximo (cinco). | Si falla, ver Flujo alterno A1 o A2 |
| 3 | Sistema | Almacena la evidencia asociada al incidente. | - |
| 4 | Sistema | Registra la acción en el módulo de auditoría. | - |
| 5 | Sistema | Retorna la confirmación. | - |

### 5. Flujo alterno

| Paso | Actor | Descripción | Excepciones |
|:--:|:--|:--|:--|
| A1 | Sistema | Detecta que el archivo no cumple el formato permitido o que excede la cantidad de archivos permitidos. | Se rechaza la operación indicando la restricción incumplida. |
| A2 | Sistema | Detecta que el incidente se encuentra en estado Cerrado. | Se rechaza la operación indicando que el incidente no admite modificaciones. |
| A3 | Actor general o brigadista | Solicita consultar las evidencias de un incidente. | - |
| A3 | Sistema | Retorna la lista de evidencias del incidente. | Si no tiene evidencias, retorna una lista vacía sin error. |

<p align="right"><a href="#indice">⬆ Volver al índice</a></p>

---

## 📌 ANÁLISIS DE REQUERIMIENTOS · RF-BEI-06 <a id="rf-bei-06"></a>

### 1. Funcionalidad

| | |
|:--|:--|
| **Código** | RF- BEI-06 |
| **Nombre** | Consultar responsabilidades. |
| **Descripción** | Permite a un brigadista ver sus propias zonas e incidentes asignados (panel de responsabilidades); el coordinador y el auditor pueden consultar el listado general con filtros y paginación. |
| **Actor principal** | Brigadista, Coordinador de emergencia, Auditor/Observador |
| **Precondiciones** | El brigadista, coordinador o auditor debe estar autenticado. El panel propio requiere el rol Brigadista. El listado general, la modificación de asignaciones y la consulta puntual requieren el rol Coordinador de emergencia. El Auditor/Observador solo podrá consultar. |

**Cómo se ejecutará**

- **Automáticamente:** Durante un evento activo se priorizarán los incidentes asociados al evento sin resolver, una vez finalice el evento se muestran los incidentes ordenados por fecha y prioridad..
- **Manualmente:** El brigadista, coordinador o auditor abre la aplicación, se autentica, despliega el menú hamburguesa y da click en "Panel de responsabilidades". Allí entra a un listado donde mostrará inicialmente los incidentes, organizados por zona asignada, en orden de prioridad. Solo el coordinador podrá cerrar un incidente y reasignar una zona. Tanto el coordinador como el brigadista podrán reportar incidentes EnProceso o Completados. El auditor solo tendrá permisos de consulta.

### 2. Datos de entrada

| Nombre | Descripción | Tipo de campo | Reglas / Aplicación | Obligatorio |
|:--|:--|:--|:--|:--:|
| asignación | Edificio, zona o punto de encuentro del que será responsable el brigadista. Incluye el tipo de ubicación y su ID. | Lista de Objetos | El tipo debe ser Edificio, Zona segura o Punto de encuentro y el objeto debe existir. | Sí |
| nombreBrigadista | Nombre del usuario brigadista que será asignado. | Texto | Debe existir según RF-AUT-01, estar activo y tener el rol Brigadista. | Sí |

### 3. Datos de salida

| Nombre | Descripción | Tipo de campo | Reglas / Aplicación | Obligatorio |
|:--|:--|:--|:--|:--:|
| asignaciones | Lista de zonas con los brigadistas asignados a ellas. | Lista de objetos | En el panel propio solo se retornan las asignaciones del brigadista autenticado. | Sí |
| tareasPendientes | Incidentes que el brigadista tiene a su cargo, con su estado y ubicación. | Lista de objetos | Solo se retorna en el panel propio del brigadista. | Sí (panel propio) |
| metadatosPaginacion | Información de paginación: total de registros y página actual. | Objeto | Se retorna junto con el listado general. | Sí (listado) |

### 4. Flujo básico

| Paso | Actor | Descripción | Excepciones |
|:--:|:--|:--|:--|
| 1 | Brigadista | Ingresa a la aplicación, se autentica, ingresa a su panel de responsabilidades para el evento vigente. | - |
| 2 | Sistema | Identifica al brigadista a partir del token JWT y obtiene sus asignaciones vigentes. | Si no tiene asignaciones, ver Flujo alterno A1. |
| 3 | Sistema | Obtiene las solicitudes de asistencia e incidentes a su cargo. | - |
| 4 | Sistema | Retorna las zonas asignadas y los incidentes pendientes por prioridad y antigüedad. | - |

### 5. Flujo alterno

| Paso | Actor | Descripción | Excepciones |
|:--:|:--|:--|:--|
| A1 | Sistema | Detecta que el brigadista no tiene asignaciones vigentes para el evento. | Retorna una lista vacía sin error. |
| A2 | Coordinador o Auditor | Solicita el listado general indicando opcionalmente evento, brigadista y estado. | - |
| A2 | Sistema | Aplica los filtros recibidos, pagina los resultados y retorna el listado con los metadatos de paginación. | Si ningún filtro es válido, se ignora y se retorna el listado completo paginado. |
| A3 | Sistema | Detecta que el identificador de la asignación consultada no existe. | Retorna un error controlado indicando que el recurso no existe. |

<p align="right"><a href="#indice">⬆ Volver al índice</a></p>

---

## 📌 ANÁLISIS DE REQUERIMIENTOS · RF-BEI-07 <a id="rf-bei-07"></a>

### 1. Funcionalidad

| | |
|:--|:--|
| **Código** | RF-BEI-07 |
| **Nombre** | Gestionar atención de incidentes |
| **Descripción** | Administra el ciclo de atención de un incidente (Reportado → Asignado → En atención → Resuelto → Cerrado) y las acciones que producen cada transición: asignar o reasignar responsable, iniciar atención, resolver y cerrar. |
| **Actor principal** | Coordinador de emergencia (asignar, reasignar, cerrar) / Brigadista responsable (iniciar atención, resolver) |
| **Precondiciones** | El actor debe estar autenticado con el rol correspondiente a la acción. La solicitud debe existir y encontrarse en el estado previo válido para la transición (ver Reglas de negocio N.° 1). El brigadista a asignar debe tener el rol Brigadista, estar activo y contar con una asignación vigente en el evento. |

**Cómo se ejecutará**

- **Automáticamente:** Si existe al menos un brigadista asignado a la zona del reporte, los incidentes reportados en dicha zona pasarán a estado Asignado.
- **Manualmente:** Un usuario general (revisar RF-BEI-04) ingresa al aplicativo, se autentica y reporta un incidente asociado a un evento, este incidente se guardará en estado Reportado. Paralelamente, el brigadista ingresa al aplicativo, se autentica, revisa su panel de responsabilidades y, de estar activo, puede iniciar la atención de un incidente pasándolo al estado EnAtencion. Una vez soluciones el incidente podrá registrarlo como Resuelto. Así mismo, el coordinador ingresa al aplicativo, se autentica y consulta el panel de gestión de brigadistas, donde podrá reasignar brigadistas activo. De igual manera, puede remitirse a la sección de incidente para finalizar la atención de uno de ellos, pasándolo al estado Cerrado.

### 2. Datos de entrada

| Nombre | Descripción | Tipo de campo | Reglas / Aplicación | Obligatorio |
|:--|:--|:--|:--|:--:|
| brigadista | Identificador del brigadista responsable de la atención. | Texto (ID de usuario) | Debe existir en MS01, estar activo y tener asignación vigente en el evento. | Sí (solo para asignar / reasignar) |
| estado | Estado del incidente en el momento de consulta. | Texto (enum) | Reportado al momento de crearse, si no hay brigadistas asignados a la zona reportada. Si los hay estará en estado Asignado y el brigadista lo verá como Pendiente. | Sí |

### 3. Datos de salida

| Nombre | Descripción | Tipo de campo | Reglas / Aplicación | Obligatorio |
|:--|:--|:--|:--|:--:|
| estadoAnterior | Estado antes de aplicar la transición. | Texto (enum) | Deben corresponder a una transición válida. | Sí |
| estadoNuevo | Estado después de aplicar la transición. | Texto (enum) | Deben corresponder a una transición válida. | Sí |
| responsable | Brigadista responsable vigente de la solicitud. | Texto (ID de usuario) | Se informa a partir de que la solicitud es asignada. | Sí |
| fechaCambio | Fecha y hora en que ocurrió la transición. | Fecha/hora | Generada automáticamente por el sistema. | Sí |

### 4. Flujo básico

| Paso | Actor | Descripción | Excepciones |
|:--:|:--|:--|:--|
| | Usuario general | Se autentica, identifica un evento activo y crea un incidente. | - |
| | Sistema | Revisa si el evento está activo y si hay brigadistas activos asignados a la zona del reporte del incidente. Si los hay se guarda como Asignado, de lo contrario se guarda como Reportado. | - |
| 1 | Coordinador | Se autentica, accede a la gestión de brigadistas y asigna o reasigna zonas a un brigadista. | - |
| 2 | Sistema | Valida que el brigadista exista y este activo y que la zona exista. | Si falla, ver Flujo alterno A1 o A2. |
| 3 | Sistema | Cambia el estado de los incidentes asociados a la zona a Asignado y registra el responsable y la fecha y hora. | Si hay incidentes sin resolver y solo queda un brigadista, ese brigadista no se podrá reasignar hasta que se resuelvan todos los incidentes. |
| 4 | Brigadista | Se autentica, ingresa al panel de responsabilidades e inicia la atención de las solicitudes en orden de prioridad. | |
| 5 | Sistema | Cambia el estado a En atención y registra la hora de primera atención. | - |
| 6 | Brigadista | Registra las observaciones y marca la solicitud como resuelta. | - |
| 7 | Sistema | Cambia el estado a Resuelta y registra la fecha y hora. | - |
| 8 | Coordinador | Verifica la resolución y cierra la solicitud. | - |
| 9 | Sistema | Cambia el estado a Cerrada, impide futuras modificaciones y registra cada transición en el módulo de auditoría. | - |

### 5. Flujo alterno

| Paso | Actor | Descripción | Excepciones |
|:--:|:--|:--|:--|
| A1 | Sistema | Detecta que la transición solicitada no corresponde al estado actual de la solicitud (por ejemplo, resolver una solicitud Reportada). | Se rechaza la operación con un error controlado. |
| A2 | Sistema | Detecta que el brigadista indicado no existe, está inactivo o no tiene asignación vigente en el evento. | Se rechaza la operación indicando el motivo. |
| A4 | Coordinador | Solicita reasignar una zona asignada a otro brigadista. | - |
| A4 | Sistema | Revisa si existen incidentes sin cerrar y Actualiza el responsable sin modificar el estado de la solicitud y registra el cambio en auditoría. | Si la solicitud está Resuelta o Cerrada, se rechaza la operación. |

<p align="right"><a href="#indice">⬆ Volver al índice</a></p>

---

## 📌 ANÁLISIS DE REQUERIMIENTOS · RF-BEI-08 <a id="rf-bei-08"></a>

### 1. Funcionalidad

| | |
|:--|:--|
| **Código** | RF-BEI-08 |
| **Nombre** | Seguimiento y cierre de incidentes |
| **Descripción** | Le permite al coordinador consultar los incidente y obtener un listado de incidentes ordenados por edificio, antigüedad, prioridad y estado. |
| **Actor principal** | Coordinador de emergencia y brigadista |
| **Precondiciones** | El coordinador debe estar autenticado. El brigadista puede consultar los incidentes de sus ubicaciones asignadas y los que reportó o tiene a cargo. |

**Cómo se ejecutará**

- **Automáticamente:** -
- **Manualmente:** **El coordinador de emergencia**: ingresa al aplicativo, se autentica, se dirige a la sección de incidentes y consulta los incidentes activos (durante el evento) y los históricos (cuando no hayan eventos activos) Solo podrá borrar un incidente en estado resuelto y una vez lo cierre no podrá editar nada relacionado la incidente. **Brigadista**: ingresa al aplicativo, se autentica, se dirige a la sección del panel de responsabilidades y consulta los incidentes activos (durante el evento) y los históricos (cuando no hayan eventos activos).

### 2. Datos de entrada

| Nombre | Descripción | Tipo de campo | Reglas / Aplicación | Obligatorio |
|:--|:--|:--|:--|:--:|
| prioridad | Filtro por prioridad del incidente. | Texto (lista) | Valor del catálogo de categorías de RF-BRI-07. | No |
| estado | Filtro por estado del incidente. | Texto (enum) | Valor del catálogo: Reportado, En atención, Resuelto, Cerrado. | No |
| zona | Filtro por ubicación | Texto / Numérico | Valores positivos; | No |

### 3. Datos de salida

| Nombre | Descripción | Tipo de campo | Reglas / Aplicación | Obligatorio |
|:--|:--|:--|:--|:--:|
| incidente (consulta puntual) | Información completa: identificador, evento, categoría, ubicación, descripción, estado, reportante, responsable, fechas de cada transición y evidencias asociadas. | Objeto | Se retorna únicamente si el identificador existe. | Sí |
| listado (consulta general) | Lista de incidentes con categoría, ubicación, estado, responsable y fecha de reporte. | Lista de objetos | Filtrada según los parámetros recibidos. | Sí |
| metadatosPaginacion | Información de paginación: total de registros y página actual. | Objeto | Se retorna junto con el listado. | Sí |

### 4. Flujo básico

| Paso | Actor | Descripción | Excepciones |
|:--:|:--|:--|:--|
| 1 | Coordinador o brigadista | Se autentica, revisa la vista de incidentes, según corresponda, y puede aplicar alguno de los filtros previamente expuestos. | - |
| 2 | Sistema | Verifica que el incidente exista y que el actor sea un coordinador o brigadista | - |
| 3 | Sistema | Retorna la información completa del incidente, incluyendo responsable, hora, ubicación y estado. | Si no existe, ver Flujo alterno A1. |

### 5. Flujo alterno

| Paso | Actor | Descripción | Excepciones |
|:--:|:--|:--|:--|
| A1 | Sistema | Detecta que no hay incidentes con las condiciones relacionadas | Retorna un error controlado indicando que el recurso no existe. |
| A1 | Sistema | Aplica los filtros recibidos, pagina los resultados y retorna el listado con los metadatos de paginación. | Si ningún filtro es válido, se ignora y se retorna el listado completo paginado. |

<p align="right"><a href="#indice">⬆ Volver al índice</a></p>

---

## 📌 ANÁLISIS DE REQUERIMIENTOS · RF-BEI-09 <a id="rf-bei-09"></a>

### 1. Funcionalidad

| | |
|:--|:--|
| **Código** | RF-BEI-09 |
| **Nombre** | Auditoría |
| **Descripción** | Registra las acciones relevantes realizadas en MS05 (asignaciones, creación de solicitudes e incidentes, cambios de estado, reasignaciones y evidencias), indicando usuario o servicio, acción, recurso afectado, fecha y hora, y resultado de la operación, y permite su consulta. |
| **Actor principal** | Sistema (registro) / Auditor - Coordinador de emergencia (consulta) |
| **Precondiciones** | Para consultar, el coordinador o auditor debe estar autenticado. |

**Cómo se ejecutará**

- **Automáticamente:** El sistema detecta un cambio de estado relevante o una acción destacable durante el flujo de su funcionamiento y guarda los datos más importantes asociados a esta.
- **Manualmente:** -

### 2. Datos de entrada

| Nombre | Descripción | Tipo de campo | Reglas / Aplicación | Obligatorio |
|:--|:--|:--|:--|:--:|
| tipoRegistro | Describe la acción efectuada sobre el módulo. | Texto | El tipo debe ser Asignación, Solicitud, Incidente o Evidencia. | Sí |
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
| 1 | Sistema | Detecta que ocurrió una acción auditable (por ejemplo, una asignación o un cambio de estado). | - |
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
| 1 | Un incidente tiene el siguiente flujo: Reportado → Asignado → En atención → Resuelto → Cerrado. No se permiten saltos ni retrocesos. |
| 2 | Un incidente en estado final (Cerrado) no puede modificarse en ninguna circunstancia. |
| 3 | Solo el coordinador de emergencia puede crear, modificar y retirar asignaciones y cerrar incidentes |
| 4 | Un brigadista puede iniciar atención y resolver las solicitudes e incidentes de los que es responsable. |
| 5 | Pueden existir varias solicitudes de asistencia por usuario solicitante y evento. |
| 6 | Las solicitudes provenientes de módulo de check-in deben incluir el usuario y zona del check-in de origen, para mantener la trazabilidad entre ambos servicios. |
| 7 | Un incidentes solo podrá registrarse mientras el evento esté activo. La atención, resolución y cierre pueden continuar después de finalizado el evento. |
| 8 | Toda transición de estado debe registrar la fecha y hora en que ocurre, de modo que el módulo de auditoría y analítica pueda calcular indicadores como el tiempo hasta la primera atención. |
| 9 | Toda acción relevante (asignación, modificación, retiro, creación, cambios de estado y evidencias) debe quedar registrada en el módulo de auditoría con el usuario o servicio responsable, la acción, el recurso afectado, y la fecha/hora. |
| 10 | Se pueden subir máximo 5 evidencias en formato png, pdf, docs, mp4. |
| 11 | Solo el módulo de eventos puede acceder directamente a la base de datos de brigadistas e incidentes; toda interacción debe realizarse mediante los endpoints expuestos. |
| 12 | Una asignación solo puede estar en alguno de los siguientes estados: Vigente, Retirada. |
| 13 | Una vez un incidente inicia a ser atendido solo aparecerá en la pantalla del brigadista que inicio la atención. |
| 14 | Solo se puede asignar una zona a la vez a un brigadista. |

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
| CHK | Módulo asociado a Evacuación y check-in |

<p align="right"><a href="#indice">⬆ Volver al índice</a></p>

---

## 🕓 HISTORIAL DE REVISIÓN <a id="historial-de-revision"></a>

| Elaborado por | Revisado por | Fecha | Descripción y justificación de cambios |
|:--|:--|:--:|:--|
| Equipo Zephyros | Pendiente | 28/09/2026 | Versión inicial del documento de análisis de requerimientos de MS05 - Brigadistas e Incidentes. |

<p align="right"><a href="#indice">⬆ Volver al índice</a></p>
