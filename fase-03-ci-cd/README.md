# Fase 03 — CI/CD: Jenkins

> **Objetivo**: Jenkins operativo, integrado con Gitea mediante webhooks y capaz de ejecutar pipelines sobre contenedores Docker.  
> **Prerrequisito**: [Fase 02](../fase-02-servicios-core/README.md) completada. Gitea funcionando.

## Estado

⬜ Pendiente

---

## Snapshot antes de empezar

```bash
sudo snapper -c root create --description "pre-fase-03-jenkins" --cleanup-algorithm number
```

---

## 3.1 Desplegar Jenkins

```bash
mkdir -p /opt/lab/jenkins/data
sudo chown -R 1000:1000 /opt/lab/jenkins/data

cat > /opt/lab/jenkins/docker-compose.yml <<'EOF'
services:
  jenkins:
    image: jenkins/jenkins:lts
    container_name: jenkins
    restart: unless-stopped
    user: root                      # Para acceder al socket Docker
    environment:
      - JAVA_OPTS=-Djenkins.install.runSetupWizard=false
    volumes:
      - ./data:/var/jenkins_home
      - /var/run/docker.sock:/var/run/docker.sock
      - /usr/bin/docker:/usr/bin/docker
    networks:
      - lab-net
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.jenkins.rule=Host(`jenkins.devops.lab`)"
      - "traefik.http.routers.jenkins.entrypoints=web"
      - "traefik.http.services.jenkins.loadbalancer.server.port=8080"

networks:
  lab-net:
    external: true
EOF
```

```bash
cd /opt/lab/jenkins
docker compose up -d

# Obtener contraseña inicial de administrador
docker exec jenkins cat /var/jenkins_home/secrets/initialAdminPassword
```

**Verificar**: `http://jenkins.devops.lab` → usar la contraseña obtenida.

---

## 3.2 Configuración inicial Jenkins

En la UI de Jenkins:

1. Instalar plugins sugeridos
2. Crear usuario administrador
3. Instalar plugins adicionales:
   - **Gitea** (integración nativa)
   - **Docker Pipeline**
   - **Blue Ocean** (UI moderna, opcional)

```
Manage Jenkins → Plugins → Available plugins
→ Buscar: "Gitea", "Docker Pipeline"
→ Install without restart
```

---

## 3.3 Conectar Jenkins con Gitea

### En Gitea: crear token de acceso

```
Gitea → Settings → Applications → Generate Token
→ Nombre: jenkins-token
→ Permisos: repo (read/write), webhooks
→ Copiar el token generado
```

### En Jenkins: añadir credencial

```
Manage Jenkins → Credentials → Global → Add Credentials
→ Kind: Username with password
→ Username: <tu_usuario_gitea>
→ Password: <token_generado>
→ ID: gitea-credentials
```

### En Jenkins: configurar servidor Gitea

```
Manage Jenkins → System → Gitea Servers → Add
→ Name: devops-lab-gitea
→ Server URL: http://gitea.devops.lab
→ Credentials: gitea-credentials
```

---

## 3.4 Primer pipeline de prueba

Crear repositorio de prueba en Gitea con un `Jenkinsfile`:

```groovy
// Jenkinsfile de ejemplo
pipeline {
    agent any

    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }
        stage('Build') {
            steps {
                sh 'echo "Build OK - $(date)"'
            }
        }
        stage('Test') {
            steps {
                sh 'echo "Tests pasados"'
            }
        }
        stage('Deploy') {
            steps {
                sh 'echo "Deploy simulado"'
            }
        }
    }

    post {
        always {
            echo "Pipeline finalizado: ${currentBuild.result}"
        }
    }
}
```

### Crear job en Jenkins

```
New Item → Pipeline
→ Pipeline script from SCM
→ SCM: Git
→ Repository URL: http://gitea.devops.lab/<usuario>/test-pipeline.git
→ Credentials: gitea-credentials
→ Branch: */main
```

---

## 3.5 Configurar webhook Gitea → Jenkins

En Gitea, en el repositorio:

```
Settings → Webhooks → Add Webhook → Gitea
→ Target URL: http://jenkins.devops.lab/gitea-webhook/post
→ Events: Push events, Pull request events
→ Active: ✓
```

Test del webhook → debe devolver HTTP 200.

---

## 3.6 Snapshot final

```bash
sudo snapper -c root create --description "post-fase-03-jenkins" --cleanup-algorithm number
```

---

## Notas y observaciones

| Fecha | Nota |
|-------|------|
| | |

---

## Checklist de fase completada

- [ ] Jenkins desplegado y accesible
- [ ] Plugins instalados (Gitea, Docker Pipeline)
- [ ] Token Gitea creado y credencial añadida en Jenkins
- [ ] Servidor Gitea configurado en Jenkins
- [ ] Primer pipeline ejecutado con éxito
- [ ] Webhook Gitea → Jenkins funcionando
- [ ] Snapshot "post-fase-03" creado

**Siguiente fase**: [Fase 04 — Registry con Harbor](../fase-04-registry/README.md)
