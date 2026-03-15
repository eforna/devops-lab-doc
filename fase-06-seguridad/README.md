# Fase 06 — Seguridad y hardening

> **Objetivo**: Firewall configurado, accesos mínimos, certificados TLS internos con mkcert, contraseñas por defecto eliminadas.  
> **Prerrequisito**: Todas las fases anteriores completadas y estables.

## Estado

⬜ Pendiente

---

## Snapshot antes de empezar

```bash
sudo snapper -c root create --description "pre-fase-06-seguridad" --cleanup-algorithm number
```

---

## 6.1 Firewall con UFW

```bash
sudo apt install -y ufw

# Política por defecto
sudo ufw default deny incoming
sudo ufw default allow outgoing

# Permitir SSH (crítico — antes de activar)
sudo ufw allow ssh

# Permitir HTTP/HTTPS (Traefik)
sudo ufw allow 80/tcp
sudo ufw allow 443/tcp

# Activar
sudo ufw enable
sudo ufw status verbose
```

> ⚠️ Asegúrate de que SSH funciona antes de cerrar la sesión actual.

---

## 6.2 Certificados TLS locales con mkcert

Para HTTPS en `*.devops.lab` sin errores de navegador:

```bash
# Instalar mkcert en el Mac
brew install mkcert
mkcert -install   # Instala CA en el keychain del Mac

# Generar certificado wildcard
mkcert "*.devops.lab" devops.lab

# Copiar al servidor
scp _wildcard.devops.lab+1.pem _wildcard.devops.lab+1-key.pem devops-lab:/opt/lab/traefik/certs/
```

### Actualizar Traefik para usar TLS

Editar `/opt/lab/traefik/config/dynamic.yml`:

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

Actualizar routers en cada `docker-compose.yml` para usar `websecure`:

```yaml
labels:
  - "traefik.http.routers.gitea.entrypoints=websecure"
  - "traefik.http.routers.gitea.tls=true"
  # Redirigir HTTP → HTTPS
  - "traefik.http.routers.gitea-http.rule=Host(`gitea.devops.lab`)"
  - "traefik.http.routers.gitea-http.entrypoints=web"
  - "traefik.http.routers.gitea-http.middlewares=redirect-to-https"
```

Middleware de redirección en `dynamic.yml`:

```yaml
http:
  middlewares:
    redirect-to-https:
      redirectScheme:
        scheme: https
        permanent: true
```

---

## 6.3 Cambiar contraseñas por defecto

```bash
# Checklist de contraseñas a cambiar:
# [ ] Keycloak admin
# [ ] Harbor admin
# [ ] Grafana admin
# [ ] Portainer admin
# [ ] Gitea admin
```

> Actualizar también los `docker-compose.yml` y ficheros `.env` correspondientes.

---

## 6.4 Usar ficheros .env para secretos

Ejemplo para cualquier servicio:

```bash
cat > /opt/lab/gitea/.env <<'EOF'
GITEA_ADMIN_USER=admin
GITEA_ADMIN_PASSWORD=<contraseña_segura>
EOF

chmod 600 /opt/lab/gitea/.env
```

En `docker-compose.yml`:
```yaml
env_file:
  - .env
```

> ⚠️ Añadir `**/.env` al `.gitignore` del repositorio.

---

## 6.5 Restricciones Docker

```bash
# Ver contenedores que corren como root
docker ps -q | xargs docker inspect --format '{{.Name}}: User={{.Config.User}}'

# Auditar volúmenes con acceso al socket Docker
docker ps -q | xargs docker inspect --format '{{.Name}}: {{.HostConfig.Binds}}'
```

---

## 6.6 .gitignore del repositorio

```bash
cat > /home/claude/devops-lab/.gitignore <<'EOF'
# Secretos
**/.env
**/*.key
**/*.pem
**/secrets/

# Datos persistentes (no versionar)
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
sudo snapper -c root create --description "post-fase-06-seguridad" --cleanup-algorithm number
```

---

## Notas y observaciones

| Fecha | Nota |
|-------|------|
| | |

---

## Checklist de fase completada

- [ ] UFW configurado y activo
- [ ] SSH funciona tras activar UFW
- [ ] mkcert instalado en Mac, CA en keychain
- [ ] Certificado wildcard generado y copiado al servidor
- [ ] Traefik usando HTTPS en todos los servicios
- [ ] Redirección HTTP → HTTPS activa
- [ ] Todas las contraseñas por defecto cambiadas
- [ ] Secretos movidos a ficheros `.env`
- [ ] `.gitignore` configurado
- [ ] Snapshot "post-fase-06" creado

**Siguiente fase**: [Fase 07 — Backup y recuperación](../fase-07-backup/README.md)
