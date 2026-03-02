# 1 Servidor Web Clonado para Alta Disponibilidad

Para garantizar la **alta disponibilidad** y el **balanceo de carga** del servicio web, se creó un **servidor web clonado** (Nodo 2) a partir del servidor web principal (Nodo 1).

**Características del servidor clonado:**

- **Tipo de clonación:** Full Clone (copia completa e independiente)
- **Función:** Servidor web redundante que comparte la misma configuración y contenido
- **IP asignada:** `192.168.18.11/24` (mientras que el Nodo 1 tiene `192.168.18.10/24`)
- **Propósito:** 
  - **Alta disponibilidad:** Si el Nodo 1 falla, el servicio continúa operativo a través del Nodo 2
  - **Balanceo de carga:** El servidor Nginx distribuye las peticiones entre ambos nodos, mejorando el rendimiento y la capacidad de respuesta

**Arquitectura del sistema:**
```
Cliente → DNS (192.168.18.1) → Nginx Reverse Proxy (192.168.18.1) → [Web1 (192.168.18.10) | Web2 (192.168.18.11)]
```

Esta configuración permite que el sistema tolere fallos en cualquiera de los servidores web sin interrumpir el servicio, cumpliendo con los requisitos de **continuidad operativa** y **escalabilidad horizontal** del proyecto.

# 2. Solución de Problemas
## Problema: "ERR_CONNECTION_REFUSED" y "Time Out"

**Síntoma:** El ping funcionaba, pero el navegador no cargaba la web o mostraba error de conexión.
**Causa:**

1. **Enrutamiento erróneo:** Proxmox tenía una regla NAT antigua que enviaba el tráfico directamente al Nodo 1 (`.10`), saltándose el Nginx. Al estar el Nodo 1 apagado, la conexión era rechazada.
2. **Base de Datos Desincronizada:** WordPress tenía guardadas las URLs como `https` pero Nginx estaba configurado en `http` (o viceversa), rompiendo la carga de imágenes (CSS/JS).

## Solución Aplicada

1. **Redirección de Puertos:** Se borraron las reglas que apuntaban a la `.10` y se crearon nuevas apuntando exclusivamente a la `.1` (Nginx).
2. **Ajuste de Base de Datos:** Se ejecutaron comandos SQL en los nodos Web para alinear las URLs con el protocolo correcto (HTTP o HTTPS).

**Comandos SQL utilizados para reparar URLs (MySQL/MariaDB):**
*(Ejecutados en Web1 y Web2 dentro de `sudo /opt/lampp/bin/mysql -u root`)*

```sql
USE grupo3;
-- Actualizar URLs principales
UPDATE wp_options SET option_value = 'https://www.connectix.es' WHERE option_name = 'siteurl';
UPDATE wp_options SET option_value = 'https://www.connectix.es' WHERE option_name = 'home';

-- Reemplazar enlaces dentro de los posts y metadatos
UPDATE wp_posts SET post_content = REPLACE(post_content, 'http://www.connectix.es', 'https://www.connectix.es');
UPDATE wp_postmeta SET meta_value = REPLACE(meta_value, 'http://www.connectix.es', 'https://www.connectix.es');
```
---

## 5. Diagnóstico desde el Cliente (Windows)
Si la web no carga, usar estos comandos para descartar caché del navegador, apagando el server1 (.10):

```powershell
# 1. Ver qué IP resuelve el DNS (Debe ser la de Proxmox: 172.16.204.233)
nslookup www.connectix.es
# 2. Comprobar conectividad real (Ping)
ping www.connectix.es
# 3. Ver la respuesta cruda del servidor (Evita caché de Chrome)
curl -I -v http://www.connectix.es
```