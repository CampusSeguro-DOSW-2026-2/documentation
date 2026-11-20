# Contratos del Gateway (GW-C)

Contiene solo lo propio del Gateway: el pipeline, la tabla de rutas, las políticas de acceso, sus errores y su configuración. Los contratos compartidos (JWT, cabeceras, `ErrorResponse`, CORS, correlación, logs, health, auditoría) están en los [Estándares de Integración](../integration/standards.md) (EST) y **no se copian aquí**.

---

## 1. Pipeline de una solicitud

| # | Paso | RF |
|---:|---|---|
| 1 | Correlación: conservar o generar `X-Correlation-ID` | RF-GW-06 |
| 2 | Resolución de ruta (prefijo más específico); si no existe, `404` | RF-GW-01 |
| 3 | CORS y preflight | RF-GW-04 |
| 4 | Política de acceso: pública, JWT (con los cortes de sesión) o clave de servicio | RF-GW-02 / RF-GW-08 |
| 5 | Cabeceras: sanitizar, eliminar credenciales, inyectar identidad | RF-GW-03 |
| 6 | Reenvío con `GATEWAY_DOWNSTREAM_TIMEOUT_MS` | RF-GW-01 |
| 7 | Validación de errores downstream (`502` si no conforme) | RF-GW-05 |
| 8 | Log técnico | RF-GW-07 |

## 2. Tabla de enrutamiento

Los prefijos los pide cada squad con la guía [Conexión al API Gateway](../integration/gateway-connection.md) y deben estar publicados en el `api/openapi.yml` de la carpeta de su microservicio. El GW conserva la ruta completa al reenviar. Cada ruta activa se publica también, por operación y con su política, en [api/openapi.yml](api/openapi.yml).

