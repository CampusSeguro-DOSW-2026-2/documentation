# MS02 — Campus y seguridad

Infraestructura del campus (campus, zonas abiertas, edificios, pisos, espacios, salidas y escaleras), zonas seguras y puntos de encuentro, rutas de evacuación (dentro del edificio por escaleras y fuera de él con OpenStreetMap), información preventiva, imágenes y códigos QR con consulta pública sin sesión. Código: repositorio [`ms02-campus-and-security`](https://github.com/CampusSeguro-DOSW-2026-2/ms02-campus-and-security).

Cada tema se documenta en un solo archivo; los demás enlazan a él.

| Documento | Única fuente de |
|---|---|
| [`1_requirements.md`](1_requirements.md) | Comportamiento: RF-MS02-01…16, reglas de negocio, RNF, dependencias y tabla de endpoints con permisos. |
| [`2_forms-inputs.md`](2_forms-inputs.md) | Formularios y entradas F-01…F-13: campos, validaciones y mensajes. |
| [`3_modelo-datos.md`](3_modelo-datos.md) | Tablas, relaciones, reglas de integridad y migraciones. |
| [`api/swagger.yml`](api/swagger.yml) | Contrato OpenAPI 3, exportado de `/v3/api-docs.yaml` del servicio. |

### Diagramas

Cómo verlos y editarlos: [`diagramas/README.md`](diagramas/README.md).

| Vista | Diagrama |
|---|---|
| C4 · Componentes | [Filtros, controladores, servicios, rutas y navegación](diagramas/ms02_c4_3_componentes.puml) |
| Clases | [Dominio](diagramas/ms02_clases.puml) |
| Despliegue | [VM en Azure con firewall y mTLS, entornos de prueba y producción](diagramas/ms02_despliegue_azure.puml) |
| Modelo de datos físico | [Tablas de PostgreSQL](diagramas/ms02_er_fisico.puml) |

**Para otros squads:** la guía de integración está en la [wiki de MS02](https://github.com/CampusSeguro-DOSW-2026-2/documentation/wiki/MS02%E2%80%90campus%E2%80%90and%E2%80%90security). Las reglas transversales están en los [Estándares](https://github.com/CampusSeguro-DOSW-2026-2/documentation/wiki/Est%C3%A1ndares-de-Integraci%C3%B3n-y-Contratos-Transversales) y la asignación inicial de permisos a los roles en la [Matriz Rol‐Permiso](https://github.com/CampusSeguro-DOSW-2026-2/documentation/wiki/Matriz-Rol%E2%80%90Permiso).
