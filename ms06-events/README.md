# MS06 — Eventos

Eventos y simulacros de emergencia del campus: creación, edición, programación, inicio automático, monitoreo, finalización y evaluación, y coordinación de los demás módulos cuando un evento cambia de estado. Código: repositorio [`ms06-drills`](https://github.com/CampusSeguro-DOSW-2026-2/ms06-drills).

Cada tema se documenta en un solo archivo; los demás enlazan a él.

| Documento | Única fuente de |
|---|---|
| [`1_requirements.md`](1_requirements.md) | Comportamiento: RF-EVE-01…05, auditoría, reglas de negocio y RNF. |
| [`2_forms-inputs.md`](2_forms-inputs.md) | Formularios F-01…F-06: campos, validaciones y mensajes. |
| [`3_data-model.md`](3_data-model.md) | Tablas, bandeja de auditoría, operaciones en BD, configuración y datos iniciales. |
| [`api/openapi.yml`](api/openapi.yml) | Contrato OpenAPI 3: endpoints, permisos (`x-permiso`), esquemas y errores. |

### Diagramas

| Vista | Diagrama |
|---|---|
| C4 · Contexto | [Contexto](diagrams/c4-context.png) |
| C4 · Contenedores | [Contenedores](diagrams/c4-containers.png) |
| Componentes | [Controllers, Services, Schedulers, Repositories y Gateways](diagrams/componentes.png) |
| Clases | [Paquetes y relaciones](diagrams/clases.png) |
| Infraestructura | [PostgreSQL en Docker y módulos externos](diagrams/infraestructura.png) |
| Modelo de datos físico | [Tablas de PostgreSQL](diagrams/er.png) |
| Secuencia | [Ciclo de vida del evento](diagrams/ciclo-de-vida-del-evento.png) |

**Para otros squads:** MS06 es la fuente de verdad del estado de un evento. Recibe las peticiones por el Gateway y llama a MS02 (valida edificios y zonas), MS03 (alertas), MS04 (check-in), MS05 (brigadistas e incidentes) y MS08 (auditoría). Con MS03 comparte el catálogo de tipos de alerta (`GET /api/v1/tipos-alerta`) y a MS04 y MS05 les informa el estado del evento y su escenario con los nombres de los edificios y zonas. Las reglas transversales están en los [Estándares](../integration/standards.md) y la asignación inicial de permisos a los roles en la [matriz de roles y permisos](../integration/permissions-matrix.md).
