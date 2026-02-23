# Guía completa: n8n + Portainer detrás de Traefik (HTTPS) en Docker (VPS Ubuntu)

## Objetivo

Dejar **n8n** y **Portainer** funcionando de forma segura en un VPS, con:

- **HTTPS** (certificados Let’s Encrypt)
- **Traefik** como reverse proxy
- **n8n y Portainer sin puertos públicos directos** (solo internos en Docker)
- **Dashboard de Traefik solo interno** (acceso por túnel SSH)
- Persistencia con volúmenes Docker (`n8n_data`, `portainer_data`)
- Configuración organizada con `docker-compose.yml` + `.env`

---

# Arquitectura final

## Flujo de acceso

- `https://n8n-devtallez.pixelia.cloud` → **Traefik** → **n8n** (interno en Docker)
- `https://portainer.pixelia.cloud` → **Traefik** → **Portainer** (interno en Docker)
- `http://localhost:8080/dashboard/` (vía túnel SSH) → **Dashboard de Traefik** (solo interno)

## Qué puertos quedan expuestos realmente

### Públicos (VPS)
- **80** (HTTP) → para redirección y challenge de Let’s Encrypt
- **443** (HTTPS)

### Solo internos / no públicos
- **n8n 5678** (solo interno, vía `expose`)
- **Portainer 9443** (solo interno, vía `expose`)
- **Traefik dashboard 8080** (solo loopback `127.0.0.1`)

---

# Requisitos previos

## 1) VPS con Ubuntu (o Debian similar)
Debes tener acceso SSH al servidor.

## 2) Docker + Docker Compose Plugin instalados
Verifica:
```bash
docker version
docker compose version

