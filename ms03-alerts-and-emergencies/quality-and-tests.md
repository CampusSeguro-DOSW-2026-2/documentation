# Calidad y pruebas — MS03 Alertas y emergencias

Estrategia de pruebas, cobertura y análisis estático de MS03. El comportamiento está en [1_requirements.md](1_requirements.md) y la seguridad en [security.md](security.md). Las cifras corresponden a la ejecución de GitHub Actions del workflow `ci.yml` sobre la rama `develop` del repositorio `ms03-alerts-and-emergencies` (commit `899709b`).

## Estrategia de pruebas

`mvn verify` ejecuta dos fases: las pruebas unitarias (Maven Surefire), que no necesitan base de datos ni otros servicios, y las pruebas de integración (Maven Failsafe, clases `*IT`). `AlertApiIntegrationIT` y `AlertMigrationIT` levantan PostgreSQL con Testcontainers y por eso requieren Docker.

| Nivel | Qué prueba | Cómo |
|---|---|---|
| Servicios (unitarias) | Ciclo de vida de la alerta, destinatarios, bandeja propia, confirmaciones, catálogos de tipos y estados, envío de notificaciones y auditoría con su outbox | JUnit 5 y Mockito: repositorios y clientes son dobles |
| Seguridad | Lectura y validación de `X-User-*`, permisos por operación, alcance de simulacro y respuestas 401/403 | Pruebas del paquete `security` |
| Controladores | Rutas, parámetros, códigos de estado, cuerpos y `ErrorResponse` | `MockMvc` con los servicios como dobles (`AlertControllerTest`, `AlertFilterControllerTest`, `GlobalExceptionHandlerTest`) |
| Clientes de otros servicios | Llamadas a MS01, MS02 y MS08 por el Gateway y a la API de Brevo: ruta, cabeceras, clave de servicio, lectura de la respuesta y errores | WireMock como servidor HTTP (`RestCatalogClientTest`, `RestCampusScopeClientTest`, `RestAuditClientTest`, `BrevoEmailSenderTest`) |
| Contrato | Que las rutas del código coinciden con el contrato OpenAPI del repositorio | `ApiContractTest` |
| Programación | Activación de alertas programadas, reintento de notificaciones y despacho de la auditoría | Pruebas del paquete `scheduler` |
| Integración | API completa sobre PostgreSQL, migraciones de Flyway y entrega de notificaciones | `AlertApiIntegrationIT` y `AlertMigrationIT` con Testcontainers (PostgreSQL), y `NotificationDeliveryIT` |

### Tecnologías y por qué se usan

| Tecnología | Uso | Por qué |
|---|---|---|
| JUnit 5 (Jupiter) | Motor de pruebas | Es el estándar de Spring Boot |
| Mockito | Dobles de repositorios y clientes | Aísla la regla de negocio |
| Spring Boot Test y `MockMvc` | Capa web | Prueba el contrato HTTP sin levantar un servidor |
| WireMock | Doble del Gateway y de Brevo | Registra la petición real que sale de MS03 |
| Testcontainers (PostgreSQL) | Base de datos real en las pruebas `*IT` | Verifica las migraciones y las consultas contra PostgreSQL |
| JaCoCo | Cobertura y umbral mínimo | La regla `check` exige al menos 80 % de líneas cubiertas; si no se cumple, `mvn verify` falla |
| SpotBugs | Análisis estático sobre el bytecode | Corre en `mvn verify` y hace fallar el build con un hallazgo |

## Resultados de la ejecución

| Fase | Pruebas | Fallos | Errores | Omitidas |
|---|---:|---:|---:|---:|
| Unitarias (Surefire) | 404 | 0 | 0 | 0 |
| Integración (Failsafe) | 28 | 0 | 0 | 0 |
| **Total** | **432** | **0** | **0** | **0** |

| Paquete de prueba | Pruebas |
|---|---:|
| `service.impl` | 91 |
| `security` | 83 |
| `service` | 47 |
| `client.email` | 35 |
| `client.catalog` | 32 |
| `integration` | 28 |
| `controller` | 25 |
| `mapper` | 23 |
| `exception` | 20 |
| `client.audit` | 10 |
| `scheduler` | 9 |
| `client.campus` | 9 |
| `validation` | 7 |
| `contract` | 6 |
| `config` | 5 |
| `event` | 2 |

## Cobertura

JaCoCo sobre todo el código de `src/main/java`, con las pruebas unitarias y de integración.

