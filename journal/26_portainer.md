# Portainer — Gestió visual de Docker

**Data:** 17 de març de 2026
**Servidor:** IT12-DEVOPS (192.168.2.5)

---

## 1. Introducció

**Portainer** és una plataforma de gestió de contenidors amb interfície web. Permet administrar entorns Docker (i Kubernetes) de forma visual, sense necessitat de fer servir la línia de comandes per a les operacions habituals.

### Funcionalitats principals

| Àrea | Funcions |
|------|----------|
| **Contenidors** | Llistar, iniciar, aturar, reiniciar, eliminar, inspeccionar logs |
| **Imatges** | Llistar, descarregar (`pull`), eliminar, inspeccionar capes |
| **Volums** | Crear, eliminar, navegar pel contingut |
| **Xarxes** | Crear, inspeccionar, connectar/desconnectar contenidors |
| **Stacks** | Desplegar i gestionar stacks Docker Compose directament des de la UI |
| **Registres** | Connectar amb registres privats (Harbor, Docker Hub, etc.) |
| **Estadístiques** | Monitorització bàsica de CPU, memòria i xarxa per contenidor |
| **Entorns** | Gestionar múltiples endpoints Docker des d'una sola instància |

### Edicions disponibles

| Edició | Descripció | Ús recomanat |
|--------|------------|--------------|
| **Community Edition (CE)** | Gratuïta, open source | Laboratori, ús personal |
| **Business Edition (BE)** | Llicència de pagament, funcions avançades | Equips, producció |

> Al laboratori fem servir la **Community Edition**.

### Port per defecte

| Port | Ús |
|------|----|
| `9000` | HTTP (UI web) |
| `9443` | HTTPS |
| `8000` | Túnel agent (Portainer Agent) |

---

## 2. Referències

| Recurs | URL |
|--------|-----|
| Lloc oficial | https://www.portainer.io |
| Documentació CE | https://docs.portainer.io |
| Docker Hub (imatge CE) | https://hub.docker.com/r/portainer/portainer-ce |
| GitHub | https://github.com/portainer/portainer |
| Instal·lació amb Docker Compose | https://docs.portainer.io/start/install-ce/server/docker/linux |
| Integració amb Traefik | https://docs.portainer.io/advanced/reverse-proxy/traefik |
| Documentació Stacks (Compose via UI) | https://docs.portainer.io/user/docker/stacks |

---

## 3. Configuració al laboratori DevOps

### Arquitectura

```
portainer.devops.lab
        │
    Traefik (port 80/443)
        │
    Portainer CE  (port 9000)
        │
    /var/run/docker.sock   ← accés directe al daemon Docker
        │
    portainer_data (volum Docker)
```

Portainer s'executa com un contenidor Docker i accedeix al daemon local a través del socket Unix `/var/run/docker.sock`.

---

### 3.1 Prerequisits

- [Fase 01](../fase-01-infraestructura/README.md) completada: Docker instal·lat i Traefik en marxa
- Xarxa `devops-net` existent:
  ```bash
  docker network ls | grep devops-net
  ```
- DNS `portainer.devops.lab` apuntant a `192.168.2.5` (configurat al Synology)

---

### 3.2 Snapshot previ

```bash
sudo /opt/devops/snapshots/snapshot.sh  # pre-portainer
```

---

### 3.3 Estructura de fitxers

```
/opt/devops/portainer/
├── docker-compose.yml
└── .env -> /opt/devops/.env   ← symlink creat per deploy.sh
```

---

### 3.4 `docker-compose.yml`

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

> **Nota de seguretat:** muntar `/var/run/docker.sock` dóna accés complet al daemon Docker. En producció, considerar l'ús de [Portainer Agent](https://docs.portainer.io/admin/environments/add/docker/agent) o restringir l'accés amb TLS.

---

### 3.5 Desplegar

```bash
cd /opt/devops/portainer
docker compose up -d
docker compose logs -f   # verificar que arrenca sense errors
```

Sortida esperada:
```
portainer  | Starting Portainer ...
portainer  | 2026/03/17 ... server: Starting Portainer server on :9000
```

---

### 3.6 Primera configuració (UI web)

> ⚠️ **Important:** Portainer requereix crear l'usuari admin en els **primers 5 minuts** d'execució. Passats 5 minuts sense accedir, el contenidor es bloqueja i cal reiniciar-lo.

1. Accedir a `http://portainer.devops.lab`
2. Crear usuari **admin** amb contrasenya segura
3. Seleccionar entorn: **Docker** (local, ja detecta el socket)
4. Clicar **Connect** → accés al dashboard

**Verificació ràpida:**
```bash
# Des del servidor
curl -s -o /dev/null -w "%{http_code}" http://portainer.devops.lab
# → 200
```

---

### 3.7 Si cal reiniciar per timeout del wizard

```bash
cd /opt/devops/portainer
docker compose down
docker volume rm portainer_portainer_data   # eliminar dades per reiniciar el wizard
docker compose up -d
```

---

### 3.8 Verificació final

```bash
docker ps --filter "name=portainer" --format "table {{.Names}}\t{{.Status}}\t{{.Ports}}"
```

| Contenidor | Estat | Port intern |
|------------|-------|-------------|
| portainer | Up | 9000/tcp |

---

### 3.9 Snapshot post-instal·lació

```bash
sudo /opt/devops/snapshots/snapshot.sh  # post-portainer
```

---

## 4. Ús habitual al laboratori

### Gestionar stacks des de la UI

1. Portainer → **Stacks** → **Add stack**
2. Enganxar el contingut del `docker-compose.yml`
3. Configurar variables d'entorn si cal
4. **Deploy the stack**

> Els stacks desplegats des de la UI es guarden a `portainer_data`. Si es prefereix gestionar per fitxers (recomanat al laboratori), fer servir sempre `docker compose` des del servidor i usar Portainer únicament per a inspecció.

### Accions ràpides habituals

| Acció | On trobar-ho |
|-------|-------------|
| Veure logs d'un contenidor | Containers → [nom] → Logs |
| Accedir a terminal del contenidor | Containers → [nom] → Console |
| Inspeccionar variables d'entorn | Containers → [nom] → Inspect |
| Veure ús de CPU/memòria | Containers → [nom] → Stats |
| Connectar registre privat Harbor | Registries → Add registry |

---

## 5. Notes i incidències

| Data | Nota |
|------|------|
| 2026-03-17 | Instal·lació inicial — Portainer CE desplegat i operatiu |

---

## Checklist

- [x] `docker-compose.yml` creat a `/opt/devops/portainer/`
- [x] Contenidor `portainer` en marxa
- [x] Accessible via `http://portainer.devops.lab`
- [x] Usuari admin creat en el wizard inicial
- [x] Entorn Docker local connectat
- [x] Snapshot post-instal·lació creat
