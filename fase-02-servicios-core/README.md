# Fase 02 — Serveis core: Gitea · Portainer · Keycloak

> **Objectiu**: Els tres serveis de gestió base del laboratori operatius i accessibles via Traefik.
> **Prerequisit**: [Fase 01](../fase-01-infraestructura/README.md) completada. Traefik en marxa.

## Estat

✅ Completat

---

## Snapshot abans de començar

```bash
sudo /opt/devops/snapshots/snapshot.sh  # snapshot pre-fase-02-serveis-core
```

---

## 2.1 Gitea — Servidor Git

> `gitea.devops.lab` — Allotjarà tots els repositoris del laboratori, inclòs aquest mateix.

### Estructura

```
/opt/devops/gitea/
├── docker-compose.yml
└── data/               ← persistència (git repos, config)
```

### `docker-compose.yml`

```bash
mkdir -p /opt/devops/gitea

cat > /opt/devops/gitea/docker-compose.yml <<'EOF'
services:
  gitea:
    image: gitea/gitea:latest
    container_name: gitea
    restart: unless-stopped
    environment:
      - USER_UID=1000
      - USER_GID=1000
      - GITEA__database__DB_TYPE=postgres
      - GITEA__database__HOST=db-gitea:5432
      - GITEA__database__NAME=gitea
      - GITEA__database__USER=gitea
      - GITEA__database__PASSWD=${GITEA_DB_PASSWORD}
      - GITEA__server__DOMAIN=gitea.devops.lab
      - GITEA__server__ROOT_URL=http://gitea.devops.lab
      - GITEA__server__HTTP_PORT=3000
    volumes:
      - /mnt/btrfs-data/gitea:/data
    networks:
      - devops-net
    depends_on:
      - db-gitea
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.gitea.rule=Host(`gitea.devops.lab`)"
      - "traefik.http.routers.gitea.entrypoints=web"
      - "traefik.http.services.gitea.loadbalancer.server.port=3000"

  db-gitea:
    image: postgres:15
    container_name: db-gitea
    restart: unless-stopped
    environment:
      - POSTGRES_USER=gitea
      - POSTGRES_PASSWORD=${GITEA_DB_PASSWORD}
      - POSTGRES_DB=gitea
    volumes:
      - /mnt/btrfs-data/gitea-db:/var/lib/postgresql/data
    networks:
      - devops-net

networks:
  devops-net:
    external: true
EOF
```

```bash
cd /opt/devops/gitea
docker compose up -d
docker compose logs -f
```

**Verificar**: `http://gitea.devops.lab` → wizard de configuració inicial.

> ℹ️ En la primera visita completar el wizard: crear usuari admin, organització i primer repositori `devops-lab`.

---

## 2.2 Portainer — UI de Docker

> `portainer.devops.lab` — Gestió visual de contenidors, imatges, volums i xarxes.

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

## 2.3 Keycloak — SSO / Identity Provider

> `keycloak.devops.lab` — Autenticació centralitzada. Més endavant integrarà Gitea, Jenkins, Grafana.

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

## 2.4 Publicar el repositori a Gitea

Un cop Gitea funciona, pujar aquest mateix repositori per editar-lo des de VS Code / Fork:

```bash
# Al Mac — clonar des del servidor o iniciar repo local
cd ~/Projects
git init devops-lab
cd devops-lab

# Copiar fitxers markdown generats (o crear-los aquí)
git add .
git commit -m "feat: estructura inicial del laboratori devops"

# Afegir remote Gitea
git remote add origin http://gitea.devops.lab/<EL_TEU_USUARI>/devops-lab.git
git push -u origin main
```

---

## 2.5 Snapshot final — punt de control "post-serveis-core"

```bash
sudo /opt/devops/snapshots/snapshot.sh  # snapshot post-fase-02-serveis-core
```

---

## Resum de serveis actius després d'aquesta fase

| Servei | URL | Usuari per defecte |
|--------|-----|--------------------|
| Gitea | http://gitea.devops.lab | Creat al wizard |
| Portainer | http://portainer.devops.lab | Creat en el primer accés |
| Keycloak | http://keycloak.devops.lab | admin / changeme_lab |

---

## Notes i observacions

| Data | Nota |
|------|------|
| | |

---

## Checklist de fase completada

- [x] Gitea desplegat i wizard completat
- [x] Repositori `devops-lab` creat a Gitea
- [x] Portainer desplegat i usuari admin creat
- [x] Keycloak desplegat i accessible
- [x] Repositori local vinculat a Gitea (Mac)
- [x] VS Code / Fork configurat amb el repo
- [x] Snapshot "post-fase-02" creat

**Fase següent**: [Fase 03 — CI/CD amb Jenkins](../fase-03-ci-cd/README.md)
