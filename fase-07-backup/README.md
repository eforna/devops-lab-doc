# Fase 07 — Backup y recuperación

> **Objetivo**: Estrategia de backup completa: snapshots BTRFS automáticos locales + backup offsite al NAS Synology. Procedimiento de recuperación documentado y probado.

## Estado

⬜ Pendiente

---

## Estrategia de backup

```
Servidor Ubuntu (BTRFS)
  │
  ├── Snapshots locales (snapper)
  │     └── Retención: 5 hourly, 7 daily, 4 weekly
  │
  └── Backup offsite → NAS Synology
        └── btrfs send/receive o rsync
              └── Retención: 30 días
```

---

## 7.1 Configurar retención automática de snapshots

```bash
# Ver configuración actual de snapper
sudo snapper -c root get-config

# Configurar retención automática
sudo snapper -c root set-config \
  "TIMELINE_CREATE=yes" \
  "TIMELINE_CLEANUP=yes" \
  "TIMELINE_LIMIT_HOURLY=5" \
  "TIMELINE_LIMIT_DAILY=7" \
  "TIMELINE_LIMIT_WEEKLY=4" \
  "TIMELINE_LIMIT_MONTHLY=3" \
  "TIMELINE_LIMIT_YEARLY=0"

# Activar timer de snapper
sudo systemctl enable --now snapper-timeline.timer
sudo systemctl enable --now snapper-cleanup.timer

# Verificar
sudo systemctl status snapper-timeline.timer
```

---

## 7.2 Script de snapshot pre/post servicios

Crear script para snapshots manuales antes de cambios importantes:

```bash
cat > /usr/local/bin/lab-snapshot <<'EOF'
#!/bin/bash
# Uso: lab-snapshot "descripción del cambio"
DESCRIPTION="${1:-manual-snapshot}"
TIMESTAMP=$(date +%Y%m%d-%H%M)

echo "📸 Creando snapshot: ${TIMESTAMP}-${DESCRIPTION}"
sudo snapper -c root create \
  --description "${TIMESTAMP}-${DESCRIPTION}" \
  --cleanup-algorithm number

echo "✅ Snapshot creado:"
sudo snapper -c root list | tail -2
EOF

chmod +x /usr/local/bin/lab-snapshot

# Uso:
lab-snapshot "antes-de-actualizar-jenkins"
```

---

## 7.3 Backup de datos Docker al NAS Synology

### Configurar acceso SSH al NAS

```bash
# Generar clave para backup (sin passphrase para automatizar)
ssh-keygen -t ed25519 -f ~/.ssh/id_backup -N ""

# Copiar clave al NAS Synology
ssh-copy-id -i ~/.ssh/id_backup.pub <usuario_nas>@<IP_NAS>

# Test
ssh -i ~/.ssh/id_backup <usuario_nas>@<IP_NAS> "echo conexión OK"
```

### Script de backup offsite

```bash
cat > /usr/local/bin/lab-backup <<'EOF'
#!/bin/bash
NAS_USER="<usuario_nas>"
NAS_IP="<IP_NAS>"
NAS_PATH="/volume1/backups/devops-lab"
LOCAL_PATH="/opt/lab"
LOG="/var/log/lab-backup.log"
SSH_KEY="$HOME/.ssh/id_backup"

echo "$(date) - Iniciando backup" >> $LOG

# Parar servicios antes del backup (opcional — para consistencia)
# docker compose -f /opt/lab/gitea/docker-compose.yml stop

# Rsync de /opt/lab al NAS
rsync -avz --delete \
  --exclude="*/data/prometheus/" \
  --exclude="*/__pycache__/" \
  -e "ssh -i $SSH_KEY" \
  $LOCAL_PATH/ \
  $NAS_USER@$NAS_IP:$NAS_PATH/

if [ $? -eq 0 ]; then
  echo "$(date) - Backup completado OK" >> $LOG
else
  echo "$(date) - ERROR en backup" >> $LOG
  exit 1
fi
EOF

chmod +x /usr/local/bin/lab-backup
```

---

## 7.4 Automatizar backup con cron

```bash
# Backup diario a las 3:00 AM
echo "0 3 * * * root /usr/local/bin/lab-backup" | sudo tee /etc/cron.d/lab-backup

# Ver logs
tail -f /var/log/lab-backup.log
```

---

## 7.5 Procedimiento de recuperación

### Caso 1: Revertir un cambio reciente (snapper)

```bash
# Ver snapshots disponibles
sudo snapper -c root list

# Comparar cambios entre snapshot N y estado actual
sudo snapper -c root diff N..0

# Revertir cambios (sin reiniciar)
sudo snapper -c root undochange N..0

# O rollback completo (requiere reinicio)
sudo snapper rollback N
sudo reboot
```

### Caso 2: Restaurar datos Docker desde NAS

```bash
# Parar el servicio afectado
docker compose -f /opt/lab/gitea/docker-compose.yml down

# Restaurar desde NAS
rsync -avz \
  -e "ssh -i ~/.ssh/id_backup" \
  <usuario_nas>@<IP_NAS>:/volume1/backups/devops-lab/gitea/ \
  /opt/lab/gitea/

# Reiniciar servicio
docker compose -f /opt/lab/gitea/docker-compose.yml up -d
```

### Caso 3: Recuperación total del servidor

```bash
# 1. Reinstalar Ubuntu con BTRFS
# 2. Instalar snapper y restaurar snapshot más reciente
# 3. O montar NAS y hacer rsync inverso de /opt/lab
# 4. Reinstalar Docker y levantar todos los compose
```

---

## 7.6 Verificar integridad del backup

```bash
# Test mensual: restaurar un servicio de prueba desde backup
# Documentar fecha y resultado aquí:
```

| Fecha | Servicio restaurado | Resultado | Tiempo |
|-------|--------------------|-----------| -------|
| | | | |

---

## 7.7 Snapshot final

```bash
sudo snapper -c root create --description "post-fase-07-backup" --cleanup-algorithm number
sudo snapper -c root list
```

---

## Notas y observaciones

| Fecha | Nota |
|-------|------|
| | |

---

## Checklist de fase completada

- [ ] Retención automática de snapshots configurada
- [ ] Timers snapper activos
- [ ] Script `lab-snapshot` creado y funcional
- [ ] Clave SSH para backup generada y copiada al NAS
- [ ] Script `lab-backup` creado y probado
- [ ] Cron de backup diario configurado
- [ ] Procedimiento de recuperación documentado
- [ ] Primera prueba de restauración realizada
- [ ] Snapshot "post-fase-07" creado

---

**🎉 Laboratorio DevOps completo y documentado.**

Vuelve al [README principal](../README.md) para actualizar el estado de todas las fases.
