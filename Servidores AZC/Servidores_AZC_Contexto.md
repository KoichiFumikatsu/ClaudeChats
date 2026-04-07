# Servidores AZC - Contexto

## Purpose & Context

Kelsie gestiona la infraestructura de servidores on-premises para AZC Legal / Grupo AZC.

**Dominios:**
- `mi.azclegal.com` - Nextcloud
- `sip.grupoazc.com` - Grandstream UCM (VoIP/SIP)
- `go.azclegal.com` - Grandstream GDMS cloud
- `grupoazc.com` - Dominio corporativo

**Servicios principales:**
- **Nextcloud**: ~308 cuentas de usuario, ~1TB de datos
- **Grandstream UCM**: Servidor de voz/SIP
- **Hestia Control Panel**: Panel de administración del servidor Linux Ubuntu

## Estado Actual

### Migración de red (servidor Linux → Omada controller)
- Hestia accesible via IP pública en puerto 8083; SSH en puerto 22
- NAT configurado en Omada controller; DNS forwarding actualizado en registrar (propagación pendiente)
- Puertos 80/443 forwarding y certificados SSL/Let's Encrypt requieren verificación post-migración
- Nextcloud: revisar `trusted_domains`, `overwrite.cli.url`, `overwritehost`, `overwriteprotocol` en config.php

### UCM Voice Server (sip.grupoazc.com → 161.10.226.94)
- Puerto 8089 (UCM web panel/WebSocket) no responde externamente
- `go.azclegal.com` funciona porque rutea a través de GDMS cloud (no el servidor local)
- Pendiente: reboot del servicio UCM para aplicar cambios HTTP Server, luego port forwarding para 8089 y 8090 en modem Movistar y router neutro

## Key Learnings & Principios

### CRITICO: Usuario non-standard en Nextcloud
- Nextcloud corre como `nube:nube`, **NO** como `www-data`
- Todos los comandos occ: `sudo -u nube php occ`
- **NUNCA** aplicar `chown -R www-data` — esto rompió el servidor y tuvo que ser revertido

### Rutas importantes
- Config Nextcloud: `/home/nube/web/mi.azclegal.com/public_html/config/config.php`
- Trashbin: `files_trashbin/files/` (archivos con sufijo `.d{timestamp}`)

### Red: Double NAT
- Modem Movistar + router neutro en double NAT
- Port forwarding debe configurarse en **ambos** dispositivos para acceso externo

### Diagnóstico
- Secuencia: interno → externo, DNS → puertos → config de aplicación
- Verificar que el servicio escuche en el puerto correcto internamente antes de troubleshoot forwarding externo

### Recuperación de archivos (trashbin)
- Mover archivos de vuelta desde trashbin y rescanear
- Restauración de versiones via bash script con `find` y `while read` loops (manejo de nombres con espacios)

## Tools & Recursos

| Categoría | Herramientas |
|-----------|-------------|
| Servidor | Linux Ubuntu, Hestia Control Panel, Nextcloud, nginx/apache |
| Red | Omada controller, modem Movistar, router neutro (double NAT) |
| Voz | Grandstream UCM, GDMS cloud, Wave (WebRTC) |
| Comandos clave | `sudo -u nube php occ`, `find`, `curl`, `nslookup`, `traceroute`, `Test-NetConnection` |
