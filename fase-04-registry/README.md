# Fase 04 — Container Registry: Harbor

> **Objetivo**: Harbor operativo como registry privado de imágenes Docker, integrado con Jenkins para push/pull automático.  
> **Prerrequisito**: [Fase 03](../fase-03-ci-cd/README.md) completada.

## Estado

⬜ Pendiente

---

## Snapshot antes de empezar

```bash
sudo snapper -c root create --description "pre-fase-04-harbor" --cleanup-algorithm number
```

---

## 4.1 Descargar Harbor installer

Harbor requiere su propio instalador (no es una imagen Docker simple):

```bash
# Descargar la última versión
cd /opt/lab
HARBOR_VERSION=$(curl -s https://api.github.com/repos/goharbor/harbor/releases/latest | jq -r .tag_name)
echo "Versión: $HARBOR_VERSION"

curl -LO "https://github.com/goharbor/harbor/releases/download/${HARBOR_VERSION}/harbor-online-installer-${HARBOR_VERSION}.tgz"
tar xvf harbor-online-installer-*.tgz
cd harbor
```

---

## 4.2 Configurar harbor.yml

```bash
cp harbor.yml.tmpl harbor.yml
```

Editar `harbor.yml`:

```yaml
# Cambiar estas líneas:
hostname: harbor.devops.lab

# Comentar la sección https (usamos HTTP en lab)
# https:
#   port: 443
#   ...

harbor_admin_password: changeme_lab   # ← cambiar

# Directorio de datos
data_volume: /opt/lab/harbor/data
```

```bash
mkdir -p /opt/lab/harbor/data
```

---

## 4.3 Instalar Harbor

```bash
cd /opt/lab/harbor
sudo ./install.sh

# Harbor levanta sus propios contenedores con docker compose
# Ver estado
docker compose ps
```

---

## 4.4 Exponer Harbor con Traefik

Harbor usa su propio `docker-compose.yml`. Añadir labels de Traefik al servicio `nginx` de Harbor:

```bash
# Editar docker-compose.yml generado por Harbor
# Buscar el servicio "nginx" y añadir:
```

```yaml
# En el servicio nginx de Harbor:
labels:
  - "traefik.enable=true"
  - "traefik.http.routers.harbor.rule=Host(`harbor.devops.lab`)"
  - "traefik.http.routers.harbor.entrypoints=web"
  - "traefik.http.services.harbor.loadbalancer.server.port=8080"
networks:
  - lab-net
  - harbor_harbor   # red interna de harbor
```

```bash
# Asegurarse que lab-net es externa en docker-compose.yml de Harbor
# Añadir al final del fichero:
# networks:
#   lab-net:
#     external: true

docker compose down && docker compose up -d
```

**Verificar**: `http://harbor.devops.lab` → login con `admin` / `changeme_lab`.

---

## 4.5 Configurar Docker para usar Harbor

Harbor sin HTTPS requiere marcarlo como "insecure registry":

```bash
sudo tee -a /etc/docker/daemon.json <<'EOF'
{
  "insecure-registries": ["harbor.devops.lab"]
}
EOF

sudo systemctl restart docker
```

**Probar login**:
```bash
docker login harbor.devops.lab
# Usuario: admin
# Password: changeme_lab
```

---

## 4.6 Crear proyecto en Harbor

En la UI de Harbor:
```
Projects → New Project
→ Name: devops-lab
→ Access Level: Private
```

---

## 4.7 Push de primera imagen de prueba

```bash
# Etiquetar imagen local para Harbor
docker pull alpine
docker tag alpine harbor.devops.lab/devops-lab/alpine:test
docker push harbor.devops.lab/devops-lab/alpine:test

# Verificar en la UI de Harbor
```

---

## 4.8 Integrar Harbor en Jenkins

```
Manage Jenkins → Credentials → Add
→ Kind: Username with password
→ Username: admin
→ Password: <harbor_password>
→ ID: harbor-credentials
```

Ejemplo en Jenkinsfile:
```groovy
stage('Push to Harbor') {
    steps {
        withCredentials([usernamePassword(
            credentialsId: 'harbor-credentials',
            usernameVariable: 'HARBOR_USER',
            passwordVariable: 'HARBOR_PASS'
        )]) {
            sh '''
                docker login harbor.devops.lab -u $HARBOR_USER -p $HARBOR_PASS
                docker build -t harbor.devops.lab/devops-lab/myapp:${BUILD_NUMBER} .
                docker push harbor.devops.lab/devops-lab/myapp:${BUILD_NUMBER}
            '''
        }
    }
}
```

---

## 4.9 Snapshot final

```bash
sudo snapper -c root create --description "post-fase-04-harbor" --cleanup-algorithm number
```

---

## Notas y observaciones

| Fecha | Nota |
|-------|------|
| | |

---

## Checklist de fase completada

- [ ] Harbor descargado e instalado
- [ ] `harbor.yml` configurado con dominio y contraseña
- [ ] Harbor accesible vía Traefik en `harbor.devops.lab`
- [ ] Docker configurado con insecure-registry
- [ ] Login Docker a Harbor funcionando
- [ ] Proyecto `devops-lab` creado en Harbor
- [ ] Primera imagen pusheada correctamente
- [ ] Credenciales Harbor en Jenkins
- [ ] Snapshot "post-fase-04" creado

**Siguiente fase**: [Fase 05 — Monitorización](../fase-05-monitoring/README.md)
