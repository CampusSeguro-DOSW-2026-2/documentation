# ADR-004 — CORS centralizado en el Gateway

**Estado:** Aceptada · **Fecha:** 2026-10-25 · **RF:** [RF-GW-04](../01-requerimientos-funcionales.md#rf-gw-04)

## Contexto
El frontend corre en el navegador con otro origen. Si cada MS configurara CORS habría políticas distintas, y el preflight `OPTIONS`, que no lleva JWT, llegaría a los MS.

## Decisión
* **CORS solo en el GW** (Estándares §2.1); los MS no lo configuran.
* **Orígenes por ambiente** (`CORS_ALLOWED_ORIGINS`, obligatoria; el perfil `local` trae el de Vite). Métodos, cabeceras permitidas (`Authorization`, `Content-Type`, `Accept`, `X-Correlation-ID`) y la expuesta (`X-Correlation-ID`) son fijos. Las cabeceras internas (`X-User-*`, `X-Service-*`) nunca se permiten al navegador.
* **Prohibido `*` con credenciales**, y prohibido un origen con ruta o barra final: el GW no arranca.
* **El preflight se resuelve en el GW** con el soporte CORS del mapeo de rutas de Spring Cloud Gateway. Se evalúa después de resolver la ruta (ruta inexistente → `404`) y antes de los filtros (no exige JWT ni llama al MS).
  * Preflight no permitido: `403 PROHIBIDO` con `ErrorResponse`.
  * Solicitud real desde un origen no permitido: se procesa sin cabeceras CORS y el navegador bloquea la lectura.
* El GW no sirve archivos estáticos (`spring.web.resources.add-mappings: false`).

## Consecuencias
* Agregar un origen exige solo configuración y reinicio.
* CORS no sustituye la autenticación: una solicitud real desde un origen permitido sigue exigiendo JWT.
* Un MS que agregue cabeceras CORS propias duplicaría la política; está prohibido por los Estándares.
