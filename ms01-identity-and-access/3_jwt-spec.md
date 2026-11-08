# Especificación técnica de la sesión (JWT) — MS01

Detalle técnico de [RF-MS01-03](1_requirements.md#rf-ms01-03-gestión-de-sesión) (credencial de acceso), [RN-MS01-07 a 09](1_requirements.md#rf-ms01-03-gestión-de-sesión) y [RN-MS01-23](1_requirements.md#rf-ms01-03-gestión-de-sesión). El contrato que verifica el Gateway (algoritmo, `iss`, `aud`, vigencia, tolerancia de reloj, cabeceras) está en los [Estándares](https://github.com/CampusSeguro-DOSW-2026-2/documentation/wiki/Est%C3%A1ndares-de-Integraci%C3%B3n-y-Contratos-Transversales) y no se repite aquí.

---

## Token emitido

JWT (RFC 7519) firmado con RS256 y una llave RSA de al menos 2048 bits.

**Header**
```json
{ "alg": "RS256", "typ": "JWT", "kid": "ms01-2026-01" }
```

**Payload**
```json
{
  "iss": "ms01-identity-and-access",
  "aud": "campusseguro",
  "sub": "3f6c2a9e-8b1d-4c7a-9e2f-1a5b7c9d0e42",
  "email": "maria.perez@escuelaing.edu.co",
  "roles": ["BRIGADISTA", "ESTUDIANTE"],
  "iat": 1792510000,
  "exp": 1792513600,
  "jti": "5b1f0c2e-8d4a-4c1b-9a77-3f2e6d1a0b9c"
}
```

| Claim | Regla de emisión en MS01 |
|---|---|
| `iss` / `aud` | Fijos en el código (`util/GlobalValues.java`), con los valores de los Estándares. |
| `sub` | UUID del usuario (`usuarios.id`). |
| `roles` | Roles vigentes al momento del login, en orden alfabético. Mínimo 1. |
| `iat` / `exp` | `exp = iat + 3600 s`. La vigencia está fija en el código. |
| `email` | Correo institucional (el Gateway no lo usa). |
| `jti` | UUID aleatorio por token (reservado para una futura revocación). |

**No incluye** contraseñas, nombres, estado ni permisos ([RN-MS01-08](1_requirements.md#rf-ms01-03-gestión-de-sesión)). Los permisos se derivan del rol con la [Matriz Rol‐Permiso](https://github.com/CampusSeguro-DOSW-2026-2/documentation/wiki/Matriz-Rol%E2%80%90Permiso).

**Respuesta de login:** `token`, `tipo: Bearer`, `expira_en` (= `exp` en ISO 8601 UTC) y `usuario` (`id`, `correo`, `roles`). Ver [`api/swagger.yml`](api/swagger.yml).

---

## Identidad en MS01

MS01 nunca recibe ni valida el token. Lee las cabeceras que inyecta el Gateway, con el formato de los Estándares: `X-User-Id` y `X-User-Role` en los endpoints protegidos, y `X-Service-Name` en los internos ([RF-MS01-09](1_requirements.md#rf-ms01-09-resolución-interna-de-usuarios)), que no exigen identidad de usuario:

| Cabecera | Contenido | Si falta o es inválida |
|---|---|---|
| `X-User-Id` | UUID del usuario (`sub`). | 401 `NO_AUTENTICADO` |
| `X-User-Role` | Roles separados por coma, en orden alfabético (p. ej. `BRIGADISTA,ESTUDIANTE`). Un rol fuera del catálogo invalida la cabecera. | 401 `NO_AUTENTICADO` |
| `X-Service-Name` | Solo en endpoints internos: servicio que llama (`ms01`…`ms08`), derivado por el Gateway de su clave de servicio. Se compara sin distinguir mayúsculas con `INTERNAL_ALLOWED_SERVICES`. | Falta o vacía: 401 `NO_AUTENTICADO`. Servicio no autorizado: 403 `PROHIBIDO`. |
| `X-Correlation-ID` | UUID v4. Si falta o no es un UUID v4, MS01 genera uno. | Nunca rechaza. |

Con los roles, MS01 calcula los permisos (unión de la Matriz) y autoriza cada operación. Si falta el permiso, responde 403 `PROHIBIDO`. Los endpoints internos no usan la Matriz: los autoriza la lista de servicios.

MS01 confía en estas cabeceras porque solo el Gateway puede llegar a él (su puerto no se publica, [Estándares §2](https://github.com/CampusSeguro-DOSW-2026-2/documentation/wiki/Est%C3%A1ndares-de-Integraci%C3%B3n-y-Contratos-Transversales)). Quien acceda a la red interna sin pasar por el Gateway podría falsificarlas.

---

## Ciclo de vida

1. **Emisión:** solo en un `POST /auth/login` exitoso de una cuenta ACTIVA ([RF-MS01-02](1_requirements.md#rf-ms01-02-autenticar-usuario-login)).
2. **Cambios de rol o estado, logout, cambio o restablecimiento de contraseña:** no revocan los tokens emitidos; se aplican desde el siguiente login ([RN-MS01-09](1_requirements.md#rf-ms01-03-gestión-de-sesión), [RN-MS01-10](1_requirements.md#rf-ms01-04-cerrar-sesión-logout)). La ventana máxima es la vigencia del token.
3. **Token inválido o expirado:** el Gateway lo rechaza con 401; nunca llega a MS01.

**Por qué así:** el Gateway es el único que verifica el token con la llave pública. Así los microservicios solo leen cabeceras y las reglas de validación cambian en un solo punto. Se acepta que no haya revocación inmediata, porque la vigencia de 60 min acota la ventana.

---

## Configuración (variables de entorno, nunca en el repositorio)

| Variable | Servicio | Valor por defecto | Descripción |
|---|---|---|---|
| `JWT_PRIVATE_KEY` | MS01 | — (obligatoria fuera de desarrollo) | Llave privada RSA (≥ 2048 bits) en PEM PKCS#8 (`BEGIN PRIVATE KEY`; PKCS#1 se rechaza). Admite saltos de línea escritos como `\n`. Sin ella, MS01 genera un par efímero solo para desarrollo, que el Gateway no puede verificar. |
| `JWT_KEY_ID` | MS01 | `ms01-2026-01` | Valor del `kid` (informativo). |
| `JWT_PUBLIC_KEY` | Gateway | — | Llave pública en PEM del mismo par. |
| `INTERNAL_ALLOWED_SERVICES` | MS01 | `ms03,ms05` | Servicios autorizados para los endpoints internos, separados por coma. |

`iss`, `aud` y la vigencia no son configurables en MS01.

Generación del par de llaves:
```bash
openssl genpkey -algorithm RSA -pkeyopt rsa_keygen_bits:2048 -out private.pem
openssl rsa -in private.pem -pubout -out public.pem
```

**Cambio de llave (sin rotación en esta versión):** se genera un par nuevo y se despliegan a la vez la llave privada en MS01 y la pública en el Gateway. Los tokens vigentes dejan de ser válidos y los usuarios inician sesión de nuevo.
