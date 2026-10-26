# Semana 4: Docker en `appserver` y despliegue de la app de finanzas

## Día 1: instalación de Docker

- Agregué a `apt` el repositorio oficial de Docker.
- Guardé la clave GPG en `/etc/apt/keyrings/docker.asc` y configuré el repositorio en `/etc/apt/sources.list.d/docker.list`.
- Instalé Docker Engine, Docker CLI, `containerd.io`, Docker Buildx y el plugin de Docker Compose.
- Verifiqué que el servicio estuviera activo y habilitado para arrancar con el sistema.

## Día 2: usuario y contenedor de prueba

- Agregué el usuario de `appserver` al grupo `docker`.
- Cerré la sesión y volví a entrar para que se aplicaran los cambios de grupo.
- Ejecuté `docker run hello-world` sin usar `sudo`.

## Día 3: preparación del proyecto


### Dockerfile del backend

Preparé `hexa-finance/Dockerfile` con una compilación en dos etapas:

- **Compilación:** imagen de Maven basada en Eclipse Temurin 25.
- **Ejecución:** imagen `eclipse-temurin:25-jre-alpine` y usuario sin privilegios (`app`).

Alpine usa `musl libc`. Para esta aplicación no representó un problema; supuestamente  si una aplicación requiere librerías nativas, hay que comprobar la compatibilidad o elegir otra imagen base.

### Archivos y configuración de Docker

- Añadí `hexa-finance/.dockerignore` para excluir archivos y carpetas que no hacen falta en la imagen, como `target/`, `.git`, `.idea/` y `.env`.
- Preparé `docker-compose.yml` con tres servicios en la red `finance-network`:
  - **`mysql-db`:** MySQL 8.4, comprobación de salud, volumen nombrado `mysql_data` y montaje de `init.sql`.
  - **`phpmyadmin`:** imagen con versión fijada.
  - **`app`:** backend publicado en el puerto 8080 y configurado para esperar a que MySQL esté saludable.
- Los puertos de MySQL y phpMyAdmin se publican solo en la interfaz de loopback del servidor. Así no quedan expuestos directamente a otros equipos de la red.
- La aplicación se conecta a MySQL mediante variables de entorno.

### Copia del proyecto a `appserver`

- Instalé `rsync` también en `appserver`.
- Primero hice una prueba en seco y luego copié el proyecto excluyendo datos locales, archivos de compilación, `.git`, `.env`, `.idea` y archivos del editor.

## Día 4: despliegue y prueba

- Ejecuté `docker compose up -d --build` en `appserver`.
- Comprobé que los tres contenedores arrancaran.
- Probé el acceso a la aplicación desde la laptop mediante un túnel SSH. Usé el puerto local 9090 porque el 8080 ya estaba ocupado por otro entorno local.
- La aplicación respondió correctamente a través del túnel.

### Problemas resueltos

- **Error de YAML:** corregí una indentación incorrecta en el archivo Compose.
- **`UnknownHostException: mysql-db`:** al probar localmente, la aplicación no encontraba el nombre del servicio porque el contenedor de MySQL no estaba en ejecución. Una vez corregido eso, la aplicación pudo conectarse.

## Estado al cierre de la semana

- `appserver` tiene Docker y la aplicación de finanzas desplegados junto con MySQL y phpMyAdmin.
- La aplicación todavía no es accesible directamente desde la LAN: `appserver` está en la red interna del homelab.
- El acceso cómodo desde otros dispositivos de la LAN está previsto para la semana 5, mediante Nginx en `gateway`.
- Hay snapshots anteriores al despliegue de la aplicación. Queda pendiente tomar uno nuevo con el stack funcionando.

## Día 5: arquitectura de contenedores

### Capas de red

De afuera hacia adentro:

1. **LAN doméstica:** conecta la laptop, otros dispositivos y la interfaz LAN de `gateway`. Se omiten las direcciones por seguridad.
2. **Red interna de Proxmox:** usa `vmbr1`, sin conexión física directa. Conecta la interfaz interna de `gateway` con `appserver`.
3. **Red de Docker:** `finance-network`, creada por Docker dentro de `appserver`. Docker le asigna un rango interno, que se puede consultar con `docker network inspect`.

### Diagrama

```text
Laptop
  |
  | SSH a través de gateway
  | Túnel actual: puerto local 9090 → puerto 8080 de appserver
  v
gateway
  |
  | Red interna de Proxmox (vmbr1)
  v
appserver
+---------------------------------------------------------+
| Docker Engine                                           |
|                                                         |
| Red: finance-network                                    |
| DNS interno de Docker por nombre de servicio            |
|                                                         |
| +----------------------+      +----------------------+ |
| | app                  |      | mysql-db             | |
| | hexa-finance-app     |----->| finance-db           | |
| | Spring Boot / Java   | 3306 | MySQL 8.4            | |
| | Usuario sin root     |      | Healthcheck          | |
| | Puerto 8080          |      | Volumen: mysql_data  | |
| | publicado en todas   |      | init.sql, solo lectura| |
| | las interfaces       |      | Puerto local: 3306   | |
| +----------------------+      +----------------------+ |
|                                      ^                  |
|                                      |                  |
| +----------------------+             |                  |
| | phpMyAdmin           |-------------+                  |
| | finance-admin        | PMA_HOST=mysql-db             |
| | Puerto local: 8081   |                                |
| +----------------------+                                |
+---------------------------------------------------------+
```

### Servicios

- **`app` (`hexa-finance-app`)**
  - Imagen construida con un Dockerfile de dos etapas.
  - Se conecta a MySQL usando `SPRING_DATASOURCE_URL`, `SPRING_DATASOURCE_USERNAME` y `SPRING_DATASOURCE_PASSWORD`.
  - Las variables de entorno proporcionadas al contenedor reemplazan los valores correspondientes de `application.properties`.
  - Espera a que MySQL esté saludable antes de iniciar.

- **`mysql-db` (`finance-db`)**
  - Usa MySQL 8.4.
  - Guarda los datos en el volumen nombrado `mysql_data`, no en una carpeta del proyecto.
  - Ejecuta `init.sql` al inicializar una base vacía.

- **`phpmyadmin` (`finance-admin`)**
  - Proporciona una interfaz para administrar MySQL.
  - Se publica únicamente en la interfaz local de `appserver`; se accede mediante un túnel SSH.

### Puertos publicados en `appserver`

| Puerto | Servicio | Alcance |
|---|---|---|
| 8080 | Aplicación | Publicado en todas las interfaces de `appserver`, dentro de la red interna |
| 8081 | phpMyAdmin | Solo en la interfaz local de `appserver`; acceso mediante túnel SSH |
| 3306 | MySQL | Solo en la interfaz local de `appserver` |
