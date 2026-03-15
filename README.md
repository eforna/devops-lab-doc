# 🧪 DevOps Lab — Geekom i7

Laboratorio DevOps personal para experimentación, aprendizaje y práctica con herramientas de infraestructura moderna.

## Hardware

| Componente | Detalle |
|------------|---------|
| Máquina | Geekom IT12 |
| CPU | Intel Core i7 |
| RAM | 32 GB |
| OS | Ubuntu Server (BTRFS) |
| Red | LAN local — dominio `devops.lab` |
| DNS | NAS Synology (DNS primario) |

## Arquitectura de servicios

```
*.devops.lab
     │
     ▼
  Traefik (reverse proxy + SSL)
     │
     ├── gitea.devops.lab        → Gitea (Git server)
     ├── jenkins.devops.lab      → Jenkins (CI/CD)
     ├── harbor.devops.lab       → Harbor (Container Registry)
     ├── keycloak.devops.lab     → Keycloak (SSO / IAM)
     ├── portainer.devops.lab    → Portainer (Docker UI)
     ├── grafana.devops.lab      → Grafana (Dashboards)
     └── prometheus.devops.lab   → Prometheus (Métricas)
```

## Estructura del repositorio

```
devops-lab/
├── README.md                       ← Este fichero
├── fase-00-base/                   ← Sistema base, SSH, BTRFS, snapshots
├── fase-01-infraestructura/        ← Docker, redes, Traefik
├── fase-02-servicios-core/         ← Gitea, Portainer, Keycloak
├── fase-03-ci-cd/                  ← Jenkins
├── fase-04-registry/               ← Harbor
├── fase-05-monitoring/             ← Prometheus + Grafana
├── fase-06-seguridad/              ← Hardening, firewall, certificados
└── fase-07-backup/                 ← Estrategia BTRFS + offsite NAS
```

## Estado del proyecto

| Fase | Descripción | Estado |
|------|-------------|--------|
| 00 | Sistema base | ⬜ Pendiente |
| 01 | Infraestructura Docker + Traefik | ⬜ Pendiente |
| 02 | Servicios core (Gitea, Portainer, Keycloak) | ⬜ Pendiente |
| 03 | CI/CD con Jenkins | ⬜ Pendiente |
| 04 | Registry con Harbor | ⬜ Pendiente |
| 05 | Monitorización | ⬜ Pendiente |
| 06 | Seguridad y hardening | ⬜ Pendiente |
| 07 | Backup y recuperación | ⬜ Pendiente |

> Actualiza el estado con: ✅ Completado · 🔄 En progreso · ⚠️ Con incidencias · ⬜ Pendiente

## Entorno de trabajo

- **Workstation**: macOS
- **Editor**: Visual Studio Code
- **Git GUI**: Fork / SourceTree
- **Acceso servidor**: SSH

## Convenciones

- Cada fase tiene su propio `README.md` con contexto, comandos y notas
- Los comandos se ejecutan directamente en el servidor y se documentan después
- Los snapshots BTRFS se crean **antes** de cada fase como punto de control
- Las variables de entorno sensibles van en ficheros `.env` (nunca en Git)

## Variables globales del laboratorio

```bash
LAB_DOMAIN="devops.lab"
LAB_IP="<IP_DEL_SERVIDOR>"       # Rellenar tras instalación
LAB_USER="<TU_USUARIO>"          # Usuario no-root del servidor
```
