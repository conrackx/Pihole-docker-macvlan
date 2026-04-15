# Pihole-docker-macvlan (Optimized for v6)
Esta guía documenta la instalación de **Pi-hole v6** sobre **Lubuntu 24.04.4 LTS** utilizando **Docker v29.3.0**. El objetivo es implementar una red `macvlan` para que Pi-hole tenga su propia IP, evitando conflictos con `systemd-resolved` y maximizando el rendimiento en hardware heredado (legacy).

---

## 1. Consideraciones y Prerrequisitos

*   **Sistema Operativo:** Lubuntu 24.04.4 LTS (Noble Numbat).
*   **Motor:** Docker Engine v29.3.0 (Repositorio Oficial).
*   **Hardware:** Dell Latitude E6400 (Intel Core 2 Duo) o similar.
*   **Red Local:**
    *   **Gateway:** `192.168.0.1`
    *   **IP Pi-hole:** `192.168.0.10`
    *   **IP Puente Host (Shim):** `192.168.0.51`
    *   **Interfaz Física:** `enp0s25` (Identificar con `ip addr`).

---

## 2. Configuración de Red Permanente (Host y Docker)

Para que el host (Lubuntu) pueda comunicarse con Pi-hole sin violar las restricciones del kernel Linux en macvlan, usaremos un puente virtual permanente.

### A. Crear Interfaz "Shim" Permanente
```bash
sudo nmcli connection add type macvlan dev enp0s25 mode bridge ifname macvlan0 con-name pihole-bridge ip4 192.168.0.51/32
sudo nmcli connection up pihole-bridge
```

### B. Crear la Red Macvlan en Docker
```bash
docker network create -d macvlan \
  --subnet=192.168.0.0/24 \
  --gateway=192.168.0.1 \
  --ip-range=192.168.0.10/32 \
  -o parent=enp0s25 pihole_macvlan
```

---

## 3. Configuración de Pi-hole (Docker Compose)

Implementamos una persistencia de datos 100% garantizada y optimización para redes IPv4 puras.

```bash
mkdir -p ~/pihole/etc-pihole ~/pihole/etc-dnsmasq.d
cd ~/pihole
```

**`docker-compose.yml` optimizado:**
```yaml
services:
  pihole:
    container_name: pihole
    image: pihole/pihole:latest
    networks:
      pihole_macvlan:
        ipv4_address: 192.168.0.10
    environment:
      TZ: 'America/Caracas'
      FTLCONF_webserver_api_password: 'tu_password'
      FTLCONF_dns_listeningMode: 'all'
      # Optimización para CPUs Legacy (Desactiva análisis IPv6 innecesario)
      FTLCONF_dns_aaaaQueryAnalysis: 'false'
      FTLCONF_dns_resolveIPv6: 'false'
    volumes:
      - './etc-pihole:/etc/pihole'
      - './etc-dnsmasq.d:/etc/dnsmasq.d'
    cap_add:
      - NET_ADMIN
      - SYS_TIME
      - SYS_NICE
    restart: unless-stopped

networks:
  pihole_macvlan:
    external: true
```

---

## 4. Despliegue y Enrutamiento Crítico

1.  **Levantar el servicio:**
    ```bash
    docker compose up -d
    ```

2.  **Ruta Estática del Host:**
    Obligamos al sistema operativo a buscar al contenedor a través del puente virtual:
    ```bash
    sudo nmcli connection modify pihole-bridge +ipv4.routes "192.168.0.10/32 0.0.0.0"
    sudo nmcli connection up pihole-bridge
    ```

---

## 5. Optimización de Hardware (Performance Tuning)

Para hardware antiguo como el Core 2 Duo, estas mejoras son vitales para evitar latencia en las consultas DNS:

### A. ZRAM (Memoria Comprimida)
Aumenta virtualmente la RAM sin usar el disco lento:
```bash
sudo apt install zram-tools
# Configurar ALGO=zstd y PERCENT=150 en /etc/default/zramswap
sudo systemctl restart zramswap
```

### B. Kernel Mitigations (Velocidad de CPU)
Recupera hasta un 20% de ciclos de CPU desactivando parches de seguridad para ataques de canal lateral (seguro en entornos aislados):
1.  Editar `/etc/default/grub`.
2.  Añadir `mitigations=off` a `GRUB_CMDLINE_LINUX_DEFAULT`.
3.  Ejecutar `sudo update-grub` y reiniciar.

---

## 6. Estrategia DNS Anti-Bypass (Router)

Para evitar que dispositivos móviles ignoren el Pi-hole (vía IPv6 o DNS Secundario):
1.  **Desactivar IPv6** en todo el router.
2.  **DNS Secundario "Fantasma":** Si el router obliga a poner un segundo DNS, usa una IP de tu red que no exista (ej. `192.168.0.254`). Esto forzará al dispositivo a re-intentar con el Primario (Pi-hole) en lugar de saltar a los DNS de Google.

---

## 🚀 Mantenimiento
Para actualizar y sincronizar el estado:
```bash
cd ~/pihole && docker compose pull && docker compose up -d
```
