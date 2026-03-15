# Projecte DEVOPS

Vull muntar un laboratori de proves amb 2 servidors: un OKD per desplegar els PODS
de les aplicacions i un servidor DEVOPS per desplegar i fer proves amb eines.

## Infraestructura disponible

| Equip | CPU | RAM | SSD | Hostname | IP |
|-------|-----|-----|-----|----------|----|
| GEEKOM IT12 I7 | Intel i7 | 32 GB | 1 TB | IT12-OKD | 192.168.1.4 |
| GEEKOM IT12 I5 | Intel i5 | 32 GB | 1 TB | IT12-DEVOPS | 192.168.1.5 |
| NAS Synology DS220J | — | — | 2 TB | synology | 192.168.1.2 |
| GEEKOM IT12 I7 | Intel i7 | 32 GB | 1 TB | IT12-OKD | 192.168.2.4 |
| GEEKOM IT12 I5 | Intel i5 | 32 GB | 1 TB | IT12-DEVOPS | 192.168.2.5 |
| NAS Synology DS220J | — | — | 2 TB | synology | 192.168.2.2 |

## Entorn de desenvolupament

- MacBook Air M3 24 GB / 512 GB SSD
- Portàtil Intel i5 8 GB / 1 TB SSD (Windows 10)

## Software

- **IT12-OKD:** OKD Baremetal en Fedora.
- **IT12-DEVOPS:** Ubuntu Server, BTRFS, Docker Compose, Portainer, Gitea,
  Traefik, Jenkins, Grafana, Prometheus, Harbor, Keycloak.

---

## Configuració IT12-DEVOPS

### Arquitectura general

```
Internet/LAN
     │
     ▼
┌─────────────┐
│   TRAEFIK   │  <- Proxy invers (porta 80/443)
│  (router)   │     Redirigeix tràfic a cada servei
└──────┼──────┘
       │ xarxa: devops-net
       │
▼        ▼        ▼        ▼        ▼        ▼
Portainer  Gitea  Jenkins  Grafana  Keycloak  Harbor
```

### Per què cada eina?

**🔀 Traefik** — Proxy invers. Sense Traefik: `192.168.2.5:8080`... Amb Traefik: `jenkins.devops.lab`...

**🐳 Portainer** — Gestió visual de Docker via navegador.

**🐙 Gitea** — Repositori Git propi (com GitHub). Jenkins llegirà d'aquí per fer els builds.

**🔧 Jenkins** — CI/CD: automatitza build, test i deploy.

**📊 Grafana + Prometheus** — Prometheus recull mètriques; Grafana les mostra en dashboards.

**🔐 Keycloak** — Autenticació centralitzada (SSO).

**📦 Harbor** — Registry de contenidors privat. Jenkins construeix imatge → puja a Hcdarbor → OKD la descarrega.

## Flux de treball complet

```
Developer (MacBook/Portàtil)
     │
     │ git push
     ▼
┌─────────┐     webhook      ┌─────────┐
│  Gitea  │ ──────────────→  │ Jenkins │
│  (codi) │                  │ (CI/CD) │
└─────────┘                  └────┼────┘cd
                                  │ build & push
                                  ▼
                            ┌─────────┐
                            │ Harbor  │
                            └────┼────┘
                                 │ pull image
                                 ▼
                          ┌────────────┐
                          │  IT12-OKD  │
                          └────────────┘
```

## Per què BTRFS amb subvolums?

```
/dev/nvme0n1p2 (BTRFS 1TB)
├── @docker      → Dades Docker engine
├── @portainer   → Dades Portainer
├── @gitea       → Repositoris Git
├── @jenkins     → Jobs i configuració
├── @grafana     → Dashboards
├── @prometheus  → Mètriques
├── @harbor      → Imatges Docker
└── @keycloak    → Usuaris i config
```

**Avantatges:** Snapshots per servei independent, rollback si algo falla, quotes d'espai per servei, esborrat net sense afectar altres serveis.

### Per on continuar?

