# Grafana — Dashboards de monitorització

> **Objectiu**: Dashboards visuals amb mètriques del servidor, Docker i tots els serveis del lab.
> **Prerequisit**: [Prometheus](../prometheus/README.md) operatiu.

## Estat

✅ Completat

---

## Docker Compose

Grafana s'afegeix al `docker-compose.yml` de `/opt/devops/grafana/` juntament amb Prometheus:

```yaml
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
```

```bash
mkdir -p /opt/devops/grafana/grafana/data
sudo chown -R 472:472 /opt/devops/grafana/grafana/data  # UID de Grafana

cd /opt/devops/grafana
docker compose up -d
```

**Verificar**: `http://grafana.devops.lab` → login `admin` / `changeme_lab`.

---

## Configurar Grafana

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

## Notes i observacions

| Data | Nota |
|------|------|
| | |

---

## Checklist completat

- [x] Grafana desplegat i accessible
- [x] Datasource Prometheus configurat a Grafana
- [x] Dashboard Node Exporter importat
- [x] Dashboard Docker/cAdvisor importat
- [x] Snapshot "post-monitoring" creat
