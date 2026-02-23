# Guía avanzada: n8n + Portainer detrás de Traefik (HTTPS) con seguridad y operación pro

## Qué incluye esta versión avanzada

Además de la configuración base (Traefik + n8n + Portainer + HTTPS), esta versión agrega:

1. **Backups automáticos de volúmenes** (`n8n_data`, `portainer_data`)
2. **Rate limiting en Traefik** (para n8n y Portainer)
3. **IP allowlist para Portainer** (opcional, pero recomendado)
4. **Healthchecks básicos** (Traefik y n8n; y validación externa por Traefik para Portainer)

---

# Arquitectura final (versión avanzada)

- **Traefik** expone solo:
  - `80/tcp` (HTTP + challenge Let’s Encrypt)
  - `443/tcp` (HTTPS)
  - `127.0.0.1:8080` (dashboard de Traefik solo local vía túnel SSH)
- **n8n** corre interno (`expose: 5678`) y se publica por:
  - `https://n8n-devtallez.pixelia.cloud`
- **Portainer** corre interno (`expose: 9443`) y se publica por:
  - `https://portainer.pixelia.cloud`
- **Backups automáticos** se guardan en `./backups` del host
- **Traefik Dashboard** solo por SSH tunnel:
  - `http://localhost:8080/dashboard/`

---

# Archivo `.env` (completo, versión avanzada)

> Crea este archivo en la misma carpeta del `docker-compose.yml`

```env
# Dominios
N8N_DOMAIN=n8n-devtallez.pixelia.cloud
PORTAINER_DOMAIN=portainer.pixelia.cloud
TRAEFIK_DOMAIN=traefik.pixelia.cloud

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
# Ejemplo: tu IP pública /32 (una sola IP)
# Si cambia seguido tu IP, puedes comentar el middleware allowlist en compose
PORTAINER_ALLOWED_IPS=TU.IP.PUBLICA.ACTUAL/32

# Backups automáticos
BACKUP_RETENTION_DAYS=14
BACKUP_INTERVAL_SECONDS=86400
```

---

# Generar variables sensibles (comandos)

## 1) Generar `N8N_ENCRYPTION_KEY` (Ubuntu)
Genera y guarda automáticamente en `.env`:

```bash
KEY="$(openssl rand -base64 32)" && \
grep -q '^N8N_ENCRYPTION_KEY=' .env \
  && sed -i "s|^N8N_ENCRYPTION_KEY=.*|N8N_ENCRYPTION_KEY=$KEY|" .env \
  || echo "N8N_ENCRYPTION_KEY=$KEY" >> .env
```

Verificar:
```bash
grep '^N8N_ENCRYPTION_KEY=' .env
```

---

## 2) Generar Basic Auth hash para Portainer (Traefik)
Instala `htpasswd` si hace falta:

```bash
sudo apt update && sudo apt install -y apache2-utils
```

Genera el hash:
```bash
htpasswd -nbB admin 'TU_PASSWORD_SUPER_SEGURA' | sed -e 's/\$/\$\$/g'
```

Copia el resultado completo y pégalo en:
```env
PORTAINER_BASICAUTH_HASH=admin:$$2y$$...
```

---

# `docker-compose.yml` (versión avanzada)

