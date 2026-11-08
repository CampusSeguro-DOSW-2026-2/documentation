# MS01 — Identidad y acceso

Usuarios, autenticación, roles, permisos y sesiones. Código: repositorio [`ms01-identity-and-access`](https://github.com/CampusSeguro-DOSW-2026-2/ms01-identity-and-access).

Cada tema se documenta en un solo archivo; los demás enlazan a él.

| Documento | Única fuente de |
|---|---|
| [`1_requirements.md`](1_requirements.md) | Comportamiento: RF-MS01-01…09, reglas de negocio, RNF y tabla de endpoints con permisos. |
| [`2_forms-inputs.md`](2_forms-inputs.md) | Formularios F-01…F-09: campos, validaciones y mensajes. |
| [`3_jwt-spec.md`](3_jwt-spec.md) | Detalle técnico de la sesión: token, cabeceras de identidad, ciclo de vida y llaves. |
| [`4_modelo-datos.md`](4_modelo-datos.md) | Tablas, almacenamiento seguro, operaciones en BD, configuración y datos iniciales. |
| [`api/swagger.yml`](api/swagger.yml) | Contrato OpenAPI 3. |

### Diagramas

Cómo verlos y editarlos: [`diagramas/README.md`](diagramas/README.md).

| Vista | Diagrama |
|---|---|
| C4 · Componentes | [Controllers, Services y Repositories](diagramas/ms01_c4_3_componentes.drawio.svg) |
| Clases | [Dominio](diagramas/ms01_clases.svg) ([`.puml`](diagramas/ms01_clases.puml)) |
| Despliegue | [Azure (propuesta, base para mejoras)](diagramas/ms01_despliegue_azure.drawio.svg) |
| Modelo de datos físico | [Tablas de PostgreSQL](diagramas/ms01_er_fisico.drawio.svg) |

**Para otros squads:** la guía de integración está en la [wiki de MS01](https://github.com/CampusSeguro-DOSW-2026-2/documentation/wiki/MS01%E2%80%90identity%E2%80%90and%E2%80%90access). Las reglas transversales están en los [Estándares](https://github.com/CampusSeguro-DOSW-2026-2/documentation/wiki/Est%C3%A1ndares-de-Integraci%C3%B3n-y-Contratos-Transversales) y los permisos en la [Matriz Rol‐Permiso](https://github.com/CampusSeguro-DOSW-2026-2/documentation/wiki/Matriz-Rol%E2%80%90Permiso).
