# Fase 05 — Monitorización: Prometheus + Grafana

> **Objetivo**: Stack de observabilidad completo: métricas del servidor, Docker y todos los servicios del lab visibles en dashboards Grafana.  
> **Prerrequisito**: [Fase 04](../fase-04-registry/README.md) completada.

## Estado

⬜ Pendiente

---

## Snapshot antes de empezar

```bash
sudo snapper -c root create --description "pre-fase-05-monitoring" --cleanup-algorithm number
```

---

## Arquitectura de monitorización

```
Servidor Ubuntu
  ├── node_exporter      → métricas sistema (CPU, RAM, disco)
  └── cAdvisor           → métricas Docker/contenedores
         ↓
      Prometheus         → recoge y almacena métricas
         ↓
       Grafana           → dashboards y alertas
```

---

## 5.1 Estructura de ficheros

```bash
mkdir -p /opt/lab/monitoring/{prometheus,grafana}
mkdir -p /opt/lab/monitoring/prometheus/data
mkdir -p /opt/lab/monitoring/grafana/data
sudo chown -R 472:472 /opt/lab/monitoring/grafana/data  # UID de Grafana
```

---

## 5.2 Configurar Prometheus

```bash
cat > /opt/lab/monitoring/prometheus/prometheus.yml <<'EOF'
global:
  scrape_interval: 15s
  evaluation_interval: 15s

scrape_configs:
  - job_name: 'prometheus'
    static_configs:
      - targets: ['localhost:9090']

  - job_name: 'node-exporter'
    static_configs:
      - targets: ['node-exporter:9100']

  - job_name: 'cadvisor'
    static_configs:
      - targets: ['cadvisor:8080']

  - job_name: 'traefik'
    static_configs:
      - targets: ['traefik:8080']
    metrics_path: /metrics
EOF
```

---

## 5.3 Docker Compose del stack

```bash
cat > /opt/lab/monitoring/docker-compose.yml <<'EOF'
services:

  prometheus:
    image: prom/prometheus:latest
    container_name: prometheus
    restart: unless-stopped
    volumes:
      - ./prometheus/prometheus.yml:/etc/prometheus/prometheus.yml:ro
      - ./prometheus/data:/prometheus
    command:
      - '--config.file=/etc/prometheus/prometheus.yml'
      - '--storage.tsdb.path=/prometheus'
      - '--storage.tsdb.retention.time=30d'
      - '--web.enable-lifecycle'
    networks:
      - lab-net
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
      - ./grafana/data:/var/lib/grafana
    networks:
      - lab-net
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
      - lab-net

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
      - lab-net

networks:
  lab-net:
    external: true
EOF
```

```bash
cd /opt/lab/monitoring
docker compose up -d
docker compose ps
```

---

## 5.4 Verificar métricas en Prometheus

Abrir `http://prometheus.devops.lab`:

```
Status → Targets → verificar que todos están UP
```

Consultas de prueba en el explorador:
```
up                                    # Ver todos los targets
node_cpu_seconds_total                # CPU del servidor
container_memory_usage_bytes          # RAM por contenedor
```

---

## 5.5 Configurar Grafana

Abrir `http://grafana.devops.lab` → login `admin` / `changeme_lab`

### Añadir datasource Prometheus

```
Configuration → Data Sources → Add → Prometheus
→ URL: http://prometheus:9090
→ Save & Test
```

### Importar dashboards oficiales

```
Dashboards → Import → ID del dashboard
```

| Dashboard | ID | Descripción |
|-----------|----|-------------|
| Node Exporter Full | `1860` | Métricas completas del servidor |
| Docker / cAdvisor | `893` | Métricas de contenedores |
| Traefik | `17346` | Métricas del reverse proxy |

---

## 5.6 Habilitar métricas en Traefik

Editar `/opt/lab/traefik/traefik.yml` y añadir:

```yaml
metrics:
  prometheus:
    addEntryPointsLabels: true
    addRoutersLabels: true
    addServicesLabels: true
```

```bash
docker compose -f /opt/lab/traefik/docker-compose.yml restart
```

---

## 5.7 Snapshot final

```bash
sudo snapper -c root create --description "post-fase-05-monitoring" --cleanup-algorithm number
```

---

## Notas y observaciones

| Fecha | Nota |
|-------|------|
| | |

---

## Checklist de fase completada

- [ ] Prometheus desplegado y accesible
- [ ] node-exporter corriendo (métricas sistema)
- [ ] cAdvisor corriendo (métricas Docker)
- [ ] Todos los targets en Prometheus: estado UP
- [ ] Grafana desplegado y accesible
- [ ] Datasource Prometheus configurado en Grafana
- [ ] Dashboard Node Exporter importado
- [ ] Dashboard Docker/cAdvisor importado
- [ ] Métricas Traefik habilitadas
- [ ] Snapshot "post-fase-05" creado

**Siguiente fase**: [Fase 06 — Seguridad y hardening](../fase-06-seguridad/README.md)
