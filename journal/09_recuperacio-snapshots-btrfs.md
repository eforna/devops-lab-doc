# Recuperació de Snapshots BTRFS

## Mida dels snapshots (08/03/2026)

```bash
# Mida de cada directori de snapshots
sudo du -sh /mnt/btrfs-snapshots/*/

# Mida total
sudo du -sh /mnt/btrfs-snapshots/
```

| Servei | Mida snapshot |
|--------|---------------|
| portainer | 488 KB |
| gitea | 4.3 MB |
| prometheus | 4.7 MB |
| keycloak | 1.6 MB |
| grafana | 102 MB |
| harbor | 106 MB |
| jenkins | 654 MB ⚠️ |
| **Total** | **~873 MB** |

> ⚠️ Jenkins és gran per les dependències de Maven. Es pot reduir excloent `war/` i `plugins/`.

---

## Pas 1 — Verificar snapshots disponibles

```bash
# Veure snapshots disponibles
ls -la /mnt/btrfs-snapshots/
ls -la /mnt/btrfs-snapshots/gitea/
```

---

## Pas 2 — Parar el servei (exemple: Gitea)

```bash
cd /opt/devops/gitea
docker compose down

# Verificar que està aturat
docker compose ps
```

---

## Pas 3 — Backup de seguretat abans de restaurar

```bash
# Snapshot de l'estat actual (per poder fer rollback del rollback)
sudo btrfs subvolume snapshot \
    /mnt/btrfs-data/gitea \
    /mnt/btrfs-snapshots/gitea/gitea-before-restore-$(date +%Y%m%d_%H%M%S)

# Verificar
ls -la /mnt/btrfs-snapshots/gitea/
```

---

## Pas 4 — Restaurar snapshot

```bash
# Eliminar subvolum actual
sudo btrfs subvolume delete /mnt/btrfs-data/gitea

# Restaurar snapshot (ajustar la data al snapshot disponible)
sudo btrfs subvolume snapshot \
    /mnt/btrfs-snapshots/gitea/gitea-20260308_024037 \
    /mnt/btrfs-data/gitea

# Verificar
ls -la /mnt/btrfs-data/gitea/
```

---

## Pas 5 — Arrancar el servei

```bash
cd /opt/devops/gitea
docker compose up -d

# Verificar
docker compose ps
docker compose logs -f
```

---

## Pas 6 — Verificar que funciona

```bash
docker compose ps
```

Accedir al navegador:
- `http://gitea.devops.net`

---

## Script automàtic de restauració

Per restaurar qualsevol servei de forma interactiva:

```bash
sudo /opt/devops/restore-snapshot.sh
```

El script llista els snapshots disponibles i guia el procés pas a pas.
