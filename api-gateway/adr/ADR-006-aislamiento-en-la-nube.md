# ADR-006 — Aislamiento en la nube: firewall, HTTPS y mTLS hacia los MS

**Estado:** Aceptada · **Fecha:** 2026-11-08 · **RF:** [RF-GW-01](../01-requerimientos-funcionales.md#rf-gw-01), [RF-GW-03](../01-requerimientos-funcionales.md#rf-gw-03), [RF-GW-08](../01-requerimientos-funcionales.md#rf-gw-08)

## Contexto
En la nube cada servicio tiene su propia VM: los MS están en cuentas distintas de Azure o AWS y el GW en otra. Así desaparece la red interna de Docker de [ADR-005](ADR-005-llamadas-internas.md), que hoy es lo único que impide llegar directo a un MS.

Los MS confían en `X-User-Id`, `X-User-Role` y `X-Service-Name` sin validar un JWT (Estándares §3 y §4). Por eso alguien que llegara directo a uno podría falsificar esas cabeceras. Hay que garantizar que solo el GW llegue a los MS, sin cambiar su código.

## Decisión
* **Tres capas:**

  | Tramo | Protección |
  |---|---|
  | Cliente → GW | HTTPS con Let's Encrypt (Caddy en la VM del GW) + JWT |
  | MS → GW | HTTPS + `X-Service-Key` ([ADR-005](ADR-005-llamadas-internas.md)) |
  | GW → MS | Firewall (NSG/Security Group) que solo acepta la IP del GW + **mTLS**: el Caddy del MS exige el certificado de cliente del GW |

* **Una CA privada por entorno** (`test` y `prod`), en poder del dueño del GW. Cada CA firma dos tipos de certificado:
  * el de cliente del GW, que solo sirve para `clientAuth`;
  * el de servidor de cada MS, que solo sirve para `serverAuth` y lleva la IP de su VM.

  Así un GW de prueba no entra a un MS de producción, y el certificado de un MS no sirve como cliente. Los MS no usan Let's Encrypt porque sus puertos solo aceptan al GW.
* **Una VM, dos entornos:** producción en `443` y prueba en `8443`, detrás de un Caddy de borde por VM. Cada entorno tiene su stack y sus secretos (`gw-prod`, `gw-test`).
* **Perfil `cloud` del GW**, solo configuración (`application-cloud.yaml`):
  * el certificado de cliente se carga con un *SSL bundle* de Spring Boot (`GW_CLIENT_KEYSTORE`), que se valida al arrancar;
  * el GW solo confía en la CA del entorno (`GW_TRUSTED_CA`).
* **TLS rechazado = destino inalcanzable:** `502 SERVICIO_NO_DISPONIBLE` (RF-GW-01 A2).
* **Kit y plantillas en el repositorio del GW:**
  * `infra/mtls/` emite los certificados;
  * `infra/edge/` es el borde del GW;
  * `infra/caddy/` es la plantilla de borde que usa cada squad.
  * La guía para los squads está en la wiki: [Conexión al API Gateway](https://github.com/CampusSeguro-DOSW-2026-2/documentation/wiki/Conexi%C3%B3n-al-API-Gateway).

## Consecuencias
* **Los MS no cambian su código.** Cada squad configura su firewall y su Caddy con la plantilla.
* **Configuración inválida:** un certificado o una CA que no cargan impiden el arranque del GW (RNF-GW-06).
* **Ataques:** nadie llega a un MS sin pasar por el GW. Quedan dos riesgos:
  * quien robe `gw-client.p12` o la llave de una CA puede hacerse pasar por el GW de ese entorno;
  * un MS con su clave de servicio puede propagar cualquier `X-User-*` ([ADR-005](ADR-005-llamadas-internas.md)).

  Los secretos se guardan solo en los GitHub Environments y en archivos `0600` de la VM.
* **Cambios de IP:** si cambia la IP del GW, cada squad actualiza su firewall. Si cambia la IP de un MS, hay que emitirle un certificado nuevo y cambiar su `MSnn_URI`.
* **Vencimientos:** los certificados vencen a los 825 días y las CA a los 3 años. Una CA nueva invalida todos los certificados de su entorno.