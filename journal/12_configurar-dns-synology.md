# Configurar DNS Server al Synology DS220J
> el paquet DNS Server al DSM del Synology i crear la zona devops.net:
DSM → Package Center → DNS Server → Instal·lar

DSM → DNS Server
→ Zones → Create Master Zone
   Domain Type: Master Zone
   Domain Name: devops.net

→ Afegir registres A:
   traefik     → 192.168.1.5
   portainer   → 192.168.1.5
   gitea       → 192.168.1.5
   jenkins     → 192.168.1.5
   grafana     → 192.168.1.5
   prometheus  → 192.168.1.5
   keycloak    → 192.168.1.5
   harbor      → 192.168.1.5

## Flux DNS resultant

```
MacBook/Portàtil (via GL.iNet)
    │
    │ gitea.devops.local?
    ▼
GL.iNet SFT1200
    │
    │ DNS primari
    ▼
NAS Synology (192.168.1.2)
    │
    │ gitea.devops.local → 192.168.1.5
    ▼
IT12-DEVOPS → Traefik → Gitea
```

## Pas 1 - Instal·lar el paquet DNS Server
DSM -> Package Center -> Cerca "DNS Server" -> Instal·la


## Pas 2 - Obrir DNS Server
DSM -> Main Menu -> DNS Server

## Pas 3 - Crear la zona devops.net
DNS Server -> Zones -> Create -> Master Zone

  Domain Name:    devops.net
  Master DNS:     192.168.1.2
  Limit zone transfer: NO (deixar per defecte)


Clic a **OK** per crear la zona.

> ✅ **Zona creada.** El Synology genera automàticament:
> - `devops.net.`    NS  86400  `ns.devops.net.`
> - `ns.devops.net.` A   86400  `192.168.1.2`


## Pas 4 - Afegir registre wildcard (recomanat)

Un registre wildcard `*.devops.net -> 192.168.1.5` cobreix tots els serveis automàticament.

DNS Server -> Zones -> devops.net -> Edit
-> Resource Records -> Create

  Name:    *
  Type:    A
  TTL:     3600
  IP:      192.168.1.5

> ✅ **Registre wildcard creat i verificat.** `*.devops.net. -> 192.168.1.5`

> **Nota:** No calen registres A individuals per cada servei. El wildcard ja cobreix tot.


## Pas 5 - Afegir entrada per al propi NAS (opcional)

DNS Server -> Zones -> devops.net -> Edit -> Resource Records -> Create

  Name:    nas
  Type:    A
  TTL:     3600
  IP:      192.168.1.2


Tambe afegir registre al DNS de les maquines Ubuntu:
nas.devops.net -> 192.168.1.2

## Pas 6 - Verificar des d'IT12-DEVOPS

```bash
# Comprovar que el servidor DNS resol correctament
dig @192.168.1.2 gitea.devops.net
dig @192.168.1.2 jenkins.devops.net
dig @192.168.1.2 harbor.devops.net

# Provar amb nslookup
nslookup gitea.devops.net 192.168.1.5
```

> ✅ **Verificat.** `gitea.devops.net -> 192.168.1.5` resolt correctament pel Synology DNS.

```bash
# Provar sense especificar servidor (ha d'usar el GL.iNet -> Synology)
nslookup gitea.devops.net
ping gitea.devops.net
```

> ⏳ Pendent de verificar (requereix netplan actualitzat o GL.iNet com a DNS per defecte)

## Nota - Clients Windows/Mac

Si `ping gitea.devops.net` falla des de Windows, el DNS del PC no apunta al Synology.
Verificar:
```powershell
nslookup gitea.devops.net
ipconfig /all | findstr "DNS Servers"
```

> ⚠️ **Problema detectat al Windows DESKTOP-TDHO715:**
> DNS actiu: `2a0e:3c80:baca::224` (IPv6 ISP) -> no resol devops.net
> El GL.iNet no propaga el DNS al Windows automaticament.

Solucio - configurar DNS manualment a Windows (PowerShell com a administrador):
```powershell
# Identificar el nom de l'adaptador
Get-NetAdapter

# Configurar DNS (substituir "Wi-Fi" pel nom real de l'adaptador)
Set-DnsClientServerAddress -InterfaceAlias "Wi-Fi" -ServerAddresses ("192.168.1.2","8.8.8.8")

# Verificar
nslookup gitea.devops.net
```

O manualment:
```
Configuracio -> Xarxa -> WiFi -> Propietats de l'adaptador
-> Protocol d'Internet versio 4 (TCP/IPv4)
-> Usar les adreces de servidor DNS seguents:
   DNS preferit:    192.168.1.2
   DNS alternatiu:  8.8.8.8
```


## Pas 9 - Actualitzar /etc/resolv.conf a IT12-DEVOPS

Si el servidor no resol per defecte, afegir el Synology com a DNS primari:


```bash
sudo nano /etc/netplan/00-installer-config.yaml
```

```yaml
network:
  version: 2
  ethernets:
    eno1:
      addresses:
        - 192.168.1.5/24
      routes:
        - to: default
          via: 192.168.1.1
      nameservers:
        addresses:
          - 192.168.1.2    # Synology DNS (devops.net)
          - 8.8.8.8        # Google (internet)
          - 8.8.4.4        # Google backup
        search:
          - devops.net
```

```bash
sudo netplan apply
```


## Pas 7 - Actualitzar docker-compose.yml amb el nou domini

