# Portainer — UI de Docker

> `portainer.devops.lab` — Gestió visual de contenidors, imatges, volums i xarxes.
> **Prerequisit**: [Docker + Traefik](../../infraestructura/docker-traefik/README.md) operatiu.

## Estat

✅ Completat

---

## Desplegar Portainer

```bash
mkdir -p /opt/devops/portainer

cat > /opt/devops/portainer/docker-compose.yml <<'EOF'
services:
  portainer:
    image: portainer/portainer-ce:latest
    container_name: portainer
    restart: unless-stopped
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
      - portainer_data:/data
    networks:
      - devops-net
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.portainer.rule=Host(`portainer.devops.lab`)"
      - "traefik.http.routers.portainer.entrypoints=web"
      - "traefik.http.services.portainer.loadbalancer.server.port=9000"

volumes:
  portainer_data:

networks:
  devops-net:
    external: true
EOF
```

```bash
cd /opt/devops/portainer
docker compose up -d
```

**Verificar**: `http://portainer.devops.lab` → crear usuari admin en els primers 5 minuts.

---

## Notes i observacions

| Data | Nota |
|------|------|
| | |

---

## Checklist completat

- [x] Portainer desplegat i usuari admin creat
