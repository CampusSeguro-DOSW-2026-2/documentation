# Pendiente de llevar al README del backend

Contenido de tipo técnico sobre el código de MS05 que no pertenece a la documentación de este repositorio y debe vivir en el `README.md` del repositorio [`ms05-brigades-and-incidents`](https://github.com/CampusSeguro-DOSW-2026-2/ms05-brigades-and-incidents), junto a las rutas, los permisos, las pruebas y el despliegue que ese README ya describe. Una vez llevado, este archivo se elimina.

## Funcionalidades principales

- Búsqueda y activación de brigadistas.
- Asignación de brigadistas a edificios, zonas seguras y puntos de encuentro por evento, con auto asignación del brigadista y panel de gestión del coordinador.
- Registro y edición de incidentes, incluidos los que genera una solicitud de asistencia del check-in.
- Atención de incidentes: asignación del responsable, inicio de atención, resolución y cierre.
- Consulta del historial de cada incidente y de sus evidencias.
- Configuración de la operación de cada evento a partir del estado que informa MS06.
- Registro y consulta de auditoría de las operaciones.

## Tecnologías

| Elemento | Tecnología |
|---|---|
| Lenguaje | Java 21 |
| Framework | Spring Boot 4 (Spring Web MVC, Spring Security, Bean Validation y Actuator) |
| Gestión de dependencias | Maven (con `mvnw`) |
| Persistencia | Spring Data JPA con PostgreSQL 17; el esquema se valida al arrancar (`ddl-auto=validate`) y lo crean los scripts de `database/` |
| Evidencias | Archivos en el sistema de archivos del servicio, en el directorio de `EVIDENCIAS_DIRECTORIO` (por defecto `./uploads/evidencias`) |
| Calidad | JaCoCo (80 % de cobertura de líneas) y SpotBugs |
| Contenedores | Docker (imagen multi-etapa con Eclipse Temurin 21) y Docker Compose |
| Puerto | `8085` (`SERVER_PORT`) |
