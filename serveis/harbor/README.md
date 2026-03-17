# Harbor — Container Registry

> **Objectiu**: Harbor operatiu com a registry privat d'imatges Docker, integrat amb Jenkins per a push/pull automàtic.
> **Prerequisit**: [Jenkins](../jenkins/README.md) configurat.

## Estat

✅ Completat

---

## Snapshot abans de començar

```bash
sudo /opt/devops/snapshots/snapshot.sh pre-harbor
```

---

## 4.1 Descarregar l'instal·lador de Harbor

Harbor requereix el seu propi instal·lador (no és una imatge Docker simple):

```bash
# Descarregar l'última versió
cd /opt/devops
HARBOR_VERSION=$(curl -s https://api.github.com/repos/goharbor/harbor/releases/latest | jq -r .tag_name)
echo "Versió: $HARBOR_VERSION"

curl -LO "https://github.com/goharbor/harbor/releases/download/${HARBOR_VERSION}/harbor-online-installer-${HARBOR_VERSION}.tgz"
tar xvf harbor-online-installer-*.tgz
cd harbor
```

---

## 4.2 Configurar harbor.yml

```bash
cp harbor.yml.tmpl harbor.yml
```

Editar `harbor.yml`:

```yaml
# Canviar aquestes línies:
hostname: harbor.devops.lab

# Comentar la secció https (usem HTTP al lab)
# https:
#   port: 443
#   ...

harbor_admin_password: changeme_lab   # ← canviar

# Directori de dades
data_volume: /mnt/btrfs-data/harbor
```

```bash
mkdir -p /mnt/btrfs-data/harbor
```

---

## 4.3 Instal·lar Harbor

```bash
cd /opt/devops/harbor
sudo ./install.sh

# Harbor aixeca els seus propis contenidors amb docker compose
docker compose ps
```

---

## 4.4 Exposar Harbor amb Traefik

Harbor usa el seu propi `docker-compose.yml`. Afegir labels de Traefik al servei `nginx` de Harbor:

```yaml
# Al servei nginx de Harbor:
labels:
  - "traefik.enable=true"
  - "traefik.http.routers.harbor.rule=Host(`harbor.devops.lab`)"
  - "traefik.http.routers.harbor.entrypoints=web"
  - "traefik.http.services.harbor.loadbalancer.server.port=8888"
networks:
  - devops-net
  - harbor_harbor   # xarxa interna de harbor
```

```bash
# Afegir al final del docker-compose.yml de Harbor:
# networks:
#   devops-net:
#     external: true

docker compose down && docker compose up -d
```

**Verificar**: `http://harbor.devops.lab:8888` → login amb `admin` / `changeme_lab`.

---

## 4.5 Configurar Docker per usar Harbor

Harbor sense HTTPS requereix marcar-lo com a "insecure registry":

```bash
sudo tee -a /etc/docker/daemon.json <<'EOF'
{
  "insecure-registries": ["harbor.devops.lab"]
}
EOF

sudo systemctl restart docker
```

**Provar login**:
```bash
docker login harbor.devops.lab
# Usuari: admin
# Contrasenya: changeme_lab
```

---

## 4.6 Crear projecte a Harbor

A la UI de Harbor:
```
Projects → New Project
→ Name: devops-lab
→ Access Level: Private
```

---

## 4.7 Push de la primera imatge de prova

```bash
docker pull alpine
docker tag alpine harbor.devops.lab/devops-lab/alpine:test
docker push harbor.devops.lab/devops-lab/alpine:test
```

---

## 4.8 Integrar Harbor a Jenkins

```
Manage Jenkins → Credentials → Add
→ Kind: Username with password
→ Username: admin
→ Password: <harbor_password>
→ ID: harbor-credentials
```

Exemple a Jenkinsfile:
```groovy
stage('Push to Harbor') {
    steps {
        withCredentials([usernamePassword(
            credentialsId: 'harbor-credentials',
            usernameVariable: 'HARBOR_USER',
            passwordVariable: 'HARBOR_PASS'
        )]) {
            sh '''
                docker login harbor.devops.lab -u $HARBOR_USER -p $HARBOR_PASS
                docker build -t harbor.devops.lab/devops-lab/myapp:${BUILD_NUMBER} .
                docker push harbor.devops.lab/devops-lab/myapp:${BUILD_NUMBER}
            '''
        }
    }
}
```

---

## 4.9 Snapshot final

```bash
sudo /opt/devops/snapshots/snapshot.sh post-harbor
```

---

## Notes i observacions

| Data | Nota |
|------|------|
| | |

---

## Checklist completat

- [x] Harbor descarregat i instal·lat
- [x] `harbor.yml` configurat amb domini i contrasenya
- [x] Harbor accessible via Traefik a `harbor.devops.lab`
- [x] Docker configurat amb insecure-registry
- [x] Login Docker a Harbor funcionant
- [x] Projecte `devops-lab` creat a Harbor
- [x] Primera imatge pujada correctament
- [x] Credencials Harbor a Jenkins
- [x] Snapshot "post-harbor" creat