| Ordre | Servei | Motiu |
|-------|--------|-------|
| 1r | Traefik | Necessari per accedir als altres |
| 2n | Portainer | Per gestionar Docker visualment |
| 3r | Gitea | Per tenir el codi |
| 4t | Jenkins | Necessita Gitea |
| 5è | Prometheus | Recull dades |
| 6è | Grafana | Necessita Prometheus |
| 7è | Keycloak | SSO per a tots els serveis |
| 8è | Harbor | Registry d'imatges |

---

## Pas 1 - Configuració volum BTRFS

```bash
# Veure els discs i particions
lsblk -f

# Veure subvolums existents
sudo btrfs subvolume list /

# Crear subvolums per a cada servei Docker
sudo btrfs subvolume create /@docker
sudo btrfs subvolume create /@portainer
sudo btrfs subvolume create /@gitea
sudo btrfs subvolume create /@jenkins
sudo btrfs subvolume create /@grafana
sudo btrfs subvolume create /@prometheus
sudo btrfs subvolume create /@harbor
sudo btrfs subvolume create /@keycloak

# Crear els punts de muntatge
sudo mkdir -p /mnt/btrfs-data/{docker,portainer,gitea,jenkins,grafana,prometheus,harbor,keycloak}

# Muntar cada subvolum
sudo mount -o subvol=/@docker /dev/nvme0n1p2 /mnt/btrfs-data/docker
sudo mount -o subvol=/@portainer /dev/nvme0n1p2 /mnt/btrfs-data/portainer
sudo mount -o subvol=/@gitea /dev/nvme0n1p2 /mnt/btrfs-data/gitea
sudo mount -o subvol=/@jenkins /dev/nvme0n1p2 /mnt/btrfs-data/jenkins
sudo mount -o subvol=/@grafana /dev/nvme0n1p2 /mnt/btrfs-data/grafana
sudo mount -o subvol=/@prometheus /dev/nvme0n1p2 /mnt/btrfs-data/prometheus
sudo mount -o subvol=/@harbor /dev/nvme0n1p2 /mnt/btrfs-data/harbor
sudo mount -o subvol=/@keycloak /dev/nvme0n1p2 /mnt/btrfs-data/keycloak
```

---

## Pas 2 - Instal·lació Docker Engine

```bash
# Eliminar versions antigues
sudo apt remove -y docker docker-engine docker.io containerd runc
sudo apt update
sudo apt install -y ca-certificates curl gnupg lsb-release

# Afegir clau GPG oficial Docker
sudo install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | \
  sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
sudo chmod a+r /etc/apt/keyrings/docker.gpg

# Instal·lar Docker
sudo apt update
sudo apt install -y docker-ce docker-ce-cli containerd.io docker-compose-plugin
sudo usermod -aG docker $USER

docker --version
docker compose version
```

✅ Docker instal·lat correctament!

| Component | Versió | Estat |
|-----------|--------|-------|
| Docker Engine | 29.3.0 | ✅ |
| Docker Compose | v5.1.0 | ✅ |

### Verificar Storage Driver BTRFS

```bash
docker info | grep -i storage
# Ha de mostrar: Storage Driver: btrfs
```

> **Error: `Storage Driver: overlayfs`** — Docker no està usant BTRFS. Solucio:

```bash
sudo systemctl stop docker
sudo nano /etc/docker/daemon.json
```

```json
{
  "data-root": "/mnt/btrfs-data/docker",
  "storage-driver": "btrfs",
  "log-driver": "json-file",
  "log-opts": { "max-size": "10m", "max-file": "3" }
}
```

```bash
sudo systemctl start docker
docker info | grep -i storage
```

### ⚠️ Permisos Docker

```bash
# Op. 1: Tancar i reconnectar sessió (recomanat)
exit
# Op. 2: Aplicar el grup sense tancar sessió
newgrp docker
groups $USER
```

---

## Pas 3 - Estructura de directoris i xarxa

