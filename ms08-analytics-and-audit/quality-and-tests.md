# Calidad y pruebas — MS08 Analítica y auditoría

Estrategia de pruebas, cobertura y análisis estático de MS08 ([RNF-AYA-06](1_requirements.md#requerimientos-no-funcionales-rnf)). El comportamiento está en [1_requirements.md](1_requirements.md) y la seguridad en [security.md](security.md). Las cifras corresponden a la ejecución de GitHub Actions del workflow `ci.yml` sobre la rama `develop` del repositorio `ms08-analytics-and-audit` (commit `8662d94`).

## Estrategia de pruebas

`mvn verify` ejecuta dos fases: las pruebas unitarias (Maven Surefire), que no necesitan base de datos, y las pruebas de integración (Maven Failsafe, clases `*IT`), que levantan MongoDB con Testcontainers y por eso requieren Docker.

| Nivel | Qué prueba | Cómo |
|---|---|---|
| Servicios (unitarias) | Registro de auditoría con sus validaciones, idempotencia y logs; consulta con filtros, visibilidad y paginación; indicadores, comparaciones y ubicaciones | JUnit 5 y Mockito: el repositorio y el `MongoTemplate` son dobles (`AuditIngestionServiceTest`, `AuditQueryServiceTest`, `EventIndicatorServiceTest`, `EventAnalyticsServiceTest`) |
| Cálculo de indicadores | Fórmulas de notificación, participación, incidentes, tiempos y ubicaciones con hechos construidos en memoria | `EventIndicatorCalculatorTest` y `LocationAnalyticsCalculatorTest`, sin dobles: el cálculo es una función pura |
| Seguridad | Autenticación por `X-Service-Name` o por `X-User-Id` + `X-User-Permission`, visibilidad por permiso, Correlation ID y respuestas 401/403 | `GatewayAuthenticationFilterTest`, `AuditVisibilityTest`, `IndicatorVisibilityTest`, `CorrelationIdFilterTest`, `CorrelationIdsTest`, `SecurityHandlersLoggingTest` |
| Errores | Traducción de excepciones a `ErrorResponse`, códigos y lo que se registra en el log | `GlobalExceptionHandlerTest`, `ApiErrorControllerTest`, `ErrorCodeTest`, `RequestRejectionLoggingTest` |
| Mapeo y retención | Conversión de la petición al documento (minúsculas, UTC, `id` derivado) y la tarea diaria de retención | `AuditEventMapperTest` y `AuditRetentionJobTest` |
| Integración con MongoDB | Endpoints completos con la seguridad real sobre MongoDB: registro e idempotencia, búsqueda, detalle, indicadores, comparaciones, ubicaciones y arranque con índices | Clases `*IT` (`AuditEventIT`, `AuditEventQueryIT`, `EventIndicatorIT`, `EventAnalyticsIT`, `AuditApplicationIT`) sobre `MongoIntegrationTest`, que levanta MongoDB con Testcontainers |

### Tecnologías y por qué se usan

| Tecnología | Uso | Por qué |
|---|---|---|
| JUnit 5 (Jupiter) | Motor de pruebas | Es el estándar de Spring Boot |
| Mockito | Dobles del repositorio y del `MongoTemplate` | Aísla la regla de negocio de la base de datos |
| Spring Boot Test y `MockMvc` | Contrato HTTP con la cadena de seguridad real | Prueba rutas, cabeceras, códigos y cuerpos tal como los recibe el Gateway |
| Testcontainers (MongoDB) | Base de datos real en las pruebas `*IT` | Verifica consultas, índices e idempotencia contra MongoDB |
| JaCoCo | Cobertura y umbral mínimo | La regla `check` exige al menos 80 % de líneas cubiertas; si no se cumple, `mvn verify` falla |

## Resultados de la ejecución

| Fase | Pruebas | Fallos | Errores | Omitidas |
|---|---:|---:|---:|---:|
| Unitarias (Surefire) | 111 | 0 | 0 | 0 |
| Integración (Failsafe) | 36 | 0 | 0 | 0 |
| **Total** | **147** | **0** | **0** | **0** |

| Paquete de prueba | Pruebas |
|---|---:|
| `audit.service.impl` | 47 |
| `audit.controller` | 33 |
| `audit.security` | 26 |
| `audit.exception` | 18 |
| `audit.service` | 14 |
| `audit.mapper` | 4 |
| `audit` (arranque) | 3 |
| `audit.scheduler` | 2 |

## Cobertura

JaCoCo sobre `src/main/java`, sin la clase principal (`AuditApplication`, que solo arranca Spring).

| Alcance | Instrucciones | Ramas | Líneas | Métodos |
|---|---:|---:|---:|---:|
| **Total** | **98,3 %** (4644/4726) | **87,4 %** (327/374) | **98,3 %** (799/813) | **96,7 %** (208/215) |
| `service.impl` | 99,9 % (1264/1265) | 92,2 % (118/128) | 100,0 % (231/231) | 100,0 % (44/44) |
| `service` | 97,6 % (1295/1327) | 80,0 % (112/140) | 98,1 % (203/207) | 94,2 % (49/52) |
| `exception` | 93,5 % (560/599) | 88,9 % (16/18) | 92,9 % (92/99) | 91,2 % (31/34) |
| `security` | 100,0 % (499/499) | 92,1 % (70/76) | 100,0 % (124/124) | 100,0 % (32/32) |
| `dto` | 100,0 % (369/369) | — (0/0) | 100,0 % (18/18) | 100,0 % (18/18) |
| `config` | 98,9 % (270/273) | 50,0 % (1/2) | 97,9 % (47/48) | 91,7 % (11/12) |
| `mapper` | 95,2 % (138/145) | 100,0 % (6/6) | 95,1 % (39/41) | 100,0 % (5/5) |
| `controller` | 100,0 % (95/95) | 100,0 % (4/4) | 100,0 % (20/20) | 100,0 % (9/9) |
| `entity` | 100,0 % (75/75) | — (0/0) | 100,0 % (7/7) | 100,0 % (3/3) |
| `scheduler` | 100,0 % (46/46) | — (0/0) | 100,0 % (12/12) | 100,0 % (4/4) |
| `repository` | 100,0 % (33/33) | — (0/0) | 100,0 % (6/6) | 100,0 % (2/2) |

Los paquetes son subpaquetes de `edu.eci.dosw.campusseguro.audit`.

## Análisis estático

| Herramienta | Versión | Hallazgos |
|---|---|---:|
| PMD | - | - |
| SpotBugs | - | - |

El `pom.xml` de MS08 no incluye PMD ni SpotBugs.

## Cómo ejecutar localmente

Requiere JDK 21 y Maven. Las pruebas de integración (`*IT`) requieren Docker en ejecución para Testcontainers.

| Comando | Resultado |
|---|---|
| `mvn test` | Pruebas unitarias |
| `mvn test -Dtest=AuditIngestionServiceTest` | Una clase de prueba |
| `mvn verify` | Pruebas unitarias, de integración, cobertura y umbral de JaCoCo |

| Reporte | Ruta |
|---|---|
| Resultados de Surefire | `target/surefire-reports/` |
| Resultados de Failsafe | `target/failsafe-reports/` |
| Cobertura | `target/site/jacoco/index.html`, `target/site/jacoco/jacoco.xml` |

## Integración continua

El workflow `ci.yml` de GitHub Actions ejecuta `mvn -B -ntp clean verify` en cada push y pull request a `main` y `develop`, guarda el reporte de JaCoCo como artefacto (`ms08-jacoco`) y, si las pruebas pasan, construye la imagen Docker; desde `main` y `develop` la publica en GHCR. El workflow no ejecuta SonarCloud.