| Alcance | Instrucciones | Ramas | Líneas | Métodos |
|---|---:|---:|---:|---:|
| **Total** | **91,0 %** (11201/12305) | **75,1 %** (763/1016) | **91,6 %** (2742/2992) | **91,7 %** (626/683) |
| `service.impl` | 90,8 % (2345/2583) | 74,7 % (198/265) | 93,5 % (620/663) | 96,4 % (80/83) |
| `service` | 91,0 % (1891/2079) | 71,2 % (131/184) | 90,3 % (457/506) | 87,0 % (87/100) |
| `client.email` | 93,8 % (1499/1598) | 83,3 % (95/114) | 93,1 % (324/348) | 98,0 % (49/50) |
| `security` | 98,2 % (1022/1041) | 89,7 % (122/136) | 97,7 % (250/256) | 96,8 % (61/63) |
| `client.catalog` | 87,3 % (847/970) | 68,3 % (71/104) | 86,1 % (180/209) | 85,7 % (42/49) |
| `exception` | 90,6 % (642/709) | 33,3 % (5/15) | 88,0 % (176/200) | 88,3 % (53/60) |
| `mapper` | 94,8 % (596/629) | 71,4 % (20/28) | 97,2 % (171/176) | 93,8 % (30/32) |
| `entity` | 94,3 % (415/440) | 70,0 % (7/10) | 95,5 % (168/176) | 94,7 % (107/113) |
| `dto.response` | 89,7 % (296/330) | 50,0 % (3/6) | 83,3 % (20/24) | 86,7 % (13/15) |
| `config` | 85,9 % (268/312) | 0,0 % (0/2) | 85,9 % (67/78) | 81,8 % (18/22) |
| `client.audit` | 95,2 % (237/249) | 75,0 % (18/24) | 98,1 % (51/52) | 90,9 % (10/11) |
| `repository` | 100,0 % (246/246) | 95,5 % (21/22) | 100,0 % (53/53) | 100,0 % (19/19) |
| `client.campus` | 97,7 % (212/217) | 72,7 % (32/44) | 96,2 % (51/53) | 100,0 % (7/7) |
| `client.alerttype` | 6,6 % (12/181) | 0,0 % (0/18) | 5,9 % (2/34) | 28,6 % (2/7) |
| `controller` | 86,7 % (163/188) | 50,0 % (2/4) | 87,3 % (48/55) | 88,9 % (16/18) |
| `scheduler` | 97,0 % (162/167) | 87,5 % (7/8) | 97,9 % (47/48) | 100,0 % (9/9) |
| `dto.request` | 92,4 % (157/170) | 90,0 % (9/10) | 83,3 % (10/12) | 90,0 % (9/10) |
| `validation` | 100,0 % (86/86) | 100,0 % (22/22) | 100,0 % (28/28) | 100,0 % (6/6) |
| `entity.enums` | 100,0 % (66/66) | — (0/0) | 100,0 % (9/9) | 100,0 % (4/4) |
| `event` | 100,0 % (36/36) | — (0/0) | 100,0 % (9/9) | 100,0 % (3/3) |
| `alertas` (clase principal) | 37,5 % (3/8) | — (0/0) | 33,3 % (1/3) | 50,0 % (1/2) |

Los paquetes son subpaquetes de `edu.eci.dosw.campusseguro.alertas`.

## Análisis estático

| Herramienta | Versión | Hallazgos |
|---|---|---:|
| PMD | - | - |
| SpotBugs (`spotbugs-maven-plugin` 4.10.5.0, esfuerzo `Max`, umbral `Medium`) | 4.10.5 | 0 |

El `pom.xml` de MS03 no incluye PMD. SpotBugs hace fallar `mvn verify` si encuentra un hallazgo (`failOnError`).

### Exclusiones de SpotBugs (`config/spotbugs-exclude.xml`)

| Patrón | Alcance | Motivo |
|---|---|---|
| `EI_EXPOSE_REP`, `EI_EXPOSE_REP2` | Todo el proyecto | Los componentes de Spring, los DTO y los records exponen sus colecciones y colaboradores por diseño; ningún consumidor modifica el estado a través de esas referencias |
| `CT_CONSTRUCTOR_THROW` | Todo el proyecto | Los componentes de Spring validan su configuración en el constructor y fallan al arrancar; ninguna de esas clases define un finalizador, así que el ataque por finalizador que reporta SpotBugs no aplica |

## Cómo ejecutar localmente

Requiere JDK 21 y Maven. Las pruebas `*IT` con Testcontainers requieren Docker en ejecución.

| Comando | Resultado |
|---|---|
| `mvn test` | Pruebas unitarias |
| `mvn test -Dtest=AlertControllerTest` | Una clase de prueba |
| `mvn verify` | Pruebas unitarias y de integración, cobertura con umbral y SpotBugs |

| Reporte | Ruta |
|---|---|
| Resultados de Surefire | `target/surefire-reports/` |
| Resultados de Failsafe | `target/failsafe-reports/` |
| Cobertura | `target/site/jacoco/index.html`, `target/site/jacoco/jacoco.xml` |
| SpotBugs | `target/spotbugsXml.xml` |

## Integración continua

El workflow `ci.yml` de GitHub Actions ejecuta `mvn -B -ntp clean verify` en cada push y pull request, guarda los reportes de JaCoCo (`backend-jacoco`) y SpotBugs (`backend-spotbugs`) como artefactos y, desde `main` y `develop`, publica la imagen en GHCR. El workflow `sonarqube.yml` repite la verificación y, si el repositorio tiene `SONAR_TOKEN`, ejecuta el análisis de SonarQube.
