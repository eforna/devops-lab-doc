# Infraestructura

Documentació de la posada en marxa del servidor Geekom IT12 com a laboratori DevOps.

## Contingut

| Carpeta | Descripció | Estat |
|---------|------------|-------|
| [base/](base/README.md) | Ubuntu Server, BTRFS, SSH, configuració base | ✅ Completat |
| [docker-traefik/](docker-traefik/README.md) | Docker Engine, xarxa `devops-net`, Traefik reverse proxy | ✅ Completat |
| [backup/](backup/README.md) | Snapshots BTRFS automàtics + backup al NAS Synology | ✅ Completat |

## Arquitectura

```
Geekom IT12 — Ubuntu Server 24.04 LTS (BTRFS)
  │
  ├── Docker Engine
  │     └── xarxa devops-net
  │           └── Traefik (reverse proxy) → *.devops.lab
  │
  ├── Snapshots BTRFS locals   (diaris, 01:00)
  └── Backup NAS Synology      (setmanal, 02:00)
```

---

Torna al [README principal](../README.md).
