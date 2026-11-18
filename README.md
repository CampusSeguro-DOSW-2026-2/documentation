# Documentation — CampusSeguro

Repositorio central de la documentación de cada microservicio: requerimientos, contratos OpenAPI y diagramas.

## Dónde está cada cosa

| **Qué** | **Dónde** | **Quién lo mantiene** |
|---|---|---|
| Reglas transversales (puertos, JWT, cabeceras, errores, correlación, paginación y auditoría) | Wiki: [Estándares de Integración](https://github.com/CampusSeguro-DOSW-2026-2/documentation/wiki/Est%C3%A1ndares-de-Integraci%C3%B3n-y-Contratos-Transversales) | Todos los squads, por PR acordado |
| Roles y permisos | Wiki: [Matriz Rol-Permiso](https://github.com/CampusSeguro-DOSW-2026-2/documentation/wiki/Matriz-Rol%E2%80%90Permiso) | Todos los squads, por PR acordado |
| Guía de integración de cada MS para los demás squads | Wiki: una página por MS (por ejemplo, [MS01](https://github.com/CampusSeguro-DOSW-2026-2/documentation/wiki/MS01%E2%80%90identity%E2%80%90and%E2%80%90access)) | Squad dueño |
| Requerimientos, contrato y diagramas de cada MS | Este repositorio, una carpeta por microservicio | Squad dueño |

## Estructura por microservicio

Cada carpeta utiliza el nombre del repositorio del microservicio y mantiene una estructura organizada para facilitar la consulta y actualización de su documentación.

```text
<msNN-nombre>/
├── README.md              índice de la documentación del MS
├── 1_requirements.md      RF, RN, RNF y tabla de endpoints
├── 2_forms-inputs.md      formularios y validaciones de entrada (si aplica)
├── ...                    especificaciones propias numeradas (3_, 4_, ...)
├── api/
│   └── swagger.yml        contrato OpenAPI 3
├── diagramas/             imágenes y modelos editables
└── anexos/                material complementario
```

## Microservicios

| **MS** | **Carpeta** | **Estado** |
|---|---|---|
| MS01 — Identidad y acceso | [`ms01-identity-and-access/`](ms01-identity-and-access/) | Publicado |
| MS02 — Campus y Seguridad | — | Pendiente de documentación |
| MS03 — Alertas y Emergencias | [`ms03-alerts-and-emergencies/`](ms03-alerts-and-emergencies/) | Publicado |
| MS04 — Evacuación y check-in | [`ms04-evacuation-check-in/`](ms04-evacuation-check-in/) | Publicado |
| MS05 — Brigadas e Incidentes | — | Pendiente de documentación |
| MS06 — Eventos | — | Pendiente de documentación |
| MS07 — Formación e IA | — | Pendiente de documentación |
| MS08 — Analítica y Auditoría | — | Pendiente de documentación |
| GW — API Gateway | [`api-gateway/`](api-gateway/) | Publicado |

## Reglas

- Un dato tiene **un solo lugar**: los documentos de cada microservicio enlazan a la Wiki en lugar de duplicar información transversal.
- El `x-permiso` y `x-roles` de cada `swagger.yml` deben coincidir con la matriz general de roles y permisos.
- El Gateway solo enruta las rutas publicadas en los contratos OpenAPI correspondientes.
- Cada squad es responsable de mantener actualizada la documentación de su microservicio.
- Todo cambio en la documentación debe realizarse mediante Pull Request y revisión del equipo responsable.

---

**CampusSeguro — Repositorio central de documentación**