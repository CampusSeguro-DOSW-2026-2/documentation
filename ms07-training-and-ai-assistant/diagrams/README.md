# Diagramas de MS07

| Diagrama | Archivo | Responde a |
|---|---|---|
| C4 Nivel 3 · Componentes | [`ms07-c4-3-components.drawio.svg`](ms07-c4-3-components.drawio.svg) | Componentes de MS07: un controlador, un servicio y un repositorio por caso de uso (evaluaciones, categorías, contenidos, visualizaciones, progreso y tutor), la auditoría transversal y las salidas a PostgreSQL, al proveedor del modelo de lenguaje y al API Gateway. |
| Modelo de datos físico | [`ms07-er-physical.drawio.svg`](ms07-er-physical.drawio.svg) | Tablas, tipos, claves, `UNIQUE` y `CHECK` del esquema que dejan las migraciones V1 a V9 ([`3_data-model.md`](../3_data-model.md)). |
| Clases (UML) | [`ms07-classes.puml`](ms07-classes.puml) · [`ms07-classes.svg`](ms07-classes.svg) | Entidades del dominio, enums y la jerarquía de adaptadores del tutor. |
| Despliegue | [`ms07-deployment-aws.drawio.svg`](ms07-deployment-aws.drawio.svg) | Entrega desde GitHub y la instancia EC2 de MS07 en AWS (us-east-1) con security group, Caddy de borde con mTLS, runner self-hosted y stacks de prueba y producción con su PostgreSQL. |

## Cómo ver y editar los diagramas

### draw.io (`.drawio.svg`)

1. En [draw.io](https://app.diagrams.net/) (web o escritorio): **File › Open** y elegir el `.drawio.svg`.
2. Ver y editar el diagrama.
3. Exportar: **File › Export as › SVG**. Para que el archivo siga siendo editable, marcar **Include a copy of my diagram** y guardarlo con la extensión `.drawio.svg`.

### PlantUML (`.puml`)

1. Abrir un editor de PlantUML, por ejemplo [plantuml.com/plantuml](https://www.plantuml.com/plantuml/uml/) o la extensión *PlantUML* de VS Code.
2. Pegar el contenido del `.puml`; la vista previa se actualiza sola.
3. Descargar en **SVG** con el mismo nombre (`ms07-classes.svg`) y guardarlo en esta carpeta junto al `.puml`.

El `.puml` es la fuente: si cambia el modelo, se edita el `.puml` y se vuelve a exportar el SVG.
