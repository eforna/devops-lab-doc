# Auditoria de les Còpies de Seguretat

**Data auditoria:** 09/03/2026
**Servidor auditat:** IT12-DEVOPS (192.168.1.5)
**Destí backup:** NAS Synology DS220J → `/volume1/backup/it12-devops/`

---

## 1. Localització dels scripts de backup

Tots els scripts es troben al servidor **IT12-DEVOPS (192.168.1.5)**:

```
/opt/devops/backup.sh               ← Backup setmanal complet al NAS
/opt/devops/backup_snapshots.sh     ← Backup diari dels snapshots BTRFS al NAS
/opt/devops/snapshot.sh             ← Snapshots BTRFS locals diaris
/opt/devops/restore-snapshot.sh     ← Script de restauració manual
```

---

## 2. Scripts de backup actius

| Script | Freqüència | Hora | Cobertura |
|--------|-----------|------|-----------|
| `snapshot.sh` | Diari | 01:00 | Snapshots BTRFS locals |
| `backup.sh` | Setmanal (diumenge) | 02:00 | Fitxers al NAS |
| `backup_snapshots.sh` | Diari | 03:00 | Snapshots BTRFS al NAS |

---

## 2. Canvis recents fets al sistema (08-09/03/2026)

| Fitxer / Directori modificat | Canvi realitzat |
|------------------------------|-----------------|
| `/etc/netplan/99-dns.yaml` | Creat nou — DNS `192.168.1.2` permanent |
| `/opt/devops/*/docker-compose.yml` | Actualitzat `nip.io → devops.net` (tots els serveis) |
| `/opt/devops/harbor/harbor/harbor.yml` | Hostname actualitzat a `devops.net` |
| `/opt/devops/harbor/harbor/common/` | `chown -R 10000:10000` — permisos Harbor corregits |
| `/mnt/btrfs-data/harbor/` | Dades Harbor (contenidors reiniciats) |

---

## 3. Cobertura per directori

### ✅ Directoris COBERTS pel backup

| Directori | Script | Mètode |
|-----------|--------|--------|
| `/etc/` (inclou `/etc/netplan/99-dns.yaml`) | `backup.sh` | `tar -czf system/etc-*.tar.gz` |
| `/opt/devops/` (inclou tots els `docker-compose.yml`, `harbor.yml`, `common/`) | `backup.sh` | `tar -czf docker-configs/opt-devops-*.tar.gz` |
| `/mnt/btrfs-data/portainer/` | `backup.sh` + `snapshot.sh` | tar + BTRFS snapshot |
| `/mnt/btrfs-data/gitea/` | `backup.sh` + `snapshot.sh` | tar + BTRFS snapshot |
| `/mnt/btrfs-data/jenkins/` (sense war/plugins) | `backup.sh` + `snapshot.sh` | tar + BTRFS snapshot |
| `/mnt/btrfs-data/grafana/` | `backup.sh` + `snapshot.sh` | tar + BTRFS snapshot |
| `/mnt/btrfs-data/prometheus/` | `backup.sh` + `snapshot.sh` | tar + BTRFS snapshot |
| `/mnt/btrfs-data/keycloak/` | `backup.sh` + `snapshot.sh` | tar + BTRFS snapshot |
| `/mnt/btrfs-data/harbor/` | `backup.sh` + `snapshot.sh` | tar + BTRFS snapshot |
| `/home/edu/` | `backup.sh` | `tar -czf home/home-*.tar.gz` |
| `/mnt/btrfs-snapshots/` | `backup_snapshots.sh` | tar al NAS |

---

### ⚠️ Directoris NO coberts (gaps identificats)

| Directori | Contingut | Risc | Recomanació |
|-----------|-----------|------|-------------|
| `/mnt/btrfs-data/docker/` | Docker engine: imatges, capes, xarxes | **Mig** — es pot reconstruir amb `docker-compose up` | Documentar les imatges usades |
| `/var/log/harbor/` | Logs de Harbor | **Baix** — no crític | No necessari |
| `/var/log/backup-it12.log` | Historial d'execucions de backup | **Baix** | Opcional afegir al backup |
| `/mnt/nas-backup/` | El destí del backup en si | **N/A** — és el NAS | El NAS té RAID/redundància pròpia |

> ⚠️ **Gap principal:** `/mnt/btrfs-data/docker/` conté les **imatges Docker** descarregades.
> En cas de reinstal·lació, les imatges es tornen a descarregar automàticament amb `docker compose up`.
> **No és bloquejant** però pot trigar temps si la connexió és lenta.

---

## 4. Verificació de cobertura dels canvis recents

| Canvi realitzat | Directori afectat | Cobert? |
|-----------------|-------------------|---------|
| Creació `/etc/netplan/99-dns.yaml` | `/etc/` | ✅ Sí (backup.sh) |
| `docker-compose.yml` actualitzats | `/opt/devops/` | ✅ Sí (backup.sh) |
| `harbor.yml` actualitzat | `/opt/devops/harbor/harbor/` | ✅ Sí (backup.sh) |
| Permisos `common/` corregits | `/opt/devops/harbor/harbor/common/` | ✅ Sí (backup.sh) |
| Dades Harbor reiniciades | `/mnt/btrfs-data/harbor/` | ✅ Sí (backup.sh + snapshot.sh) |

---

## 5. Resultat de l'auditoria

**✅ TOTS els canvis recents estan coberts pel backup.**

El sistema de backup actual és adequat per a la recuperació del laboratori.
L'únic gap és `/mnt/btrfs-data/docker/` (imatges Docker), que és acceptable
perquè les imatges es poden regenerar automàticament.

### Millora implementada (09/03/2026)
Afegit al `backup.sh` la llista d'imatges en ús per poder-les re-descarregar ràpidament:
```bash
# Documentar imatges Docker en ús
docker images --format "{{.Repository}}:{{.Tag}}" > ${BACKUP_DIR}/docker-images-list.txt
```
> ✅ Implementat. El backup ara genera `docker-images-list.txt` amb totes les imatges actives.
