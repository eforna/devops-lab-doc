# Instal·lar Tailscale al GL.iNet SFT1200
> ⚠️ **Resultat:** ❌ No compatible  
> **Motiu:** MIPS32r2 softfloat — Tailscale no ofereix binari compatible  
> **Alternativa:** Instal·lar Tailscale directament al router principal si és compatible

## Context
El GL.iNet SFT1200 usa OpenWrt i **no disposa del paquet Tailscale** al repositori oficial.
Cal instal·lar-lo manualment descarregant el binari estàtic per a l'arquitectura MIPS.

**Dades del dispositiu:**
- Model: GL.iNet SFT1200
- IP: 192.168.1.3
- OS: OpenWrt
- Arquitectura: MIPS interAptiv (multi) V2.8

---

## Pas 1 - Connectar per SSH

```bash
ssh -oHostKeyAlgorithms=+ssh-rsa -oPubkeyAcceptedAlgorithms=+ssh-rsa root@192.168.1.3
```

---

## Pas 2 - Verificar arquitectura

```bash
uname -m
cat /proc/cpuinfo | grep "cpu model"
```

> Resultat: `mips` / `MIPS interAptiv (multi) V2.8`

---

## Pas 3 - Consultar binaris disponibles

```bash
wget -q -O- https://pkgs.tailscale.com/stable/ | grep -o 'tailscale_[^"]*mips[^"]*\.tgz'
```

> Resultat: nomes existeix `tailscale_1.94.2_mips.tgz` (no hi ha softfloat)
> La versio 1.68.1_mips fallava per incompatibilitat ABI. La 1.94.2 funciona.

---

## Pas 4 - Descarregar i instal·lar binari correcte

```bash
cd /root
wget https://pkgs.tailscale.com/stable/tailscale_1.94.2_mips.tgz
tar xzf tailscale_1.94.2_mips.tgz
cd tailscale_1.94.2_mips

cp tailscale /usr/sbin/
cp tailscaled /usr/sbin/
chmod +x /usr/sbin/tailscale /usr/sbin/tailscaled

# Verificar
tailscale version
```

---

## Pas 5 - Crear directori d'estat i servei procd

```bash
mkdir -p /var/lib/tailscale
```

```bash
cat > /etc/init.d/tailscaled << 'EOF'
#!/bin/sh /etc/rc.common
# Tailscale service for OpenWrt / GL.iNet

START=99
STOP=1
USE_PROCD=1

start_service() {
    mkdir -p /var/lib/tailscale
    procd_open_instance
    procd_set_param command /usr/sbin/tailscaled --state=/var/lib/tailscale/tailscaled.state
    procd_set_param respawn
    procd_close_instance
}
EOF

chmod +x /etc/init.d/tailscaled
```

---

## Pas 6 - Habilitar i arrencar el servei

```bash
/etc/init.d/tailscaled enable
/etc/init.d/tailscaled start

# Verificar que el daemon esta en marxa
ps | grep tailscaled
```

---

## Pas 7 - Autenticar Tailscale

```bash
tailscale up
# Obre el link al navegador i autentifica't com efornaguera@gmail.com
```

```bash
# Verificar
tailscale status
tailscale ip
```

---

## Pas 8 - Configurar DHCP per propagar DNS Synology

Perque tots els clients WiFi del GL.iNet rebin `192.168.1.2` com a DNS
i resolguin `devops.net` automaticament:

```bash
uci set dhcp.@dnsmasq[0].server='192.168.1.2'
uci set dhcp.@dnsmasq[0].noresolv='1'
uci commit dhcp
/etc/init.d/dnsmasq restart
```

> Despres que els clients renovin la IP rebran el DNS del Synology automaticament.

---

## Pas 9 - Verificar DNS des del Windows

```powershell
ipconfig /release
ipconfig /renew
nslookup gitea.devops.net
# Ha de retornar -> 192.168.1.5
```

---

## Resum

| Pas | Tasca                                      | Estat |
|-----|--------------------------------------------|-------|
| 1   | Connexio SSH                               | ✅    |
| 2   | Verificar arquitectura (MIPS interAptiv)   | ✅    |
| 3   | Consultar binaris disponibles              | ✅    |
| 4   | Instal·lar tailscale_1.94.2_mips           | ⏳    |
| 5   | Crear servei procd                         | ⏳    |
| 6   | Habilitar servei a l'inici                 | ⏳    |
| 7   | Autenticar Tailscale                       | ⏳    |
| 8   | Configurar DHCP DNS -> 192.168.1.2         | ⏳    |
| 9   | Verificar DNS des del Windows              | ⏳    |

---

## Problemes detectats

| Problema | Causa | Solucio |
|----------|-------|---------|
| `not found / syntax error` amb v1.68.1_mips | Incompatibilitat ABI hardfloat | Usar v1.94.2_mips |
| `HTTP error 404` amb mips-softfloat | El paquet no existeix | Usar mips estandard |
| Windows no resol devops.net | DNS del PC apunta a ISP, no Synology | Configurar DHCP GL.iNet (Pas 8) |