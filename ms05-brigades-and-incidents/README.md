# MS05 — Brigadistas e incidentes

Activación, búsqueda y asignación de brigadistas a edificios, zonas seguras y puntos de encuentro, auto asignación por evento, panel de responsabilidades, registro y atención de incidentes y solicitudes de asistencia (reportado, asignado, en atención, resuelto y cerrado) con su historial y sus evidencias, y auditoría de cada operación. MS06 le informa el estado de cada evento. Código: repositorio [`ms05-brigades-and-incidents`](https://github.com/CampusSeguro-DOSW-2026-2/ms05-brigades-and-incidents).

| Documento | Contenido |
|---|---|
| [`1_requirements.md`](1_requirements.md) | RF-BEI-01…09 y reglas de negocio. |
| [`2_forms-inputs.md`](2_forms-inputs.md) | Formularios y entradas: campos, validaciones y mensajes. |
| [`3_data-model.md`](3_data-model.md) | Tablas, catálogos, relaciones y reglas de integridad de la base de datos de MS05. |
| [`api/openapi.yml`](api/openapi.yml) | Contrato OpenAPI 3: endpoints, permisos (`x-permiso`), esquemas y errores. |
| [`quality-and-tests.md`](quality-and-tests.md) | Pruebas, cobertura y análisis estático. |
| [`security.md`](security.md) | Identidad, autorización por permiso, rutas internas de MS06, llamadas salientes y evidencias. |

### Diagramas

| Vista | Diagrama |
|---|---|
| C4 · Contexto | [MS05 y su entorno](diagrams/ms05-c4-1-context.drawio.svg) |
| C4 · Contenedores | [Contenedores de MS05](diagrams/ms05-c4-2-containers.drawio.svg) |
| C4 · Componentes | [Componentes internos](diagrams/ms05-c4-3-components.drawio.svg) |
| Clases | [Modelo de software](diagrams/ms05-classes.drawio.svg) |
| Modelo de datos | [Entidad-relación](diagrams/ms05-er.drawio.svg) |
| Infraestructura | [Despliegue](diagrams/ms05-infrastructure.drawio.svg) |
| Interfaz | [Baja fidelidad](diagrams/ms05-low-fidelity.png) |

Los archivos `.drawio.svg` se abren y se editan con [draw.io](https://app.diagrams.net/) (o con su extensión de VS Code) sin perder el diagrama.

**Para otros squads:** las reglas transversales están en los [Estándares](../integration/standards.md) y los permisos en la [matriz de roles y permisos](../integration/permissions-matrix.md). Las llamadas de MS05 a MS01, MS02 y MS03, las de MS04 y MS06 a MS05 y los eventos de auditoría hacia MS08 están en la [tabla de llamadas internas](../integration/standards.md#5-llamadas-internas-ms--gw--ms) y en el [catálogo de auditoría](../integration/audit-catalog.md#ms05--brigadistas-e-incidentes).
