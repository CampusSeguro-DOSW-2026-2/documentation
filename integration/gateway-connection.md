# Conexión al API Gateway en la nube

Guía para cualquier squad (MS02 a MS08) que quiera conectar su microservicio al API Gateway (GW). Diga siempre su número de MS: en esta guía, `NN` es ese número (`05` para MS05) y `808N` su puerto ([Estándares §2](standards.md)).

**Su squad solo hace cuatro cosas:**
1. crear su VM con dos reglas de firewall;
2. enviar un mensaje al equipo del GW;
3. cargar variables y secretos en **su** repositorio;
4. desplegar.

**El equipo del GW hace el resto:** certificados, rutas en el GW, claves de servicio, roles de prueba, y les ayuda con el borde y el despliegue sin que tengan que entender el código del GW. Ante cualquier duda, **pida ayuda al equipo del GW** antes de improvisar.

## 0. Datos del GW (cópielos tal cual)

| Dato | Valor |
|---|---|
| Dominio del GW | `campusseguro-gw.northcentralus.cloudapp.azure.com` |
| URL del GW en **prueba** | `https://campusseguro-gw.northcentralus.cloudapp.azure.com:8443` |
| URL del GW en **producción** | `https://campusseguro-gw.northcentralus.cloudapp.azure.com` |
| IP del GW (para su firewall) | `135.232.229.138` |
| Origen del frontend permitido en prueba | `http://localhost:5173` y `https://campusseguroftest.z41.web.core.windows.net` |

## 1. Cómo funciona (en una línea por tramo)

| Tramo | Cómo |
|---|---|
| Frontend o Postman → GW | HTTPS + JWT. El GW valida el token y le pasa a su MS `X-User-Id`, `X-User-Role` y `X-User-Permission`. |
| GW → su MS | HTTPS con **mTLS** a la IP de su VM: `:8443` prueba, `:443` producción. Su firewall solo acepta la IP del GW y su Caddy de borde solo acepta el certificado del GW. |
| Su MS → otro MS | **Siempre por el GW** (`GATEWAY_URL`), con su `X-Service-Key`. Nunca directo. |

Su MS **no** hace CORS ni valida JWT: de eso se encarga el GW.

## 2. Quién hace qué

| # | Su squad | Equipo del GW |
|---|---|---|
| 1 | Crea la VM con IP estática y el firewall (§3) | — |
| 2 | Envía el mensaje de alta (§4) | Emite sus certificados, genera sus claves de servicio y se los entrega (§5) |
| 3 | Instala Docker y el borde Caddy en su VM (§6). O le da acceso SSH temporal al equipo del GW, que lo hace por usted. | Le ayuda o lo hace por usted |
| 4 | Carga sus variables y secretos en su repositorio (§7) | Le dice qué valores van |
| 5 | Despliega su MS en su VM (§8) | Le ayuda a montar el despliegue automático (CI/CD) como el de MS01, si lo quiere |
| 6 | Avisa que está desplegado | Da de alta su ruta en el GW, prueba y le confirma (§9) |

## 3. Crear la VM y el firewall (su squad)

| Campo | Valor |
|---|---|
| Sistema | **Ubuntu Server 24.04 LTS, x64** (no ARM) |
| Tamaño | 2 vCPU y 4 GiB (Azure `Standard_B2s` o `B2ls_v2`; AWS `t3.medium`) |
| Disco | 30 GiB |
| IP pública | **Estática**. Azure: *Public IP › Assignment = Static*. AWS: *Elastic IP* asociada a la instancia. |

**Reglas de entrada.** No abra nada más: ni el `808N` de su MS ni el `5432` de su BD.

| Puerto | Origen | Para qué |
|---|---|---|
| `22/tcp` | Solo su IP (*My IP address*) | SSH |
| `443/tcp`, `8443/tcp` | Solo `135.232.229.138/32` | El GW: producción y prueba |

* **Azure (NSG):** *Networking › Add inbound port rule*. Source: *IP Addresses* `135.232.229.138/32`; Destination: **Any**; Service: *Custom*; ports `443,8443`; Protocol: *TCP*; Action: *Allow*.
* **AWS (Security Group):** *Inbound rules › Add rule*, dos reglas *Custom TCP* (`443` y `8443`) con Source *Custom* `135.232.229.138/32`.
* **Salida:** déjela abierta, como viene por defecto.

Si alguna vez cambia la IP de su VM, avise: hay que emitir certificados nuevos.

### IP estática en AWS (Elastic IP)

En AWS, la IP pública que EC2 asigna por defecto **cambia cada vez que la instancia se detiene y se vuelve a iniciar**, y los certificados del GW se emiten para su IP. Por eso su VM necesita una *Elastic IP*.

