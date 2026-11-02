# Semana 7: VPN interna en VM1

## Objetivo

Configurar WireGuard en la VM `gateway` para poder llegar a la VM
privada `appserver` desde dispositivos autorizados (celular y
notebook) sin estar en la misma LAN. Solo accede quien tenga una
clave autorizada.

## Trabajo realizado

### Día 1: Instalación de WireGuard

- Instalación del paquete `wireguard` en `gateway` con `apt`.
- Generación del par de claves del servidor con `wg genkey` y
  `wg pubkey`, en `/etc/wireguard/`, con permisos `600` (solo root).

### Día 2: Configuración de la interfaz wg0

- `wg0` es una interfaz de red virtual en `gateway`: la entrada del
  túnel. Se define en `/etc/wireguard/wg0.conf`.
- Se eligió una subred propia para la VPN, distinta de la LAN, de la
  red interna y de las redes de Docker, para evitar solapamientos.
- La clave privada creada el dia anterior se inserta en `wg0.conf`
- Firewall (ufw), dos reglas:
  - Entrada al puerto UDP 51820 (WireGuard). Puede quedar abierto
    porque ignora todo lo que no esté firmado con una clave
    autorizada.
  - `ufw route` desde `wg0` hacia la red interna. Hace falta porque
    la política de forwarding está en `DROP`: todo lo que atraviesa
    `gateway` debe estar permitido de forma explícita.
- Servicio habilitado con `systemctl enable --now wg-quick@wg0`.
- Verificación: `wg show` muestra la interfaz escuchando 
`ip a show wg0` muestra la dirección asignada.

### Día 3: Configuración de los clientes

- En mi caso los clientes VPN son mi celu y mi laptop
- Entre dispositivos solo se intercambian
  las claves públicas.
- En el servidor, cada cliente se autoriza con un bloque `[Peer]` en
  `wg0.conf`, con su clave pública y su IP dentro del túnel
  (`AllowedIPs` con máscara /32). Ahí funciona como lista de permisos:
  solo se aceptan paquetes con esa IP de origen.
- En el cliente, el bloque `[Peer]` describe al servidor: su clave
  pública, el `Endpoint` (dirección y puerto UDP), `AllowedIPs` y
  `PersistentKeepalive = 25`.
- `AllowedIPs` en el cliente decide qué tráfico va por el túnel. Solo
  se envían la red de la VPN y la red interna, no todo internet, así
  la navegación normal no cambia.
- Notebook (Linux): paquete `wireguard`, claves con `wg genkey` y
  `wg pubkey`, configuración en `/etc/wireguard/` y túnel con
  `wg-quick up`.
- Celular: app oficial de WireGuard. Para esta prueba, el `Endpoint`
  apunta a la IP del servidor en la LAN, porque todavía no está
  configurado el acceso desde internet.
- Verificación:
  - `wg show` muestra `latest handshake` en el peer: cliente y
    servidor se reconocieron con sus claves.
  - Ping al servidor dentro del túnel con TTL 64 (conexión directa).
  - Ping a `appserver` con TTL 63: el paquete pasó por `gateway`, lo
    que confirma el reenvío.
  - `curl` a la app por la red interna: respuesta HTTP 200.
  - Prueba de contraste: sin el túnel no hay ruta a la red interna.
- Estado: túnel funcionando en notebook y celular, probado desde la
  LAN. Falta probar desde fuera de casa (Día 4).

### Problemas que tuve

Escribí la clave a mano y confundí una l con una I..... 
Con `tcpdump` en el servidor mostró los
paquetes del celular llegando cada pocos segundos, sin respuesta.
WireGuard descarta lo que no puede validar, habia puesto mal la clave publica del celu. 
**Moraleja:** las claves conviene transferirlas por copiar y pegar.

### Día 4: omitido

Iba a brir el puerto apra porbar desde fuera, pero decidí no abrir ningún puerto para no tocar el router. La VPN se probó
solo desde dentro de la LAN y ,por lo anterior, el acceso desde datos móviles no se
validó.

### Día 5: Documentación del túnel

**Cómo funciona**

- `gateway` tiene una interfaz virtual, `wg0`, que es la entrada del
  túnel. Cada dispositivo autorizado (el celu, la notebook) tiene su
  propio par de claves.
- La clave privada nunca sale del dispositivo. Entre dispositivos
  solo se intercambian claves públicas.
- Un peer está autorizado solo si su clave pública figura en
  `wg0.conf`. WireGuard no responde a paquetes que no puede validar
  con una clave conocida.
- `AllowedIPs` cumple dos funciones. En el servidor, define de qué IP
  puede ser cada peer (filtro de origen). En el cliente, define qué
  tráfico va por el túnel.
- El tráfico de los clientes hacia la red interna atraviesa
  `gateway`: entra por `wg0`, sale por la interfaz interna, y el
  firewall lo permite con reglas `ufw route` explícitas porque la
  política de forwarding es `DROP`.