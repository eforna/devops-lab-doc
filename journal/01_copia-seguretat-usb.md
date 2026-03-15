# Còpia de seguretat IT12 amb USB d'arrancada

## Usar otro PC Windows

Si tienes acceso a cualquier PC Windows:

- Instala Macrium Reflect Free.
- Conecta el pendrive.
- Crea el Rescue Media.

## Descarregar Macrium Reflect Home (30 dies de Free)

<https://www.macrium.com/reflectfree?utm_source=copilot.com> — descarregar versió de 64 bits Reflect Home

Descarregat a: `NAS/soft/reflect_home_setup_x64[CUND-9P2D].exe`

## Problemes amb els drivers de xarxa de GEEKOM per al USB bootable

Els drivers es poden descarregar des d'aquí:  
<https://service.geekompc.com/es/it12/>

## Per arrancar els serveis de xarxa

```cmd
net start workstation
```

Provar ping després.

## Per connectar unitat de xarxa a SMB

```cmd
net use z: \\192.168.1.2\backup
```
