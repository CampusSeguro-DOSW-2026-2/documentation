# MS07 – Formación y asistente IA

Documentación del microservicio de formación preventiva y asistente de inteligencia artificial de CampusSeguro.

## Documentos

- [Requerimientos](1_requirements.md): funcionalidades, reglas de negocio y comportamiento del servicio.
- [Modelo de datos](3_data-model.md): entidades, tablas, relaciones y restricciones.
- [Contrato API](api/swagger.yml): especificación OpenAPI 3 de los endpoints.

## Alcance

MS07 administra categorías y contenidos de formación preventiva, evaluaciones, seguimiento del progreso de los usuarios y consultas al Tutor CampusSeguro.

## Integración

La autenticación y la propagación de identidad son responsabilidad del API Gateway. MS07 recibe las cabeceras de identidad y autorización y valida los permisos mediante `X-User-Permission`.

Los permisos utilizados por MS07 están definidos en la [Matriz Rol-Permiso](https://github.com/CampusSeguro-DOSW-2026-2/documentation/wiki/Matriz-Rol%E2%80%90Permiso).

Las reglas transversales de comunicación están documentadas en los [Estándares de Integración y Contratos Transversales](https://github.com/CampusSeguro-DOSW-2026-2/documentation/wiki/Est%C3%A1ndares-de-Integraci%C3%B3n-y-Contratos-Transversales).

La guía de integración específica de MS07 se publicará en la Wiki del repositorio central.
