# Semana 2: Segmentación de red, NAT y acceso SSH

## Objetivos

- Crear una red interna aislada en Proxmox.
- Conectar las máquinas virtuales `gateway` y `appserver`.
- Darle salida a Internet a `appserver` a través de NAT en `gateway`.
- Configurar un acceso SSH a `appserver` mediante `ProxyJump`.
- Documentar la topología de red.

## Configuración de Proxmox

- Configuré dos bridges:
  - `vmbr0`: conectado a la interfaz Ethernet física y a la LAN doméstica.
  - `vmbr1`: bridge interno sin conexión a una interfaz física.
- La red conectada a `vmbr1` es privada y aislada de la LAN doméstica.
- Configuré reservas DHCP por MAC para el host Proxmox y la interfaz de `gateway` conectada a `vmbr0`. Las direcciones se omiten por seguridad.

## VM `gateway`

La VM tiene dos interfaces de red:

- `ens18`, conectada a `vmbr0`, para comunicarse con la LAN y acceder al router.
- `ens19`, conectada a `vmbr1`, con una dirección estática que funciona como gateway de la red interna.

Configuré el reenvío de paquetes IPv4 y reglas de `iptables` para que el tráfico de la red interna pueda salir a Internet a través de `gateway`.

La regla NAT usa `MASQUERADE`. La elegí porque la interfaz de salida está configurada mediante DHCP, aunque el router tenga una reserva para que la dirección se mantenga estable.

Instalé `iptables-persistent` y guardé las reglas para que se restauren después de reiniciar.

## VM `appserver`

- Creé la VM con 2 núcleos, 3 GB de RAM y un disco de 20 GB.
- Instalé Debian 13 con una configuración mínima.
- Durante la instalación, conecté temporalmente la VM a `vmbr0` para descargar los paquetes necesarios. Después la conecté a `vmbr1`.
- La VM tiene una sola interfaz de red, conectada a `vmbr1`.
- Configuré una dirección estática, la puerta de enlace apuntando a `gateway` y servidores DNS.
- Comprobé que tenía salida a Internet tanto por dirección IP como por nombre de dominio.

## Acceso SSH mediante `ProxyJump`

Configuré `~/.ssh/config` en la laptop para poder conectarme a `appserver` pasando por `gateway`:

```sshconfig
Host gateway
    HostName <IP-DE-GATEWAY-EN-LA-LAN>
    User <USUARIO-ADMINISTRATIVO>

Host appserver
    HostName <IP-INTERNA-DE-APPSERVER>
    User <USUARIO-DE-APPSERVER>
    ProxyJump gateway
```

Con esta configuración, el comando `ssh appserver` conecta primero con `gateway` y luego usa esa conexión para llegar a `appserver`.

## Topología de red

```text
Internet
   |
Router doméstico
   |
LAN doméstica
   |
Laptop ----------------------> gateway
   |                         acceso SSH directo
   |
   +---- SSH a appserver ----> gateway ----> appserver
                               ProxyJump
                                  |
                           +------+------+
                           |             |
                      interfaz LAN   interfaz interna
                           |             |
                        vmbr0         vmbr1
                                          |
                                      appserver
```

- `vmbr0` conecta `gateway` con la LAN doméstica.
- `vmbr1` conecta `gateway` y `appserver` en una red interna aislada.
- `appserver` no tiene una conexión física directa a la LAN.
- La salida a Internet de `appserver` pasa por el NAT configurado en `gateway`.

## Recorrido de un paquete desde `appserver` hacia Internet

1. `appserver` envía el paquete por su interfaz conectada a `vmbr1`.
2. `gateway` lo recibe por su interfaz de la red interna.
3. El reenvío IPv4 permite que el kernel pase el tráfico a la interfaz conectada a `vmbr0`.
4. La regla `MASQUERADE` cambia la dirección de origen por la dirección de salida de `gateway`.
5. El paquete sale hacia el router y, desde allí, a Internet.
6. La respuesta vuelve por el camino inverso.

## Conceptos repasados

- **Bridge:** conecta interfaces a nivel de enlace de datos. `vmbr0` conecta con la LAN; `vmbr1` proporciona conectividad entre las VMs de la red interna.
- **Reenvío IP:** permite que el sistema operativo reenvíe paquetes entre interfaces.
- **NAT:** modifica las direcciones de los paquetes para que los equipos de una red privada puedan compartir la salida a Internet de otro equipo.
- **`iptables`:** herramienta para administrar reglas de filtrado y traducción de paquetes.
- **`MASQUERADE` y `SNAT`:** `MASQUERADE` usa la dirección actual de la interfaz de salida; `SNAT` especifica una dirección fija en la regla.
- **`ProxyJump`:** permite usar un servidor SSH como salto para conectarse a otro equipo. Se configura en el cliente, en este caso, la laptop.

## Pendientes

- **Semana 3:** endurecer SSH, configurar el firewall y Fail2ban en `gateway`.
- **Semana 4:** instalar Docker y Docker Compose en `appserver` y desplegar una aplicación.
