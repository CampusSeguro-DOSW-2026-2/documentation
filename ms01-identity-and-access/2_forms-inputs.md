# Diccionario de Datos, Formularios y Entradas

## Índice de Contenido
1. [Reglas generales de entrada](#reglas-generales-de-entrada)
2. [Formularios y Entradas](#formularios-y-entradas)
   - [F-01 · Registro de usuario](#f-01-registro-de-usuario)
   - [F-02 · Inicio de sesión](#f-02-inicio-de-sesión)
   - [F-03 · Perfil de usuario](#f-03-perfil-de-usuario)
   - [F-04 · Solicitud de recuperación de contraseña](#f-04-solicitud-de-recuperación-de-contraseña)
   - [F-05 · Restablecer contraseña con código](#f-05-restablecer-contraseña-con-código)
   - [F-06 · Cambio de contraseña autenticado](#f-06-cambio-de-contraseña-autenticado)
   - [F-07 · Búsqueda y filtros de usuarios](#f-07-búsqueda-y-filtros-de-usuarios)
   - [F-08 · Cambio de estado de usuario](#f-08-cambio-de-estado-de-usuario)
   - [F-09 · Asignación de roles](#f-09-asignación-de-roles)

---

## Reglas generales de entrada

* **RG-01**: Todo texto se recorta (trim) de espacios al inicio y al final antes de validar. El correo se normaliza a minúsculas. La contraseña no se recorta ni se normaliza.
* **RG-02**: Un campo obligatorio vacío o con solo espacios se considera faltante.
* **RG-03**: Se rechazan entradas con etiquetas HTML o scripts (<, >) en campos de texto libre.
* **RG-04**: Se rechazan (HTTP 400) los campos no definidos en el contrato (en especial roles, estado o id enviados por el usuario en registro y perfil).
* **RG-05**: Los mensajes de error indican el campo y la regla incumplida, salvo en login y recuperación ([RN-MS01-05](1_requirements.md#rf-ms01-02-autenticar-usuario-login), [RN-MS01-20](1_requirements.md#rf-ms01-08-recuperación-y-cambio-de-contraseña)).
* **RG-06**: Los errores de validación responden 400 `VALIDACION`, y un correo duplicado 409 `CONFLICTO`, con el `ErrorResponse` de los [Estándares](https://github.com/CampusSeguro-DOSW-2026-2/documentation/wiki/Est%C3%A1ndares-de-Integraci%C3%B3n-y-Contratos-Transversales). Cada campo y su mensaje van en `errores`.
* **RG-07**: Ningún formulario incluye la identidad del solicitante ([RN-MS01-23](1_requirements.md#rf-ms01-03-gestión-de-sesión)) ni datos de auditoría ([RNF-MS01-10](1_requirements.md#requerimientos-no-funcionales-rnf)).

---

## Formularios y Entradas

### F-01: Registro de usuario
Referencia: [RF-MS01-01](1_requirements.md#rf-ms01-01-registrar-usuario-institucional)

| Campo | Tipo | Obligatorio | Longitud / formato | Reglas de validación | Mensaje de error | Ejemplo válido |
|---|---|---|---|---|---|---|
| Primer nombre | Texto | Sí | 2 a 30 caracteres | Solo letras (con tildes y ñ), espacios, guion y apóstrofe. Sin números ni símbolos. | "El primer nombre debe tener entre 2 y 30 letras." | María |
| Segundo nombre | Texto | No | 2 a 30 caracteres | Opcional. Si se envía, misma regla que el primer nombre. | "El segundo nombre debe tener entre 2 y 30 letras." | José |
| Primer apellido | Texto | Sí | 2 a 30 caracteres | Solo letras (con tildes y ñ), espacios, guion y apóstrofe. Sin números ni símbolos. | "El primer apellido debe tener entre 2 y 30 letras." | Pérez |
| Segundo apellido | Texto | No | 2 a 30 caracteres | Opcional. Si se envía, misma regla que el primer apellido. | "El segundo apellido debe tener entre 2 y 30 letras." | Gómez |
| Correo institucional | Correo | Sí | Máx. 100 caracteres | Formato de correo válido. Dominio igual al institucional configurado. Único en el sistema (sin distinguir mayúsculas). | Formato o dominio: "Correo no válido." Duplicado (409): "El correo ya está registrado." | maria.perez@escuelaing.edu.co |
| Contraseña | Texto oculto | Sí | 8 a 64 caracteres | Al menos 1 mayúscula, 1 minúscula, 1 número y 1 carácter especial. Sin espacios. | "La contraseña debe tener 8 a 64 caracteres, con mayúscula, minúscula, número y carácter especial." | Campus#2026 |
| Confirmar contraseña | Texto oculto | Sí | Igual a Contraseña | Debe ser idéntica a Contraseña. | "Las contraseñas no coinciden." | Campus#2026 |

*Mockup F-01: (espacio reservado para insertar imagen del formulario de registro)*

---

### F-02: Inicio de sesión
Referencia: [RF-MS01-02](1_requirements.md#rf-ms01-02-autenticar-usuario-login)

| Campo | Tipo | Obligatorio | Longitud / formato | Reglas de validación | Mensaje de error | Ejemplo válido |
|---|---|---|---|---|---|---|
| Correo institucional | Correo | Sí | Máx. 100 caracteres | Formato de correo válido. | Formato (400): "Ingrese un correo válido." | maria.perez@escuelaing.edu.co |
| Contraseña | Texto oculto | Sí | 1 a 64 caracteres | No vacía. No se revalida la política en el login. | Vacía (400): "Ingrese su contraseña." | Campus#2026 |

**Mensajes globales de formulario:**
* Errores de autenticación: credenciales incorrectas o cuenta inexistente (401): "Credenciales incorrectas."
* Cuenta inactiva (403): "Cuenta inactiva. Contacte al administrador."

*Mockup F-02: (espacio reservado para insertar imagen del formulario de login)*

---

### F-03: Perfil de usuario
Referencia: [RF-MS01-06](1_requirements.md#rf-ms01-06-gestión-del-perfil)

| Campo | Tipo | Obligatorio | Editable | Longitud / formato | Reglas de validación | Mensaje de error |
|---|---|---|---|---|---|---|
| Primer nombre | Texto | Sí | Sí | 2 a 30 caracteres | Misma regla que [F-01](#f-01-registro-de-usuario). | "El primer nombre debe tener entre 2 y 30 letras." |
| Segundo nombre | Texto | No | Sí | 2 a 30 caracteres | Misma regla que [F-01](#f-01-registro-de-usuario). | "El segundo nombre debe tener entre 2 y 30 letras." |
| Primer apellido | Texto | Sí | Sí | 2 a 30 caracteres | Misma regla que [F-01](#f-01-registro-de-usuario). | "El primer apellido debe tener entre 2 y 30 letras." |
| Segundo apellido | Texto | No | Sí | 2 a 30 caracteres | Misma regla que [F-01](#f-01-registro-de-usuario). | "El segundo apellido debe tener entre 2 y 30 letras." |
| Correo institucional | Correo | — | No (solo lectura) | — | Cualquier intento de cambio se rechaza ([RN-MS01-15](1_requirements.md#rf-ms01-06-gestión-del-perfil)). | "El correo no se puede modificar." |
| Roles | Lista | — | No (solo lectura) | — | Se modifican solo por el administrador ([F-09](#f-09-asignación-de-roles)). | "Los roles no se pueden modificar desde el perfil." |
| Estado | Etiqueta | — | No (solo lectura) | ACTIVO / INACTIVO | Se modifica solo por el administrador ([F-08](#f-08-cambio-de-estado-de-usuario)). | "El estado no se puede modificar desde el perfil." |

*Mockup F-03: (espacio reservado — insertar imagen de la pantalla de perfil)*

---

### F-04: Solicitud de recuperación de contraseña
Referencia: [RF-MS01-08](1_requirements.md#rf-ms01-08-recuperación-y-cambio-de-contraseña)

| Campo | Tipo | Obligatorio | Longitud / formato | Reglas de validación | Mensaje de error | Ejemplo válido |
|---|---|---|---|---|---|---|
| Correo institucional | Correo | Sí | Máx. 100 caracteres | Formato de correo válido. | Formato (400): "Ingrese un correo válido." | maria.perez@escuelaing.edu.co |

**Mensaje global de formulario:**
Respuesta única (exista o no el correo): "Si el correo está registrado, enviamos un código de recuperación."

*Mockup F-04: (espacio reservado para insertar imagen)*

---

### F-05: Restablecer contraseña con código
Referencia: [RF-MS01-08](1_requirements.md#rf-ms01-08-recuperación-y-cambio-de-contraseña)

| Campo | Tipo | Obligatorio | Longitud / formato | Reglas de validación | Mensaje de error | Ejemplo válido |
|---|---|---|---|---|---|---|
| Correo institucional | Correo | Sí | Máx. 100 caracteres | Formato de correo válido. | "Ingrese un correo válido." | maria.perez@escuelaing.edu.co |
| Código de recuperación | Texto numérico | Sí | 6 dígitos | Solo dígitos. Vigente, no usado y sin agotar sus intentos ([RN-MS01-21](1_requirements.md#rf-ms01-08-recuperación-y-cambio-de-contraseña)). | "Código incorrecto o vencido." (genérico) | 482913 |
| Nueva contraseña | Texto oculto | Sí | 8 a 64 caracteres | Misma política que [F-01](#f-01-registro-de-usuario). | "La contraseña debe tener 8 a 64 caracteres, con mayúscula, minúscula, número y carácter especial." | Nueva#2026 |
| Confirmar nueva contraseña | Texto oculto | Sí | Igual a Nueva contraseña | Debe ser idéntica. | "Las contraseñas no coinciden." | Nueva#2026 |

*Mockup F-05: (espacio reservado para insertar imagen)*

---

### F-06: Cambio de contraseña autenticado
Referencia: [RF-MS01-08](1_requirements.md#rf-ms01-08-recuperación-y-cambio-de-contraseña)

| Campo | Tipo | Obligatorio | Longitud / formato | Reglas de validación | Mensaje de error | Ejemplo válido |
|---|---|---|---|---|---|---|
| Contraseña actual | Texto oculto | Sí | 1 a 64 caracteres | Debe coincidir con la almacenada. | "La contraseña actual es incorrecta." | Campus#2026 |
| Nueva contraseña | Texto oculto | Sí | 8 a 64 caracteres | Misma política que [F-01](#f-01-registro-de-usuario). Distinta de la actual. | "La nueva contraseña no cumple la política o es igual a la actual." | Nueva#2026 |
| Confirmar nueva contraseña | Texto oculto | Sí | Igual a Nueva contraseña | Debe ser idéntica. | "Las contraseñas no coinciden." | Nueva#2026 |

*Mockup F-06: (espacio reservado para insertar imagen)*

---

### F-07: Búsqueda y filtros de usuarios
Referencia: [RF-MS01-07](1_requirements.md#rf-ms01-07-administración-de-usuarios) (y [RF-MS01-05](1_requirements.md#rf-ms01-05-gestión-de-roles-y-permisos), paso 1)

| Campo | Tipo | Obligatorio | Longitud / formato | Reglas de validación | Mensaje de error | Ejemplo válido |
|---|---|---|---|---|---|---|
| Búsqueda por correo | Texto | No | 1 a 100 caracteres | Búsqueda parcial. Sin etiquetas HTML. | "Búsqueda no válida." | maria.perez |
| Filtro de estado | Lista | No | ACTIVO / INACTIVO / Todos | Solo valores del catálogo. | "Estado no válido." | ACTIVO |
| Filtro de rol | Lista | No | Uno de los 8 roles / Todos | Solo valores del catálogo. | "Rol no válido." | BRIGADISTA |
| Página | Número entero | No | ≥ 0 (por defecto 0) | Entero no negativo. | "Página no válida." | 0 |
| Tamaño de página | Número entero | No | 1 a 100 (por defecto 20) | Entero en el rango. | "Tamaño de página no válido (1 a 100)." | 20 |

*Mockup F-07: (espacio reservado para insertar imagen del listado de usuarios)*

---

### F-08: Cambio de estado de usuario
Referencia: [RF-MS01-07](1_requirements.md#rf-ms01-07-administración-de-usuarios)

| Campo | Tipo | Obligatorio | Longitud / formato | Reglas de validación | Mensaje de error | Ejemplo válido |
|---|---|---|---|---|---|---|
| Usuario(s) | Selección de uno o más usuarios | Sí | UUID existentes | Deben existir. No puede incluir al propio administrador. | Inexistente (404): "Usuario no encontrado." Propio (403): "No puede cambiar su propio estado." | 3f6c2a9e-8b1d-4c7a-9e2f-1a5b7c9d0e42 |
| Nuevo estado | Lista | Sí | ACTIVO / INACTIVO | Solo valores del catálogo. | "Estado no válido." | INACTIVO |

*Mockup F-08: (espacio reservado para insertar imagen de la confirmación de cambio de estado)*

---

### F-09: Asignación de roles
Referencia: [RF-MS01-05](1_requirements.md#rf-ms01-05-gestión-de-roles-y-permisos)

| Campo | Tipo | Obligatorio | Longitud / formato | Reglas de validación | Mensaje de error | Ejemplo válido |
|---|---|---|---|---|---|---|
| Usuario(s) | Selección de uno o más usuarios | Sí | UUID existentes | Deben existir. No puede incluir al propio administrador. | Inexistente (404): "Usuario no encontrado." Propio (403): "No puede modificar sus propios roles." | 3f6c2a9e-8b1d-4c7a-9e2f-1a5b7c9d0e42 |
| Roles a agregar | Selección múltiple | No | Si se envía, 1 a 8 roles | Solo roles del catálogo, sin repetir. | "Seleccione roles válidos." | BRIGADISTA |
| Roles a retirar | Selección múltiple | No | Si se envía, 1 a 8 roles | Solo roles del catálogo, sin repetir. | "Seleccione roles válidos." | PROFESOR |

**Mensajes globales de formulario:**
* Ninguna lista enviada: "Indique roles a agregar o retirar."
* Un rol en ambas listas: "Un rol no puede agregarse y retirarse a la vez."
* Algún usuario quedaría sin roles: "Cada usuario debe conservar al menos un rol."

*Mockup F-09: (espacio reservado para insertar imagen de la edición de roles)*