# Infraestructura AZC - Contexto

## Purpose & Context

Kelsie gestiona la infraestructura de red para un entorno multi-sede en Colombia (Cali). Involucra despliegue y mantenimiento de TP-Link Omada SDN, configuración multi-WAN, y soporte a usuarios finales.

**ISPs activos:**
| ISP | Tipo | Puerto | Notas |
|-----|------|--------|-------|
| Liberty Networks | Static /31 + bloque /30 | WAN1 | IP estática, enlace punto-a-punto RFC 3021 |
| Movistar | DHCP | WAN2 | |
| Claro | DHCP | WAN3/WAN4 | |
| Emcali | Pendiente | — | Por agregar como 4to ISP |

**Sitio de referencia:** ER707-M2 V1.20 corriendo con 4 ISPs (configuración funcional como baseline).

## Estado Actual

### Multi-WAN Router (ER707-M2 V1.0)
- Resuelto: reboot loop causado por OC200 empujando mala configuración al agregar Claro como 3er ISP
- Solución: reset del Omada controller
- Router estable con 3 ISPs (Liberty WAN1 static, Movistar WAN2 DHCP, Claro WAN3/WAN4 DHCP)

### APs - Issue resuelto
- APs "Sala Pacífico" y "Zona C" mostraban Connected pero 0 clientes
- Causa raíz: VLAN config push fallido al switch "Switch 8 PoE Rack 2" (timeout error `switch_-6`)
- Solución: re-aplicar config vía Configuration Result

### Modem Movistar (Askey RTF8115VW)
- Perdidas credenciales admin, ISP se niega a ayudar
- URL correcta: `https://192.168.1.1:8000`
- Credenciales superadmin candidatas identificadas

### Email (azc.com.co)
- Diagnosticado bounce hacia findingtc.com — DNS/MX de Kelsie correcto, fallo en dominio destinatario
- MX: `mx1/mx2.supremebox.com`, DMARC activo

### Firewall ("Keeper")
- Kelsie preguntó sobre configuración de firewall al final de una sesión — retomar si se menciona de nuevo

## On the Horizon

- Agregar **Emcali** como 4to ISP al ER707-M2 V1.0
- **Automatización**: script para consultar WAN status vía Omada REST API v2 → alertas formateadas a Discord (sin middleware como n8n/Make)
- Monitoreo periódico de Omada **Configuration Result** para detectar config pushes fallidos

## Key Learnings & Principios

### Omada Controller como causa raíz
- Cuando un router se comporta inesperadamente (reboot loops, anomalías de config), el OC200 empujando configuraciones es probable culpable — no solo el hardware o la conexión ISP

### "Connected" ≠ Funcional
- AP mostrando "Connected" en Omada solo confirma reachability del management VLAN
- Fallos en data VLAN (ej. config push fallido) pueden silenciosamente prevenir asociación de clientes

### Configuration Result es herramienta crítica
- Omada Configuration Result muestra fallos silenciosos (como switch timeouts) que no generan alertas obvias
- **Revisión periódica y re-aplicar configs fallidas es hábito de mantenimiento recomendado**

### Subnets /31 (Liberty Networks)
- RFC 3021 punto-a-punto: regla par/impar determina host vs gateway
- El bloque /30 acompañante es para usos secundarios (1:1 NAT, virtual IPs), NO para config WAN directa

### Omada Webhooks ↔ Discord
- Payload de webhook nativo de Omada NO es compatible con formato esperado por Discord
- Se requiere middleware o script custom para el puente

### Bloqueo de puertos a nivel ISP
- Algunos puertos pueden estar bloqueados a nivel red del ISP sin importar la config del modem/router

## Approach & Patterns

- Troubleshoot sistemático: descartar causas comunes capa por capa antes de escalar a acciones disruptivas (ej. factory reset)
- Usa sitio de referencia (ER707-M2 V1.20 con 4 ISPs) como baseline de comparación
- Interesado en automatización para reducir overhead de monitoreo manual (API + Discord)

## Tools & Recursos

| Categoría | Herramientas |
|-----------|-------------|
| Routers | ER707-M2 V1.0 (oficina principal), ER707-M2 V1.20 (sitio referencia), ER605 (spare) |
| Controller | OC200 Omada SDN (firmware 6.2.0.x) |
| Switches | TL-SG2008P, TL-SG2210P + switches downstream |
| ISPs | Liberty Networks (static /31 + /30), Movistar (DHCP), Claro (DHCP), Emcali (pendiente) |
| Email | azc.com.co en supremebox.com/supremedns.com, DMARC activo |
| Automatización | Omada REST API v2, Discord webhooks, n8n/Make (candidatos) |
| Documentación | TP-Link Omada API documentation PDF |
