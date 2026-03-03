# Documentación: Balanceador de Carga Nginx y Enrutamiento de Red

## 1. Arquitectura del Sistema

Nginx actúa en esta arquitectura como un servidor proxy inverso y balanceador de carga de alto rendimiento. Su función principal es recibir las peticiones de los clientes externos y distribuirlas eficientemente entre los nodos del clúster WordPress, optimizando el uso de recursos y asegurando la disponibilidad del servicio incluso si uno de los servidores web falla.

* **Entrada:** Todo el tráfico externo (Windows) llega a la IP pública/puente de Proxmox (`172.16.204.233`).
* **Portero (Nginx - 192.168.18.1):** Proxmox redirige el tráfico al servidor Nginx. Este descifra el HTTPS (si aplica) y decide a qué servidor web enviar la petición.
* **Workers (Web1 y Web2):** Servidores WordPress que reciben el tráfico ya limpio en el puerto 80.

---

## 2. Configuración de Nginx (VM 101)

El archivo de configuración define un grupo de servidores (*upstream*) y gestiona las cabeceras para que WordPress no se pierda.

**Archivo:** `/etc/nginx/sites-available/default`

```bash
# --- 1. DEFINICIÓN DEL CLÚSTER (UPSTREAM) ---
upstream backend_wordpress {
    # Estrategia: Round Robin (por defecto)
    # Nginx repartirá las peticiones una a una entre estos servidores
    server 192.168.18.10; # Web1
    server 192.168.18.11; # Web2
}

# --- 2. SERVIDOR PRINCIPAL (BALANCEADOR HTTP) ---
server {
    # Escucha tráfico web normal (sin cifrar)
    listen 80;
    server_name connectix.es www.connectix.es;

    location / {
        # Envío al clúster definido arriba
        proxy_pass http://backend_wordpress;

        # --- CABECERAS CRÍTICAS ---
        # Pasa el dominio real, no la IP interna del balanceador
        proxy_set_header Host $host;
        
        # Pasa la IP real del cliente (Windows) para que WordPress sepa quién visita
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    }
}
```

### Comandos de Gestión Nginx

```bash
# Verificar que la sintaxis es correcta
sudo nginx -t
# Reiniciar el servicio para aplicar cambios
sudo systemctl restart nginx
# Ver estado
sudo systemctl status nginx

```
---

## 3. Configuración de Red y Firewall (IPTables en Proxmox)
El problema principal residía en cómo Proxmox manejaba el tráfico entrante (NAT). Se configuró para que actúe como un router transparente.

### Reglas de NAT (En el Nodo Físico Proxmox)
Estas reglas toman el tráfico que llega a la tarjeta de red del laboratorio y lo empujan hacia dentro (a la VM .1).

```bash
# 1. Limpieza de reglas antiguas (Importante si apuntaban a servidores muertos)
iptables -t nat -D PREROUTING -i vmbr0 -p tcp --dport 80 -j DNAT --to-destination 192.168.18.10:80
# 2. Reglas de redirección al BALANCEADOR (.1)
# Tráfico Web (HTTP)
iptables -t nat -A PREROUTING -i vmbr0 -p tcp --dport 80 -j DNAT --to-destination 192.168.18.1:80
# Tráfico Seguro (HTTPS)
iptables -t nat -A PREROUTING -i vmbr0 -p tcp --dport 443 -j DNAT --to-destination 192.168.18.1:443
# Tráfico DNS (Para resolución de nombres)
iptables -t nat -A PREROUTING -i vmbr0 -p udp --dport 53 -j DNAT --to-destination 192.168.18.1:53
# 3. Permitir el tráfico de retorno (Masquerade)
# Esto permite que las VMs respondan a Internet usando la IP de Proxmox
iptables -t nat -A POSTROUTING -s 192.168.18.0/24 -o vmbr0 -j MASQUERADE
```
### Comandos para guardar reglas (Persistencia)
Para evitar perder la configuración al reiniciar Proxmox:

```bash
netfilter-persistent save
```
