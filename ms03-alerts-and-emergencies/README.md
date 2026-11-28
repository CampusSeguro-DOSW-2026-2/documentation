# MS03 — Alertas y emergencias

Alertas de emergencia y de simulacro: creación, programación, activación y archivado, notificación por correo a los destinatarios, bandeja del usuario con confirmación de recepción, catálogo de tipos de alerta replicado desde MS06 y auditoría de cada alerta. Código: repositorio [`ms03-alerts-and-emergencies`](https://github.com/CampusSeguro-DOSW-2026-2/ms03-alerts-and-emergencies).

Cada tema se documenta en un solo archivo; los demás enlazan a él.

| Documento | Única fuente de |
|---|---|
| [`1_requirements.md`](1_requirements.md) | Comportamiento: RF-ALT-01…06 y reglas de negocio. |
| [`2_forms-inputs.md`](2_forms-inputs.md) | Formularios y entradas: campos, validaciones y mensajes. |
| [`3_data-model.md`](3_data-model.md) | Tablas, relaciones, reglas de integridad y migraciones. |
| [`api/openapi.yml`](api/openapi.yml) | Contrato OpenAPI 3.1: endpoints, permisos (`x-permiso`), esquemas y errores. |
| [`quality-and-tests.md`](quality-and-tests.md) | Pruebas, cobertura y análisis estático. |
| [`security.md`](security.md) | Identidad, autorización por permiso, llamadas salientes y datos sensibles. |

### Diagramas

| Vista | Diagrama |
|---|---|
| C4 · Contexto | [MS03 y su entorno](diagrams/ms03-c4-1-context.drawio.svg) |
| C4 · Contenedores | [Contenedores de MS03](diagrams/ms03-c4-2-containers.drawio.svg) |
| C4 · Componentes | [Componentes internos](diagrams/ms03-c4-3-components.drawio.svg) |
| Clases | [Dominio](diagrams/ms03-classes.drawio.svg) |
| Modelo de datos | [Entidad-relación](diagrams/ms03-er.drawio.svg) |
| Flujos | [Flujos de alertas](diagrams/ms03-flows.drawio.pdf) |
| Infraestructura | [Despliegue](diagrams/ms03-infrastructure.drawio.svg) |

Los archivos `.drawio.svg` se abren y se editan con [draw.io](https://app.diagrams.net/) (o con su extensión de VS Code) sin perder el diagrama.

**Para otros squads:** las reglas transversales están en los [Estándares](../integration/standards.md) y la asignación inicial de permisos a los roles en la [matriz de roles y permisos](../integration/permissions-matrix.md).
