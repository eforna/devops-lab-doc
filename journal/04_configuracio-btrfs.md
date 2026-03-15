# Configuració BTRFS

## 1. Comprovar que esteu sobre BTRFS

Després del primer arrencada, comprova el sistema de fitxers:

```bash
df -T /
```

Ha de sortir:

```
/dev/nvme0n1p2   btrfs
```

Això confirma que la instal·lació és correcta.

## 2. Muntar BTRFS en mode "raw" per crear subvolums

Primer crea un directori temporal:

```bash
sudo mkdir /mnt/btrfs
```

Ara munta la partició sense subvolum:

```bash
sudo mount -o subvol=/ /dev/nvme0n1p2 /mnt/btrfs
```

> Substitueix `/dev/nvme0n1p2` pel teu dispositiu si és diferent.

## 3. Crear els subvolums recomanats

Estructura ideal per a K3s, Jenkins, Gitea, Prometheus, Grafana i backups:

```bash
sudo btrfs subvolume create /mnt/btrfs/@
sudo btrfs subvolume create /mnt/btrfs/@home
sudo btrfs subvolume create /mnt/btrfs/@log
sudo btrfs subvolume create /mnt/btrfs/@k3s
sudo btrfs subvolume create /mnt/btrfs/@pvc
sudo btrfs subvolume create /mnt/btrfs/@jenkins
sudo btrfs subvolume create /mnt/btrfs/@gitea
sudo btrfs subvolume create /mnt/btrfs/@prometheus
sudo btrfs subvolume create /mnt/btrfs/@grafana
sudo btrfs subvolume create /mnt/btrfs/@snapshots
sudo btrfs subvolume create /mnt/btrfs/@backups
```

## 4. Muntar els subvolums al sistema

Primer desmunta:

```bash
sudo umount /mnt/btrfs
```

Ara edita `/etc/fstab`:

```bash
sudo nano /etc/fstab
```

Afegeix les línies següents (substitueix `UUID=XXXX` pel teu UUID real, obtingut amb `blkid`):

```
UUID=XXXX /                            btrfs  subvol=@,compress=zstd,noatime,ssd,space_cache=v2 0 0
UUID=XXXX /home                        btrfs  subvol=@home,compress=zstd,noatime,ssd,space_cache=v2 0 0
UUID=XXXX /var/log                     btrfs  subvol=@log,compress=zstd,noatime,ssd,space_cache=v2 0 0
UUID=XXXX /var/lib/rancher/k3s         btrfs  subvol=@k3s,compress=zstd,noatime,ssd,space_cache=v2 0 0
UUID=XXXX /var/lib/rancher/k3s/storage btrfs  subvol=@pvc,compress=zstd,noatime,ssd,space_cache=v2 0 0
UUID=XXXX /var/lib/jenkins             btrfs  subvol=@jenkins,compress=zstd,noatime,ssd,space_cache=v2 0 0
UUID=XXXX /var/lib/gitea               btrfs  subvol=@gitea,compress=zstd,noatime,ssd,space_cache=v2 0 0
UUID=XXXX /var/lib/prometheus          btrfs  subvol=@prometheus,compress=zstd,noatime,ssd,space_cache=v2 0 0
UUID=XXXX /var/lib/grafana             btrfs  subvol=@grafana,compress=zstd,noatime,ssd,space_cache=v2 0 0
```

## 5. Reiniciar i verificar

```bash
sudo reboot
```

Després comprova que tots els subvolums estan muntats:

```bash
mount | grep btrfs
```

## 6. Preparar snapshots automàtics (opcional però recomanat)

**Opció lleugera — btrbk:**

```bash
sudo apt install btrbk
```

**Opció avançada — snapper:**

```bash
sudo apt install snapper
```

## 7. Preparar enviament de snapshots al NAS Synology

Com que el DS220j no suporta BTRFS natiu, enviarem snapshots com a fitxers:

```bash
sudo btrfs send /@k3s-snap | gzip > /mnt/nas/k3s-$(date +%F).btrfs.gz
```

Després el NAS pot fer versions amb Hyper Backup.

## 8. Instal·lar K3s sobre subvolums BTRFS

Ara ja tens:

- Snapshots instantànies
- Compressió zstd
- Integritat de dades
- Subvolums separats per cada servei

És el moment ideal per instal·lar K3s:

```bash
curl -sfL https://get.k3s.io | sudo sh -
```
