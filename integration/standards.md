# Estándares de Integración y Contratos Transversales — CampusSeguro

**Ámbito:** obligatorio para el API Gateway (GW) y MS01–MS08.
**Fuente única:** los valores transversales (puertos, JWT, cabeceras, errores, timeouts, correlación, paginación, auditoría) se definen **solo aquí**. Los documentos de cada squad enlazan a este documento en lugar de copiar sus valores.
**Cambios:** se acuerdan entre los squads afectados antes de modificar este documento.

Los términos **DEBE** y **NO DEBE** son obligatorios.

---

## 1. Valores globales

| Clave | Valor |
|---|---|
| Puerto externo del GW | `8080` (el único publicado en Docker local). En la nube, `443` (producción) y `8443` (prueba) a través de su Caddy. |
| Prefijo funcional | `/api/v1` |
| Algoritmo JWT | `RS256` (RSA 2048) |
| `iss` / `aud` | `ms01-identity-and-access` / `campusseguro` |
| Vigencia del JWT | `3600 s` (60 min) |
| Tolerancia de reloj | `30 s` |
| Consulta de cortes de sesión (GW → MS01) | Cada `4 s`, con timeout de `3 s` (un corte se aplica en `7 s` como máximo); la instantánea cubre los últimos `3700 s` |
| Timeout GW → MS | `3000 ms`. Excepción: `/api/v1/tutor/**` (MS07) espera la respuesta hasta `10000 ms`, y MS07 corta su llamada al modelo de lenguaje a los `8000 ms` para responder antes que el GW |
| Timeout del cliente HTTP en llamadas internas | `2000 ms` |
| Correlación | UUID v4 en `X-Correlation-ID` |
| Paginación | `page` desde 0; `size` 20 por defecto (1–100) |
| Healthcheck | `GET /actuator/health`, `≤ 200 ms` |

---

## 2. Topología, puertos y enrutamiento

* **Puerta única:** todo el tráfico externo (frontend, Postman, etc.) **e interno** (MS → MS) **DEBE** pasar por el GW. Los microservicios **NO DEBEN** llamarse directamente entre sí.
* **Aislamiento:** solo el GW recibe tráfico de afuera.
  * **Docker local:** solo se publica el puerto del GW y los MS operan en la red interna.
  * **Nube (cada servicio en su propia VM):** el firewall de cada MS solo acepta la IP del GW, y su Caddy exige el certificado de cliente del GW (mTLS). Todo viaja por HTTPS. Cómo conectar un MS: [Conexión al API Gateway](gateway-connection.md).
* **Destinos por ambiente:** las URI se configuran con variables de entorno. Está prohibido hardcodear URLs o usar `localhost` para llegar a otro contenedor. En la nube, el destino de un MS es `https://<IP de su VM>:443` en producción y `:8443` en prueba.

| Servicio | Nombre DNS interno (= repositorio) | Puerto |
|---|---|---|
| GW | `api-gateway` | `8080` (externo) · `9090` gestión, solo interno |
| MS01 | `ms01-identity-and-access` | `8081` |
| MS02 | `ms02-campus-and-security` | `8082` |
| MS03 | `ms03-alerts-and-emergencies` | `8083` |
| MS04 | `ms04-evacuation-and-check-in` | `8084` |
| MS05 | `ms05-brigades-and-incidents` | `8085` |
| MS06 | `ms06-drills` | `8086` |
| MS07 | `ms07-training-and-ai-assistant` | `8087` |
| MS08 | `ms08-analytics-and-audit` | `8088` |

* **Rutas:** todo endpoint funcional vive bajo `/api/v1/`. El GW reenvía método, ruta, query params y cuerpo **sin modificarlos**; solo cambia las cabeceras técnicas.
* **Contrato primero:** el GW solo enruta lo que está publicado en el `api/openapi.yml` de la carpeta del MS en este repositorio. Está prohibido inventar rutas o prefijos. Excepción: las rutas del tag *Interno (Gateway)* de un OpenAPI (hoy `GET /api/v1/internal/session-cuts` de MS01) las consume el propio GW y no se publican a través de él.
* **Política de acceso por ruta:**