```bash
sudo mkdir -p /opt/devops/{traefik,portainer,gitea,jenkins,grafana,prometheus,harbor,keycloak}
sudo chown -R $USER:$USER /opt/devops
docker network create devops-net
docker network ls
```

**Fitxer `/opt/devops/.env`:**

```env
SERVER_IP=192.168.1.5
DOMAIN=devops.net
BTRFS_DATA=/mnt/btrfs-data
ADMIN_USER=admin
ADMIN_PASSWORD=B1ny2l3s@
```

---

## Pas 6 - Traefik (Proxy Invers)

**`/opt/devops/traefik/docker-compose.yml`:**

```yaml
services:
  traefik:
    image: traefik:v2.11
    container_name: traefik
    restart: always
    command:
      - "--api.dashboard=true"
      - "--api.insecure=true"
      - "--providers.docker=true"
      - "--providers.docker.exposedbydefault=false"
      - "--providers.docker.network=devops-net"
      - "--entrypoints.web.address=:80"
      - "--log.level=INFO"
    ports:
      - "80:80"
      - "8080:8080"
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock:ro
    networks:
      - devops-net
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.dashboard.rule=Host(`traefik.devops.net`)"
      - "traefik.http.routers.dashboard.service=api@internal"

networks:
  devops-net:
    external: true
```

```bash
cd /opt/devops/traefik
docker compose up -d
docker compose ps
curl http://localhost:8080/api/rawdata
```

---

## Pas 7 - Portainer

**`/opt/devops/portainer/docker-compose.yml`:**

```yaml
services:
  portainer:
    image: portainer/portainer-ce:latest
    container_name: portainer
    restart: always
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
      - /mnt/btrfs-data/portainer:/data
    networks:
      - devops-net
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.portainer.rule=Host(`portainer.devops.net`)"
      - "traefik.http.routers.portainer.entrypoints=web"
      - "traefik.http.services.portainer.loadbalancer.server.port=9000"

networks:
  devops-net:
    external: true
```

```bash
cd /opt/devops/portainer
docker compose up -d
docker compose ps
```

> **Problema timeout:** Si el timeout de 5 minuts ha caducat sense crear l'admin:

```bash
docker compose down
sudo rm -rf /mnt/btrfs-data/portainer/*
docker compose up -d
```

Accedir **immediatament** a `http://portainer.devops.net` i crear usuari `admin / B1ny2l3s@`.
Després: "Get Started" → "local".

---

## Pas 8 - Gitea

**`/opt/devops/gitea/docker-compose.yml`:**

```yaml
services:
  gitea:
    image: gitea/gitea:latest
    container_name: gitea
    restart: always
    environment:
      - USER_UID=1000
      - USER_GID=1000
      - GITEA__database__DB_TYPE=sqlite3
      - GITEA__server__DOMAIN=gitea.devops.net
      - GITEA__server__ROOT_URL=http://gitea.devops.net
      - GITEA__server__SSH_DOMAIN=192.168.1.5
      - GITEA__server__SSH_PORT=222
    volumes:
      - /mnt/btrfs-data/gitea:/data
    ports:
      - "222:22"
    networks:
      - devops-net
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.gitea.rule=Host(`gitea.devops.net`)"
      - "traefik.http.routers.gitea.entrypoints=web"
      - "traefik.http.services.gitea.loadbalancer.server.port=3000"

networks:
  devops-net:
    external: true
```

```bash
cd /opt/devops/gitea
docker compose up -d
docker compose ps
```

Accedir a `http://gitea.devops.net` i completar la configuració inicial:
Database=SQLite3, Admin=admin, Password=B1ny2l3s@, Email=efornaguera@gmail.com.

### 💬 Dubte: En els logs veiem `0.0.0.0:3000`

`0.0.0.0` = el servei escolta a **totes** les interfícies del contenidor.
Dins Docker és normal i segur; Traefik controla l'accés exterior.

### 💬 Dubte: IPs dels contenidors, puc fer ping?

