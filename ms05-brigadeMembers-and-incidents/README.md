# MS05 — Brigadistas e Incidentes

**Plataforma:** CampusSeguro  
**Equipo:** Zephyros  
**Repositorio:** [ms05-brigades-and-incidents](https://github.com/CampusSeguro-DOSW-2026-2/ms05-brigades-and-incidents)

## 1. Descripción general

El microservicio MS05 es responsable de gestionar los brigadistas, sus asignaciones y los incidentes que ocurren dentro del campus. Su propósito es facilitar la coordinación de la atención de incidentes, permitiendo registrar situaciones, asignar responsables y realizar seguimiento hasta su cierre.

Este componente hace parte de CampusSeguro, una plataforma orientada a apoyar la prevención y respuesta ante situaciones de emergencia en el entorno universitario.

## 2. Funcionalidades principales

- Gestión de brigadistas.
- Administración de asignaciones a eventos y operaciones.
- Registro y seguimiento de incidentes.
- Gestión de los estados de los incidentes.
- Consulta del historial de incidentes.
- Registro de auditoría de las operaciones.
- Administración de la configuración de operación de eventos.

## 3. Tecnologías utilizadas

- **Lenguaje:** Java 21.
- **Framework:** Spring Boot.
- **Gestión de dependencias:** Maven.
- **Persistencia:** Spring Data JPA.
- **Base de datos:** PostgreSQL.
- **Contenedores:** Docker.
- **Documentación de API:** OpenAPI / Swagger.
- **Seguridad:** autenticación mediante encabezados enviados por el API Gateway.

## 4. Estructura de la documentación

La documentación del microservicio está organizada en tres carpetas principales:

| Carpeta | Contenido |
|---|---|
| `requerimientos/` | Requerimientos funcionales, no funcionales y backlog de historias de usuario. |
| `api/` | Contrato de la API documentado mediante OpenAPI. |
| `diagramas/` | Diagramas de arquitectura, diseño y modelo de datos. |

### 4.1 Requerimientos y backlog

Documentos relacionados con las necesidades, funcionalidades e historias de usuario del microservicio:

- [Requerimientos MS05](requerimientos/Requerimientos_MS05_BrigadistasEIncidentes.md)
- [Backlog MS05](requerimientos/Backlog_MS05_BrigadistasEIncidentes.docx)

### 4.2 API

El contrato de la API describe los endpoints, métodos HTTP, parámetros, solicitudes y respuestas disponibles para la interacción con MS05.

- [Contrato OpenAPI](api/openapi.yml)

### 4.3 Diagramas

Los diagramas permiten comprender la arquitectura, organización interna, relaciones y funcionamiento del microservicio.

1. **Diagrama de contexto:** representa MS05 y su relación con los usuarios y otros microservicios de CampusSeguro.

   [Ver diagrama de contexto](diagramas/diagrama_contexto%28ms05%29.drawio.svg)

2. **Diagrama de contenedores:** muestra los principales contenedores y elementos tecnológicos que componen el microservicio.

   [Ver diagrama de contenedores](diagramas/diagrama_contenedores%28ms05%29.drawio.svg)

3. **Diagrama de componentes:** representa la organización de los componentes internos y sus responsabilidades.

   [Ver diagrama de componentes](diagramas/diagrama_componentes%28ms05%29.drawio.svg)

4. **Diagrama de clases:** describe las principales clases, atributos, operaciones y relaciones del modelo de software.

   [Ver diagrama de clases](diagramas/diagrama_clases%28ms05%29.drawio.svg)

5. **Diagrama entidad-relación:** representa las entidades de la base de datos y las relaciones entre ellas.

   [Ver diagrama entidad-relación](diagramas/diagrama-entidad-relacion%28ms05%29.drawio.svg)

6. **Diagrama de infraestructura:** representa la infraestructura tecnológica y la distribución de los elementos necesarios para ejecutar el microservicio.

   [Ver diagrama de infraestructura](diagramas/diagrama-infraestructura.drawio.svg)

7. **Diagrama de baja fidelidad:** presenta una representación inicial de las interfaces y funcionalidades visuales del sistema.

   [Ver diagrama de baja fidelidad](diagramas/diagramaBajaFidelidadMS05.png)

## 5. Gestión de incidentes

Los incidentes siguen un ciclo de vida compuesto por los siguientes estados:

`REPORTADO → ASIGNADO → EN_ATENCION → RESUELTO → CERRADO`

Cada transición permite realizar seguimiento a la atención del incidente y conservar su historial. Una vez que un incidente alcanza el estado `CERRADO`, no puede modificarse.

## 6. Modelo de datos

MS05 utiliza PostgreSQL para almacenar y gestionar la información relacionada con brigadistas, asignaciones e incidentes.

Entre sus principales entidades se encuentran:

- **Brigadista:** almacena la información de los integrantes de las brigadas.
- **AsignacionBrigadista:** representa las asignaciones de los brigadistas.
- **Incidente:** contiene la información de los incidentes registrados.
- **HistorialIncidente:** conserva los cambios realizados sobre los incidentes.
- **ConfiguracionOperacionEvento:** almacena la configuración asociada a la operación de un evento.
- **RegistroAuditoria:** permite realizar seguimiento y trazabilidad de las operaciones.

## 7. Seguridad e integración

MS05 recibe la información de identidad y rol mediante los encabezados enviados por el API Gateway:

| Encabezado | Descripción |
|---|---|
| `X-User-Id` | Identificador del usuario que realiza la solicitud. |
| `X-User-Role` | Rol del usuario dentro de la plataforma. |

Entre los roles contemplados se encuentran:

- `COORDINATOR`
- `BRIGADISTA`
- `AUDITOR`
- `SERVICE_MS06`

El microservicio contempla la integración con otros componentes de CampusSeguro:

- **MS01 — Identidad y Acceso:** proporciona el contexto de identidad y autorización de los usuarios.
- **MS02 — Campus y Seguridad:** aporta información relacionada con las instalaciones y la seguridad del campus.
- **MS06 — Eventos:** permite la interacción con la gestión de eventos y sus operaciones.

Las operaciones disponibles dependen de los permisos y reglas de negocio definidos para cada funcionalidad.

---

**CampusSeguro — Equipo Zephyros**  
*Documentación del microservicio MS05 — Brigadistas e Incidentes.*