# Calidad y pruebas — MS02 Campus y seguridad

Estrategia de pruebas, cobertura y análisis estático de MS02. El comportamiento está en [1_requirements.md](1_requirements.md) y la seguridad en [security.md](security.md). Las cifras corresponden a la ejecución de GitHub Actions del workflow `ci.yml` sobre la rama `develop` del repositorio `ms02-campus-and-security` (commit `e5c79d2`).

## Estrategia de pruebas

Las pruebas de `mvn verify` no necesitan PostgreSQL, Docker ni el proveedor de navegación: los servicios se prueban con dobles y las pruebas que arrancan la aplicación usan H2 en memoria en modo PostgreSQL (con el esquema creado por Hibernate, Flyway desactivado y la generación automática de rutas exteriores apagada). El arranque real contra PostgreSQL con las migraciones de Flyway lo verifica un job aparte del workflow.

| Nivel | Qué prueba | Cómo |
|---|---|---|
| Servicios (unitarias) | Reglas de campus, edificios, pisos, espacios, zonas seguras y abiertas, salidas, escaleras, recomendaciones, imágenes y códigos QR; consulta y generación de rutas | JUnit 5 y Mockito (`@ExtendWith(MockitoExtension.class)`): repositorios, almacenamiento y proveedor de navegación son dobles |
| Controladores y contrato HTTP | Rutas, parámetros, códigos de estado, cuerpos, `ErrorResponse` y permisos por ruta | `@WebMvcTest` por controlador con `MockMvc` y la cadena de seguridad real; los servicios son dobles |
| Rutas | Dominio del grafo de rutas, cálculo de recorridos, adaptador de OpenStreetMap (OSRM) y generación automática | Pruebas de `ruta.domain`, `ruta.service` y `ruta.navegacion`; el servidor de navegación es un doble |
| Adaptadores internos y almacenamiento | Puertos de rutas (`gateway`) y almacenamiento de imágenes local y en Azure Blob | Pruebas de `gateway` y `storage` con dobles |
| Integración en H2 | Flujos que cruzan varias capas con la base de datos | `@SpringBootTest` con el perfil de pruebas sobre H2 |
| Configuración y contrato | Permisos de las rutas, el OpenAPI que genera la aplicación y el cumplimiento de los Estándares de Integración | `ContratoPermisosTest`, `OpenApiTest`, `EstandaresIntegracionTest` |

### Tecnologías y por qué se usan

| Tecnología | Uso | Por qué |
|---|---|---|
| JUnit 5 (Jupiter) | Motor de pruebas | Es el estándar de Spring Boot |
| Mockito | Dobles de repositorios, almacenamiento y proveedor de navegación | Aísla la regla de negocio de la base de datos y de servicios externos |
| Spring Boot Test, `@WebMvcTest` y `MockMvc` | Capa web con la seguridad real | Prueba el contrato HTTP sin levantar la aplicación completa |
| H2 en modo PostgreSQL | Base de datos de las pruebas de integración | Ejecuta JPA sin Docker ni PostgreSQL |
| JaCoCo | Cobertura | Genera el `jacoco.xml` que lee SonarCloud |

## Resultados de la ejecución

| Pruebas | Clases de prueba | Fallos | Errores | Omitidas |
|---:|---:|---:|---:|---:|
| 557 | 55 | 0 | 0 | 0 |

| Paquete de prueba | Pruebas |
|---|---:|
| `service` | 198 |
| `controller` | 109 |
| `ruta.service` | 72 |
| `security` | 66 |
| `config` | 25 |
| `gateway` | 21 |
| `ruta.navegacion` | 15 |
| `storage` | 14 |
| `ruta.controller` | 10 |
| `ruta.domain` | 8 |
| `integration` | 7 |
| `ruta.integration` | 6 |
| `exception` | 5 |
| `campus` (arranque) | 1 |

## Cobertura

JaCoCo sobre todo el código de `src/main/java`.

