# Jenkins — CI/CD

> **Objectiu**: Jenkins operatiu, integrat amb Gitea mitjançant webhooks i capaç d'executar pipelines sobre contenidors Docker.
> **Prerequisit**: [Gitea](../gitea/README.md) funcionant.

## Estat

✅ Completat

---

## Snapshot abans de començar

```bash
sudo /opt/devops/snapshots/snapshot.sh  # snapshot pre-jenkins
```

---

## 3.1 Desplegar Jenkins

```bash
mkdir -p /opt/devops/jenkins
sudo chown -R 1000:1000 /opt/devops/jenkins

cat > /opt/devops/jenkins/docker-compose.yml <<'EOF'
services:
  jenkins:
    image: jenkins/jenkins:lts
    container_name: jenkins
    restart: unless-stopped
    user: root                      # Per accedir al socket Docker
    environment:
      - JAVA_OPTS=-Djenkins.install.runSetupWizard=false
    volumes:
      - /mnt/btrfs-data/jenkins:/var/jenkins_home
      - /var/run/docker.sock:/var/run/docker.sock
      - /usr/bin/docker:/usr/bin/docker
    networks:
      - devops-net
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.jenkins.rule=Host(`jenkins.devops.lab`)"
      - "traefik.http.routers.jenkins.entrypoints=web"
      - "traefik.http.services.jenkins.loadbalancer.server.port=8080"

networks:
  devops-net:
    external: true
EOF
```

```bash
cd /opt/devops/jenkins
docker compose up -d

# Obtenir contrasenya inicial d'administrador
docker exec jenkins cat /var/jenkins_home/secrets/initialAdminPassword
```

**Verificar**: `http://jenkins.devops.lab` → fer servir la contrasenya obtinguda.

---

## 3.2 Configuració inicial Jenkins

A la UI de Jenkins:

1. Instal·lar plugins suggerits
2. Crear usuari administrador
3. Instal·lar plugins addicionals:
   - **Gitea** (integració nativa)
   - **Docker Pipeline**
   - **Blue Ocean** (UI moderna, opcional)

```
Manage Jenkins → Plugins → Available plugins
→ Cercar: "Gitea", "Docker Pipeline"
→ Install without restart
```

---

## 3.3 Connectar Jenkins amb Gitea

### A Gitea: crear token d'accés

```
Gitea → Settings → Applications → Generate Token
→ Nom: jenkins-token
→ Permisos: repo (read/write), webhooks
→ Copiar el token generat
```

### A Jenkins: afegir credencial

```
Manage Jenkins → Credentials → Global → Add Credentials
→ Kind: Username with password
→ Username: <el_teu_usuari_gitea>
→ Password: <token_generat>
→ ID: gitea-credentials
```

### A Jenkins: configurar servidor Gitea

```
Manage Jenkins → System → Gitea Servers → Add
→ Name: devops-lab-gitea
→ Server URL: http://gitea.devops.lab
→ Credentials: gitea-credentials
```

---

## 3.4 Primer pipeline de prova

Crear repositori de prova a Gitea amb un `Jenkinsfile`:

```groovy
// Jenkinsfile d'exemple
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
                sh 'echo "Tests passats"'
            }
        }
        stage('Deploy') {
            steps {
                sh 'echo "Deploy simulat"'
            }
        }
    }

    post {
        always {
            echo "Pipeline finalitzat: ${currentBuild.result}"
        }
    }
}
```

### Crear job a Jenkins

```
New Item → Pipeline
→ Pipeline script from SCM
→ SCM: Git
→ Repository URL: http://gitea.devops.lab/<usuari>/test-pipeline.git
→ Credentials: gitea-credentials
→ Branch: */main
```

---

## 3.5 Configurar webhook Gitea → Jenkins

A Gitea, al repositori:

```
Settings → Webhooks → Add Webhook → Gitea
→ Target URL: http://jenkins.devops.lab/gitea-webhook/post
→ Events: Push events, Pull request events
→ Active: ✓
```

Test del webhook → ha de retornar HTTP 200.

---

## 3.6 Snapshot final

```bash
sudo /opt/devops/snapshots/snapshot.sh  # snapshot post-jenkins
```

---

## Notes i observacions

| Data | Nota |
|------|------|
| | |

---

## Checklist completat

- [x] Jenkins desplegat i accessible
- [x] Plugins instal·lats (Gitea, Docker Pipeline)
- [x] Token Gitea creat i credencial afegida a Jenkins
- [x] Servidor Gitea configurat a Jenkins
- [x] Primer pipeline executat amb èxit
- [x] Webhook Gitea → Jenkins funcionant
- [x] Snapshot "post-jenkins" creat

**Vegeu també**: [Harbor](../harbor/README.md) (registry per a les imatges)
