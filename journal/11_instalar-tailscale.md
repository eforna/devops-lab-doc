# Instal·lar i Configurar Tailscale

## Pas 1 — Instal·lar Tailscale

```bash
# Afegir repositori oficial
curl -fsSL https://tailscale.com/install.sh | sh

# Verificar instal·lació
tailscale version
```

## Pas 2 — Autenticar Tailscale

```bash
# Iniciar Tailscale
sudo tailscale up

# Et donarà un link per autenticar:
# https://login.tailscale.com/a/XXXXXXXXX
# Obre el link al navegador i autentifica't
```

## Pas 3 — Verificar connexió

```bash
# Veure IP de Tailscale
tailscale ip

# Veure estat
tailscale status

# Veure tots els dispositius de la xarxa
tailscale status --peers
```

## Pas 4 — Subnet router

> ✅ **No aplica.** Si tots els servidors estan donats d'alta a Tailscale,
> cada un té la seva pròpia IP Tailscale i es comuniquen directament.
> El subnet router és útil només per a dispositius que no poden tenir Tailscale
> instal·lat (impressores, càmeres, etc.).

Passos documentats per referència:

```bash
# Habilitar IP forwarding
echo 'net.ipv4.ip_forward = 1' | sudo tee -a /etc/sysctl.conf
echo 'net.ipv6.conf.all.forwarding = 1' | sudo tee -a /etc/sysctl.conf
sudo sysctl -p

# Anunciar la xarxa local
sudo tailscale up --advertise-routes=192.168.1.0/24

# Aprovar les routes al dashboard de Tailscale
# https://login.tailscale.com/admin/machines
```

## Pas 5 — Habilitar a l'inici

```bash
sudo systemctl enable tailscaled
sudo systemctl status tailscaled
```

---

## Resum

| Pas | Tasca | Estat |
|-----|-------|-------|
| 1 | Instal·lació v1.94.2 | ✅ |
| 2 | Autenticació | ✅ |
| 3 | Verificar connexió | ✅ |
| 4 | Subnet router | ⏭️ No necessari |
| 5 | Habilitar a l'inici | ✅ |

## Dades de connexió

| Camp | Valor |
|------|-------|
| Hostname | it12-devops |
| IP Tailscale | 100.108.144.100 |
| Estat | Connected |
| Usuari | efornaguera@gmail.com |
| Arrenca automàticament | ✅ |

```bash
# Connexió SSH remota
ssh edu@100.108.144.100
```

---

## MagicDNS

Permet accedir als dispositius pel nom en comptes de la IP Tailscale:

```
Sense MagicDNS:  ssh edu@100.108.144.100
Amb MagicDNS:    ssh edu@it12-devops
```

### Pas 1 — Activar MagicDNS

1. Anar a <https://login.tailscale.com/admin/dns>
2. Activar **MagicDNS**

### Pas 2 — Verificar resolució de noms

```bash
tailscale status
ping it12-devops
ping desktop-tdho715
ping synology
```

### Pas 3 — DNS personalitzat per als serveis (opcional)

1. Anar a <https://login.tailscale.com/admin/dns>
2. **Nameservers → Add nameserver → Custom**
   - IP: `100.108.144.100` ← IP Tailscale IT12-DEVOPS
   - Restrict to domain: `devops.local`

Resultat d'accés:

| Recurs | Accés local | Accés remot (Tailscale) |
|--------|-------------|-------------------------|
| IT12-DEVOPS | 192.168.1.5 | it12-devops |
| IT12-OKD | 192.168.1.4 | it12-okd |
| NAS Synology | 192.168.1.2 | synology |
| Gitea | gitea.devops.local | gitea.devops.local |

---

## DNS Forwarding al GL.iNet SFT1200

Accés: `http://192.168.8.1` → Network → DNS → DNS Forwarding

| Ordre | DNS | Funció |
|-------|-----|--------|
| 1 | 192.168.1.2 | NAS Synology (resol devops.local) |
| 2 | 8.8.8.8 | Google (resol internet) |
| 3 | 8.8.4.4 | Google secundari (backup) |
