# Bitácora de investigación — MS01 (no normativa)

> **Este documento no es un contrato.** Registra por qué se tomaron las decisiones. Las reglas vigentes están en los [Estándares de Integración](https://github.com/CampusSeguro-DOSW-2026-2/documentation/wiki/Est%C3%A1ndares-de-Integraci%C3%B3n-y-Contratos-Transversales) (EST) y en los requerimientos de MS01. Ante cualquier diferencia, prevalecen estos.

## Índice
1. [Autenticación con JWT y estrategia de validación](#jwt)
2. [Trazabilidad con Correlation ID](#correlacion)
3. [Centralización de la auditoría](#auditoria)
4. [Propagación de identidad y edición del perfil](#identidad-perfil)
5. [Pruebas de integración del flujo de cabeceras](#pruebas)

---

<a id="jwt"></a>
## 1. Autenticación con JWT y estrategia de validación

**Problema:** controlar el acceso a todos los servicios sin consultar a MS01 ni a su base de datos en cada petición.

**Alternativas evaluadas:**
* **A, descentralizada:** el Gateway pasa el JWT y cada microservicio valida la firma.
* **B, centralizada (seleccionada):** el Gateway valida el JWT (firma RS256, `iss`, `aud`, `exp`), lo elimina y envía al microservicio cabeceras limpias (`X-User-Id`, `X-User-Role`, `X-Correlation-ID`).

**Por qué B:**
* Los microservicios no necesitan librerías criptográficas; solo leen cabeceras.
* Las reglas de validación cambian en un solo punto (el Gateway).
* El usuario inicia sesión una vez y el token se verifica localmente con la llave pública, sin llamar a MS01.

**Límite aceptado:** no hay revocación inmediata; la vigencia corta (60 min) acota la ventana.

**Resultado:** Estándares, [`3_jwt-spec.md`](../3_jwt-spec.md).

<a id="correlacion"></a>
## 2. Trazabilidad con Correlation ID

**Problema:** una acción del usuario recorre varios servicios (p. ej. Gateway → MS04 → Gateway → MS05), y sin un identificador común no se puede reconstruir en los logs.

**Decisión:** un UUID v4 en `X-Correlation-ID`. El Gateway conserva el valor si el cliente envía uno válido y, si no, genera uno nuevo. Cada servicio lo imprime en cada línea de log, lo devuelve en los errores y lo propaga en sus llamadas internas y en sus eventos de auditoría.

**Resultado:** Estándares.

<a id="auditoria"></a>
## 3. Centralización de la auditoría

**Problema:** registrar acciones relevantes sin penalizar el tiempo de respuesta.

**Decisión:** cada microservicio (no el Gateway) emite eventos **asíncronos** a MS08 a través del Gateway, con el `X-Correlation-ID`. Si MS08 falla, la respuesta al usuario no cambia. MS01 no tiene tablas de auditoría.

**Resultado:** Estándares, RNF-MS01-10.

<a id="identidad-perfil"></a>
## 4. Propagación de identidad y edición del perfil

**Lectura de roles:** el Gateway traduce los claims `sub` y `roles` a cabeceras de texto plano. El microservicio autoriza con `X-User-Role` contra la matriz, sin llamadas de red adicionales:

```text
[Cliente] --(Authorization: Bearer JWT)--> [API Gateway]
                                               | valida el JWT y lo elimina
                                               v
[Microservicio] <-- X-User-Id: 3f6c2a9e-…  X-User-Role: ESTUDIANTE  X-Correlation-ID: c19b472d-…
```

> El JWT va **firmado** (JWS), no cifrado: cualquiera puede leer sus claims, pero nadie puede alterarlos sin invalidar la firma. Por eso no contiene datos sensibles (RN-MS01-08).

**Edición del perfil:** `PUT /api/v1/users/me`. El usuario se identifica solo por `X-User-Id` (nunca por el cuerpo), así que solo puede modificarse a sí mismo. Solo son editables los cuatro campos de nombre. Enviar correo, roles, estado o id se **rechaza con 400** (RN-MS01-15, RG-04), y la contraseña se cambia en `PUT /users/me/password`.

<a id="pruebas"></a>
## 5. Pruebas de integración del flujo de cabeceras

Escenarios mínimos que deben automatizarse en el pipeline (detalle en las pruebas del Gateway, INT-GW-*):

| Escenario | Entrada | Resultado esperado |
|---|---|---|
| Petición válida | JWT válido con rol `ESTUDIANTE` hacia `GET /api/v1/users/me` | El Gateway elimina `Authorization` e inyecta `X-User-Id`, `X-User-Role: ESTUDIANTE` y `X-Correlation-ID`; MS01 responde 200. |
| Token inválido o expirado | Token alterado o vencido | El Gateway responde `401 NO_AUTENTICADO` y no llama a MS01. |
| Suplantación | Cliente envía `X-User-Role: ADMIN` con un JWT de `ESTUDIANTE` | El Gateway elimina las cabeceras `X-User-*` del cliente e inyecta `ESTUDIANTE`. `X-Correlation-ID` se conserva si es un UUID v4 válido. |