Un cop verificat que `devops.net` funciona, actualitzar els Traefik labels:

```bash
# Substituir nip.io per devops.net a tots els docker-compose.yml
sudo sed -i 's/192\.168\.1\.5\.nip\.io/devops.net/g' /opt/devops/*/docker-compose.yml

# Verificar canvis
grep -r "devops.net" /opt/devops/*/docker-compose.yml

# Reiniciar tots els serveis
cd /opt/devops/traefik && docker compose up -d
cd /opt/devops/portainer && docker compose up -d
cd /opt/devops/gitea && docker compose up -d
cd /opt/devops/jenkins && docker compose up -d
cd /opt/devops/grafana && docker compose up -d
cd /opt/devops/keycloak && docker compose up -d
```
```

> **Warning Harbor:** Harbor te la URL al fitxer `harbor.yml`, cal editar-lo manualment.

---

## URLs finals amb devops.net

| Servei     | URL                                 | Credencials     |
|------------|-------------------------------------|-----------------|
| Traefik    | http://traefik.devops.net           | -               |
| Portainer  | http://portainer.devops.net         | admin/B1ny2l3s@ |
| Gitea      | http://gitea.devops.net             | admin/B1ny2l3s@ |
| Jenkins    | http://jenkins.devops.net           | admin/B1ny2l3s@ |
| Grafana    | http://grafana.devops.net           | admin/B1ny2l3s@ |
| Prometheus | http://prometheus.devops.net        | -               |
| Keycloak   | http://keycloak.devops.net          | admin/B1ny2l3s@ |
| Harbor     | http://harbor.devops.net:8888       | admin/B1ny2l3s@ |
| NAS DSM    | http://nas.devops.net:5000          | -               |

---

## Resum

| Pas | Tasca                              | Estat |
|-----|------------------------------------|-------|
| 1   | Instal·lar DNS Server Package      | ✅    |
| 2   | Crear zona devops.net              | ✅    |
| 3   | Afegir wildcard *.devops.net       | ✅    |
| 4   | Verificar resolució des de Linux   | ✅    |
| 5   | Actualitzar netplan (si cal)       | ⏳    |
| 6   | Actualitzar docker-compose.yml     | ⏳    |

---

## Resum de sessió - 08/03/2026

### Que s'ha fet

**DNS Server al Synology DS220J (192.168.1.2)**

| Registre          | Tipus | TTL   | IP / Valor       | Estat |
|-------------------|-------|-------|------------------|-------|
| devops.net.       | NS    | 86400 | ns.devops.net.   | ✅ auto |
| ns.devops.net.    | A     | 86400 | 192.168.1.2      | ✅ auto |
| *.devops.net.     | A     | 3600  | 192.168.1.5      | ✅ manual |

> Nota: El registre wildcard es va introduir inicialment amb IP incorrecta (192.169.1.5).
> Corregit a 192.168.1.5.

**GL.iNet SFT1200 - DNS Forwarding**

| Ordre | DNS           | Funcio                        |
|-------|---------------|-------------------------------|
| 1     | 192.168.1.2   | Synology - resol devops.net   |
| 2     | 8.8.8.8       | Google - resol internet       |
| 3     | 8.8.4.4       | Google - backup               |

### Estat actual del DNS

El DNS del Synology ja resol `*.devops.net -> 192.168.1.5` per a tots els serveis.
Els clients de la xarxa que usen el GL.iNet com a gateway resoldran automaticament.

### Passos pendents

1. **Verificar resolucio** des d'IT12-DEVOPS:
   ```bash
   dig @192.168.1.2 gitea.devops.net
   ping grafana.devops.net
   ```

2. **Actualitzar netplan** a IT12-DEVOPS si no resol automaticament:
   ```bash
   sudo nano /etc/netplan/00-installer-config.yaml
   # nameservers: [192.168.1.2, 8.8.8.8]
   # search: [devops.net]
   sudo netplan apply
   ```

3. **Actualitzar docker-compose.yml** (canviar domini nip.io -> devops.net):
   ```bash
   sudo sed -i 's/192\.168\.1\.5\.nip\.io/devops.net/g' /opt/devops/*/docker-compose.yml
   ```

4. **Reiniciar serveis** Docker per aplicar el nou domini.

--------------------------------
> **Nota:** No calen CNAMEs ni DNS per als ports. El DNS no gestiona ports — això és feina de Traefik.

El flux és:

```
Client                   DNS              Traefik (port 80)      Docker
gitea.devops.net  →  192.168.1.5  →  Host: gitea.devops.net  →  gitea:3000
jenkins.devops.net →  192.168.1.5  →  Host: jenkins.devops.net → jenkins:8080
grafana.devops.net →  192.168.1.5  →  Host: grafana.devops.net → grafana:3000
```

El wildcard *.devops.net → 192.168.1.5 ja porta tot al servidor. Traefik, gràcies als labels dels docker-compose.yml, llegeix el Host: de la petició HTTP i la redirigeix al contenidor correcte al port intern corresponent.

Els CNAMEs serien útils per apuntar un nom a un altre nom (p.ex. www → gitea), però aquí no és necessari.

> **Important:** L'únic que cal fer ara és actualitzar els labels de Traefik als docker-compose.yml per canviar el domini de nip.io a devops.net:
sudo sed -i 's/192\.168\.1\.5\.nip\.io/devops.net/g' /opt/devops/*/docker-compose.yml