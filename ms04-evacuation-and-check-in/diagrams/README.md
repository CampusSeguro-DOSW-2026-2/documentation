# Diagramas de MS04

| Diagrama | Archivo | Responde a |
|---|---|---|
| C4 Nivel 3 · Componentes | [`ms04-c4-3-components.drawio.svg`](ms04-c4-3-components.drawio.svg) | Controllers, services, repositories y clientes de MS04 en una vista simplificada, con una línea de color por interfaz: de dónde llegan las peticiones (API Gateway y las rutas internas de MS06) y a qué se conectan (PostgreSQL y, por el Gateway, MS01, MS02, MS05 y MS08). |
| Modelo de datos físico | [`ms04-er-physical.drawio.svg`](ms04-er-physical.drawio.svg) | Tablas, tipos, claves, `CHECK`, índices, la vista `v_checkins_vigentes` y el trigger de solo inserción (scripts 001 a 006 de `database/`). |
| Clases (UML) | [`ms04-classes.puml`](ms04-classes.puml) · [SVG](ms04-classes.svg) | Entidades y enums del dominio, identidad y permisos del solicitante, y auditoría con su contexto de evento. |
| Despliegue | [`ms04-deployment-aws.drawio.svg`](ms04-deployment-aws.drawio.svg) | EC2 de MS04 en AWS con Security Group, Caddy de borde con mTLS y stacks prod/test, y otra EC2 sin IP pública con una PostgreSQL por entorno; el API Gateway sigue en Azure. |

## Cómo ver y editar los diagramas

### PlantUML (`.puml`)

1. Abrir el [editor de PlantUML](https://www.vpascode.com/es/editor/#plantuml:dLPDR-Cs4BthLn2vLDvkZExL288rBBKhrEiOZWFqLj3Is2P2aIf99LMx_UyTVfYIRBKeUh798Vduviq7zLaRganRyuIGn333WNoH1ZZvIWKzGGt2aETE6wYJv3kImP4984mOK4TQWaxYa1GxqA3Uw04X_aGqGhudDNs5O3PCYhHijP4rg0LPXT57Kb43Ua4o_DjHSOBPWXoav41502cfA84FOLG_Bge5UJQAYTFjkJEKhyitoZWzCCvCbzgh_4s3pNybGoUsj1jvq2Y9RceqMobIMijMc0MfP8kxRgvaMhVeyB4LrNyaUG8Ji9yDDQrEPvUjbbXKd1wqKRGqfEHKwsGy9LjGMeh0-PwmU5qm25h3bGbgrENArfob6Ezp_8fgG2fQDuWvPLz9OP1P8OxoPhbjR0I6EnZ-3BHvawmQ-LrAKIemu1t5VFCENx8dkinbHxdfjX93YSc4ls71I4CvIjkDh8s28oZ0l4Bpccdpu2ay1g90bLWF8LCiasubzJ3qZgpVhw02gCSdcfxBLDJ3vZq3TQCJhQfjP9ZsP2UKydhifAoZ5WedcxO-s38RnTSkxBi6ONPX51PAsj0I2sGi4Ii14sl3JkVeTLZ3q0jcsAHuc7uw51s7Qxznzchl_d-DG05ED3Y0iVwsYYpV2TpPYBlJTkdeTwbtexjMDKBQYwQgCcZUrtSl_LJDNqIsxQWGMUn1G_jJQsuMNEewjwxaruxJOiZ0qxoiDr0oYXtpjnQqJS5HSfMKbot7Q_OHFxqM2Slko5GzxBbZTO8pKrPq9Rq9dQzywvnE1Rol_77lxaToiU3Hc_OSev6rAZAnQUhQH2vQZB6p9u3ZcDaX9C_xvIPRxZ8toVBroo-vjxyKUp_QxlBd8iit-sAv_YVCQvq3-3RVPCNc9pTUFMr-B7PVSt_Jwak--ZdFVYqse-dJozxFbvjLlbxZPdVX3hW3rsVMQ9mNyJbYTSFT6-pJ31ymVOQASDiQtvav4mmibWgsesJu_3vycyz7-RdeaNGItxX_tuP4bfTdXeapA6RiGqnIBJcRvX3UeOTlz_TzYruG-Ds0g1pYIjQDrCnxuu4gXe-gg6QOkAwDCKqQ_2GXyZZabWnvVa3ykq_p-OSxVsHX4zPG9ZHvTBorbbW1uHRd4-bPpppjkkM6DPpPnfT4jdUVxeX5_y6ZZyInNpFXXWmQT0bxSXU3GfMIxzQBY5MvJPhWuTXmu-9y_jWdwuASgRuyOWGxgo87x7ILFNhZaMFl6nt1BWjyf8rtGBh_GGnivqMyG-VnTa0v-yEVGA7x3o3SSFLnaSh6RgFSjauMXgYtWD98DPLuFYJ3LFgCEk0tvjy0) (abre con un diagrama de ejemplo).
2. Borrar el ejemplo y pegar el contenido del `.puml` (p. ej. [`ms04-classes.puml`](ms04-classes.puml)); la vista previa se actualiza sola.
3. Descargar en **SVG** desde el editor.

### draw.io (`.drawio.svg`)

1. En [draw.io](https://app.diagrams.net/) (web o escritorio): **File › Open** y elegir el `.drawio.svg` (p. ej. [`ms04-c4-3-components.drawio.svg`](ms04-c4-3-components.drawio.svg)).
2. Ver y editar el diagrama con las figuras UML de draw.io.
3. Exportar: **File › Export as › SVG**. Para que el archivo siga siendo editable, marcar **Include a copy of my diagram** y guardarlo con la extensión `.drawio.svg`.
