# API Gateway (GW)

El Gateway es la única puerta HTTP de CampusSeguro, tanto para el tráfico externo como entre microservicios. Enruta a MS01–MS08, valida el JWT (firma y claims en local, más los cortes de sesión que publica MS01) y propaga la identidad (usuario, roles y permisos) en cabeceras. Además centraliza CORS, la normalización de errores, la correlación y el logging técnico. No administra identidad ni ejecuta lógica de dominio.

## Documentación

| Documento | Contenido |
|---|---|
| [1_requirements.md](1_requirements.md) | RF-GW-01…08 con reglas y criterios de aceptación. |
| [contracts.md](contracts.md) | Pipeline, tabla de rutas, políticas de acceso, errores propios, configuración y aislamiento en la nube. |
| [quality-and-tests.md](quality-and-tests.md) | RNF-GW-01…07 y pruebas de integración INT-GW-01…19. |
| [api/openapi.yml](api/openapi.yml) | Contrato OpenAPI 3 del GW: rutas expuestas, política de acceso, credenciales, cabeceras y errores propios. Estándar para que el frontend y los MS llamen al GW. |
| [diagrams/gw-c4-1-context.drawio.svg](diagrams/gw-c4-1-context.drawio.svg) | C4 nivel 1: quién usa el GW (comunidad del campus, equipo de operación) y con qué sistemas interactúa (aplicación web, MS01, MS02–MS08), sin tecnología. |
| [diagrams/gw-c4-2-containers.drawio.svg](diagrams/gw-c4-2-containers.drawio.svg) | C4 nivel 2: frontend, GW y MS01…MS08 con su tecnología, el bus de rutas con mTLS y las llamadas internas con clave de servicio. |
| [diagrams/gw-c4-3-components.drawio.svg](diagrams/gw-c4-3-components.drawio.svg) | C4 nivel 3 (UML 2): la cadena de filtros del GW en su orden real, los componentes que requiere cada filtro, la consulta periódica de los cortes de sesión a MS01 y la salida con mTLS a MS01…MS08. |
| [diagrams/gw-deployment-azure.drawio.svg](diagrams/gw-deployment-azure.drawio.svg) | Despliegue: VM del GW en Azure con NSG, Caddy de borde con Let's Encrypt, stacks `gw-prod` y `gw-test` con su certificado de cliente, y la salida con mTLS a la VM de cada MS. |
| [diagrams/gw-classes.puml](diagrams/gw-classes.puml) · [SVG](diagrams/gw-classes.svg) | Clases (UML): tabla de rutas, políticas de acceso, identidad del usuario y del servicio llamante, cortes de sesión y errores propios. |
| Edición de los diagramas | Los `.drawio.svg` se abren y se editan en [draw.io](https://app.diagrams.net/); el `.svg` de clases se genera desde su `.puml` con PlantUML. |
| [Estándares de integración](../integration/standards.md) | Valores y contratos compartidos con todos los squads. Es la fuente de verdad. |
| [Matriz de roles y permisos](../integration/permissions-matrix.md) | Asignación inicial de permisos por rol. El GW no la aplica: cada MS autoriza con `X-User-Permission`. |
| [Conexión al API Gateway](../integration/gateway-connection.md) | Guía para que un squad conecte su microservicio en la nube. |

Los contratos OpenAPI de los microservicios están en la carpeta `api/` de cada uno, y el GW solo enruta lo publicado allí.

## Alcance

**Incluye:** enrutamiento por configuración, validación del JWT RS256 con los cortes de sesión de MS01, sanitización y propagación de identidad, llamadas internas con clave de servicio, CORS, `ErrorResponse` (con `502` ante errores downstream no conformes), correlación, logs, healthcheck y, en la nube, mTLS hacia los MS con despliegue a prueba y producción ([contracts.md §7](contracts.md#7-aislamiento-en-la-nube)).

**No incluye:**
* Usuarios, contraseñas, roles y sesiones: son de MS01.
* Autorización por permiso: la hace cada MS con `X-User-Permission`.
* Revocación individual de tokens (los cortes de sesión invalidan por usuario o por rol), refresh tokens y JWKS o múltiples llaves.
* Descubrimiento de servicios (Eureka) y reintentos.
* Auditoría de dominio: la emite cada MS a MS08.

## Microservicios enlazados

Los puertos son internos; solo se publica el `8080` del GW. En la nube cada MS está en su propia VM y el GW lo alcanza por `https://<IP>:443` (producción) o `:8443` (prueba), con mTLS ([guía para los squads](../integration/gateway-connection.md)).

| Servicio | Responsabilidad | Destino interno |
|---|---|---|
| ms01-identity-and-access | Usuarios, autenticación, roles, permisos y sesiones | `ms01-identity-and-access:8081` |
| ms02-campus-and-security | Edificios, espacios, zonas seguras, puntos de encuentro y rutas | `ms02-campus-and-security:8082` |
| ms03-alerts-and-emergencies | Creación, activación, distribución y seguimiento de alertas | `ms03-alerts-and-emergencies:8083` |
| ms04-evacuation-and-check-in | Estado de las personas y consolidación por punto de encuentro | `ms04-evacuation-and-check-in:8084` |
| ms05-brigades-and-incidents | Coordinación operativa, solicitudes de ayuda e incidentes | `ms05-brigades-and-incidents:8085` |
| ms06-drills | Planeación, ejecución, cierre y evaluación de simulacros | `ms06-drills:8086` |
| ms07-training-and-ai-assistant | Contenidos preventivos, evaluaciones y tutor inteligente | `ms07-training-and-ai-assistant:8087` |
| ms08-analytics-and-audit | Indicadores, históricos y trazabilidad | `ms08-analytics-and-audit:8088` |
