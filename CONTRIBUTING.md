# Cómo se escribe y se cambia la documentación

Este documento reúne las convenciones de este repositorio y de su [wiki](https://github.com/CampusSeguro-DOSW-2026-2/documentation/wiki). Aplican a todos los squads.

## Estructura del repositorio

```text
README.md                 punto de entrada
CONTRIBUTING.md           este documento
integration/              lo que comparten todos los squads
├── standards.md          reglas transversales
├── permissions-matrix.md roles y permisos
├── gateway-connection.md conexión de un microservicio al API Gateway
├── audit-catalog.md      acciones de auditoría de cada microservicio
└── architecture/         diagramas C4 de contexto y de contenedores, e infraestructura global de despliegue
api-gateway/              documentación del API Gateway
ms01-identity-and-access/ una carpeta por microservicio, con el nombre de su repositorio
...
ms08-analytics-and-audit/
```

La carpeta de un microservicio contiene solo lo que ese microservicio necesita:

```text
<msNN-nombre>/
├── README.md             índice de la documentación del microservicio
├── 1_requirements.md     requerimientos funcionales, reglas de negocio y requerimientos no funcionales
├── 2_forms-inputs.md     formularios y validaciones de entrada (si tiene formularios)
├── 3_data-model.md       modelo de datos (si tiene base de datos)
├── quality-and-tests.md  calidad y pruebas
├── security.md           seguridad
├── api/
│   └── openapi.yml       contrato OpenAPI 3
├── diagrams/             diagramas y sus fuentes editables
└── integration-guide.md  guía para los demás squads (solo si el microservicio ofrece una)
```

`quality-and-tests.md` y `security.md` son obligatorios en todo microservicio y en el API Gateway:

* **`quality-and-tests.md`:** estrategia de pruebas (qué se prueba en cada nivel y cómo), tecnologías de prueba y por qué se usan, resultado de la ejecución de Maven Surefire (pruebas, fallos, errores y omitidas), cobertura de JaCoCo (total y por paquete: instrucciones, ramas, líneas y métodos), análisis estático con PMD y SpotBugs (hallazgos y cómo se corrigieron, y cada exclusión con su motivo), cómo ejecutar las pruebas en local y su relación con SonarCloud. Las cifras indican la rama y el commit de los que salen.
* **`security.md`:** cómo el microservicio recibe la identidad (cabeceras que inyecta el Gateway) y autoriza cada operación, sus rutas internas y la clave de servicio con la que llama a otros microservicios, CORS y mTLS, el manejo de los datos sensibles (qué guarda, qué registra en logs y qué nunca expone) y las demás medidas de seguridad. El registro, la autenticación y la formación del JWT se documentan en el `security.md` de MS01, y la validación del token en el del API Gateway; los demás enlazan a ellos.

Un microservicio que todavía no tiene documentación conserva su carpeta con un archivo `.gitkeep`. Un documento propio de un microservicio, como `jwt-spec.md` en MS01, se agrega con un nombre que lo describa.

## Un dato, un solo lugar

Cada dato se escribe en un único documento y los demás enlazan a él. Así se evita que dos documentos digan cosas distintas.

| Dato | Único lugar |
|---|---|
| Puertos, JWT, cabeceras, llamadas internas, errores, correlación, paginación y auditoría | `integration/standards.md` |
| Códigos de permiso y su asignación inicial a los roles | `integration/permissions-matrix.md` |
| Acciones de auditoría de cada microservicio y sus atributos | `integration/audit-catalog.md` |
| Cómo conectar un microservicio al API Gateway | `integration/gateway-connection.md` |
| Endpoints, métodos, parámetros, respuestas y el permiso de cada operación | `api/openapi.yml` del microservicio |
| Comportamiento: requerimientos, reglas de negocio y flujos | `1_requirements.md` del microservicio |
| Tablas, columnas y restricciones, y por qué se eligió ese tipo de base de datos | `3_data-model.md` del microservicio |
| Pruebas, cobertura y análisis estático | `quality-and-tests.md` del microservicio |
| Medidas de seguridad | `security.md` del microservicio |

El `x-permiso` de cada operación del `openapi.yml` coincide con un código de la matriz de permisos; cuando una operación acepta más de uno, se escriben separados por `|`. El `x-roles` es informativo y no autoriza nada. El Gateway solo enruta las rutas publicadas en los `openapi.yml`.

## Nombres de archivos y carpetas

* En inglés, solo caracteres ASCII, en minúsculas y con guiones: `data-model`, `gateway-connection`. Sin espacios, tildes ni mayúsculas.
* La numeración se usa únicamente en `1_requirements.md`, `2_forms-inputs.md` y `3_data-model.md`. `quality-and-tests.md` y `security.md` no llevan número.
* Las carpetas de los microservicios se llaman como su repositorio (`ms05-brigades-and-incidents`).
* El contrato es siempre `api/openapi.yml`.
* Los diagramas van en `diagrams/` y se llaman `<prefijo>-<vista>.<extensión>`, donde el prefijo es `ms<NN>` o `gw`: `ms04-c4-3-components.drawio.svg`, `ms01-classes.puml`, `ms03-er.drawio.svg`. Se conserva siempre la fuente editable: un `.drawio.svg` se abre y se edita con [draw.io](https://app.diagrams.net/), y un `.svg` generado con PlantUML va junto a su `.puml`.
* Las páginas de la wiki siguen la misma regla de nombres, con la primera letra en mayúscula y guiones entre palabras (`Frontend-cloud-deployment`).

Excepciones:

* Los diagramas de arquitectura de `integration/architecture/` tienen una carpeta con un `README.md` que describe qué debe mostrar el diagrama. Ese `README.md` se elimina cuando se agrega el diagrama.
* El diagrama de infraestructura global de despliegue conserva su nombre original: `integration/architecture/architecture_diagram/architecture_diagram.drawio.svg`.

## Qué se escribe y qué no

* **Español** en el contenido; los nombres de archivos, de rutas y de campos técnicos, como los define el código.
* **El estado real.** Cada documento describe cómo funciona el sistema hoy. Cuando el código cambia, se cambia el documento en el mismo Pull Request.
* **Sin encabezados de versión, estado o responsable, y sin historial de cambios:** Git guarda quién cambió qué y cuándo. Tampoco se agregan secciones de pendientes, de decisiones abiertas ni de decisiones de diseño por separado; la decisión se escribe en el documento donde aplica, como un hecho.
* **Sin datos sensibles.** No se escriben contraseñas, claves, tokens ni direcciones internas de infraestructura. Las variables y los secretos se nombran, nunca se copian sus valores.
* **Ejemplos:** usar siempre datos ficticios.

## La wiki

La wiki contiene guías paso a paso para hacer una tarea: agregar un microservicio al gateway y desplegar el frontend. Las reglas, los contratos y la documentación de cada microservicio viven en este repositorio, no en la wiki. La `Home` de la wiki enlaza a las guías y al `README.md` de este repositorio.

## Cómo se cambia algo

1. El cambio se hace en una rama y se abre un Pull Request.
2. Lo revisa el squad dueño del documento. Los cambios en `integration/` afectan a todos los squads: se acuerdan con los squads afectados antes de mezclarlos.
3. Antes de pedir la revisión se comprueba que los enlaces funcionan y que el `x-permiso` de cada operación nueva o modificada está en la matriz de permisos.
