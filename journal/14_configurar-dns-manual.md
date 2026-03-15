# Configurar DNS manual als clients Windows/Mac

## Context
El router principal (192.168.1.1) no permet canviar els DNS del DHCP.
Cal configurar el DNS manualment a cada client.

---

## Netejar cache DNS (important!)

Si un nom resol malament o retorna una IP antiga, Windows pot tenir la resolució en cache.
Cal buidar-la per forçar una nova consulta al DNS:

```powershell
# Buidar cache DNS Windows
Clear-DnsClientCache

# Verificar que ha canviat
nslookup synology.devops.net
nslookup gitea.devops.net
```

> ✅ **Cas real (09/03/2026):** `synology.devops.net` retornava `192.168.1.5` en lloc de `192.168.1.2`.
> Solucio: `Clear-DnsClientCache` -> resolucio correcta a `192.168.1.2`.

Per Linux/Mac:
```bash
# Linux (systemd-resolved)
sudo systemd-resolve --flush-caches

# Mac
sudo dscacheutil -flushcache
sudo killall -HUP mDNSResponder
```

---



## Windows — Configurar DNS via PowerShell

```powershell
# Veure adaptadors actius
Get-NetAdapter | Where-Object {$_.Status -eq "Up"} | Select-Object Name, InterfaceDescription

# Configurar DNS
Set-DnsClientServerAddress -InterfaceAlias "Wi-Fi" -ServerAddresses ("192.168.1.2","8.8.8.8")

# Buidar cache DNS
Clear-DnsClientCache

# Verificar
nslookup gitea.devops.net

# Comprovar que es persistent
Get-DnsClientServerAddress -InterfaceAlias "Wi-Fi"
```

> ✅ **Verificat a DESKTOP-TDHO715 (08/03/2026):**
> ```
> Wi-Fi    IPv4    {192.168.1.2, 8.8.8.8}
> nslookup gitea.devops.net -> 192.168.1.5
> ```
> La configuracio es **persistent** (sobreviu als reinicis).

---

## Mac - Terminal

```bash
# Anar a System Settings -> Network -> Wi-Fi -> Details -> DNS
# Afegir: 192.168.1.2  i  8.8.8.8

# O per terminal
sudo networksetup -setdnsservers Wi-Fi 192.168.1.2 8.8.8.8
sudo dscacheutil -flushcache
nslookup gitea.devops.net
```

---

## IT12-DEVOPS - Netplan (permanent)

> **Per què dos fitxers netplan?**
> - **`50-cloud-init.yaml`** — generat automàticament per cloud-init (interfícies, DHCP, WiFi). **No s'edita** perquè es pot sobreescriure en qualsevol moment.
> - **`99-dns.yaml`** — creat per nosaltres, només conté els DNS. Els fitxers s'apliquen per ordre numèric, i el `99` sempre guanya sobre el `50`.
>
> Netplan fusiona els dos fitxers quan fas `netplan apply`. Resultat: xarxa del `50` + DNS del `99`.

El fitxer a editar és `/etc/netplan/50-cloud-init.yaml` (generat per cloud-init).
No s'edita directament — es crea un fitxer separat per no perdre els canvis:

```bash
# Crear fitxer DNS independent (no tocar el 50-cloud-init.yaml)
sudo tee /etc/netplan/99-dns.yaml << 'EOF'
network:
  version: 2
  ethernets:
    enp87s0:
      nameservers:
        addresses: [192.168.1.2, 8.8.8.8, 8.8.4.4]
        search: [devops.net]
  wifis:
    wlp86s0:
      nameservers:
        addresses: [192.168.1.2, 8.8.8.8, 8.8.4.4]
        search: [devops.net]
EOF

# Corregir permisos (obligatori, sinó netplan avisa)
sudo chmod 600 /etc/netplan/99-dns.yaml

# Aplicar
sudo netplan apply

# Verificar
nslookup gitea.devops.net
```

> ✅ **Verificat a IT12-DEVOPS (09/03/2026):**
> ```
> Server: 127.0.0.53
> gitea.devops.net -> 192.168.1.5
> ```

---

## Resum DNS per dispositiu

| Dispositiu        | DNS configurat        | Estat |
|-------------------|-----------------------|-------|
| IT12-DEVOPS       | /etc/netplan/99-dns.yaml (192.168.1.2, 8.8.8.8) | ✅    |
| DESKTOP-TDHO715   | 192.168.1.2, 8.8.8.8  | ✅    |
| MacBook Air M3    | pendent               | ⏳    |

---

## Actualitzar docker-compose.yml (08/03/2026)

```bash
sudo sed -i 's/192\.168\.1\.5\.nip\.io/devops.net/g' /opt/devops/*/docker-compose.yml
```

> ✅ **Verificat.** Tots els serveis reiniciats i funcionant amb `devops.net`.

| Servei     | URL                           | Estat |
|------------|-------------------------------|-------|
| Traefik    | http://traefik.devops.net     | ✅    |
| Portainer  | http://portainer.devops.net   | ✅    |
| Gitea      | http://gitea.devops.net       | ✅    |
| Jenkins    | http://jenkins.devops.net     | ✅    |
| Grafana    | http://grafana.devops.net     | ✅    |
| Prometheus | http://prometheus.devops.net  | ✅    |
| Keycloak   | http://keycloak.devops.net    | ✅    |
| Harbor     | http://harbor.devops.net:8888 | ✅    |
| NAS DSM    | http://nas.devops.net:5000    | ✅    |
