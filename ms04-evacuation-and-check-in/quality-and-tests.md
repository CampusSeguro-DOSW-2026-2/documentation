# Calidad y pruebas — MS04 Evacuación y check-in

Estrategia de pruebas, cobertura y análisis estático de MS04 ([RNF-MS04-14](1_requirements.md#requerimientos-no-funcionales-rnf)). El comportamiento está en [1_requirements.md](1_requirements.md) y la seguridad en [security.md](security.md). Las cifras corresponden a la rama `develop` del repositorio `ms04-evacuation-and-check-in` (commit `8a52df9`).

## Estrategia de pruebas

Las pruebas de `mvn verify` no necesitan PostgreSQL, Docker ni otros microservicios: la base de datos, el Gateway y MS01, MS02, MS05 y MS08 se reemplazan por dobles. La concurrencia y los tiempos de respuesta, que sí dependen de PostgreSQL real, se verifican aparte con la prueba de carga ([Rendimiento](#rendimiento-y-concurrencia)).

| Nivel | Qué prueba | Cómo |
|---|---|---|
| Servicios (unitarias) | Recepción del estado del evento y sus transiciones, habilitación del check-in, registro del check-in con sus validaciones, estado propio, consolidado, personas, pendientes, asistencia, histórico, resultados, consulta de eventos, solicitudes de asistencia a MS05 y su reenvío periódico | JUnit 5 y Mockito: los repositorios JPA y los clientes de MS01, MS02 y MS05 son dobles. Se verifica el resultado, el error con su código y su mensaje, lo que se guarda y el evento de auditoría emitido |
| Validación de entradas | Cada campo de los formularios ([2_forms-inputs.md](2_forms-inputs.md)) y del cuerpo que envía MS06, con sus mensajes y la normalización | Pruebas parametrizadas (`@ParameterizedTest`) sobre `CheckinValidator` y `EstadoEventoValidator` |
| Controladores y contrato HTTP | Rutas, parámetros, códigos de estado, cuerpos JSON, `ErrorResponse`, cabeceras `X-User-*`, `X-Service-Name` y `X-Correlation-ID`, autorización por permiso | `@WebMvcTest` con las dos cadenas de Spring Security reales (identidad de usuario y rutas internas de MS06) y `MockMvc`; los servicios son dobles. `ObjectMapper` construye los cuerpos y lee las respuestas (`ContractHttpTest`, `DashboardHttpTest`, `EstadoPropioHttpTest`, `HistorialHttpTest`, `SecurityHttpTest`) |
| Clientes de otros microservicios | Llamadas a MS01, MS02 y MS05 a través del Gateway: ruta, método, cabeceras propagadas, clave de servicio, lectura de la respuesta y manejo de errores y tiempos agotados | Un servidor HTTP local del JDK (`com.sun.net.httpserver`) hace de Gateway, registra la petición recibida y responde lo que cada prueba necesita |
| Persistencia sin base de datos | Consultas de los repositorios, entidades y scripts del esquema | `CheckinRepositoryTest` y `EventoRepositoryTest` prueban los métodos por defecto que arman los parámetros de las consultas nativas y las declaraciones de las consultas; `EntitiesTest` prueba las entidades; `SchemaScriptsTest` lee los scripts `database/*.sql` y comprueba tablas, restricciones, la vista de estados vigentes y los triggers que impiden modificar o borrar check-ins |
| Arranque de la aplicación | Que el contexto de Spring arranca con su configuración y sus beans | `ApplicationTest` usa `@SpringBootTest` sin la autoconfiguración de `DataSource`, Hibernate ni repositorios JPA, con los repositorios y el `PlatformTransactionManager` como dobles; `ApplicationConfigurationTest` verifica los valores de `application.yaml` |
| Auditoría | Formato de los eventos que se emiten a MS08 y su envío asíncrono a través del Gateway | `AuditRecorderTest` y `AuditEventPublisherTest`, con el servidor HTTP local como Gateway |

### Tecnologías y por qué se usan

| Tecnología | Uso | Por qué |
|---|---|---|
| JUnit 5 (Jupiter) | Motor de pruebas y pruebas parametrizadas | Es el estándar de Spring Boot; `@ParameterizedTest` recorre todos los valores válidos e inválidos de un campo sin repetir código |
| Mockito | Dobles de repositorios, clientes HTTP y publicador de auditoría | Aísla la regla de negocio y permite verificar qué se guardó, qué se envió y qué se emitió |
| Spring Boot Test, `@WebMvcTest` y `MockMvc` | Capa web con la seguridad real | Prueba el contrato HTTP sin levantar la aplicación completa ni la base de datos |
| Jackson `ObjectMapper` | Serializar peticiones y leer respuestas | Comprueba los nombres de los campos del contrato (snake_case, y camelCase en las rutas de MS06) tal como viajan |
| Servidor HTTP del JDK (`com.sun.net.httpserver`) | Doble del Gateway para los clientes de MS01, MS02, MS05 y MS08 | Registra la petición real que sale de MS04 sin dependencias externas ni Docker |
| JaCoCo | Cobertura | Genera el `jacoco.xml` que lee SonarCloud |
| k6, WireMock y Docker Compose | Prueba de carga con MS04 y PostgreSQL reales | Miden tiempos y concurrencia, que dependen de la base real y no se pueden verificar con dobles |

## Resultados de la ejecución

Ejecución de `mvn verify` (Maven Surefire) sobre `develop`:

| Pruebas | Clases de prueba | Fallos | Errores | Omitidas |
|---:|---:|---:|---:|---:|
| 602 | 42 | 0 | 0 | 0 |

| Paquete de prueba | Pruebas |
|---|---:|
| `service` | 231 |
| `controller` | 186 |
| `client` | 64 |
| `security` | 29 |
| `repository` | 27 |
| `evacuation_service` (arranque y configuración) | 17 |
| `exception` | 17 |
| `audit` | 14 |
| `entity` | 9 |
| `util` | 6 |
| `config` | 2 |

## Cobertura

JaCoCo sobre todo el código de `src/main/java`.

| Alcance | Instrucciones | Ramas | Líneas | Métodos |
|---|---:|---:|---:|---:|
| **Total** | **97,2 %** (8139/8377) | **88,6 %** (584/659) | **97,5 %** (1686/1729) | **98,3 %** (412/419) |
| `evacuation_service` (clase principal) | 37,5 % (3/8) | — (0/0) | 33,3 % (1/3) | 50,0 % (1/2) |
| `audit` | 93,3 % (611/655) | 85,7 % (48/56) | 95,6 % (109/114) | 100,0 % (23/23) |
| `client` | 98,9 % (1376/1392) | 82,8 % (111/134) | 98,1 % (303/309) | 100,0 % (53/53) |
| `config` | 93,4 % (270/289) | 50,0 % (1/2) | 98,5 % (64/65) | 100,0 % (30/30) |
| `controller` | 100,0 % (274/274) | 100,0 % (10/10) | 100,0 % (76/76) | 100,0 % (23/23) |
| `dto` | 98,3 % (536/545) | 50,0 % (5/10) | 97,7 % (42/43) | 100,0 % (30/30) |
| `entity` | 93,7 % (399/426) | 100,0 % (14/14) | 95,2 % (119/125) | 95,2 % (60/63) |
| `exception` | 89,6 % (576/643) | 60,9 % (28/46) | 88,8 % (103/116) | 94,6 % (35/37) |
| `repository` | 100,0 % (113/113) | 100,0 % (12/12) | 100,0 % (26/26) | 100,0 % (8/8) |
| `security` | 95,3 % (591/620) | 86,8 % (33/38) | 96,5 % (138/143) | 100,0 % (37/37) |
| `service` | 99,7 % (3059/3068) | 96,8 % (272/281) | 99,7 % (627/629) | 99,0 % (102/103) |
| `util` | 96,2 % (331/344) | 89,3 % (50/56) | 97,5 % (78/80) | 100,0 % (10/10) |

Los paquetes son subpaquetes de `edu.eci.dosw.campusseguro.evacuation`. La clase principal solo contiene el `main` que arranca Spring, que las pruebas no ejecutan.

## Análisis estático

PMD (reglas de código sobre el fuente) y SpotBugs (errores sobre el bytecode, esfuerzo `Max`) corren en la fase `verify` y solo informan: nunca hacen fallar el build. El análisis que decide la integración es el Quality Gate de SonarCloud.

| Herramienta | Versión | Hallazgos antes de corregir | Hallazgos actuales |
|---|---|---:|---:|
| PMD (`maven-pmd-plugin` 3.28.0) | 7.17.0 | 1 | 0 |
| SpotBugs (`spotbugs-maven-plugin` 4.10.5.0, 126 clases) | 4.10.5 | 39 | 0 |

### Cómo se corrigieron

| Hallazgo | Cantidad | Corrección |
|---|---:|---|
| PMD `EmptyCatchBlock` (prioridad 3) | 1 | El `catch` vacío de `ValidationErrors.jsonName` (un campo anidado o heredado, que no está declarado en la clase, conserva su propio nombre en el error) nombra su excepción `ignored`, la convención que PMD reconoce para una excepción ignorada a propósito |
| SpotBugs `EI_EXPOSE_REP` / `EI_EXPOSE_REP2` | 39 | Copias defensivas en `ZonasAsignadas` y en `EstadoEventoValidator.EstadoValidado`; los getters de listas de la entidad `Evento` devuelven vistas de solo lectura. Los demás casos son referencias compartidas por diseño y se excluyen con `spotbugs-exclude.xml` (tabla siguiente) |

Ninguna corrección cambió el contrato de la API ni el comportamiento hacia otros microservicios.

### Exclusiones de SpotBugs (`spotbugs-exclude.xml`)

Solo se excluyen `EI_EXPOSE_REP` y `EI_EXPOSE_REP2`, y solo donde compartir la referencia es el diseño. Cualquier otra clase que exponga un objeto mutable se sigue reportando.

| Patrón | Alcance | Motivo |
|---|---|---|
| `EI_EXPOSE_REP2` | `Ms01PoblacionObjetivoClient.jsonMapper` | Bean que inyecta Spring: singleton compartido por diseño que no se puede copiar |
| `EI_EXPOSE_REP`, `EI_EXPOSE_REP2` | Paquete `dto` | DTO del contrato: records puros sin constructores, que se construyen y se serializan dentro de la misma petición; nadie conserva ni modifica sus listas |
| `EI_EXPOSE_REP`, `EI_EXPOSE_REP2` | `SolicitudAsistenciaService$EnvioSolicitud`, `SolicitudAsistenciaService$RegistroSolicitud` | Resultados internos que llevan la entidad JPA de la solicitud de asistencia, que es mutable por diseño: JPA la sigue para guardar su estado de envío en la misma transacción |
| `EI_EXPOSE_REP` | `AuditContext.atributos` | El campo ya es una copia de solo lectura (`Collections.unmodifiableMap` sobre un `LinkedHashMap` nuevo, que conserva el orden de los atributos que se envían a MS08); SpotBugs no reconoce esa copia |

## Rendimiento y concurrencia

La carpeta `rendimiento/` del repositorio levanta MS04 con PostgreSQL real y WireMock como Gateway (MS01, MS02, MS05 y MS08 con 20 a 120 ms de latencia) y ejecuta k6 con tres escenarios: 500 usuarios que reportan y cambian su estado a la vez, 20 reportes simultáneos del mismo usuario y la medición del tiempo hasta que el consolidado cuenta un check-in. Verifica [RNF-MS04-01](1_requirements.md#requerimientos-no-funcionales-rnf) (80 % de los check-ins en menos de 5 s), RNF-MS04-02 (500 check-ins simultáneos sin pérdidas ni duplicados), RNF-MS04-04 (reportes simultáneos del mismo usuario serializados) y RNF-MS04-06 (reflejo en el consolidado en menos de 5 s); k6 termina con un código distinto de 0 si algún umbral falla. RNF-MS04-07 (reporte en máximo 3 interacciones) se verifica con la prueba de usabilidad manual de `rendimiento/prueba-usabilidad.md`. Los comandos están en `rendimiento/README.md`.

## Cómo ejecutar localmente

Requiere JDK 21 y Maven; las pruebas de `mvn verify` no requieren PostgreSQL, Docker ni otros servicios.

| Comando | Resultado |
|---|---|
| `mvn test` | Todas las pruebas |
| `mvn test -Dtest=DashboardServiceTest` | Una clase de prueba |
| `mvn verify` | Pruebas, cobertura, PMD y SpotBugs |
| `mvn spotbugs:gui` | Navegador gráfico de los hallazgos de SpotBugs, después de `mvn verify` |

Reportes que deja `mvn verify`:

| Reporte | Ruta |
|---|---|
| Resultados de Surefire | `target/surefire-reports/` |
| Cobertura (HTML y XML para SonarCloud) | `target/site/jacoco/index.html`, `target/site/jacoco/jacoco.xml` |
| PMD | `target/pmd.xml` |
| SpotBugs | `target/spotbugsXml.xml` |

## Relación con SonarCloud

El workflow `ci.yml` de GitHub Actions ejecuta `mvn -B verify` en cada push y pull request, y después el análisis de SonarCloud con el `jacoco.xml` de esa ejecución y `sonar.qualitygate.wait=true`: si el Quality Gate no aprueba, el job falla y el cambio no se integra. PMD y SpotBugs son la revisión local, previa a SonarCloud: detectan en la máquina del desarrollador lo que SonarCloud revisaría después, sin bloquear el build.