| Alcance | Instrucciones | Ramas | Líneas | Métodos |
|---|---:|---:|---:|---:|
| **Total** | **90,8 %** (10438/11498) | **79,2 %** (665/840) | **90,2 %** (1940/2151) | **89,9 %** (545/606) |
| `service` | 85,6 % (3553/4153) | 76,5 % (323/422) | 85,2 % (698/819) | 85,9 % (164/191) |
| `ruta.service` | 97,0 % (1920/1979) | 88,4 % (137/155) | 97,3 % (404/415) | 97,5 % (78/80) |
| `security` | 97,6 % (782/801) | 67,9 % (19/28) | 97,0 % (129/133) | 100,0 % (18/18) |
| `gateway` | 84,5 % (604/715) | 75,6 % (62/82) | 95,0 % (115/121) | 76,7 % (33/43) |
| `dto` | 95,2 % (591/621) | — (0/0) | 96,8 % (30/31) | 96,8 % (30/31) |
| `config` | 100,0 % (591/591) | 97,4 % (37/38) | 100,0 % (111/111) | 100,0 % (22/22) |
| `ruta.navegacion` | 97,3 % (462/475) | 79,6 % (39/49) | 96,3 % (78/81) | 100,0 % (21/21) |
| `exception` | 95,5 % (406/425) | 58,3 % (14/24) | 91,1 % (82/90) | 100,0 % (36/36) |
| `controller` | 76,3 % (338/443) | — (0/0) | 72,6 % (61/84) | 83,6 % (56/67) |
| `ruta.domain` | 97,7 % (301/308) | 100,0 % (8/8) | 95,9 % (71/74) | 93,5 % (29/31) |
| `ruta.dto` | 100,0 % (270/270) | — (0/0) | 100,0 % (27/27) | 100,0 % (14/14) |
| `storage` | 81,2 % (246/303) | 82,1 % (23/28) | 79,0 % (64/81) | 85,7 % (12/14) |
| `entity` | 87,8 % (144/164) | 0,0 % (0/2) | 75,0 % (27/36) | 66,7 % (6/9) |
| `ruta.port` | 93,3 % (140/150) | 75,0 % (3/4) | 85,7 % (12/14) | 90,0 % (9/10) |
| `ruta.controller` | 91,5 % (54/59) | — (0/0) | 94,7 % (18/19) | 90,0 % (9/10) |
| `ruta.exception` | 100,0 % (21/21) | — (0/0) | 100,0 % (10/10) | 100,0 % (5/5) |
| `ruta.event` | 100,0 % (12/12) | — (0/0) | 100,0 % (2/2) | 100,0 % (2/2) |
| `campus` (clase principal) | 37,5 % (3/8) | — (0/0) | 33,3 % (1/3) | 50,0 % (1/2) |

Los paquetes son subpaquetes de `edu.eci.dosw.campusseguro.campus`. Los repositorios son interfaces de Spring Data sin código propio y no aparecen en la tabla.

## Análisis estático

| Herramienta | Versión | Hallazgos |
|---|---|---:|
| PMD | - | - |
| SpotBugs | - | - |

El `pom.xml` de MS02 no incluye PMD ni SpotBugs; el análisis estático lo hace SonarCloud.

## Cómo ejecutar localmente

Requiere JDK 21 y Maven; las pruebas no requieren PostgreSQL, Docker ni el proveedor de navegación.

| Comando | Resultado |
|---|---|
| `mvn test` | Todas las pruebas |
| `mvn test -Dtest=EdificioServiceTest` | Una clase de prueba |
| `mvn verify` | Pruebas y cobertura |

| Reporte | Ruta |
|---|---|
| Resultados de Surefire | `target/surefire-reports/` |
| Cobertura (HTML y XML para SonarCloud) | `target/site/jacoco/index.html`, `target/site/jacoco/jacoco.xml` |

## Relación con SonarCloud

El workflow `ci.yml` de GitHub Actions ejecuta `mvn -B verify` en cada push y pull request a `main` y `develop`, guarda los reportes de pruebas y cobertura como artefacto (`reportes`) y, si el repositorio tiene `SONAR_TOKEN`, ejecuta el análisis de SonarCloud con ese `jacoco.xml`. Un segundo job arranca MS02 contra PostgreSQL 16 con las migraciones de Flyway y espera a que `/actuator/health` responda; si las dos verificaciones pasan, se construye la imagen Docker.
