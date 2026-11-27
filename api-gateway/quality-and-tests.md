# Calidad y pruebas — API Gateway

Requisitos no funcionales verificables, estrategia de pruebas, cobertura, análisis estático y pruebas de integración que deciden si el GW está listo para integrarse (Propuesta, *Definition of Done*). Los RF están en [1_requirements.md](1_requirements.md), la configuración en [contracts.md](contracts.md) (GW-C) y la seguridad en [security.md](security.md). Las cifras corresponden a la rama `develop` del repositorio `api-gateway` (commit `b217347`).

## Requerimientos no funcionales

| ID | Categoría | Criterio de aceptación verificable |
|---|---|---|
| <a id="rnf-gw-01"></a>**RNF-GW-01** | Rendimiento | Con un backend sano y una carga de **50 solicitudes/s durante 60 s**, la latencia que agrega el GW es **≤ 50 ms en P95** y no hay 5xx atribuibles al GW. Script y resultado: `api-gateway/perf/`. |
| <a id="rnf-gw-02"></a>**RNF-GW-02** | Confiabilidad | Timeout `GATEWAY_DOWNSTREAM_TIMEOUT_MS`, `502` ante conexión fallida, `504` ante timeout, sin reintentos y con los recursos liberados al cancelar. La consulta de cortes de sesión no bloquea las solicitudes: si MS01 no responde, el GW conserva los últimos cortes y sigue operando. |
| <a id="rnf-gw-03"></a>**RNF-GW-03** | Seguridad | Cero secretos en el repositorio; cero vulnerabilidades críticas en el Quality Gate; ninguna respuesta con detalles internos; solo el puerto `SERVER_PORT` publicado. En la nube solo Caddy publica puertos (`80`, `443`, `8443`) y los MS solo aceptan al GW por mTLS ([aislamiento en la nube](contracts.md#7-aislamiento-en-la-nube)). |
| <a id="rnf-gw-04"></a>**RNF-GW-04** | Observabilidad | Logs con los campos de los Estándares; health en `≤ 200 ms`; métricas de Actuator disponibles solo en `MANAGEMENT_SERVER_PORT`. |
| <a id="rnf-gw-05"></a>**RNF-GW-05** | CI/CD | Java 21, Spring Boot y Maven; imagen Docker reproducible; GitHub Actions con el job de compilación y pruebas en **≤ 5 min**; cobertura de líneas **≥ 70 %** de todo el código (JaCoCo, excepto la clase principal); Quality Gate de SonarCloud aprobado. La imagen se publica en GHCR desde `develop` y `main` con la etiqueta `sha-xxxxxxx`; el despliegue a prueba es automático en cada push a `develop` y el de producción es manual desde `main`, con aprobación y la misma imagen. |
| <a id="rnf-gw-06"></a>**RNF-GW-06** | Configurabilidad | Cambiar rutas, destinos, JWT, cortes de sesión, CORS, claves o timeout solo exige configuración y reinicio. Una configuración inválida (incluidos el certificado de cliente y la CA del perfil `cloud`) impide el arranque con un mensaje claro. |
| <a id="rnf-gw-07"></a>**RNF-GW-07** | Contratos | Pruebas de contrato para las rutas públicas, cabeceras, JWT (con `roles` y `permissions`) y `ErrorResponse`. El enrutamiento, el JWT, CORS, los errores y las llamadas internas están descritos en [1_requirements.md](1_requirements.md) y [contracts.md](contracts.md). |

## Estrategia de pruebas

El GW no tiene base de datos ni lógica de dominio: su comportamiento es la cadena de filtros que recibe una petición, la valida y la reenvía. Por eso las pruebas se dividen en dos niveles:

| Nivel | Qué prueba | Cómo |
|---|---|---|
| Unitarias | Cada filtro y componente por separado: validación del JWT, política de acceso, clave de servicio, formato de la identidad, correlación, CORS, validación de los errores downstream, cortes de sesión, propiedades de configuración y formato del log | JUnit 5 y Mockito sobre la clase, con `MockServerWebExchange` de Spring para simular la petición y `StepVerifier` de Reactor para las cadenas reactivas. Las llaves RSA y los JWT de prueba se generan en memoria con Nimbus JOSE + JWT |
| Integración (etiqueta `integracion`) | El GW completo, de extremo a extremo: la petición entra por HTTP, pasa por todos los filtros y llega a un MS simulado | `@SpringBootTest` con el GW en un puerto aleatorio y `WebTestClient` como cliente. Cada MS de destino y la ruta de cortes de sesión de MS01 se reemplazan por un servidor HTTP local (`DownstreamStub`, sobre `com.sun.net.httpserver` del JDK) que registra lo que recibe; el mTLS del perfil `cloud` se prueba con un servidor HTTPS local que exige el certificado de cliente (`MtlsStub`) |

Las pruebas de integración comprueban lo que el MS de destino recibe (método, ruta, cuerpo, cabeceras de identidad, ausencia de `Authorization` y `X-Service-Key`) y lo que recibe el cliente (estado, `ErrorResponse` y `X-Correlation-ID`). No necesitan Docker ni MS reales.

### Tecnologías y por qué se usan

| Tecnología | Uso | Por qué |
|---|---|---|
| JUnit 5 (Jupiter) | Motor de pruebas; `@ParameterizedTest` para recorrer variantes de un mismo caso (formatos de token, cabeceras, rutas) | Es el estándar de Spring Boot y permite agrupar por etiquetas (`integracion`) |
| Mockito | Dobles de los componentes que recibe cada filtro | Aísla la unidad probada sin levantar el contexto de Spring |
| Spring Boot Test y `WebTestClient` | Contexto completo del GW y cliente HTTP reactivo | Prueba la cadena real de Spring Cloud Gateway (WebFlux), con su orden de filtros |
| Reactor Test (`StepVerifier`) | Verificación de `Mono` y `Flux` | Los filtros del GW son reactivos y su resultado es asíncrono |
| Servidor HTTP del JDK (`com.sun.net.httpserver`) | Dobles de MS01…MS08 y de MS01 en la consulta de cortes | Responde lo que cada prueba necesita y registra la petición recibida, sin dependencias externas ni Docker |
| Nimbus JOSE + JWT | Firma de JWT de prueba con llaves RSA generadas en memoria | Es la misma librería con la que el GW valida el token |
| JaCoCo | Cobertura | Genera el `jacoco.xml` que lee SonarCloud y exige un mínimo de cobertura en el build |
| k6 | Prueba de carga de RNF-GW-01 (`perf/load-test.js`) | Mide la latencia que agrega el GW frente a un doble de MS01 con latencia fija |

## Resultados de la ejecución

Ejecución de `mvn verify` (Maven Surefire) sobre `develop`:

| Pruebas | Clases de prueba | Fallos | Errores | Omitidas |
|---:|---:|---:|---:|---:|
| 419 | 40 | 0 | 0 | 0 |

De ellas, 103 son de integración (15 clases) y 316 unitarias (25 clases).

| Paquete de prueba | Pruebas |
|---|---:|
| `security` | 213 |
| `sessioncuts` | 65 |
| `error` | 51 |
| `routing` | 21 |
| `cors` | 20 |
| `config` | 18 |
| `correlation` | 14 |
| `logging` | 13 |
| `health` | 3 |
| `gateway_service` (arranque del contexto) | 1 |

La prueba de carga de RNF-GW-01 (`perf/results/rnf-gw-01.txt`) cumple el criterio: con 50 solicitudes/s durante 60 s a una ruta protegida, la latencia que agrega el GW es de 4,5 ms en P95 y no hay respuestas 5xx del GW.

## Cobertura

JaCoCo, sin la clase principal (`GatewayServiceApplication`). El build falla si la cobertura de líneas baja de 70 % (`jacoco.minimum.coverage`).

| Alcance | Instrucciones | Ramas | Líneas | Métodos |
|---|---:|---:|---:|---:|
| **Total** | **99,1 %** (3261/3292) | **91,1 %** (337/370) | **98,7 %** (603/611) | **99,5 %** (206/207) |
| `config` | 98,5 % (268/272) | 81,2 % (52/64) | 100,0 % (29/29) | 100,0 % (18/18) |
| `correlation` | 100,0 % (105/105) | 100,0 % (4/4) | 100,0 % (20/20) | 100,0 % (12/12) |
| `cors` | 100,0 % (186/186) | 81,8 % (18/22) | 100,0 % (39/39) | 100,0 % (13/13) |
| `error` | 98,8 % (741/750) | 95,5 % (63/66) | 99,3 % (144/145) | 97,8 % (45/46) |
| `health` | 93,2 % (41/44) | 50,0 % (2/4) | 100,0 % (8/8) | 100,0 % (5/5) |
| `logging` | 99,5 % (217/218) | 100,0 % (12/12) | 97,8 % (45/46) | 100,0 % (19/19) |
| `routing` | 100,0 % (200/200) | 92,9 % (13/14) | 100,0 % (30/30) | 100,0 % (11/11) |
| `security` | 98,7 % (1102/1116) | 94,6 % (123/130) | 97,2 % (205/211) | 100,0 % (61/61) |
| `sessioncuts` | 100,0 % (401/401) | 92,6 % (50/54) | 100,0 % (83/83) | 100,0 % (22/22) |

Los paquetes son subpaquetes de `edu.eci.dosw.campusseguro.gateway.gateway_service`.

## Análisis estático

PMD (reglas de código sobre el fuente) y SpotBugs (errores sobre el bytecode, esfuerzo `Max`) corren en la fase `verify` y solo informan: nunca hacen fallar el build. El análisis que decide la integración es el Quality Gate de SonarCloud.

| Herramienta | Versión | Hallazgos antes de corregir | Hallazgos actuales |
|---|---|---:|---:|
| PMD (`maven-pmd-plugin` 3.28.0) | 7.17.0 | 4 | 0 |
| SpotBugs (`spotbugs-maven-plugin` 4.10.5.0, 55 clases) | 4.10.5 | 30 | 0 |

### Cómo se corrigieron

| Hallazgo | Cantidad | Corrección |
|---|---:|---|
| PMD `UnnecessaryFullyQualifiedName` (prioridad 4) | 3 | Los filtros usaban `Ordered.HIGHEST_PRECEDENCE` dentro de clases que ya implementan `Ordered`; ahora usan `HIGHEST_PRECEDENCE` |
| PMD `EmptyCatchBlock` (prioridad 3) | 1 | El `catch` vacío de `AccessLogFilter` (el log nunca bloquea la respuesta) nombra su excepción `ignored`, la convención que PMD reconoce para una excepción ignorada a propósito |
| SpotBugs `EI_EXPOSE_REP` / `EI_EXPOSE_REP2` | 30 | `SessionCutSnapshot` guarda y devuelve copias inmodificables de sus mapas. Los demás casos son referencias compartidas por diseño y se excluyen con `spotbugs-exclude.xml` (tabla siguiente) |

### Exclusiones de SpotBugs (`spotbugs-exclude.xml`)

Solo se excluyen `EI_EXPOSE_REP` y `EI_EXPOSE_REP2`, y solo en las clases donde compartir la referencia es el diseño. Cualquier otra clase que exponga un objeto mutable se sigue reportando.

| Patrón | Clases y campos | Motivo |
|---|---|---|
| `EI_EXPOSE_REP2` | `DownstreamErrorValidationFilter.writer`, `DownstreamErrorValidator.jsonMapper`, `ErrorResponseWriter.jsonMapper`, `GatewayErrorWebExceptionHandler.resolvers` y `.writer`, `SessionCutsPoller.store` | Son beans que inyecta Spring: singletons compartidos por diseño. Copiarlos rompería el estado compartido (por ejemplo, el almacén de cortes de sesión que el filtro del JWT lee y el sondeo actualiza) |
| `EI_EXPOSE_REP`, `EI_EXPOSE_REP2` | `ErrorResponse`, `SessionCutsResponse` | DTO del contrato: records puros sin constructores, que se construyen y se serializan dentro de la misma petición; nadie conserva ni modifica sus listas |
| `EI_EXPOSE_REP`, `EI_EXPOSE_REP2` | `GatewayRoutesProperties`, `GatewayRoutesProperties$Route`, `CorsProperties`, `AccessPolicyProperties`, `ServiceKeyProperties` | Records de `@ConfigurationProperties`: se enlazan una sola vez al arrancar y se validan con Bean Validation, que necesita los valores recibidos (incluido `null`) para reportar sus propios mensajes. Las clases que los consumen copian las listas (por ejemplo, `AccessPolicyResolver` usa `List.copyOf`) |

## Cómo ejecutar localmente

Requiere JDK 21 y Maven; no requiere Docker ni MS reales.

| Comando | Resultado |
|---|---|
| `mvn test` | Todas las pruebas (unitarias e integración) |
| `mvn test -Dgroups=integracion` | Solo las pruebas de integración |
| `mvn verify` | Pruebas, cobertura con su mínimo, PMD y SpotBugs |
| `mvn spotbugs:gui` | Navegador gráfico de los hallazgos de SpotBugs, después de `mvn verify` |

Reportes que deja `mvn verify`:

| Reporte | Ruta |
|---|---|
| Resultados de Surefire | `target/surefire-reports/` |
| Cobertura (HTML y XML para SonarCloud) | `target/site/jacoco/index.html`, `target/site/jacoco/jacoco.xml` |
| PMD | `target/pmd.xml` |
| SpotBugs | `target/spotbugsXml.xml` |

La prueba de carga se ejecuta con k6 sobre el GW y su doble de MS01 según `perf/load-test.js`.

## Relación con SonarCloud

El workflow `ci.yml` de GitHub Actions ejecuta `mvn -B verify` en cada push y pull request, y después el análisis de SonarCloud con el `jacoco.xml` de esa ejecución y `sonar.qualitygate.wait=true`: si el Quality Gate no aprueba, el job falla y el cambio no se integra (RNF-GW-03, RNF-GW-05). PMD y SpotBugs son la revisión local, previa a SonarCloud: detectan en la máquina del desarrollador lo que SonarCloud revisaría después, sin bloquear el build.

## Pruebas de integración obligatorias

Se ejecutan a través del GW, con MS01 real o con un doble que cumpla su `openapi.yml`. En el repositorio del GW corren con el doble de MS01 y forman la suite `mvn test -Dgroups=integracion`.

| ID | Escenario | Resultado esperado | Prueba automatizada |
|---|---|---|---|
| INT-GW-01 | `POST /api/v1/auth/register`, `/register/verify`, `/register/resend` y `/login` sin JWT | Llegan a MS01 sin cambios; ni las credenciales ni el token aparecen en los logs. | `RoutingIntegrationTest`, `JwtIntegrationTest`, `LoggingIntegrationTest` |
| INT-GW-02 | Solicitud protegida con un JWT válido de MS01 | MS01 recibe `X-User-Id`, `X-User-Role`, `X-User-Permission` y `X-Correlation-ID`, sin `Authorization`. | `IdentityIntegrationTest` |
| INT-GW-03 | JWT ausente, expirado, alterado o con `iss`/`aud`/`alg` incorrectos, o sin `permissions` | `401 NO_AUTENTICADO`; MS01 no recibe la solicitud. | `JwtIntegrationTest` |
| INT-GW-04 | Cliente envía `X-User-Id`, `X-User-Role`, `X-User-Permission` o `X-Service-Name` falsos | MS01 recibe solo la identidad del JWT. | `IdentityIntegrationTest` |
| INT-GW-05 | `PATCH /api/v1/users/status` con un JWT de `ESTUDIANTE` | El GW propaga la identidad y MS01 responde `403 PROHIBIDO` válido, que llega sin cambios. | `IdentityIntegrationTest` |
| INT-GW-06 | MS01 responde `400`, `401`, `403`, `404` o `409` con un `ErrorResponse` válido | Llega sin cambios, con `X-Correlation-ID`. | `ErrorPropagationIntegrationTest` |
| INT-GW-07 | El MS responde un error sin `correlation_id`, con otro valor o con el cuerpo por defecto de Spring | `502 SERVICIO_NO_DISPONIBLE`. | `ErrorPropagationIntegrationTest` |
| INT-GW-08 | Sin `X-Correlation-ID` o con un valor que no es UUID v4 | Se genera uno nuevo, se propaga y se devuelve. | `CorrelationIntegrationTest` |
| INT-GW-09 | El MS tarda más que `GATEWAY_DOWNSTREAM_TIMEOUT_MS` | `504 TIEMPO_AGOTADO`. | `DownstreamFailureIntegrationTest` |
| INT-GW-10 | Destino inaccesible | `502 SERVICIO_NO_DISPONIBLE` sin detalles internos. | `DownstreamFailureIntegrationTest` |
| INT-GW-11 | Llamada interna con una clave de servicio válida, con las tres `X-User-*` o sin ninguna | El destino recibe `X-Service-Name` y, si vino, la identidad, sin `X-Service-Key`. | `InternalCallsIntegrationTest`, `Ms01InternalRoutesIntegrationTest`, `RoutingIntegrationTest` (`GET /api/v1/tipos-alerta` con la clave de MS03) |
| INT-GW-12 | Llamada interna con una clave inválida, o ruta interna sin clave | `401 NO_AUTENTICADO`; el destino no recibe nada. | `InternalCallsIntegrationTest`, `Ms01InternalRoutesIntegrationTest`, `RoutingIntegrationTest` |
| INT-GW-13 | Acceso directo desde fuera al puerto de un MS | Bloqueado por la red/despliegue. | Docker: `docker-compose.yml`; nube: firewall y mTLS (verificación manual del README, paso 10 del despliegue en la nube) |
| INT-GW-14 | `GET /actuator/health` | `200 {"status":"UP"}` en `≤ 200 ms`; `/actuator/metrics` en `SERVER_PORT` responde `404`. | `HealthIntegrationTest` |
| INT-GW-15 | Preflight CORS desde un origen permitido, uno no permitido y uno que pide `X-User-Permission`, `X-User-Role` o `X-Service-Key` | `200` con cabeceras CORS / `403` / `403`. | `CorsIntegrationTest` |
| INT-GW-16 | Perfil `cloud`: llamada a un MS por mTLS; MS con certificado de otra CA; MS que rechaza el certificado del GW; certificado o CA del GW inválidos | El MS recibe el certificado del GW / `502 SERVICIO_NO_DISPONIBLE` / `502` / el GW no arranca. | `CloudMtlsIntegrationTest`, `CloudProfileStartupTest` |
| INT-GW-17 | `/api/v1/roles`, `/api/v1/roles/**` y `/api/v1/permissions`; `GET /api/v1/internal/session-cuts` | Las primeras llegan a MS01 con el método, la ruta y el cuerpo sin cambios; la ruta interna responde `404 NO_ENCONTRADO` sin llamar a MS01. | `RoutingIntegrationTest` |
| INT-GW-18 | Token emitido antes del corte del usuario o de uno de sus roles; token emitido en el segundo del corte o después; cortes nuevos en MS01; MS01 sin responder o con un cuerpo inválido | `401 NO_AUTENTICADO` genérico / se acepta / el GW aplica la instantánea nueva en el siguiente ciclo de consulta / conserva los últimos cortes, registra un `WARN` y no bloquea las solicitudes. | `SessionCutsIntegrationTest`, `JwtValidatorTest`, `SessionCutsClientTest`, `SessionCutsPollerTest` |
| INT-GW-19 | Llamada interna con la identidad incompleta, repetida o con formato inválido (`X-User-Id`, `X-User-Role`, `X-User-Permission`) | `400 VALIDACION`; el destino no recibe nada. | `InternalCallsIntegrationTest`, `AccessControlFilterTest` |

## Condición de cierre

El GW se considera integrable con MS01 cuando pasan INT-GW-01 a INT-GW-10, INT-GW-17 e INT-GW-18 y los criterios de aceptación de RF-GW-01…07. INT-GW-11, 12 y 19 se exigen desde que hay llamadas internas activas (MS03 y MS05 → MS01), INT-GW-13 a 15 al desplegar en Docker, e INT-GW-16 al desplegar en la nube.
