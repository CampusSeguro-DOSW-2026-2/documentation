# Especificación del JWT — emisión en MS01

Define cómo MS01 emite el token. El contrato que verifica el Gateway (algoritmo, `iss`, `aud`, vigencia, tolerancia, cabeceras) está en [Estándares](https://github.com/CampusSeguro-DOSW-2026-2/documentation/wiki/Est%C3%A1ndares-de-Integraci%C3%B3n-y-Contratos-Transversales) y no se repite aquí. Complementa [RF-MS01-03](1_requirements.md#rf-ms01-03), [RN-MS01-07 a 09](1_requirements.md#rn-ms01-07) y [RN-MS01-23](1_requirements.md#rn-ms01-23).

---

## Token emitido

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
| `sub` | UUID del usuario (`usuarios.id`). |
| `roles` | Roles vigentes al momento del login, en orden alfabético. Mínimo 1. |
| `iat` / `exp` | `exp = iat + JWT_EXPIRATION_MINUTES × 60`. No puede superar el máximo de los Estándares (3600 s). |
| `email` | Correo institucional (el Gateway no lo usa). |
| `jti` | UUID aleatorio por token (reservado para una futura revocación). |

**No incluye** contraseñas, hashes, nombres, estado ni permisos (RN-MS01-08). Los permisos se derivan del rol con la [Matriz Rol‐Permiso](https://github.com/CampusSeguro-DOSW-2026-2/documentation/wiki/Matriz-Rol%E2%80%90Permiso).

**Respuesta de login:** `token`, `tipo: Bearer`, `expira_en` (= `exp` en ISO 8601 UTC) y `usuario` (`id`, `correo`, `roles`). Ver [`api/swagger.yml`](api/swagger.yml).

---

## Ciclo de vida

1. **Emisión:** solo en un `POST /auth/login` exitoso con la cuenta ACTIVA ([RF-MS01-02](1_requirements.md#rf-ms01-02)).
2. **Cambios de rol o estado, logout, cambio o restablecimiento de contraseña:** no revocan los tokens ya emitidos; aplican a partir del siguiente login (RN-MS01-09, RN-MS01-10). La ventana máxima es la vigencia del token.
3. **Token inválido o expirado:** lo rechaza el Gateway con 401; nunca llega a MS01.

---

## Configuración (variables de entorno, nunca en el repositorio)

| Variable | Servicio | Valor por defecto | Descripción |
|---|---|---|---|
| `JWT_PRIVATE_KEY` | MS01 | — (obligatoria fuera de desarrollo) | Llave privada RSA 2048 en PEM (PKCS#8). Sin ella, MS01 genera un par efímero solo para desarrollo, que el Gateway no puede verificar. |
| `JWT_PUBLIC_KEY` | Gateway | — | Llave pública en PEM, del mismo par. |
| `JWT_EXPIRATION_MINUTES` | MS01 | `60` | Debe ser ≤ 60. |
| `JWT_ISSUER` / `JWT_AUDIENCE` | MS01 y Gateway | `ms01-identity-and-access` / `campusseguro` | Deben ser idénticos en ambos. |
| `JWT_KEY_ID` | MS01 | `ms01-2026-01` | Valor del `kid` (informativo). |

Generación del par de llaves:
```bash
openssl genpkey -algorithm RSA -pkeyopt rsa_keygen_bits:2048 -out private.pem
openssl rsa -in private.pem -pubout -out public.pem
```

**Cambio de llave (sin rotación en esta versión):** se genera un par nuevo y se despliegan a la vez la privada en MS01 y la pública en el Gateway. Los tokens vigentes dejan de ser válidos y los usuarios inician sesión de nuevo.
