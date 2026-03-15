# Comprovar i configurar el DNS al Mac

## 1. Comprovar la configuració manual de DNS
- Ves a Preferències del Sistema → Xarxa → Wi-Fi/Ethernet → Avançat → DNS.
- Revisa si la IP del servidor DNS és 192.168.2.2.
- Si no és correcte, elimina la configuració manual i selecciona "Utilitza DHCP" per agafar el DNS del servidor.

## 2. Comprovar el DNS utilitzat per terminal
```bash
scutil --dns
```
Busca la secció "nameserver" i verifica que apareix 192.168.2.2.

## 3. Comprovar la resolució DNS
```bash
nslookup gitea.devops.lab
dig gitea.devops.lab
```
La resposta ha de venir de 192.168.2.2.

# Com verificar que els serveis funcionen correctament

## 1. Veure l'estat de tots els contenidors Docker
```bash
docker ps --format 'table {{.Names}}\t{{.Status}}\t{{.Ports}}'
```

## 2. Veure l'estat d'un servei concret
```bash
cd /opt/devops/nom_servei
docker compose ps
```

## 3. Veure els logs d'un servei
```bash
cd /opt/devops/nom_servei
docker compose logs -f
```

## 4. Comprovar l'accés web
- Obre el navegador i accedeix a l'URL del servei (ex: http://gitea.devops.lab)
- Comprova que la pàgina es carrega i pots iniciar sessió.

## 5. Comprovar la resolució DNS
```bash
nslookup gitea.devops.lab
dig gitea.devops.lab
```

## 6. Comprovar la connexió per ping
```bash
ping gitea.devops.lab
ping 192.168.2.5
```
# Scripts útils per actualitzar la IP i reiniciar serveis

## Actualitzar la IP en tots els docker-compose.yml

```bash
cd /opt/devops
for svc in traefik portainer gitea jenkins grafana prometheus keycloak harbor; do
	if [ -f "$svc/docker-compose.yml" ]; then
		sed -i 's/192.168.1.5/192.168.2.5/g' $svc/docker-compose.yml
	fi
done
```

## Reiniciar tots els serveis Docker

```bash
for svc in traefik portainer gitea jenkins grafana prometheus keycloak harbor; do
	if [ -d "$svc" ]; then
		cd $svc
		docker compose down
		docker compose up -d
		cd ..
	fi
done
```
## Què són els directoris que comencen amb @?

Els directoris que comencen amb @ (per exemple, @docker, @gitea, @jenkins...) són subvolums BTRFS. Un subvolum BTRFS és una partició virtual dins el sistema de fitxers BTRFS, creada per separar les dades de cada servei.

### Avantatges dels subvolums BTRFS:
- Permet snapshots independents per servei.
- Facilita backups i recuperació selectiva.
- Permet quotes d’espai i esborrat net sense afectar altres serveis.

Cada subvolum es munta en un directori dedicat dins `/mnt/btrfs-data/`, garantint una gestió modular i segura de les dades.

# Arbre de fitxers del projecte DEVOPS

A continuació es mostra l'estructura principal de directoris i subvolums utilitzada en el laboratori DEVOPS:

```
/opt/devops/
├── traefik/
│   └── docker-compose.yml
├── portainer/
│   └── docker-compose.yml
├── gitea/
│   └── docker-compose.yml
├── jenkins/
│   └── docker-compose.yml
├── grafana/
│   ├── docker-compose.yml
│   └── prometheus.yml
├── prometheus/
│   └── docker-compose.yml
├── keycloak/
│   └── docker-compose.yml
├── harbor/
│   ├── harbor.yml
│   └── docker-compose.yml (si existeix)
└── .env

/mnt/btrfs-data/
├── docker/
├── portainer/
├── gitea/
├── jenkins/
├── grafana/
├── prometheus/
├── harbor/
└── keycloak/
```

## Descripció
- `/opt/devops/`: Directori base per a la configuració i desplegament dels serveis Docker.
- `/mnt/btrfs-data/`: Punt de muntatge dels subvolums BTRFS per a dades persistents de cada servei.
- Cada servei té el seu subvolum dedicat per facilitar snapshots, backups i recuperació.
- Els fitxers `docker-compose.yml` defineixen la configuració de cada servei.
- El fitxer `.env` centralitza variables d'entorn comunes.

Aquesta estructura permet una gestió modular, segura i eficient dels serveis del laboratori DEVOPS.