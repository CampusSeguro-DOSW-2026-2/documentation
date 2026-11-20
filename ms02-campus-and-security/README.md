# MS02 — Campus y seguridad

Infraestructura del campus (campus, zonas abiertas, edificios, pisos, espacios, salidas y escaleras), zonas seguras y puntos de encuentro, rutas de evacuación (dentro del edificio por escaleras y fuera de él con OpenStreetMap), información preventiva, imágenes y códigos QR con consulta pública sin sesión. Código: repositorio [`ms02-campus-and-security`](https://github.com/CampusSeguro-DOSW-2026-2/ms02-campus-and-security).

Cada tema se documenta en un solo archivo; los demás enlazan a él.

| Documento | Única fuente de |
|---|---|
| [`1_requirements.md`](1_requirements.md) | Comportamiento: RF-MS02-01…16, reglas de negocio y RNF. |
| [`2_forms-inputs.md`](2_forms-inputs.md) | Formularios y entradas F-01…F-13: campos, validaciones y mensajes. |
| [`3_data-model.md`](3_data-model.md) | Tablas, relaciones, reglas de integridad y migraciones. |
| [`api/openapi.yml`](api/openapi.yml) | Contrato OpenAPI 3, exportado de `/v3/api-docs.yaml` del servicio: endpoints, permisos (`x-permiso`), esquemas y errores. |

### Diagramas

Cómo verlos y editarlos: [`diagrams/README.md`](diagrams/README.md).

| Vista | Diagrama |
|---|---|
| C4 · Componentes | [Filtros, controladores, servicios, rutas y navegación](diagrams/ms02-c4-3-components.puml) |
| Clases | [Dominio](diagrams/ms02-classes.puml) |
| Despliegue | [VM en Azure con firewall y mTLS, entornos de prueba y producción](diagrams/ms02-deployment-azure.puml) |
| Modelo de datos físico | [Tablas de PostgreSQL](diagrams/ms02-er-physical.puml) |

**Para otros squads:** las reglas transversales están en los [Estándares](../integration/standards.md) y la asignación inicial de permisos a los roles en la [matriz de roles y permisos](../integration/permissions-matrix.md).
