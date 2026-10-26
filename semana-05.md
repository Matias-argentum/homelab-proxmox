# Semana 5: Nginx como reverse proxy

## Objetivo

Configurar Nginx en la VM `gateway` para que los dispositivos de la LAN puedan acceder a la aplicación alojada en la VM privada `appserver`.

## Arquitectura de red

```text
┌──────────────────────────────────────────────────────────────┐
│                         Red LAN                              │
│                    <LAN_SUBNET>                              │
│                                                              │
│  ┌────────────────┐           ┌────────────────┐             │
│  │ Cliente (PC)   │           │ Cliente (celu) │             │
│  │ <CLIENT_IP>    │           │ <CLIENT_IP>    │             │
│  └───────┬────────┘           └───────┬────────┘             │
│          └─────────────┬──────────────┘                      │
└────────────────────────┼─────────────────────────────────────┘
                         │ HTTP :80
                         ▼
┌──────────────────────────────────────────────────────────────┐
│                    PROXMOX (Mini PC)                         │
│                                                              │
│  ┌────────────────────────────────────────────────────────┐  │
│  │ VM GATEWAY — Debian 13                                  │  │
│  │ Interfaz LAN: <GATEWAY_LAN_IP>                          │  │
│  │                                                        │  │
│  │ Nginx reverse proxy                                    │  │
│  │ - Escucha en: <GATEWAY_LAN_IP>:80                      │  │
│  │ - proxy_pass: http://<APPSERVER_PRIVATE_IP>:8080        │  │
│  │ - Cabeceras: X-Real-IP, X-Forwarded-For                │  │
│  └──────────────────────────┬─────────────────────────────┘  │
│                             │                                │
│                             │ Red interna: vmbr1             │
│                             │ <PRIVATE_SUBNET>               │
│                             ▼                                │
│  ┌────────────────────────────────────────────────────────┐  │
│  │ VM APPSERVER — Debian 13                                │  │
│  │ IP privada: <APPSERVER_PRIVATE_IP>                      │  │
│  │                                                        │  │
│  │ Aplicación en el puerto 8080                            │  │
│  │ - No tiene conexión directa a la LAN                    │  │
│  │ - Acceso a Internet a través de gateway                 │  │
│  └────────────────────────────────────────────────────────┘  │
└──────────────────────────────────────────────────────────────┘
```

`gateway` tiene una interfaz conectada a la LAN y otra a la red interna `vmbr1`. `appserver` está conectada únicamente a `vmbr1`, por lo que los clientes de la LAN acceden a la aplicación a través de Nginx.

## Trabajo realizado

### Día 1: Instalación de Nginx

Se instaló Nginx de forma nativa en la VM `gateway` para utilizarla como reverse proxy.

### Día 2: Configuración del Virtual Host

Se configuró un Virtual Host para recibir las solicitudes HTTP en el puerto 80 de la interfaz LAN de `gateway` y reenviarlas a la aplicación en `appserver`.

- **Dirección de escucha:** `<GATEWAY_LAN_IP>:80`
- **Destino del proxy:** `http://<APPSERVER_PRIVATE_IP>:8080`

### Día 3: Configuración de cabeceras

Se configuraron las cabeceras `X-Real-IP` y `X-Forwarded-For` para conservar la IP de origen del cliente y permitir que la aplicación o sus registros la identifiquen detrás del reverse proxy.

### Día 4: Pruebas de acceso

Se probó el acceso a la aplicación desde el navegador de un equipo y un celular conectados a la Wi-Fi doméstica.

La solicitud llega a Nginx en `gateway`, que la reenvía a la aplicación en `appserver` a través de la red interna.

## Flujo de tráfico

1. Un dispositivo conectado a la LAN solicita la aplicación por HTTP.
2. La solicitud llega a Nginx en la VM `gateway`.
3. Nginx reenvía la solicitud a `appserver` por la red privada `vmbr1`.
4. La aplicación responde a través de Nginx.

## Resultado

- La aplicación es accesible desde la LAN mediante la VM `gateway`.
- `appserver` continúa aislada de la LAN.
- Nginx funciona como punto de entrada HTTP y reverse proxy.
- Las cabeceras configuradas conservan información de la IP del cliente.