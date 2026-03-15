# Instal·lació Ubuntu amb partició Btrfs — Servidor GEEKOM IT12-DEVOPS

## 1. Preparació del medi d'instal·lació

- Descarga la ISO de Ubuntu Server desde el sitio oficial.
- Crea un USB booteable (Rufus en Windows o `dd` en Linux).
- Inserta el USB en el servidor y arranca desde él (configura la BIOS/UEFI si es necesario).

## 2. Inicio del instalador de Ubuntu

- Selecciona "Instalar Ubuntu Server".
- Elige el idioma y la distribución del teclado.
- Conéctate a la red si es posible (opcional, para descargar actualizaciones).

## 3. Configuración de almacenamiento con Btrfs

- En la sección "Discos y volúmenes", elige "Usar todo el disco".
- Selecciona el disco destinado (`/dev/sda` o similar).
- Marca la opción "Configuración avanzada".
- Escoge el esquema de particionado Btrfs.
- Si quieres subvolúmenes, crea `@` para root, `@home` para /home y `@snapshots`.
- Libre de swap o crea uno como archivo en Btrfs si lo necesitas.

## 4. Instalación del sistema base

- El instalador formatea y crea las estructuras Btrfs.
- Sigue con la instalación normal: zona horaria, usuario y contraseña, nombre del equipo (`geekom-it12-devops`).

## 5. Ajustes post-instalación

- Agrega en `/etc/fstab` si usaste subvolúmenes con las entradas correspondientes.
- Verifica que GRUB apunte al disco correcto.
- Habilita snapshots automáticos con `snapper` o `timeshift`.

## 6. Primer arranque y verificación

- Reinicia y arranca desde disco.
- Comprueba estado de Btrfs con `sudo btrfs filesystem df /` y `sudo btrfs subvolume list /`.
- Crea un snapshot de prueba:

```bash
sudo btrfs subvolume snapshot / /@snapshots/preinstal
```

## 7. Opciones adicionales

- Configurar compresión `zstd` en subvolúmenes (`compress=zstd` en fstab).
- Usar `btrfs scrub` en cron para integridad.
- Establecer rotación de snapshots.

> **Nota:** Si no utilizas subvolúmenes, basta con seleccionar Btrfs en "usar todo el disco" sin personalizar; se creará una sola partición.

Con estos pasos tendrás tu Ubuntu Server instalado en `geekom it12-devops` usando Btrfs en toda la unidad.
