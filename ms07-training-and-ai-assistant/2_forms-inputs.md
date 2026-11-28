# Formularios y entradas — MS07 Formación y asistente de IA

Campos que recibe MS07, sus reglas de validación y los mensajes que llegan al frontend. El comportamiento está en [1_requirements.md](1_requirements.md) y los endpoints en [`api/openapi.yml`](api/openapi.yml).

## Índice de Contenido
1. [Reglas generales de entrada](#reglas-generales-de-entrada)
2. [F-01 Categoría](#f-01-categoría)
3. [F-02 Contenido](#f-02-contenido)
4. [F-03 Evaluación](#f-03-evaluación)
5. [F-04 Intento de evaluación](#f-04-intento-de-evaluación)
6. [F-05 Pregunta al tutor](#f-05-pregunta-al-tutor)
7. [F-06 Listados y paginación](#f-06-listados-y-paginación)

---

## Reglas generales de entrada

* Los cuerpos son JSON con los nombres de campo en `camelCase` (`categoryId`, `questionId`).
* Un error de validación responde `400 VALIDACION` con el mensaje `Datos inválidos.` y, en `errores`, el campo y su mensaje.
* Una regla de negocio incumplida responde `422 REGLA_NEGOCIO`; un conflicto con el estado actual, `409 CONFLICTO`; un recurso inexistente, `404 NO_ENCONTRADO`.
* Un JSON mal formado o un valor de tipo incorrecto (por ejemplo, un UUID inválido) responde `400 VALIDACION`.
* Los campos que MS07 no reconoce se ignoran.

---

## F-01 Categoría

**Endpoints:** `POST /api/v1/training/categories` y `PUT /api/v1/training/categories/{id}` · **Permiso:** `ms07:content:manage`

| Campo | Tipo | Obligatorio | Longitud / formato | Mensaje de error | Ejemplo válido |
|---|---|---|---|---|---|
| `name` | string | Sí | Máximo 100 caracteres; único entre las categorías | `El nombre es obligatorio.` / `El nombre debe tener máximo 100 caracteres.` / `Ya existe una categoría con ese nombre.` (409) | `Evacuación` |
| `description` | string | No | Máximo 500 caracteres | `La descripción debe tener máximo 500 caracteres.` | `Protocolos para salir del edificio` |

---

## F-02 Contenido

**Endpoints:** `POST /api/v1/training/contents` y `PUT /api/v1/training/contents/{id}` · **Permiso:** `ms07:content:manage`

| Campo | Tipo | Obligatorio | Longitud / formato | Mensaje de error | Ejemplo válido |
|---|---|---|---|---|---|
| `title` | string | Sí | Máximo 150 caracteres | `El título es obligatorio.` / `El título debe tener máximo 150 caracteres.` | `Qué hacer durante un sismo` |
| `body` | string | Sí | Máximo 20000 caracteres | `El cuerpo es obligatorio.` / `El cuerpo debe tener máximo 20000 caracteres.` | `Agáchese, cúbrase y sujétese…` |
| `categoryId` | string (UUID) | Sí | UUID de una categoría existente y activa | `La categoría es obligatoria.` / `La categoría no existe.` (422) / `La categoría '<nombre>' está inactiva.` (422) | `5b1e8c2a-3f4d-4e6a-9b7c-1d2e3f4a5b6c` |

---

## F-03 Evaluación

**Endpoints:** `POST /api/v1/training/evaluations` y `PUT /api/v1/training/evaluations/{id}` · **Permiso:** `ms07:content:manage`

| Campo | Tipo | Obligatorio | Longitud / formato | Mensaje de error | Ejemplo válido |
|---|---|---|---|---|---|
| `title` | string | Sí | Máximo 150 caracteres | `El título es obligatorio.` / `El título debe tener máximo 150 caracteres.` | `Evaluación de evacuación` |
| `topic` | string | Sí | Máximo 150 caracteres | `El tema es obligatorio.` / `El tema debe tener máximo 150 caracteres.` | `Rutas de evacuación` |
| `categoryId` | string (UUID) | Sí | UUID de una categoría existente y activa | `La categoría es obligatoria.` / `La categoría no existe.` (422) / `La categoría '<nombre>' está inactiva.` (422) | `5b1e8c2a-3f4d-4e6a-9b7c-1d2e3f4a5b6c` |
| `questions` | lista | Sí | De 1 a 50 preguntas, en el orden en que se muestran | `La evaluación debe tener al menos una pregunta.` (422) / `La evaluación debe tener máximo 50 preguntas.` | — |
| `questions[].statement` | string | Sí | Máximo 500 caracteres | `El enunciado es obligatorio.` / `El enunciado debe tener máximo 500 caracteres.` | `¿Por dónde se sale en caso de incendio?` |
| `questions[].options` | lista | Sí | De 2 a 10 opciones, exactamente una correcta | `La pregunta <n> debe tener al menos dos opciones.` (422) / `Cada pregunta debe tener máximo 10 opciones.` / `La pregunta <n> debe tener exactamente una opción correcta.` (422) | — |
| `questions[].options[].text` | string | Sí | Máximo 300 caracteres | `El texto de la opción es obligatorio.` / `El texto de la opción debe tener máximo 300 caracteres.` | `Por la escalera de emergencia` |
| `questions[].options[].correct` | booleano | No (`false` por defecto) | — | — | `true` |

Una evaluación que ya tiene intentos no se puede editar: responde `409` con `La evaluación ya tiene intentos registrados: inactívela y cree una nueva.`

---

## F-04 Intento de evaluación

**Endpoint:** `POST /api/v1/training/evaluations/{id}/attempts` · **Permiso:** `ms07:quiz:take`

| Campo | Tipo | Obligatorio | Reglas | Mensaje de error |
|---|---|---|---|---|
| `answers` | lista | Sí | Una respuesta por cada pregunta de la evaluación | `Las respuestas son obligatorias.` / `Falta responder la pregunta <n>.` / `Cada pregunta se responde una sola vez.` / `Hay respuestas a preguntas que no son de esta evaluación.` |
| `answers[].questionId` | string (UUID) | Sí | Pregunta de esta evaluación | `La pregunta es obligatoria.` |
| `answers[].optionId` | string (UUID) | Sí | Opción de esa pregunta | `La opción es obligatoria.` / `La opción elegida no pertenece a la pregunta <n>.` |

Responder una evaluación inactiva responde `422` con `La evaluación no está activa.`

---

## F-05 Pregunta al tutor

**Endpoint:** `POST /api/v1/tutor/questions` · **Permiso:** `ms07:tutor:use`

| Campo | Tipo | Obligatorio | Reglas | Mensaje de error | Ejemplo válido |
|---|---|---|---|---|---|
| `question` | string | Sí | Se convierte en una sola línea con espacios simples y sin caracteres de control ni invisibles; después de limpiarla no puede quedar vacía ni pasar de 500 caracteres. Conserva tildes, `ñ` y `¿?` | `La pregunta es obligatoria.` / `La pregunta debe tener máximo 500 caracteres.` | `¿Qué hago si hay un sismo en el auditorio?` |

Si el proveedor del modelo de lenguaje no responde o no está configurado, responde `503 DEPENDENCIA_NO_DISPONIBLE` con `El tutor no está disponible en este momento. Intenta de nuevo o consulta el centro de aprendizaje.`

---

## F-06 Listados y paginación

**Endpoints:** `GET /api/v1/training/categories`, `GET /api/v1/training/contents` y `GET /api/v1/training/evaluations`

| Parámetro | Tipo | Obligatorio | Reglas | Mensaje de error |
|---|---|---|---|---|
| `categoryId` | UUID | No | Solo en contenidos y evaluaciones: filtra por categoría | Error de tipo si no es un UUID |
| `includeInactive` | booleano | No (`false` por defecto) | Incluye los elementos inactivos; solo tiene efecto con `ms07:content:manage` | — |
| `page` | entero | No (`0` por defecto) | 0 o mayor | `La página debe ser 0 o mayor.` |
| `size` | entero | No (`20` por defecto) | Entre 1 y 100 | `El tamaño de página debe estar entre 1 y 100.` |
