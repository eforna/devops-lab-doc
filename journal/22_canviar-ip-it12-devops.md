# Canvi d'IP del servidor IT12-DEVOPS

## Connexió SSH

```
ssh -p2222 usuari@192.168.1.5   # IP actual (abans del canvi)
ssh -p2222 usuari@192.168.2.5   # IP nova (després del canvi)
```

## Objectiu

Canviar la IP del servidor IT12-DEVOPS de la xarxa antiga a la nova, mantenint:

- Ubuntu Server sobre BTRFS (subvolums i muntatges actuals)
- Backups cap al NAS Synology
- Resolucio DNS dels serveis DevOps

## Dades de xarxa

| Element | Abans | Despres |
|---|---|---|
| IP servidor IT12-DEVOPS | 192.168.1.5/24 | 192.168.2.5/24 |
| Gateway | 192.168.1.1 | 192.168.2.1 |
| DNS principal (Synology) | 192.168.1.2 | 192.168.2.2 |
| Domini DNS | devops.net | devops.lab |

## Impacte esperat

- Traefik i serveis (portainer, keycloak, harbor, prometheus, grafana, jenkins, gitea) passen a resoldre a 192.168.2.5.
- Muntatge NFS del NAS passa a 192.168.2.2.
- Cal actualitzar qualsevol referencia hardcoded a 192.168.1.5, 192.168.1.2 i 192.168.1.1.
- Cal migrar FQDN i configuracions de devops.net a devops.lab.

## Pas 0 - Seguretat abans del canvi (BTRFS + backup)

Important: com que la instal lacio esta en BTRFS, no tocar UUID ni subvolums de l'arrel durant el canvi de xarxa.

1. Comprovar estat BTRFS:

    sudo btrfs filesystem df /
    sudo btrfs subvolume list /

2. Fer snapshot previ del sistema (recomanat):

    sudo mkdir -p /mnt/btrfs-root
    sudo mount -o subvol=/ /dev/nvme0n1p2 /mnt/btrfs-root
    sudo btrfs subvolume snapshot -r /mnt/btrfs-root/@ /mnt/btrfs-root/@snapshots/pre-ip-change-$(date +%F-%H%M)
    sudo btrfs subvolume list / | grep pre-ip-change
    sudo umount /mnt/btrfs-root

3. Executar backup manual abans de canviar xarxa:

    sudo /opt/devops/backup.sh

4. Verificar que la copia existeix al NAS abans de continuar.

## Pas 1 - Canvi d'IP a Ubuntu (netplan)

1. Identificar el nom de la interfície:

    ip -br a

2. Editar el fitxer netplan que s'estigui usant (normalment /etc/netplan/00-installer-config.yaml o /etc/netplan/99-dns.yaml):

    sudo nano /etc/netplan/00-installer-config.yaml

3. Deixar la configuracio equivalent a:

    network:
      version: 2
      renderer: networkd
      ethernets:
         enp87s0:
            dhcp4: no
            addresses: [192.168.2.5/24]
            routes:
              - to: default
                 via: 192.168.2.1
            nameservers:
              addresses: [192.168.2.2,8.8.8.8]

4. Aplicar amb validacio:

    sudo netplan generate
    sudo netplan try
    sudo netplan apply

5. Verificar:

    ip a
    ip r
    resolvectl status

## Pas 2 - Ajustar backups i muntatge NAS

1. Actualitzar IP del NAS a /etc/fstab:

    Abans:
    192.168.1.2:/volume1/backup  /mnt/nas-backup  nfs  defaults,_netdev,auto  0  0

    Despres:
    192.168.2.2:/volume1/backup  /mnt/nas-backup  nfs  defaults,_netdev,auto  0  0

2. Aplicar i validar:

    sudo systemctl daemon-reload
    sudo mount -a
    df -h | grep nas
    showmount -e 192.168.2.2

3. Synology DSM: canviar NFS Permissions del share backup.

- Treure host antic 192.168.1.5
- Afegir host nou 192.168.2.5 amb Read/Write

## Pas 3 - DNS Synology (zona devops.lab)

Actualitzar la zona amb serial nou (incrementar de 22 a 23 o superior):

    $ORIGIN devops.lab.
    $TTL 86400
    devops.lab. IN SOA ns.devops.lab. mail.devops.lab. (
         23
         43200
         180
         1209600
         10800
    )
    portainer.devops.lab.      86400 A 192.168.2.5
    nas.devops.lab.            86400 A 192.168.2.2
    gl-mt-3000.devops.lab.     86400 A 192.168.2.1
    gl-sf1200-2.devops.lab.    86400 A 192.168.2.10
    gl-sf1200-1.devops.lab.    86400 A 192.168.2.3
    synology.devops.lab.       86400 A 192.168.2.2
    keycloak.devops.lab.       86400 A 192.168.2.5
    harbor.devops.lab.         86400 A 192.168.2.5
    prometheus.devops.lab.     86400 A 192.168.2.5
    traefik.devops.lab.        86400 A 192.168.2.5
    grafana.devops.lab.        86400 A 192.168.2.5
    jenkins.devops.lab.        86400 A 192.168.2.5
    gitea.devops.lab.          86400 A 192.168.2.5
    api.okd.devops.lab.        86400 A 192.168.2.4
    api-int.okd.devops.lab.    86400 A 192.168.2.4
    *.devops.lab.              86400 A 192.168.2.2
    *.apps.okd.devops.lab.     86400 A 192.168.2.4
    devops.lab.                NS    ns.devops.lab.
    ns.devops.lab.             A     192.168.2.2

