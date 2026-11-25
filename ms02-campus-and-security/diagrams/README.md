# Diagramas de MS02

| Diagrama | Archivo | Responde a |
|---|---|---|
| C4 Nivel 3 · Componentes (1/3) | [`ms02-c4-3-components-1-campus.drawio.svg`](ms02-c4-3-components-1-campus.drawio.svg) | Infraestructura del campus: controladores, servicios y repositorios de campus, edificios, pisos, espacios, escaleras, salidas, zonas abiertas y zonas seguras. |
| C4 Nivel 3 · Componentes (2/3) | [`ms02-c4-3-components-2-public-lookup.drawio.svg`](ms02-c4-3-components-2-public-lookup.drawio.svg) | Imágenes, códigos QR, consulta pública sin sesión y recomendaciones preventivas, con el almacenamiento de fotografías. |
| C4 Nivel 3 · Componentes (3/3) | [`ms02-c4-3-components-3-evacuation-routes.drawio.svg`](ms02-c4-3-components-3-evacuation-routes.drawio.svg) | Rutas de evacuación: servicios, puertos de `ruta/port` y sus implementaciones, y el proveedor de navegación (OSRM). |
| Modelo de datos físico | [`ms02-er-physical.drawio.svg`](ms02-er-physical.drawio.svg) | Tablas, tipos, claves, `UNIQUE` y `CHECK` del esquema que dejan las migraciones V1 a V17 ([`3_data-model.md`](../3_data-model.md)). |
| Clases (UML) | [`ms02-classes.puml`](ms02-classes.puml) · [`ms02-classes.svg`](ms02-classes.svg) | Entidades, records, enums y el puerto de navegación del dominio, y cómo se referencian entre sí. |
| Despliegue | [`ms02-deployment-azure.drawio.svg`](ms02-deployment-azure.drawio.svg) | Entrega desde GitHub y VM de MS02 en Azure (Mexico Central) con NSG, Caddy de borde con mTLS, runner self-hosted, Azure Database for PostgreSQL y Blob Storage para las fotografías. |

El componente se divide en tres vistas porque sus 15 controladores, 17 servicios y 11 repositorios no caben legibles en un solo lienzo.

## Cómo ver y editar los diagramas

### draw.io (`.drawio.svg`)

1. En [draw.io](https://app.diagrams.net/) (web o escritorio): **File › Open** y elegir el `.drawio.svg`.
2. Ver y editar el diagrama.
3. Exportar: **File › Export as › SVG**. Para que el archivo siga siendo editable, marcar **Include a copy of my diagram** y guardarlo con la extensión `.drawio.svg`.

### PlantUML (`.puml`)

1. Abrir un editor de PlantUML, por ejemplo [plantuml.com/plantuml](https://www.plantuml.com/plantuml/uml/) o la extensión *PlantUML* de VS Code.
2. Pegar el contenido del `.puml`; la vista previa se actualiza sola.
3. Descargar en **SVG** con el mismo nombre (`ms02-classes.svg`) y guardarlo en esta carpeta junto al `.puml`.

El `.puml` es la fuente: si cambia el modelo, se edita el `.puml` y se vuelve a exportar el SVG.
