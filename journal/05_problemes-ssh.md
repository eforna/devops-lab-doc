# Problemes SSH

## Solució ràpida

**1. Accedeix físicament o per consola al servidor.**

**2. Verifica i arrenca el servei SSH:**

```bash
sudo systemctl status ssh
sudo systemctl start ssh
sudo systemctl enable ssh
```

**3. Comprova que el port 22 està escoltant:**

```bash
sudo ss -tlnp | grep :22
```

**4. Si uses firewall (UFW), permet SSH:**

```bash
sudo ufw allow ssh
sudo ufw reload
```

---

## Canviar o obrir ports

```bash
# Permetre port 22 i 2222 per UFW
sudo ufw allow 22/tcp
sudo ufw allow 2222/tcp

# Redirigir port 22 al 2222 amb iptables
sudo iptables -t nat -A PREROUTING -p tcp --dport 22 -j REDIRECT --to-port 2222
```
