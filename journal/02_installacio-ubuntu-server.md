# Instal·lació Ubuntu Server 24.04 LTS

**Versió:** Ubuntu Server 24.04.4 LTS  
**Arquitectura:** amd64 (x86_64 / 64-bit)

---

## Distribucions recomanables per a Intel i7

| Distribució | Descripció |
|-------------|------------|
| Ubuntu 24.04 LTS (amd64) | Estable i fàcil d'instal·lar |
| Debian 12 (amd64) | Molt estable i lleugera |
| Fedora 40 (x86_64) | Més moderna i actualitzada |
| Linux Mint 21.3 (amd64) | Ideal per a ús d'escriptori |
| Arch Linux (x86_64) | Per a usuaris avançats |

---

## Opció PXE — Instal·lació per xarxa via Synology (opcional)

### 1. Activar TFTP en Synology

**DSM → Panel de control → Serveis de fitxers → TFTP**

- Activar TFTP
- Triar carpeta on poseu els fitxers d'arrancada (p. ex. `/tftpboot`)

### 2. Preparar els fitxers d'arrancada d'Ubuntu 24.04

```bash
mkdir /volume1/tftp/tftpboot
cd /volume1/tftp/tftpboot
wget https://releases.ubuntu.com/24.04/ubuntu-24.04.4-netboot-amd64.tar.gz
```

### 3. Arrancar el servidor per xarxa

1. Encén el servidor.
2. A BIOS/UEFI selecciona **Network Boot / PXE**.
3. El servidor demanarà IP per DHCP.
4. El Synology li enviarà el bootloader PXE.
5. Veuràs el menú d'instal·lació d'Ubuntu Server.

---

## Instal·lació pas a pas — Ubuntu Server 24.04.4 LTS

> **IMPORTANT — SERVEIS EN XARXA:**  
> SYNOLOGY-NAS (192.168.1.2) amb servei DNS actiu.  
> Assegura't que el DNS estigui **ACTIVAT** en Synology abans d'instal·lar Ubuntu.

**STATUS ACTUAL:** Ha arrencat ISO des d'USB, responent qüestionari d'arrancada.

### Configuració específica

| Paràmetre | Valor |
|-----------|-------|
| Idioma | English |
| Keyboard Layout | Spanish (España) |
| Address | 192.168.1.5/24 |
| Gateway | 192.168.1.1 |
| DNS primari (Synology NAS) | 192.168.1.2 |
| DNS secundari | 8.8.8.8 |
| Submask | 255.255.255.0 |
| Search domains | local |

---

## Pas 1 — Pantalla de benvinguda

- ✅ English (recomanat per a servidors)
- Prem Enter per continuar

## Pas 2 — Keyboard Layout

- ✅ Spanish (España)
- Prem "Done"

## Pas 3 — Network Configuration

> ⚠️ CONFIGURACIÓ IP ESTÀTICA REQUERIDA

Passos exactes a l'instal·lador:

1. A la pantalla "Network connections" selecciona la interfície de xarxa.
2. Prem ENTER o selecciona "Edit IPv4".
3. Canvia **DHCP4 → Manual**.
4. Introdueix:

| Camp | Valor |
|------|-------|
| Subnet | 192.168.1.0/24 |
| Address | 192.168.1.5 |
| Gateway | 192.168.1.1 |
| Nameservers | 192.168.1.2 8.8.8.8 |
| Search domains | local |

5. Prem "Save" i després "Done".

✅ Verifica que vegi: `192.168.1.5/24 via 192.168.1.1`

> **Aclariment — Search domains:** Són opcionals. Serveixen per resoldre noms locals
> (p. ex. `mi-servidor.local`). Si no tens domini local, deixa'l buit.

## Pas 4 — Configuració Proxy

- Si NO tens proxy: deixa'l buit.
- Si tens proxy: introdueix URL (p. ex. `http://proxy.domain:8080`).
- Prem "Done".

## Pas 5 — Ubuntu Archive Mirror

- Selecciona el mirror més proper al teu país.
- Per defecte: `archive.ubuntu.com` ✅
- Prem "Done".

## Pas 6 — Guided Storage Configuration

> ⚠️ PAS CRÍTIC — Configuració del disc

### Opció 1 (Recomanada) — Disc sencer

- Selecciona **"Use an entire disk"**.
- Escull el disc físic correcte (p. ex. `/dev/sda`).
- Ubuntu crearà automàticament:
  - Partició EFI (~512 MB)
  - Partició d'arrancada (`/boot`)
  - Grup LVM per a la resta
  - Volums lògics: `/` (20-50 GB), swap (automàtic)

### Opció 2 — Personalitzat

