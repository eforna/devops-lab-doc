# Instal·lar certificat autosignat HTTPS al Synology.devops.lab (DNS)

## 1. Generar el certificat autosignat

Des d’un terminal (MacBook, Linux o Synology SSH):

```bash
openssl req -x509 -nodes -days 365 -newkey rsa:4096 \
  -keyout dns-server.key -out dns-server.crt \
  -subj "/CN=synology.devops.lab"
```

- Això genera `dns-server.key` (clau privada) i `dns-server.crt` (certificat).
- Pots ajustar el nombre de dies (`-days 365`) o el nom (`/CN=...`).

## 2. Instal·lar el certificat al Synology

### Generar i instal·lar certificat autosignat des d'Ubuntu (IT12-DEVOPS)

### 1. Generar el certificat autosignat

```bash
openssl req -x509 -nodes -days 365 -newkey rsa:4096 \
  -keyout dns-server.key -out dns-server.crt \
  -subj "/CN=synology.devops.lab"
```

- Això genera `dns-server.key` (clau privada) i `dns-server.crt` (certificat).

### 2. Transferir el certificat al Synology

```bash
scp dns-server.crt dns-server.key admin@synology.devops.lab:/tmp/
```

- Substitueix `admin` pel teu usuari Synology si cal.

### 3. Importar el certificat al Synology

1. Accedeix a DSM → Panell de control → Seguretat → Certificat.
2. Fes clic a “Importa certificat”.
3. Selecciona:
   - Fitxer de certificat: `/tmp/dns-server.crt`
   - Clau privada: `/tmp/dns-server.key`
4. Assigna el certificat al servei DNS (o a tots els serveis).

### 4. Configurar el servei DNS per HTTPS (si aplica)

- Si el DNS Server de Synology permet HTTPS, assigna el certificat a la interfície web.
- Si només és per accés web a DSM, el certificat protegirà l’accés a la consola d’administració.

### 5. Acceptar el certificat en els clients

- El navegador mostrarà un avís de seguretat (per ser autosignat).
- Pots importar el certificat `dns-server.crt` als dispositius clients per evitar l’avís.

## 3. Configurar el servei DNS per HTTPS (si aplica)

- Si el DNS Server de Synology permet HTTPS, assigna el certificat a la interfície web.
- Si només és per accés web a DSM, el certificat protegirà l’accés a la consola d’administració.

## 4. Acceptar el certificat en els clients

- El navegador mostrarà un avís de seguretat (per ser autosignat).
- Pots importar el certificat `dns-server.crt` als dispositius clients per evitar l’avís.

## 5. Referència ràpida

```bash
# Comprovar el certificat
openssl x509 -in dns-server.crt -text -noout
```

---

Aquesta guia serveix per proves. Per producció, utilitza Let’s Encrypt o una CA reconeguda.

## Generar certificat autosignat directament des de DSM (Synology)

1. Accedeix a DSM → Panell de control → Seguretat → Certificat.
2. Fes clic a “Crea certificat” → “Crea un certificat autosignat”.
3. Introdueix el nom del domini (ex: synology.devops.lab) i les dades requerides.
4. DSM generarà el certificat i la clau automàticament.
5. Assigna el certificat als serveis que vulguis (DNS, consola web, etc.).

### Avantatges
- No cal transferir fitxers ni generar-los manualment.
- El certificat es crea i s’assigna directament des de la interfície DSM.

### Notes
- El certificat generat per DSM és autosignat i mostrarà avís de seguretat als navegadors.
- Pots exportar el certificat per importar-lo als clients si vols evitar l’avís.

## Assignar el certificat als serveis Synology

1. Accedeix a DSM → Panell de control → Seguretat → Certificat.
2. Veuràs la llista de certificats disponibles.
3. Fes clic a “Assigna servei” (Assign Service).
4. Selecciona el certificat que vols assignar.
5. Tria el servei:
   - Consola web DSM
   - DNS Server
   - File Station
   - Altres serveis compatibles
6. Desa els canvis.

A partir d’ara, el servei seleccionat utilitzarà el certificat assignat per HTTPS.

## Instal·lar el certificat autosignat al navegador Chrome

