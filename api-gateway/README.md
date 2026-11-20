# API GATEWAY (punto de enlace)

GW - Single point of entry, routing, and cross-cutting controls orchestrator.

| Name                            | GitHub Username                   |
|---------------------------------|-----------------------------------|
| Daniel Santiago Morales Perdomo | `Fukuyaku88`                      |
| Edgar Daniel Ruiz Patiño        | `DaniRuiz11`                      |
| Miguel Ángel Acero Laverde      | `Miguelim101`                     |
| Thomas Sebastian García Gomez   | `thomassebastiangarciagomez-ctrl` |
| Rodrigo Gualtero                | `Rodrigoreg92`                    |

---

El Gateway es la única puerta HTTP de CampusSeguro, tanto para el tráfico externo como entre microservicios. Enruta a MS01–MS08, valida el JWT (firma y claims en local, más los cortes de sesión que publica MS01) y propaga la identidad (usuario, roles y permisos) en cabeceras. Además centraliza CORS, la normalización de errores, la correlación y el logging técnico. No administra identidad ni ejecuta lógica de dominio.

## Documentación

| Documento | Contenido |
|---|---|
| [Estándares de Integración](https://github.com/CampusSeguro-DOSW-2026-2/documentation/wiki/Est%C3%A1ndares-de-Integraci%C3%B3n-y-Contratos-Transversales) (wiki) | **Valores y contratos compartidos con todos los squads.** Es la fuente de verdad. |
| [01-requerimientos-funcionales.md](01-requerimientos-funcionales.md) | RF-GW-01…08 con reglas y criterios de aceptación. |
| [02-contratos-integracion.md](02-contratos-integracion.md) | Pipeline, tabla de rutas, políticas de acceso, errores propios y configuración. |
| [03-calidad-y-pruebas.md](03-calidad-y-pruebas.md) | RNF-GW-01…07 y pruebas de integración INT-GW-01…19. |
| [adr/](adr/) | Decisiones de arquitectura ADR-001…006 (enrutamiento, errores, JWT, CORS, llamadas internas, aislamiento en la nube). |
| [diagramas/gw_c4_3_componentes.drawio.svg](diagramas/gw_c4_3_componentes.drawio.svg) | C4 nivel 3 (UML 2): la cadena de filtros del GW en su orden real, sus componentes de apoyo y los puertos `:8080`, `:9090` y `MSnn_URI`. Se edita abriendo el `.drawio.svg` en draw.io. |
| [api/swagger.yml](api/swagger.yml) | Contrato OpenAPI 3 del GW: rutas expuestas, política de acceso, credenciales, cabeceras y errores propios. **Estándar para que el frontend y los MS llamen al GW.** |
| [Matriz Rol‐Permiso](https://github.com/CampusSeguro-DOSW-2026-2/documentation/wiki/Matriz-Rol%E2%80%90Permiso) (wiki) | Asignación inicial de permisos por rol (el GW no la aplica; cada MS autoriza con `X-User-Permission`). |
| [Contratos OpenAPI de los MS](https://github.com/CampusSeguro-DOSW-2026-2/documentation) | El GW solo enruta lo publicado allí. |

## Alcance de esta versión

**Incluye:** enrutamiento por configuración, validación del JWT RS256 con los cortes de sesión de MS01, sanitización y propagación de identidad, llamadas internas con clave de servicio, CORS, `ErrorResponse` (con `502` ante errores downstream no conformes), correlación, logs, healthcheck y, en la nube, mTLS hacia los MS con despliegue a prueba y producción ([ADR-006](adr/ADR-006-aislamiento-en-la-nube.md)).

**No incluye:**
* Usuarios, contraseñas, roles y sesiones: son de MS01.
* Autorización por permiso: la hace cada MS con `X-User-Permission`.
* Revocación individual de tokens (los cortes de sesión invalidan por usuario o por rol), refresh tokens y JWKS o múltiples llaves.
* Descubrimiento de servicios (Eureka) y reintentos.
* Auditoría de dominio: la emite cada MS a MS08.

## Microservicios enlazados

Los puertos son internos; solo se publica el `8080` del GW. En la nube cada MS está en su propia VM y el GW lo alcanza por `https://<IP>:443` (prod) o `:8443` (test), con mTLS ([guía para los squads](https://github.com/CampusSeguro-DOSW-2026-2/documentation/wiki/Conexi%C3%B3n-al-API-Gateway)).

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

## Pendientes con otros squads

Se siguen en [Estándares — Decisiones abiertas](https://github.com/CampusSeguro-DOSW-2026-2/documentation/wiki/Est%C3%A1ndares-de-Integraci%C3%B3n-y-Contratos-Transversales). Los prefijos de MS02–MS06 y MS08 ya están en la tabla de rutas ([02-contratos-integracion.md](02-contratos-integracion.md) §2); falta:
* Publicar el OpenAPI de MS05, MS06 y MS08, y completar el de MS02.
* MS05: sus incidentes están bajo `/api/v1/eventos/{eventoId}/incidentes`, que pertenece al prefijo de MS06.
* La IP de la VM de cada MS para su `MSnn_URI` en la nube (MS04 ya la entregó).
