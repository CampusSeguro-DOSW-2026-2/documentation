# MS04 — Evacuación y check-in

Reporte del estado de las personas (Estoy a salvo, Me estoy evacuando, Requiero asistencia, No estoy en el campus) y consolidación por punto de encuentro, durante los eventos (simulacros y emergencias) que define MS06. MS06 le informa a MS04 el estado de cada evento (programado, en ejecución, finalizado) y MS04 habilita el check-in solo mientras el evento está en ejecución. Código: repositorio [`ms04-evacuation-and-check-in`](https://github.com/CampusSeguro-DOSW-2026-2/ms04-evacuation-and-check-in).

| Documento | Contenido |
|---|---|
| [`1_requirements.md`](1_requirements.md) | Estados del evento y del check-in, RF-MS04-01…13, reglas de negocio y RNF. |
| [`2_forms-inputs.md`](2_forms-inputs.md) | Formularios y entradas F-01…F-07 (incluidas las rutas internas de MS06): campos, validaciones y mensajes. |
| [`3_data-model.md`](3_data-model.md) | Justificación de la base relacional, tablas (eventos, check-ins y solicitudes de asistencia), vista del estado vigente, reglas de integridad y operaciones. |
| [`quality-and-tests.md`](quality-and-tests.md) | Estrategia y tecnologías de prueba, resultados, cobertura, análisis estático (PMD y SpotBugs) y relación con SonarCloud. |
| [`security.md`](security.md) | Identidad propagada por el Gateway, autorización por permiso, rutas internas de MS06, clave de servicio, mTLS y manejo de datos sensibles. |
| [`api/openapi.yml`](api/openapi.yml) | Contrato OpenAPI 3 (rutas `/api/v1/chi/eventos`): endpoints, permisos (`x-permiso`), esquemas y errores. |
| [`diagrams/`](diagrams/) | C4 de componentes, modelo de datos físico, clases y despliegue en AWS. |

**Para otros squads:** las reglas transversales están en los [Estándares](../integration/standards.md) y los permisos en la [matriz de roles y permisos](../integration/permissions-matrix.md). Las llamadas de MS04 a MS01, MS02 y MS05, las de MS06 a MS04 y los eventos de auditoría hacia MS08 están en la [tabla de llamadas internas](../integration/standards.md#5-llamadas-internas-ms--gw--ms) y en el [catálogo de auditoría](../integration/audit-catalog.md#ms04--evacuación-y-check-in).
