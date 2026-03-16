# Projecte DEVOPS — Visió global

Laboratori DevOps personal amb 2 servidors: un OKD per desplegar aplicacions i un servidor DEVOPS per eines de CI/CD, monitoring i gestió.

## Infraestructura disponible

| Equip | CPU | RAM | SSD | Hostname | IP |
|-------|-----|-----|-----|----------|----|
| GEEKOM IT12 I7 | Intel i7 | 32 GB | 1 TB | IT12-OKD | 192.168.2.4 |
| GEEKOM IT12 I7 | Intel i7 | 32 GB | 1 TB | IT12-DEVOPS | 192.168.2.5 |
| NAS Synology DS220J | — | — | 2 TB | synology | 192.168.2.2 |

## Entorn de desenvolupament

- MacBook Air M3 24 GB / 512 GB SSD
- Portàtil Intel i5 8 GB / 1 TB SSD (Windows 10)

## Software

- **IT12-OKD:** OKD Baremetal en Fedora.
- **IT12-DEVOPS:** Ubuntu Server, BTRFS, Docker Compose, Portainer, Gitea,
  Traefik, Jenkins, Grafana, Prometheus, Harbor, Keycloak.

---

## Arquitectura general IT12-DEVOPS

```
Internet/LAN
     │
     ▼
┌─────────────┐
│   TRAEFIK   │  <- Proxy invers (porta 80/443)
│  (router)   │     Redirigeix tràfic a cada servei
└──────┼──────┘
       │ xarxa: devops-net
       │
▼        ▼        ▼        ▼        ▼        ▼
Portainer  Gitea  Jenkins  Grafana  Keycloak  Harbor
```

### Per què cada eina?

| Eina | Funció |
|------|--------|
| **Traefik** | Proxy invers. Sense Traefik: `192.168.2.5:8080`. Amb Traefik: `jenkins.devops.lab` |
| **Portainer** | Gestió visual de Docker via navegador |
| **Gitea** | Repositori Git propi. Jenkins llegeix d'aquí per fer els builds |
| **Jenkins** | CI/CD: automatitza build, test i deploy |
| **Grafana + Prometheus** | Prometheus recull mètriques; Grafana les mostra en dashboards |
| **Keycloak** | Autenticació centralitzada (SSO) |
| **Harbor** | Registry de contenidors privat |

## Flux de treball complet

```
Developer (MacBook/Portàtil)
     │
     │ git push
     ▼
┌─────────┐     webhook      ┌─────────┐
│  Gitea  │ ──────────────→  │ Jenkins │
│  (codi) │                  │ (CI/CD) │
└─────────┘                  └────┼────┘
                                  │ build & push
                                  ▼
                            ┌─────────┐
                            │ Harbor  │
                            └────┼────┘
                                 │ pull image
                                 ▼
                          ┌────────────┐
                          │  IT12-OKD  │
                          └────────────┘
```

## Per què BTRFS amb subvolums?

```
/dev/nvme0n1p2 (BTRFS 1TB)
├── @docker      → Dades Docker engine
├── @portainer   → Dades Portainer
├── @gitea       → Repositoris Git
├── @jenkins     → Jobs i configuració
├── @grafana     → Dashboards
├── @prometheus  → Mètriques
├── @harbor      → Imatges Docker
└── @keycloak    → Usuaris i config
```

**Avantatges:** Snapshots per servei independent, rollback si algo falla, quotes d'espai per servei, esborrat net sense afectar altres serveis.

## Ordre d'instal·lació dels serveis

| Ordre | Servei | Motiu |
|-------|--------|-------|
| 1r | Traefik | Necessari per accedir als altres |
| 2n | Portainer | Per gestionar Docker visualment |
| 3r | Gitea | Per tenir el codi |
| 4t | Jenkins | Necessita Gitea |
| 5è | Prometheus | Recull dades |
| 6è | Grafana | Necessita Prometheus |
| 7è | Keycloak | SSO per a tots els serveis |
| 8è | Harbor | Registry d'imatges |

> Per al detall pas a pas de cada instal·lació, consulta [annex-instalacio-serveis.md](annex-instalacio-serveis.md).
