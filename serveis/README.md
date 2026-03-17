# Serveis

Documentació de cada servei del laboratori DevOps.

## Serveis desplegats

| Servei | URL | Descripció | Estat |
|--------|-----|------------|-------|
| [Gitea](gitea/README.md) | http://gitea.devops.lab | Servidor Git | ✅ |
| [Portainer](portainer/README.md) | http://portainer.devops.lab | UI de Docker | ✅ |
| [Keycloak](keycloak/README.md) | http://keycloak.devops.lab | SSO / Identity Provider | ✅ |
| [Jenkins](jenkins/README.md) | http://jenkins.devops.lab | CI/CD | ✅ |
| [Harbor](harbor/README.md) | http://harbor.devops.lab | Container Registry | ✅ |
| [Grafana](grafana/README.md) | http://grafana.devops.lab | Dashboards de monitorització | ✅ |
| [Prometheus](prometheus/README.md) | http://prometheus.devops.lab | Recollida de mètriques | ✅ |

## Arquitectura

```
*.devops.lab  →  Traefik (reverse proxy)
                   │
                   ├── gitea.devops.lab      → Gitea :3000
                   ├── portainer.devops.lab  → Portainer :9000
                   ├── keycloak.devops.lab   → Keycloak :8080
                   ├── jenkins.devops.lab    → Jenkins :8080
                   ├── harbor.devops.lab     → Harbor :8888
                   ├── grafana.devops.lab    → Grafana :3000
                   └── prometheus.devops.lab → Prometheus :9090
```

Tots els serveis estan a la xarxa Docker `devops-net`.

---

Torna al [README principal](../README.md).