### Windows
1. Obre Chrome i ves a Configuració → Privadesa i seguretat → Seguretat.
2. Fes clic a “Gestiona certificats” (Manage certificates).
3. A la pestanya “Arrel de confiança” (Trusted Root Certification Authorities), fes clic a “Importa”.
4. Selecciona el fitxer `dns-server.crt` i segueix l’assistent.
5. Accepta i reinicia Chrome.

### Mac
1. Obre “Accés a Keychain” (Keychain Access).
2. Arrossega el fitxer `dns-server.crt` a “Sistema”.
3. Fes doble clic al certificat i marca “Confiar sempre”.
4. Reinicia Chrome.

### Linux
1. Copia el fitxer `dns-server.crt` a `/usr/local/share/ca-certificates/`.
2. Executa:
   ```bash
   sudo update-ca-certificates
   ```
3. Reinicia Chrome.

Així Chrome confiarà en el certificat autosignat i no mostrarà avís de seguretat.

## Configurar Traefik amb certificat autosignat (pas a pas)

### 1. Copia el certificat i la clau al servidor
- Posa `dns-server.crt` i `dns-server.key` a una carpeta, ex: `/opt/devops/traefik/certs/`

### 2. Modifica el docker-compose de Traefik

Exemple:
```yaml
docker-compose.yml
version: '3.7'
services:
  traefik:
    image: traefik:v2.10
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - /srv/traefik/certs/dns-server.crt:/certs/dns-server.crt:ro
      - /srv/traefik/certs/dns-server.key:/certs/dns-server.key:ro
      - /var/run/docker.sock:/var/run/docker.sock:ro
      - ./traefik.yml:/traefik.yml:ro
    restart: always
```

### 3. Configura traefik.yml

Exemple:
```yaml
entryPoints:
  web:
    address: ":80"
  websecure:
    address: ":443"
    tls:
      certificates:
        - certFile: "/certs/dns-server.crt"
          keyFile: "/certs/dns-server.key"
```

### 4. Reinicia Traefik
```bash
sudo docker compose down
sudo docker compose up -d
```

### 5. Comprova HTTPS
- Accedeix a https://gitea.devops.lab o https://synology.devops.lab
- El certificat autosignat hauria d’estar actiu.

---

## Configuració Traefik per HTTPS amb traefik.yml (configuració estàtica)

### 1. Crea la carpeta de certificats
```bash
mkdir -p /opt/devops/traefik/certs
```

### 2. Copia els fitxers dns-server.crt i dns-server.key a /opt/devops/traefik/certs

### 3. Crea o edita el fitxer traefik.yml

Exemple:
```yaml
entryPoints:
  web:
    address: ":80"
  websecure:
    address: ":443"
    tls:
      certificates:
        - certFile: "/certs/dns-server.crt"
          keyFile: "/certs/dns-server.key"
```

### 4. Edita el docker-compose.yml de Traefik

Exemple:
```yaml
services:
  traefik:
    image: traefik:v2.11
    container_name: traefik
    restart: always
    command:
      - "--api.dashboard=true"
      - "--api.insecure=true"
      - "--providers.docker=true"
      - "--providers.docker.exposedbydefault=false"
      - "--providers.docker.network=devops-net"
      - "--entrypoints.web.address=:80"
      - "--entrypoints.websecure.address=:443"
      - "--log.level=INFO"
      - "--accesslog=true"
      - "--configFile=/traefik.yml"
    ports:
      - "80:80"
      - "8080:8080"
      - "443:443"
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock:ro
      - /opt/devops/traefik/certs/dns-server.crt:/certs/dns-server.crt:ro
      - /opt/devops/traefik/certs/dns-server.key:/certs/dns-server.key:ro
      - /opt/devops/traefik/traefik.yml:/traefik.yml:ro
    networks:
      - devops-net
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.dashboard.rule=Host(`traefik.devops.lab`)"
      - "traefik.http.routers.dashboard.service=api@internal"
networks:
  devops-net:
    external: true
```

### 5. Reinicia Traefik
```bash
sudo docker compose down
sudo docker compose up -d
```

### 6. Comprova HTTPS
- Accedeix a https://gitea.devops.lab o https://synology.devops.lab
- El certificat autosignat hauria d’estar actiu.

---

> Aquesta configuració utilitza traefik.yml per definir TLS/certificats. És la forma recomanada per Traefik v2.x. No posis la configuració TLS/certificats directament als flags del command.