> Reutiliza tus volúmenes existentes: `n8n_data` y `portainer_data`

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

      # Dashboard (solo interno)
      - --api.dashboard=true

      # Ping interno para healthcheck
      - --ping=true
      - --entrypoints.ping.address=:8082
      - --ping.entrypoint=ping

      # Logs (útil para troubleshooting)
      - --log.level=INFO
      - --accesslog=true

    ports:
      - "80:80"
      - "443:443"
      # Dashboard solo local en VPS (loopback)
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

      # Encriptación de credenciales (fija)
      - N8N_ENCRYPTION_KEY=${N8N_ENCRYPTION_KEY}

      # Ejecuciones / retención (opcionales)
      - EXECUTIONS_DATA_SAVE_ON_ERROR=all
      - EXECUTIONS_DATA_SAVE_ON_SUCCESS=all
      - EXECUTIONS_DATA_PRUNE=true
      - EXECUTIONS_DATA_MAX_AGE=504
      - EXECUTIONS_DATA_PRUNE_MAX_COUNT=10000
      - N8N_CONCURRENCY_PRODUCTION_LIMIT=12

    volumes:
      - n8n_data:/home/node/.n8n

    # Healthcheck básico (usa Node que sí está en la imagen de n8n)
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

      # Rate limit (protección básica)
      - traefik.http.middlewares.n8n-ratelimit.ratelimit.average=100
      - traefik.http.middlewares.n8n-ratelimit.ratelimit.burst=200
      - traefik.http.middlewares.n8n-ratelimit.ratelimit.period=1s

      # Headers de seguridad básicos
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

    expose:
      - "9443"
      # Solo si usas Edge Agents; si no, puedes quitarlo
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

      # Doble candado: Basic Auth (Traefik)
      - traefik.http.middlewares.portainer-auth.basicauth.users=${PORTAINER_BASICAUTH_HASH}

      # Rate limit Portainer (más estricto)
      - traefik.http.middlewares.portainer-ratelimit.ratelimit.average=20
      - traefik.http.middlewares.portainer-ratelimit.ratelimit.burst=40
      - traefik.http.middlewares.portainer-ratelimit.ratelimit.period=1s

      # IP allowlist (tu IP pública)
      - traefik.http.middlewares.portainer-ipallow.ipallowlist.sourcerange=${PORTAINER_ALLOWED_IPS}

      # Headers de seguridad
      - traefik.http.middlewares.portainer-secure-headers.headers.stsSeconds=31536000
      - traefik.http.middlewares.portainer-secure-headers.headers.stsIncludeSubdomains=true
      - traefik.http.middlewares.portainer-secure-headers.headers.contentTypeNosniff=true
      - traefik.http.middlewares.portainer-secure-headers.headers.frameDeny=true

      # Middlewares en orden: allowlist -> auth -> ratelimit -> headers
      - traefik.http.routers.portainer.middlewares=portainer-ipallow,portainer-auth,portainer-ratelimit,portainer-secure-headers

      # Portainer backend HTTPS en 9443 (self-signed)
      - traefik.http.services.portainer.loadbalancer.server.port=9443
      - traefik.http.services.portainer.loadbalancer.server.scheme=https

      # Traefik acepta cert self-signed de Portainer
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
  n8n_data:
    external: true
  portainer_data:
    external: true

networks:
  proxy:
    name: proxy
```

---

# Explicación de las mejoras avanzadas

## 1) Backups automáticos de volúmenes
Se agrega el servicio `volume-backup`:

- Monta los volúmenes **en solo lectura**:
  - `n8n_data`
  - `portainer_data`
- Genera archivos `.tar.gz` en `./backups`
- Corre en bucle cada `BACKUP_INTERVAL_SECONDS` (por defecto 24h)
- Borra backups antiguos según `BACKUP_RETENTION_DAYS`

### Ventajas
- No dependes de tareas manuales
- Los datos críticos se respaldan de forma continua
- Los backups quedan en el host (`./backups`) y son fáciles de copiar a otro servidor / nube

### Restauración (referencia)
> Hazlo con contenedores detenidos para evitar corrupción.

Ejemplo restaurar n8n:
```bash
docker compose stop n8n
docker run --rm -v n8n_data:/restore -v $(pwd)/backups:/backups alpine:3.20 sh -c "cd /restore && rm -rf ./* && tar -xzf /backups/n8n_data_YYYY-MM-DD_HH-MM-SS.tar.gz --strip-components=1"
docker compose start n8n
```

---

## 2) Rate limiting (Traefik)
Agregamos middlewares `ratelimit` para:

- **n8n** (más permisivo)
- **Portainer** (más estricto)

### ¿Para qué sirve?
Reduce abuso, bots, scans agresivos y picos de requests.

### Valores recomendados (base)
- `n8n`: `average=100`, `burst=200`
- `Portainer`: `average=20`, `burst=40`

Puedes ajustarlos según tu tráfico real.

---

## 3) IP allowlist para Portainer
Middleware:
```yaml
traefik.http.middlewares.portainer-ipallow.ipallowlist.sourcerange=${PORTAINER_ALLOWED_IPS}
```

### ¿Para qué sirve?
Solo permite acceso desde tu IP pública (o rango de IPs).

### Recomendación
- Si tienes IP fija → excelente
- Si tu IP cambia seguido → actualiza `.env` y reinicia:
  ```bash
  docker compose up -d
  ```

> Si quieres desactivar temporalmente la allowlist, quita `portainer-ipallow` de la cadena de middlewares.

---

## 4) Healthchecks básicos
### Traefik
Usa `traefik healthcheck --ping` con `--ping=true`.

### n8n
Usa un healthcheck con Node para probar `http://127.0.0.1:5678`.

