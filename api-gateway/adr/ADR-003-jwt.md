# ADR-003 — Validación local y sin estado del JWT

**Estado:** Aceptada · **Fecha:** 2026-10-25 · **RF:** [RF-GW-02](../01-requerimientos-funcionales.md#rf-gw-02)

## Contexto
MS01 emite el JWT y el GW debe aceptarlo en cada solicitud protegida sin agregar latencia ni depender de MS01 en caliente (Estándares §3).

## Decisión
* **Política de acceso por método + ruta exacta**, en configuración (`gateway.access`): 5 rutas públicas y 1 interna, sin comodines (se valida al arrancar). Todo lo demás es protegido. La política solo se evalúa en rutas con destino; una ruta inexistente responde `404`.
* **RS256 con una sola llave pública** (`JWT_PUBLIC_KEY`, RSA ≥ 2048 bits) verificada localmente con Nimbus JOSE + JWT. El algoritmo se exige **antes** de verificar la firma, así que se rechazan `none`, `HS256` y cualquier otro.
* Se validan `iss`, `aud`, `exp`, `iat`, `nbf` (con tolerancia de reloj), `sub` UUID, `roles` no vacío con formato `^[A-Z][A-Z0-9_]*$` y `exp − iat ≤ JWT_MAX_LIFETIME_SECONDS`. `kid`, `email` y `jti` se ignoran.
* **Sin JWKS, sin rotación, sin introspección ni revocación.**
* Todo rechazo es `401 NO_AUTENTICADO`; más de una cabecera `Authorization` es `400 VALIDACION`. El motivo va al log en `DEBUG` y el token nunca se registra.

## Consecuencias
* Validar un token no genera tráfico de red: si MS01 cae, los tokens vigentes siguen funcionando.
* Un cambio de rol, una inactivación o un logout se aplican al siguiente token: la ventana máxima es la vigencia (60 min).
* Cambiar la llave exige desplegar MS01 y el GW juntos e invalida los tokens vigentes.
* Agregar una ruta pública exige cambiar los Estándares §2 y `gateway.access` en el mismo PR.
