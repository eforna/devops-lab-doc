# Configurar Snapshots BTRFS

## Estratègia

```
/mnt/btrfs-data/          ← dades en producció
/mnt/btrfs-snapshots/     ← snapshots automàtics
    ├── portainer/
    ├── gitea/
    ├── jenkins/
    ├── grafana/
    ├── prometheus/
    ├── keycloak/
    └── harbor/
```

---

## Pas 1 — Crear directori de snapshots

```bash
# Crear directori per snapshots
sudo mkdir -p /mnt/btrfs-snapshots

# Verificar que btrfs-data són subvolums
sudo btrfs subvolume list /mnt/btrfs-data
```

---

## Pas 2 — Script de snapshots

```bash
sudo tee /opt/devops/snapshots/snapshot.sh > /dev/null << 'EOF'
# NOTA: Aquest script es troba al repo it12-devops a opt/devops/snapshots/snapshot.sh
#!/bin/bash

DATE=$(date +%Y%m%d_%H%M%S)
LOG_FILE="/var/log/snapshot-it12.log"
SNAP_DIR="/mnt/btrfs-snapshots"

echo "========================================" | tee -a $LOG_FILE
echo "Snapshot iniciat: ${DATE}" | tee -a $LOG_FILE
echo "========================================" | tee -a $LOG_FILE

for dir in portainer gitea jenkins grafana prometheus keycloak harbor; do
    echo "  Snapshot ${dir}..." | tee -a $LOG_FILE
    mkdir -p ${SNAP_DIR}/${dir}
    btrfs subvolume snapshot -r \
        /mnt/btrfs-data/${dir} \
        ${SNAP_DIR}/${dir}/${dir}-${DATE}
    echo "  OK ${dir}: ${SNAP_DIR}/${dir}/${dir}-${DATE}" | tee -a $LOG_FILE
done

# Netejar snapshots antics (més de 7 dies)
echo "Netejant snapshots antics..." | tee -a $LOG_FILE
for dir in portainer gitea jenkins grafana prometheus keycloak harbor; do
    find ${SNAP_DIR}/${dir} -maxdepth 1 -type d -mtime +7 | while read snap; do
        echo "  Eliminant ${snap}..." | tee -a $LOG_FILE
        btrfs subvolume delete ${snap}
    done
done

echo "Snapshot completat: ${DATE}" | tee -a $LOG_FILE
echo "========================================" | tee -a $LOG_FILE
EOF

sudo chmod +x /opt/devops/snapshots/snapshot.sh
```

---

## Pas 3 — Provar el script

```bash
# Executar snapshot
sudo /opt/devops/snapshots/snapshot.sh

# Verificar snapshots creats
sudo btrfs subvolume list /mnt/btrfs-snapshots

# Veure estructura
ls -la /mnt/btrfs-snapshots/
ls -la /mnt/btrfs-snapshots/gitea/
```

---

## Pas 4 — Automatitzar amb cron

```bash
sudo crontab -e
```

Afegir les línies següents:

```
# Snapshot diari a la 01:00
0 1 * * * /opt/devops/snapshots/snapshot.sh

# Backup setmanal (diumenge) a les 02:00
0 2 * * 0 /opt/devops/backup/backup.sh

# Backup snapshots diari a les 03:00
0 3 * * * /opt/devops/backup/backup_snapshots.sh
```

---

## Pas 5 — Script de restauració de snapshot

```bash
sudo tee /opt/devops/snapshots/restore-snapshot.sh > /dev/null << 'EOF'
#!/bin/bash

echo "Snapshots disponibles:"
for dir in portainer gitea jenkins grafana prometheus keycloak harbor; do
    echo ""
    echo "  ${dir}:"
    ls /mnt/btrfs-snapshots/${dir}/
done

echo ""
echo "Introdueix el servei a restaurar (ex: gitea):"
read SERVICE

echo "Introdueix el snapshot a restaurar:"
ls /mnt/btrfs-snapshots/${SERVICE}/
read SNAPSHOT

SNAP_PATH="/mnt/btrfs-snapshots/${SERVICE}/${SNAPSHOT}"

if [ ! -d "${SNAP_PATH}" ]; then
    echo "ERROR: No existeix el snapshot ${SNAP_PATH}"
    exit 1
fi

echo "Restaurant ${SERVICE} des de ${SNAPSHOT}..."

cd /opt/devops/${SERVICE}
docker compose down

# Backup de seguretat de l'estat actual
btrfs subvolume snapshot \
    /mnt/btrfs-data/${SERVICE} \
    /mnt/btrfs-snapshots/${SERVICE}/${SERVICE}-before-restore-$(date +%Y%m%d_%H%M%S)

btrfs subvolume delete /mnt/btrfs-data/${SERVICE}
btrfs subvolume snapshot ${SNAP_PATH} /mnt/btrfs-data/${SERVICE}

docker compose up -d

echo "Restauració completada!"
echo "Verificar: docker compose ps"
EOF

sudo chmod +x /opt/devops/snapshots/restore-snapshot.sh
```

---

## Resum

| Pas | Tasca | Estat |
|-----|-------|-------|
| 1 | Directori `/mnt/btrfs-snapshots` creat | ✅ |
| 2 | Script `/opt/devops/snapshots/snapshot.sh` creat | ✅ |
| 3 | Snapshot de prova executat | ✅ |
| 4 | Cron configurat (diari 01:00) | ✅ |
| 5 | Script `/opt/devops/snapshots/restore-snapshot.sh` creat | ✅ |