- Selecciona **"Custom storage layout"**.
- Crear particions manualment:
  - `/boot/efi`: 512 MB (EFI System)
  - `/`: 50 GB+ (ext4 — sistema arrel)
  - `/home`: resta (ext4 — dades d'usuari)

✅ Revisa el SUMMARY abans de confirmar. Prem "Done" per aplicar canvis.

## Pas 7 — Confirmació d'emmagatzematge

> ⚠️ ÚLTIM AVÍS ABANS D'ESCRIURE AL DISC

- Revisa els canvis que farà.
- Escriu "Confirm" si estàs segur.
- L'instal·lador escriurà al disc.

## Pas 8 — Profile Setup

| Camp | Valor |
|------|-------|
| Your name | edu |
| Server's name | it12-devops |
| Username | edu |
| Password | [contrasenya forta] |

Prem "Done".

## Pas 9 — SSH Server

- ✅ Instal·lar **OpenSSH Server** (RECOMANAT per a administració remota).
- Opcionalment: selecciona "Import SSH public key" des de GitHub → Settings → SSH keys.
- Si no vols importar, deixa buit i crea la clau després amb `ssh-keygen`.

## Pas 10 — Featured Server Snaps

- Selecciona el que necessitis (docker, postgres, etc.).
- Per a instal·lació bàsica: pots deixar buit.
- Prem "Done".

## Pas 11 — Instal·lació en progrés

⏳ El sistema està instal·lant paquets... (5-15 minuts)

Espera fins que vegis "Installation complete".

## Pas 12 — Reboot

1. Quan aparegui el missatge de finalització, prem **"Reboot Now"**.
2. **RETIRA la USB** quan vegis "Press ENTER to boot".
3. El sistema reiniciarà des del disc dur.

## Pas 13 — Primer Boot

1. Espera que el boot acabi (1-2 min).
2. Veuràs el prompt: `ubuntu-server login:`
3. Connecta amb l'usuari creat al Pas 8.

---

## Connexió SSH

```bash
ssh edu@it12-devops.local   # o bé:
ssh edu@192.168.1.5
```

### Resolució de problemes SSH

- **Nom local no resolt (`*.local`):** Usa mDNS/Avahi:

```bash
sudo apt install avahi-daemon
sudo systemctl enable --now avahi-daemon
```

- **Windows no resol `*.local`:** Instal·la Apple Bonjour o afegeix manualment a
  `C:\Windows\System32\drivers\etc\hosts`:

```
192.168.1.5  it12-devops.local it12-devops
```

- **Servidor no accepta contrasenya:** Verifica que `PasswordAuthentication yes`
  estigui a `/etc/ssh/sshd_config` i reinicia SSH:

```bash
sudo sed -i 's/^PasswordAuthentication no/PasswordAuthentication yes/' /etc/ssh/sshd_config
sudo systemctl restart ssh
```

- **Depurar SSH:**

```bash
ssh -vvv edu@192.168.1.5
```

- **Forçar ús de contrasenya:**

```bash
ssh -o PubkeyAuthentication=no edu@192.168.1.5
```

---

## Primeres accions després de la instal·lació

```bash
# 1. Actualitzar el sistema
sudo apt update
sudo apt upgrade -y

# 2. Instal·lar eines bàsiques
sudo apt install -y build-essential curl wget git vim htop

# 3. Configurar SSH
sudo vi /etc/ssh/sshd_config
# Canviar port (opcional): Port 2222
# Deshabilitar root: PermitRootLogin no
sudo systemctl restart ssh

# 4. Crear usuari addicional (si cal)
sudo useradd -m -s /bin/bash nouusuari
sudo usermod -aG sudo nouusuari

# 5. Configurar firewall (si cal)
sudo apt install -y ufw
sudo ufw allow 22/tcp
sudo ufw enable
```

---

## Preguntes freqüentment fetes (FAQ)

**Què significa "amd64"?**  
Arquitectura de 64 bits per a processadors Intel/AMD moderns.

**Puc canviar el layout de teclat després?**  
`sudo dpkg-reconfigure keyboard-configuration`

**Necessito DHCP obligatòriament?**  
No, pots usar IP estàtica. Ambdues funcionen igual.

**Puc instal·lar GUI (escriptori)?**  
Sí, però Ubuntu Server és millor sense GUI per a servidors.

**Quina mida ha de tenir /boot/efi?**  
Mínim 256 MB, recomanat 512 MB.

**És segur triar "disc sencer"?**  
Sí, si és una instal·lació neta. Assegura't de seleccionar el disc CORRECTE.

**Cada cop que uso sudo em demana contrasenya?**  
Per defecte sí. L'instal·lador crea una sessió de sudo de 5 minuts.

```bash
# Ajustar durada (editant amb visudo):
sudo visudo
# Afegeix: Defaults timestamp_timeout=15

# Treure contrasenya per a l'usuari (menys segur):
# edu ALL=(ALL) NOPASSWD: ALL
```

**Comandos bàsics de Vim:**

| Tecla | Acció |
|-------|-------|
| `i` | Entrar en mode inserció |
| `Esc` | Sortir a mode normal |
| `:w` | Guardar |
| `:q` | Sortir |
| `:wq` | Guardar i sortir |
| `u` | Desfer |
| `dd` | Esborrar línia |
| `yy` | Copiar línia |
| `p` | Enganxar |
| `/text` | Buscar |
| `:set number` | Mostrar núm. de línia |

---

## Notes addicionals

- Durant la instal·lació, Ubuntu verificarà actualitzacions.
- Alguns passos poden tardar, sigues pacient.
- Guarda la contrasenya de root/usuari en un lloc segur.
- Anota el hostname per a futures connexions SSH.

## Comandos útils del servidor

```bash
# Desactivar avahi (si no cal)
sudo systemctl stop avahi-daemon
sudo apt purge -y avahi-daemon avahi-utils
sudo rm -rf /etc/avahi

# Apagar el servidor
sudo shutdown -h now

# Reiniciar
sudo reboot
```
