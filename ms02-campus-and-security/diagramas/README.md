# Diagramas de MS02

| Diagrama | Archivo | Responde a |
|---|---|---|
| C4 Nivel 3 · Componentes | [`ms02_c4_3_componentes.puml`](ms02_c4_3_componentes.puml) | Componentes de MS02: entrada por el API Gateway, filtros de correlación y seguridad, controladores, servicios, módulo de rutas con sus puertos, proveedor de navegación (OSRM), QR, almacenamiento de imágenes (disco o Azure Blob Storage) y PostgreSQL. |
| Modelo de datos físico | [`ms02_er_fisico.puml`](ms02_er_fisico.puml) | Tablas, tipos, claves, `UNIQUE` y `CHECK` del esquema que dejan las migraciones V1 a V17 ([`3_modelo-datos.md`](../3_modelo-datos.md)). |
| Clases (UML) | [`ms02_clases.puml`](ms02_clases.puml) | Entidades y enums del dominio y sus relaciones. |
| Despliegue | [`ms02_despliegue_azure.puml`](ms02_despliegue_azure.puml) | VM de MS02 en Azure con NSG, Caddy de borde con mTLS, stacks de prueba y producción, Azure Blob Storage para las fotografías, runner self-hosted e imagen en GHCR. |

## Cómo ver y editar los diagramas

1. Abrir un editor de PlantUML, por ejemplo [plantuml.com/plantuml](https://www.plantuml.com/plantuml/uml/) o la extensión *PlantUML* de VS Code.
2. Pegar el contenido del `.puml`; la vista previa se actualiza sola.
3. Para publicar la imagen, descargarla en **SVG** con el mismo nombre (p. ej. `ms02_clases.svg`) y guardarla en esta carpeta junto al `.puml`.

El `.puml` es la fuente: si cambia el modelo, se edita el `.puml` y se vuelve a exportar el SVG.
