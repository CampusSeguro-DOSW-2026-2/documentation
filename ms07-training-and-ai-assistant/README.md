# MS07 – Formación y asistente IA

Documentación del microservicio de formación preventiva y asistente de inteligencia artificial de CampusSeguro.

## Documentos

- [Requerimientos](1_requirements.md): funcionalidades, reglas de negocio y comportamiento del servicio.
- [Modelo de datos](3_data-model.md): entidades, tablas, relaciones y restricciones.
- [Contrato API](api/openapi.yml): especificación OpenAPI 3 de los endpoints.
- [Diagramas](diagrams/README.md): componentes, despliegue, modelo de datos físico y clases.

## Alcance

MS07 administra categorías y contenidos de formación preventiva, evaluaciones, seguimiento del progreso de los usuarios y consultas al Tutor CampusSeguro.

## Integración

La autenticación y la propagación de identidad son responsabilidad del API Gateway. MS07 recibe las cabeceras de identidad y autorización y valida los permisos mediante `X-User-Permission`.

- La asignación de permisos a los roles se encuentra en la [Matriz de permisos](../integration/permissions-matrix.md).
- Las reglas transversales de comunicación están definidas en los [Estándares de integración](../integration/standards.md).
- Los eventos de auditoría se rigen por el [Catálogo de auditoría](../integration/audit-catalog.md).
- La comunicación con el API Gateway sigue la [Guía de conexión con Gateway](../integration/gateway-connection.md).
