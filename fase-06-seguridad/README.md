# Fase 06 — Seguretat i hardening

> **Objectiu**: Firewall configurat, accessos mínims, certificats TLS interns amb mkcert, contrasenyes per defecte eliminades.
> **Prerequisit**: Totes les fases anteriors completades i estables.

## Estat

🔄 En progrés

---

## Snapshot abans de començar

```bash
sudo /opt/devops/snapshots/snapshot.sh pre-fase-06-seguritat
```

---

## 6.1 Firewall amb UFW

```bash
sudo apt install -y ufw

# Política per defecte
sudo ufw default deny incoming
sudo ufw default allow outgoing

# Permetre SSH (crític — abans d'activar)
sudo ufw allow 2222/tcp

# Permetre HTTP/HTTPS (Traefik)
sudo ufw allow 80/tcp
sudo ufw allow 443/tcp

# Activar
sudo ufw enable
sudo ufw status verbose
```

> Assegura't que SSH funciona abans de tancar la sessió actual.

---

## 6.2 Certificats TLS locals amb mkcert

Per a HTTPS a `*.devops.lab` sense errors de navegador:

```bash
# Instal·lar mkcert al Mac
brew install mkcert
mkcert -install   # Instal·la CA al keychain del Mac

# Generar certificat wildcard
mkcert "*.devops.lab" devops.lab

# Copiar al servidor
scp _wildcard.devops.lab+1.pem _wildcard.devops.lab+1-key.pem devops-lab:/opt/devops/traefik/certs/
```

### Actualitzar Traefik per usar TLS

Editar `/opt/devops/traefik/config/dynamic.yml`:

```yaml
tls:
  certificates:
    - certFile: /certs/_wildcard.devops.lab+1.pem
      keyFile: /certs/_wildcard.devops.lab+1-key.pem
  stores:
    default:
      defaultCertificate:
        certFile: /certs/_wildcard.devops.lab+1.pem
        keyFile: /certs/_wildcard.devops.lab+1-key.pem
```

Actualitzar routers a cada `docker-compose.yml` per usar `websecure`:

```yaml
labels:
  - "traefik.http.routers.gitea.entrypoints=websecure"
  - "traefik.http.routers.gitea.tls=true"
  # Redirigir HTTP → HTTPS
  - "traefik.http.routers.gitea-http.rule=Host(`gitea.devops.lab`)"
  - "traefik.http.routers.gitea-http.entrypoints=web"
  - "traefik.http.routers.gitea-http.middlewares=redirect-to-https"
```

Middleware de redirecció a `dynamic.yml`:

```yaml
http:
  middlewares:
    redirect-to-https:
      redirectScheme:
        scheme: https
        permanent: true
```

---

## 6.3 Canviar contrasenyes per defecte

```bash
# Checklist de contrasenyes a canviar:
# [ ] Keycloak admin
# [ ] Harbor admin
# [ ] Grafana admin
# [ ] Portainer admin
# [ ] Gitea admin
```

> Actualitzar també els `docker-compose.yml` i fitxers `.env` corresponents.

---

## 6.4 Usar fitxers .env per a secrets

Exemple per a qualsevol servei:

```bash
cat > /opt/devops/gitea/.env <<'EOF'
GITEA_ADMIN_USER=admin
GITEA_ADMIN_PASSWORD=<contrasenya_segura>
EOF

chmod 600 /opt/devops/gitea/.env
```

En `docker-compose.yml`:
```yaml
env_file:
  - .env
```

> Afegir `**/.env` al `.gitignore` del repositori.

---

## 6.5 Restriccions Docker

```bash
# Veure contenidors que s'executen com a root
docker ps -q | xargs docker inspect --format '{{.Name}}: User={{.Config.User}}'

# Auditar volums amb accés al socket Docker
docker ps -q | xargs docker inspect --format '{{.Name}}: {{.HostConfig.Binds}}'
```

---

## 6.6 .gitignore del repositori

```bash
cat > /opt/devops/.gitignore <<'EOF'
# Secrets
**/.env
**/*.key
**/*.pem
**/secrets/

# Dades persistents (no versionar)
**/data/
**/certs/

# OS
.DS_Store
Thumbs.db

# Editor
.vscode/
*.swp
EOF
```

---

## 6.7 Snapshot final

```bash
sudo /opt/devops/snapshots/snapshot.sh post-fase-06-seguritat
```

---

## Notes i observacions

| Data | Nota |
|------|------|
| | |

---

## Checklist de fase completada

- [ ] UFW configurat i actiu
- [ ] SSH funciona després d'activar UFW (port 2222)
- [ ] mkcert instal·lat al Mac, CA al keychain
- [ ] Certificat wildcard generat i copiat al servidor
- [ ] Traefik usant HTTPS a tots els serveis
- [ ] Redirecció HTTP → HTTPS activa
- [ ] Totes les contrasenyes per defecte canviades
- [ ] Secrets moguts a fitxers `.env`
- [ ] `.gitignore` configurat
- [ ] Snapshot "post-fase-06" creat

**Fase següent**: [Fase 07 — Backup i recuperació](../fase-07-backup/README.md)
