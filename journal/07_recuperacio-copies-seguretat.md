# Recuperació de Còpies de Seguretat

## Estratègia de proves

> ⚠️ **No tocar el sistema en producció!**
> Totes les proves es fan en un directori temporal `/tmp/restore-test/`.

El backup de referència utilitzat per les proves: `20260308_021319`
Ubicació al NAS: `/mnt/nas-backup/it12-devops/`

---

## Pas 1 — Crear directori de proves

```bash
# Crear directori temporal per les proves
sudo mkdir -p /tmp/restore-test/{docker-configs,btrfs-data,system,home}

# Veure backups disponibles
ls -la /mnt/nas-backup/it12-devops/
```

---

## Pas 2 — Provar recuperació `/opt/devops`

```bash
# Llistar contingut del backup sense extreure
tar -tzvf /mnt/nas-backup/it12-devops/20260308_021319/docker-configs/opt-devops-20260308_021319.tar.gz

# Extreure al directori de proves
sudo tar -xzvf /mnt/nas-backup/it12-devops/20260308_021319/docker-configs/opt-devops-20260308_021319.tar.gz \
    -C /tmp/restore-test/docker-configs/

# Verificar
ls -la /tmp/restore-test/docker-configs/opt/devops/
```

---

## Pas 3 — Provar recuperació `/mnt/btrfs-data`

```bash
# Provar amb gitea (és el més petit)
tar -tzvf /mnt/nas-backup/it12-devops/20260308_021319/btrfs-data/gitea-20260308_021319.tar.gz | head -20

# Extreure al directori de proves
sudo tar -xzvf /mnt/nas-backup/it12-devops/20260308_021319/btrfs-data/gitea-20260308_021319.tar.gz \
    -C /tmp/restore-test/btrfs-data/

# Verificar
ls -la /tmp/restore-test/btrfs-data/
ls -la /tmp/restore-test/btrfs-data/mnt/btrfs-data/gitea/
```

---

## Pas 4 — Provar recuperació `/etc`

```bash
sudo tar -xzvf /mnt/nas-backup/it12-devops/20260308_021319/system/etc-20260308_021319.tar.gz \
    -C /tmp/restore-test/system/

# Verificar fitxers clau recuperats
cat /tmp/restore-test/system/etc/fstab
cat /tmp/restore-test/system/etc/netplan/99-dns.yaml
```

---

## Pas 5 — Provar recuperació `/home/edu`

```bash
sudo tar -xzvf /mnt/nas-backup/it12-devops/20260308_021319/home/home-20260308_021319.tar.gz \
    -C /tmp/restore-test/home/

# Verificar
ls -la /tmp/restore-test/home/edu/
```

---

## Pas 5b — Recuperar imatges Docker (afegit 09/03/2026)

El backup inclou un fitxer `docker-images-list.txt` amb totes les imatges actives en el moment del backup.
En cas de reinstal·lació, usar-lo per re-descarregar les imatges:

```bash
# Veure quines imatges hi havia
cat /mnt/nas-backup/it12-devops/20260308_021319/docker-images-list.txt

# Re-descarregar totes les imatges de cop
while IFS= read -r image; do
    echo "Descarregant $image..."
    docker pull "$image"
done < /mnt/nas-backup/it12-devops/20260308_021319/docker-images-list.txt

# Un cop descarregades, arrancar tots els serveis
cd /opt/devops/traefik       && docker compose up -d
cd /opt/devops/portainer     && docker compose up -d
cd /opt/devops/gitea         && docker compose up -d
cd /opt/devops/jenkins       && docker compose up -d
cd /opt/devops/grafana       && docker compose up -d
cd /opt/devops/keycloak      && docker compose up -d
cd /opt/devops/harbor/harbor && docker compose up -d
```

---

## Pas 6 — Netejar directori de proves

```bash
# Un cop verificat tot, netejar
sudo rm -rf /tmp/restore-test/

# Verificar que s'ha netejat
ls /tmp/ | grep restore
```

---

## Resum del sistema de backup

| Tasca | Estat |
|-------|-------|
| NFS configurat al Synology | ✅ |
| NAS muntat a `/mnt/nas-backup` | ✅ |
| `fstab` configurat | ✅ |
| Script `/opt/devops/backup.sh` | ✅ |
| Script `/opt/devops/backup_snapshots.sh` | ✅ |
| Script `/opt/devops/snapshot.sh` | ✅ |
| Script `/opt/devops/restore-snapshot.sh` | ✅ |
| Cron configurat (01:00 / 02:00 / 03:00) | ✅ |
| Proves de recuperació | ✅ |
| Llista imatges Docker al backup | ✅ |
