# Còpia de Seguretat IT12-DEVOPS → NAS Synology

## Arquitectura

```
IT12-DEVOPS (192.168.1.5)
        │
        │ NFS mount
        ▼
NAS Synology DS220J (192.168.1.2)
        │
        └── /volume1/backup/it12-devops/
            ├── docker-configs/
            ├── btrfs-data/
            ├── btrfs-snapshots/
            ├── system/
            └── home/
```

---

## Pas 1 — Preparar el NAS Synology (DSM)

```
DSM → Control Panel → File Services
  → NFS → Activar NFS + NFSv4
  → SMB/CIFS → Activar SMB

DSM → Control Panel → Shared Folders
  → Carpeta: "backup"
  → NFS Permissions → Create:
      Hostname/IP:  192.168.1.5
      Privilege:    Read/Write
      Squash:       No mapping
      Security:     sys
      ✅ Enable asynchronous
      ✅ Allow connections from non-privileged ports
      ✅ Allow users to access mounted subfolders
```

---

## Pas 2 — Instal·lar client NFS a Ubuntu

```bash
sudo apt install -y nfs-common cifs-utils

# Crear punt de muntatge
sudo mkdir -p /mnt/nas-backup

# Verificar que el NAS exporta la carpeta
showmount -e 192.168.1.2
```

---

## Pas 3 — Muntar el NAS (manual)

```bash
# Provar muntatge manual
sudo mount -t nfs 192.168.1.2:/volume1/backup /mnt/nas-backup

# Verificar
df -h | grep nas
ls -la /mnt/nas-backup

# Crear directori del servidor
mkdir -p /mnt/nas-backup/it12-devops
```

---

## Pas 4 — Muntatge automàtic (fstab)

```bash
sudo nano /etc/fstab
```

Afegir al final:

```
# NAS Synology - Backup
192.168.1.2:/volume1/backup  /mnt/nas-backup  nfs  defaults,_netdev,auto  0  0
```

```bash
# Aplicar sense reiniciar
sudo systemctl daemon-reload
sudo mount -a

# Verificar
df -h | grep nas
```

---

## Pas 5 — Script de backup

El script definitiu es troba a `/opt/devops/backup.sh`.
Veure document [10_optimitzacio backups] per la versió final actualitzada.

```bash
sudo tee /opt/devops/backup.sh > /dev/null << 'EOF'
#!/bin/bash

DATE=$(date +%Y%m%d_%H%M%S)
BACKUP_DIR="/mnt/nas-backup/it12-devops/${DATE}"
LOG_FILE="/var/log/backup-it12.log"

echo "========================================" | tee -a $LOG_FILE
echo "Backup iniciat: ${DATE}" | tee -a $LOG_FILE
echo "========================================" | tee -a $LOG_FILE

mkdir -p ${BACKUP_DIR}/{docker-configs,btrfs-data,system,home}

echo "Backup /opt/devops..." | tee -a $LOG_FILE
tar -czf ${BACKUP_DIR}/docker-configs/opt-devops-${DATE}.tar.gz -C / opt/devops/

echo "Backup /mnt/btrfs-data..." | tee -a $LOG_FILE
for dir in portainer gitea grafana prometheus keycloak harbor; do
    tar -czf ${BACKUP_DIR}/btrfs-data/${dir}-${DATE}.tar.gz -C /mnt/btrfs-data ${dir}/
done

echo "  Backup jenkins (sense war i plugins)..." | tee -a $LOG_FILE
tar -czf ${BACKUP_DIR}/btrfs-data/jenkins-${DATE}.tar.gz \
    --exclude=jenkins/war --exclude=jenkins/plugins \
    -C /mnt/btrfs-data jenkins/

echo "Backup /etc..." | tee -a $LOG_FILE
tar -czf ${BACKUP_DIR}/system/etc-${DATE}.tar.gz -C / etc/

echo "Backup /home/edu..." | tee -a $LOG_FILE
tar -czf ${BACKUP_DIR}/home/home-${DATE}.tar.gz -C /home edu/

# Documentar imatges Docker en us
docker images --format "{{.Repository}}:{{.Tag}}" > ${BACKUP_DIR}/docker-images-list.txt

echo "Netejant backups antics (>7 dies)..." | tee -a $LOG_FILE
find /mnt/nas-backup/it12-devops -maxdepth 1 -type d -mtime +7 \
    ! -name "*_snapshots" -exec rm -rf {} \;

echo "Backup completat: ${DATE}" | tee -a $LOG_FILE
echo "Espai total: $(du -sh ${BACKUP_DIR})" | tee -a $LOG_FILE
echo "========================================" | tee -a $LOG_FILE
EOF

sudo chmod +x /opt/devops/backup.sh
```

---

## Pas 6 — Provar el backup

```bash
# Executar
sudo /opt/devops/backup.sh

# Seguir el progrés en temps real
tail -f /var/log/backup-it12.log

# Verificar resultat
ls -la /mnt/nas-backup/it12-devops/
```

> ✅ El missatge `tar: Removing leading '/'` és **normal** — no és un error.
> Tar elimina el `/` inicial dels paths per seguretat.

---

## Pas 7 — Automatitzar amb cron

```bash
sudo crontab -e
```

```
# Snapshot BTRFS diari a les 01:00
0 1 * * * /opt/devops/snapshot.sh

# Backup complet setmanal (diumenge) a les 02:00
0 2 * * 0 /opt/devops/backup.sh

# Backup snapshots diari a les 03:00
0 3 * * * /opt/devops/backup_snapshots.sh
```

---

## Resum de l'estat del backup (08/03/2026)

| Directori | Mida | Estat |
|-----------|------|-------|
| `/opt/devops` | — | ✅ |
| `/mnt/btrfs-data/portainer` | — | ✅ |
| `/mnt/btrfs-data/gitea` | — | ✅ |
| `/mnt/btrfs-data/jenkins` | — | ✅ |
| `/mnt/btrfs-data/grafana` | — | ✅ |
| `/mnt/btrfs-data/prometheus` | — | ✅ |
| `/mnt/btrfs-data/keycloak` | — | ✅ |
| `/mnt/btrfs-data/harbor` | — | ✅ |
| `/etc` | — | ✅ |
| `/home/edu` | — | ✅ |
| **Total** | **~297 MB** | ✅ |

| Pas | Tasca | Estat |
|-----|-------|-------|
| 1 | NFS configurat al Synology | ✅ |
| 2 | NAS muntat a `/mnt/nas-backup` | ✅ |
| 3 | `fstab` configurat | ✅ |
| 4 | Script `/opt/devops/backup.sh` | ✅ |
| 5 | Cron configurat | ✅ |
