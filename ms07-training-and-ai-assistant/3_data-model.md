# Modelo de datos de MS07

## 1. Descripción general

MS07 administra una base de datos relacional propia. El esquema se crea y evoluciona mediante migraciones Flyway.

Los identificadores de las entidades son UUID. Las fechas de creación, actualización, visualización e intento se almacenan como marcas de tiempo con zona horaria.

## 2. Entidades y tablas

### 2.1. category

Almacena las categorías de formación preventiva.

| Campo | Tipo | Restricción | Descripción |
|---|---|---|---|
| `id` | UUID | PK | Identificador de la categoría. |
| `name` | VARCHAR(100) | NOT NULL, único sin distinguir mayúsculas | Nombre de la categoría. |
| `description` | VARCHAR(500) | NULL | Descripción de la categoría. |
| `status` | VARCHAR(10) | NOT NULL | Estado: `ACTIVO` o `INACTIVO`. |

Las categorías no se eliminan físicamente; se desactivan.

### 2.2. content

Almacena los contenidos preventivos.

| Campo | Tipo | Restricción | Descripción |
|---|---|---|---|
| `id` | UUID | PK | Identificador del contenido. |
| `title` | VARCHAR(150) | NOT NULL | Título del contenido. |
| `body` | VARCHAR(20000) | NOT NULL | Cuerpo del contenido. |
| `category_id` | UUID | FK, NOT NULL | Categoría a la que pertenece. |
| `status` | VARCHAR(10) | NOT NULL | Estado: `ACTIVO` o `INACTIVO`. |
| `created_at` | TIMESTAMPTZ | NOT NULL | Fecha de creación. |
| `updated_at` | TIMESTAMPTZ | NOT NULL | Fecha de última actualización. |

Cada contenido pertenece a una categoría. Una categoría puede tener varios contenidos.

### 2.3. content_view

Registra el seguimiento de visualización de contenidos por usuario.

| Campo | Tipo | Restricción | Descripción |
|---|---|---|---|
| `id` | UUID | PK | Identificador del registro. |
| `user_id` | UUID | NOT NULL | Usuario que consultó el contenido. |
| `content_id` | UUID | FK, NOT NULL | Contenido visualizado. |
| `first_viewed_at` | TIMESTAMPTZ | NOT NULL | Primera visualización. |
| `last_viewed_at` | TIMESTAMPTZ | NOT NULL | Última visualización. |

Existe una restricción única sobre `(user_id, content_id)`, por lo que cada usuario tiene un solo registro por contenido. Las visualizaciones posteriores actualizan `last_viewed_at`.

### 2.4. evaluation

Almacena las evaluaciones asociadas a categorías.

| Campo | Tipo | Restricción | Descripción |
|---|---|---|---|
| `id` | UUID | PK | Identificador de la evaluación. |
| `title` | VARCHAR(150) | NOT NULL | Título de la evaluación. |
| `topic` | VARCHAR(150) | NOT NULL | Tema de la evaluación. |
| `category_id` | UUID | FK, NOT NULL | Categoría asociada. |
| `status` | VARCHAR(10) | NOT NULL | Estado: `ACTIVO` o `INACTIVO`. |
| `created_at` | TIMESTAMPTZ | NOT NULL | Fecha de creación. |
| `updated_at` | TIMESTAMPTZ | NOT NULL | Fecha de última actualización. |

Una categoría puede tener varias evaluaciones.

### 2.5. question

Almacena las preguntas de una evaluación.

| Campo | Tipo | Restricción | Descripción |
|---|---|---|---|
| `id` | UUID | PK | Identificador de la pregunta. |
| `evaluation_id` | UUID | FK, NOT NULL | Evaluación a la que pertenece. |
| `position` | INTEGER | NOT NULL | Orden de la pregunta. |
| `statement` | VARCHAR(500) | NOT NULL | Enunciado de la pregunta. |

La eliminación de una evaluación elimina en cascada sus preguntas.

### 2.6. question_option

Almacena las opciones de respuesta de cada pregunta.

