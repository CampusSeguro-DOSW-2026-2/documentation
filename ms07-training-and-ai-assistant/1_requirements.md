# Requerimientos de MS07 – Formación y asistente IA

Los campos de entrada y sus mensajes están en [2_forms-inputs.md](2_forms-inputs.md); los endpoints, en [api/openapi.yml](api/openapi.yml); y las pruebas, en [quality-and-tests.md](quality-and-tests.md).

## 1. Descripción

El microservicio MS07 ofrece funcionalidades de formación preventiva para la comunidad de CampusSeguro. Permite consultar contenidos institucionales, realizar evaluaciones cortas, registrar el progreso de los usuarios y consultar al Tutor CampusSeguro.

El servicio administra su propia información y expone sus funcionalidades mediante una API REST.

## 2. Requerimientos funcionales

### RF-08 – Gestionar contenidos de capacitación

El sistema permite administrar contenidos de capacitación preventiva organizados por categorías.

Los contenidos incluyen un título, un cuerpo, una categoría y un estado. Se pueden crear, actualizar, desactivar y reactivar.

### RF-09 – Consultar contenidos por categoría

El usuario puede consultar contenidos preventivos y filtrarlos por categoría.

Los contenidos activos están disponibles para la comunidad. La consulta de contenidos y categorías se realiza mediante los endpoints publicados en el contrato OpenAPI.

### RF-10 – Registrar seguimiento de progreso

El sistema registra la visualización de contenidos por parte de los usuarios autenticados.

Se conserva la primera fecha de visualización y se actualiza la última fecha cuando el usuario vuelve a consultar el mismo contenido.

### RF-11 – Gestionar evaluaciones cortas

El sistema permite crear y administrar evaluaciones asociadas a una categoría.

Cada evaluación contiene un título, un tema y un conjunto ordenado de preguntas. Cada pregunta contiene opciones de respuesta y permite identificar la opción correcta.

Los usuarios pueden presentar varios intentos de una misma evaluación. Cada intento registra el usuario, la evaluación, el tema, el puntaje y la fecha.

### RF-12 – Exponer información mediante API

Los contenidos, categorías, evaluaciones y datos de progreso se ofrecen mediante una API REST documentada con OpenAPI 3.

Los contratos de endpoints, parámetros, solicitudes, respuestas y permisos se encuentran en [api/openapi.yml](api/openapi.yml).

### RF-13 – Gestionar consultas al asistente IA

El sistema permite que los usuarios autenticados consulten al Tutor CampusSeguro.

El tutor genera respuestas utilizando contenidos preventivos institucionales disponibles en MS07. No responde con información ajena al alcance de esos contenidos.

Cuando no existe información suficiente, el tutor indica que no puede responder y remite al protocolo oficial de emergencias de la institución.

Las consultas se registran con metadatos mínimos para auditoría. No se almacenan el texto de la pregunta ni el contenido de la respuesta en el registro de auditoría.

## 3. Reglas de negocio

- Las categorías, los contenidos y las evaluaciones no se eliminan físicamente mediante las operaciones normales del servicio; se desactivan y pueden reactivarse.
- Los contenidos se relacionan con una categoría existente.
- Las evaluaciones se relacionan con una categoría existente.
- Las preguntas pertenecen a una evaluación y sus opciones pertenecen a una pregunta.
- Las preguntas y opciones conservan su orden mediante un campo de posición.
- Al actualizar una evaluación, sus preguntas y opciones se reemplazan como parte de la actualización.
- Una evaluación tiene de 1 a 50 preguntas, y cada pregunta de 2 a 10 opciones con exactamente una correcta.
- Una evaluación que ya tiene intentos no se edita: se desactiva y se crea una nueva.
- Un intento responde una vez cada pregunta de la evaluación y solo se acepta si la evaluación está activa.
- Un contenido o una evaluación solo se asocia a una categoría activa.
- El nombre de una categoría es único.
- Un usuario puede presentar varios intentos de una misma evaluación.
- El puntaje de un intento está comprendido entre 0 y 100.
- El seguimiento de visualización mantiene un único registro por usuario y contenido. Las nuevas visualizaciones actualizan la última fecha sin perder la primera.
- El progreso y los intentos se consultan para el usuario identificado por `X-User-Id`; no se permite consultar el progreso de otro usuario mediante estos endpoints.
- El tutor utiliza contenidos activos como fuente de información.
- La pregunta al tutor no puede quedar vacía ni superar 500 caracteres después de limpiarla.
- Si el tutor no encuentra información suficiente, responde indicando que la consulta está fuera de su alcance.
- Los registros de auditoría del tutor contienen metadatos y resultado, pero no guardan preguntas ni respuestas.

## 4. Seguridad y autorización

MS07 no valida el JWT: autoriza cada operación con los permisos de `X-User-Permission` que inyecta el API Gateway (`ms07:content:read`, `ms07:quiz:take`, `ms07:tutor:use` y `ms07:content:manage`). La identidad, la autorización por ruta, las llamadas salientes y el manejo de datos sensibles están en [security.md](security.md), y la asignación de permisos a los roles en la [matriz de permisos](../integration/permissions-matrix.md).

## 5. Integración con inteligencia artificial

El tutor utiliza adaptadores para comunicarse con proveedores de modelos de lenguaje. El proveedor se selecciona mediante configuración del servicio.

Las variables de configuración son `LLM_PROVIDER`, `LLM_URL`, `LLM_API_KEY`, `LLM_MODEL`, `LLM_TIMEOUT`, `LLM_MAX_TOKENS` y `LLM_TEMPERATURE`.

La clave del proveedor llega por variable de entorno y no está en el repositorio.

Si el proveedor no está disponible, no está configurado o supera el tiempo de espera, el servicio responde `503 DEPENDENCIA_NO_DISPONIBLE`.

## 6. Auditoría

MS07 registra las consultas al tutor y las operaciones administrativas de categorías, contenidos y evaluaciones.

Los eventos se envían a MS08 mediante el API Gateway. El catálogo de acciones, recursos y atributos se encuentra en [integration/audit-catalog.md](../integration/audit-catalog.md).

No se auditan las consultas de progreso ni las visualizaciones de contenidos.
