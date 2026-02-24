# Guía avanzada completa: n8n + Portainer detrás de Traefik (HTTPS) en Docker (VPS Ubuntu)

## Objetivo

Esta guía **paso a paso** te lleva desde una instalación existente (n8n/Portainer corriendo en Docker) hasta una arquitectura segura y mantenible con:

- **Traefik v3** como reverse proxy
- **HTTPS** automático con **Let's Encrypt**
- **n8n** y **Portainer** **sin puertos públicos directos** (solo internos en Docker)
- **Dashboard de Traefik solo interno** (loopback + túnel SSH)
- **Basic Auth** para Portainer (doble capa de seguridad)
- **Rate limiting** (n8n y Portainer)
- **IP allowlist** para Portainer (opcional pero recomendado)
- **Healthchecks básicos**
- **Backups automáticos** de volúmenes (`n8n_data`, `portainer_data`)
- **Comandos de operación** (iniciar, detener, reiniciar, actualizar)
- **Troubleshooting**, **glosario**, **FAQ** y **exportación** de esta guía

---

# Resumen de arquitectura final

## Dominios usados

- **n8n:** `https://n8n-devtallez.{VPS_DOMAIN}`
- **Portainer:** `https://portainer.{VPS_DOMAIN}`
- **Traefik dashboard:** **NO público** (solo interno por SSH tunnel)

## Qué puertos quedan expuestos

