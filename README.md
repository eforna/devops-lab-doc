# DevOps Lab — Geekom IT12

Laboratori DevOps personal per a experimentació, aprenentatge i pràctica amb eines d'infraestructura moderna.

## Hardware

| Component | Detall |
|-----------|--------|
| Màquina | Geekom IT12 |
| CPU | Intel Core i7 |
| RAM | 32 GB |
| SSD | 1 TB (NVMe) |
| OS | Ubuntu Server 24.04 LTS (BTRFS) |
| IP | 192.168.2.5 |
| Hostname | it12-devops |
| Usuari | edu |

## Arquitectura de serveis

```
*.devops.lab  (DNS wildcard → 192.168.2.5, NAS Synology 192.168.2.2)
     │
     ▼
  Traefik (reverse proxy, port 80/443)   xarxa: devops-net
     │
     ├── gitea.devops.lab        → Gitea + PostgreSQL
     ├── jenkins.devops.lab      → Jenkins (CI/CD)
     ├── harbor.devops.lab:8888  → Harbor (Container Registry)
     ├── keycloak.devops.lab     → Keycloak (SSO / IAM)
     ├── portainer.devops.lab    → Portainer (Docker UI)
     ├── grafana.devops.lab      → Grafana (Dashboards)
     └── prometheus.devops.lab   → Prometheus (Mètriques)
```

## Repositoris del projecte

| Repo | Contingut |
|------|-----------|
| [devops-lab-doc](https://github.com/eforna/devops-lab-doc) | Aquest repo — documentació, guies i journal |
| [it12-devops](https://github.com/eforna/it12-devops) | Fitxers de configuració del servidor (mirror filesystem) |

## Estructura d'aquest repo

```
devops-lab-doc/
├── README.md
├── REFERENCIA-RAPIDA.md
├── docs/
│   ├── arquitectura.md          ← visió global del projecte
│   └── ubuntu/                  ← referència filesystem Ubuntu
├── journal/                     ← notes reals de cada sessió de treball
│   ├── 01_copia-seguretat-usb.md
│   ├── 02_installacio-ubuntu-server.md
│   └── ...
├── infraestructura/             ← posada en marxa del servidor
│   ├── base/                    ← Ubuntu Server, BTRFS, SSH
│   ├── docker-traefik/          ← Docker Engine, xarxa, Traefik
│   └── backup/                  ← Snapshots BTRFS + NAS Synology
├── serveis/                     ← documentació per servei
│   ├── gitea/
│   ├── portainer/
│   ├── keycloak/
│   ├── jenkins/
│   ├── harbor/
│   ├── grafana/
│   └── prometheus/
└── seguretat/                   ← hardening, TLS, contrasenyes
```

## Estat del projecte

| Àrea | Descripció | Estat |
|------|------------|-------|
| [infraestructura/base](infraestructura/base/README.md) | Ubuntu Server, BTRFS, SSH | ✅ Completat |
| [infraestructura/docker-traefik](infraestructura/docker-traefik/README.md) | Docker + Traefik + DNS | ✅ Completat |
| [infraestructura/backup](infraestructura/backup/README.md) | BTRFS snapshots + NAS | ✅ Completat |
| [serveis/gitea](serveis/gitea/README.md) | Servidor Git | ✅ Completat |
| [serveis/portainer](serveis/portainer/README.md) | Docker UI | ✅ Completat |
| [serveis/keycloak](serveis/keycloak/README.md) | SSO / IAM | ✅ Completat |
| [serveis/jenkins](serveis/jenkins/README.md) | CI/CD | ✅ Completat |
| [serveis/harbor](serveis/harbor/README.md) | Container Registry | ✅ Completat |
| [serveis/grafana](serveis/grafana/README.md) | Dashboards | ✅ Completat |
| [serveis/prometheus](serveis/prometheus/README.md) | Mètriques | ✅ Completat |
| [seguretat](seguretat/README.md) | Hardening, HTTPS/TLS | 🔄 En progrés |

> ✅ Completat · 🔄 En progrés · ⚠️ Amb incidències · ⬜ Pendent

## Entorn de treball

- **Workstation**: MacBook Air M3
- **Editor**: Visual Studio Code
- **Git GUI**: Fork / SourceTree
- **Accés servidor**: SSH (`ssh edu@192.168.2.5 -p 2222`)

## Convencions

- Cada fase té el seu `README.md` amb context, comandos i notes
- El `journal/` recull les notes reals de cada sessió (incidències, canvis, decisions)
- Els snapshots BTRFS es creen **abans** de cada canvi important al servidor
- Les credencials van al `.env` del servidor — mai al repo