1. *EC2 › Network & Security › Elastic IPs › Allocate Elastic IP address*. Use la misma región de la instancia y *Amazon's pool of IPv4 addresses*. Clic en *Allocate*.
2. Seleccione la IP y elija *Actions › Associate Elastic IP address*. En *Resource type* ponga *Instance*, elija su instancia y su IP privada, y dé clic en *Associate*.
3. En *EC2 › Instances*, compruebe que el *Public IPv4 address* de su instancia sea la Elastic IP.

Equivalente con AWS CLI:

```bash
aws ec2 allocate-address --domain vpc                     # devuelve AllocationId y PublicIp
aws ec2 associate-address --instance-id <i-...> --allocation-id <eipalloc-...>
```

* **Orden:** asóciela **antes** del mensaje de alta (§4). La Elastic IP es la que reporta al GW y la que usa para SSH (`ubuntu@<Elastic IP>`).
* **Costo y limpieza:** AWS cobra cada IPv4 pública, esté asociada o no. Si elimina la VM, primero *Disassociate* y luego *Release* la Elastic IP.

## 4. Mensaje de alta al equipo del GW (su squad)

Copie, complete y envíe este mensaje por un canal privado (mensaje directo):

```text
Alta en el GW
- Microservicio: MSNN (<nombre del repositorio>)
- IP estática de la VM: <IP>
- Puerto interno del MS: 808N
- Prefijos de rutas de nuestro openapi.yml: /api/v1/<prefijo>/**, ...
- Rutas internas que exponemos (método + ruta exacta) y qué MS las llaman: <o "ninguna">
- ¿Llamamos a otros MS por el GW? <sí: a cuáles rutas / no>
- Entornos que vamos a usar: prueba, producción
- ¿Necesitamos ayuda con el borde o el despliegue? <sí/no>
```

Los prefijos y las rutas deben estar publicados en el `api/openapi.yml` de la carpeta de su microservicio en el repositorio `documentation`: el GW no enruta nada que no esté ahí.

## 5. Qué recibe del equipo del GW

Llega por un canal privado:

| Qué | Para qué | Dónde lo guarda |
|---|---|---|
| Carpeta `certs/` con `prod/` y `test/`, cada una con `servidor.crt`, `servidor.key` y `ca.pem` | Su Caddy de borde (mTLS) | **Solo en su VM**, en `~/campusseguro/edge/certs/`. Nunca en el repositorio. |
| `GATEWAY_SERVICE_KEY` de prueba y de producción (son distintas) | Llamar a otros MS por el GW | Secreto de GitHub (§7). Solo si su MS llama a otros. |
| URL del GW | `GATEWAY_URL` | Variable de GitHub (§7), con el valor de §0 |

## 6. Preparar la VM: Docker y borde Caddy (su squad, o el equipo del GW por usted)

Por SSH en su VM. Usuario: Azure, el que eligió (p. ej. `azureuser`); AWS, `ubuntu`.

```bash
# 1. Docker (una vez). Luego salga de la sesión SSH y vuelva a entrar
curl -fsSL https://get.docker.com | sudo sh
sudo usermod -aG docker "$USER"

# 2. Borde Caddy con mTLS (una vez). Cambie NN y 808N por los suyos
mkdir -p ~/campusseguro/edge && cd ~/campusseguro/edge
curl -fsSLO https://raw.githubusercontent.com/CampusSeguro-DOSW-2026-2/api-gateway/develop/infra/caddy/Caddyfile.microservicio
curl -fsSLO https://raw.githubusercontent.com/CampusSeguro-DOSW-2026-2/api-gateway/develop/infra/caddy/docker-compose.microservicio.yml
printf 'UPSTREAM_PROD=msNN-prod:808N\nUPSTREAM_TEST=msNN-test:808N\n' > .env
```

Desde **su PC**, copie la carpeta `certs/` recibida (en Windows funciona igual en PowerShell):

```bash
scp -i <su-llave>.pem -r certs <usuario>@<IP de su VM>:~/campusseguro/edge/
```

De nuevo **en la VM**:

```bash
cd ~/campusseguro/edge
chmod 600 certs/*/servidor.key
docker compose -f docker-compose.microservicio.yml up -d
docker compose -f docker-compose.microservicio.yml ps     # caddy: Up
```

Mientras su MS no esté desplegado, el borde responde `502` al GW. Es normal.

**¿Prefiere que lo haga el equipo del GW?**
1. Pídale su llave pública SSH.
2. Agréguela en la VM con `echo "<llave pública>" >> ~/.ssh/authorized_keys`.
3. Abra temporalmente el puerto `22` a la IP que le indiquen.
4. Cuando terminen, quite la llave y la regla.

## 7. Variables y secretos en su repositorio (su squad)

