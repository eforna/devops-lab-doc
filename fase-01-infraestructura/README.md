# Fase 01 — Infraestructura: Docker + Traefik

> **Objetivo**: Docker instalado, red interna `lab-net` creada, Traefik operativo como reverse proxy con subdominios `*.devops.lab`.  
> **Prerrequisito**: [Fase 00](../fase-00-base/README.md) completada.

## Estado

⬜ Pendiente

---

## Snapshot antes de empezar

```bash
sudo snapper -c root create --description "pre-fase-01-docker-traefik" --cleanup-algorithm number
```

---

## 1.1 Instalar Docker Engine

```bash
# Eliminar versiones antiguas si las hay
sudo apt remove -y docker docker-engine docker.io containerd runc 2>/dev/null

# Dependencias
sudo apt update
sudo apt install -y ca-certificates curl gnupg lsb-release

# Clave GPG oficial de Docker
sudo install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | \
  sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
sudo chmod a+r /etc/apt/keyrings/docker.gpg

# Repositorio Docker
echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] \
  https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

# Instalar
sudo apt update
sudo apt install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin

# Verificar
docker --version
docker compose version
```

```bash
# Añadir usuario al grupo docker (evitar sudo constante)
sudo usermod -aG docker $USER
newgrp docker

# Test
docker run --rm hello-world
```

---

## 1.2 Configurar Docker para BTRFS

Docker funciona bien con BTRFS de serie. Verificar el storage driver:

```bash
docker info | grep "Storage Driver"
# Debe mostrar: overlay2 o btrfs
```

Configuración recomendada para logs y límites (`/etc/docker/daemon.json`):

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

## 1.3 Crear estructura de directorios del lab

```bash
sudo mkdir -p /opt/lab/{traefik,gitea,jenkins,harbor,keycloak,portainer,prometheus,grafana}
sudo mkdir -p /opt/lab/traefik/{certs,config}
sudo chown -R $USER:$USER /opt/lab
```

---

## 1.4 Crear red Docker compartida

```bash
docker network create lab-net
docker network ls | grep lab-net
```

> Todos los servicios del lab se conectarán a `lab-net`. Traefik la usará para descubrir contenedores.

---

## 1.5 Configurar DNS local en Synology

En el NAS Synology → DNS Server → Zonas → `devops.lab`:

Añadir registros tipo A o wildcard:

| Registro | Tipo | Valor |
|----------|------|-------|
| `devops.lab` | A | `<LAB_IP>` |
| `*.devops.lab` | A | `<LAB_IP>` |

> Con un wildcard `*` todos los subdominios nuevos se resuelven automáticamente sin tocar el DNS.

**Verificar desde Mac**:
```bash
nslookup gitea.devops.lab
ping traefik.devops.lab
```

---

## 1.6 Desplegar Traefik

### Estructura de ficheros

```
/opt/lab/traefik/
├── docker-compose.yml
├── traefik.yml          ← configuración estática
├── config/
│   └── dynamic.yml      ← configuración dinámica
└── certs/               ← certificados (si se usan)
```

### `traefik.yml` — configuración estática

```bash
cat > /opt/lab/traefik/traefik.yml <<'EOF'
api:
  dashboard: true
  insecure: true          # Dashboard accesible sin auth (lab only)

entryPoints:
  web:
    address: ":80"
  websecure:
    address: ":443"

providers:
  docker:
    exposedByDefault: false
    network: lab-net
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
cat > /opt/lab/traefik/docker-compose.yml <<'EOF'
services:
  traefik:
    image: traefik:v3.0
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
      - lab-net
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.traefik.rule=Host(`traefik.devops.lab`)"
      - "traefik.http.routers.traefik.entrypoints=web"
      - "traefik.http.services.traefik.loadbalancer.server.port=8080"

networks:
  lab-net:
    external: true
EOF
```

### Arrancar Traefik

```bash
cd /opt/lab/traefik
docker compose up -d
docker compose logs -f
```

**Verificar**: Abrir `http://traefik.devops.lab` en el Mac → debe mostrar el dashboard de Traefik.

---

## 1.7 Snapshot final — punto de control "post-infraestructura"

```bash
sudo snapper -c root create --description "post-fase-01-docker-traefik" --cleanup-algorithm number
sudo snapper -c root list
```

---

## Comandos útiles de mantenimiento

```bash
# Ver contenedores activos
docker ps

# Ver logs de Traefik en tiempo real
docker logs -f traefik

# Reiniciar Traefik tras cambios de config
docker compose -f /opt/lab/traefik/docker-compose.yml restart

# Ver rutas descubiertas por Traefik
curl http://localhost:8080/api/rawdata | jq .
```

---

## Notas y observaciones

<!-- Documenta aquí lo que encuentres durante la ejecución -->

| Fecha | Nota |
|-------|------|
| | |

---

## Checklist de fase completada

- [ ] Docker Engine instalado y funcionando
- [ ] Usuario añadido al grupo docker
- [ ] `daemon.json` configurado
- [ ] Estructura `/opt/lab/` creada
- [ ] Red `lab-net` creada
- [ ] DNS wildcard `*.devops.lab` configurado en Synology
- [ ] Traefik desplegado y dashboard accesible
- [ ] Snapshot "post-fase-01" creado

**Siguiente fase**: [Fase 02 — Servicios core](../fase-02-servicios-core/README.md)
