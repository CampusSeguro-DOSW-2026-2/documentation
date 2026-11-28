# MS08 — Auditoría y analítica: guía de integración

Lo que los demás squads (MS01–MS07) necesitan para enviar eventos de auditoría a MS08. Las reglas generales (cabeceras, errores, correlación) están en los [Estándares](../integration/standards.md) y no se repiten aquí; el contrato del evento está en su **§12**.

## 1. Qué debe hacer su microservicio

MS08 es el único que guarda y consulta la auditoría. Su servicio **solo emite eventos**: no guarda auditoría como destino final, no expone endpoints de auditoría y no llama a MS08 directamente.

* Emita **un evento por cada acción relevante de su dominio**, con `resultado` `EXITO` o `FALLO`.
* Envíelo por el Gateway, de forma **asíncrona**: la respuesta al usuario nunca depende de la auditoría. Si el envío falla, regístrelo en su log con el `correlation_id` y siga.
* Si la acción ocurre dentro de un simulacro o emergencia, **identifique el evento** (sección 3): con eso MS08 calcula indicadores y decide quién ve cada registro.
* **Registre sus acciones** en el [catálogo de auditoría](../integration/audit-catalog.md) (sección 5).

## 2. Cómo enviar el evento

| | |
|---|---|
| Ruta | `POST ${GATEWAY_URL}/api/v1/audit/events` (ruta **interna**: clave de servicio, sin JWT) |
| Respuesta | `202 Accepted`, sin cuerpo |
| Timeout del cliente | `2000 ms` ([Estándares §1](../integration/standards.md)) |

| Cabecera | Valor |
|---|---|
| `X-Service-Key` | La `GATEWAY_SERVICE_KEY` de su servicio |
| `X-Correlation-ID` | El de la petición que originó la acción |
| `Content-Type` | `application/json` |
| `X-User-Id`, `X-User-Role`, `X-User-Permission` | Solo si la acción la origina un usuario: las tres juntas y tal como las recibió. Si falta alguna, el Gateway responde `400 VALIDACION` y el evento se pierde |

* No hace falta `X-Service-Name` ni JWT: su servicio se identifica por la clave y por el campo `modulo`.
* La URL del Gateway y la clave van en variables de entorno; está prohibido hardcodearlas.

### Cuerpo del evento

| Campo | Tipo | Regla |
|---|---|---|
| `id` | UUID | Lo genera su servicio. Recomendado; **obligatorio si reintenta**. |
| `usuario_id` | UUID \| null | Quien ejecuta la acción; `null` si es anónimo. La clave siempre va. |
| `modulo` | string | Su servicio: `MS01` … `MS07`. |
| `accion` | string | `MAYUSCULAS_CON_GUION_BAJO`, del catálogo de su servicio. |
| `recurso_afectado` | string \| null | `<tipo>:<id>`, p. ej. `incidente:7d5e…`. La clave siempre va; `null` solo si no hay recurso. |
| `resultado` | string | `EXITO` o `FALLO`. |
| `fecha_hora` | string | ISO 8601 UTC de **cuando ocurrió la acción**, no del reintento. |
| `correlation_id` | UUID | El de la petición. |
| `detalle` | string \| null | Código genérico del motivo (`CREDENCIALES_INCORRECTAS`). Sin texto libre. |
| `evento_id` | UUID \| null | Id del simulacro o emergencia de MS06; `null` si la acción no pertenece a un evento de MS06. |
| `tipo_evento` | string \| null | `SIMULACRO` o `REAL`. **Obligatorio si hay `evento_id`**; sin `evento_id`, solo `REAL` o `null`. |
| `atributos` | objeto \| null | Datos para indicadores (sección 4). |

```json
{
  "id": "0b8f2f6e-3c5a-4d2e-9a54-6f1d7c1b8e10",
  "usuario_id": "3f6c2a9e-8b1d-4c7a-9e2f-1a5b7c9d0e42",
  "modulo": "MS04",
  "accion": "REGISTRO_CHECKIN",
  "recurso_afectado": "checkin:7d5e4c1a-2b3f-4a6d-8e9f-0a1b2c3d4e5f",
  "resultado": "EXITO",
  "fecha_hora": "2026-11-18T14:32:10Z",
  "correlation_id": "c9a1e2f4-5b6d-4789-a0b1-c2d3e4f5a6b7",
  "detalle": null,
  "evento_id": "5a4b3c2d-1e0f-4a9b-8c7d-6e5f4a3b2c1d",
  "tipo_evento": "SIMULACRO",
  "atributos": { "estado": "SAFE", "punto_encuentro_id": "3f2a9c1e-7b4d-4e8a-9a55-0c1d2e3f4a5b" }
}
```

### Qué auditar

* **Audite:** acciones que crean, modifican, inactivan o eliminan datos de su dominio; acciones de seguridad (login, cambios de acceso, intentos prohibidos); hechos de un evento que sirven para indicadores; y los **fallos** de todas ellas.
* **No audite:** lecturas rutinarias, health checks, llamadas del Gateway ni cada petición HTTP: la auditoría no es un log de accesos.
* **Una acción, un evento**; no envíe uno por cada paso interno.
* **Nunca incluya** contraseñas, hashes, tokens, códigos de recuperación ni datos personales distintos de los ids (nombres, correos, documentos). Aplica también a `detalle` y `atributos`.

## 3. Eventos: simulacro, emergencia real y acciones administrativas

El valor de `evento_id` y `tipo_evento` decide en qué indicadores cuenta el registro y quién lo ve:

| `evento_id` | `tipo_evento` | Qué es | Quién lo ve |
|---|---|---|---|
| con valor | `SIMULACRO` o `REAL` | Acción de un evento de MS06 | Según el tipo |
| `null` | `REAL` | Emergencia real sin evento de MS06 (p. ej. una alerta manual de MS03) | Quien tiene `ms08:audit_real_event:read` |
| `null` | `null` | Administrativo (login, gestión de usuarios…) | Quien tiene ambos permisos |

* No existe `SIMULACRO` sin `evento_id`: todo simulacro es un evento de MS06.
* Cada servicio decide qué acciones son una emergencia real; una alerta informativa o de prueba va con `tipo_evento = null`.
* Las acciones sin `evento_id` no cuentan en los indicadores por evento, pero sí aparecen en la consulta de auditoría.

Ejemplo, una alerta manual de emergencia real (no registrada como evento en MS06):

```json
{
  "modulo": "MS03",
  "accion": "ALERTA_ACTIVADA",
  "recurso_afectado": "alerta:2b9d8c7e-6f5a-4b3c-8d2e-1f0a9b8c7d6e",
  "resultado": "EXITO",
  "evento_id": null,
  "tipo_evento": "REAL",
  "atributos": { "origen": "MANUAL" }
}
```

(Se omiten `id`, `usuario_id`, `fecha_hora`, `correlation_id` y `detalle` para abreviar; siempre se envían.)

## 4. Analítica: `atributos`

MS08 calcula los indicadores con los mismos eventos de auditoría: participantes, confirmación de alertas, check-ins por estado y punto de encuentro, incidentes por categoría, tiempos de respuesta, comparación entre simulacros y zonas o edificios más lentos.

* `atributos` lleva **solo ids, códigos y conteos**: objeto plano (sin anidar), claves en `snake_case`, valores `string` (máx. 100 caracteres), número o booleano, y máximo 20 claves.
* Documente sus claves en el bloque de su servicio del [catálogo de auditoría](../integration/audit-catalog.md), con su significado.
* Los hechos de cada servicio y los indicadores que alimentan están en ese mismo catálogo.

## 5. Registre sus acciones en el catálogo

Cada servicio mantiene su bloque en el [catálogo de auditoría](../integration/audit-catalog.md), con una tabla como esta:

| `accion` | Cuándo se emite | `recurso_afectado` | Evento | `atributos` |
|---|---|---|---|---|
| `REGISTRO_CHECKIN` | La persona registra su primer estado en el evento | `checkin:<id>` | Sí | `estado`, `punto_encuentro_id` |
| `LOGIN_FALLIDO` | Credenciales incorrectas | `null` | No | — |

## 6. Respuestas, reintentos y validación

| HTTP | `codigo` | Qué hacer |
|---|---|---|
| 202 | — | Listo. También se responde si el `id` ya existía. |
| 400 | `VALIDACION` | Evento mal formado: corrija su servicio, regístrelo en el log y no reintente. |
| 401 | `NO_AUTENTICADO` | Falta la clave de servicio o es inválida: revise `GATEWAY_SERVICE_KEY`. |
| 403 | `PROHIBIDO` | Su clave no está autorizada para la ruta: avise al equipo del Gateway. |
| 503 | `SERVICIO_NO_DISPONIBLE` | MS08 no puede guardar ahora: reintente si lo implementó, o registre en el log. |

Los errores usan el `ErrorResponse` de [Estándares §7.1](../integration/standards.md).

* **Reintentos (opcionales):** puede solo registrar el fallo en su log, o reintentar. Si reintenta, envíe siempre el mismo `id`, reintente solo ante timeout, `503` o `5xx` (nunca ante `4xx`), con espera creciente y un máximo de **5 intentos** y **24 horas**, y sin bloquear nunca la operación del usuario. La cola es solo para eventos de auditoría. MS08 es idempotente por `id`.
* **Orden:** MS08 ordena por `fecha_hora`, no por orden de llegada.
* **Qué valida MS08:** `modulo` entre `MS01` y `MS08`; `accion` con formato `^[A-Z][A-Z0-9_]*$`; `resultado` `EXITO` o `FALLO`; `tipo_evento` `SIMULACRO` o `REAL`, obligatorio si hay `evento_id` y, sin él, solo `REAL`; `fecha_hora` en UTC, no más de 5 minutos en el futuro ni más antigua que la retención; claves de primer nivel desconocidas se rechazan.

## 7. Quién consulta la auditoría

Su servicio **no** consulta la auditoría: la ofrece MS08 a los usuarios, con estos permisos ([matriz de roles y permisos](../integration/permissions-matrix.md)):

| Permiso | Qué permite ver |
|---|---|
| `ms08:audit_drill:read` | Registros de simulacros (`tipo_evento = SIMULACRO`) |
| `ms08:audit_real_event:read` | Registros de eventos reales (`tipo_evento = REAL`), con o sin evento |

* Quien tiene ambos ve todo, incluidos los registros administrativos.
* Asignación inicial: `ORGANIZADOR` → simulacros; `COORDINADOR` → eventos reales; `AUDITOR` y `ADMIN` → ambos.
* La auditoría es de solo lectura. **Retención: 6 meses** desde `fecha_hora`; después se elimina. Los indicadores por evento se conservan.

## 8. Pruebas locales

* No necesita MS08: use un servidor simulado (WireMock o similar) que responda `202`.
* Para trabajar sin MS08, apague el envío con una variable de su servicio (p. ej. `AUDIT_ENABLED=false`).
* Pruebe como mínimo: el cuerpo exacto que envía su servicio, que un fallo de MS08 no cambia la respuesta al usuario y que no incluye datos prohibidos.
* Antes de integrar: ¿envía `X-Service-Key` y `X-Correlation-ID`? ¿`fecha_hora` es la real, en UTC? ¿las acciones de un evento llevan `evento_id` y `tipo_evento`? ¿publicó su catálogo?
