# Pihole-docker-macvlan
Esta guía documenta la instalación de **Pi-hole v6** sobre **Lubuntu 24.04.3 LTS** utilizando **Docker version 29.1.3, build f52814d**. El objetivo es usar una red `macvlan` para que Pi-hole tenga su propia IP (evitando conflictos con `systemd-resolved`) y configurar un puente permanente mediante `nmcli` para que el host y el contenedor puedan comunicarse.

---

## 1\. Consideraciones y Prerrequisitos

* **Sistema Operativo:** Lubuntu 24.04.3 LTS (Noble Numbat).
* **Software:** Docker Engine versión 29.1.3 (instalado desde repositorios oficiales de Docker, no de Ubuntu).
* **Hardware:** Conexión vía Ethernet (recomendado para estabilidad de `macvlan`).
* **Red Local:**
* **Gateway (Router):** `192.168.0.1` (ajustar si es distinto).
* **IP Pi-hole:** `192.168.0.10`.
* **IP Puente Host (Shim):** `192.168.0.51`.



* **Interfaz Física:** Identifica tu tarjeta con `ip addr`. En este ejemplo usaremos `enp3s0`.

---

## 2\. Instalación de Docker Engine (Versión Oficial 29.1.3)

No utilizaremos los paquetes de Lubuntu para garantizar que tienes la versión **29.1.3**.

1. **Limpiar versiones antiguas:**

```bash
sudo apt-get remove docker docker-engine docker.io containerd runc

```



2. **Instalar el repositorio de Docker:**

```bash
sudo apt-get update
sudo apt-get install ca-certificates curl
sudo install -m 0755 -d /etc/apt/keyrings
sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc

echo "deb \[arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/ubuntu $(. /etc/os-release \&\& echo "$VERSION\_CODENAME") stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
sudo apt-get update

```



3. **Instalar Docker:**

```bash
sudo apt-get install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin

```



*Verifica la versión con `docker --version`.*

---

## 3\. Configuración de Red Permanente (Host y Docker)

Para que el host (Lubuntu) pueda hablar con Pi-hole y para que la configuración sobreviva a reinicios, usaremos **NetworkManager (nmcli)**.

### A. Crear Interfaz "Shim" Permanente en el Host

Ejecuta lo siguiente (sustituye `enp3s0` por tu interfaz real):

```bash
sudo nmcli connection add type macvlan dev enp3s0 mode bridge ifname macvlan-shim con-name pihole-bridge ip4 192.168.0.51/32
sudo nmcli connection up pihole-bridge

```

> \*\*Nota:\*\* Esto crea un dispositivo virtual en Lubuntu con la IP `192.168.0.51` que permite saltar la restricción del kernel Linux que impide la comunicación Host-Macvlan.

### B. Crear la Red Macvlan en Docker

Llamaremos a la red `macvlan0` como solicitaste:

```bash
docker network create -d macvlan \\
  --subnet=192.168.0.0/24 \\
  --gateway=192.168.0.1 \\
  --ip-range=192.168.0.10/32 \\
  -o parent=enp3s0 macvlan0

```

---

## 4\. Configuración de Pi-hole (Docker Compose)

Crea la carpeta de trabajo y el archivo:

```bash
mkdir -p ~/pihole/etc-pihole
cd ~/pihole
nano docker-compose.yml

```

**Pega el siguiente contenido (actualizado con `macvlan0`):**

```yaml
# Docker Compose para Pi-hole v6 sobre Lubuntu 24.04
services:
  pihole:
    container\_name: pihole
    image: pihole/pihole:latest
    networks:
      macvlan0:
        ipv4\_address: 192.168.0.10   # IP propia para Pi-hole
    environment:
      TZ: 'America/Caracas'
      FTLCONF\_webserver\_api\_password: 'jimmyshoes' # Tu contraseña
      FTLCONF\_dns\_listeningMode: 'all'
    volumes:
      - './etc-pihole:/etc/pihole'
    cap\_add:
      - NET\_ADMIN
      - SYS\_TIME
      - SYS\_NICE
    restart: unless-stopped

networks:
  macvlan0:
    external: true

```

---

