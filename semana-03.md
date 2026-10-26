# Semana 3: Hardening de SSH y firewall en `gateway`

**Completada:** días 1 a 5

## Día 1: usuario y claves SSH

- El usuario administrativo ya tenía `sudo`, configurado durante la semana 1.
- Generé un par de claves Ed25519 en la laptop y copié la clave pública a `gateway` con `ssh-copy-id`.
- Confirmé desde una sesión nueva que podía conectarme sin ingresar la contraseña de la cuenta.

## Día 2: endurecimiento de SSH

Creé `/etc/ssh/sshd_config.d/00-hardening.conf` con estos ajustes:

```text
PasswordAuthentication no
PermitRootLogin no
PubkeyAuthentication yes
KbdInteractiveAuthentication no
```

Antes de aplicar los cambios, validé la configuración con `sudo sshd -t`. Luego la apliqué con `sudo systemctl reload ssh`, sin cerrar la sesión activa.

Verifiqué desde una segunda terminal que:

- El acceso con clave funcionaba sin pedir la contraseña de la cuenta.
- El intento de acceso usando solo contraseña era rechazado.
- El acceso SSH como `root` era rechazado.


## Día 3: firewall con UFW

Antes de instalar UFW, guardé como referencia las reglas existentes de `iptables` para NAT y reenvío. Luego instalé UFW y retiré `iptables-persistent` y `netfilter-persistent` para evitar mantener dos sistemas de persistencia de reglas a la vez.

Para conservar el acceso SSH y la salida a Internet de `appserver`, ajusté la configuración de UFW:

- Cambié `DEFAULT_FORWARD_POLICY` en `/etc/default/ufw` de `DROP` a `ACCEPT`.
- Añadí en `/etc/ufw/before.rules` la regla `MASQUERADE` y las reglas de reenvío necesarias entre las interfaces de la LAN y la red interna.
- Restringí el acceso SSH a la dirección de la laptop principal, sin publicar esa dirección en este documento.
- Activé UFW y comprobé que podía seguir conectándome por SSH y que `appserver` conservaba la salida a Internet.

Tomé un snapshot con el nombre `ufw-ssh-restringido-ok`.

## Día 4: Fail2ban

Instalé Fail2ban y configuré `/etc/fail2ban/jail.local`, sin modificar directamente `jail.conf`:

```ini
[DEFAULT]
bantime = 600
findtime = 600
maxretry = 3
banaction = ufw

[sshd]
enabled = true
```

Configuré `banaction = ufw` para que Fail2ban aplique los bloqueos mediante el firewall que usa el servidor.

Comprobé el estado general y el de la jaula SSH con `fail2ban-client status` y `fail2ban-client status sshd`.

También probé el baneo desde una segunda laptop para no poner en riesgo el acceso principal. Tras tres intentos fallidos, confirmé que Fail2ban había registrado y bloqueado la dirección de origen. Después de la prueba, eliminé la regla temporal de UFW que había permitido el acceso de esa laptop.

## Día 5: consulta de registros

En esta instalación de Debian, los eventos de SSH y Fail2ban se consultan con `journald`, en vez de buscarse en el archivo de texto `/var/log/auth.log`.

Usé estos comandos para consultar los registros:

```bash
sudo journalctl -u ssh --since "X days ago"
sudo journalctl -u fail2ban --since "X days ago"
```

Repetí la prueba de baneo y confirmé el resultado con `fail2ban-client status sshd`.

Al terminar, eliminé la regla temporal de UFW que había agregado para la segunda laptop.

## Estado de `gateway` al cierre de la semana

- **SSH:** acceso por clave; autenticación por contraseña deshabilitada; acceso SSH de `root` deshabilitado.
- **UFW:** activo; acceso SSH limitado a la laptop principal; NAT y reenvío hacia `appserver` funcionando.
- **Fail2ban:** activo, con la jaula `sshd` habilitada. El baneo se probó y confirmó dos veces.
- **Snapshot:** el más reciente es `ufw-ssh-restringido-ok`, tomado antes de instalar y configurar Fail2ban.

## Conceptos repasados

- **Archivos de configuración locales:** en Fail2ban, usar `jail.local` en vez de editar `jail.conf`; en SSH, usar archivos dentro de `sshd_config.d` en vez de modificar el archivo principal cuando corresponda.
- **UFW:** administra reglas de firewall mediante una interfaz simplificada. Al migrar reglas existentes, revisé por separado las de NAT y reenvío.
- **Journald:** permite consultar los registros de servicios con `journalctl`.
- **Orden de reglas:** el orden importa;

## Pendiente para la semana 4

Instalar Docker Engine en `appserver` y desplegar la aplicacióncon Docker Compose.
