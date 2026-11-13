# ADR-003 — Validación local del JWT y cortes de sesión

**Estado:** Aceptada · **Fecha:** 2026-11-10 · **RF:** [RF-GW-02](../01-requerimientos-funcionales.md#rf-gw-02)

## Contexto
MS01 emite el JWT y el GW debe aceptarlo en cada solicitud protegida sin agregar latencia ni depender de MS01 en caliente (Estándares §3). Como el administrador gestiona los roles y sus permisos, un cambio de acceso (roles de un usuario, permisos de un rol o estado de una cuenta) debe alcanzar a las sesiones abiertas antes de que venza su token.

## Decisión
* **Política de acceso por método + ruta exacta**, en configuración (`gateway.access`): 5 rutas públicas y 3 internas, sin comodines (se valida al arrancar). Todo lo demás es protegido. La política solo se evalúa en rutas con destino; una ruta inexistente responde `404`.
* **RS256 con una sola llave pública** (`JWT_PUBLIC_KEY`, RSA ≥ 2048 bits) verificada localmente con Nimbus JOSE + JWT. El algoritmo se exige **antes** de verificar la firma, así que se rechazan `none`, `HS256` y cualquier otro.
* Se validan `iss`, `aud`, `exp`, `iat`, `nbf` (con tolerancia de reloj), `sub` UUID, `roles` no vacío con formato `^[A-Z][A-Z0-9_]{1,29}$`, `permissions` no vacío con formato `^ms[0-9]{2}:[a-z][a-z0-9_]*:[a-z][a-z0-9_]*$` y `exp − iat ≤ JWT_MAX_LIFETIME_SECONDS`. `kid`, `email` y `jti` se ignoran. Los roles y los permisos son datos de MS01: el GW valida su formato, no su existencia.
* **Sin JWKS, sin rotación y sin introspección por solicitud.**
* **Cortes de sesión con una instantánea en memoria.** Al arrancar y luego cada `SESSION_CUTS_INTERVAL_SECONDS` (4 s, con un timeout de `SESSION_CUTS_TIMEOUT_SECONDS`, 3 s), el GW llama a `GET SESSION_CUTS_URL` con `X-Service-Name: gateway` y reemplaza por completo dos tablas en memoria: el corte de cada usuario y el de cada rol. Validar un token solo consulta esas tablas: lo rechaza si su `iat` es estrictamente menor que el corte de `sub` o de alguno de sus `roles`; con un `iat` igual al corte lo acepta, porque ambos instantes se truncan a segundos.
* **Si MS01 no responde** (error, timeout o cuerpo inválido), el GW conserva la última instantánea, registra un `WARN` y no bloquea el tráfico.
* La consulta sale del propio GW, por la misma conexión que las rutas hacia MS01 (mTLS en la nube, [ADR-006](ADR-006-aislamiento-en-la-nube.md)); no usa clave de servicio y su ruta no se publica a través del GW. Los cortes no se aplican a las llamadas internas con clave de servicio, que no llevan JWT ([ADR-005](ADR-005-llamadas-internas.md)).
* Todo rechazo es `401 NO_AUTENTICADO` con el mismo mensaje, también el de un corte, para no revelar el motivo; más de una cabecera `Authorization` es `400 VALIDACION`. El motivo va al log en `DEBUG` y el token nunca se registra.

## Consecuencias
* Validar un token no genera tráfico de red: si MS01 cae, los tokens vigentes siguen funcionando con los últimos cortes conocidos.
* Un cambio de roles, de permisos de un rol o de estado de una cuenta invalida las sesiones afectadas en el siguiente ciclo de consulta (7 s como máximo con los valores por defecto: 4 s de intervalo más 3 s de timeout). Mientras MS01 no responde no se conocen cortes nuevos. El logout y el cambio de contraseña no cortan la sesión: el token sigue válido hasta vencer (máximo 60 min).
* El `iat` del token y los cortes salen de la hora de la base de datos de MS01, un solo reloj para todas sus instancias: el GW solo los compara y no depende de que esas instancias estén sincronizadas.
* La instantánea cubre los últimos 3700 s, más que la vigencia máxima de un token (3600 s más la tolerancia de reloj): un corte más antiguo ya no afecta a ningún token vigente.
* Cada instancia del GW mantiene su propia instantánea y consulta a MS01 por su cuenta.
* Cambiar la llave exige desplegar MS01 y el GW juntos e invalida los tokens vigentes.
* Agregar una ruta pública exige cambiar los Estándares §2 y `gateway.access` en el mismo PR.