## 5\. Despliegue y Ruta de Comunicación

1. **Levantar el contenedor:**

```bash
sudo docker compose up -d

```



2. **Ruta Estática Permanente:**
   Para que el host sepa que debe buscar a la IP `.10` a través de la interfaz `macvlan-shim`, añade la ruta en NetworkManager:

```bash
sudo nmcli connection modify pihole-bridge +ipv4.routes "192.168.0.10/32 0.0.0.0"
sudo nmcli connection up pihole-bridge

```



---

## 6\. Verificación Final

|Acción|Resultado Esperado|
|-|-|
|**Acceso Web**|Entra a `http://192.168.0.10/admin` desde cualquier equipo.|
|**Prueba DNS**|Ejecuta `nslookup google.com 192.168.0.10` y debe responder.|
|**Persistencia**|Reinicia Lubuntu; la interfaz `macvlan-shim` y el contenedor deben subir solos.|

**¿Por qué es mejor así?**
Al usar `nmcli`, la interfaz virtual y la ruta se gestionan como cualquier otra conexión de red de Lubuntu. No necesitas scripts en el arranque ni desactivar `systemd-resolved`, ya que Pi-hole vive en su propia "parcela" de red (`192.168.0.10`) y no interfiere con el puerto 53 de la IP principal del host.

---

## 🚀 Paso Extra Opcional: Gestión Visual con Portainer CE

Para una administración avanzada y visual de tus contenedores, instalaremos **Portainer Community Edition (CE)**. Esto permite monitorear el rendimiento de Pi-hole, ver logs y gestionar actualizaciones de **Docker 29.1.3** desde un panel web, evitando el uso constante de la terminal.

### 1. Consideraciones de Configuración

* **Sin puerto 8000:** Se ha omitido este puerto para prescindir de las funciones "Edge", simplificando la exposición de puertos.
* **Socket de Docker:** El contenedor accede al socket local para controlar el motor Docker del host.
* **Persistencia de datos:** Toda la configuración de usuarios y entornos se guarda en un directorio local.

### 2. Implementación mediante Docker Compose

En lugar de un comando largo, utilizaremos un archivo de configuración dedicado.

1. **Crea el directorio y el archivo:**
```bash
mkdir -p ~/portainer && cd ~/portainer
nano portainer-compose.yaml

```


2. **Pega el siguiente contenido en el archivo:**
```yaml
services:
  portainer:
    image: portainer/portainer-ce:latest
    container_name: portainer
    restart: always
    security_opt:
      - no-new-privileges:true
    volumes:
      - /etc/localtime:/etc/localtime:ro
      - /var/run/docker.sock:/var/run/docker.sock # Control del socket local
      - ./portainer_data:/data                    # Datos persistentes
    ports:
      - "9443:9443"                               # Solo puerto HTTPS seguro

```


3. **Despliega el panel:**
```bash
sudo docker compose -f portainer-compose.yaml up -d

```



### 3. Acceso y Comportamiento de Red

Gracias a la estructura de red que configuramos para la macvlan y el bridge (`shim`), notarás que el panel es accesible desde dos rutas distintas dentro de tu red local:

* **Ruta Física:** `https://192.168.0.200:9443`
* **Ruta del Puente (Shim):** `https://192.168.0.51:9443`

Este acceso dual ocurre porque Portainer opera en la red *bridge* interna de Docker, la cual se vincula automáticamente a todas las interfaces lógicas de Lubuntu, incluyendo la interfaz virtual creada para comunicarse con Pi-hole.

### 4. Configuración en la Interfaz (GUI)

1. **Certificado de Seguridad:** Al entrar, el navegador mostrará una advertencia. Selecciona **Avanzado** y luego **Continuar** (es normal, ya que Portainer usa un certificado auto-firmado).
2. **Registro de Administrador:** Crea tu usuario y contraseña de inmediato. Si pasan más de 12 minutos sin completar este paso, deberás reiniciar el contenedor por seguridad: `sudo docker restart portainer`.
3. **Conexión con el Entorno:** Una vez dentro, selecciona el entorno **"Local"**. Verás instantáneamente tu stack de Pi-hole y podrás gestionar sus recursos de forma visual.

