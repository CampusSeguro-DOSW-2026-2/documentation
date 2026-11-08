# Calidad y pruebas — API Gateway

Requisitos no funcionales verificables y pruebas de integración que deciden si el GW está listo para integrarse (Propuesta, *Definition of Done*). Los RF están en [01-requerimientos-funcionales.md](01-requerimientos-funcionales.md) y la configuración en [02-contratos-integracion.md](02-contratos-integracion.md) (GW-C).

## Requerimientos no funcionales

| ID | Categoría | Criterio de aceptación verificable |
|---|---|---|
| <a id="rnf-gw-01"></a>**RNF-GW-01** | Rendimiento | Con un backend sano y una carga de **50 solicitudes/s durante 60 s**, la latencia que agrega el GW es **≤ 50 ms en P95** y no hay 5xx atribuibles al GW. Script y resultado: `api-gateway/perf/`. |
| <a id="rnf-gw-02"></a>**RNF-GW-02** | Confiabilidad | Timeout `GATEWAY_DOWNSTREAM_TIMEOUT_MS`, `502` ante conexión fallida, `504` ante timeout, sin reintentos y con los recursos liberados al cancelar. |
| <a id="rnf-gw-03"></a>**RNF-GW-03** | Seguridad | Cero secretos en el repositorio; cero vulnerabilidades críticas en el Quality Gate; ninguna respuesta con detalles internos; solo el puerto `SERVER_PORT` publicado. En la nube solo Caddy publica puertos (`80`, `443`, `8443`) y los MS solo aceptan al GW por mTLS ([ADR-006](adr/ADR-006-aislamiento-en-la-nube.md)). |
| <a id="rnf-gw-04"></a>**RNF-GW-04** | Observabilidad | Logs con los campos de los Estándares; health en `≤ 200 ms`; métricas de Actuator disponibles solo en `MANAGEMENT_SERVER_PORT`. |
| <a id="rnf-gw-05"></a>**RNF-GW-05** | CI/CD | Java 21, Spring Boot y Maven; imagen Docker reproducible; GitHub Actions con el job de compilación y pruebas en **≤ 5 min**; cobertura de líneas **≥ 70 %** de todo el código (JaCoCo, excepto la clase principal); Quality Gate de SonarCloud aprobado. La imagen se publica en GHCR desde `develop` y `main` con la etiqueta `sha-xxxxxxx`; el despliegue a prueba es automático en cada push a `develop` y el de producción es manual desde `main`, con aprobación y la misma imagen. |
| <a id="rnf-gw-06"></a>**RNF-GW-06** | Configurabilidad | Cambiar rutas, destinos, JWT, CORS, claves o timeout solo exige configuración y reinicio. Una configuración inválida (incluidos el certificado de cliente y la CA del perfil `cloud`) impide el arranque con un mensaje claro. |
| <a id="rnf-gw-07"></a>**RNF-GW-07** | Contratos | Pruebas de contrato para las rutas públicas, cabeceras, JWT y `ErrorResponse`. Las decisiones de enrutamiento, JWT, CORS, errores y llamadas internas se documentan en ADR. |

## Pruebas de integración obligatorias

Se ejecutan a través del GW, con MS01 real o con un doble que cumpla su `swagger.yml`. En el repositorio del GW corren con el doble de MS01 y forman la suite `mvn test -Dgroups=integracion`.

| ID | Escenario | Resultado esperado | Prueba automatizada |
|---|---|---|---|
| INT-GW-01 | `POST /api/v1/auth/register` y `/login` sin JWT | Llegan a MS01 sin cambios; ni las credenciales ni el token aparecen en los logs. | `RoutingIntegrationTest`, `JwtIntegrationTest`, `LoggingIntegrationTest` |
| INT-GW-02 | Solicitud protegida con un JWT válido de MS01 | MS01 recibe `X-User-Id`, `X-User-Role` y `X-Correlation-ID`, sin `Authorization`. | `IdentityIntegrationTest` |
| INT-GW-03 | JWT ausente, expirado, alterado o con `iss`/`aud`/`alg` incorrectos | `401 NO_AUTENTICADO`; MS01 no recibe la solicitud. | `JwtIntegrationTest` |
| INT-GW-04 | Cliente envía `X-User-Id`, `X-User-Role` o `X-Service-Name` falsos | MS01 recibe solo la identidad del JWT. | `IdentityIntegrationTest` |
| INT-GW-05 | `PATCH /api/v1/users/status` con un JWT de `ESTUDIANTE` | El GW propaga la identidad y MS01 responde `403 PROHIBIDO` válido, que llega sin cambios. | `IdentityIntegrationTest` |
| INT-GW-06 | MS01 responde `400`, `401`, `403`, `404` o `409` con un `ErrorResponse` válido | Llega sin cambios, con `X-Correlation-ID`. | `ErrorPropagationIntegrationTest` |
| INT-GW-07 | El MS responde un error sin `correlation_id`, con otro valor o con el cuerpo por defecto de Spring | `502 SERVICIO_NO_DISPONIBLE`. | `ErrorPropagationIntegrationTest` |
| INT-GW-08 | Sin `X-Correlation-ID` o con un valor que no es UUID v4 | Se genera uno nuevo, se propaga y se devuelve. | `CorrelationIntegrationTest` |
| INT-GW-09 | El MS tarda más que `GATEWAY_DOWNSTREAM_TIMEOUT_MS` | `504 TIEMPO_AGOTADO`. | `DownstreamFailureIntegrationTest` |
| INT-GW-10 | Destino inaccesible | `502 SERVICIO_NO_DISPONIBLE` sin detalles internos. | `DownstreamFailureIntegrationTest` |
| INT-GW-11 | Llamada interna con una clave de servicio válida y `X-User-*` | El destino recibe `X-Service-Name` y la identidad, sin `X-Service-Key`. | `InternalCallsIntegrationTest`, `Ms01InternalRoutesIntegrationTest` |
| INT-GW-12 | Llamada interna con una clave inválida, o ruta interna sin clave | `401 NO_AUTENTICADO`; el destino no recibe nada. | `InternalCallsIntegrationTest`, `Ms01InternalRoutesIntegrationTest` |
| INT-GW-13 | Acceso directo desde fuera al puerto de un MS | Bloqueado por la red/despliegue. | Docker: `docker-compose.yml`; nube: firewall y mTLS (verificación manual del README, paso 10 del despliegue en la nube) |
| INT-GW-14 | `GET /actuator/health` | `200 {"status":"UP"}` en `≤ 200 ms`; `/actuator/metrics` en `SERVER_PORT` responde `404`. | `HealthIntegrationTest` |
| INT-GW-15 | Preflight CORS desde un origen permitido y uno no permitido | `200` con cabeceras CORS / `403`. | `CorsIntegrationTest` |
| INT-GW-16 | Perfil `cloud`: llamada a un MS por mTLS; MS con certificado de otra CA; MS que rechaza el certificado del GW; certificado o CA del GW inválidos | El MS recibe el certificado del GW / `502 SERVICIO_NO_DISPONIBLE` / `502` / el GW no arranca. | `CloudMtlsIntegrationTest`, `CloudProfileStartupTest` |

## Condición de cierre

El GW se considera integrable con MS01 cuando pasan INT-GW-01 a INT-GW-10 y los criterios de aceptación de RF-GW-01…07. INT-GW-11 y 12 se exigen desde que hay llamadas internas activas (MS03 y MS05 → MS01), INT-GW-13 a 15 al desplegar en Docker, e INT-GW-16 al desplegar en la nube.
