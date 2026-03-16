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
├── fase-00-base/                ← guia: sistema base, SSH, BTRFS
├── fase-01-infraestructura/     ← guia: Docker, xarxa, Traefik
├── fase-02-servicios-core/      ← guia: Gitea, Portainer, Keycloak
├── fase-03-ci-cd/               ← guia: Jenkins
├── fase-04-registry/            ← guia: Harbor
├── fase-05-monitoring/          ← guia: Prometheus + Grafana
├── fase-06-seguridad/           ← guia: Hardening, TLS
└── fase-07-backup/              ← guia: BTRFS snapshots + NAS
```

## Estat del projecte

| Fase | Descripció | Estat |
|------|------------|-------|
| 00 | Sistema base, BTRFS, SSH | ✅ Completat |
| 01 | Docker + Traefik + DNS | ✅ Completat |
| 02 | Gitea, Portainer, Keycloak | ✅ Completat |
| 03 | Jenkins CI/CD | ✅ Completat |
| 04 | Harbor registry | ✅ Completat |
| 05 | Prometheus + Grafana | ✅ Completat |
| 06 | Seguretat — HTTPS/TLS | 🔄 En progres |
| 07 | Backup BTRFS + NAS | ✅ Completat |

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
