# ADR-005 — Llamadas internas a través del Gateway con clave de servicio

**Estado:** Aceptada · **Fecha:** 2026-10-25 · **RF:** [RF-GW-08](../01-requerimientos-funcionales.md#rf-gw-08)

## Contexto
Algunos MS llaman a otros (p. ej. MS04 → MS05, MS01…MS07 → MS08). Si se llamaran directamente, esas llamadas quedarían fuera de la correlación, los logs, el `ErrorResponse` y la autenticación del GW (Estándares §5).

## Decisión
* **Todo el tráfico MS → MS pasa por el GW**, en `${GATEWAY_URL}/api/v1/...`.
* **Una clave por MS** (`GW_SERVICE_KEY_MS01` … `_MS08`, ≥ 32 caracteres, sin repetidas; se valida al arrancar sin mostrar las claves). El GW compara resúmenes SHA-256 en tiempo constante contra todas las claves y obtiene el llamante.
* **Políticas:** una ruta interna solo acepta clave; una protegida acepta JWT **o** clave. `Authorization` y `X-Service-Key` juntas son `400`.
* **Identidad propagada:** si el MS actúa por un usuario, envía los `X-User-Id`/`X-User-Role` que recibió; el GW exige que vengan juntos y con formato válido (si no, `400`) y los reenvía.
* **El GW elimina `X-Service-Key` e inyecta `X-Service-Name`** (`ms01`…`ms08`). El log de acceso registra `origen = msNN`; la clave nunca se registra ni se reenvía.
* **La autorización es del destino:** por `X-Service-Name` (ruta interna) o por `X-User-Role` (ruta protegida). El GW no decide qué servicio llama a cuál.
* **Aislamiento de red:** en Docker solo se publica el 8080 del GW; los MS y el 9090 viven en una red interna (`docker-compose.yml`).

## Consecuencias
* Agregar o cambiar una clave exige configuración y reinicio del GW y del MS llamante.
* Una clave filtrada permite actuar como ese MS hasta rotarla; cada MS la guarda solo en su variable de entorno.
* Las rutas internas (MS08, MS05) se agregan cuando sus OpenAPI se publiquen (decisiones abiertas 1 y 3 de los Estándares).
