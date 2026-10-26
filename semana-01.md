# Semana 1: Instalación de Proxmox VE y primera VM

**Completada:** 28 de agosto de 2026

## Hardware

- Mini PC GMKtec NucBox
- AMD Ryzen 5 3500U
- 16 GB de RAM (12,6 GB disponibles el resto se reserva para la GPU integrada)
- Disco de 256 GB

## Días 1 y 2: instalación de Proxmox VE

- Instalé Proxmox VE 9.2 en el mini PC.
- Conecté el servidor por Ethernet (`nic0`, controlador `r8169`).
- **Problema:** durante la instalación creo que el cable ethernet quedó mal enchufado y se asignó una IP provisional en vez de obtener una dirección por DHCP. Lo corregí editando `/etc/network/interfaces`, cambiando la configuración de `vmbr0` de `inet static` a `inet dhcp`, y reiniciando el servicio de red.
- La IP final de Proxmox quedó dentro de la LAN doméstica.
- El panel web quedó accesible en `https://<IP-DE-PROXMOX>:8006`.
- **Pendiente:** configurar una reserva DHCP por MAC en el router.

## Red doméstica

Estas tareas se hicieron durante la misma semana, aunque no formaban parte del cronograma original:

- Cambié el nombre y la contraseña de la red Wi-Fi principal.
- Creé una red de invitados aislada..

## Día 3: creación de la VM `gateway`

- Creé la primera máquina virtual en Proxmox:
  - 1 núcleo de CPU
  - 2 GB de RAM
  - Disco de 16 GB, con *discard* activado
  - Interfaz de red conectada a `vmbr0`
- Usé la ISO netinst de Debian 13 para arquitectura AMD64, descargada directamente en el almacenamiento local de Proxmox.

## Día 4: instalación de Debian en la VM

- Instalé Debian sin entorno gráfico.
- Configuré el hostname como `gateway.homelab.local`.
- Usé el particionado guiado con el disco completo y una sola partición.
- Seleccioné únicamente el servidor SSH y las utilidades estándar del sistema.
- Creé un usuario administrativo sin privilegios de root.
- Comprobé que el acceso SSH de root estaba deshabilitado. Para obtener una sesión de root desde la VM, usé `su -`.
- Instalé `sudo` y agregué el usuario administrativo al grupo `sudo`.
- La VM obtuvo una dirección por DHCP.
- Confirmé el acceso SSH desde la red local.
- Instalé y habilité `qemu-guest-agent` dentro de la VM y activé **Use QEMU Guest Agent** en las opciones de la VM en Proxmox. Esto sirve consultar información de la VM desde Proxmox muestra la IP etc etc.

## Día 5: snapshots y prueba de rollback

- Tomé un snapshot de la VM `gateway` en estado limpio, con `sudo` y el agente invitado funcionando. Incluí el estado de la RAM.
- Para probar la recuperación, desinstalé `sudo` intencionalmente con `apt remove --purge sudo`.
- Restauré el snapshot desde **Snapshots > Rollback** y confirmé que `sudo` volvió a funcionar.

## Pendientes

- Configurar reservas DHCP por MAC para Proxmox y la VM `gateway`.
- **Semana 2:** crear `vmbr1` para la red interna, agregar una segunda interfaz a `gateway` y crear una segunda VM para el servidor de aplicaciones.
- **Semana 3:** configurar explícitamente el endurecimiento de SSH, el firewall y Fail2ban.