```bash
docker network inspect devops-net
docker inspect -f '{{.Name}} - {{range .NetworkSettings.Networks}}{{.IPAddress}}{{end}}' $(docker ps -q)
```

Des d'IT12-DEVOPS: ping per IP o nom del contenidor. Des del MacBook: no és accessible directament.

### 💬 Dubte: Es pot canviar el nom del domini?

Sí! S'ha configurat el NAS Synology DS220J com a servidor DNS local amb zona `devops.net`.

```bash
# Canvi massiu de domini a tots els docker-compose
sed -i 's/192.168.1.5.nip.io/devops.net/g' /opt/devops/*/docker-compose.yml
grep -r 'devops.net' /opt/devops/*/docker-compose.yml

# Reiniciar tots els serveis
for svc in traefik portainer gitea jenkins grafana keycloak; do
  cd /opt/devops/$svc && docker compose down && docker compose up -d
done
```

---

## Pas 9 - Jenkins

**`/opt/devops/jenkins/docker-compose.yml`:**

```yaml
services:
  jenkins:
    image: jenkins/jenkins:lts
    container_name: jenkins
    restart: always
    user: root
    environment:
      - JAVA_OPTS=-Djenkins.install.runSetupWizard=true
    volumes:
      - /mnt/btrfs-data/jenkins:/var/jenkins_home
      - /var/run/docker.sock:/var/run/docker.sock
    networks:
      - devops-net
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.jenkins.rule=Host(`jenkins.devops.net`)"
      - "traefik.http.routers.jenkins.entrypoints=web"
      - "traefik.http.services.jenkins.loadbalancer.server.port=8080"

networks:
  devops-net:
    external: true
```

```bash
cd /opt/devops/jenkins
docker compose up -d
docker exec jenkins cat /var/jenkins_home/secrets/initialAdminPassword
```

Accedir a `http://jenkins.devops.net`, introduir la clau inicial, instal·lar plugins suggerits.

**Crear usuari admin:** Username=admin, Password=B1ny2l3s@, Email=admin@it12-devops.local

### Plugins recomanats per a pipelines Maven

| Plugin | Descripció |
|--------|-------------|
| Pipeline | Suport bàsic pipelines |
| Pipeline: Stage View | Visualització stages |
| Maven Integration | Compilar projectes Maven |
| Git | Clonar repositoris Git |
| Gitea | Webhook amb Gitea |
| Docker Pipeline | Construir imatges Docker |
| Blue Ocean | UI moderna per pipelines |
| JUnit + Jacoco | Tests i cobertura |

### Configurar Maven i JDK

**Manage Jenkins → Tools → Maven:** Name=Maven-3.9, Version=3.9.13, Install automatically=✅

**Manage Jenkins → Tools → JDK:** Name=JDK-21, Vendor=Adoptium, Install automatically=✅

### Exemple Jenkinsfile

```groovy
pipeline {
    agent any
    tools { maven 'Maven-3.9'; jdk 'JDK-21' }
    environment {
        GITEA_URL = 'http://gitea.devops.net'
        HARBOR_URL = 'harbor.devops.net:8888'
        APP_NAME = 'myapp'
    }
    stages {
        stage('Checkout') {
            steps { git branch: 'main', url: "${GITEA_URL}/admin/${APP_NAME}.git" }
        }
        stage('Build') {
            steps { sh 'mvn clean package -DskipTests' }
        }
        stage('Test') {
            steps { sh 'mvn test' }
            post { always { junit '**/target/surefire-reports/*.xml'; jacoco() } }
        }
        stage('Docker Build & Push') {
            steps {
                sh "docker build -t ${HARBOR_URL}/${APP_NAME}:${BUILD_NUMBER} ."
                withCredentials([usernamePassword(credentialsId: 'harbor-credentials',
                    usernameVariable: 'HARBOR_USER', passwordVariable: 'HARBOR_PASS')]) {
                    sh '''
                        docker login ${HARBOR_URL} -u ${HARBOR_USER} -p ${HARBOR_PASS}
                        docker push ${HARBOR_URL}/${APP_NAME}:${BUILD_NUMBER}
                        docker push ${HARBOR_URL}/${APP_NAME}:latest
                    '''
                }
            }
        }
    }
    post {
        success { echo '✅ Pipeline completat correctament!' }
        failure { echo '❌ Pipeline ha fallat!' }
    }
}
```

