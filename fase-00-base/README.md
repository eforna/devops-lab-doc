# Fase 00 — Sistema base

> **Objectiu**: Ubuntu instal·lat, accés SSH segur, BTRFS configurat amb subvolums i snapshots operatius.
> **Punt de control BTRFS**: Abans d'aquesta fase el sistema és l'instal·lador base. El primer snapshot es crea en finalitzar.

## Estat

✅ Completat

---

## 0.1 Verificar instal·lació Ubuntu

```bash
# Versió del sistema
lsb_release -a

# Particions i tipus de filesystem
lsblk -f

# Subvolums BTRFS existents
sudo btrfs subvolume list /
```

**Resultat esperat**: filesystem tipus `btrfs` a la partició principal.

---

## 0.2 Configurar subvolums BTRFS

L'estructura recomanada per a snapshots granulars:

```
@           →  /              (sistema)
@home       →  /home          (usuaris)
@var        →  /var           (logs, docker data)
@snapshots  →  /.snapshots    (snapshots)
```

```bash
# Veure subvolums actuals
sudo btrfs subvolume list /

# Si no existeixen, crear-los (adaptar segons instal·lació)
sudo btrfs subvolume create /.snapshots
sudo mkdir -p /.snapshots
```

> ⚠️ Si Ubuntu s'ha instal·lat amb subvolums propis (com `@` i `@home`), verificar abans de modificar.

---

## 0.3 Snapshot inicial — punt de control "base"

```bash
# Snapshot manual abans de qualsevol canvi
sudo /opt/devops/snapshots/snapshot.sh  # snapshot base-sistema-net
```

---

## 0.4 Configurar accés SSH des del Mac

```bash
# Al servidor: verificar que SSH està actiu
sudo systemctl status ssh

# Permetre només autenticació per clau (més segur)
sudo nano /etc/ssh/sshd_config
# → PasswordAuthentication no
# → PubkeyAuthentication yes

sudo systemctl reload ssh
```

```bash
# Al Mac: copiar clau pública al servidor
ssh-copy-id -i ~/.ssh/id_ed25519 -p 2222 edu@192.168.2.5

# Provar connexió
ssh -p 2222 edu@192.168.2.5
```

**Opcional — àlies al Mac** (`~/.ssh/config`):

```
Host devops-lab
    HostName 192.168.2.5
    User edu
    Port 2222
    IdentityFile ~/.ssh/id_ed25519
```

Des d'aleshores: `ssh devops-lab`

---

## 0.5 Configuració bàsica del servidor

```bash
# Timezone
sudo timedatectl set-timezone Europe/Andorra
timedatectl status

# Hostname
sudo hostnamectl set-hostname devops-lab
hostname

# Actualitzar sistema
sudo apt update && sudo apt upgrade -y

# Eines bàsiques
sudo apt install -y curl wget git vim htop net-tools unzip jq
```

---

## 0.6 Snapshot final — punt de control "post-base"

```bash
sudo /opt/devops/snapshots/snapshot.sh  # snapshot post-fase-00-base
```

---

## Com revertir a un snapshot

```bash
# Llistar subvolums de snapshot disponibles
sudo btrfs subvolume list /

# Revertir manualment muntant el subvolum desitjat
# (consultar documentació BTRFS per al rollback complet)
sudo reboot
```

---

## Notes i observacions

<!-- Documenta aquí el que trobis durant l'execució -->

| Data | Nota |
|------|------|
| | |

---

## Checklist de fase completada

- [x] Ubuntu instal·lat i actualitzat
- [x] Subvolums BTRFS verificats
- [x] Script de snapshots BTRFS configurat (`/opt/devops/snapshots/snapshot.sh`)
- [x] Snapshot "base" creat
- [x] SSH funcionant amb clau pública des del Mac (`edu@192.168.2.5` port `2222`)
- [x] Àlies SSH configurat al Mac
- [x] Snapshot "post-fase-00" creat

**Fase següent**: [Fase 01 — Infraestructura Docker + Traefik](../fase-01-infraestructura/README.md)
