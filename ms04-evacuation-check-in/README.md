# MS04 — Evacuación y check-in

Reporte del estado de las personas (Estoy a salvo, Me estoy evacuando, Requiero asistencia, No estoy en el campus) y consolidación por punto de encuentro, durante los eventos (simulacros y emergencias) que define MS06. MS06 le informa a MS04 el estado de cada evento (programado, en ejecución, finalizado) y MS04 habilita el check-in solo mientras el evento está en ejecución. Código: repositorio [`ms04-evacuation-and-check-in`](https://github.com/CampusSeguro-DOSW-2026-2/ms04-evacuation-and-check-in).

| Documento | Contenido |
|---|---|
| [`1_requirements.md`](1_requirements.md) | Estados del evento y del check-in, RF-MS04-01…13, reglas de negocio, RNF, dependencias y decisiones abiertas, y tabla de endpoints con permisos. |
| [`2_forms-inputs.md`](2_forms-inputs.md) | Formularios y entradas F-01…F-07 (incluidas las rutas internas de MS06): campos, validaciones y mensajes. |
| [`3_modelo-datos.md`](3_modelo-datos.md) | Tablas (eventos, check-ins y solicitudes de asistencia), vista del estado vigente, reglas de integridad y operaciones. |
| [`api/swagger.yml`](api/swagger.yml) | Contrato OpenAPI 3 (rutas `/api/v1/chi/eventos`). |
| [`diagramas/`](diagramas/) | Componentes (C4 nivel 3), modelo de datos físico, clases y despliegue en AWS (propuesta). |

**Para otros squads:** la guía de integración de MS04 en la wiki está pendiente. Las reglas transversales están en los [Estándares](https://github.com/CampusSeguro-DOSW-2026-2/documentation/wiki/Est%C3%A1ndares-de-Integraci%C3%B3n-y-Contratos-Transversales) y los permisos en la [Matriz Rol‐Permiso](https://github.com/CampusSeguro-DOSW-2026-2/documentation/wiki/Matriz-Rol%E2%80%90Permiso). Los contratos que MS04 necesita de MS01, MS02, MS05, MS06 y MS08 están en [Dependencias y decisiones abiertas](1_requirements.md#dependencias-y-decisiones-abiertas).