### Configurar Webhook Gitea → Jenkins

**Jenkins:** New Item → Multibranch Pipeline → Branch Sources → Gitea  
**Gitea:** Repositori → Settings → Webhooks → `http://jenkins.devops.net/gitea-webhook/post`

---

## Pas 10 - Grafana + Prometheus

**`/opt/devops/grafana/docker-compose.yml`:**

```yaml
services:
  prometheus:
    image: prom/prometheus:latest
    container_name: prometheus
    restart: always
    volumes:
      - ./prometheus.yml:/etc/prometheus/prometheus.yml
      - /mnt/btrfs-data/prometheus:/prometheus
    networks:
      - devops-net
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.prometheus.rule=Host(`prometheus.devops.net`)"
      - "traefik.http.routers.prometheus.entrypoints=web"
      - "traefik.http.services.prometheus.loadbalancer.server.port=9090"

  grafana:
    image: grafana/grafana:latest
    container_name: grafana
    restart: always
    environment:
      - GF_SECURITY_ADMIN_USER=admin
      - GF_SECURITY_ADMIN_PASSWORD=B1ny2l3s@
      - GF_USERS_ALLOW_SIGN_UP=false
    volumes:
      - /mnt/btrfs-data/grafana:/var/lib/grafana
    networks:
      - devops-net
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.grafana.rule=Host(`grafana.devops.net`)"
      - "traefik.http.routers.grafana.entrypoints=web"
      - "traefik.http.services.grafana.loadbalancer.server.port=3000"

networks:
  devops-net:
    external: true
```

**`/opt/devops/grafana/prometheus.yml`:**

```yaml
global:
  scrape_interval: 15s

scrape_configs:
  - job_name: "prometheus"
    static_configs:
      - targets: ["prometheus:9090"]
  - job_name: "docker"
    static_configs:
      - targets: ["192.168.1.5:9323"]
  - job_name: "node"
    static_configs:
      - targets: ["192.168.1.5:9100"]
    - job_name: "docker"
      static_configs:
        - targets: ["192.168.2.5:9323"]
    - job_name: "node"
      static_configs:
        - targets: ["192.168.2.5:9100"]
```

```bash
cd /opt/devops/grafana
docker compose up -d
docker compose ps
```

| Servei | URL | Credencials |
|--------|-----|-------------|
| Grafana | http://grafana.devops.net | admin / B1ny2l3s@ |
| Prometheus | http://prometheus.devops.net | — |

### ⚠️ Error: Grafana — permisos (usuari 472)

```bash
cd /opt/devops/grafana && docker compose down
sudo chown -R 472:472 /mnt/btrfs-data/grafana
docker compose up -d
```

### ⚠️ Error: Prometheus 404 (usuari 65534 = nobody)

```bash
cd /opt/devops/grafana && docker compose down
sudo chown -R 65534:65534 /mnt/btrfs-data/prometheus
docker compose up -d
```

---

## Pas 11 - Keycloak

**`/opt/devops/keycloak/docker-compose.yml`:**

```yaml
services:
  keycloak:
    image: quay.io/keycloak/keycloak:24.0.1
    container_name: keycloak
    restart: always
    environment:
      - KEYCLOAK_ADMIN=admin
      - KEYCLOAK_ADMIN_PASSWORD=B1ny2l3s@
      - KC_PROXY=edge
      - KC_HTTP_ENABLED=true
      - KC_HOSTNAME=keycloak.devops.net
      - KC_HOSTNAME_STRICT=false
    command: start-dev
    volumes:
      - /mnt/btrfs-data/keycloak:/opt/keycloak/data
    networks:
      - devops-net
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.keycloak.rule=Host(`keycloak.devops.net`)"
      - "traefik.http.routers.keycloak.entrypoints=web"
      - "traefik.http.services.keycloak.loadbalancer.server.port=8080"

networks:
  devops-net:
    external: true
```

