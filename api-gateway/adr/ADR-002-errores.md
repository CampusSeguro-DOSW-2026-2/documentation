# ADR-002 — Normalización de errores y regla del 502

**Estado:** Aceptada · **Fecha:** 2026-10-25 · **RF:** [RF-GW-05](../01-requerimientos-funcionales.md#rf-gw-05)

## Contexto
El cliente debe recibir siempre errores con el mismo formato y sin detalles internos. Los errores vienen del propio GW o de MS01–MS08, y un MS puede devolver por error el cuerpo por defecto de Spring, un HTML o una traza.

## Decisión
* **`ErrorResponse` estricto** de los Estándares §7.1: exactamente `status`, `codigo`, `mensaje`, `errores` y `correlation_id`, nunca `null`.
* **Catálogo cerrado** de los 7 errores del GW (GW-C §4) con mensajes fijos. Toda excepción pasa por un único manejador global; lo que no se reconoce sale como `500 ERROR_INTERNO` y el stack trace solo va al log.
* **Regla del 502:** un `4xx`/`5xx` de un MS se propaga sin cambios solo si es JSON con los cinco campos, `status` igual al código HTTP y `correlation_id` igual al de la solicitud. Si no, el GW responde `502 SERVICIO_NO_DISPONIBLE`. El log registra el motivo, nunca el cuerpo. Las respuestas `2xx`/`3xx` no se inspeccionan.
* El cuerpo de error que se inspecciona tiene un límite de **64 KiB**. Uno mayor se considera no conforme.

## Consecuencias
* Un MS que incumple el contrato no puede filtrar información: el cliente ve un `502` genérico y el equipo del MS encuentra el motivo en el log del GW por `correlation_id`.
* Cada MS debe responder sus errores con `ErrorResponse` y devolver el `X-Correlation-ID` recibido, incluidos los errores del framework (404, 405, 415, 400).
* Agregar un error del GW exige agregarlo al catálogo y a GW-C §4.
