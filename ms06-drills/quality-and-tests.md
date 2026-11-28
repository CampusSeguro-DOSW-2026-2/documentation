# Calidad y pruebas — MS06 Eventos

Estrategia de pruebas, cobertura y análisis estático de MS06. El comportamiento está en [1_requirements.md](1_requirements.md) y la seguridad en [security.md](security.md). Las cifras corresponden a la ejecución de GitHub Actions del workflow `ci.yml` sobre el pull request de la rama `fix/chi-outage-does-not-block-events` (commit `08cf8b1`), integrada en `develop` en el commit `2f50948`.

## Estrategia de pruebas

`mvn verify` ejecuta dos fases: las pruebas unitarias (Maven Surefire), que no necesitan base de datos ni otros servicios, y las pruebas de integración (Maven Failsafe, clases `*IT`), que heredan de `PostgresIntegrationTest`, levantan PostgreSQL con Testcontainers y por eso requieren Docker.

| Nivel | Qué prueba | Cómo |
|---|---|---|
| Servicios (unitarias) | Creación, edición, programación, inicio, finalización y evaluación de eventos; coordinación con MS02, MS03, MS04 y MS05; bandeja de auditoría y su envío | JUnit 5 y Mockito: repositorios y gateways de los demás microservicios son dobles |
| Seguridad | Lectura de `X-User-*`, permisos por tipo de evento (simulacro o real) y respuestas 401/403; el perfil local con su emisor de tokens | Pruebas de los paquetes `security` y `security.local` |
| Clientes de otros microservicios | Llamadas a MS02, MS03, MS04, MS05 y MS08 por el Gateway: ruta, cabeceras, clave de servicio, lectura de la respuesta, errores y tiempos agotados | WireMock como Gateway (`WireMockTestBase` y las pruebas `Rest*GatewayTest`) |
| Errores | Traducción de excepciones a `ErrorResponse` | `GlobalExceptionHandlerTest` y `ApiErrorControllerTest` con `MockMvc` |
| Programación | Inicio automático, sincronización de brigadistas, reintento de evaluaciones y envío de auditoría | Pruebas del paquete `scheduler` |
| Integración | API completa, creación, edición, inicio y finalización de eventos, sincronización de brigadistas, bandeja de auditoría y repositorios sobre PostgreSQL | Clases `*IT` (`EventApiIT`, `EventCreationIT`, `EventUpdateIT`, `EventStartIT`, `EventFinishIT`, `BrigadierSyncIT`, `AuditOutboxIT`, `EventRepositoryIT`, `RelatedRepositoriesIT`) con Testcontainers |

### Tecnologías y por qué se usan

| Tecnología | Uso | Por qué |
|---|---|---|
| JUnit 5 (Jupiter) | Motor de pruebas | Es el estándar de Spring Boot |
| Mockito | Dobles de repositorios y gateways | Aísla la regla de negocio |
| Spring Boot Test y `MockMvc` | Capa web y contexto completo en las pruebas `*IT` | Prueba el contrato HTTP con la configuración real |
| WireMock | Doble del Gateway | Registra la petición real que sale de MS06 |
| Testcontainers (PostgreSQL) | Base de datos real en las pruebas `*IT` | Verifica el esquema y las consultas contra PostgreSQL |
| JaCoCo | Cobertura y umbral mínimo | La regla `check` exige al menos 80 % de líneas cubiertas; si no se cumple, `mvn verify` falla |

## Resultados de la ejecución

| Fase | Pruebas | Fallos | Errores | Omitidas |
|---|---:|---:|---:|---:|
| Unitarias (Surefire) | 321 | 0 | 0 | 0 |
| Integración (Failsafe) | 83 | 0 | 0 | 0 |
| **Total** | **404** | **0** | **0** | **0** |

| Paquete de prueba | Pruebas |
|---|---:|
| `service.impl` | 154 |
| `security` | 45 |
| `controller` | 40 |
| `service` | 35 |
| `repository` | 23 |
| `exception` | 21 |
| `client.alert` | 16 |
| `client` | 12 |
| `client.campus` | 12 |
| `client.brigade` | 11 |
| `scheduler` | 10 |
| `client.checkin` | 10 |
| `client.audit` | 6 |
| `config` | 3 |
| `security.local` | 3 |
| `entity` | 3 |

## Cobertura

