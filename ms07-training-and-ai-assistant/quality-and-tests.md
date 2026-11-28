# Calidad y pruebas — MS07 Formación y asistente de IA

Estrategia de pruebas, cobertura y análisis estático de MS07. El comportamiento está en [1_requirements.md](1_requirements.md) y la seguridad en [security.md](security.md). Las cifras corresponden a la ejecución de GitHub Actions del workflow `ci.yml` sobre la rama `develop` del repositorio `ms07-training-and-ai-assistant` (commit `14bf052`).

## Estrategia de pruebas

Las pruebas de `mvn verify` no necesitan PostgreSQL, Docker ni otros servicios: arrancan la aplicación sobre H2 en memoria en modo PostgreSQL (perfil `test`, con el esquema creado por Hibernate y Flyway desactivado), y el proveedor del modelo de lenguaje y el API Gateway se reemplazan por dobles.

| Nivel | Qué prueba | Cómo |
|---|---|---|
| Controladores y contrato HTTP | Categorías, contenidos, evaluaciones, intentos, visualizaciones, progreso y tutor: rutas, permisos, códigos de estado, cuerpos y `ErrorResponse` | `@SpringBootTest` con `MockMvc` sobre H2, con la cadena de seguridad real y las cabeceras `X-User-*` (`CategoryControllerTest`, `ContentControllerTest`, `EvaluationControllerTest`, `EvaluationAttemptControllerTest`, `ContentViewControllerTest`, `ProgressControllerTest`, `TutorControllerTest`) |
| Tutor | Limpieza y validación de la pregunta, búsqueda de contenidos relacionados, términos de alcance, falla de la base de datos y de la auditoría | `InputAnalyzerTest`, `ContentRetrieverTest`, `TutorTermsTest`, `TutorContentDatabaseFailureTest`, `TutorAuditFailureTest`, con un proveedor de IA falso |
| Adaptadores de IA | Petición a la API de *chat completions* (autenticación, cuerpo, tiempo agotado y errores del proveedor) | `ChatCompletionsAdapterTest` con `MockRestServiceServer` como proveedor |
| Auditoría | Envío asíncrono a MS08 por el Gateway con la clave de servicio y el Correlation ID | `AuditGatewayClientTest` con `MockRestServiceServer` como Gateway |
| Seguridad | Construcción de la identidad a partir de las cabeceras del Gateway | `HeaderAuthenticationFilterTest` |
| Arranque | Que el contexto de Spring arranca | `Ms07TrainingApplicationTest` |

### Tecnologías y por qué se usan

| Tecnología | Uso | Por qué |
|---|---|---|
| JUnit 5 (Jupiter) | Motor de pruebas | Es el estándar de Spring Boot |
| Spring Boot Test y `MockMvc` | Contrato HTTP con la seguridad real | Prueba rutas, permisos y respuestas sin levantar un servidor |
| H2 en modo PostgreSQL | Base de datos de las pruebas | Ejecuta los repositorios JPA sin Docker ni PostgreSQL |
| `MockRestServiceServer` | Dobles del proveedor de IA y del Gateway | Verifica la petición HTTP que sale de MS07 sin llamar a servicios externos |
| JaCoCo | Cobertura | Genera el reporte que se publica como artefacto del workflow |

## Resultados de la ejecución

| Pruebas | Clases de prueba | Fallos | Errores | Omitidas |
|---:|---:|---:|---:|---:|
| 148 | 16 | 0 | 0 | 0 |

| Paquete de prueba | Pruebas |
|---|---:|
| `tutor` | 44 |
| `evaluation` | 35 |
| `progress` | 16 |
| `category` | 15 |
| `content` | 13 |
| `tutor.ai` | 12 |
| `audit.gateway` | 6 |
| `security` | 6 |
| `training` (arranque) | 1 |

## Cobertura

JaCoCo sobre todo el código de `src/main/java`.

| Alcance | Instrucciones | Ramas | Líneas | Métodos |
|---|---:|---:|---:|---:|
| **Total** | **89,8 %** (5677/6320) | **73,3 %** (269/367) | **89,4 %** (1043/1167) | **92,4 %** (340/368) |
| `tutor` | 99,9 % (1512/1514) | 95,7 % (88/92) | 100,0 % (170/170) | 100,0 % (43/43) |
| `tutor.ai` | 100,0 % (423/423) | 77,8 % (28/36) | 100,0 % (83/83) | 100,0 % (26/26) |
| `evaluation` | 98,8 % (1121/1135) | 94,6 % (53/56) | 98,3 % (233/237) | 98,8 % (79/80) |
| `content` | 97,8 % (486/497) | 95,8 % (23/24) | 98,2 % (111/113) | 100,0 % (43/43) |
| `category` | 96,8 % (389/402) | 95,5 % (21/22) | 95,8 % (91/95) | 100,0 % (37/37) |
| `progress` | 98,5 % (391/397) | 87,5 % (7/8) | 97,2 % (70/72) | 93,3 % (28/30) |
| `security` | 94,4 % (408/432) | 76,9 % (20/26) | 94,2 % (81/86) | 100,0 % (18/18) |
| `audit` | 86,4 % (171/198) | 42,9 % (6/14) | 88,9 % (48/54) | 88,9 % (16/18) |
| `audit.gateway` | 98,6 % (212/215) | 70,0 % (14/20) | 100,0 % (39/39) | 100,0 % (7/7) |
| `common` | 91,7 % (66/72) | 66,7 % (4/6) | 90,9 % (10/11) | 100,0 % (4/4) |
| `exception` | 57,4 % (247/430) | 12,0 % (3/25) | 56,0 % (47/84) | 68,6 % (24/35) |
| `config` | 41,5 % (248/597) | 5,3 % (2/38) | 49,2 % (59/120) | 56,0 % (14/25) |
| `training` (clase principal) | 37,5 % (3/8) | — (0/0) | 33,3 % (1/3) | 50,0 % (1/2) |

Los paquetes son subpaquetes de `edu.eci.dosw.campusseguro.training`. La clase principal solo contiene el `main` que arranca Spring.

## Análisis estático

| Herramienta | Versión | Hallazgos |
|---|---|---:|
| PMD | - | - |
| SpotBugs | - | - |

El `pom.xml` de MS07 no incluye PMD ni SpotBugs.

## Cómo ejecutar localmente

Requiere JDK 21; el repositorio incluye el Maven Wrapper (`mvnw`). Las pruebas no requieren PostgreSQL, Docker ni una clave del proveedor de IA.

| Comando | Resultado |
|---|---|
| `./mvnw test` | Todas las pruebas |
| `./mvnw test -Dtest=TutorControllerTest` | Una clase de prueba |
| `./mvnw verify` | Pruebas y cobertura |

| Reporte | Ruta |
|---|---|
| Resultados de Surefire | `target/surefire-reports/` |
| Cobertura | `target/site/jacoco/index.html`, `target/site/jacoco/jacoco.xml` |

## Integración continua

El workflow `ci.yml` de GitHub Actions ejecuta `./mvnw --batch-mode clean verify`, guarda el reporte de JaCoCo como artefacto (`ms07-jacoco-report`) y, si las pruebas pasan, construye la imagen Docker y la publica en GHCR. El workflow no ejecuta SonarCloud.
