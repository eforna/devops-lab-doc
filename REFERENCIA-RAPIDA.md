# Referencia rápida — DevOps Lab

Comandos de uso frecuente para el día a día del laboratorio.

---

## Snapshots BTRFS

```bash
# Crear snapshot manual
lab-snapshot "descripcion-del-cambio"

# Listar snapshots
sudo snapper -c root list

# Ver diferencias entre snapshot N y estado actual
sudo snapper -c root diff N..0

# Revertir cambios
sudo snapper -c root undochange N..0

# Rollback completo (requiere reboot)
sudo snapper rollback N && sudo reboot
```

---

## Docker — comandos globales

```bash
# Estado de todos los contenedores
docker ps -a

# Contenedores en ejecución con uso de recursos
docker stats

# Ver logs de un contenedor
docker logs -f <nombre>

# Reiniciar todos los servicios del lab
for dir in /opt/lab/*/; do
  [ -f "$dir/docker-compose.yml" ] && docker compose -f "$dir/docker-compose.yml" restart
done

# Limpiar imágenes y contenedores no usados
docker system prune -f
```

---

## Servicios individuales

```bash
# Traefik
cd /opt/lab/traefik && docker compose [up -d | down | restart | logs -f]

# Gitea
cd /opt/lab/gitea && docker compose [up -d | down | restart | logs -f]

# Jenkins
cd /opt/lab/jenkins && docker compose [up -d | down | restart | logs -f]

# Portainer
cd /opt/lab/portainer && docker compose [up -d | down | restart | logs -f]

# Keycloak
cd /opt/lab/keycloak && docker compose [up -d | down | restart | logs -f]

# Harbor
cd /opt/lab/harbor && docker compose [up -d | down | restart | logs -f]

# Monitoring (Prometheus + Grafana)
cd /opt/lab/monitoring && docker compose [up -d | down | restart | logs -f]
```

---

## URLs del laboratorio

| Servicio | URL |
|----------|-----|
| Traefik Dashboard | http://traefik.devops.lab |
| Gitea | http://gitea.devops.lab |
| Jenkins | http://jenkins.devops.lab |
| Portainer | http://portainer.devops.lab |
| Keycloak | http://keycloak.devops.lab |
| Harbor | http://harbor.devops.lab |
| Prometheus | http://prometheus.devops.lab |
| Grafana | http://grafana.devops.lab |

---

## Backup

```bash
# Backup manual al NAS
lab-backup

# Ver log de backups
tail -50 /var/log/lab-backup.log
```

---

## Red y conectividad

```bash
# Ver IP del servidor
ip addr show | grep "inet "

# Ver red lab-net de Docker
docker network inspect lab-net

# Test DNS desde Mac
nslookup gitea.devops.lab
```

---

## Git — flujo de trabajo

```bash
# En el Mac, desde VS Code o Fork:
# 1. Editar markdown de la fase correspondiente
# 2. Commit con mensaje descriptivo
# 3. Push a Gitea

# Ejemplo desde terminal Mac:
git add fase-01-infraestructura/README.md
git commit -m "docs: documentar instalación Docker completada"
git push origin main
```
