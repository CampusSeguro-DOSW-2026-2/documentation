# Documentation — CampusSeguro

Repositorio central de la documentación de cada microservicio: requerimientos, contrato OpenAPI y diagramas.

## Dónde está cada cosa

| Qué | Dónde | Quién lo mantiene |
|---|---|---|
| Reglas transversales (puertos, JWT, cabeceras, errores, correlación, paginación, auditoría) | Wiki: [Estándares de Integración](https://github.com/CampusSeguro-DOSW-2026-2/documentation/wiki/Est%C3%A1ndares-de-Integraci%C3%B3n-y-Contratos-Transversales) | Todos los squads, por PR acordado |
| Roles y permisos | Wiki: [Matriz Rol‐Permiso](https://github.com/CampusSeguro-DOSW-2026-2/documentation/wiki/Matriz-Rol%E2%80%90Permiso) | Todos los squads, por PR acordado |
| Guía de integración de cada MS para los demás squads | Wiki: una página por MS (p. ej. [MS01](https://github.com/CampusSeguro-DOSW-2026-2/documentation/wiki/MS01%E2%80%90identity%E2%80%90and%E2%80%90access)) | Squad dueño |
| Requerimientos, contrato y diagramas de cada MS | Este repositorio, una carpeta por MS | Squad dueño |

## Estructura por microservicio

Cada carpeta usa el nombre del repositorio del MS y la misma estructura:

```text
<msNN-nombre>/
├── README.md              índice de la carpeta
├── 1_requirements.md      RF, RN, RNF y tabla de endpoints
├── 2_forms-inputs.md      formularios y validaciones de entrada (si aplica)
├── ...                    especificaciones propias numeradas (3_, 4_, …)
├── api/swagger.yml        contrato OpenAPI 3 (fuente que usa el Gateway para enrutar)
├── diagramas/             imágenes y modelos
└── anexos/                material no normativo (bitácoras, investigación)
```

## Microservicios

| MS | Carpeta | Estado |
|---|---|---|
| MS01 — Identidad y acceso | [`ms01-identity-and-access/`](ms01-identity-and-access/) | Publicado |
| MS02–MS08 | — | Pendiente de cada squad |
| GW — API Gateway | [`api-gateway/`](api-gateway/) | Publicado |

## Reglas

* Un dato tiene **un solo lugar**: los documentos de un MS enlazan a la wiki en lugar de copiar valores transversales.
* El `x-permiso`/`x-roles` de cada `swagger.yml` debe coincidir con la matriz.
* El Gateway solo enruta rutas publicadas en un `swagger.yml` de este repositorio.
* Todo cambio entra por Pull Request con revisión.
