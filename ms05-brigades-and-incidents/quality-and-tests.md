# Calidad y pruebas — MS05 Brigadistas e incidentes

Estrategia de pruebas, cobertura y análisis estático de MS05. El comportamiento está en [1_requirements.md](1_requirements.md) y la seguridad en [security.md](security.md). Las cifras corresponden a la ejecución de GitHub Actions del workflow `ci-quality.yml` sobre la rama `develop` del repositorio `ms05-brigades-and-incidents` (commit `1de2549`).

## Estrategia de pruebas

Las pruebas de `mvn verify` no necesitan PostgreSQL, Docker ni otros microservicios: los repositorios y los clientes de MS01, MS02, MS03 y MS08 se reemplazan por dobles.

| Nivel | Qué prueba | Cómo |
|---|---|---|
| Servicios (unitarias) | Asignación y búsqueda de brigadistas, activación, incidentes y su ciclo de atención, evidencias, responsabilidades, integración con MS06 y auditoría | JUnit 5 y Mockito (`@ExtendWith(MockitoExtension.class)`): repositorios y clientes son dobles |
| Controladores | Rutas, parámetros, códigos de estado, cuerpos, `ErrorResponse` y permisos | `@WebMvcTest` con `MockMvc` y la cadena de seguridad real; los servicios son dobles |
| Clientes de otros microservicios | Llamadas a MS01, MS02, MS03 y MS08 por el Gateway, con la clave de servicio y la identidad propagada | Pruebas del paquete `client` con el cliente HTTP como doble |
| Seguridad | Lectura de `X-User-Id`, `X-User-Permission` y `X-Service-Name` y respuestas 401/403 | Pruebas del paquete `security` |

### Tecnologías y por qué se usan

| Tecnología | Uso | Por qué |
|---|---|---|
| JUnit 5 (Jupiter) | Motor de pruebas | Es el estándar de Spring Boot |
| Mockito | Dobles de repositorios y clientes | Aísla la regla de negocio de la base de datos y de los demás microservicios |
| Spring Boot Test, `@WebMvcTest` y `MockMvc` | Capa web con la seguridad real | Prueba el contrato HTTP sin levantar la aplicación completa |
| JaCoCo | Cobertura y umbral mínimo | La regla `check` exige al menos 80 % de líneas cubiertas; si no se cumple, `mvn verify` falla |
| SpotBugs | Análisis estático sobre el bytecode | Corre en `mvn verify` y hace fallar el build con un hallazgo |

## Resultados de la ejecución

| Pruebas | Clases de prueba | Fallos | Errores | Omitidas |
|---:|---:|---:|---:|---:|
| 417 | 41 | 0 | 0 | 0 |

| Paquete de prueba | Pruebas |
|---|---:|
| `service` | 250 |
| `controller` | 59 |
| `client` | 52 |
| `exception` | 19 |
| `entity` | 13 |
| `security` | 11 |
| `config` | 8 |
| `mapper` | 3 |
| `event` | 2 |

## Cobertura

JaCoCo sobre todo el código de `src/main/java`.

| Alcance | Instrucciones | Ramas | Líneas | Métodos |
|---|---:|---:|---:|---:|
| **Total** | **84,8 %** (10314/12164) | **64,9 %** (761/1172) | **84,8 %** (2635/3108) | **82,5 %** (597/724) |
| `service` | 88,5 % (6059/6849) | 70,2 % (567/808) | 90,2 % (1640/1818) | 87,7 % (235/268) |
| `client` | 69,2 % (1199/1733) | 38,8 % (66/170) | 67,7 % (243/359) | 73,7 % (56/76) |
| `dto` | 96,9 % (821/847) | 56,7 % (17/30) | 90,4 % (85/94) | 100,0 % (42/42) |
| `entity` | 72,2 % (604/836) | 38,2 % (13/34) | 71,4 % (240/336) | 70,0 % (133/190) |
| `controller` | 82,1 % (421/513) | 75,0 % (9/12) | 83,0 % (132/159) | 88,0 % (44/50) |
| `security` | 98,2 % (324/330) | 88,9 % (32/36) | 98,5 % (65/66) | 95,2 % (20/21) |
| `event` | 96,0 % (312/325) | 65,0 % (13/20) | 92,8 % (77/83) | 100,0 % (13/13) |
| `exception` | 99,4 % (310/312) | 93,3 % (28/30) | 100,0 % (83/83) | 100,0 % (30/30) |
| `config` | 88,0 % (184/209) | 100,0 % (16/16) | 81,6 % (40/49) | 84,6 % (11/13) |
| `repository` | 14,4 % (21/146) | 0,0 % (0/16) | 19,4 % (7/36) | 50,0 % (7/14) |
| `mapper` | 100,0 % (56/56) | — (0/0) | 100,0 % (22/22) | 100,0 % (5/5) |
| `brigades_and_incidents` (clase principal) | 37,5 % (3/8) | — (0/0) | 33,3 % (1/3) | 50,0 % (1/2) |

Los paquetes son subpaquetes de `edu.eci.dosw.brigades_and_incidents`.

## Análisis estático

| Herramienta | Versión | Hallazgos |
|---|---|---:|
| PMD | - | - |
| SpotBugs (`spotbugs-maven-plugin` 4.10.5.0, esfuerzo `Max`, umbral `Medium`) | - | 0 |

El `pom.xml` de MS05 no incluye PMD. MS05 no usa archivo de exclusiones de SpotBugs: los casos aceptados se marcan en el código con `@SuppressFBWarnings` y su justificación.

## Cómo ejecutar localmente

Requiere JDK 21; el repositorio incluye el Maven Wrapper (`mvnw`). Las pruebas no requieren PostgreSQL, Docker ni otros servicios.

| Comando | Resultado |
|---|---|
| `./mvnw test` | Todas las pruebas |
| `./mvnw verify` | Pruebas, cobertura con umbral y SpotBugs |

| Reporte | Ruta |
|---|---|
| Resultados de Surefire | `target/surefire-reports/` |
| Cobertura | `target/site/jacoco/index.html`, `target/site/jacoco/jacoco.xml` |
| SpotBugs | `target/spotbugsXml.xml` |

## Integración continua

El workflow `ci-quality.yml` de GitHub Actions ejecuta `./mvnw -B -ntp clean verify`, guarda el reporte de JaCoCo como artefacto (`backend-jacoco`) y, en push a `develop` o `main`, construye y publica la imagen en GHCR; desde `develop` además despliega en el ambiente de prueba. El workflow no ejecuta SonarCloud.
