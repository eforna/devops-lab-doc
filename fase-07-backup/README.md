# Fase 07 — Backup i recuperació

> **Objectiu**: Estratègia de backup completa: snapshots BTRFS automàtics locals + backup al NAS Synology. Procediment de recuperació documentat i provat.

## Estat

✅ Completat

---

## Estratègia de backup

```
Servidor Ubuntu (BTRFS)
  │
  ├── Snapshots BTRFS locals  (/opt/devops/snapshots/snapshot.sh)
  │     └── Execució diària a les 01:00
  │
  ├── Backup complet al NAS    (/opt/devops/backup/backup.sh)
  │     └── Execució setmanal (diumenges) a les 02:00
  │
  └── Snapshots al NAS         (/opt/devops/backup/backup_snapshots.sh)
        └── Execució diària a les 03:00
              └── NAS: 192.168.2.2 (Synology DS220J)
                    └── Destí: /mnt/nas-backup/it12-devops/
```

---

## 7.1 Scripts de backup desplegats

Tots els scripts estan desplegats via el repositori `it12-devops` i el seu `deploy.sh`.

### `/opt/devops/snapshots/snapshot.sh`

Snapshot BTRFS local. S'executa diàriament a les 01:00.

```bash
# Ús manual:
sudo /opt/devops/snapshots/snapshot.sh <descripció>

# Exemples:
sudo /opt/devops/snapshots/snapshot.sh pre-actualitzacio
sudo /opt/devops/snapshots/snapshot.sh post-canvis-traefik
```

### `/opt/devops/backup/backup.sh`

Backup complet al NAS Synology (192.168.2.2). S'executa cada diumenge a les 02:00.

```bash
# Ús manual:
sudo /opt/devops/backup/backup.sh
```

Fa rsync de `/opt/devops/` cap a `/mnt/nas-backup/it12-devops/` al NAS.

### `/opt/devops/backup/backup_snapshots.sh`

Còpia dels snapshots BTRFS locals al NAS. S'executa diàriament a les 03:00.

```bash
# Ús manual:
sudo /opt/devops/backup/backup_snapshots.sh
```

---

## 7.2 Configuració de cron

La configuració de cron ja està activa al servidor. Les tasques programades són:

```
# Snapshot BTRFS diari — 01:00
0 1 * * * root /opt/devops/snapshots/snapshot.sh auto-daily

# Backup complet al NAS — diumenges 02:00
0 2 * * 0 root /opt/devops/backup/backup.sh

# Snapshots al NAS — diari 03:00
0 3 * * * root /opt/devops/backup/backup_snapshots.sh
```

Verificar que els timers/crons estan actius:

```bash
sudo crontab -l
# o
cat /etc/cron.d/devops-backup
```

---

## 7.3 Accés SSH al NAS

La clau SSH per a backup ja està configurada sense passphrase per permetre l'automatització:

```bash
# Verificar accés al NAS
ssh -i ~/.ssh/id_backup backup@192.168.2.2 "echo connexió OK"

# Verificar que el destí existeix al NAS
ssh -i ~/.ssh/id_backup backup@192.168.2.2 "ls /mnt/nas-backup/it12-devops/"
```

---

## 7.4 Procediment de recuperació

### Cas 1: Revertir un canvi recent (snapshot BTRFS local)

```bash
# Veure snapshots disponibles
sudo /opt/devops/snapshots/restore-snapshot.sh --list

# Restaurar un snapshot concret
sudo /opt/devops/snapshots/restore-snapshot.sh <nom-del-snapshot>
```

### Cas 2: Restaurar dades Docker des del NAS

```bash
# Aturar el servei afectat
docker compose -f /opt/devops/gitea/docker-compose.yml down

# Restaurar des del NAS
rsync -avz \
  -e "ssh -i ~/.ssh/id_backup" \
  backup@192.168.2.2:/mnt/nas-backup/it12-devops/gitea/ \
  /opt/devops/gitea/

# Reiniciar el servei
docker compose -f /opt/devops/gitea/docker-compose.yml up -d
```

### Cas 3: Recuperació total del servidor

```bash
# 1. Reinstal·lar Ubuntu amb BTRFS
# 2. Clonar el repositori it12-devops i executar deploy.sh
# 3. Restaurar dades des del NAS amb rsync invers cap a /opt/devops/
# 4. Reinstal·lar Docker i aixecar tots els compose
```

---

## 7.5 Verificar integritat del backup

```bash
# Prova manual de backup
sudo /opt/devops/backup/backup.sh

# Revisar logs
journalctl -u cron | grep backup
# o
tail -f /var/log/syslog | grep backup
```

Registre de proves de restauració:

| Data | Servei restaurat | Resultat | Temps |
|------|-----------------|----------|-------|
| | | | |

---

## 7.6 Snapshot final

```bash
sudo /opt/devops/snapshots/snapshot.sh post-fase-07-backup
```

---

## Notes i observacions

| Data | Nota |
|------|------|
| | |

---

## Checklist de fase completada

- [x] Script `snapshot.sh` desplegat i funcional
- [x] Script `backup.sh` desplegat i provat
- [x] Script `backup_snapshots.sh` desplegat i provat
- [x] Cron de snapshot diari configurat (01:00)
- [x] Cron de backup setmanal configurat (diumenges 02:00)
- [x] Cron de snapshots al NAS configurat (03:00)
- [x] Accés SSH al NAS (192.168.2.2) configurat sense passphrase
- [x] Procediment de recuperació documentat
- [x] Primera prova de restauració realitzada
- [x] Snapshot "post-fase-07" creat

---

**Laboratori DevOps complet i documentat.**

Torna al [README principal](../README.md) per actualitzar l'estat de totes les fases.
