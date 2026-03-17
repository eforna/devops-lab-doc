# Prometheus — Recollida de mètriques

> **Objectiu**: Prometheus recull mètriques del servidor (node_exporter), contenidors Docker (cAdvisor) i Traefik.
> **Prerequisit**: [Docker + Traefik](../../infraestructura/docker-traefik/README.md) operatiu.

## Estat

✅ Completat

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

Veure [Grafana](../grafana/README.md) per a la configuració de dashboards.

---

## Snapshot abans de començar

```bash
sudo /opt/devops/snapshots/snapshot.sh pre-monitoring
```

---

## Estructura de fitxers

```bash
mkdir -p /opt/devops/grafana/prometheus/data
sudo chown -R 65534:65534 /opt/devops/grafana/prometheus/data  # UID de Prometheus
```

---

## Configurar Prometheus

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

## Docker Compose

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

> ℹ️ Grafana s'afegeix al mateix `docker-compose.yml`. Veure [Grafana](../grafana/README.md).

```bash
cd /opt/devops/grafana
docker compose up -d
docker compose ps
```

---

## Verificar mètriques

Obrir `http://prometheus.devops.lab`:

```
Status → Targets → verificar que tots estan UP
```

Consultes de prova:
```
up                                    # Veure tots els targets
node_cpu_seconds_total                # CPU del servidor
container_memory_usage_bytes          # RAM per contenidor
```

---

## Habilitar mètriques a Traefik

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

## Notes i observacions

| Data | Nota |
|------|------|
| | |

---

## Checklist completat

- [x] Prometheus desplegat i accessible
- [x] node-exporter en execució (mètriques sistema)
- [x] cAdvisor en execució (mètriques Docker)
- [x] Tots els targets a Prometheus: estat UP
- [x] Mètriques Traefik habilitades
