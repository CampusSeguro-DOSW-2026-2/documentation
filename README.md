# CampusSeguro — Documentación

Repositorio central de la documentación de CampusSeguro: las reglas que comparten todos los squads, la documentación de cada microservicio (requerimientos, contrato OpenAPI, modelo de datos y diagramas) y la del API Gateway. Las guías paso a paso están en la [wiki](https://github.com/CampusSeguro-DOSW-2026-2/documentation/wiki).

## Dónde está cada cosa

| Qué | Dónde |
|---|---|
| Reglas transversales: puertos, JWT, cabeceras, llamadas internas, errores, correlación, paginación y auditoría | [`integration/standards.md`](integration/standards.md) |
| Roles y permisos de cada microservicio | [`integration/permissions-matrix.md`](integration/permissions-matrix.md) |
| Conexión de un microservicio al API Gateway en la nube | [`integration/gateway-connection.md`](integration/gateway-connection.md) |
| Acciones de auditoría de cada microservicio e indicadores que alimentan | [`integration/audit-catalog.md`](integration/audit-catalog.md) |
| Diagramas de arquitectura del sistema (C4 contexto y contenedores) | [`integration/architecture/`](integration/architecture/) |
| Infraestructura global de despliegue (Azure y AWS) | [`integration/architecture/architecture_diagram/architecture_diagram.drawio.svg`](integration/architecture/architecture_diagram/architecture_diagram.drawio.svg) |
| Requerimientos, contrato, modelo de datos y diagramas de cada microservicio | La carpeta del microservicio (tabla siguiente) |
| Guías paso a paso: agregar un microservicio al gateway y desplegar el frontend | [Wiki](https://github.com/CampusSeguro-DOSW-2026-2/documentation/wiki) |
| Convenciones para escribir y cambiar la documentación | [`CONTRIBUTING.md`](CONTRIBUTING.md) |

## Microservicios

| Servicio | Documentación | Código |
|---|---|---|
| API Gateway | [`api-gateway/`](api-gateway/) | [`api-gateway`](https://github.com/CampusSeguro-DOSW-2026-2/api-gateway) |
| MS01 — Identidad y acceso | [`ms01-identity-and-access/`](ms01-identity-and-access/) | [`ms01-identity-and-access`](https://github.com/CampusSeguro-DOSW-2026-2/ms01-identity-and-access) |
| MS02 — Campus y seguridad | [`ms02-campus-and-security/`](ms02-campus-and-security/) | [`ms02-campus-and-security`](https://github.com/CampusSeguro-DOSW-2026-2/ms02-campus-and-security) |
| MS03 — Alertas y emergencias | [`ms03-alerts-and-emergencies/`](ms03-alerts-and-emergencies/) | [`ms03-alerts-and-emergencies`](https://github.com/CampusSeguro-DOSW-2026-2/ms03-alerts-and-emergencies) |
| MS04 — Evacuación y check-in | [`ms04-evacuation-and-check-in/`](ms04-evacuation-and-check-in/) | [`ms04-evacuation-and-check-in`](https://github.com/CampusSeguro-DOSW-2026-2/ms04-evacuation-and-check-in) |
| MS05 — Brigadistas e incidentes | [`ms05-brigades-and-incidents/`](ms05-brigades-and-incidents/) | [`ms05-brigades-and-incidents`](https://github.com/CampusSeguro-DOSW-2026-2/ms05-brigades-and-incidents) |
| MS06 — Eventos | [`ms06-drills/`](ms06-drills/) | [`ms06-drills`](https://github.com/CampusSeguro-DOSW-2026-2/ms06-drills) |
| MS07 — Formación y asistente de IA | [`ms07-training-and-ai-assistant/`](ms07-training-and-ai-assistant/) | [`ms07-training-and-ai-assistant`](https://github.com/CampusSeguro-DOSW-2026-2/ms07-training-and-ai-assistant) |
| MS08 — Analítica y auditoría | [`ms08-analytics-and-audit/`](ms08-analytics-and-audit/) | [`ms08-analytics-and-audit`](https://github.com/CampusSeguro-DOSW-2026-2/ms08-analytics-and-audit) |
| Frontend | — | [`frontend`](https://github.com/CampusSeguro-DOSW-2026-2/frontend) |

Cada carpeta lleva el nombre del repositorio del microservicio y la estructura que describe [`CONTRIBUTING.md`](CONTRIBUTING.md).

## Equipos

| Microservicios | Nombre | Usuario de GitHub |
|---|---|---|
| API Gateway, MS01, MS04 | Daniel Santiago Morales Perdomo | [`Fukuyaku88`](https://github.com/Fukuyaku88) |
| API Gateway, MS01, MS04 | Edgar Daniel Ruiz Patiño | [`DaniRuiz11`](https://github.com/DaniRuiz11) |
| API Gateway, MS01, MS04 | Miguel Ángel Acero Laverde | [`Miguelim101`](https://github.com/Miguelim101) |
| API Gateway, MS01, MS04 | Thomas Sebastian García Gomez | [`thomassebastiangarciagomez-ctrl`](https://github.com/thomassebastiangarciagomez-ctrl) |
| API Gateway, MS04 | Rodrigo Gualtero | [`Rodrigoreg92`](https://github.com/Rodrigoreg92) |
| MS02, MS07 | | [`nataliaRT12`](https://github.com/nataliaRT12) |
| MS02, MS07 | | [`Marix201`](https://github.com/Marix201) |
| MS02, MS07 | | [`AndreaSolano23`](https://github.com/AndreaSolano23) |
| MS03, MS05, MS06, MS08 (squad Zephyros) | Rodrigo Humberto Gualtero Martínez | [`ginnko2019`](https://github.com/ginnko2019) |
| MS03, MS05, MS06, MS08 (squad Zephyros) | Tomás Olaya Díaz | [`iAxstral`](https://github.com/iAxstral) |
| MS03, MS05, MS06, MS08 (squad Zephyros) | José Daniel García Pineda | [`KenjiMaster`](https://github.com/KenjiMaster) |
| MS03, MS05, MS06, MS08 (squad Zephyros) | Jerónimo Moreno Herrera | [`Dracodec113`](https://github.com/Dracodec113) |
| MS03, MS05, MS06, MS08 (squad Zephyros) | Paula Alejandra Novoa Castellanos | [`Aleja15-31`](https://github.com/Aleja15-31) |
| MS03, MS05, MS06, MS08 (squad Zephyros) | Derly Valeria Pachón Pinzón | [`itsValepP`](https://github.com/itsValepP) |

## Cómo se cambia la documentación

Todo cambio se hace mediante Pull Request y lo revisa el squad dueño de lo que cambia. Las convenciones de nombres, de estructura y de contenido están en [`CONTRIBUTING.md`](CONTRIBUTING.md).