| Campo | Tipo | Restricción | Descripción |
|---|---|---|---|
| `id` | UUID | PK | Identificador de la opción. |
| `question_id` | UUID | FK, NOT NULL | Pregunta a la que pertenece. |
| `position` | INTEGER | NOT NULL | Orden de la opción. |
| `text` | VARCHAR(300) | NOT NULL | Texto de la opción. |
| `is_correct` | BOOLEAN | NOT NULL | Indica si la opción es correcta. |

La eliminación de una pregunta elimina en cascada sus opciones.

### 2.7. evaluation_attempt

Registra cada intento de un usuario al responder una evaluación.

| Campo | Tipo | Restricción | Descripción |
|---|---|---|---|
| `id` | UUID | PK | Identificador del intento. |
| `user_id` | UUID | NOT NULL | Usuario que realizó el intento. |
| `evaluation_id` | UUID | FK, NOT NULL | Evaluación respondida. |
| `topic` | VARCHAR(150) | NOT NULL | Tema copiado de la evaluación al realizar el intento. |
| `score` | INTEGER | NOT NULL, entre 0 y 100 | Puntaje obtenido. |
| `attempted_at` | TIMESTAMPTZ | NOT NULL | Fecha y hora del intento. |

Un usuario puede registrar varios intentos para una misma evaluación.

### 2.8. audit_event

Almacena metadatos mínimos de auditoría de las consultas al tutor y otras acciones del servicio.

| Campo | Tipo | Restricción | Descripción |
|---|---|---|---|
| `id` | UUID | PK | Identificador del evento. |
| `user_id` | UUID | NOT NULL | Usuario relacionado con el evento. |
| `event_type` | VARCHAR(40) | NOT NULL | Tipo de evento registrado. |
| `occurred_at` | TIMESTAMPTZ | NOT NULL | Fecha y hora del evento. |
| `correlation_id` | VARCHAR(36) | NULL | Identificador de correlación. |
| `result` | VARCHAR(20) | NULL | Resultado de la consulta al tutor, cuando aplica. |

El resultado de una consulta al tutor puede ser `RESPONDIDA` o `FUERA_DE_ALCANCE`. No se almacenan el texto de la pregunta ni el de la respuesta.

## 3. Relaciones

- `category` 1:N `content`.
- `category` 1:N `evaluation`.
- `evaluation` 1:N `question`.
- `question` 1:N `question_option`.
- `content` 1:N `content_view`.
- `evaluation` 1:N `evaluation_attempt`.

Las referencias `user_id` identifican usuarios administrados por MS01. MS07 no mantiene una tabla local de usuarios.

## 4. Índices y restricciones

- Índice único `ux_category_name` sobre `lower(name)`.
- Índice `ix_content_category` sobre `content.category_id`.
- Restricción única `ux_content_view_user_content` sobre `content_view(user_id, content_id)`.
- Índice `ix_content_view_content` sobre `content_view.content_id`.
- Índice `ix_evaluation_category` sobre `evaluation.category_id`.
- Índice `ix_question_evaluation` sobre `question.evaluation_id`.
- Índice `ix_question_option_question` sobre `question_option.question_id`.
- Índices `ix_evaluation_attempt_user` y `ix_evaluation_attempt_evaluation`.
- Restricción `CHECK` del puntaje de `evaluation_attempt` entre 0 y 100.
- Índices `idx_audit_event_user_id` y `idx_audit_event_occurred_at`.

## 5. Migraciones

El esquema se administra mediante Flyway. Las migraciones actuales son:

| Migración | Descripción |
|---|---|
| V1 | Creación de `category`. |
| V2 | Creación de `content`. |
| V3 | Inserción inicial de categorías. |
| V4 | Inserción inicial de contenidos. |
| V5 | Creación de `content_view`. |
| V6 | Creación de `evaluation`, `question` y `question_option`. |
| V7 | Creación de `evaluation_attempt`. |
| V8 | Creación de `audit_event`. |
| V9 | Adición del campo `result` a `audit_event`. |
