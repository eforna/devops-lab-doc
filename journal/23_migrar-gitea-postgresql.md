# Migració de Gitea de SQLite a PostgreSQL

## Objectiu
Migrar Gitea per utilitzar PostgreSQL com a base de dades, en lloc de SQLite. Aquesta guia explica cada pas i el raonament de cada canvi.

---

## Pas 1: Preparar el container PostgreSQL

### 1.1. Crear el servei db-gitea
Afegeix aquest bloc al docker-compose:

```yaml
services:
  db-gitea:
    image: postgres:15
    container_name: db-gitea
    restart: always
    environment:
      POSTGRES_DB: gitea
      POSTGRES_USER: gitea
      POSTGRES_PASSWORD: canviasegura
    volumes:
      - /mnt/btrfs-data/db-gitea:/var/lib/postgresql/data
    networks:
      - devops-net

networks:
  devops-net:
    external: true
```

### 1.2. Arrencar el container

```bash
docker compose up -d db-gitea
```

---

## Pas 2: Modificar la configuració de Gitea

### 2.1. Variables d'entorn
Canvia el bloc d'entorn de Gitea:

```yaml
      - GITEA__database__DB_TYPE=postgres
      - GITEA__database__HOST=db-gitea:5432
      - GITEA__database__USER=gitea
      - GITEA__database__PASS=canviasegura
      - GITEA__database__NAME=gitea
```

### 2.2. Eliminar referències a SQLite
Elimina o comenta qualsevol línia amb `sqlite3`.

---

## Pas 3: Arrencar Gitea i completar la configuració

```bash
cd /opt/devops/gitea
sudo docker compose up -d
```

Accedeix a `http://gitea.devops.lab` i completa la configuració inicial. Ara la bbdd serà PostgreSQL.

---

## Pas 4: Recomanacions de backup

- Per fer backup de Gitea:
  - Backup del directori `/mnt/btrfs-data/gitea`
  - Backup de la bbdd PostgreSQL:

```bash
docker exec db-gitea pg_dump -U gitea gitea > /mnt/nas-backup/it12-devops/gitea-postgres-backup.sql
```

- Per restaurar:

```bash
docker exec -i db-gitea psql -U gitea gitea < /mnt/nas-backup/it12-devops/gitea-postgres-backup.sql
```

---

## Avantatges del canvi
- Millor rendiment i escalabilitat
- Backups més fiables
- Evita problemes de concurrència

---

## Notes
- Si no hi ha dades, no cal migrar res: simplement arrenca Gitea amb PostgreSQL.
- Si tens dades a SQLite, consulta la documentació de Gitea per migrar-les.

---

**Document creat el 14/03/2026 per migració controlada de Gitea a PostgreSQL.**