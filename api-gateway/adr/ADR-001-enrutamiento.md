# ADR-001 — Enrutamiento por configuración estática

**Estado:** Aceptada · **Fecha:** 2026-10-25 · **RF:** [RF-GW-01](../01-requerimientos-funcionales.md#rf-gw-01)

## Contexto
El GW debe llegar a MS01–MS08 sin que el cliente conozca sus URI y debe responder de forma controlada si un destino falla. Los destinos son pocos, con nombres DNS fijos en Docker (EST §2).

## Decisión
* **Tabla de rutas en configuración externa** (`gateway.routes` en `application.yaml`, destinos `MSnn_URI`), validada al arrancar: destino ausente, URI mal formada, prefijo fuera de `/api/v1/` o prefijos duplicados impiden el arranque.
* **Solo rutas publicadas** en el `swagger.yml` del MS (RN-GW-03). Cada patrón es una ruta y gana el **prefijo literal más largo**.
* **URI estáticas por ambiente** (perfiles `local` y `docker`). **Sin Eureka** ni balanceo.
* **Reenvío sin cambios** de método, ruta completa, query params y cuerpo; el GW no restringe métodos.
* **Timeout único** `GATEWAY_DOWNSTREAM_TIMEOUT_MS` para conexión y respuesta, **sin reintentos**: timeout → `504`; destino inalcanzable → `502`.

## Consecuencias
* Agregar o cambiar una ruta exige configuración y reinicio, no recompilar (RN-GW-02).
* Un destino caído no se reintenta ni se sustituye: el cliente recibe `502`/`504` en máximo el timeout.
* Escalar un MS a varias instancias requiere un balanceador externo o revisar esta decisión.
