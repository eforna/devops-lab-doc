# Keycloak — SSO / Identity Provider

> `keycloak.devops.lab` — Autenticació centralitzada. Integra Gitea, Jenkins, Grafana.
> **Prerequisit**: [Docker + Traefik](../../infraestructura/docker-traefik/README.md) operatiu.

## Estat

✅ Completat

---

## Desplegar Keycloak

```bash
mkdir -p /opt/devops/keycloak

cat > /opt/devops/keycloak/docker-compose.yml <<'EOF'
services:
  keycloak:
    image: quay.io/keycloak/keycloak:latest
    container_name: keycloak
    restart: unless-stopped
    command: start-dev
    environment:
      - KEYCLOAK_ADMIN=admin
      - KEYCLOAK_ADMIN_PASSWORD=changeme_lab   # ← canviar
      - KC_HOSTNAME=keycloak.devops.lab
      - KC_HTTP_ENABLED=true
      - KC_PROXY=edge
    networks:
      - devops-net
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.keycloak.rule=Host(`keycloak.devops.lab`)"
      - "traefik.http.routers.keycloak.entrypoints=web"
      - "traefik.http.services.keycloak.loadbalancer.server.port=8080"

networks:
  devops-net:
    external: true
EOF
```

> ⚠️ `start-dev` és per a laboratori. En producció fer servir `start` amb base de dades externa (PostgreSQL).

```bash
cd /opt/devops/keycloak
docker compose up -d
docker compose logs -f keycloak
# Esperar missatge: "Keycloak ... started in"
```

**Verificar**: `http://keycloak.devops.lab` → login amb `admin` / `changeme_lab`.

---

## Notes i observacions

| Data | Nota |
|------|------|
| | |

---

## Checklist completat

- [x] Keycloak desplegat i accessible
- [ ] Contrasenya per defecte canviada (veure [Seguretat](../../seguretat/README.md))
