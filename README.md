# MS03 — Alertas y Emergencias

Documentación del microservicio **MS03 — Alertas y Emergencias** de la plataforma CampusSeguro. Este microservicio se encarga de gestionar las alertas relacionadas con situaciones de emergencia y eventos programados, desde su creación y programación hasta su activación, distribución, seguimiento y finalización.

Esta carpeta centraliza los requerimientos, el contrato de API y los diagramas que describen la arquitectura, la estructura y el funcionamiento del microservicio.

## Información general

| **Elemento**           | **Descripción**               |
| ---------------------- | ----------------------------- |
| Microservicio          | MS03 — Alertas y Emergencias  |
| Nombre del repositorio | `ms03-alerts-and-emergencies` |
| Squad responsable      | Zephyros                      |
| Tecnología principal   | Java 21 y Spring Boot         |
| Base de datos          | PostgreSQL                    |
| Contrato de API        | OpenAPI 3.1                   |
| Puerto local           | `8080`                        |

## Responsabilidades principales

- **Gestión de alertas:** crear, consultar, modificar y administrar alertas.
- **Ciclo de vida:** programar, activar y finalizar alertas, respetando las reglas de transición entre estados.
- **Segmentación:** definir los alcances y destinatarios de las alertas según la información del campus.
- **Confirmación de recepción:** registrar las confirmaciones de los usuarios que reciben las alertas.
- **Notificaciones:** gestionar la distribución de alertas mediante correo electrónico.
- **Auditoría:** registrar y consultar los eventos relacionados con las operaciones realizadas sobre las alertas.
- **Integración:** consultar información de usuarios y destinatarios en MS01 y validar los alcances mediante MS02.
- **Integración con eventos:** gestionar alertas relacionadas con eventos y simulacros originados desde MS06, de acuerdo con el contrato de integración.

## Documentación del microservicio

La documentación de MS03 está organizada en las siguientes carpetas:

| **Documento**  | **Ubicación**                        | **Contenido**                                                                                       |
| -------------- | ------------------------------------ | --------------------------------------------------------------------------------------------------- |
| Requerimientos | [`requerimientos/`](requerimientos/) | Requerimientos funcionales, no funcionales y reglas de negocio del microservicio.                   |
| Contrato API   | [`api/`](api/)                       | Especificación OpenAPI de los endpoints, parámetros, cabeceras, esquemas y respuestas.              |
| Diagramas      | [`diagramas/`](diagramas/)           | Representaciones gráficas de la arquitectura, estructura, infraestructura y comportamiento de MS03. |

## Diagramas

Los diagramas del microservicio se encuentran en la carpeta [`diagramas/`](diagramas/). Estos permiten comprender MS03 desde diferentes perspectivas, tanto a nivel arquitectónico como técnico y funcional.

Se incluyen los siguientes diagramas:

### 1. Diagrama de contexto

Representa el microservicio MS03 dentro del ecosistema CampusSeguro. Identifica los actores, sistemas externos y microservicios con los que interactúa, mostrando los límites del sistema y las principales relaciones de comunicación.

### 2. Diagrama de contenedores

Muestra la organización de MS03 a nivel de contenedores y aplicaciones. Permite identificar los elementos principales que conforman la solución, sus responsabilidades y las comunicaciones entre ellos.

### 3. Diagrama de componentes

Representa la estructura interna del microservicio, mostrando sus principales componentes, responsabilidades, dependencias y relaciones. Permite comprender cómo se organiza el backend y cómo interactúan sus diferentes módulos.

### 4. Diagrama de clases

Describe la estructura estática del sistema mediante sus clases, atributos, métodos, relaciones, dependencias y responsabilidades. Incluye los elementos principales del dominio, servicios, controladores, repositorios, DTO y demás componentes de la implementación.

### 5. Diagrama entidad-relación

Representa el modelo de datos de MS03, identificando las entidades persistentes, sus atributos, claves y relaciones. Permite comprender cómo se organiza y almacena la información en la base de datos.

### 6. Diagramas de flujos alternos

Describen los caminos alternativos de los procesos del microservicio, incluyendo situaciones excepcionales, validaciones fallidas, errores y condiciones que modifican el flujo principal de una operación.

### 7. Diagrama de infraestructura

Representa la infraestructura tecnológica necesaria para desplegar y ejecutar MS03, identificando los elementos de infraestructura, servicios, conexiones y recursos involucrados en su funcionamiento.

## Integraciones con otros microservicios

MS03 se comunica con otros microservicios de CampusSeguro para obtener información y coordinar funcionalidades relacionadas con la gestión de alertas.

| **Microservicio**         | **Responsabilidad en la integración**                                          |
| ------------------------- | ------------------------------------------------------------------------------ |
| MS01 — Identidad y acceso | Consulta y validación de información relacionada con usuarios y destinatarios. |
| MS02 — Campus y Seguridad | Validación de los alcances asociados a edificios y zonas del campus.           |
| MS06 — Eventos            | Integración de alertas relacionadas con eventos y simulacros programados.      |

Las integraciones deben respetar los contratos establecidos entre los microservicios y los estándares transversales definidos para CampusSeguro.

## Seguridad y permisos

La autenticación y autorización de las solicitudes se gestionan mediante el API Gateway y los mecanismos de seguridad establecidos para la comunicación entre microservicios.

MS03 recibe información de identidad y contexto del usuario mediante las siguientes cabeceras:

- `X-User-Id`: identificador del usuario que realiza la solicitud.
- `X-User-Role`: rol del usuario.
- `X-Correlation-ID`: identificador de correlación para el seguimiento de las operaciones.

Los permisos funcionales asociados a MS03 se encuentran definidos en la matriz general de roles y permisos de CampusSeguro.

| **Permiso**               | **Descripción**                                              |
| ------------------------- | ------------------------------------------------------------ |
| `ms03:alert:receive`      | Recibir alertas y confirmar su recepción.                    |
| `ms03:alert:manage`       | Administrar alertas de emergencia.                           |
| `ms03:alert:manage_drill` | Administrar alertas de tipo simulacro originadas desde MS06. |
| `ms03:alert:read_history` | Consultar el historial de alertas.                           |

Consulta la [Matriz de Roles y Permisos](https://github.com/CampusSeguro-DOSW-2026-2/documentation/wiki/Matriz-Rol%E2%80%90Permiso) para conocer los roles autorizados para cada operación.