### Públicos (VPS)
- `80/tcp` → HTTP (redirección + Let's Encrypt challenge)
- `443/tcp` → HTTPS

### Internos / no públicos
- `5678/tcp` → n8n (solo interno por Docker network)
- `9443/tcp` → Portainer (solo interno por Docker network)
- `8080/tcp` → dashboard Traefik (solo `127.0.0.1` en el VPS)
- `8000/tcp` → Portainer Edge Agents (solo interno; opcional)

---

# Antes de empezar

## Requisitos

1. **VPS Ubuntu** (con acceso SSH)
2. **Docker** y **Docker Compose plugin** instalados
3. **DNS** configurado apuntando a la IP del VPS:
   - `n8n-devtallez.{VPS_DOMAIN}`
   - `portainer.{VPS_DOMAIN}`
   - `traefik.{VPS_DOMAIN}` *(aunque el dashboard no será público)*
4. **Firewall** permitiendo `80` y `443`
5. Volúmenes existentes:
   - `n8n_data`
   - `portainer_data`

## Verificar Docker y Compose
```bash
docker version
docker compose version
```

## Verificar firewall (UFW) y abrir puertos si hace falta
```bash
sudo ufw allow 80/tcp
sudo ufw allow 443/tcp
sudo ufw status
```

---

# Paso a paso (orden recomendado)

## Paso 1) Verificar contenedores y puertos actuales

Esto te muestra si `n8n` o `portainer` están expuestos públicamente ahora mismo.

```bash
docker ps --format "table {{.Names}}\t{{.Ports}}"
```

### Qué debes observar
- Si ves `0.0.0.0:5678->5678/tcp` → n8n está público
- Si ves `0.0.0.0:9443->9443/tcp` → Portainer está público
- En la configuración final **ya no debe aparecer** eso

## Paso 2) Confirmar volúmenes existentes

```bash
docker volume ls | grep -E 'n8n_data|portainer_data'
```

Si **no existen**, créalos:
```bash
docker volume create n8n_data
docker volume create portainer_data
```

## Paso 3) Detener y eliminar contenedores actuales (sin borrar datos)

> Esto **no borra tus datos (workflows, credenciales, configuraciones, etc.)**, porque los datos viven en los volúmenes (n8n_data, portainer_data).

```bash
docker stop n8n portainer
docker rm n8n portainer
```

> Si Traefik viejo existe y lo vas a reemplazar:
```bash
docker stop traefik || true
docker rm traefik || true
```

## Paso 4) Crear carpeta de trabajo

```bash
mkdir -p ~/infra-traefik-n8n-portainer
cd ~/infra-traefik-n8n-portainer
```

## Paso 5) Crear archivo `.env`

Crea el archivo:
```bash
nano .env
```

Pega este contenido completo:

```env
# Dominios
N8N_DOMAIN=n8n-devtallez.{VPS_DOMAIN}
PORTAINER_DOMAIN=portainer.{VPS_DOMAIN}
TRAEFIK_DOMAIN=traefik.{VPS_DOMAIN}

# Let's Encrypt
LETSENCRYPT_EMAIL=mora.corp@yahoo.com.mx

# Zona horaria
TZ=America/Mexico_City
GENERIC_TIMEZONE=America/Mexico_City

# n8n encryption key (IMPORTANTE: no cambiar después)
N8N_ENCRYPTION_KEY=PEGA_AQUI_TU_LLAVE_GENERADA

# Basic Auth para Portainer (Traefik) -> pega el hash generado con htpasswd
PORTAINER_BASICAUTH_HASH=admin:$$2y$$05$$REEMPLAZA_CON_TU_HASH_COMPLETO

# IP allowlist Portainer (opcional pero recomendado)
# Usa tu IP pública actual /32, ejemplo:
PORTAINER_ALLOWED_IPS=TU.IP.PUBLICA.ACTUAL/32

# Backups automáticos
BACKUP_RETENTION_DAYS=14
BACKUP_INTERVAL_SECONDS=86400
```

## Paso 6) Generar `N8N_ENCRYPTION_KEY` (Ubuntu)

Esta llave se usa para encriptar credenciales dentro de n8n (tokens, passwords, API keys).

Genera una llave fuerte y guárdala automáticamente dentro del `.env`:

```bash
KEY="$(openssl rand -base64 32)" && \
grep -q '^N8N_ENCRYPTION_KEY=' .env \
  && sed -i "s|^N8N_ENCRYPTION_KEY=.*|N8N_ENCRYPTION_KEY=$KEY|" .env \
  || echo "N8N_ENCRYPTION_KEY=$KEY" >> .env
```

Verifica:
```bash
grep '^N8N_ENCRYPTION_KEY=' .env
```

> ⚠️ **No cambies esta llave después**, o n8n no podrá descifrar credenciales existentes.

## Paso 7) Generar Basic Auth para Portainer (Traefik)

Este hash se usa para autenticar a Portainer detrás de Traefik. Esto agrega una capa extra antes del login propio de Portainer.

Instalar utilidad htpasswd (Ubuntu)
```bash
sudo apt update && sudo apt install -y apache2-utils
```

Genera hash:
```bash
htpasswd -nbB admin 'TU_PASSWORD_SUPER_SEGURA' | sed -e 's/\$/\$\$/g'
```

Copia el resultado (ej. `admin:$$2y$$...`) y reemplázalo en `.env`:
```env
PORTAINER_BASICAUTH_HASH=admin:$$2y$$...
```

## Paso 8) Crear `docker-compose.yml`

Crea el archivo:
```bash
nano docker-compose.yml
```

Pega este contenido completo:

```yaml
services:
  traefik:
    image: traefik:v3.6.8
    container_name: traefik
    restart: unless-stopped
    command:
      # Docker provider
      - --providers.docker=true
      - --providers.docker.exposedbydefault=false

      # Entrypoints públicos
      - --entrypoints.web.address=:80
      - --entrypoints.websecure.address=:443

      # Redirect HTTP -> HTTPS
      - --entrypoints.web.http.redirections.entrypoint.to=websecure
      - --entrypoints.web.http.redirections.entrypoint.scheme=https

      # Let's Encrypt (HTTP challenge)
      - --certificatesresolvers.letsencrypt.acme.email=${LETSENCRYPT_EMAIL}
      - --certificatesresolvers.letsencrypt.acme.storage=/letsencrypt/acme.json
      - --certificatesresolvers.letsencrypt.acme.httpchallenge=true
      - --certificatesresolvers.letsencrypt.acme.httpchallenge.entrypoint=web

      # Dashboard solo interno (vía 127.0.0.1:8080)
      - --api.dashboard=true

      # Ping interno para healthcheck
      - --ping=true
      - --entrypoints.ping.address=:8082
      - --ping.entrypoint=ping

      # Logs útiles
      - --log.level=INFO
      - --accesslog=true

    ports:
      - "80:80"
      - "443:443"
      # Dashboard SOLO local (loopback en VPS)
      - "127.0.0.1:8080:8080"

    volumes:
      - /var/run/docker.sock:/var/run/docker.sock:ro
      - ./letsencrypt:/letsencrypt

    networks:
      - proxy

    healthcheck:
      test: ["CMD", "traefik", "healthcheck", "--ping"]
      interval: 30s
      timeout: 5s
      retries: 3
      start_period: 15s

  n8n:
    image: docker.n8n.io/n8nio/n8n:2.8.3
    container_name: n8n
    restart: unless-stopped
    networks:
      - proxy

    # Solo interno: visible para Traefik, NO público
    expose:
      - "5678"

    environment:
      - TZ=${TZ}
      - GENERIC_TIMEZONE=${GENERIC_TIMEZONE}

      # n8n detrás de HTTPS (Traefik)
      - N8N_PROTOCOL=https
      - N8N_HOST=${N8N_DOMAIN}
      - N8N_EDITOR_BASE_URL=https://${N8N_DOMAIN}/
      - WEBHOOK_URL=https://${N8N_DOMAIN}/
      - N8N_SECURE_COOKIE=true

      # Encriptación de credenciales
      - N8N_ENCRYPTION_KEY=${N8N_ENCRYPTION_KEY}

      # Configuración de ejecuciones (opcional)
      - EXECUTIONS_DATA_SAVE_ON_ERROR=all
      - EXECUTIONS_DATA_SAVE_ON_SUCCESS=all
      - EXECUTIONS_DATA_PRUNE=true
      - EXECUTIONS_DATA_MAX_AGE=504
      - EXECUTIONS_DATA_PRUNE_MAX_COUNT=10000
      - N8N_CONCURRENCY_PRODUCTION_LIMIT=12

    volumes:
      - n8n_data:/home/node/.n8n

    healthcheck:
      test:
        [
          "CMD",
          "node",
          "-e",
          "require('http').get('http://127.0.0.1:5678',r=>process.exit(r.statusCode<500?0:1)).on('error',()=>process.exit(1))"
        ]
      interval: 30s
      timeout: 10s
      retries: 5
      start_period: 30s

    labels:
      - traefik.enable=true

      # Router HTTPS
      - traefik.http.routers.n8n.rule=Host(`${N8N_DOMAIN}`)
      - traefik.http.routers.n8n.entrypoints=websecure
      - traefik.http.routers.n8n.tls=true
      - traefik.http.routers.n8n.tls.certresolver=letsencrypt

      # Service backend
      - traefik.http.services.n8n.loadbalancer.server.port=5678

      # Rate limit
      - traefik.http.middlewares.n8n-ratelimit.ratelimit.average=100
      - traefik.http.middlewares.n8n-ratelimit.ratelimit.burst=200
      - traefik.http.middlewares.n8n-ratelimit.ratelimit.period=1s

      # Headers de seguridad
      - traefik.http.middlewares.n8n-secure-headers.headers.stsSeconds=31536000
      - traefik.http.middlewares.n8n-secure-headers.headers.stsIncludeSubdomains=true
      - traefik.http.middlewares.n8n-secure-headers.headers.stsPreload=true
      - traefik.http.middlewares.n8n-secure-headers.headers.contentTypeNosniff=true
      - traefik.http.middlewares.n8n-secure-headers.headers.browserXssFilter=true
      - traefik.http.middlewares.n8n-secure-headers.headers.frameDeny=true

      # Aplica middlewares
      - traefik.http.routers.n8n.middlewares=n8n-ratelimit,n8n-secure-headers

  portainer:
    image: portainer/portainer-ce:lts
    container_name: portainer
    restart: unless-stopped
    networks:
      - proxy

    # Solo interno: visible para Traefik, NO público
    expose:
      - "9443"
      # Si NO usas Edge Agents, puedes quitar esto
      - "8000"

    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
      - portainer_data:/data

    labels:
      - traefik.enable=true

      # Router HTTPS
      - traefik.http.routers.portainer.rule=Host(`${PORTAINER_DOMAIN}`)
      - traefik.http.routers.portainer.entrypoints=websecure
      - traefik.http.routers.portainer.tls=true
      - traefik.http.routers.portainer.tls.certresolver=letsencrypt

      # Basic Auth (doble candado)
      - traefik.http.middlewares.portainer-auth.basicauth.users=${PORTAINER_BASICAUTH_HASH}

      # IP allowlist (tu IP pública / rango)
      - traefik.http.middlewares.portainer-ipallow.ipallowlist.sourcerange=${PORTAINER_ALLOWED_IPS}

      # Rate limit (más estricto)
      - traefik.http.middlewares.portainer-ratelimit.ratelimit.average=20
      - traefik.http.middlewares.portainer-ratelimit.ratelimit.burst=40
      - traefik.http.middlewares.portainer-ratelimit.ratelimit.period=1s

      # Headers de seguridad
      - traefik.http.middlewares.portainer-secure-headers.headers.stsSeconds=31536000
      - traefik.http.middlewares.portainer-secure-headers.headers.stsIncludeSubdomains=true
      - traefik.http.middlewares.portainer-secure-headers.headers.contentTypeNosniff=true
      - traefik.http.middlewares.portainer-secure-headers.headers.frameDeny=true

      # Aplica middlewares (orden importante)
      - traefik.http.routers.portainer.middlewares=portainer-ipallow,portainer-auth,portainer-ratelimit,portainer-secure-headers

      # Backend Portainer (HTTPS interno self-signed)
      # Portainer corre HTTPS en 9443 con self-signed por defecto
      - traefik.http.services.portainer.loadbalancer.server.port=9443
      - traefik.http.services.portainer.loadbalancer.server.scheme=https
      - traefik.http.serversTransports.portainer-transport.insecureSkipVerify=true
      - traefik.http.services.portainer.loadbalancer.serverstransport=portainer-transport@docker

  volume-backup:
    image: alpine:3.20
    container_name: volume-backup
    restart: unless-stopped
    environment:
      - TZ=${TZ}
      - BACKUP_RETENTION_DAYS=${BACKUP_RETENTION_DAYS}
      - BACKUP_INTERVAL_SECONDS=${BACKUP_INTERVAL_SECONDS}
    volumes:
      - n8n_data:/src/n8n_data:ro
      - portainer_data:/src/portainer_data:ro
      - ./backups:/backups
    command: >
      sh -c "
      apk add --no-cache tzdata tar gzip coreutils &&
      mkdir -p /backups &&
      while true; do
        TS=$$(date +%F_%H-%M-%S);
        echo '[backup] creando respaldo '$$TS;
        tar -czf /backups/n8n_data_$$TS.tar.gz -C /src n8n_data &&
        tar -czf /backups/portainer_data_$$TS.tar.gz -C /src portainer_data &&
        find /backups -type f -name '*.tar.gz' -mtime +$${BACKUP_RETENTION_DAYS} -delete;
        echo '[backup] completado, siguiente en '$${BACKUP_INTERVAL_SECONDS}'s';
        sleep $${BACKUP_INTERVAL_SECONDS};
      done
      "

volumes:
  # Reutiliza volúmenes existentes (n8n_data y portainer_data)
  # Esto referencia el volumen existente (NO lo recrea ni borra)
  n8n_data:
    external: true
  portainer_data:
    external: true

networks:
  proxy:
    name: proxy
```

## Paso 9) Preparar directorios y permisos (Let's Encrypt + backups)

```bash
mkdir -p letsencrypt backups
touch letsencrypt/acme.json
chmod 600 letsencrypt/acme.json
```

## Paso 10) Validar configuración antes de levantar (recomendado)

```bash
docker compose config
```

> Este comando te ayuda a detectar errores de sintaxis, variables faltantes o problemas de interpolación.

## Paso 11) Levantar stack completo

```bash
docker compose up -d
```

## Paso 12) Verificar estado y logs

```bash
docker compose ps
docker compose logs -f traefik
```

En otra terminal (si quieres):
```bash
docker compose logs -f n8n
docker compose logs -f portainer
docker compose logs -f volume-backup
```

## Paso 13) Probar accesos

- **n8n:** `https://n8n-devtallez.{VPS_DOMAIN}`
- **Portainer:** `https://portainer.{VPS_DOMAIN}`

### Flujo de acceso de Portainer (doble seguridad)
1. Te pedirá **Basic Auth** (Traefik)
2. Luego verás el **login de Portainer**

## Paso 14) Acceder al dashboard de Traefik (solo interno por SSH tunnel)

Desde tu computadora local:

```bash
ssh -L 8080:127.0.0.1:8080 usuario@VPS
```

Luego abre:
- `http://localhost:8080/dashboard/`

> **Por qué se hace así:** evita exponer el dashboard a internet, aunque tengas auth. Es más seguro.
> Esto funciona porque Traefik expone el dashboard solo en loopback del VPS (127.0.0.1), no en internet.
> Si quieres exponer el dashboard por dominio, puedes descomentar las líneas correspondientes en el archivo `docker-compose.yml` y configurar el dominio en la variable `TRAEFIK_DOMAIN`.


## Paso 15) Comprobar que n8n y Portainer ya NO están expuestos directamente

### Ver puertos Docker publicados
```bash
docker ps --format "table {{.Names}}\t{{.Ports}}"
```

### Resultado esperado
- `traefik` → `80`, `443`, y `127.0.0.1:8080`
- `n8n` → **sin** `0.0.0.0:5678->...`
- `portainer` → **sin** `0.0.0.0:9443->...`

### Ver puertos a nivel sistema (opcional)
```bash
sudo ss -tulpn | grep -E '(:80|:443|:8080|:5678|:9443)'
```

Resultado esperado:
- `:80, :443` públicos
- `127.0.0.1:8080` local
- `No 0.0.0.0:5678 ni 0.0.0.0:9443`

---

# Explicación de cada configuración (`.env`)

## `N8N_DOMAIN`
Dominio público donde abrirás n8n.

## `PORTAINER_DOMAIN`
Dominio público donde abrirás Portainer.

## `TRAEFIK_DOMAIN`
Subdominio reservado para Traefik. En esta guía el dashboard no se expone públicamente, pero puedes conservarlo para organización futura.

## `LETSENCRYPT_EMAIL`
Correo usado por Let's Encrypt para notificaciones (renovación, errores, expiración de certificado, etc.).

## `TZ` / `GENERIC_TIMEZONE`
Zona horaria para n8n y contenedores auxiliares (ej. backups).

## `N8N_ENCRYPTION_KEY`
Llave fija para cifrar credenciales en n8n.
- Debe ser estable (fija)
- **No debe cambiar** (si cambia, no podrás descifrar credenciales guardadas en n8n)
- Guárdala en un password manager

## `PORTAINER_BASICAUTH_HASH`
Hash BCrypt para Basic Auth de Traefik antes de llegar a Portainer.

## `PORTAINER_ALLOWED_IPS`
Lista de IPs/rangos permitidos para Portainer (IP allowlist). Ejemplo: `203.0.113.10/32`.

## `BACKUP_RETENTION_DAYS`
Días que se conservan backups.

## `BACKUP_INTERVAL_SECONDS`
Frecuencia de backup en segundos (`86400` = 24h).

---

# Explicación de cada service en `docker-compose.yml`

## 1) Servicio `traefik` (reverse proxy + HTTPS)

### Qué hace
Traefik es el reverse proxy que recibe todo el tráfico HTTP/HTTPS y lo enruta a cada contenedor `n8n` y `portainer` por dominio.

```yaml
image: traefik:v3.6.8
```
Usa Traefik versión fija (mejor que latest para evitar cambios inesperados).

Si quieres actualizar Traefik, edita el tag en el archivo `docker-compose.yml` y ejecuta:
```bash
docker compose pull
docker compose up -d
```

### Puntos clave
- image: traefik:v3.6.8 → usa la versión 3.6.8 (mejor que latest para evitar cambios inesperados)
- `providers.docker=true` → detecta contenedores (lea servicios) Docker automáticamente
- `exposedbydefault=false` → seguridad (solo expone contenedores con `traefik.enable=true`)
- `entrypoints.web` y `websecure` address → puertos 80/443
- Redirección HTTP→HTTPS automática. Todo lo que llegue se redirige a HTTPS.
```yaml
- --entrypoints.web.http.redirections.entrypoint.to=websecure
- --entrypoints.web.http.redirections.entrypoint.scheme=https
```
- Let's Encrypt por HTTP challenge (ACME)
- Permite emitir y renovar certificados automáticamente.
```yaml
- --certificatesresolvers.letsencrypt.acme.email=${LETSENCRYPT_EMAIL}
- --certificatesresolvers.letsencrypt.acme.storage=/letsencrypt/acme.json
- --certificatesresolvers.letsencrypt.acme.httpchallenge=true
- --certificatesresolvers.letsencrypt.acme.httpchallenge.entrypoint=web
```
- Dashboard habilitado **pero solo local** (`127.0.0.1:8080`) (loopback)
```yaml
- --api.dashboard=true
```
- `--ping=true` + `healthcheck` → healthcheck nativo
- `docker.sock` en `:ro` (solo lectura) → Traefik detecta servicios
- Volumen `./letsencrypt` → guarda certificados (Let's Encrypt)
- ports: 80/443 → puertos públicos (HTTP/HTTPS)
```yaml
- "80:80"
- "443:443"
```
- `127.0.0.1:8080:8080` → dashboard solo local (loopback)
```yaml
- "127.0.0.1:8080:8080"
```
- volumes:
- `docker.sock` en `:ro` (solo lectura) → Traefik detecta servicios
- Volumen `./letsencrypt` → guarda certificados (Let's Encrypt)
```yaml
  - /var/run/docker.sock:/var/run/docker.sock:ro
  - ./letsencrypt:/letsencrypt
```
- `networks:` → conecta Traefik a la red Docker proxy para alcanzar n8n y Portainer.
```yaml
  proxy:
    name: proxy # nombre de la red Docker
```
- `healthcheck:` → checkea la salud de Traefik.

### Por qué el dashboard no es público
Aunque tenga auth, sigue siendo una superficie de ataque. Con loopback + SSH tunnel, solo tú con acceso SSH puedes verlo.

---

## 2) Servicio `n8n` (automatizaciones)

### Qué hace
Ejecuta n8n y sirve el editor + webhooks. Es tu motor de automatización.

### Puntos clave
- image: docker.n8n.io/n8nio/n8n:2.8.3 → usa la versión 2.8.3 (imagen del registry oficial de n8n)
- `expose: 5678` → puerto interno (visible para Traefik, no internet)
- Variables HTTPS:
  - `N8N_PROTOCOL=https`
  - `N8N_HOST`
  - `N8N_EDITOR_BASE_URL`
  - `WEBHOOK_URL`
  - `N8N_SECURE_COOKIE=true`
  Muy importantes para URLs correctas del editor, webhooks funcionales, cookies seguras, etc.
- `N8N_ENCRYPTION_KEY` → cifra credenciales guardadas
- Volumen `n8n_data` → persistencia de workflows/config
- Labels Traefik:
  - Router por `Host(...)`
  - TLS + certresolver
  - Service al puerto interno `5678`
- Middlewares de seguridad:
  - Rate limit
  - Headers (HSTS, frame deny, nosniff, etc.)
- Healthcheck con Node → valida que responda en `127.0.0.1:5678`

Variables de ejecuciones (opcionales)
Controlan retención de datos, pruning y límites.
```yaml
- EXECUTIONS_DATA_SAVE_ON_ERROR=all
- EXECUTIONS_DATA_SAVE_ON_SUCCESS=all
- EXECUTIONS_DATA_PRUNE=true
- EXECUTIONS_DATA_MAX_AGE=504
- EXECUTIONS_DATA_PRUNE_MAX_COUNT=10000
- N8N_CONCURRENCY_PRODUCTION_LIMIT=12
```
- volumes:
```yaml
- n8n_data:/home/node/.n8n
```
---

## 3) Servicio `portainer` (administración Docker)

### Qué hace
UI para administrar Docker/containers/volúmenes.

### Puntos clave
- image: portainer/portainer-ce:lts → usa la versión LTS (estable recomendada por Portainer)
- `expose: 9443` → UI HTTPS interna (no pública)
- `expose: 8000` → Edge agents (opcional)
- Volúmenes:
  - `/var/run/docker.sock` → acceso a Docker host
  - `portainer_data` → persistencia
- Labels Traefik:
  - Router HTTPS por `PORTAINER_DOMAIN`
  - TLS + certresolver
  - Backend HTTPS interno (`server.scheme=https`)
  - `insecureSkipVerify=true` Portainer suele usar cert self-signed interno, por lo que Traefik debe aceptarlo.
- Middlewares de seguridad:
  - **IP allowlist** (solo tu IP)
  - **Basic Auth** (doble candado)
  - **Rate limit**
  - Headers de seguridad

### Flujo real de seguridad de Portainer
1. **IP allowlist** → si no eres tu IP, ni siquiera entra
2. **Basic Auth** de Traefik
3. **Login propio de Portainer**

---

## 4) Servicio `volume-backup` (backups automáticos)

### Qué hace
Crea respaldos `.tar.gz` de `n8n_data` y `portainer_data` de forma periódica.

### Puntos clave
- Monta volúmenes en **solo lectura**
- Guarda backups en `./backups`
- Ejecuta cada `BACKUP_INTERVAL_SECONDS`
- Borra backups viejos (`BACKUP_RETENTION_DAYS`)
- Usa `alpine` con `tar`, `gzip`, `find`

### Importante
Para restaurar, detén temporalmente el servicio correspondiente (`n8n` o `portainer`) antes de extraer backup.

---

# Tabla de comandos (referencia rápida)

| Comando | Qué hace | Cuándo usarlo |
|---|---|---|
| `docker version` | Muestra versión de Docker | Verificar instalación |
| `docker compose version` | Muestra versión de Compose plugin | Verificar instalación |
| `docker ps --format "table {{.Names}}\t{{.Ports}}"` | Ver puertos publicados por contenedor | Verificar exposición de puertos |
| `docker volume ls` | Listar volúmenes | Confirmar `n8n_data` / `portainer_data` |
| `docker stop n8n portainer` | Detener contenedores | Antes de migrar |
| `docker rm n8n portainer` | Eliminar contenedores | Recrear con compose |
| `mkdir -p letsencrypt backups` | Crear carpetas locales | Preparación |
| `touch letsencrypt/acme.json && chmod 600 letsencrypt/acme.json` | Preparar archivo de certificados | Preparación Traefik |
| `docker compose config` | Validar compose y variables | Antes de levantar |
| `docker compose up -d` | Levantar stack en segundo plano | Iniciar o aplicar cambios |
| `docker compose ps` | Ver estado de servicios | Ver si todo subió |
| `docker compose logs -f traefik` | Logs de Traefik | Troubleshooting HTTPS/rutas/auth |
| `docker compose logs -f n8n` | Logs de n8n | Debug de n8n |
| `docker compose logs -f portainer` | Logs de Portainer | Debug de Portainer |
| `docker compose logs -f volume-backup` | Logs del backup automático | Verificación backups |
| `docker compose restart` | Reiniciar todo | Mantenimiento |
| `docker compose restart n8n` | Reiniciar sólo n8n | Cambios puntuales |
| `docker compose stop` | Detener stack (sin borrar) | Mantenimiento |
| `docker compose down` | Bajar stack y quitar contenedores/red | Rebuild o mantenimiento |
| `docker compose pull` | Descargar nuevas imágenes | Actualizaciones |
| `docker image prune -f` | Limpiar imágenes no usadas | Liberar espacio |
| `sudo ss -tulpn | grep ...` | Ver puertos escuchando en host | Confirmar exposición real |
| `htpasswd -nbB admin 'PASS' \| sed -e 's/\$/\$\$/g'` | Generar hash Basic Auth compatible con YAML | Configurar protección de Traefik/Portainer |
| `ssh -L 8080:127.0.0.1:8080 usuario@VPS` | Túnel SSH al dashboard de Traefik | Acceso interno seguro |
| `pandoc guia.md -o guia.pdf` | Exportar la guía a PDF | Documentación |

---

# Comandos de operación diaria (iniciar, detener, reiniciar, actualizar)

## Iniciar todo o aplicar cambios / levantar stack
```bash
docker compose up -d
```

## Ver estado
```bash
docker compose ps
```

## Ver logs
```bash
docker compose logs -f traefik
docker compose logs -f n8n
docker compose logs -f portainer
docker compose logs -f volume-backup
```

## Reiniciartodo
```bash
docker compose restart
```

## Reiniciar servicios individuales
```bash
docker compose restart traefik
docker compose restart n8n
docker compose restart portainer
```

## Detener (sin borrar)
```bash
docker compose stop
```

## Bajar stack (elimina contenedores y red del stack)
```bash
docker compose down
```

## Actualizar a versiones más nuevas (Traefik / n8n / Portainer)
1. Edita los tags en `docker-compose.yml` (recomendado usar versiones fijas)
2. Ejecuta:
```bash
docker compose pull
docker compose up -d
```
3. Verifica logs de Traefik y n8n para asegurarte de que todo esté funcionando correctamente.
```bash
docker compose logs -f traefik
docker compose logs -f n8n
```
> Tip: actualiza primero en horarios de bajo uso, especialmente n8n.

## Limpiar imágenes viejas
```bash
docker image prune -f
# o más agresivo:
docker system prune -f
```

---

# Cómo actualizar Docker en Ubuntu

## Ver versión actual
```bash
docker version
docker compose version
```

## Actualizar Docker Engine + Compose Plugin
```bash
sudo apt update
sudo apt install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
```

## Reiniciar Docker
```bash
sudo systemctl restart docker
sudo systemctl status docker
```

---

# Backups automáticos y restauración

## Ver backups generados
```bash
ls -lh backups/
```

## Restaurar n8n desde un backup (ejemplo)

> **Detén n8n antes de restaurar** para evitar corrupción.

```bash
docker compose stop n8n
docker run --rm -v n8n_data:/restore -v $(pwd)/backups:/backups alpine:3.20 \
  sh -c "cd /restore && rm -rf ./* && tar -xzf /backups/n8n_data_YYYY-MM-DD_HH-MM-SS.tar.gz --strip-components=1"
docker compose start n8n
```

## Restaurar Portainer desde un backup (ejemplo)

```bash
docker compose stop portainer
docker run --rm -v portainer_data:/restore -v $(pwd)/backups:/backups alpine:3.20 \
  sh -c "cd /restore && rm -rf ./* && tar -xzf /backups/portainer_data_YYYY-MM-DD_HH-MM-SS.tar.gz --strip-components=1"
docker compose start portainer
```

---

# Troubleshooting (problemas comunes y soluciones)

## 1) Let's Encrypt no emite certificado

### Síntomas
- Error de TLS
- Traefik logs con errores ACME
- El sitio no abre por HTTPS

### Causas comunes
- DNS no apunta al VPS
- Puerto 80 bloqueado
- Otro proceso usa el puerto 80
- `acme.json` sin permisos correctos

### Solución
```bash
docker compose logs -f traefik
sudo ss -tulpn | grep ':80'
ls -l letsencrypt/acme.json
```
Asegúrate de:
- DNS correcto
- UFW permite `80/443`
- `chmod 600 letsencrypt/acme.json`

---

## 2) n8n abre pero webhooks no funcionan

### Causa típica
Variables de URL mal configuradas (http vs https o dominio incorrecto)

### Verifica en `n8n.environment`
```yaml
- N8N_PROTOCOL=https
- N8N_HOST=${N8N_DOMAIN}
- N8N_EDITOR_BASE_URL=https://${N8N_DOMAIN}/
- WEBHOOK_URL=https://${N8N_DOMAIN}/
```

### Aplica cambios
```bash
docker compose up -d
docker compose restart n8n
```

---

## 3) Portainer no carga detrás de Traefik

### Causa común
Traefik no está configurado para conectar por HTTPS al backend de Portainer.

### Verifica labels clave
```yaml
- traefik.http.services.portainer.loadbalancer.server.scheme=https
- traefik.http.serversTransports.portainer-transport.insecureSkipVerify=true
- traefik.http.services.portainer.loadbalancer.serverstransport=portainer-transport@docker
```

---

## 4) Me bloqueé del Portainer por IP allowlist

### Solución
Actualiza tu IP en `.env`:
```env
PORTAINER_ALLOWED_IPS=TU.IP.PUBLICA.NUEVA/32
```
Luego:
```bash
docker compose up -d
```

> Si necesitas acceso urgente, puedes quitar temporalmente `portainer-ipallow` de la cadena de middlewares de Portainer y aplicar cambios.

---

## 5) El backup container no genera archivos

### Verifica logs
```bash
docker compose logs -f volume-backup
```

### Verifica carpeta
```bash
ls -lh backups/
```

### Posibles causas
- Carpeta `./backups` no existe
- Problemas de permisos del host
- El contenedor `volume-backup` no inició

---

## 6) n8n aparece `unhealthy`

### Qué revisar
```bash
docker compose ps
docker compose logs -f n8n
```

### Nota
Al arrancar puede tardar algunos segundos. El `start_period` ya contempla eso.

---

## 7) Dashboard de Traefik no abre por SSH tunnel

### Verifica
- Que Traefik expone `127.0.0.1:8080:8080`
- Que el túnel SSH está activo

### Túnel correcto
```bash
ssh -L 8080:127.0.0.1:8080 usuario@VPS
```

### URL correcta
- `http://localhost:8080/dashboard/`

---

## 8) Error `volume not found`
Si usas `external: true` y el volumen no existe:

### Verifica si los volúmenes existen
```bash
docker volume ls
```
Si no existen, créalos:
```bash
docker volume create n8n_data
docker volume create portainer_data
```

---

## 9) `sudo: user is not in sudoers file`
Entra con `root` o con un usuario con sudo. Agrega el usuario al grpupo sudo:
```bash
usermod -aG sudo TU_USUARIO
```

## 10) N8N_ENCRYPTION_KEY rota credenciales

### Causa
La llave de encriptación se perdió o se cambió (configuró) después de crear credenciales en n8n.

### Solución
Si ya existía un volumen `n8n_data`, leer la llave de encriptación del volumen:
```bash
docker exec -it n8n cat /home/node/.n8n/config.json | grep -E 'encryptionKey|N8N_ENCRYPTION_KEY'
```
Si no existe, genera una nueva:
```bash
KEY="$(openssl rand -base64 32)" && \
grep -q '^N8N_ENCRYPTION_KEY=' .env \
  && sed -i "s|^N8N_ENCRYPTION_KEY=.*|N8N_ENCRYPTION_KEY=$KEY|" .env \
  || echo "N8N_ENCRYPTION_KEY=$KEY" >> .env
```
---

# Glosario de terminología

## Docker
Plataforma para ejecutar aplicaciones en contenedores.

## Contenedor
Instancia aislada de una aplicación (n8n, Traefik, Portainer).

## Imagen
Plantilla para crear contenedores (ej. `traefik:v3.6.8`).

## Docker Compose
Forma de describir múltiples contenedores en un solo archivo YAML.

## Volumen
Almacenamiento persistente fuera del contenedor. Conserva datos al recrear contenedores.

## Reverse Proxy
Servicio que recibe tráfico HTTP/HTTPS y lo redirige a otros servicios.

## Traefik
Reverse proxy moderno con integración nativa para Docker y Let's Encrypt. Reverse proxy moderno para Docker/Kubernetes, con soporte para HTTPS automático, TLS, certificados, etc.

## Let's Encrypt
Autoridad certificadora gratuita para emitir certificados TLS/SSL.

## ACME
Protocolo usado para solicitar/renovar certificados Let's Encrypt.

## HTTP Challenge
Método de validación de dominio usando el puerto 80.

## Entrypoint (Traefik)
Puerto/protocolo donde Traefik escucha  (ej. web=80, websecure=443).

## Router (Traefik)
Regla que decide qué servicio recibe el tráfico (por dominio, path, etc.).

## Service (Traefik)
Destino interno al que Traefik reenvía tráfico (puerto del contenedor).

## Middleware (Traefik)
Procesamiento intermedio: auth, rate limit, headers, IP allowlist, etc.

## Basic Auth
Autenticación simple (usuario/contraseña) a nivel HTTP (como capa extra de seguridad).

## IP allowlist
Filtro que solo permite acceso desde IPs específicas.

## Rate limit
Límite de solicitudes por unidad de tiempo para mitigar abuso.

## Healthcheck
Chequeo automático de salud de un contenedor.

## Loopback (`127.0.0.1`)
Interfaz local del servidor, no expuesta a internet.

## SSH Tunnel (Port Forwarding)
Túnel seguro para acceder desde tu computadora a un puerto interno del VPS.

## `expose` (Docker Compose)
Hace visible un puerto solo a otros contenedores de la misma red Docker.

## `ports` (Docker Compose)
Publica un puerto del contenedor hacia el host (potencialmente público).

## `docker.sock`
Socket del daemon Docker; permite a Traefik y Portainer descubrir/gestionar contenedores.

---

# Preguntas frecuentes (FAQ)

## 1) ¿Por qué no usar `latest` en las imágenes?
Porque puede cambiar sin aviso y romper compatibilidad. Es mejor usar versiones fijas (pinneadas).

## 2) ¿Puedo seguir entrando a n8n por `:5678`?
No en esta arquitectura. Ahora se accede por:
- `https://n8n-devtallez.{VPS_DOMAIN}`

## 3) ¿Por qué Traefik dashboard no es público?
Por seguridad. Aunque tenga auth, exponerlo aumenta superficie de ataque. Loopback + SSH es más seguro.

## 4) ¿Portainer necesita el puerto 8000?
Solo si usas Edge Agents. Si no, puedes quitarlo de `expose`.

## 5) ¿Puedo quitar la IP allowlist de Portainer?
Sí, eliminando `portainer-ipallow` de los middlewares del router. No es lo más seguro, pero se puede.

## 6) ¿Puedo poner IP allowlist también a n8n?
Sí. Crea un middleware igual y agrégalo a `traefik.http.routers.n8n.middlewares`.

## 7) ¿Qué pasa si pierdo `N8N_ENCRYPTION_KEY`?
No podrás descifrar credenciales guardadas en n8n. Guárdala bien.

## 8) ¿Qué hago si mi IP pública cambia seguido?
Actualiza `PORTAINER_ALLOWED_IPS` en `.env` y ejecuta `docker compose up -d`.

## 9) ¿Cómo actualizo Traefik/n8n/Portainer?
Edita tags en el compose, luego:
```bash
docker compose pull
docker compose up -d
```

## 10) ¿Cómo exporto esta guía?
Puedes guardarla como `.md` y convertirla con `pandoc` a PDF o HTML (ver sección de exportación).

## 11) ¿Qué pasa si reinicio el VPS?
Si reinicias el VPS, los contenedores se reiniciarán y perderás la configuración actual.
Para evitar esto, puedes usar un script de reinicio que guarde la configuración actual y la restaure después del reinicio.
```bash
sudo nano /etc/init.d/reboot.sh
```
```bash
sudo chmod +x /etc/init.d/reboot.sh
```
> Los servicios se levantan solos por el `restart: unless-stopped` en el compose.

## 12) ¿Cómo puedo hacer backups de los volúmenes?
Puedes usar `docker compose exec` para ejecutar comandos en los contenedores.
```bash
docker compose exec n8n tar -czvf /backups/n8n_data_$(date +%Y%m%d).tar.gz /home/node/.n8n
```
```bash
docker compose exec portainer tar -czvf /backups/portainer_data_$(date +%Y%m%d).tar.gz /var/lib/docker/volumes/portainer_data/_data
```

## 13) ¿Cómo puedo restaurar los volúmenes desde un backup?
Puedes usar `docker compose exec` para ejecutar comandos en los contenedores.
```bash
docker compose exec n8n tar -xzvf /backups/n8n_data_$(date +%Y%m%d).tar.gz -C /home/node/.n8n
```
```bash
docker compose exec portainer tar -xzvf /backups/portainer_data_$(date +%Y%m%d).tar.gz -C /var/lib/docker/volumes/portainer_data/_data
```

## 14) ¿Puedo agregar más apps detrás de Traefik?
Sí. Solo agregas otro service con labels de Traefik y un dominio/subdominio.

## 15) ¿Puedo quitar el Basic Auth de Portainer?
Sí. Solo elimina el middleware `portainer-basicauth` de los labels de Portainer.
> No es recomendado. Es una capa extra de seguridad muy útil.

---

# Exportar la guía (Markdown → PDF/HTML)

## Guardar este documento
Nombre sugerido:
- `guia-avanzada-completa-traefik-n8n-portainer.md`

## Convertir a PDF (Ubuntu)
Instala Pandoc:
```bash
sudo apt update && sudo apt install -y pandoc
```

Convierte:
```bash
pandoc guia-avanzada-completa-traefik-n8n-portainer.md -o guia-avanzada-completa-traefik-n8n-portainer.pdf
```

## Convertir a HTML
```bash
pandoc guia-avanzada-completa-traefik-n8n-portainer.md -o guia-avanzada-completa-traefik-n8n-portainer.html
```

## Abrir/editar en VS Code
```bash
code guia-avanzada-completa-traefik-n8n-portainer.md
```

---

# Checklist final (paso a paso)

- [ ] DNS de `n8n-devtallez.{VPS_DOMAIN}` apunta al VPS
- [ ] DNS de `portainer.{VPS_DOMAIN}` apunta al VPS
- [ ] DNS de `traefik.{VPS_DOMAIN}` apunta al VPS (opcional para esta versión)
- [ ] Puertos `80/tcp` y `443/tcp` abiertos en el VPS (UFW permite `80/tcp` y `443/tcp`)
- [ ] Puertos `5678/tcp` y `9443/tcp` cerrados en el VPS (solo internos por Docker network)
- [ ] Puertos `8080/tcp` cerrado en el VPS (solo loopback `127.0.0.1`)
- [ ] Existen volúmenes `n8n_data` y `portainer_data`
- [ ] `.env` creado con dominios, email y timezones
- [ ] `N8N_ENCRYPTION_KEY` generada y guardada
- [ ] `PORTAINER_BASICAUTH_HASH` generado y guardado
- [ ] `PORTAINER_ALLOWED_IPS` configurado con tu IP pública
- [ ] `docker-compose.yml` creado
- [ ] `letsencrypt/acme.json` creado y con `chmod 600`
- [ ] Carpeta `backups/` creada
- [ ] `docker compose config` pasa sin errores
- [ ] `docker compose up -d` ejecutado
- [ ] `docker compose ps` muestra servicios arriba
- [ ] n8n responde en HTTPS
- [ ] Portainer responde en HTTPS (doble auth + allowlist)
- [ ] `docker ps` confirma que `5678` y `9443` no están públicos
- [ ] Backups `.tar.gz` aparecen en `./backups`
- [ ] Dashboard Traefik accesible por SSH tunnel

---

## Nota final

Si más adelante quieres una versión aún más robusta, se puede extender con:
- Backups remotos (S3/Backblaze/Google Drive con `rclone`)
- Alertas (Telegram/Slack) para healthchecks/backups
- Autenticación adicional (Authelia, OAuth, etc.)
- Monitoreo con Prometheus/Grafana