| Prefijo | Destino | OpenAPI |
|---|---|---|
| `/actuator/health` | GW (propio) | [OpenAPI del GW](api/openapi.yml) |
| `/api/v1/auth/**`, `/api/v1/users`, `/api/v1/users/**`, `/api/v1/roles`, `/api/v1/roles/**`, `/api/v1/permissions` | `MS01_URI` | [MS01](../ms01-identity-and-access/api/openapi.yml) |
| `/api/v1/buildings`, `/api/v1/buildings/**`, `/api/v1/floors/**`, `/api/v1/spaces/**`, `/api/v1/safe-zones`, `/api/v1/safe-zones/**`, `/api/v1/evacuation-routes`, `/api/v1/evacuation-routes/**`, `/api/v1/preventive-recommendations`, `/api/v1/preventive-recommendations/**`, `/api/v1/campuses`, `/api/v1/campuses/**`, `/api/v1/open-zones/**`, `/api/v1/exits/**`, `/api/v1/stairs/**`, `/api/v1/public/**`, `/api/v1/images`, `/api/v1/images/**` | `MS02_URI` | [MS02](../ms02-campus-and-security/api/openapi.yml) |
| `/api/v1/alertas`, `/api/v1/alertas/**` | `MS03_URI` | [MS03](../ms03-alerts-and-emergencies/api/openapi.yml) |
| `/api/v1/chi/**` | `MS04_URI` | [MS04](../ms04-evacuation-and-check-in/api/openapi.yml) |
| `/api/v1/brigadistas/**`, `/api/v1/incidentes/**`, `/api/v1/bei/**`, `/api/v1/asignaciones/**`, `/api/v1/responsabilidades/**` | `MS05_URI` | [MS05](../ms05-brigades-and-incidents/api/openapi.yml) |
| `/api/v1/eventos`, `/api/v1/eventos/**`, `/api/v1/tipos-evento`, `/api/v1/tipos-evento/**`, `/api/v1/tipos-alerta` | `MS06_URI` | Sin OpenAPI en este repositorio |
| `/api/v1/training/**`, `/api/v1/tutor/**` | `MS07_URI` | Sin OpenAPI en este repositorio. `/api/v1/tutor/**` espera hasta `GATEWAY_TUTOR_TIMEOUT_MS` |
| `/api/v1/analitica/**`, `/api/v1/auditoria/**`, `/api/v1/audit/events` | `MS08_URI` | Sin OpenAPI en este repositorio. `audit/events` es el contrato de los [Estándares §12](../integration/standards.md#12-auditoría-ms08) |

`GET /api/v1/internal/session-cuts` de MS01 (tag *Interno (Gateway)*) no figura en la tabla: la consume el propio GW (§5, `SESSION_CUTS_URL`) y no se publica a través de él, así que responde `404` aunque se envíe una clave de servicio.

## 3. Políticas de acceso

La política es independiente del prefijo. Se evalúa por **método + ruta exacta**; lo que no aparece aquí es **protegido**.

| Política | Método | Ruta | Credencial |
|---|---|---|---|
| Pública | `GET` | `/actuator/health` | Ninguna |
| Pública | `POST` | `/api/v1/auth/register` | Ninguna |
| Pública | `POST` | `/api/v1/auth/register/verify` | Ninguna |
| Pública | `POST` | `/api/v1/auth/register/resend` | Ninguna |
| Pública | `POST` | `/api/v1/auth/login` | Ninguna |
| Pública | `POST` | `/api/v1/auth/password/forgot` | Ninguna |
| Pública | `POST` | `/api/v1/auth/password/reset` | Ninguna |
| Interna | `POST` | `/api/v1/audit/events` | Clave de servicio |
| Interna | `POST` | `/api/v1/users/batch` | Clave de servicio |
| Interna | `GET` | `/api/v1/users/by-role` | Clave de servicio |
| Interna | `GET` | `/api/v1/tipos-alerta` | Clave de servicio |
| Protegida | Todo lo demás (incluido `POST /api/v1/auth/logout`) | | JWT o clave de servicio |

Las rutas públicas deben coincidir con EST. Las rutas internas deben coincidir con las llamadas registradas en EST.

## 4. Errores que genera el GW

Formato: `ErrorResponse` de EST. El GW solo genera estos casos:

| HTTP | `codigo` | Cuándo | `mensaje` |
|---:|---|---|---|
| 400 | `VALIDACION` | Varias cabeceras `Authorization` o `X-Service-Key`; `Authorization` junto con `X-Service-Key`; identidad de usuario incompleta, repetida o con formato inválido (`X-User-Id`, `X-User-Role`, `X-User-Permission`) en una llamada interna | `Solicitud inválida.` |
| 401 | `NO_AUTENTICADO` | JWT ausente, inválido o anterior a un corte de sesión (RF-GW-02); clave de servicio ausente o inválida (RF-GW-08) | `Token ausente, inválido o expirado.` |
| 403 | `PROHIBIDO` | Preflight CORS no permitido | `Acceso denegado por la política perimetral.` |
| 404 | `NO_ENCONTRADO` | Ruta sin destino | `Ruta no encontrada.` |
| 500 | `ERROR_INTERNO` | Excepción no controlada en el GW | `Error interno del servidor.` |
| 502 | `SERVICIO_NO_DISPONIBLE` | MS inalcanzable (conexión rechazada o cortada, DNS no resuelto, TLS rechazado) o error downstream no conforme (RF-GW-05) | `Servicio temporalmente no disponible.` |
| 504 | `TIEMPO_AGOTADO` | El MS supera `GATEWAY_DOWNSTREAM_TIMEOUT_MS` | `El servicio de destino no respondió a tiempo.` |

## 5. Configuración

Los valores por defecto **deben coincidir** con EST. Si cambian allí, se actualizan aquí en el mismo PR.

| Variable | Defecto | Uso |
|---|---|---|
| `SPRING_PROFILES_ACTIVE` | — (el `Dockerfile` fija `docker`) | `local` (MS01 en `localhost:8081`) o `docker` (MS01 por su nombre DNS en la red Docker). En la nube, `docker,cloud` (§7). |
| `SERVER_PORT` | `8080` | Puerto externo (el único publicado). En la nube lo publica Caddy en `443` (prod) y `8443` (test). |
| `MANAGEMENT_SERVER_PORT` | `9090` | Actuator (métricas); solo red interna. |
| `MS01_URI` … `MS08_URI` | Los perfiles `local` y `docker` las traen (`http://localhost:808N` y `http://<repositorio>:808N`) | En la nube, `https://<IP de la VM del MS>:443` (prod) o `:8443` (test): `MS01_URI` es variable propia y las demás van en `GW_EXTRA_ENV`. Mientras un MS no tenga VM, su ruta responde `502 SERVICIO_NO_DISPONIBLE`. |
| `JWT_PUBLIC_KEY` | — (obligatoria) | Llave pública RSA en PEM. |
| `JWT_ISSUER` / `JWT_AUDIENCE` | `ms01-identity-and-access` / `campusseguro` | Iguales a los de MS01. |
| `JWT_CLOCK_SKEW_SECONDS` | `30` | Tolerancia de reloj. |
| `JWT_MAX_LIFETIME_SECONDS` | `3600` | Máximo `exp − iat`. |
| `SESSION_CUTS_URL` | `${MS01_URI}/api/v1/internal/session-cuts` | URI `http(s)` absoluta de la ruta interna de MS01 con los cortes de sesión (EST §5). Usa la misma conexión que las rutas hacia MS01 (mTLS en la nube). |
| `SESSION_CUTS_INTERVAL_SECONDS` | `4` | Cada cuántos segundos consulta los cortes de sesión, además de la consulta al arrancar. Con el timeout, fija el retraso máximo de un corte: 7 s con los valores por defecto. |
| `SESSION_CUTS_TIMEOUT_SECONDS` | `3` | Tiempo máximo de cada consulta de cortes. |
| `GATEWAY_DOWNSTREAM_TIMEOUT_MS` | `3000` | Timeout por llamada al MS. |
| `GATEWAY_TUTOR_TIMEOUT_MS` | `10000` | Tiempo de respuesta solo para `/api/v1/tutor/**` (`gateway.path-timeouts`): el tutor de MS07 consulta un modelo de lenguaje y corta su llamada a los 8 s. La conexión sigue con `GATEWAY_DOWNSTREAM_TIMEOUT_MS`. |
| `GW_SERVICE_KEY_MS01` … `GW_SERVICE_KEY_MS08` | — (obligatoria por MS que llame) | Clave de servicio de cada MS (≥ 32 caracteres). |
| `CORS_ALLOWED_ORIGINS` | — (obligatoria) | Orígenes separados por coma, sin ruta ni barra final. |
| `CORS_ALLOW_CREDENTIALS` | `false` | Si es `true`, se prohíbe `*`. |
| `CORS_MAX_AGE_SECONDS` | `3600` | Vigencia del preflight. |
| `LOG_LEVEL` | `INFO` | Nivel base. |
| `GW_CLIENT_KEYSTORE` | — (obligatoria con `cloud`) | PKCS12 con el certificado de cliente del GW (`file:/ruta`). |
| `GW_CLIENT_KEYSTORE_PASSWORD` | — (obligatoria con `cloud`) | Contraseña del PKCS12. |
| `GW_TRUSTED_CA` | — (obligatoria con `cloud`) | CA del entorno en PEM (`file:/ruta`); el GW solo confía en ella hacia los MS. |

* La configuración se valida al arrancar: si falta una variable obligatoria o es inválida (rutas y timeout, RF-GW-01 A5; políticas de acceso; JWT; cortes de sesión, con una URI `http(s)` absoluta y un intervalo y un timeout mayores que 0; CORS; claves de servicio, RN-GW-20; con `cloud`, el certificado de cliente y la CA), el GW no arranca y el log indica la causa.
* Los perfiles `local` y `docker` solo cambian las `MSnn_URI`; ambos traen `CORS_ALLOWED_ORIGINS=http://localhost:5173` (Vite). El perfil `cloud` se suma a `docker` y activa el mTLS hacia los MS. Los secretos nunca van en el repositorio.

## 6. Dependencias

* **MS01:** emite el JWT y publica los cortes de sesión. El GW no lo consulta por solicitud: lee los cortes de una instantánea en memoria que actualiza cada `SESSION_CUTS_INTERVAL_SECONDS` (RF-GW-02); cada instancia del GW mantiene la suya. Si MS01 no responde, conserva los últimos cortes.
* **Bases de datos y MS08:** el GW no depende de ellos.
* **Descubrimiento de servicios:** no se usa (sin Eureka); los destinos son URI estáticas por ambiente.
* **Reintentos:** no hay.
* **Instancias:** cada destino es una sola URI estática; varias instancias de un MS exigen un balanceador externo.
* **En la nube:** Caddy de borde (HTTPS) y la CA de cada entorno para el mTLS (§7).

## 7. Aislamiento en la nube

En la nube cada servicio corre en su propia VM (los MS en cuentas distintas de Azure o AWS y el GW en otra), así que no existe la red interna de Docker. Como los MS confían en `X-User-Id`, `X-User-Role`, `X-User-Permission` y `X-Service-Name` sin validar un JWT, nadie debe llegar a un MS sin pasar por el GW. Se garantiza sin cambiar el código de los MS, con tres capas:

| Tramo | Protección |
|---|---|
| Cliente → GW | HTTPS con Let's Encrypt (Caddy en la VM del GW) + JWT |
| MS → GW | HTTPS + `X-Service-Key` |
| GW → MS | Firewall (NSG o Security Group) que solo acepta la IP del GW + **mTLS**: el Caddy del MS exige el certificado de cliente del GW |

* **Una CA privada por entorno** (`test` y `prod`), en poder del equipo del GW. Cada CA firma dos tipos de certificado: el de cliente del GW, que solo sirve para `clientAuth`, y el de servidor de cada MS, que solo sirve para `serverAuth` y lleva la IP de su VM. Así un GW de prueba no entra a un MS de producción y el certificado de un MS no sirve como cliente. Los MS no usan Let's Encrypt porque sus puertos solo aceptan al GW.
* **Una VM, dos entornos:** producción en `443` y prueba en `8443`, detrás de un Caddy de borde por VM. Cada entorno tiene su propio stack y sus secretos (GitHub Environments `gw-prod` y `gw-test`).
* **Perfil `cloud`** (`application-cloud.yaml`, solo configuración): el certificado de cliente se carga con un *SSL bundle* de Spring Boot (`GW_CLIENT_KEYSTORE`), que se valida al arrancar, y el GW solo confía en la CA del entorno (`GW_TRUSTED_CA`).
* **TLS rechazado equivale a destino inalcanzable:** `502 SERVICIO_NO_DISPONIBLE` (RF-GW-01, A2).
* **Cortes de sesión:** la consulta a MS01 usa la misma conexión del GW hacia MS01, así que también viaja con mTLS.
* **Kit en el repositorio del GW:** `infra/mtls/` emite los certificados, `infra/edge/` es el borde del GW e `infra/caddy/` es la plantilla de borde que usa cada squad. La guía para los squads es [Conexión al API Gateway](../integration/gateway-connection.md).
* **Riesgos que quedan:** quien robe `gw-client.p12` o la llave de una CA puede hacerse pasar por el GW de ese entorno, y un MS con su clave de servicio puede propagar cualquier `X-User-*` (RF-GW-08). Los secretos se guardan solo en los GitHub Environments y en archivos `0600` de la VM.
* **Cambios de IP:** si cambia la IP del GW, cada squad actualiza su firewall. Si cambia la IP de un MS, se le emite un certificado nuevo y se cambia su `MSnn_URI`.
* **Vencimientos:** los certificados vencen a los 825 días y las CA a los 3 años. Una CA nueva invalida todos los certificados de su entorno.
