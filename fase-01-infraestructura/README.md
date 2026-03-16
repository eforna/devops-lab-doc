# Fase 01 — Infraestructura: Docker + Traefik

> **Objectiu**: Docker instal·lat, xarxa interna `devops-net` creada, Traefik operatiu com a reverse proxy amb subdominis `*.devops.lab`.
> **Prerequisit**: [Fase 00](../fase-00-base/README.md) completada.

## Estat

✅ Completat

---

## Snapshot abans de començar

```bash
sudo /opt/devops/snapshots/snapshot.sh  # snapshot pre-fase-01
```

---

## 1.1 Instal·lar Docker Engine

```bash
# Eliminar versions antigues si n'hi ha
sudo apt remove -y docker docker-engine docker.io containerd runc 2>/dev/null

# Dependències
sudo apt update
sudo apt install -y ca-certificates curl gnupg lsb-release

# Clau GPG oficial de Docker
sudo install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | \
  sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
sudo chmod a+r /etc/apt/keyrings/docker.gpg

# Repositori Docker
echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] \
  https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

# Instal·lar
sudo apt update
sudo apt install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin

# Verificar
docker --version
docker compose version
```

```bash
# Afegir usuari al grup docker (evitar sudo constant)
sudo usermod -aG docker $USER
newgrp docker

# Test
docker run --rm hello-world
```

---

## 1.2 Configurar Docker per a BTRFS

Docker funciona bé amb BTRFS de sèrie. Verificar el storage driver:

```bash
docker info | grep "Storage Driver"
# Ha de mostrar: overlay2 o btrfs
```

Configuració recomanada per a logs i límits (`/etc/docker/daemon.json`):

```bash
sudo tee /etc/docker/daemon.json <<EOF
{
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "10m",
    "max-file": "3"
  },
  "storage-driver": "overlay2"
}
EOF

sudo systemctl restart docker
docker info | grep -A5 "Storage Driver"
```

---

## 1.3 Crear estructura de directoris del lab

```bash
sudo mkdir -p /opt/devops/{traefik,gitea,jenkins,harbor,keycloak,portainer,grafana}
sudo mkdir -p /opt/devops/traefik/{certs,config}
sudo chown -R $USER:$USER /opt/devops
```

---

## 1.4 Crear xarxa Docker compartida

```bash
docker network create devops-net
docker network ls | grep devops-net
```

> Tots els serveis del lab es connectaran a `devops-net`. Traefik la farà servir per descobrir contenidors.

---

## 1.5 Configurar DNS local al Synology

Al NAS Synology → DNS Server → Zones → `devops.lab`:

Afegir registres tipus A o wildcard:

| Registre | Tipus | Valor |
|----------|-------|-------|
| `devops.lab` | A | `<LAB_IP>` |
| `*.devops.lab` | A | `<LAB_IP>` |

> Amb un wildcard `*` tots els subdominis nous es resolen automàticament sense tocar el DNS.

**Verificar des del Mac**:
```bash
nslookup gitea.devops.lab
ping traefik.devops.lab
```

---

## 1.6 Desplegar Traefik

### Estructura de fitxers

```
/opt/devops/traefik/
├── docker-compose.yml
├── traefik.yml          ← configuració estàtica
├── config/
│   └── dynamic.yml      ← configuració dinàmica
└── certs/               ← certificats (si s'utilitzen)
```

### `traefik.yml` — configuració estàtica

```bash
cat > /opt/devops/traefik/traefik.yml <<'EOF'
api:
  dashboard: true
  insecure: true          # Dashboard accessible sense auth (lab only)

entryPoints:
  web:
    address: ":80"
  websecure:
    address: ":443"

providers:
  docker:
    exposedByDefault: false
    network: devops-net
  file:
    filename: /config/dynamic.yml
    watch: true

log:
  level: INFO

accessLog: {}
EOF
```

### `docker-compose.yml` de Traefik

```bash
cat > /opt/devops/traefik/docker-compose.yml <<'EOF'
services:
  traefik:
    image: traefik:v2.11
    container_name: traefik
    restart: unless-stopped
    ports:
      - "80:80"
      - "443:443"
      - "8080:8080"       # Dashboard Traefik
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock:ro
      - ./traefik.yml:/traefik.yml:ro
      - ./config:/config:ro
      - ./certs:/certs
    networks:
      - devops-net
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.traefik.rule=Host(`traefik.devops.lab`)"
      - "traefik.http.routers.traefik.entrypoints=web"
      - "traefik.http.services.traefik.loadbalancer.server.port=8080"

networks:
  devops-net:
    external: true
EOF
```

### Arrancar Traefik

```bash
cd /opt/devops/traefik
docker compose up -d
docker compose logs -f
```

**Verificar**: Obrir `http://traefik.devops.lab` al Mac → ha de mostrar el dashboard de Traefik.

---

## 1.7 Snapshot final — punt de control "post-infraestructura"

```bash
sudo /opt/devops/snapshots/snapshot.sh  # snapshot post-fase-01-docker-traefik
```

---

## Comandes útils de manteniment

```bash
# Veure contenidors actius
docker ps

# Veure logs de Traefik en temps real
docker logs -f traefik

# Reiniciar Traefik després de canvis de config
docker compose -f /opt/devops/traefik/docker-compose.yml restart

# Veure rutes descobertes per Traefik
curl http://localhost:8080/api/rawdata | jq .
```

---

## Notes i observacions

<!-- Documenta aquí el que trobis durant l'execució -->

| Data | Nota |
|------|------|
| | |

---

## Checklist de fase completada

- [x] Docker Engine instal·lat i funcionant
- [x] Usuari afegit al grup docker
- [x] `daemon.json` configurat
- [x] Estructura `/opt/devops/` creada
- [x] Xarxa `devops-net` creada
- [x] DNS wildcard `*.devops.lab` configurat al Synology
- [x] Traefik desplegat i dashboard accessible
- [x] Snapshot "post-fase-01" creat

**Fase següent**: [Fase 02 — Serveis core](../fase-02-servicios-core/README.md)
