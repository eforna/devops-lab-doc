# Primer deploy del repo it12-devops al servidor

**Data:** 16 de març de 2026
**Servidor:** IT12-DEVOPS (192.168.2.5)

---

## Objectiu

Clonar el repo `it12-devops` al servidor i fer el primer deploy per sincronitzar
els fitxers de configuració amb el filesystem.

---

## Incidències i solucions

### 1. Clonació sense sudo

**Problema:** `sudo git clone` falla perquè root no té la clau SSH de GitHub.

**Solució:**
```bash
# Sense sudo — la clau SSH és de l'usuari edu
git clone git@github.com:eforna/it12-devops.git ~/it12-devops
```

Si no existeix clau SSH al servidor:
```bash
ssh-keygen -t ed25519 -C "edu@it12-devops"
cat ~/.ssh/id_ed25519.pub
# Afegir a GitHub → Settings → SSH and GPG keys → New SSH key
ssh -T git@github.com   # verificar
```

---

### 2. CRLF en deploy.sh (fitxers editats des de Windows/VSCode)

**Problema:** `$'\r': command not found` al executar `deploy.sh`.
Els fitxers tenen salts de línia Windows (CRLF) en lloc de Unix (LF).

**Diagnòstic:**
```bash
file ~/it12-devops/deploy.sh
# → Bourne-Again shell script ... with CRLF line terminators
```

**Solució puntual (al servidor):**
```bash
sed -i $'s/\r//' ~/it12-devops/deploy.sh
```
> ⚠️ Cal el `$` davant les cometes simples per interpretar `\r` correctament.

**Solució permanent (al repo):** afegit `.gitattributes` que força LF per a tots els fitxers:
```
* text=auto eol=lf
*.sh text eol=lf
*.yml text eol=lf
```

**Problema addicional:** `git pull` mostra error perquè el `sed -i` marca el fitxer com a modificat:
```bash
git reset --hard HEAD   # descartar canvis locals
rm ~/it12-devops/deploy.sh && git pull   # si reset no funciona
```

---

### 3. rsync --relative creava paths incorrectes

**Problema:** L'opció `--relative` amb un path absolut com a font creava l'estructura completa del path al destí:
```
/opt/devops/home/edu/it12-devops/opt/devops/...   ← incorrecte
```

**Causa:** `rsync --relative /home/edu/it12-devops/opt/devops/ /opt/devops/` preserva el path complet de la font.

**Solució:** Treure `--relative` del rsync del `deploy.sh`. Sense `--relative`, rsync copia el *contingut* de la font al destí directament.

**Neteja:**
```bash
sudo rm -rf /opt/devops/home
```

---

### 4. .env.example exclòs del rsync

**Problema:** `cp /opt/devops/.env.example /opt/devops/.env` falla — el fitxer no existeix al servidor.

**Causa:** `deploy.sh` exclou explícitament `.env.example` del rsync per seguretat.

**Solució:** Copiar des del repo local, no des de `/opt/devops/`:
```bash
cp ~/it12-devops/opt/devops/.env.example /opt/devops/.env
nano /opt/devops/.env
```

---

### 5. Symlinks .env per a cada servei Docker

**Problema:** Docker Compose busca el `.env` al directori on s'executa (p.ex. `/opt/devops/gitea/`), però el `.env` és a `/opt/devops/`.

**Solució:** `deploy.sh` crea automàticament symlinks en cada directori de servei:
```bash
/opt/devops/gitea/.env -> /opt/devops/.env
/opt/devops/traefik/.env -> /opt/devops/.env
# ... etc.
```

Verificar:
```bash
ls -la /opt/devops/gitea/.env
# → lrwxrwxrwx ... /opt/devops/gitea/.env -> /opt/devops/.env
```

---

### 6. Traefik — error "field not found, node: tls"

**Problema:** Traefik arrenca en bucle de restart amb l'error:
```
command traefik error: field not found, node: tls
```

**Causa:** La configuració `tls.certificates` sota `entryPoints.websecure` no és vàlida a la configuració **estàtica** de Traefik v2. Els certificats van a la configuració **dinàmica** (`config/dynamic.yml`).

**Solució:** Treure el bloc `tls` de `traefik.yml`:
```yaml
# INCORRECTE (traefik.yml — config estàtica)
entryPoints:
  websecure:
    address: ":443"
    tls:
      certificates:
        - certFile: "/certs/..."   # ← no vàlid aquí

# CORRECTE
entryPoints:
  websecure:
    address: ":443"
# Els certificats van a config/dynamic.yml (fase 06)
```

---

### 7. Gitea — error d'autenticació PostgreSQL

**Problema:** Gitea no pot connectar a `db-gitea`:
```
pq: password authentication failed for user "gitea"
```

**Causa 1:** PostgreSQL inicialitzat prèviament amb contrasenya diferent.
**Solució:** Reinicialitzar la base de dades:
```bash
docker compose down
sudo rm -rf /mnt/btrfs-data/db-gitea/
sudo mkdir -p /mnt/btrfs-data/db-gitea
docker compose up -d
```

**Causa 2:** Nom incorrecte de la variable d'entorn.
El camp de contrasenya a Gitea és `PASSWD`, no `PASS`:

```yaml
# INCORRECTE
- GITEA__database__PASS=${GITEA_DB_PASSWORD}

# CORRECTE
- GITEA__database__PASSWD=${GITEA_DB_PASSWORD}
```

**Diagnòstic útil:**
```bash
# Verificar que les variables es resolen
docker compose config | grep -i "pass"

# Verificar l'app.ini de Gitea
cat /mnt/btrfs-data/gitea/gitea/conf/app.ini | grep -A10 "\[database\]"
```

---

## Resultat final

```bash
docker ps --format 'table {{.Names}}\t{{.Status}}'
```

| Servei | Estat |
|--------|-------|
| traefik | ✅ Up |
| gitea + db-gitea | ✅ Up |
| jenkins | ✅ Up |
| portainer | ✅ Up |
| grafana | ✅ Up |
| prometheus | ✅ Up |
| keycloak | ✅ Up |
| harbor (9 contenidors) | ✅ Up |

---

## Flux de treball correcte (per al futur)

```bash
# 1. Al Mac: editar → commit → push
git add . && git commit -m "feat: ..." && git push

# 2. Al servidor: actualitzar i desplegar
cd ~/it12-devops && git pull
bash ~/it12-devops/deploy.sh

# 3. Reiniciar servei si cal
cd /opt/devops/<servei> && docker compose up -d
```
