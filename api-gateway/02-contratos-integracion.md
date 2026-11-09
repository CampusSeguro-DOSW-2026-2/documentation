# Contratos del Gateway (GW-C)

Contiene solo lo propio del Gateway: el pipeline, la tabla de rutas, las políticas de acceso, sus errores y su configuración. Los contratos compartidos (JWT, cabeceras, `ErrorResponse`, CORS, correlación, logs, health, auditoría) están en los [Estándares de Integración](https://github.com/CampusSeguro-DOSW-2026-2/documentation/wiki/Est%C3%A1ndares-de-Integraci%C3%B3n-y-Contratos-Transversales) (EST) y **no se copian aquí**.

---

## 1. Pipeline de una solicitud

| # | Paso | RF |
|---:|---|---|
| 1 | Correlación: conservar o generar `X-Correlation-ID` | RF-GW-06 |
| 2 | Resolución de ruta (prefijo más específico); si no existe, `404` | RF-GW-01 |
| 3 | CORS y preflight | RF-GW-04 |
| 4 | Política de acceso: pública, JWT o clave de servicio | RF-GW-02 / RF-GW-08 |
| 5 | Cabeceras: sanitizar, eliminar credenciales, inyectar identidad | RF-GW-03 |
| 6 | Reenvío con `GATEWAY_DOWNSTREAM_TIMEOUT_MS` | RF-GW-01 |
| 7 | Validación de errores downstream (`502` si no conforme) | RF-GW-05 |
| 8 | Log técnico | RF-GW-07 |

## 2. Tabla de enrutamiento

Los prefijos los pide cada squad (guía de la wiki, *Conexión al API Gateway*) y deben estar publicados en el `swagger.yml` del MS en el [repositorio documentation](https://github.com/CampusSeguro-DOSW-2026-2/documentation). El GW conserva la ruta completa al reenviar. Cada ruta activa se publica también, por operación y con su política, en [api/swagger.yml](api/swagger.yml).

| Prefijo | Destino | Contrato | Estado |
|---|---|---|---|
| `/actuator/health` | GW (propio) | EST | Activo |
| `/api/v1/auth/**` | `MS01_URI` | [swagger MS01](https://github.com/CampusSeguro-DOSW-2026-2/documentation/blob/main/ms01-identity-and-access/api/swagger.yml) | Activo |
| `/api/v1/users`, `/api/v1/users/**` | `MS01_URI` | swagger MS01 | Activo |
| `/api/v1/roles` | `MS01_URI` | swagger MS01 | Activo |
| `/api/v1/buildings`, `/api/v1/buildings/**`, `/api/v1/floors/**`, `/api/v1/spaces/**`, `/api/v1/safe-zones`, `/api/v1/safe-zones/**`, `/api/v1/evacuation-routes`, `/api/v1/evacuation-routes/**`, `/api/v1/preventive-recommendations`, `/api/v1/preventive-recommendations/**`, `/api/v1/campuses`, `/api/v1/campuses/**`, `/api/v1/open-zones/**`, `/api/v1/exits/**`, `/api/v1/stairs/**`, `/api/v1/public/**` | `MS02_URI` | swagger MS02 (parcial: falta publicar pisos, campus, zonas abiertas, salidas, escaleras, recomendaciones y `public`) | Enrutado; MS sin desplegar |
| `/api/v1/alertas`, `/api/v1/alertas/**` | `MS03_URI` | OpenAPI MS03 | Enrutado; MS sin desplegar |
| `/api/v1/chi/**` | `MS04_URI` | [swagger MS04](https://github.com/CampusSeguro-DOSW-2026-2/documentation/blob/main/ms04-evacuation-check-in/api/swagger.yml) | Enrutado; destino en la nube configurado, MS sin desplegar |
| `/api/v1/brigadistas/**`, `/api/v1/incidentes/**`, `/api/v1/bei/**` | `MS05_URI` | Pendiente (sin OpenAPI publicado) | Enrutado; MS sin desplegar |
| `/api/v1/eventos`, `/api/v1/eventos/**` | `MS06_URI` | Pendiente (sin OpenAPI publicado) | Enrutado; MS sin desplegar |
| `/api/v1/analitica/**`, `/api/v1/auditoria/**`, `/api/v1/audit/events` | `MS08_URI` | Pendiente (sin OpenAPI publicado); `audit/events` sale de EST §12 | Enrutado; MS sin desplegar |

## 3. Políticas de acceso

La política es independiente del prefijo. Se evalúa por **método + ruta exacta**; lo que no aparece aquí es **protegido**.

| Política | Método | Ruta | Credencial |
|---|---|---|---|
| Pública | `GET` | `/actuator/health` | Ninguna |
| Pública | `POST` | `/api/v1/auth/register` | Ninguna |
| Pública | `POST` | `/api/v1/auth/login` | Ninguna |
| Pública | `POST` | `/api/v1/auth/password/forgot` | Ninguna |
| Pública | `POST` | `/api/v1/auth/password/reset` | Ninguna |
| Interna | `POST` | `/api/v1/audit/events` | Clave de servicio |
| Interna | `POST` | `/api/v1/users/batch` | Clave de servicio |
| Interna | `GET` | `/api/v1/users/by-role` | Clave de servicio |
| Protegida | Todo lo demás (incluido `POST /api/v1/auth/logout`) | | JWT o clave de servicio |

Las rutas públicas deben coincidir con EST. Las rutas internas deben coincidir con las llamadas registradas en EST.

## 4. Errores que genera el GW

Formato: `ErrorResponse` de EST. El GW solo genera estos casos:

| HTTP | `codigo` | Cuándo | `mensaje` |
|---:|---|---|---|
| 400 | `VALIDACION` | Varias cabeceras `Authorization` o `X-Service-Key`; `Authorization` junto con `X-Service-Key`; `X-User-*` inválidos en una llamada interna | `Solicitud inválida.` |
| 401 | `NO_AUTENTICADO` | JWT ausente o inválido (RF-GW-02); clave de servicio ausente o inválida (RF-GW-08) | `Token ausente, inválido o expirado.` |
| 403 | `PROHIBIDO` | Preflight CORS no permitido | `Acceso denegado por la política perimetral.` |
| 404 | `NO_ENCONTRADO` | Ruta sin destino | `Ruta no encontrada.` |
| 500 | `ERROR_INTERNO` | Excepción no controlada en el GW | `Error interno del servidor.` |
| 502 | `SERVICIO_NO_DISPONIBLE` | MS inalcanzable (conexión rechazada o cortada, DNS no resuelto, TLS rechazado) o error downstream no conforme (RF-GW-05) | `Servicio temporalmente no disponible.` |
| 504 | `TIEMPO_AGOTADO` | El MS supera `GATEWAY_DOWNSTREAM_TIMEOUT_MS` | `El servicio de destino no respondió a tiempo.` |

## 5. Configuración

Los valores por defecto **deben coincidir** con EST. Si cambian allí, se actualizan aquí en el mismo PR.

| Variable | Defecto | Uso |
|---|---|---|
| `SPRING_PROFILES_ACTIVE` | — (el `Dockerfile` fija `docker`) | `local` (MS01 en `localhost:8081`) o `docker` (MS01 por su nombre DNS en la red Docker). En la nube, `docker,cloud` ([ADR-006](adr/ADR-006-aislamiento-en-la-nube.md)). |
| `SERVER_PORT` | `8080` | Puerto externo (el único publicado). En la nube lo publica Caddy en `443` (prod) y `8443` (test). |
| `MANAGEMENT_SERVER_PORT` | `9090` | Actuator (métricas); solo red interna. |
| `MS01_URI` … `MS08_URI` (sin `MS07_URI`, que no tiene ruta) | Los perfiles `local` y `docker` las traen (`http://localhost:808N` y `http://<repositorio>:808N`) | En la nube, `https://<IP de la VM del MS>:443` (prod) o `:8443` (test): `MS01_URI` es variable propia y las demás van en `GW_EXTRA_ENV`. Mientras un MS no tenga VM, su ruta responde `502 SERVICIO_NO_DISPONIBLE`. |
| `JWT_PUBLIC_KEY` | — (obligatoria) | Llave pública RSA en PEM. |
| `JWT_ISSUER` / `JWT_AUDIENCE` | `ms01-identity-and-access` / `campusseguro` | Iguales a los de MS01. |
| `JWT_CLOCK_SKEW_SECONDS` | `30` | Tolerancia de reloj. |
| `JWT_MAX_LIFETIME_SECONDS` | `3600` | Máximo `exp − iat`. |
| `GATEWAY_DOWNSTREAM_TIMEOUT_MS` | `3000` | Timeout por llamada al MS. |
| `GW_SERVICE_KEY_MS01` … `GW_SERVICE_KEY_MS08` | — (obligatoria por MS que llame) | Clave de servicio de cada MS (≥ 32 caracteres). |
| `CORS_ALLOWED_ORIGINS` | — (obligatoria) | Orígenes separados por coma. |
| `CORS_ALLOW_CREDENTIALS` | `false` | Si es `true`, se prohíbe `*`. |
| `CORS_MAX_AGE_SECONDS` | `3600` | Vigencia del preflight. |
| `LOG_LEVEL` | `INFO` | Nivel base. |
| `GW_CLIENT_KEYSTORE` | — (obligatoria con `cloud`) | PKCS12 con el certificado de cliente del GW (`file:/ruta`). |
| `GW_CLIENT_KEYSTORE_PASSWORD` | — (obligatoria con `cloud`) | Contraseña del PKCS12. |
| `GW_TRUSTED_CA` | — (obligatoria con `cloud`) | CA del entorno en PEM (`file:/ruta`); el GW solo confía en ella hacia los MS. |

* La configuración se valida al arrancar: si falta una variable obligatoria o es inválida (rutas y timeout, RF-GW-01 A5; políticas de acceso; JWT; CORS; claves de servicio, RN-GW-20; con `cloud`, el certificado de cliente y la CA), el GW no arranca y el log indica la causa.
* Los perfiles `local` y `docker` solo cambian las `MSnn_URI`; ambos traen `CORS_ALLOWED_ORIGINS=http://localhost:5173` (Vite). El perfil `cloud` se suma a `docker` y activa el mTLS hacia los MS. Los secretos nunca van en el repositorio.

## 6. Dependencias

* **MS01:** solo emite el JWT. El GW no lo consulta por solicitud.
* **Bases de datos y MS08:** el GW no depende de ellos.
* **Descubrimiento de servicios:** no se usa (sin Eureka); los destinos son URI estáticas por ambiente.
* **Reintentos:** no hay.
* **En la nube:** Caddy de borde (HTTPS) y la CA de cada entorno para el mTLS ([ADR-006](adr/ADR-006-aislamiento-en-la-nube.md)).
