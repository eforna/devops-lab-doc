# Optimització Backups

## Script 1 — backup.sh

```bash
cd /opt/devops
sudo rm /opt/devops/backup.sh

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
    echo "  Backup ${dir}..." | tee -a $LOG_FILE
    tar -czf ${BACKUP_DIR}/btrfs-data/${dir}-${DATE}.tar.gz -C /mnt/btrfs-data ${dir}/
    echo "  OK ${dir}" | tee -a $LOG_FILE
done

echo "  Backup jenkins (sense war i plugins)..." | tee -a $LOG_FILE
tar -czf ${BACKUP_DIR}/btrfs-data/jenkins-${DATE}.tar.gz \
    --exclude=jenkins/war \
    --exclude=jenkins/plugins \
    -C /mnt/btrfs-data jenkins/
echo "  OK jenkins" | tee -a $LOG_FILE

echo "Backup /etc..." | tee -a $LOG_FILE
tar -czf ${BACKUP_DIR}/system/etc-${DATE}.tar.gz -C / etc/

echo "Backup /home/edu..." | tee -a $LOG_FILE
tar -czf ${BACKUP_DIR}/home/home-${DATE}.tar.gz -C /home edu/

echo "Netejant backups antics..." | tee -a $LOG_FILE
find /mnt/nas-backup/it12-devops -maxdepth 1 -type d -mtime +7 \
    ! -name "*_snapshots" -exec rm -rf {} \;

echo "Backup completat: ${DATE}" | tee -a $LOG_FILE
echo "Espai total: $(du -sh ${BACKUP_DIR})" | tee -a $LOG_FILE
echo "========================================" | tee -a $LOG_FILE
EOF

sudo chmod +x /opt/devops/backup.sh
```

## Script 2 — backup_snapshots.sh

```bash
sudo tee /opt/devops/backup_snapshots.sh > /dev/null << 'EOF'
#!/bin/bash

DATE=$(date +%Y%m%d_%H%M%S)
BACKUP_DIR="/mnt/nas-backup/it12-devops/${DATE}_snapshots"
LOG_FILE="/var/log/backup-it12.log"

echo "========================================" | tee -a $LOG_FILE
echo "Backup snapshots iniciat: ${DATE}" | tee -a $LOG_FILE
echo "========================================" | tee -a $LOG_FILE

mkdir -p ${BACKUP_DIR}/btrfs-snapshots

echo "Backup /mnt/btrfs-snapshots..." | tee -a $LOG_FILE
for dir in portainer gitea jenkins grafana prometheus keycloak harbor; do
    echo "  Backup snapshots ${dir}..." | tee -a $LOG_FILE
    tar -czf ${BACKUP_DIR}/btrfs-snapshots/${dir}-snapshots-${DATE}.tar.gz \
        -C /mnt/btrfs-snapshots ${dir}/
    echo "  OK snapshots ${dir}" | tee -a $LOG_FILE
done

echo "Netejant backups snapshots antics..." | tee -a $LOG_FILE
find /mnt/nas-backup/it12-devops -maxdepth 1 -type d -mtime +7 \
    -name "*_snapshots" -exec rm -rf {} \;

echo "Backup snapshots completat: ${DATE}" | tee -a $LOG_FILE
echo "Espai total: $(du -sh ${BACKUP_DIR})" | tee -a $LOG_FILE
echo "========================================" | tee -a $LOG_FILE
EOF

sudo chmod +x /opt/devops/backup_snapshots.sh
```

## Estructura al NAS

```
/mnt/nas-backup/it12-devops/
    ├── 20260308_020000/              ← backup normal
    │   ├── docker-configs/
    │   ├── btrfs-data/
    │   ├── system/
    │   └── home/
    └── 20260308_030000_snapshots/    ← backup snapshots
        └── btrfs-snapshots/
```

## Planificació de tasques

| Script | Freqüència | Hora | Dia |
|--------|-----------|------|-----|
| `snapshot.sh` | Diari | 01:00 | Tots |
| `backup.sh` | Setmanal | 02:00 | Diumenge |
| `backup_snapshots.sh` | Diari | 03:00 | Tots |

## Resum del sistema de backup

Cada dia a les **01:00** → `snapshot.sh`
- Crea snapshots BTRFS de tots els serveis
- Retenció 7 dies

Cada dia a les **03:00** → `backup_snapshots.sh`
- Còpia snapshots al NAS
- Retenció 7 dies

Cada diumenge a les **02:00** → `backup.sh`
- Còpia completa al NAS: docker-configs, btrfs-data, etc, home
- Retenció 7 dies
