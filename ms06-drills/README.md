# MS06 — Eventos

Eventos y simulacros de emergencia del campus: creación, edición, programación, inicio automático, monitoreo, finalización y evaluación, y coordinación de los demás módulos cuando un evento cambia de estado. Código: repositorio [`ms06-drills`](https://github.com/CampusSeguro-DOSW-2026-2/ms06-drills).

Cada tema se documenta en un solo archivo; los demás enlazan a él.

| Documento | Única fuente de |
|---|---|
| [`1_requirements.md`](1_requirements.md) | Comportamiento: RF-EVE-01…05, auditoría, reglas de negocio y RNF. |
| [`2_forms-inputs.md`](2_forms-inputs.md) | Formularios F-01…F-06: campos, validaciones y mensajes. |
| [`3_data-model.md`](3_data-model.md) | Tablas, bandeja de auditoría, operaciones en BD, configuración y datos iniciales. |
| [`api/openapi.yml`](api/openapi.yml) | Contrato OpenAPI 3: endpoints, permisos (`x-permiso`), esquemas y errores. |
| [`quality-and-tests.md`](quality-and-tests.md) | Pruebas, cobertura y análisis estático. |
| [`security.md`](security.md) | Identidad, autorización por permiso, llamadas salientes y datos sensibles. |

### Diagramas

| Vista | Diagrama |
|---|---|
| C4 · Contexto | [MS06 y su entorno](diagrams/ms06-c4-1-context.drawio.svg) |
| C4 · Contenedores | [Contenedores de MS06](diagrams/ms06-c4-2-containers.drawio.svg) |
| C4 · Componentes | [Controllers, Services, Schedulers, Repositories y Gateways](diagrams/ms06-c4-3-components.drawio.svg) |
| Clases | [Paquetes y relaciones](diagrams/ms06-classes.drawio.svg) |
| Modelo de datos | [Entidad-relación](diagrams/ms06-er.drawio.svg) |
| Flujos | [Flujos de los eventos](diagrams/ms06-flows-events.pdf) |
| Infraestructura | [Despliegue](diagrams/ms06-infrastructure.drawio.svg) |
| Interfaz | [Baja fidelidad](diagrams/ms06-low-fidelity.png) |

Los archivos `.drawio.svg` se abren y se editan con [draw.io](https://app.diagrams.net/) (o con su extensión de VS Code) sin perder el diagrama.

**Para otros squads:** MS06 es la fuente de verdad del estado de un evento. Recibe las peticiones por el Gateway y llama a MS02 (valida edificios y zonas), MS03 (alertas), MS04 (check-in), MS05 (brigadistas e incidentes) y MS08 (auditoría). Con MS03 comparte el catálogo de tipos de alerta (`GET /api/v1/tipos-alerta`) y a MS04 y MS05 les informa el estado del evento y su escenario con los nombres de los edificios y zonas. Las reglas transversales están en los [Estándares](../integration/standards.md) y la asignación inicial de permisos a los roles en la [matriz de roles y permisos](../integration/permissions-matrix.md).
