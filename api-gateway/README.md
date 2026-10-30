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

El Gateway es la única puerta HTTP de CampusSeguro, tanto para el tráfico externo como entre microservicios. Enruta a MS01–MS08, valida localmente el JWT y propaga la identidad en cabeceras. Además centraliza CORS, la normalización de errores, la correlación y el logging técnico. No administra identidad ni ejecuta lógica de dominio.

## Documentación

| Documento | Contenido |
|---|---|
| [Estándares de Integración](https://github.com/CampusSeguro-DOSW-2026-2/documentation/wiki/Est%C3%A1ndares-de-Integraci%C3%B3n-y-Contratos-Transversales) (wiki) | **Valores y contratos compartidos con todos los squads.** Es la fuente de verdad. |
| [01-requerimientos-funcionales.md](01-requerimientos-funcionales.md) | RF-GW-01…08 con reglas y criterios de aceptación. |
| [02-contratos-integracion.md](02-contratos-integracion.md) | Pipeline, tabla de rutas, políticas de acceso, errores propios y configuración. |
| [03-calidad-y-pruebas.md](03-calidad-y-pruebas.md) | RNF-GW-01…07 y pruebas de integración INT-GW-01…15. |
| [adr/](adr/) | Decisiones de arquitectura ADR-001…005 (enrutamiento, errores, JWT, CORS, llamadas internas). |
| [api/swagger.yml](api/swagger.yml) | Contrato OpenAPI 3 del GW: rutas expuestas, política de acceso, credenciales, cabeceras y errores propios. **Estándar para que el frontend y los MS llamen al GW.** |
| [Matriz Rol‐Permiso](https://github.com/CampusSeguro-DOSW-2026-2/documentation/wiki/Matriz-Rol%E2%80%90Permiso) (wiki) | Permisos por rol (el GW no la aplica; la usa cada MS). |
| [Contratos OpenAPI de los MS](https://github.com/CampusSeguro-DOSW-2026-2/documentation) | El GW solo enruta lo publicado allí. |

## Alcance de esta versión

**Incluye:** enrutamiento por configuración, validación local del JWT RS256, sanitización y propagación de identidad, llamadas internas con clave de servicio, CORS, `ErrorResponse` (con `502` ante errores downstream no conformes), correlación, logs y healthcheck.

**No incluye:**
* Usuarios, contraseñas, roles y sesiones: son de MS01.
* Autorización por rol: la hace cada MS con la matriz.
* Revocación de tokens, refresh tokens y JWKS o múltiples llaves.
* Descubrimiento de servicios (Eureka) y reintentos.
* Auditoría de dominio: la emite cada MS a MS08.

## Microservicios enlazados

Los puertos son internos; solo se publica el `8080` del GW.

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

Se siguen en [Estándares — Decisiones abiertas](https://github.com/CampusSeguro-DOSW-2026-2/documentation/wiki/Est%C3%A1ndares-de-Integraci%C3%B3n-y-Contratos-Transversales): el OpenAPI de MS08 para la auditoría y los prefijos de MS02–MS07.