### Portainer
No añadimos `healthcheck` directo dentro del contenedor porque la imagen no siempre trae `curl/wget`.
**En su lugar**, Traefik valida el backend por HTTPS al enrutar requests (y verás errores claros en logs si falla).

> Si quieres healthcheck interno de Portainer sí o sí, se puede hacer con un sidecar o una imagen custom con `curl`.

---

# Comandos (operación diaria)

## Levantar stack
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

## Reiniciar
```bash
docker compose restart
docker compose restart n8n
docker compose restart portainer
docker compose restart traefik
```

## Detener / bajar
```bash
docker compose stop
docker compose down
```

## Actualizar imágenes (futuro)
1. Cambia tags en `docker-compose.yml` (recomendado)
2. Ejecuta:
```bash
docker compose pull
docker compose up -d
```

## Verificar que NO están expuestos `5678` y `9443`
```bash
docker ps --format "table {{.Names}}\t{{.Ports}}"
```

---

# Exportar la guía (Markdown)

## Opción 1: Guardar este contenido como archivo `.md`
Guarda este documento como:
- `guia-avanzada-traefik-n8n-portainer.md`

## Opción 2: Convertir a PDF desde Ubuntu (rápido con Pandoc)
Instala:
```bash
sudo apt update && sudo apt install -y pandoc
```

Convierte:
```bash
pandoc guia-avanzada-traefik-n8n-portainer.md -o guia-avanzada-traefik-n8n-portainer.pdf
```

## Opción 3: Convertir a HTML
```bash
pandoc guia-avanzada-traefik-n8n-portainer.md -o guia-avanzada-traefik-n8n-portainer.html
```

## Opción 4: Abrir/editar en VS Code
```bash
code guia-avanzada-traefik-n8n-portainer.md
```

---

# Troubleshooting adicional (versión avanzada)

## A) Me bloqueé del Portainer por IP allowlist
- Actualiza `PORTAINER_ALLOWED_IPS` en `.env` con tu IP pública actual
- Aplica cambios:
  ```bash
  docker compose up -d
  ```

## B) El backup container no genera archivos
- Revisa logs:
  ```bash
  docker compose logs -f volume-backup
  ```
- Verifica carpeta:
  ```bash
  ls -lh backups/
  ```

## C) El healthcheck de n8n aparece `unhealthy`
- Espera 30–60s después del arranque (el `start_period` ayuda)
- Revisa logs:
  ```bash
  docker compose logs -f n8n
  ```

## D) HTTPS funciona en n8n pero no en Portainer
Verifica estas labels (son clave):
- `server.scheme=https`
- `insecureSkipVerify=true`
- `serverstransport=...`

---

# FAQ (avanzado)

## ¿Puedo hacer backup en otro horario?
Sí. Cambia:
- `BACKUP_INTERVAL_SECONDS` (ej. `43200` = cada 12h)

## ¿Puedo mandar backups a otra nube (S3, GDrive, etc.)?
Sí. Puedes agregar un paso extra (rclone / s3 sync) o usar otra estrategia de backup.

## ¿Puedo poner IP allowlist también a n8n?
Sí, con otro middleware igual al de Portainer y agregándolo a `n8n.middlewares`.

## ¿Qué pasa si quiero desactivar rate limiting?
Quita el middleware de la cadena de middlewares del router correspondiente.

## ¿Portainer necesita el puerto 8000?
Solo si usas Edge Agents. Si no, puedes quitar `8000` del `expose`.

---

# Checklist final (versión avanzada)

- [ ] `.env` completo con dominios, email, key y hash
- [ ] `N8N_ENCRYPTION_KEY` definida
- [ ] `PORTAINER_BASICAUTH_HASH` definido
- [ ] `PORTAINER_ALLOWED_IPS` correcta
- [ ] `letsencrypt/acme.json` creado con permisos `600`
- [ ] `docker-compose.yml` creado
- [ ] `docker compose up -d` ejecutado
- [ ] `https://n8n-devtallez.pixelia.cloud` responde
- [ ] `https://portainer.pixelia.cloud` responde (doble auth)
- [ ] `backups/` se llena con `.tar.gz`
- [ ] Dashboard Traefik accesible por SSH tunnel

---
