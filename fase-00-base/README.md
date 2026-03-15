# Fase 00 — Sistema base

> **Objetivo**: Ubuntu instalado, acceso SSH seguro, BTRFS configurado con subvolúmenes y snapshots operativos.  
> **Punto de control BTRFS**: Antes de esta fase el sistema es el instalador base. El primer snapshot se crea al finalizar.

## Estado

⬜ Pendiente

---

## 0.1 Verificar instalación Ubuntu

```bash
# Versión del sistema
lsb_release -a

# Particiones y tipo de filesystem
lsblk -f

# Subvolúmenes BTRFS existentes
sudo btrfs subvolume list /
```

**Resultado esperado**: filesystem tipo `btrfs` en la partición principal.

---

## 0.2 Configurar subvolúmenes BTRFS

La estructura recomendada para snapshots granulares:

```
@           →  /              (sistema)
@home       →  /home          (usuarios)
@var        →  /var           (logs, docker data)
@snapshots  →  /.snapshots    (snapshots)
```

```bash
# Ver subvolúmenes actuales
sudo btrfs subvolume list /

# Si no existen, crearlos (adaptar según instalación)
sudo btrfs subvolume create /.snapshots
sudo mkdir -p /.snapshots
```

> ⚠️ Si Ubuntu se instaló con subvolúmenes propios (como `@` y `@home`), verificar antes de modificar.

---

## 0.3 Instalar snapper (gestor de snapshots)

```bash
sudo apt update && sudo apt install -y snapper

# Crear configuración para root
sudo snapper -c root create-config /

# Ver configuración creada
sudo snapper -c root get-config
```

---

## 0.4 Crear primer snapshot — punto de control "base"

```bash
# Snapshot manual antes de cualquier cambio
sudo snapper -c root create --description "base-sistema-limpio" --cleanup-algorithm number

# Verificar
sudo snapper -c root list
```

---

## 0.5 Configurar acceso SSH desde Mac

```bash
# En el servidor: verificar que SSH está activo
sudo systemctl status ssh

# Permitir solo autenticación por clave (más seguro)
sudo nano /etc/ssh/sshd_config
# → PasswordAuthentication no
# → PubkeyAuthentication yes

sudo systemctl reload ssh
```

```bash
# En el Mac: copiar clave pública al servidor
ssh-copy-id <LAB_USER>@<LAB_IP>

# Probar conexión
ssh <LAB_USER>@<LAB_IP>
```

**Opcional — alias en Mac** (`~/.ssh/config`):

```
Host devops-lab
    HostName <LAB_IP>
    User <LAB_USER>
    IdentityFile ~/.ssh/id_ed25519
```

Desde entonces: `ssh devops-lab`

---

## 0.6 Configuración básica del servidor

```bash
# Timezone
sudo timedatectl set-timezone Europe/Madrid
timedatectl status

# Hostname
sudo hostnamectl set-hostname devops-lab
hostname

# Actualizar sistema
sudo apt update && sudo apt upgrade -y

# Herramientas básicas
sudo apt install -y curl wget git vim htop net-tools unzip jq
```

---

## 0.7 Crear usuario devops (si no existe)

```bash
sudo adduser devops
sudo usermod -aG sudo devops

# Verificar
id devops
```

---

## 0.8 Snapshot final — punto de control "post-base"

```bash
sudo snapper -c root create --description "post-fase-00-base" --cleanup-algorithm number
sudo snapper -c root list
```

---

## Cómo revertir a un snapshot

```bash
# Listar snapshots disponibles
sudo snapper -c root list

# Revertir al snapshot N (sin borrar el actual)
sudo snapper -c root undochange <N>..0

# O con rollback completo (requiere reinicio)
sudo snapper -c root rollback <N>
sudo reboot
```

---

## Notas y observaciones

<!-- Documenta aquí lo que encuentres durante la ejecución -->

| Fecha | Nota |
|-------|------|
| | |

---

## Checklist de fase completada

- [ ] Ubuntu instalado y actualizado
- [ ] Subvolúmenes BTRFS verificados
- [ ] Snapper instalado y configurado
- [ ] Snapshot "base" creado
- [ ] SSH funcionando con clave pública desde Mac
- [ ] Alias SSH configurado en Mac
- [ ] Usuario devops creado
- [ ] Snapshot "post-fase-00" creado

**Siguiente fase**: [Fase 01 — Infraestructura Docker + Traefik](../fase-01-infraestructura/README.md)