```bash
cd /opt/devops/keycloak
docker compose up -d
docker compose logs -f  # triga 1-2 minuts
```

Accedir a `http://keycloak.devops.net` amb `admin / B1ny2l3s@`.

### ⚠️ Error: Keycloak — permisos H2 (usuari 1000)

```bash
cd /opt/devops/keycloak && docker compose down
sudo chown -R 1000:1000 /mnt/btrfs-data/keycloak
docker compose up -d
```

---

## Pas 12 - Harbor (Registry d'imatges)

> Harbor no té docker-compose oficial. S'instal·la amb el seu propi installer.

```bash
# Descarregar i descomprimir
cd /opt/devops/harbor
wget https://github.com/goharbor/harbor/releases/download/v2.10.0/harbor-online-installer-v2.10.0.tgz
tar xzvf harbor-online-installer-v2.10.0.tgz
cd harbor

# Configurar
cp harbor.yml.tmpl harbor.yml
nano harbor.yml
```

Configuració mínima al `harbor.yml`:

```yaml
hostname: harbor.devops.net
http:
  port: 8888
harbor_admin_password: B1ny2l3s@
data_volume: /mnt/btrfs-data/harbor
```

```bash
# Instal·lar
cd /opt/devops/harbor/harbor
sudo ./install.sh
docker compose ps
```

Accedir a `http://harbor.devops.net:8888` amb `admin / B1ny2l3s@`.

**Configurar Traefik per Harbor** — afegir labels al servei proxy de Harbor:

```yaml
labels:
  - "traefik.enable=true"
  - "traefik.http.routers.harbor.rule=Host(`harbor.devops.net`)"
  - "traefik.http.routers.harbor.entrypoints=web"
  - "traefik.http.services.harbor.loadbalancer.server.port=8888"
```

### ⚠️ Error: Harbor — falta docker-compose.yml

```bash
cat harbor.yml | grep hostname
sudo ./install.sh 2>&1 | tee install.log
```

---

## 🎉 Servidor IT12-DEVOPS completat!

### Resum de serveis actius

| Servei | URL | Credencials |
|--------|-----|-------------|
| Traefik | http://traefik.devops.net | — |
| Portainer | http://portainer.devops.net | admin / B1ny2l3s@ |
| Gitea | http://gitea.devops.net | admin / B1ny2l3s@ |
| Jenkins | http://jenkins.devops.net | admin / B1ny2l3s@ |
| Grafana | http://grafana.devops.net | admin / B1ny2l3s@ |
| Prometheus | http://prometheus.devops.net | — |
| Keycloak | http://keycloak.devops.net | admin / B1ny2l3s@ |
| Harbor | http://harbor.devops.net:8888 | admin / B1ny2l3s@ |

### Estat del projecte

| Pas | Tasca | Estat |
|-----|-------|-------|
| 1 | Ubuntu Server instal·lat | ✅ |
| 2 | BTRFS subvolums creats | ✅ |
| 3 | Docker instal·lat | ✅ |
| 4 | Estructura directoris | ✅ |
| 5 | Xarxa Docker | ✅ |
| 6 | Traefik | ✅ |
| 7 | Portainer | ✅ |
| 8 | Gitea | ✅ |
| 9 | Jenkins | ✅ |
| 10 | Grafana + Prometheus | ✅ |
| 11 | Keycloak | ✅ |
| 12 | Harbor | ✅ |

## Propers passos

| Projecte | Descripció |
|----------|-------------|
| IT12-OKD | Instal·lar OKD Baremetal a Fedora (192.168.1.4) |
| Pipeline | Primer pipeline Jenkins → Gitea → Harbor → OKD |
| Monitoring | Dashboards Grafana per monitoritzar tot el lab |
| SSO | Integrar Keycloak amb tots els serveis |

