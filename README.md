# Homelab con Proxmox

Este repositorio documenta mi servidor casero. La idea surgió mientras
hacía un curso de ciberseguridad.

**Hardware:** mini PC GMKtec (NucBox) con Ryzen 5 3500U, 16 GB de RAM
(12,6 GB utilizables) y 256 GB de disco.

## Cómo llegué acá

Al principio pensaba instalar Debian directamente sobre la máquina,
aplicar hardening y, más adelante, abrir algún puerto para exponer
servicios a internet, o al menos dentro de mi LAN. El profesor del
curso me recomendó no hacerlo así: si algo se rompía, iba a perder
todas las configuraciones. Me sugirió virtualizar con Proxmox.

## Qué es Proxmox

Proxmox VE es un hipervisor de tipo 1: se instala directamente sobre
el hardware y permite crear y gestionar máquinas virtuales y
contenedores. Tiene una consola web en el puerto 8006. Su ventaja
principal para aprender es que los snapshots permiten romper cosas y
volver atrás en segundos.

## Arquitectura

Dentro de Proxmox hay dos máquinas virtuales con Debian 13:

- **gateway:** tiene una interfaz en la LAN y otra en una red interna
  aislada. Ejecuta Nginx como proxy inverso, el firewall (ufw),
  fail2ban y el NAT.
- **appserver:** vive solo en la red interna (10.10.10.0/24) y
  contiene las aplicaciones web en contenedores Docker.

Me conecto a gateway, y Nginx redirige el tráfico hacia la aplicación
correspondiente.

## Tecnologías

Proxmox VE, Debian 13, Docker y Docker Compose, Nginx, ufw, fail2ban,
Spring Boot, MySQL.

Lo que sigue -->  WireGuard, Loki, Grafana y Cloudflare
Tunnel.

## Índice por semana

| Semana | Tema | Documento |
|---|---|---|
| 1 | Proxmox y primera VM | [semana-01.md](semana-01.md) |
| 2 | Segmentación de red, NAT y SSH jump host | [semana-02.md](semana-02.md) |
| 3 | Hardening: SSH por claves, ufw y fail2ban | [semana-03.md](semana-03.md) |
| 4 | Docker y despliegue de la app de finanzas | [semana-04.md](semana-04.md) |
| 5 | Nginx como proxy inverso | [semana-05.md](semana-05.md) |


## Seguridad

NO hay ningun`.env` ni contraseñas ni ips ṕúbicas acá!!!
Ah, y sigan a LearnLinuxTV en youtube!
