# Calidad y pruebas — API Gateway

Requisitos no funcionales verificables y pruebas de integración que deciden si el GW está listo para integrarse (Propuesta, *Definition of Done*). Los RF están en [01-requerimientos-funcionales.md](01-requerimientos-funcionales.md) y la configuración en [02-contratos-integracion.md](02-contratos-integracion.md) (GW-C).

## Requerimientos no funcionales

| ID | Categoría | Criterio de aceptación verificable |
|---|---|---|
| <a id="rnf-gw-01"></a>**RNF-GW-01** | Rendimiento | Con un backend sano y una carga de **50 solicitudes/s durante 60 s**, la latencia que agrega el GW es **≤ 50 ms en P95** y no hay 5xx atribuibles al GW. |
| <a id="rnf-gw-02"></a>**RNF-GW-02** | Confiabilidad | Timeout `GATEWAY_DOWNSTREAM_TIMEOUT_MS`, `502` ante conexión fallida, `504` ante timeout, sin reintentos y con los recursos liberados al cancelar. |
| <a id="rnf-gw-03"></a>**RNF-GW-03** | Seguridad | Cero secretos en el repositorio; cero vulnerabilidades críticas en el Quality Gate; ninguna respuesta con detalles internos; solo el puerto `SERVER_PORT` publicado. |
| <a id="rnf-gw-04"></a>**RNF-GW-04** | Observabilidad | Logs con los campos de los Estándares; health en `≤ 200 ms`; métricas de Actuator disponibles solo en `MANAGEMENT_SERVER_PORT`. |
| <a id="rnf-gw-05"></a>**RNF-GW-05** | CI/CD | Java 21, Spring Boot y Maven; imagen Docker reproducible; GitHub Actions con un pipeline de **≤ 5 min**; cobertura **≥ 70 %** en filtros, configuración y manejo de errores; Quality Gate de SonarCloud aprobado. |
| <a id="rnf-gw-06"></a>**RNF-GW-06** | Configurabilidad | Cambiar rutas, destinos, JWT, CORS, claves o timeout solo exige configuración y reinicio. Una configuración inválida impide el arranque con un mensaje claro. |
| <a id="rnf-gw-07"></a>**RNF-GW-07** | Contratos | Pruebas de contrato para las rutas públicas, cabeceras, JWT y `ErrorResponse`. Las decisiones de enrutamiento, JWT, CORS, errores y llamadas internas se documentan en ADR. |

## Pruebas de integración obligatorias

Se ejecutan a través del GW, con MS01 real o con un doble que cumpla su `swagger.yml`.

| ID | Escenario | Resultado esperado |
|---|---|---|
| INT-GW-01 | `POST /api/v1/auth/register` y `/login` sin JWT | Llegan a MS01 sin cambios; ni las credenciales ni el token aparecen en los logs. |
| INT-GW-02 | Solicitud protegida con un JWT válido de MS01 | MS01 recibe `X-User-Id`, `X-User-Role` y `X-Correlation-ID`, sin `Authorization`. |
| INT-GW-03 | JWT ausente, expirado, alterado o con `iss`/`aud`/`alg` incorrectos | `401 NO_AUTENTICADO`; MS01 no recibe la solicitud. |
| INT-GW-04 | Cliente envía `X-User-Id`, `X-User-Role` o `X-Service-Name` falsos | MS01 recibe solo la identidad del JWT. |
| INT-GW-05 | `PATCH /api/v1/users/status` con un JWT de `ESTUDIANTE` | El GW propaga la identidad y MS01 responde `403 PROHIBIDO` válido, que llega sin cambios. |
| INT-GW-06 | MS01 responde `400`, `401`, `403`, `404` o `409` con un `ErrorResponse` válido | Llega sin cambios, con `X-Correlation-ID`. |
| INT-GW-07 | El MS responde un error sin `correlation_id`, con otro valor o con el cuerpo por defecto de Spring | `502 SERVICIO_NO_DISPONIBLE`. |
| INT-GW-08 | Sin `X-Correlation-ID` o con un valor que no es UUID v4 | Se genera uno nuevo, se propaga y se devuelve. |
| INT-GW-09 | El MS tarda más que `GATEWAY_DOWNSTREAM_TIMEOUT_MS` | `504 TIEMPO_AGOTADO`. |
| INT-GW-10 | Destino inaccesible | `502 SERVICIO_NO_DISPONIBLE` sin detalles internos. |
| INT-GW-11 | Llamada interna con una clave de servicio válida y `X-User-*` | El destino recibe `X-Service-Name` y la identidad, sin `X-Service-Key`. |
| INT-GW-12 | Llamada interna con una clave inválida, o ruta interna sin clave | `401 NO_AUTENTICADO`; el destino no recibe nada. |
| INT-GW-13 | Acceso directo desde fuera al puerto de un MS | Bloqueado por la red/despliegue. |
| INT-GW-14 | `GET /actuator/health` | `200 {"status":"UP"}` en `≤ 200 ms`; `/actuator/metrics` en `SERVER_PORT` responde `404`. |
| INT-GW-15 | Preflight CORS desde un origen permitido y uno no permitido | `200` con cabeceras CORS / `403`. |

## Condición de cierre

El GW se considera integrable con MS01 cuando pasan INT-GW-01 a INT-GW-10 y los criterios de aceptación de RF-GW-01…07. INT-GW-11 y 12 se exigen al activar la primera llamada interna (MS→MS08), e INT-GW-13 a 15 al desplegar en Docker.