En **su** repositorio de GitHub: *Settings › Environments*. Cree `msNN-test` (*Deployment branches*: `develop`) y `msNN-prod` (*Deployment branches*: `main`; *Required reviewers*: su líder). Cargue:

| Nombre | Tipo | `msNN-test` | `msNN-prod` | ¿Cuándo? |
|---|---|---|---|---|
| `GATEWAY_URL` | Variable | `https://campusseguro-gw.northcentralus.cloudapp.azure.com:8443` | `https://campusseguro-gw.northcentralus.cloudapp.azure.com` | Siempre |
| `GATEWAY_SERVICE_KEY` | Secreto | La de prueba que le entregó el GW | La de producción | Si llama a otros MS |
| `DB_PASSWORD` | Secreto | `openssl rand -base64 24` | Otra distinta | Si tiene BD |
| Las propias de su MS (correo, APIs externas, etc.) | Variable o secreto | — | — | Según su MS |

Su repositorio también tiene que estar protegido:
* **Si es público:** en *Settings › Actions › General › Fork pull request workflows from outside collaborators*, elija **Require approval for all external contributors**.
* **Nunca suba al repositorio:** `certs/`, `*.key`, `*.pem`, `.env` ni claves. Agréguelos a su `.gitignore`.

Su código lee todo por variables de entorno. Nada de URLs fijas ni `localhost` para llegar a otro servicio.

## 8. Desplegar su MS (su squad, con ayuda del GW si la pide)

Su MS se une a la red `campus-edge` de la VM con el alias `msNN-test` (prueba) o `msNN-prod` (producción). **No publica puertos**: solo el borde Caddy recibe tráfico.

