# Calidad y pruebas — MS01 Identidad y acceso

Estrategia de pruebas, cobertura y análisis estático de MS01 ([RNF-MS01-07](1_requirements.md#requerimientos-no-funcionales-rnf)). El comportamiento está en [1_requirements.md](1_requirements.md) y la seguridad en [security.md](security.md). Las cifras corresponden a la rama `develop` del repositorio `ms01-identity-and-access` (commit `2d5cdbe`).

## Estrategia de pruebas

Las pruebas no necesitan PostgreSQL, Docker, servidor de correo ni otros microservicios: cada dependencia externa se reemplaza por un doble, de modo que la suite corre igual en la máquina del desarrollador y en GitHub Actions.

| Nivel | Qué prueba | Cómo |
|---|---|---|
| Servicios (unitarias) | Reglas de negocio de registro, login, sesión, perfil, contraseña, usuarios, roles, catálogo de permisos, cortes de sesión y consultas internas | JUnit 5 y Mockito: los repositorios JPA, `JdbcTemplate`, el envío de correo, la hora de la base de datos y el publicador de auditoría son dobles. Se verifica el resultado, el error con su código y su mensaje, lo que se guarda y el evento de auditoría emitido |
| Validación de entradas | Cada campo de cada formulario ([2_forms-inputs.md](2_forms-inputs.md)), la normalización (recorte, mayúsculas, minúsculas) y los mensajes | Pruebas parametrizadas (`@ParameterizedTest`) sobre los validadores y los DTO de petición, incluido el rechazo de campos desconocidos (`StrictRequest`) |
| Controladores y contrato HTTP | Rutas, códigos de estado, cuerpo JSON, `ErrorResponse`, cabeceras `X-User-*` y `X-Correlation-ID` | `@WebMvcTest` con la cadena de Spring Security real (filtros de identidad y de servicio interno) y `MockMvc` en modo `standaloneSetup` para los controladores; los servicios son dobles de Mockito. `ObjectMapper` construye los cuerpos y lee las respuestas |
| Seguridad | Autenticación por cabeceras, rutas internas por `X-Service-Name`, autorización por permiso (403), 401 sin identidad, firma y claims del JWT, llaves RSA | Pruebas de la configuración de seguridad con `MockMvc`; `JwtServiceTest` genera un par de llaves RSA en memoria y verifica la firma y los claims del token con `java.security.Signature` del JDK |
| Persistencia y esquema | Correspondencia de las entidades con las tablas, orden de roles y permisos, restricciones de los scripts | Pruebas de entidades sin base de datos y lectura de los scripts `database/*.sql` para comprobar tablas, restricciones, triggers y la siembra de la Matriz (`RoleSchemaScriptsTest`, `PendingRegistrationsSchemaScriptsTest`, `PermissionMatrixTest`) |
| Integraciones salientes | Envío de eventos a MS08 a través del Gateway y envío de correos | `AuditEventPublisherTest` levanta un servidor HTTP local del JDK (`com.sun.net.httpserver`) que registra la petición; `EmailServiceTest` usa un doble de `JavaMailSender` |

### Tecnologías y por qué se usan

| Tecnología | Uso | Por qué |
|---|---|---|
| JUnit 5 (Jupiter) | Motor de pruebas y pruebas parametrizadas | Es el estándar de Spring Boot; `@ParameterizedTest` recorre todos los valores válidos e inválidos de un campo sin repetir código |
| Mockito | Dobles de repositorios, `JdbcTemplate`, correo, reloj de la base y auditoría | Aísla la regla de negocio y permite verificar qué se guardó y qué se emitió |
| Spring Boot Test, `@WebMvcTest` y `MockMvc` | Capa web con la seguridad real | Prueba el contrato HTTP (rutas, estados, `ErrorResponse`) sin levantar la aplicación completa ni la base de datos |
| Jackson `ObjectMapper` | Serializar peticiones y leer respuestas | Comprueba los nombres de los campos del contrato (snake_case en español) tal como viajan |
| Servidor HTTP del JDK (`com.sun.net.httpserver`) | Doble del Gateway en la emisión de auditoría | Registra la petición real que sale de MS01 sin dependencias externas |
| JaCoCo | Cobertura | Genera el `jacoco.xml` que lee SonarCloud |

## Resultados de la ejecución

Ejecución de `mvn verify` (Maven Surefire) sobre `develop`:

| Pruebas | Clases de prueba | Fallos | Errores | Omitidas |
|---:|---:|---:|---:|---:|
| 1195 | 78 | 0 | 0 | 0 |

Las clases con más pruebas son `PasswordServiceTest` (95), `RoleAdminServiceTest` (70), `PasswordRequestValidationTest` (52), `RoleRequestValidatorTest` (51), `RoleNamesTest` (46), `RoleServiceTest` (44), `ErrorCodeTest` (40), `RegistrationServiceTest` (40) y `UserCreateRequestValidationTest` (36).

## Cobertura

JaCoCo sobre todo el código de `src/main/java`.

| Alcance | Instrucciones | Ramas | Líneas | Métodos |
|---|---:|---:|---:|---:|
| **Total** | **97,0 %** (10292/10613) | **93,2 %** (634/680) | **96,7 %** (2346/2425) | **96,7 %** (582/602) |
| `identity_service` (clase principal) | 37,5 % (3/8) | — (0/0) | 33,3 % (1/3) | 50,0 % (1/2) |
| `audit` | 96,0 % (498/519) | 86,7 % (26/30) | 97,1 % (100/103) | 100,0 % (19/19) |
| `config` | 91,5 % (767/838) | 88,6 % (39/44) | 91,3 % (167/183) | 89,5 % (51/57) |
| `controller` | 100,0 % (337/337) | 100,0 % (4/4) | 100,0 % (77/77) | 100,0 % (35/35) |
| `dto` | 96,8 % (886/915) | 90,0 % (36/40) | 96,3 % (236/245) | 97,7 % (126/129) |
| `entity` | 95,9 % (464/484) | 80,0 % (8/10) | 95,1 % (156/164) | 93,5 % (86/92) |
| `exception` | 84,3 % (498/591) | 45,0 % (9/20) | 82,8 % (106/128) | 92,0 % (46/50) |
| `security` | 95,3 % (860/902) | 95,8 % (69/72) | 95,5 % (189/198) | 100,0 % (41/41) |
| `service` | 99,5 % (5237/5265) | 97,1 % (340/350) | 99,4 % (1136/1143) | 100,0 % (144/144) |
| `util` | 98,2 % (445/453) | 97,3 % (72/74) | 99,1 % (107/108) | 100,0 % (19/19) |
| `validation` | 98,7 % (297/301) | 86,1 % (31/36) | 97,3 % (71/73) | 100,0 % (14/14) |

Los paquetes son subpaquetes de `edu.eci.dosw.campusseguro.identity`. La clase principal solo contiene el `main` que arranca Spring, que las pruebas no ejecutan.

## Análisis estático

PMD (reglas de código sobre el fuente) y SpotBugs (errores sobre el bytecode, esfuerzo `Max`) corren en la fase `verify` y solo informan: nunca hacen fallar el build. El análisis que decide la integración es el Quality Gate de SonarCloud.

| Herramienta | Versión | Hallazgos antes de corregir | Hallazgos actuales |
|---|---|---:|---:|
| PMD (`maven-pmd-plugin` 3.28.0) | 7.17.0 | 0 | 0 |
| SpotBugs (`spotbugs-maven-plugin` 4.10.5.0, 153 clases) | 4.10.5 | 66 | 0 |

### Cómo se corrigieron

| Hallazgo | Cantidad | Corrección |
|---|---:|---|
| SpotBugs `MS_PKGPROTECT` | 1 | La lista de rutas internas de `InternalSecurityConfig` pasó a ser `private` |
| SpotBugs `CT_CONSTRUCTOR_THROW` | 1 | `JwtServiceImpl`, cuyo constructor valida la llave y puede lanzar una excepción, es `final`: ninguna subclase puede quedar a medio construir |
| SpotBugs `EI_EXPOSE_REP` / `EI_EXPOSE_REP2` | 64 | Copias defensivas en `ChangeRolesCommand` (con `LinkedHashSet`, que conserva el orden de los roles), `CreateRoleCommand` y `RolePermissionsCommand`; `RoleEntity.getPermissions()` y `UserEntity.getRoles()` devuelven vistas de solo lectura. Los demás casos son referencias compartidas por diseño y se excluyen con `spotbugs-exclude.xml` (tabla siguiente) |

Ninguna corrección cambió el contrato de la API ni el comportamiento hacia otros microservicios.

### Exclusiones de SpotBugs (`spotbugs-exclude.xml`)

Solo se excluyen `EI_EXPOSE_REP` y `EI_EXPOSE_REP2`, y solo donde compartir la referencia es el diseño. Cualquier otra clase que exponga un objeto mutable se sigue reportando.

| Patrón | Alcance | Motivo |
|---|---|---|
| `EI_EXPOSE_REP2` | Campos `userRepository`, `pendingRepository`, `permissionRepository`, `jdbcTemplate` y `roleAdminService` | Son beans que inyecta Spring (repositorios, `JdbcTemplate` y servicios): singletons compartidos por diseño que no se pueden copiar |
| `EI_EXPOSE_REP`, `EI_EXPOSE_REP2` | Paquete `dto` | DTO del contrato. Los cuerpos de petición extienden `StrictRequest` para reportar cada campo desconocido con su nombre, por lo que no pueden ser records con copias en el constructor sin cambiar los errores de validación del contrato; las respuestas se construyen y se serializan dentro de la misma petición y nadie conserva ni modifica sus listas |

## Cómo ejecutar localmente

Requiere JDK 21 y Maven; no requiere PostgreSQL, Docker ni otros servicios. El proyecto compila con JDK 21, la versión del `pom.xml` y de la imagen Docker; con JDK 27, Lombok 1.18.46 falla al compilar.

| Comando | Resultado |
|---|---|
| `mvn test` | Todas las pruebas |
| `mvn test -Dtest=RoleAdminServiceTest` | Una clase de prueba |
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

El workflow `ci.yml` de GitHub Actions ejecuta `mvn -B verify` en cada push y pull request, y después el análisis de SonarCloud con `sonar.qualitygate.wait=true`: SonarCloud toma la cobertura del `target/site/jacoco/jacoco.xml` de esa ejecución y, si el Quality Gate no aprueba, el job falla y el cambio no se integra. PMD y SpotBugs son la revisión local, previa a SonarCloud: detectan en la máquina del desarrollador lo que SonarCloud revisaría después, sin bloquear el build.
