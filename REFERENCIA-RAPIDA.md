# Referència ràpida — DevOps Lab

Comandes d'ús freqüent per al dia a dia del laboratori.

---

## Snapshots BTRFS

```bash
# Crear snapshot manual de tots els serveis
sudo /opt/devops/snapshots/snapshot.sh

# Restaurar un snapshot d'un servei
sudo /opt/devops/snapshots/restore-snapshot.sh

# Llistar snapshots disponibles
ls /mnt/btrfs-snapshots/

# Llistar snapshots d'un servei concret
ls /mnt/btrfs-snapshots/gitea/

# Veure logs de snapshots
tail -50 /var/log/snapshot-it12.log
```

---

## Docker — comandes globals

```bash
# Estat de tots els contenidors
docker ps -a

# Contenidors en execució amb ús de recursos
docker stats

# Veure logs d'un contenidor
docker logs -f <nom>

# Reiniciar tots els serveis del lab
for dir in /opt/devops/*/; do
  [ -f "$dir/docker-compose.yml" ] && docker compose -f "$dir/docker-compose.yml" restart
done

# Netejar imatges i contenidors no usats
docker system prune -f
```

---

## Serveis individuals

```bash
# Traefik
cd /opt/devops/traefik && docker compose [up -d | down | restart | logs -f]

# Gitea
cd /opt/devops/gitea && docker compose [up -d | down | restart | logs -f]

# Jenkins
cd /opt/devops/jenkins && docker compose [up -d | down | restart | logs -f]

# Portainer
cd /opt/devops/portainer && docker compose [up -d | down | restart | logs -f]

# Keycloak
cd /opt/devops/keycloak && docker compose [up -d | down | restart | logs -f]

# Harbor
cd /opt/devops/harbor && docker compose [up -d | down | restart | logs -f]

# Monitoring (Prometheus + Grafana)
cd /opt/devops/grafana && docker compose [up -d | down | restart | logs -f]
```

---

## URLs del laboratori

| Servei | URL |
|--------|-----|
| Traefik Dashboard | http://traefik.devops.lab |
| Gitea | http://gitea.devops.lab |
| Jenkins | http://jenkins.devops.lab |
| Portainer | http://portainer.devops.lab |
| Keycloak | http://keycloak.devops.lab |
| Harbor | http://harbor.devops.lab:8888 |
| Prometheus | http://prometheus.devops.lab |
| Grafana | http://grafana.devops.lab |

---

## Backup

```bash
# Backup manual complet al NAS
sudo /opt/devops/backup/backup.sh

# Backup manual dels snapshots al NAS
sudo /opt/devops/backup/backup_snapshots.sh

# Veure log de backups
tail -50 /var/log/backup-it12.log
```

---

## Xarxa i connectivitat

```bash
# Veure IP del servidor
ip addr show | grep "inet "

# Veure xarxa devops-net de Docker
docker network inspect devops-net

# Test DNS des del Mac
nslookup gitea.devops.lab
```

---

## Deploy — sincronitzar repo amb servidor

```bash
# Des del servidor, previsualitzar canvis
bash /opt/devops/deploy.sh --dry-run

# Aplicar canvis (sincronitza repo → filesystem)
bash /opt/devops/deploy.sh

# Flux habitual des del Mac:
# 1. Editar fitxers a it12-devops/ (VSCode)
# 2. Commit + push a GitHub
# Al servidor:
cd ~/it12-devops && git pull
bash /opt/devops/deploy.sh
```

---

## Git — flux de treball

```bash
# Documentació (devops-lab-doc):
git add fase-01-infraestructura/README.md
git commit -m "docs: documentar instal·lació Docker completada"
git push origin main

# Configuració servidor (it12-devops):
git add opt/devops/traefik/docker-compose.yml
git commit -m "feat: afegir middleware autenticació Traefik"
git push origin main
```

---

## SSH

```bash
# Connectar al servidor des del Mac
ssh edu@192.168.2.5 -p 2222

# Connectar via Tailscale
ssh edu@100.108.144.100 -p 2222
```