**Despliegue automático, como MS01.** Cada merge a `develop` despliega prueba y producción se lanza con un botón. El equipo del GW le ayuda a adaptar estos archivos del [repositorio de MS01](https://github.com/CampusSeguro-DOSW-2026-2/ms01-identity-and-access):
* `.github/workflows/ci.yml`, `deploy.yml` y `deploy-prod.yml`;
* `infra/deploy.sh` e `infra/docker-compose.prod.yml`.

Usted solo pone las variables del §7 y registra un runner en su VM (*Settings › Actions › Runners › New self-hosted runner*, con la etiqueta `msNN`).

```bash
# Comprobar dentro de la VM (el GW no enruta el health)
docker run --rm --network campus-edge curlimages/curl:8.11.1 -s http://msNN-test:808N/actuator/health
```

Debe responder `{"status":"UP"}`.

## 9. Alta en el GW y verificación

Cuando su health responda `UP`, avise al equipo del GW:
1. El equipo del GW agrega su ruta, su URI y sus claves, y redespliega el GW de prueba (unos 5 minutos). El procedimiento está en la guía de la wiki [Cómo agregar un microservicio al gateway](https://github.com/CampusSeguro-DOSW-2026-2/documentation/wiki/Como-agregar-un-microservicio-al-gateway).
2. Le confirma, y usted verifica desde su PC:

```bash
GW=https://campusseguro-gw.northcentralus.cloudapp.azure.com:8443
# Directo a su VM: debe dar timeout (solo el GW entra)
curl -m 5 -k https://<IP de su VM>:8443/
# Por el GW, sin token: 401 NO_AUTENTICADO
curl -i $GW/api/v1/<su prefijo>
# Por el GW, con token (§10): la respuesta de su MS
curl -i $GW/api/v1/<su prefijo> -H "Authorization: Bearer <token>"
# [Windows]: use curl.exe con los mismos argumentos
```

Mientras su MS no esté dado de alta, el GW responde `404 NO_ENCONTRADO` a sus rutas.

**Producción:** cuando prueba funcione, avise. El equipo del GW agrega su URI de producción y la lanza junto con la suya.

## 10. Probar contra el entorno de prueba compartido

GW y MS01 ya están desplegados en prueba. Puede integrarse desde su PC antes de desplegar su MS:
* **Usuarios:** regístrelos con `POST $GW/api/v1/auth/register`, con correo `@escuelaing.edu.co` o `@mail.escuelaing.edu.co`. MS01 envía un código de 6 dígitos a ese correo y la cuenta solo se crea al confirmarlo con `POST $GW/api/v1/auth/register/verify` (`{"correo", "codigo", "token_registro"}`, donde `token_registro` es el que devolvió el registro); por eso el correo debe existir y usted debe poder leerlo. Para darles un rol distinto de `ESTUDIANTE`, envíe el `id` al equipo del GW.
* **Token:** `POST $GW/api/v1/auth/login` con `{"correo":"...","contrasena":"..."}` devuelve `token`. Envíelo como `Authorization: Bearer <token>`.
* **Llamadas internas a otro MS:** envíe su clave de prueba como `X-Service-Key`, el mismo `X-Correlation-ID` que recibió y, si actúa por un usuario, sus `X-User-Id`, `X-User-Role` y `X-User-Permission`. Ejemplo de MS03, MS04 o MS05 a MS01:
  ```bash
  curl -i "$GW/api/v1/users/by-role?rol=BRIGADISTA" -H "X-Service-Key: <su clave de prueba>"
  ```
* **Cambios de roles o permisos:** si un administrador cambia los roles de un usuario, los permisos de un rol o el estado de una cuenta, el token anterior deja de aceptarse en 7 segundos como máximo (`401 NO_AUTENTICADO`). Vuelva a iniciar sesión para obtener un token con los datos nuevos.
* **Frontend local:** con `VITE_API_URL=$GW` en `http://localhost:5173`.
* **Producción no se usa para pruebas.** Los datos de prueba se pueden borrar en cualquier momento: no use datos reales.

## 11. Lo que su código debe cumplir

Detalle en los [Estándares](standards.md):
* Escucha en `808N` y expone `GET /actuator/health`.
* Todo bajo `/api/v1/`, y solo las rutas de su `openapi.yml`.
* **Rutas que requieren usuario:** lee `X-User-Id`, `X-User-Role` y `X-User-Permission` (los dos últimos, valores separados por coma) y autoriza con los permisos de `X-User-Permission` contra el `x-permiso` de la operación. Si faltan, responde `401 NO_AUTENTICADO`; si falta el permiso, `403 PROHIBIDO`. Los roles ya no son un conjunto fijo: no los compare por nombre ([Estándares §6](standards.md)).
* **Rutas internas:** lee `X-Service-Name` (`ms01`…`ms08`) para saber quién llama.
* **Errores:** todos salen como `ErrorResponse` (§7 de los Estándares), incluidos los del framework.
* **Llamadas a otros MS:** solo a `GATEWAY_URL`, con `X-Service-Key`, timeout de `2000 ms` y sin reintentar operaciones no idempotentes.
* Sin CORS ni validación de JWT.

## 12. Errores frecuentes

| Síntoma | Causa | Solución |
|---|---|---|
| `404 NO_ENCONTRADO` en sus rutas | Su MS aún no está dado de alta, o el prefijo no coincide con el `openapi.yml` | Avise al equipo del GW (§9) |
| `504 TIEMPO_AGOTADO` | El firewall no acepta `135.232.229.138`, o el borde está caído | Revise la regla de §3 y `docker compose ps` en `~/campusseguro/edge` |
| `502 SERVICIO_NO_DISPONIBLE` | Su MS está caído o el alias no coincide con `UPSTREAM_*`; o el certificado no es de su IP, o prueba y producción quedaron cruzados | Health dentro de la VM (§8); logs del borde: `docker compose -f docker-compose.microservicio.yml logs` |
| `401` en su MS con token válido | Su MS no lee `X-User-Id` / `X-User-Role` / `X-User-Permission` | §11 |
| `401 NO_AUTENTICADO` con un token que antes funcionaba | Un cambio de roles, permisos o estado cortó la sesión | Inicie sesión de nuevo (§10) |
| `403 PROHIBIDO` en una llamada interna | Su MS no está autorizado para esa ruta, o la clave es de otro entorno | Pida al equipo del GW que revise la ruta interna y use la clave del mismo entorno |
| `421 Misdirected Request` | Se modificó el Caddyfile | Use la plantilla sin cambios |

## 13. Resumen: dónde vive cada cosa

| Qué | Dónde | Nunca en |
|---|---|---|
| Código, `Dockerfile` y compose sin secretos | Su repositorio | — |
| `GATEWAY_URL`, `GATEWAY_SERVICE_KEY`, `DB_PASSWORD` y otros secretos | GitHub Environments `msNN-test` / `msNN-prod` | El repositorio, chats públicos, capturas |
| Certificados `servidor.crt`, `servidor.key`, `ca.pem` | Su VM: `~/campusseguro/edge/certs/` | El repositorio |
| IP de su VM, prefijos y rutas internas | Mensaje al equipo del GW (§4) | — |
| Su ruta, su URI y sus claves en el GW | Repositorio y Environments del GW (los gestiona el equipo del GW) | Su repositorio |

## 14. Despliegue del frontend

El frontend son archivos estáticos: no usa VM, Caddy, mTLS, certificados ni `GATEWAY_SERVICE_KEY`. Se publica en el **sitio web estático de Azure Storage** y el navegador llama al GW por HTTPS con JWT. Del lado del GW solo hay que agregar su URL a `CORS_ALLOWED_ORIGINS`. Detalle completo en [Frontend — Despliegue en la nube](https://github.com/CampusSeguro-DOSW-2026-2/documentation/wiki/Frontend-cloud-deployment).
