# Gitea — Servidor Git

> `gitea.devops.lab` — Allotja tots els repositoris del laboratori.
> **Prerequisit**: [Docker + Traefik](../../infraestructura/docker-traefik/README.md) operatiu.

## Estat

✅ Completat

---

## Snapshot abans de començar

```bash
sudo /opt/devops/snapshots/snapshot.sh  # snapshot pre-gitea
```

---

## Desplegar Gitea

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

## Publicar el repositori a Gitea

Un cop Gitea funciona, pujar aquest repositori de documentació:

```bash
# Al Mac
cd ~/Projects/devops-lab-doc
git remote add origin http://gitea.devops.lab/<USUARI>/devops-lab-doc.git
git push -u origin main
```

---

## Notes i observacions

| Data | Nota |
|------|------|
| | |

---

## Checklist completat

- [x] Gitea desplegat i wizard completat
- [x] Repositori `devops-lab-doc` creat a Gitea
- [x] Repositori local vinculat a Gitea (Mac)
- [x] VS Code / Fork configurat amb el repo

**Vegeu també**: [Jenkins](../jenkins/README.md) (integració webhooks)