Nota: hi havia un error de tecleig a la wildcard (*.devops.lab) amb 19.168.2.2. El valor correcte es 192.168.2.2.

### Verificacio post-actualitzacio DNS (2026-03-14)

    dig @192.168.2.2 it12-devops.devops.lab +short  # resultat: 192.168.2.2 ❌ (error: apuntava al NAS)
    dig @192.168.2.2 gitea.devops.lab +short         # resultat: 192.168.2.5 ✅
    dig @192.168.2.2 nas.devops.lab +short           # resultat: 192.168.2.2 ✅

Correccio aplicada: registre A de it12-devops.devops.lab canviat de 192.168.2.2 a 192.168.2.5 al DSM.
Afegits registres A faltants: it12-devops.devops.lab → 192.168.2.5, it12-okd.devops.lab → 192.168.2.4.

Verificacio final DNS OK:

    dig @192.168.2.2 it12-devops.devops.lab +short  # 192.168.2.5 ✅
    dig @192.168.2.2 it12-okd.devops.lab +short     # 192.168.2.4 ✅
    dig @192.168.2.2 api.okd.devops.lab +short       # 192.168.2.4 ✅

## Pas 3.2 - Harbor (configuració específica)

Harbor no usa docker-compose.yml estàndard a /opt/devops/harbor sinó que té el seu propi harbor.yml.
El hostname estava configurat com a `harbor.192.168.1.5.nip.io` i s'ha canviat a `harbor.devops.lab`.

    sudo sed -i 's/hostname: harbor\.192\.168\.1\.5\.nip\.io/hostname: harbor.devops.lab/' /opt/devops/harbor/harbor/harbor.yml
    cd /opt/devops/harbor/harbor && sudo docker compose down && sudo docker compose up -d

Nota: Harbor no passa per Traefik. Accessible directament a http://harbor.devops.lab:8888

## Pas 3.1 - Migracio de domini devops.net a devops.lab

1. Revisar i actualitzar referencies antigues de domini als serveis:

    sudo rg -n "devops\\.net" /opt/devops /etc

2. Actualitzar hostnames en fitxers de configuracio (labels Traefik, URL base, webhook/callbacks, SSO).

3. Si hi ha certificats TLS emesos per devops.net, reemetre certificats per devops.lab.

4. Comprovacio posterior (ha de retornar buit o nomes historic):

    sudo rg -n "devops\\.net" /opt/devops /etc

## Pas 4 - Revisar configuracions de serveis

Fer cerca d'IPs i domini antics i corregir:

    sudo rg -n "192\\.168\\.1\\.5|192\\.168\\.1\\.2|192\\.168\\.1\\.1|devops\\.net" /opt/devops /etc

Punts habituals a revisar:

- /opt/devops/*/docker-compose.yml
- Configuracio Traefik (regles i middlewares)
- Configuracio Prometheus (targets node-exporter/docker)
- Scripts i variables d'entorn a /opt/devops

## Pas 5 - Validacions finals

1. Connectivitat de xarxa:

    ping -c 4 192.168.2.1
    ping -c 4 192.168.2.2

2. DNS:

    dig @192.168.2.2 gitea.devops.lab +short
    dig @192.168.2.2 jenkins.devops.lab +short
    dig @192.168.2.2 traefik.devops.lab +short
    dig @192.168.2.2 gitea.devops.net +short

3. Serveis:

    curl -I http://gitea.devops.lab
    curl -I http://jenkins.devops.lab
    curl -I http://grafana.devops.lab

Nota: la consulta de gitea.devops.net hauria de no resoldre (o quedar com a alias temporal controlat, si has definit una fase de convivencia).

4. Backups:

    sudo /opt/devops/backup.sh
    tail -n 50 /var/log/backup-it12.log

5. BTRFS (integritat post-canvi):

    mount | grep btrfs
    sudo btrfs subvolume list /

## Pla de rollback rapid

Si es perd conectivitat:

1. Obrir consola local (pantalla/teclat o KVM).
2. Tornar al netplan anterior (192.168.1.5/24, gateway 192.168.1.1, DNS 192.168.1.2).
3. Aplicar netplan i recuperar servei.
4. Restaurar DNS zona antiga temporalment.

## Registre d'execucio

Data del canvi: 14/03/2026

| Tasca | Estat | Notes |
|---|---|---|
| Snapshot BTRFS previ | ✅ | ID 494: .snapshots/pre-ip-change-2026-03-14-1142 |
| Backup manual previ | ✅ | Executat abans del canvi |
| Netplan actualitzat a 192.168.2.5 | ✅ | enp87s0, dhcp4: no, accept-ra: no |
| NFS/fstab actualitzat a 192.168.2.2 | ✅ | mount OK, test-ip-change creat |
| Permisos NFS Synology actualitzats | ✅ | 192.168.2.5 Read/Write |
| Zona DNS actualitzada (serial nou) | ✅ | it12-devops + it12-okd afegits, wildcard corregit |
| Migracio de devops.net a devops.lab validada | ✅ | sed -i als 6 docker-compose.yml + docker compose up -d |
| Serveis accessibles per FQDN | ✅ | traefik 405, gitea 200, grafana 302 |
| Backup post-canvi correcte | ✅ | 20260314_140108, 39M al NAS |
| Integritat BTRFS post-canvi | ✅ | 8 subvolums actius, snapshots diaris OK |