| Política | Quién puede llamar | Credencial exigida por el GW |
|---|---|---|
| **Pública** | Cualquiera | Ninguna |
| **Protegida** | Usuarios y MS internos | JWT válido **o** clave de servicio |
| **Interna** | Solo MS internos | Clave de servicio |

* **Rutas públicas (lista cerrada, método + ruta exacta).** Está prohibido usar comodines como `/api/v1/auth/**`:

| Método | Ruta | Servicio |
|---|---|---|
| `POST` | `/api/v1/auth/register` | MS01 |
| `POST` | `/api/v1/auth/register/verify` | MS01 |
| `POST` | `/api/v1/auth/register/resend` | MS01 |
| `POST` | `/api/v1/auth/login` | MS01 |
| `POST` | `/api/v1/auth/password/forgot` | MS01 |
| `POST` | `/api/v1/auth/password/reset` | MS01 |
| `GET` | `/api/v1/public/spaces` | MS02 (consulta por QR, sin datos personales) |
| `GET` | `/api/v1/public/exits` | MS02 (consulta por QR, sin datos personales) |
| `GET` | `/api/v1/public/safe-zones` | MS02 (consulta por QR, sin datos personales) |
| `GET` | `/actuator/health` | GW (propio, no se enruta) |

* **Swagger/OpenAPI de los servicios:** habilitado solo en desarrollo local. No se expone a través del GW.

### 2.1 CORS

* Se gestiona **únicamente en el GW**. Los MS **NO DEBEN** configurar CORS ni usar `@CrossOrigin`.
* Los orígenes se configuran por ambiente. Está prohibido usar `*` cuando hay credenciales.
* Métodos: `GET`, `POST`, `PUT`, `PATCH`, `DELETE`, `OPTIONS`. Cabeceras permitidas: `Authorization`, `Content-Type`, `Accept`, `X-Correlation-ID`. Cabecera expuesta: `X-Correlation-ID`.
* El preflight `OPTIONS` lo resuelve el GW: no exige JWT ni invoca al MS.

---

## 3. Autenticación de usuarios (JWT)

**Flujo:** el usuario inicia sesión **una vez** en MS01 (`POST /auth/login`) y recibe un JWT con vigencia de 60 min. En cada petición lo envía como `Authorization: Bearer <JWT>`. El GW verifica la firma **localmente** con la llave pública (no consulta a MS01 en cada petición; solo consulta cada pocos segundos los cortes de sesión, ver más abajo) e inyecta la identidad en cabeceras. Al expirar el token, o cuando un corte de sesión lo invalida, el usuario vuelve a iniciar sesión (no hay refresh token).

| Componente | Responsabilidad |
|---|---|
| MS01 | **Único emisor** del JWT. Firma con la llave privada. |
| GW | **Único verificador** (firma y claims) y quien aplica los cortes de sesión que publica MS01. No crea ni renueva tokens. |
| MS01–MS08 | **NO DEBEN** recibir ni validar el JWT. Confían en las cabeceras del GW. |

**Contenido del token:**

| Elemento | Regla |
|---|---|
| Header `alg` | `RS256`. El GW rechaza cualquier otro valor (incluido `none` y los algoritmos simétricos). |
| Header `kid` | Informativo; el GW no lo usa en esta versión. |
| `iss` / `aud` | Ver valores globales. |
| `sub` | UUID del usuario (el mismo `id` de la API de MS01). |
| `roles` | Arreglo no vacío de nombres de rol, en orden alfabético. Cada nombre usa mayúsculas, dígitos y guion bajo, empieza con una letra y tiene de 2 a 30 caracteres. Los roles son datos de MS01 y ya no son un conjunto fijo. Un usuario tiene como máximo 50 roles. |
| `permissions` | Arreglo no vacío de permisos con formato `ms<dos dígitos>:<recurso>:<acción>`: la unión de los permisos de todos los roles del usuario al emitir el token, sin repetidos y en orden alfabético. |
| `iat` / `exp` | Epoch en segundos. `exp − iat ≤ 3600`. |
| `email`, `jti` | MS01 los emite; el GW no los usa. |

