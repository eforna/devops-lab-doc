# Fase 05 — Monitorització: Prometheus + Grafana

> **Objectiu**: Stack d'observabilitat complet: mètriques del servidor, Docker i tots els serveis del lab visibles en dashboards Grafana.
> **Prerequisit**: [Fase 04](../fase-04-registry/README.md) completada.

## Estat

✅ Completat

---

## Snapshot abans de començar

```bash
sudo /opt/devops/snapshots/snapshot.sh pre-fase-05-monitoring
```

---

## Arquitectura de monitorització

```
Servidor Ubuntu
  ├── node_exporter      → mètriques sistema (CPU, RAM, disc)
  └── cAdvisor           → mètriques Docker/contenidors
         ↓
      Prometheus         → recull i emmagatzema mètriques
         ↓
       Grafana           → dashboards i alertes
```

---

## 5.1 Estructura de fitxers

```bash
mkdir -p /opt/devops/grafana/{prometheus,grafana}
mkdir -p /opt/devops/grafana/prometheus/data
mkdir -p /opt/devops/grafana/grafana/data
sudo chown -R 472:472 /opt/devops/grafana/grafana/data  # UID de Grafana
```

---

## 5.2 Configurar Prometheus

```bash
cat > /opt/devops/grafana/prometheus/prometheus.yml <<'EOF'
global:
  scrape_interval: 15s
  evaluation_interval: 15s

scrape_configs:
  - job_name: 'prometheus'
    static_configs:
      - targets: ['localhost:9090']

  - job_name: 'node-exporter'
    static_configs:
      - targets: ['192.168.2.5:9100']

  - job_name: 'cadvisor'
    static_configs:
      - targets: ['192.168.2.5:9323']

  - job_name: 'traefik'
    static_configs:
      - targets: ['traefik:8080']
    metrics_path: /metrics
EOF
```

---

## 5.3 Docker Compose de l'stack

```bash
cat > /opt/devops/grafana/docker-compose.yml <<'EOF'
services:

  prometheus:
    image: prom/prometheus:latest
    container_name: prometheus
    restart: unless-stopped
    volumes:
      - ./prometheus/prometheus.yml:/etc/prometheus/prometheus.yml:ro
      - /mnt/btrfs-data/prometheus:/prometheus
    command:
      - '--config.file=/etc/prometheus/prometheus.yml'
      - '--storage.tsdb.path=/prometheus'
      - '--storage.tsdb.retention.time=30d'
      - '--web.enable-lifecycle'
    networks:
      - devops-net
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.prometheus.rule=Host(`prometheus.devops.lab`)"
      - "traefik.http.routers.prometheus.entrypoints=web"
      - "traefik.http.services.prometheus.loadbalancer.server.port=9090"

  grafana:
    image: grafana/grafana:latest
    container_name: grafana
    restart: unless-stopped
    environment:
      - GF_SECURITY_ADMIN_USER=admin
      - GF_SECURITY_ADMIN_PASSWORD=changeme_lab
      - GF_SERVER_ROOT_URL=http://grafana.devops.lab
    volumes:
      - /mnt/btrfs-data/grafana:/var/lib/grafana
    networks:
      - devops-net
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.grafana.rule=Host(`grafana.devops.lab`)"
      - "traefik.http.routers.grafana.entrypoints=web"
      - "traefik.http.services.grafana.loadbalancer.server.port=3000"

  node-exporter:
    image: prom/node-exporter:latest
    container_name: node-exporter
    restart: unless-stopped
    pid: host
    volumes:
      - /proc:/host/proc:ro
      - /sys:/host/sys:ro
      - /:/rootfs:ro
    command:
      - '--path.procfs=/host/proc'
      - '--path.sysfs=/host/sys'
      - '--path.rootfs=/rootfs'
      - '--collector.filesystem.mount-points-exclude=^/(sys|proc|dev|host|etc)($$|/)'
    networks:
      - devops-net

  cadvisor:
    image: gcr.io/cadvisor/cadvisor:latest
    container_name: cadvisor
    restart: unless-stopped
    privileged: true
    volumes:
      - /:/rootfs:ro
      - /var/run:/var/run:ro
      - /sys:/sys:ro
      - /var/lib/docker:/var/lib/docker:ro
      - /dev/disk/:/dev/disk:ro
    devices:
      - /dev/kmsg
    networks:
      - devops-net

networks:
  devops-net:
    external: true
EOF
```

```bash
cd /opt/devops/grafana
docker compose up -d
docker compose ps
```

---

## 5.4 Verificar mètriques a Prometheus

Obrir `http://prometheus.devops.lab`:

```
Status → Targets → verificar que tots estan UP
```

Consultes de prova a l'explorador:
```
up                                    # Veure tots els targets
node_cpu_seconds_total                # CPU del servidor
container_memory_usage_bytes          # RAM per contenidor
```

---

## 5.5 Configurar Grafana

Obrir `http://grafana.devops.lab` → login `admin` / `changeme_lab`

### Afegir datasource Prometheus

```
Configuration → Data Sources → Add → Prometheus
→ URL: http://prometheus:9090
→ Save & Test
```

### Importar dashboards oficials

```
Dashboards → Import → ID del dashboard
```

| Dashboard | ID | Descripció |
|-----------|----|------------|
| Node Exporter Full | `1860` | Mètriques completes del servidor |
| Docker / cAdvisor | `893` | Mètriques de contenidors |
| Traefik | `17346` | Mètriques del reverse proxy |

---

## 5.6 Habilitar mètriques a Traefik

Editar `/opt/devops/traefik/traefik.yml` i afegir:

```yaml
metrics:
  prometheus:
    addEntryPointsLabels: true
    addRoutersLabels: true
    addServicesLabels: true
```

```bash
docker compose -f /opt/devops/traefik/docker-compose.yml restart
```

---

## 5.7 Snapshot final

```bash
sudo /opt/devops/snapshots/snapshot.sh post-fase-05-monitoring
```

---

## Notes i observacions

| Data | Nota |
|------|------|
| | |

---

## Checklist de fase completada

- [x] Prometheus desplegat i accessible
- [x] node-exporter en execució (mètriques sistema)
- [x] cAdvisor en execució (mètriques Docker)
- [x] Tots els targets a Prometheus: estat UP
- [x] Grafana desplegat i accessible
- [x] Datasource Prometheus configurat a Grafana
- [x] Dashboard Node Exporter importat
- [x] Dashboard Docker/cAdvisor importat
- [x] Mètriques Traefik habilitades
- [x] Snapshot "post-fase-05" creat

**Fase següent**: [Fase 06 — Seguretat i hardening](../fase-06-seguridad/README.md)
