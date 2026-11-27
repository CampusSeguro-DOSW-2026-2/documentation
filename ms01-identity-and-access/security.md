# Seguridad — MS01 Identidad y acceso

Cómo MS01 registra y autentica a las personas, cómo forma la credencial de acceso (JWT), cómo recibe la identidad que propaga el Gateway y cómo protege los datos sensibles. Los valores transversales (algoritmo, `iss`, `aud`, vigencia, cabeceras, claves de servicio y aislamiento de red) están en los [Estándares de Integración](../integration/standards.md); el detalle de la sesión, en [jwt-spec.md](jwt-spec.md); y lo que hace el Gateway con el token, en la [seguridad del Gateway](../api-gateway/security.md).

## Índice

1. [Registro](#registro)
2. [Autenticación (login)](#autenticación-login)
3. [Formación del JWT](#formación-del-jwt)
4. [Identidad propagada por el Gateway](#identidad-propagada-por-el-gateway)
5. [Rutas internas y claves de servicio](#rutas-internas-y-claves-de-servicio)
6. [Corte de sesión](#corte-de-sesión)
7. [Contraseñas y recuperación](#contraseñas-y-recuperación)
8. [CORS y mTLS](#cors-y-mtls)
9. [Datos sensibles](#datos-sensibles)
10. [Otras medidas](#otras-medidas)

---

## Registro

El registro es público (`POST /auth/register`, `/register/verify` y `/register/resend`) y crea la cuenta solo cuando la persona demuestra que controla el correo institucional ([RF-MS01-01](1_requirements.md#rf-ms01-01-registrar-usuario-institucional)).

| Paso | Medida de seguridad |
|---|---|
| Solicitud | El correo se normaliza (recorte y minúsculas) y debe pertenecer a un dominio institucional (`INSTITUTIONAL_EMAIL_DOMAINS`). La contraseña cumple la política de [RN-MS01-03](1_requirements.md#rf-ms01-01-registrar-usuario-institucional): 8 a 64 caracteres, sin espacios, con mayúscula, minúscula, número y carácter especial. El rol no se recibe: toda cuenta nueva es `ESTUDIANTE` ([RN-MS01-02](1_requirements.md#rf-ms01-01-registrar-usuario-institucional)). El cuerpo no admite campos desconocidos |
| Registro pendiente | La solicitud no crea un usuario: se guarda en `pending_registrations` con la contraseña ya convertida en hash BCrypt. MS01 genera un código de 6 dígitos y un token de registro de 32 bytes con `SecureRandom`; guarda el código como hash BCrypt y el token como hash SHA-256, y devuelve el token solo a quien hizo la solicitud |
| Envío del código | El código en claro solo viaja en el correo. Entre dos códigos del mismo correo hay una espera mínima y cada correo tiene un límite de códigos cada 24 horas ([RN-MS01-44](1_requirements.md#rf-ms01-01-registrar-usuario-institucional)); la condición va en la misma sentencia SQL que escribe el código, así que peticiones simultáneas no envían más códigos de los permitidos |
| Verificación | Exige el token y el código. MS01 compara el hash del token en tiempo constante (`MessageDigest.isEqual`) antes de mirar el código, así que solo quien hizo el registro lo confirma ([RN-MS01-46](1_requirements.md#rf-ms01-01-registrar-usuario-institucional)). Cada código incorrecto suma un intento y, al llegar al máximo, el código se invalida. El mensaje de error es el mismo para un código incorrecto, vencido o reemplazado, un token de otro registro o un correo sin registro pendiente |
| Creación de la cuenta | Con el código correcto, en una transacción: crea el usuario `ACTIVO` con el hash de la contraseña que ya estaba guardado (la contraseña en claro no vuelve a existir), le asigna `ESTUDIANTE` y borra el registro pendiente |
| Limpieza | Una tarea programada borra los registros pendientes vencidos una vez termina su ventana diaria ([RN-MS01-45](1_requirements.md#rf-ms01-01-registrar-usuario-institucional)) |

Cada intento de registro, exitoso o fallido, se audita en MS08 ([catálogo de auditoría](../integration/audit-catalog.md#ms01--identidad-y-acceso)).

## Autenticación (login)

`POST /auth/login` es público ([RF-MS01-02](1_requirements.md#rf-ms01-02-autenticar-usuario-login)):

1. Busca el usuario por el correo normalizado.
2. Verifica la contraseña contra su hash con BCrypt (`BCryptPasswordEncoder` de Spring Security).
3. Si el correo no existe o la contraseña no coincide, responde `401 CREDENCIALES_INCORRECTAS` con el mensaje genérico `No autenticado.`, sin revelar cuál de los dos falló ([RN-MS01-05](1_requirements.md#rf-ms01-02-autenticar-usuario-login)).
4. Si la cuenta está `INACTIVO`, responde `403 CUENTA_INACTIVA` aunque las credenciales sean correctas ([RN-MS01-06](1_requirements.md#rf-ms01-02-autenticar-usuario-login)).
5. Lee los roles del usuario y los permisos de esos roles en la base de datos y emite el JWT ([formación del JWT](#formación-del-jwt)).
6. Responde `200` con `token`, `tipo: Bearer`, `expira_en` y `usuario` (`id`, `correo`, `roles`).

Cada login, exitoso o fallido, se audita en MS08 sin la contraseña.

## Formación del JWT

MS01 es el único emisor de la credencial de acceso. El contrato completo está en los [Estándares §3](../integration/standards.md) y en [jwt-spec.md](jwt-spec.md); aquí se resume cómo la forma MS01 (`JwtServiceImpl`).

| Elemento | Cómo lo forma MS01 |
|---|---|
| Algoritmo | RS256 (`SHA256withRSA` del JDK) con la llave privada `JWT_PRIVATE_KEY`, RSA PKCS#8 de al menos 2048 bits. MS01 no arranca con una llave que no sea RSA o tenga menos de 2048 bits |
| Header | `{"alg":"RS256","typ":"JWT","kid":"<JWT_KEY_ID>"}` |
| `iss` / `aud` | `ms01-identity-and-access` / `campusseguro`, fijos en el código |
| `sub` | UUID del usuario (`users.id`); es el valor que el Gateway propaga en `X-User-Id` |
| `email` | Correo institucional del usuario |
| `roles` | Nombres de los roles del usuario al momento del login, sin repetidos y en orden alfabético; al menos uno |
| `permissions` | Unión de los permisos de todos sus roles, leídos de `role_permissions` y `permissions` al emitir, sin repetidos y en orden alfabético; al menos uno, porque todo rol tiene `ms01:profile:read` y `ms01:session:logout` |
| `iat` | Hora de la base de datos truncada a segundos: el mismo reloj con el que MS01 registra los cortes de sesión, compartido por todas sus instancias |
| `exp` | `iat + 3600` segundos, fijo en el código |
| `jti` | UUID aleatorio por token |
| Firma | Header y payload en JSON, codificados en Base64URL sin relleno y firmados con la llave privada; la firma va en Base64URL como tercera parte del token |

El token no lleva contraseñas, nombres ni el estado de la cuenta ([RN-MS01-08](1_requirements.md#rf-ms01-03-gestión-de-sesión)). MS01 no guarda los tokens que emite: la sesión es *stateless* y solo se guardan las marcas de corte de sesión. Solo el Gateway verifica el token, con la llave pública `JWT_PUBLIC_KEY` del mismo par; MS01 nunca lo recibe.

## Identidad propagada por el Gateway

MS01 no valida JWT. En sus endpoints protegidos lee la identidad de las cabeceras que inyecta el Gateway, después de validar el token ([Identidad en MS01](jwt-spec.md#identidad-en-ms01)):

| Cabecera | Validación en MS01 | Si falta o es inválida |
|---|---|---|
| `X-User-Id` | UUID | `401 NO_AUTENTICADO` |
| `X-User-Role` | Lista separada por coma de nombres con el formato `^[A-Z][A-Z0-9_]{1,29}$` | `401 NO_AUTENTICADO` |
| `X-User-Permission` | Lista separada por coma de códigos con el formato `ms<dos dígitos>:<recurso>:<acción>` | `401 NO_AUTENTICADO` |

El filtro `IdentityHeadersAuthenticationFilter` de Spring Security convierte las tres cabeceras en la identidad de la petición. Cada operación exige su permiso (`x-permiso` del [contrato](api/openapi.yml)) contra `X-User-Permission`; si falta, responde `403 PROHIBIDO`. MS01 no recalcula los permisos a partir de los roles ni consulta la base para autorizar. La identidad nunca se toma del cuerpo ni de los parámetros ([RN-MS01-23](1_requirements.md#rf-ms01-03-gestión-de-sesión)), y nadie cambia sus propios roles ni su estado ([RN-MS01-12](1_requirements.md#rf-ms01-05-gestión-de-roles-y-permisos), [RN-MS01-19](1_requirements.md#rf-ms01-07-administración-de-usuarios)).

## Rutas internas y claves de servicio

| Dirección | Ruta | Cómo se autentica |
|---|---|---|
| Entrante | `POST /users/batch`, `GET /users/by-role` | El MS que llama envía su clave de servicio (`X-Service-Key`) al Gateway; el Gateway la valida, la elimina e inyecta `X-Service-Name`. MS01 solo acepta los servicios de `INTERNAL_ALLOWED_SERVICES` (`ms03`, `ms04` y `ms05` por defecto): sin `X-Service-Name` responde 401 y con otro servicio, 403 |
| Entrante | `GET /internal/session-cuts` | La llama el propio Gateway con `X-Service-Name: gateway`; MS01 solo acepta ese valor. La ruta no se publica a través del Gateway |
| Saliente | `POST /api/v1/audit/events` (MS08) | MS01 llama al Gateway (`GATEWAY_URL`) con su propia clave de servicio (`GATEWAY_SERVICE_KEY`, de al menos 32 caracteres) y el `X-Correlation-ID` de la petición original, de forma asíncrona y con un timeout de 2000 ms |

Las rutas internas no usan permisos de usuario: se autorizan por el servicio que llama ([RN-MS01-24](1_requirements.md#rf-ms01-09-resolución-interna-de-usuarios)). Como `X-Service-Name` es una cabecera, su seguridad depende de que nadie llegue a MS01 sin pasar por el Gateway ([CORS y mTLS](#cors-y-mtls)).

## Corte de sesión

Un cambio de los roles de un usuario, de los permisos de un rol, de su nombre o de su eliminación, o del estado de una cuenta, registra en la misma transacción una marca `valid_from` en `users` o en `roles`. El Gateway consulta cada 4 segundos la instantánea de cortes y rechaza con `401` los tokens con `iat` anterior, así que un cambio de acceso surte efecto en 7 segundos como máximo sin esperar a que el token venza ([RF-MS01-11](1_requirements.md#rf-ms01-11-corte-de-sesión-por-cambios-de-acceso), [Corte de sesión](jwt-spec.md#corte-de-sesión)). El logout y el cambio o restablecimiento de contraseña no cortan la sesión.

## Contraseñas y recuperación

| Caso | Medida |
|---|---|
| Almacenamiento | Solo el hash BCrypt (`users.password_hash`); la contraseña en claro nunca se guarda, nunca se devuelve y nunca se registra en logs ([RN-MS01-04](1_requirements.md#rf-ms01-01-registrar-usuario-institucional)) |
| Cambio autenticado | Exige la contraseña actual; la nueva cumple la política y es distinta de la actual ([RN-MS01-22](1_requirements.md#rf-ms01-08-recuperación-y-cambio-de-contraseña)) |
| Solicitud de código | La respuesta es la misma exista o no el correo ([RN-MS01-20](1_requirements.md#rf-ms01-08-recuperación-y-cambio-de-contraseña)). El código, de 6 dígitos con `SecureRandom`, se guarda como hash BCrypt y su valor en claro solo viaja en el correo |
| Uso del código | Un solo uso, vigencia corta, máximo de intentos fallidos y espera mínima entre solicitudes del mismo usuario ([RN-MS01-21](1_requirements.md#rf-ms01-08-recuperación-y-cambio-de-contraseña)); un código nuevo invalida los anteriores |
| Primer administrador | Se crea al arrancar solo si no hay ningún `ADMIN` y están definidas `BOOTSTRAP_ADMIN_EMAIL` y `BOOTSTRAP_ADMIN_PASSWORD`, que nunca van en el repositorio ([datos iniciales](3_data-model.md#datos-iniciales)) |

## CORS y mTLS

* **CORS:** MS01 no configura CORS. El navegador nunca llama a MS01: la única política CORS es la del Gateway ([RNF-MS01-03](1_requirements.md#requerimientos-no-funcionales-rnf)).
* **Aislamiento de red:** MS01 confía en `X-User-*` y `X-Service-Name` porque solo el Gateway puede llegar a él. En local su puerto no se publica fuera de la red Docker. En la nube, el firewall de su VM solo acepta la IP del Gateway y su Caddy de borde exige el certificado de cliente del Gateway (mTLS) emitido por la CA privada del entorno ([aislamiento en la nube](../api-gateway/contracts.md#7-aislamiento-en-la-nube)).
* **Llamadas salientes:** van al Gateway por HTTPS con la clave de servicio de MS01, nunca directamente a otro MS.

## Datos sensibles

| Dato | Tratamiento |
|---|---|
| Contraseñas | Hash BCrypt; nunca en claro, en respuestas, en logs ni en auditoría |
| Códigos de verificación y de recuperación | Hash BCrypt; el valor en claro solo viaja en el correo |
| Token de registro | Hash SHA-256; el valor en claro solo se entrega una vez a quien hizo el registro |
| JWT | No se guarda ni se registra; no lleva contraseñas ni nombres |
| Llave privada del JWT, credenciales de la base de datos y del correo, clave de servicio, datos del primer administrador | Variables de entorno, nunca en el repositorio ([RNF-MS01-06](1_requirements.md#requerimientos-no-funcionales-rnf)) |
| Datos personales (nombres y correo) | Solo los guarda MS01. Los demás microservicios guardan el UUID del usuario y consultan los datos por las rutas internas ([guía de integración](integration-guide.md#2-datos-de-usuario)) |
| Logs | Cada línea lleva el Correlation ID; no registran contraseñas, códigos, tokens ni cuerpos de petición ([RNF-MS01-05](1_requirements.md#requerimientos-no-funcionales-rnf)) |
| Errores | Todo error sale como `ErrorResponse` sin trazas, clases ni detalles internos ([RNF-MS01-04](1_requirements.md#requerimientos-no-funcionales-rnf)) |

## Otras medidas

* **Validación en el servidor:** todas las entradas se validan con las reglas de [2_forms-inputs.md](2_forms-inputs.md); los cuerpos de petición rechazan campos desconocidos y cada error indica el campo.
* **Integridad en la base de datos:** restricciones `CHECK`, `UNIQUE`, claves foráneas y triggers protegen los roles de sistema, los permisos obligatorios de cada rol y el permiso `ms01:role:manage`, aunque alguien escriba directamente en la base ([modelo de datos](3_data-model.md#reglas-de-integridad)). La aplicación no crea ni modifica el esquema (`ddl-auto: validate`).
* **Superficie expuesta:** Actuator solo expone `/actuator/health`; Swagger UI y la especificación OpenAPI en tiempo de ejecución están desactivados salvo que se active `SWAGGER_ENABLED` en desarrollo local.
* **Auditoría:** el registro, el login, el logout, el cambio de roles y de estado de los usuarios, la creación, edición, cambio de permisos y eliminación de roles, el cambio de contraseña y la recuperación se emiten a MS08 con su resultado (`EXITO` o `FALLO`) y el Correlation ID ([catálogo de auditoría](../integration/audit-catalog.md#ms01--identidad-y-acceso)).
* **Análisis estático:** PMD, SpotBugs y el Quality Gate de SonarCloud revisan el código en cada cambio ([quality-and-tests.md](quality-and-tests.md)).