* El GW rechaza con `401 NO_AUTENTICADO` todo token ausente, mal formado o expirado, con firma inválida, `iss`/`aud` incorrectos, `sub` que no sea UUID, `roles` o `permissions` ausente, vacío o con formato inválido, vigencia mayor a 3600 s, o un `iat` anterior al corte de sesión del usuario o de alguno de sus roles. Aplica la tolerancia de reloj de los valores globales.
* **Corte de sesión:** un cambio de los roles de un usuario, de los permisos de un rol o del estado de una cuenta corta las sesiones afectadas. MS01 registra el corte (`valido_desde`) y el GW lo consulta cada 4 s (7 s como máximo hasta que se aplica; [RF-MS01-11](../ms01-identity-and-access/1_requirements.md#rf-ms01-11-corte-de-sesión-por-cambios-de-acceso), [detalle técnico](../ms01-identity-and-access/jwt-spec.md)): rechaza con `401 NO_AUTENTICADO`, con el mensaje genérico de siempre, todo token con `iat` estrictamente menor que el corte del usuario o de alguno de sus roles. El `iat` del token y el corte salen de la hora de la base de datos de MS01, un solo reloj para todas sus instancias: el GW solo los compara. Si MS01 no responde, el GW conserva los últimos cortes y sigue aceptando los tokens válidos. El logout y el cambio de contraseña no cortan la sesión: el token sigue válido hasta vencer (máximo 60 min).
* **Llaves:** un solo par RSA. La privada solo está en MS01 (`JWT_PRIVATE_KEY`) y la pública en el GW (`JWT_PUBLIC_KEY`), ambas fuera del repositorio. **Sin rotación en esta versión:** cambiar de llave exige desplegar MS01 y el GW juntos, e invalida los tokens vigentes.

---

## 4. Cabeceras técnicas

| Cabecera | Dirección | Regla |
|---|---|---|
| `Authorization` | Cliente → GW | `Bearer <JWT>`. El GW **la elimina** antes de reenviar, incluso en rutas públicas. |
| `X-User-Id` | GW → MS | `sub` del JWT o, en llamadas internas, el valor propagado por el MS llamante. |
| `X-User-Role` | GW → MS | Roles separados por coma, sin espacios, en orden alfabético. Ej.: `BRIGADISTA,ESTUDIANTE`. Informativa: para autorizar se usa `X-User-Permission` (§6). |
| `X-User-Permission` | GW → MS | Permisos separados por coma, sin espacios, en orden alfabético: el claim `permissions`. Ej.: `ms04:checkin:write,ms07:content:read`. |
| `X-Correlation-ID` | Transversal | UUID v4. |
| `X-Service-Key` | MS → GW | Clave de servicio. El GW **la elimina** y nunca la reenvía. |
| `X-Service-Name` | GW → MS | Solo en llamadas internas: `ms01`…`ms08`, derivado de la clave. |

1. **Sanitización:** en toda petición externa, el GW elimina (sin distinguir mayúsculas/minúsculas) cualquier `X-User-Id`, `X-User-Role`, `X-User-Permission` o `X-Service-Name` enviado por el cliente.
2. **Rutas públicas:** el GW no inyecta identidad.
3. **Rutas protegidas e internas:** el MS **DEBE** exigir `X-User-Id`, `X-User-Role` y `X-User-Permission` cuando su operación requiera usuario. Si faltan, responde `401 NO_AUTENTICADO`.
4. **OpenAPI de cada MS:** documenta estas cabeceras como `parameters` y **NO DEBE** declarar `bearerAuth`.
5. **Pruebas locales sin GW:** se envían `X-User-Id`, `X-User-Role` y `X-User-Permission` manualmente (Postman, cURL).

---

## 5. Llamadas internas (MS → GW → MS)

El GW es el orquestador de todo el tráfico, así que toda llamada entre servicios queda trazada y sujeta a las mismas reglas.

1. Cada MS tiene una clave secreta propia `GATEWAY_SERVICE_KEY` (aleatoria, ≥ 32 caracteres, en una variable de entorno). El GW conoce la clave de cada MS (`GW_SERVICE_KEY_MS01` … `GW_SERVICE_KEY_MS08`).
2. El MS llama a `${GATEWAY_URL}/api/v1/...` (en la nube, `https://<dominio del GW>` en producción y `https://<dominio del GW>:8443` en prueba) con:
   * `X-Service-Key`: su clave.
   * `X-Correlation-ID`: el mismo que recibió.
   * `X-User-Id`, `X-User-Role` y `X-User-Permission`: los que recibió, sin modificarlos, si actúa en nombre de un usuario.
3. El GW valida la clave (comparación en tiempo constante), elimina `X-Service-Key`, inyecta `X-Service-Name`, conserva los `X-User-*` enviados (si tienen formato válido) y **no exige JWT**.
   * Clave inválida: `401 NO_AUTENTICADO`.
   * `Authorization` y `X-Service-Key` en la misma petición: `400 VALIDACION`.
4. El MS destino autoriza así:
   * **Ruta protegida:** por `X-User-Permission` contra el `x-permiso` de la operación.
   * **Ruta interna:** por `X-Service-Name`.
5. El MS llamante usa un timeout de cliente de `2000 ms` y no reintenta operaciones no idempotentes. Si la llamada falla, responde con su propio `ErrorResponse` y nunca reenvía tal cual el cuerpo del otro servicio.

**Llamadas internas registradas.** Toda llamada nueva se acuerda entre los squads involucrados y se agrega aquí. En una llamada a una ruta **protegida** el MS llamante propaga la identidad del usuario por el que actúa (o una identidad de sistema) y el destino autoriza con `X-User-Permission`; en una **interna** el destino autoriza con `X-Service-Name`.

| Origen → Destino | Propósito | Rutas | Identidad propagada | Política en el destino |
|---|---|---|---|---|
| MS01…MS07 → MS08 | Eventos de auditoría (§12) | `POST /api/v1/audit/events` | Usuario si existe; si no, ninguna | Interna |
| MS03 → MS01 | Resolver destinatarios de alertas (usuarios por id y por rol) | `POST /api/v1/users/batch`, `GET /api/v1/users/by-role` | Ninguna | Interna (`X-Service-Name: ms03`) |
| MS04 → MS01 | Resolver la población objetivo de un evento (usuarios por rol) y los nombres de los participantes (usuarios por id) | `GET /api/v1/users/by-role`, `POST /api/v1/users/batch` | Usuario que hace la consulta, si existe | Interna (`X-Service-Name: ms04`) |
| MS05 → MS01 | Consultar los usuarios con rol BRIGADISTA y sus datos | `GET /api/v1/users/by-role`, `POST /api/v1/users/batch` | Ninguna | Interna (`X-Service-Name: ms05`) |
| MS03 → MS02 | Validar los edificios y las zonas seguras que alcanza una alerta | `GET /api/v1/buildings/{id}`, `GET /api/v1/safe-zones/{id}` | Usuario que crea la alerta | Protegida |
| MS04 → MS02 | Consultar el punto de encuentro (zona segura) de un evento | `GET /api/v1/safe-zones/{id}` | Usuario que consulta | Protegida |
| MS05 → MS02 | Consultar la ubicación de un incidente (edificio y zona segura) | `GET /api/v1/buildings/{id}`, `GET /api/v1/safe-zones/{id}` | Usuario que registra el incidente | Protegida |
| MS06 → MS02 | Consultar el escenario de un evento (edificio, zona segura y rutas de evacuación) | `GET /api/v1/buildings/{id}`, `GET /api/v1/safe-zones/{id}`, `GET /api/v1/buildings/{id}/evacuation-routes` | Usuario organizador | Protegida |
| MS04 → MS05 | Crear la solicitud de asistencia desde el check-in y consultar las zonas de responsabilidad del brigadista | `POST /api/v1/incidentes/eventos/{eventoId}` (`ms05:incident:report`), `GET /api/v1/responsabilidades/mias` | Usuario que hace el check-in o brigadista | Protegida |
| MS05 → MS03 | Crear la alerta que origina un incidente | `POST /api/v1/alertas` | Usuario que registra el incidente | Protegida |
| MS06 → MS03 | Crear, programar, activar y archivar la alerta de un simulacro | `POST /api/v1/alertas`, `POST /api/v1/alertas/{id}/programar`, `/activar` y `/archivar` | Usuario ORGANIZADOR | Protegida (`ms03:alert:manage_drill`) |
| MS03 → MS06 | Replicar el catálogo de tipos de alerta | `GET /api/v1/tipos-alerta` | Identidad de sistema | Interna (`X-Service-Name: ms03`) |
| MS06 → MS04 | Informar el estado del evento y consultar el consolidado y los check-ins | `/api/v1/chi/eventos/{eventoId}/estado`, `/consolidado` y `/checkins` | Identidad de sistema (`ms04:dashboard:read`) | Protegida |
| MS06 → MS05 | Informar el estado del evento y consultar los brigadistas y los indicadores | `/api/v1/bei/eventos/{eventoId}/estado`, `/brigadistas` y `/indicadores` | Ninguna | MS05 autoriza por `X-Service-Name: ms06` |
| GW → MS01 | Consultar cada 4 s los cortes de sesión | `GET /api/v1/internal/session-cuts` | Ninguna | Interna, solo para el GW (`X-Service-Name: gateway`) |

La llamada GW → MS01 sale del propio GW hacia MS01, por la misma conexión mTLS con la que le reenvía las peticiones: no pasa por el flujo de claves de servicio de arriba, porque el GW es el origen. El GW fija `X-Service-Name: gateway`, MS01 solo acepta ese valor en esa ruta y la ruta no se publica a través del GW.

---

## 6. Roles y permisos

* **Roles iniciales (en MAYÚSCULAS):** `ESTUDIANTE`, `PROFESOR`, `ADMINISTRATIVO`, `BRIGADISTA`, `COORDINADOR`, `ORGANIZADOR`, `ADMIN`, `AUDITOR`. `ADMIN` es el identificador del actor *Administrador*. MS01 los crea al desplegar; para ellos están prohibidos los alias y las variaciones.
* **Roles en ejecución:** los roles son datos de MS01. Con el permiso `ms01:role:manage` (exclusivo de `ADMIN`) el administrador crea roles, los edita, cambia sus permisos y borra o renombra los que no son de sistema (MS01 recorta los espacios del nombre y lo pasa a mayúsculas); `ESTUDIANTE` y `ADMIN` son de sistema y no se pueden borrar ni renombrar ([RF-MS01-10](../ms01-identity-and-access/1_requirements.md#rf-ms01-10-administración-de-roles-y-catálogo-de-permisos)). La lista vigente la da `GET /api/v1/roles/details` de MS01. Por eso ningún MS debe asumir un conjunto fijo de nombres de rol.
* **Matriz y catálogo de permisos:** la [matriz de roles y permisos](permissions-matrix.md) es el registro oficial de los códigos de permiso (`<ms>:<recurso>:<acción>`) y de su asignación **inicial** a los roles iniciales. El catálogo de permisos vive en la base de datos de MS01 y es de solo lectura (ninguna ruta lo modifica); lo que cada rol tiene en ejecución es dato de MS01 y lo edita el administrador. Un usuario con varios roles tiene la unión de sus permisos.
* **OpenAPI:** cada operación protegida declara `x-permiso`, que **DEBE** ser un código de la matriz. `x-roles` es opcional e informativo (los roles iniciales que tienen el permiso al desplegar): no se compara con nada ni se usa para autorizar.

```yaml
x-permiso: ms02:route_status:write   # un código, o una lista si la operación usa varios
x-roles: [COORDINADOR, ADMIN]         # opcional, informativo
```

* La autorización la implementa cada MS en su código, con los permisos de `X-User-Permission` (ya son la unión de los de todos los roles del usuario). Si falta el permiso exigido, la respuesta es `403 PROHIBIDO`. El MS no consulta a MS01 para autorizar.

---

## 7. Errores

### 7.1 Esquema `ErrorResponse`

Toda respuesta `4xx`/`5xx` del GW o de un MS **DEBE** tener exactamente estos cinco campos, todos obligatorios y nunca `null`:

```json
{
  "status": 400,
  "codigo": "VALIDACION",
  "mensaje": "Datos inválidos.",
  "errores": [{ "campo": "primer_nombre", "mensaje": "El primer nombre debe tener entre 2 y 30 letras." }],
  "correlation_id": "c19b472d-7a2a-4d3b-a12b-abcdef123456"
}
```

| Campo | Regla |
|---|---|
| `status` | Igual al código HTTP de la respuesta. |
| `codigo` | Código del catálogo de estados y códigos, en `MAYUSCULAS_CON_GUION_BAJO`. |
| `mensaje` | Texto seguro en español. |
| `errores` | `[{campo, mensaje}]`; `[]` si no hay errores por campo. |
| `correlation_id` | Idéntico al `X-Correlation-ID` recibido. |

Está prohibido agregar campos (como `timestamp` o `error`) y exponer stack traces, secretos, tokens, hashes, IPs o nombres de contenedores.

### 7.2 Catálogo de estados y códigos

| HTTP | `codigo` | Uso | Lo genera |
|---|---|---|---|
| 400 | `VALIDACION` | Formato o campos inválidos | GW / MS |
| 400 | `CODIGO_INVALIDO` | Código de recuperación o de verificación del registro inválido o vencido | MS01 |
| 401 | `NO_AUTENTICADO` | JWT o clave de servicio inválidos; cabeceras de identidad ausentes | GW / MS |
| 401 | `CREDENCIALES_INCORRECTAS` | Login fallido | MS01 |
| 403 | `PROHIBIDO` | Sin permiso; preflight CORS rechazado | GW / MS |
| 403 | `CUENTA_INACTIVA` | Login de una cuenta inactiva | MS01 |
| 404 | `NO_ENCONTRADO` | Recurso o ruta inexistente | GW / MS |
| 405 | `METODO_NO_PERMITIDO` | Verbo no soportado por la ruta | MS |
| 409 | `CONFLICTO` | Duplicado o conflicto de estado (p. ej. nombre de rol repetido, renombrar o borrar un rol de sistema) | MS |
| 415 | `TIPO_NO_SOPORTADO` | `Content-Type` no soportado | MS |
| 422 | `REGLA_NEGOCIO` | Viola una regla de negocio (p. ej. dar `ms01:role:manage` a un rol que no es ADMIN) | MS |
| 500 | `ERROR_INTERNO` | Falla no controlada | GW / MS |
| 502 | `SERVICIO_NO_DISPONIBLE` | MS inalcanzable o error downstream no conforme | GW |
| 504 | `TIEMPO_AGOTADO` | El MS supera el timeout del GW | GW |

Un MS solo puede usar códigos adicionales si los declara en su OpenAPI.

### 7.3 Reglas para los microservicios y el GW

* **MS:** también los errores del framework **DEBEN** salir como `ErrorResponse`: ruta inexistente (404), método no soportado (405), `Content-Type` (415), tipo de parámetro inválido (400) y cabecera de identidad ausente (401). Esto se resuelve con un manejador global de excepciones.
* **GW:** propaga sin cambios las respuestas `2xx`/`3xx`. Una respuesta `4xx`/`5xx` de un MS es **válida** si es JSON, tiene los cinco campos, `status` coincide con el código HTTP y `correlation_id` coincide con el de la petición:
  * **Válida:** se propaga sin cambios.
  * **Inválida:** el GW responde `502 SERVICIO_NO_DISPONIBLE` con un mensaje genérico y deja el detalle solo en su log.

---

## 8. Correlación y logs

* **`X-Correlation-ID`:** el GW conserva el valor recibido si es un UUID v4 válido y único; si no, genera uno nuevo (la correlación nunca causa un rechazo). Lo envía al MS y lo devuelve en **toda** respuesta.
* **Cada MS DEBE:**
  * leer el valor e incluirlo en cada línea de log;
  * devolverlo en `correlation_id`;
  * propagarlo en las llamadas internas y en los eventos de auditoría.
* **Campos mínimos del log del GW:** `timestamp`, `level`, `correlation_id`, `method`, `path`, `status`, `duration_ms`, `upstream` (`ms01`…`ms08` o `gw`) y `origen` (`externo` o `ms01`…`ms08`). Nivel: `INFO` para 2xx/3xx, `WARN` para 4xx, `ERROR` para 5xx.
* **Campos mínimos del log de un MS:** `timestamp`, `level`, `correlation_id`, `mensaje`.
* **Prohibido registrar:** tokens, `Authorization`, `X-Service-Key`, contraseñas, códigos de recuperación y cuerpos con datos personales.

---

## 9. Paginación

Colecciones: query params `page` (desde 0, por defecto 0) y `size` (por defecto 20, mínimo 1, máximo 100). Está prohibido usar `limit`/`offset`. Respuesta:

```json
{ "contenido": [], "page": 0, "size": 20, "total_elementos": 150, "total_paginas": 8 }
```

## 10. Inactivación lógica

Las entidades con estado `ACTIVO`/`INACTIVO` se inactivan cambiando su estado, no con un `DELETE` físico, salvo que el dominio diga lo contrario.

## 11. Healthcheck y Actuator

* **GW:** `GET /actuator/health` es público y responde `200 {"status":"UP"}` (o `503 {"status":"DOWN"}` si el propio GW no está sano) en `≤ 200 ms`, sin detalles y sin consultar a otros servicios. El resto de endpoints de Actuator (métricas, etc.) solo se exponen en el puerto de gestión interno `9090`, nunca a través de `8080`.
* **MS:** `GET /actuator/health` solo en la red interna (el GW no lo enruta); en la nube se consulta desde la propia VM. Responde `200 {"status":"UP"}` o `503 {"status":"DOWN"}`, sin cascada a otros servicios ni detalles.

## 12. Auditoría (MS08)

* El GW **no** audita. Cada MS emite los eventos de las acciones de su dominio. MS08 es el único que almacena y consulta la auditoría (las acciones del propio MS08 las registra él mismo, sin pasar por el GW).
* Los mismos eventos alimentan los indicadores de MS08: toda acción que ocurre dentro de un simulacro o emergencia registrada como evento en MS06 **DEBE** identificar el evento al que pertenece (`evento_id`, `tipo_evento`). Una emergencia real que no está registrada como evento en MS06 (p. ej. una alerta manual de MS03) se envía con `evento_id = null` y `tipo_evento = REAL`.
* **Transporte:** `POST /api/v1/audit/events` a través del GW (ruta interna), asíncrono y sin esperar el resultado. MS08 responde `202`. La respuesta al usuario no cambia nunca por un fallo de auditoría. Los MS **NO DEBEN** llamar a MS08 directamente (§2).
* **Cabeceras del emisor:** `X-Service-Key`, `X-Correlation-ID` y `Content-Type: application/json`. Si la acción la origina un usuario, también `X-User-Id`, `X-User-Role` y `X-User-Permission`, las tres juntas y tal como las recibió, sin modificarlas (§5); si no, ninguna. Si llega solo una o dos, el GW responde `400 VALIDACION` y el evento no llega a MS08. Este endpoint no exige `X-Service-Name` ni JWT: el emisor se identifica por la clave de servicio y por `modulo`.
* **Reintentos (opcionales):** un MS puede reintentar, o solo registrar el fallo en su log. Si reintenta, **DEBE** enviar siempre el mismo `id` en todos los intentos, reintentar solo ante timeout, `503` o `5xx` (nunca ante `4xx`), con espera creciente y un máximo de 5 intentos y 24 horas, y sin bloquear nunca la operación del usuario. La cola solo puede usarse para reenviar eventos de auditoría. MS08 es idempotente por `id`, así que el reintento cumple la regla de §5 sobre operaciones no idempotentes. MS08 ordena por `fecha_hora`, no por orden de llegada.
* **Cuerpo del evento:**

| Campo | Tipo | Regla |
|---|---|---|
| `id` | UUID | Identificador del evento de auditoría, generado por el emisor. Obligatorio si el emisor reintenta; recomendado siempre. Si falta, MS08 calcula una huella con `correlation_id`, `modulo`, `accion`, `recurso_afectado` y `fecha_hora`. |
| `usuario_id` | UUID \| null | Usuario que ejecuta la acción; `null` si es anónimo (p. ej. login con correo inexistente). La clave siempre va. |
| `modulo` | string | `MS01` … `MS08`. |
| `accion` | string | `MAYUSCULAS_CON_GUION_BAJO`. Cada MS publica su catálogo de acciones en el [catálogo de auditoría](audit-catalog.md). |
| `recurso_afectado` | string \| null | `<tipo>:<id>`, p. ej. `usuario:3f6c2a9e-…`; `null` solo si no hay un recurso identificable. La clave siempre va. |
| `resultado` | string | `EXITO` o `FALLO`. |
| `fecha_hora` | string | ISO 8601 UTC del momento en que ocurrió la acción (no del reintento). |
| `correlation_id` | UUID | El de la petición. |
| `detalle` | string \| null | Opcional: código genérico del motivo (p. ej. `CREDENCIALES_INCORRECTAS`). Sin texto libre. |
| `evento_id` | UUID \| null | Id del evento (simulacro o emergencia) de MS06 al que pertenece la acción; `null` si no pertenece a ninguno (login, gestión de usuarios, alerta manual…). |
| `tipo_evento` | string \| null | `SIMULACRO` o `REAL`. **Obligatorio si `evento_id` no es `null`.** Con `evento_id = null` solo puede ser `REAL` (emergencia real sin evento de MS06) o `null` (acción administrativa). |
| `atributos` | objeto \| null | Datos para indicadores. Objeto plano (sin anidar), claves en `snake_case`, valores `string` (máx. 100 caracteres), número o booleano, máximo 20 claves. Solo ids, códigos y conteos. Cada MS documenta sus claves en el [catálogo de auditoría](audit-catalog.md). |

Está prohibido incluir contraseñas, hashes, tokens, códigos de recuperación o datos personales distintos de los ids.

* **Validación en MS08:** `modulo` entre `MS01` y `MS08`; `accion` con formato `^[A-Z][A-Z0-9_]*$`; `resultado` solo `EXITO` o `FALLO`; `tipo_evento` solo `SIMULACRO` o `REAL`, obligatorio si hay `evento_id` y, sin `evento_id`, solo `REAL`; `fecha_hora` en UTC y no más de 5 minutos en el futuro ni más antigua que la retención; claves de primer nivel desconocidas se rechazan.
* **Respuestas:**

| HTTP | `codigo` | Cuándo |
|---|---|---|
| 202 | — | MS08 aceptó el evento (también si ya existía el mismo `id`). Sin cuerpo. |
| 400 | `VALIDACION` | El cuerpo no cumple la validación. |
| 401 | `NO_AUTENTICADO` | Falta la clave de servicio o es inválida (lo responde el GW). |
| 403 | `PROHIBIDO` | La clave no está autorizada para la ruta. |
| 503 | `SERVICIO_NO_DISPONIBLE` | MS08 no puede guardar en este momento. |

* **Consulta:** las rutas de consulta son **Protegidas** y se autorizan con `X-User-Permission`: `ms08:audit_drill:read` permite ver los registros con `tipo_evento = SIMULACRO` y `ms08:audit_real_event:read` los de `tipo_evento = REAL`. Los registros con `tipo_evento = REAL` y sin `evento_id` (emergencia real sin evento de MS06) los ve quien tiene `ms08:audit_real_event:read`. Los registros sin evento y sin tipo (`tipo_evento = null`, administrativos) solo los ve quien tiene ambos permisos. La auditoría es de solo lectura: MS08 no expone rutas para modificar ni borrar registros. Los códigos y su asignación inicial están en la [matriz de roles y permisos](permissions-matrix.md).
* **Retención:** los registros de auditoría se conservan 6 meses desde `fecha_hora` y luego se eliminan. Los indicadores por evento (agregados, sin datos personales) se conservan.
* **Guía para los squads:** cómo emitir, qué auditar y cómo documentar las acciones en el catálogo: [Auditoría: guía de integración](../ms08-analytics-and-audit/integration-guide.md).