JaCoCo sobre `src/main/java`, sin la clase principal (`EventsApplication`), con las pruebas unitarias y de integración.

| Alcance | Instrucciones | Ramas | Líneas | Métodos |
|---|---:|---:|---:|---:|
| **Total** | **95,2 %** (10783/11328) | **81,0 %** (585/722) | **95,0 %** (2202/2318) | **95,8 %** (678/708) |
| `service.impl` | 98,1 % (4927/5021) | 89,4 % (328/367) | 98,5 % (968/983) | 98,0 % (194/198) |
| `security` | 99,0 % (877/886) | 83,3 % (85/102) | 100,0 % (183/183) | 100,0 % (56/56) |
| `client.alert` | 90,1 % (674/748) | 61,1 % (33/54) | 90,4 % (132/146) | 93,6 % (44/47) |
| `exception` | 97,8 % (621/635) | 89,5 % (17/19) | 98,2 % (161/164) | 97,8 % (45/46) |
| `client.checkin` | 95,7 % (538/562) | 65,4 % (17/26) | 95,7 % (88/92) | 100,0 % (29/29) |
| `client.brigade` | 95,5 % (489/512) | 66,7 % (12/18) | 96,7 % (87/90) | 100,0 % (30/30) |
| `entity` | 94,7 % (480/507) | 100,0 % (6/6) | 94,2 % (147/156) | 93,6 % (132/141) |
| `dto.response` | 100,0 % (383/383) | — (0/0) | 100,0 % (21/21) | 100,0 % (21/21) |
| `client.campus` | 93,5 % (362/387) | 74,0 % (37/50) | 93,2 % (82/88) | 100,0 % (17/17) |
| `config` | 55,1 % (199/361) | 0,0 % (0/6) | 54,7 % (47/86) | 66,7 % (20/30) |
| `security.local` | 100,0 % (196/196) | 100,0 % (8/8) | 100,0 % (41/41) | 100,0 % (10/10) |
| `client` | 66,9 % (188/281) | 42,5 % (17/40) | 65,7 % (44/67) | 82,4 % (14/17) |
| `service` | 100,0 % (185/185) | 100,0 % (14/14) | 100,0 % (32/32) | 100,0 % (16/16) |
| `client.audit` | 100,0 % (170/170) | 90,0 % (9/10) | 100,0 % (34/34) | 100,0 % (9/9) |
| `entity.enums` | 100,0 % (126/126) | — (0/0) | 100,0 % (24/24) | 100,0 % (6/6) |
| `mapper` | 100,0 % (121/121) | — (0/0) | 100,0 % (38/38) | 100,0 % (6/6) |
| `scheduler` | 100,0 % (112/112) | 100,0 % (2/2) | 100,0 % (47/47) | 100,0 % (13/13) |
| `controller` | 100,0 % (99/99) | — (0/0) | 100,0 % (23/23) | 100,0 % (13/13) |
| `dto.request` | 100,0 % (36/36) | — (0/0) | 100,0 % (3/3) | 100,0 % (3/3) |

Los paquetes son subpaquetes de `edu.escuelaing.campusseguro.events`. Los repositorios son interfaces de Spring Data sin código propio y no aparecen en la tabla.

## Análisis estático

| Herramienta | Versión | Hallazgos |
|---|---|---:|
| PMD | - | - |
| SpotBugs | - | - |

El `pom.xml` de MS06 no incluye PMD ni SpotBugs.

## Cómo ejecutar localmente

Requiere JDK 21 y Maven. Las pruebas `*IT` requieren Docker en ejecución para Testcontainers.

| Comando | Resultado |
|---|---|
| `mvn test` | Pruebas unitarias |
| `mvn verify` | Pruebas unitarias, de integración y cobertura con umbral |

| Reporte | Ruta |
|---|---|
| Resultados de Surefire | `target/surefire-reports/` |
| Resultados de Failsafe | `target/failsafe-reports/` |
| Cobertura | `target/site/jacoco/index.html`, `target/site/jacoco/jacoco.xml` |

## Integración continua

El workflow `ci.yml` de GitHub Actions ejecuta `mvn -B -ntp clean verify` en cada pull request a `main` o `develop` y en cada push a `main`, guarda el reporte de JaCoCo como artefacto (`ms06-jacoco`) y construye la imagen Docker, que solo se publica en GHCR desde `main`. El workflow no ejecuta SonarCloud.
