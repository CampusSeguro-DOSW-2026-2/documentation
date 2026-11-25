# MS08 — Analítica y Auditoría

Registro central de auditoría de CampusSeguro e indicadores de simulacros y eventos reales: recibe los hechos que envían los demás microservicios, los conserva 6 meses, permite consultarlos y calcula los indicadores de cada evento y su comparación. Código: repositorio [`ms08-analytics-and-audit`](https://github.com/CampusSeguro-DOSW-2026-2/ms08-analytics-and-audit).

Cada tema se documenta en un solo archivo; los demás enlazan a él.

| Documento | Única fuente de |
|---|---|
| [`1_requirements.md`](1_requirements.md) | Comportamiento: RF-AYA-01…07, reglas de negocio y RNF. |
| [`2_forms-inputs.md`](2_forms-inputs.md) | Formularios F-01…F-06: campos, validaciones y mensajes. |
| [`3_data-model.md`](3_data-model.md) | Colección MongoDB, índices, retención, operaciones, configuración y datos iniciales. |
| [`api/openapi.yml`](api/openapi.yml) | Contrato OpenAPI 3: endpoints, permisos, esquemas y errores. |

**Para otros squads:** MS08 es la fuente de verdad de la auditoría. Cada servicio le envía sus hechos con `POST /api/v1/audit/events` a través del Gateway, usando las acciones de su sección del [catálogo de auditoría](../integration/audit-catalog.md); el contrato es estricto y no admite datos personales. Solo MS08 accede a su base de datos. Las reglas transversales están en los [Estándares](../integration/standards.md) y la asignación inicial de permisos a los roles en la [matriz de roles y permisos](../integration/permissions-matrix.md).
