# Diagramas de MS02

| Diagrama | Archivo | Responde a |
|---|---|---|
| C4 Nivel 3 · Componentes | [`ms02-c4-3-components.puml`](ms02-c4-3-components.puml) | Componentes de MS02: entrada por el API Gateway, filtros de correlación y seguridad, controladores, servicios, módulo de rutas con sus puertos, proveedor de navegación (OSRM), QR, almacenamiento de imágenes y PostgreSQL. |
| Modelo de datos físico | [`ms02-er-physical.puml`](ms02-er-physical.puml) | Tablas, tipos, claves, `UNIQUE` y `CHECK` del esquema que dejan las migraciones V1 a V17 ([`3_data-model.md`](../3_data-model.md)). |
| Clases (UML) | [`ms02-classes.puml`](ms02-classes.puml) | Entidades y enums del dominio y sus relaciones. |
| Despliegue | [`ms02-deployment-azure.puml`](ms02-deployment-azure.puml) | VM de MS02 en Azure con NSG, Caddy de borde con mTLS, stacks de prueba y producción, runner self-hosted e imagen en GHCR. |

## Cómo ver y editar los diagramas

1. Abrir un editor de PlantUML, por ejemplo [plantuml.com/plantuml](https://www.plantuml.com/plantuml/uml/) o la extensión *PlantUML* de VS Code.
2. Pegar el contenido del `.puml`; la vista previa se actualiza sola.
3. Para publicar la imagen, descargarla en **SVG** con el mismo nombre (p. ej. `ms02-classes.svg`) y guardarla en esta carpeta junto al `.puml`.

El `.puml` es la fuente: si cambia el modelo, se edita el `.puml` y se vuelve a exportar el SVG.
