# MS01 — Identidad y acceso

Usuarios, autenticación, roles, permisos y sesiones. Código: repositorio [`ms01-identity-and-access`](https://github.com/CampusSeguro-DOSW-2026-2/ms01-identity-and-access).

Cada tema se documenta en un solo archivo; los demás enlazan a él.

| Documento | Única fuente de |
|---|---|
| [`1_requirements.md`](1_requirements.md) | Comportamiento: RF-MS01-01…11, reglas de negocio y RNF. |
| [`2_forms-inputs.md`](2_forms-inputs.md) | Formularios F-01…F-14: campos, validaciones y mensajes. |
| [`jwt-spec.md`](jwt-spec.md) | Detalle técnico de la sesión: token, cabeceras de identidad, ciclo de vida, corte de sesión y llaves. |
| [`3_data-model.md`](3_data-model.md) | Justificación de la base relacional, tablas (incluida `roles`), almacenamiento seguro, operaciones en BD, configuración y datos iniciales. |
| [`quality-and-tests.md`](quality-and-tests.md) | Estrategia y tecnologías de prueba, resultados, cobertura, análisis estático (PMD y SpotBugs) y relación con SonarCloud. |
| [`security.md`](security.md) | Registro, autenticación, formación del JWT, identidad propagada, rutas internas, contraseñas y manejo de datos sensibles. |
| [`api/openapi.yml`](api/openapi.yml) | Contrato OpenAPI 3: endpoints, permisos (`x-permiso`), esquemas y errores. |
| [`integration-guide.md`](integration-guide.md) | Guía de integración para el Gateway y los demás squads: qué recibe su servicio de MS01, roles y permisos, y consulta de datos de usuario. |

### Diagramas

Cómo verlos y editarlos: [`diagrams/README.md`](diagrams/README.md).

| Vista | Diagrama |
|---|---|
| C4 · Componentes | [Controllers, Services y Repositories](diagrams/ms01-c4-3-components.drawio.svg) |
| C4 · Código | [Clases del dominio](diagrams/ms01-classes.svg) ([`.puml`](diagrams/ms01-classes.puml)) |
| Despliegue | [EC2 en AWS con firewall y mTLS, entornos de prueba y producción y Amazon RDS](diagrams/ms01-deployment-aws.drawio.svg) |
| Modelo de datos físico | [Tablas de PostgreSQL](diagrams/ms01-er-physical.drawio.svg) |

**Para otros squads:** empiece por la [guía de integración](integration-guide.md). Las reglas transversales están en los [Estándares](../integration/standards.md) y la asignación inicial de permisos a los roles en la [matriz de roles y permisos](../integration/permissions-matrix.md).
