# Fase 02 — Servicios core: Gitea · Portainer · Keycloak

> **Objetivo**: Los tres servicios de gestión base del laboratorio operativos y accesibles vía Traefik.  
> **Prerrequisito**: [Fase 01](../fase-01-infraestructura/README.md) completada. Traefik corriendo.

## Estado

⬜ Pendiente

---

## Snapshot antes de empezar

```bash
sudo snapper -c root create --description "pre-fase-02-servicios-core" --cleanup-algorithm number
```

---

## 2.1 Gitea — Servidor Git

> `gitea.devops.lab` — Alojará todos los repositorios del laboratorio, incluyendo este mismo.

### Estructura

```
/opt/lab/gitea/
├── docker-compose.yml
└── data/               ← persistencia (git repos, config)
```

### `docker-compose.yml`

```bash
mkdir -p /opt/lab/gitea/data

cat > /opt/lab/gitea/docker-compose.yml <<'EOF'
services:
  gitea:
    image: gitea/gitea:latest
    container_name: gitea
    restart: unless-stopped
    environment:
      - USER_UID=1000
      - USER_GID=1000
      - GITEA__database__DB_TYPE=sqlite3
      - GITEA__server__DOMAIN=gitea.devops.lab
      - GITEA__server__ROOT_URL=http://gitea.devops.lab
      - GITEA__server__HTTP_PORT=3000
    volumes:
      - ./data:/data
    networks:
      - lab-net
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.gitea.rule=Host(`gitea.devops.lab`)"
      - "traefik.http.routers.gitea.entrypoints=web"
      - "traefik.http.services.gitea.loadbalancer.server.port=3000"

networks:
  lab-net:
    external: true
EOF
```

```bash
cd /opt/lab/gitea
docker compose up -d
docker compose logs -f
```

**Verificar**: `http://gitea.devops.lab` → wizard de configuración inicial.

> ℹ️ En la primera visita completar el wizard: crear usuario admin, organización y primer repositorio `devops-lab`.

---

## 2.2 Portainer — UI de Docker

> `portainer.devops.lab` — Gestión visual de contenedores, imágenes, volúmenes y redes.

```bash
mkdir -p /opt/lab/portainer

cat > /opt/lab/portainer/docker-compose.yml <<'EOF'
services:
  portainer:
    image: portainer/portainer-ce:latest
    container_name: portainer
    restart: unless-stopped
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
      - portainer_data:/data
    networks:
      - lab-net
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.portainer.rule=Host(`portainer.devops.lab`)"
      - "traefik.http.routers.portainer.entrypoints=web"
      - "traefik.http.services.portainer.loadbalancer.server.port=9000"

volumes:
  portainer_data:

networks:
  lab-net:
    external: true
EOF
```

```bash
cd /opt/lab/portainer
docker compose up -d
```

**Verificar**: `http://portainer.devops.lab` → crear usuario admin en los primeros 5 minutos.

---

## 2.3 Keycloak — SSO / Identity Provider

> `keycloak.devops.lab` — Autenticación centralizada. Más adelante integrará Gitea, Jenkins, Grafana.

```bash
mkdir -p /opt/lab/keycloak

cat > /opt/lab/keycloak/docker-compose.yml <<'EOF'
services:
  keycloak:
    image: quay.io/keycloak/keycloak:latest
    container_name: keycloak
    restart: unless-stopped
    command: start-dev
    environment:
      - KEYCLOAK_ADMIN=admin
      - KEYCLOAK_ADMIN_PASSWORD=changeme_lab   # ← cambiar
      - KC_HOSTNAME=keycloak.devops.lab
      - KC_HTTP_ENABLED=true
      - KC_PROXY=edge
    networks:
      - lab-net
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.keycloak.rule=Host(`keycloak.devops.lab`)"
      - "traefik.http.routers.keycloak.entrypoints=web"
      - "traefik.http.services.keycloak.loadbalancer.server.port=8080"

networks:
  lab-net:
    external: true
EOF
```

> ⚠️ `start-dev` es para laboratorio. En producción usar `start` con base de datos externa (PostgreSQL).

```bash
cd /opt/lab/keycloak
docker compose up -d
docker compose logs -f keycloak
# Esperar mensaje: "Keycloak ... started in"
```

**Verificar**: `http://keycloak.devops.lab` → login con `admin` / `changeme_lab`.

---

## 2.4 Publicar el repositorio en Gitea

Una vez Gitea está funcionando, subir este mismo repositorio para editarlo desde VS Code / Fork:

```bash
# En el Mac — clonar desde el servidor o iniciar repo local
cd ~/Projects
git init devops-lab
cd devops-lab

# Copiar ficheros markdown generados (o crearlos aquí)
git add .
git commit -m "feat: estructura inicial del laboratorio devops"

# Añadir remote Gitea
git remote add origin http://gitea.devops.lab/<TU_USUARIO>/devops-lab.git
git push -u origin main
```

---

## 2.5 Snapshot final — punto de control "post-servicios-core"

```bash
sudo snapper -c root create --description "post-fase-02-servicios-core" --cleanup-algorithm number
sudo snapper -c root list
```

---

## Resumen de servicios activos tras esta fase

| Servicio | URL | Usuario por defecto |
|----------|-----|---------------------|
| Gitea | http://gitea.devops.lab | Creado en wizard |
| Portainer | http://portainer.devops.lab | Creado en primer acceso |
| Keycloak | http://keycloak.devops.lab | admin / changeme_lab |

---

## Notas y observaciones

| Fecha | Nota |
|-------|------|
| | |

---

## Checklist de fase completada

- [ ] Gitea desplegado y wizard completado
- [ ] Repositorio `devops-lab` creado en Gitea
- [ ] Portainer desplegado y usuario admin creado
- [ ] Keycloak desplegado y accesible
- [ ] Repositorio local vinculado a Gitea (Mac)
- [ ] VS Code / Fork configurado con el repo
- [ ] Snapshot "post-fase-02" creado

**Siguiente fase**: [Fase 03 — CI/CD con Jenkins](../fase-03-ci-cd/README.md)
