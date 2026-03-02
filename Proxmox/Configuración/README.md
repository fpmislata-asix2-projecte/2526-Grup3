# Documentación de la Infraestructura de Red y Virtualización en Proxmox VE

Este documento detalla los pasos seguidos en el nodo Proxmox (`pve`) para establecer la red interna, el enrutamiento (NAT) y la creación de las máquinas virtuales (VMs) de DNS/DHCP y Web.

## 1\. Configuración de Red del Nodo Proxmox (Host)

La configuración de red se gestiona a través del archivo `/etc/network/interfaces` en el servidor Proxmox. Se utilizan dos puentes (`vmbr0` y `redwp`) para separar el tráfico WAN y LAN.

![](../../imágenes/px_conf/px_conf_3.png)
![](../../imágenes/px_conf/px_conf_4.png)

### 1.1 Creación de la Red Interna (Linux Bridge)

Se crea el puente **`redwp`** (Linux Bridge) que actuará como la red interna (LAN) para todas las máquinas virtuales. Este puente no tiene un puerto físico y está configurado para la subred privada **`192.168.18.0/24`** y su IP estática **`192.168.18.254/24`**.

### 1.2 Configuración del Archivo `/etc/network/interfaces`

Esta configuración consolida el puente WAN (`vmbr0`) con el puente LAN (`redwp`) e incluye las reglas de NAT para asegurar que la red interna pueda salir a Internet.

La configuración es la siguiente:

![](../../imágenes/px_conf/px_conf_1.png)

Una vez modificado el archivo, se aplican los cambios:

```bash
systemctl restart networking
```

## 2\. Configuración de NAT en el Nodo Proxmox.

Se configuran las reglas de NAT (Enmascaramiento) y Forzamiento de Tráfico (Forwarding) de manera persistente usando el paquete `iptables-persistent`.

### 2.1 Activar IP forwarding


1. Creamos el fichero en `/etc/sysctl.d/` :
```bash
sudo nano /etc/sysctl.d/99-ipforward.conf
```
2. Escribimos en su interior el siguiente código:
```bash
net.ipv4.ip_forward=1
```
3. Aplicamos los cambios del sistema guardando.
```bash
sysctl --system
```
4. Comprobamos que quedó activo:
```bash
sysctl net.ipv4.ip_forward
# Debe devolver: net.ipv4.ip_forward = 1
```
![](../../imágenes/px_conf/px_conf_2.png)


### 2.2 Preparación e Instalación de `iptables-persistent`

1.  **Añadir Repositorios de Debian:**
    Editar `/etc/apt/sources.list` y agregar las líneas de repositorios de Debian:

    ```bash
    sudo nano /etc/apt/sources.list
    deb http://deb.debian.org/debian bookworm main contrib
    deb http://deb.debian.org/debian bookworm-updates main contrib
    ```

2.  **Actualizar e Instalar:**

    ```bash
    sudo apt update
    sudo apt install -y iptables-persistent
    ```
### 2.3 Reglas NAT y Forwarding

Aplicamos las iptable correspondientes para dar accedo a Internet a la red de nuestra **`redwp`**:

1.  **Regla de Enmascaramiento (NAT):** Permite que el tráfico de la red interna (`192.168.18.0/24`) salga por la interfaz WAN (`vmbr0`) usando la IP pública del Proxmox.
    ```bash
    sudo iptables -t nat -A POSTROUTING -s 192.168.18.0/24 -o vmbr0 -j MASQUERADE
    ```
2.  **Reglas de Forwarding (Apertura de tráfico):** Permite el tráfico de salida y de retorno.
    ```bash
    sudo iptables -A FORWARD -s 192.168.18.0/24 -o vmbr0 -j ACCEPT
    sudo iptables -A FORWARD -d 192.168.18.0/24 -m state --state ESTABLISHED,RELATED -i vmbr0 -j ACCEPT
    ```
### 2.4 Hacer las Reglas Persistentes

Finalmente, se guardan las reglas activas para que se carguen automáticamente en cada reinicio:

```bash
sudo netfilter-persistent save
```

## 3\. Creación y Configuración de Máquinas Virtuales (VMs)

Se crearon al menos dos máquinas virtuales (DNS/DHCP y WordPress) siguiendo estos pasos.

1.  **Creación de la VM:**
      * **General:** Asignar un ID y nombre.
      * **OS:** Cargar la imagen ISO de Ubuntu Server.
      * **System, Disks:** Configuración estándar de hardware.
      * **CPU/RAM:** Asignar recursos suficientes.

![](../../imágenes/px_conf/px_conf_5.png)
![](../../imágenes/px_conf/px_conf_6.png)
![](../../imágenes/px_conf/px_conf_7.png)
![](../../imágenes/px_conf/px_conf_8.png)
![](../../imágenes/px_conf/px_conf_9.png)
![](../../imágenes/px_conf/px_conf_10.png)
![](../../imágenes/px_conf/px_conf_11.png)
![](../../imágenes/px_conf/px_conf_12.png)

2.  **Conexión a la Red Interna:**

    En la pestaña **Hardware** de la VM, en la configuración de la **Tarjeta de Red**, se selecciona el puente **`redwp`** para que las VMs estén en la red privada `192.168.18.x`.

![](../../imágenes/px_conf/px_conf_13.png)
![](../../imágenes/px_conf/px_conf_14.png)

3. **Instalación de los servers**
    
    Cuando instalemos los Servidores, es seguir los pasos del Instalador, elegir el nombre, contraseña, etc... al gusto y importante activar la opción de SSH.

4.  **Configuración de IPs:**
    En nuestro Server DNS/DHCP pondremos una IP estática y el Server WEB recibira una IP del DHCP de nuestro Server DHCP:
    **Netplan** (el archivo `.yaml` dentro de `/etc/netplan/`):

| Máquina | IP Asignada | Gateway | DNS |
| :--- | :--- | :--- | :--- |
| **DNS/DHCP Server** | `192.168.18.1/24` | `192.168.18.254` | `1.1.1.1.1`, `8.8.8.8` |
| **WordPress Server** | `DHCP` | `DHCP` | `192.168.18.1, 8.8.8.8` |

![](../../imágenes/px_conf/px_conf_15.png)
![](../../imágenes/px_conf/px_conf_16.png)

Si hemos hecho la configuración de la red Interna y el NAT, ambos Servers deberían tener conexión entre sí y al Servidor Proxmox.
![](../../imágenes/px_conf/px_conf_18.png)
![](../../imágenes/px_conf/px_conf_19.png)
Y acceso a Internet.
![](../../imágenes/px_conf/px_conf_20.png)
![](../../imágenes/px_conf/px_conf_21.png)

# 4. Resolución de Incidencias: Ampliación de Almacenamiento
Durante el proceso de escalado de la infraestructura para implementar la Alta Disponibilidad (Clonación de nodos), nos encontramos con un error crítico de almacenamiento que impedía el funcionamiento del servidor Proxmox.

## 4.1. Descripción del Problema
Al intentar realizar un **Full Clone** de la máquina virtual `ServerWEB` (VM 100) para crear el segundo nodo, el proceso fallaba o dejaba el servidor inoperativo (bloqueo de interfaz web y errores *connection timed out*).

El log de tareas de Proxmox arrojó el siguiente error crítico:

> `WARNING: Sum of all thin volume sizes (96.00 GiB) exceeds the size of whole volume group (<59.00 GiB).`

### Análisis del Error
Este mensaje indicaba un problema de **Sobreasignación (Overprovisioning)**.

* **Capacidad real asignada a VMs:** El volumen lógico `data` tenía un tamaño físico de **~20 GB**.
* **Demanda de recursos:** Se intentaron iniciar 3 máquinas virtuales con discos de 32 GB cada una (Total: 96 GB).
* **Consecuencia:** Al superar el espacio físico real disponible, el *Thin Pool* colapsaba, bloqueando los servicios `pvestatd` y `pvedaemon` del hipervisor.

## 4.2. Diagnóstico

Para verificar la estructura del disco físico, accedimos a la terminal del nodo y ejecutamos el comando `lsblk`.
**Resultado del diagnóstico:**
Se observó que, aunque el disco físico (`nvme0n1`) tenía una capacidad de **465.8 GB**, Proxmox solo estaba utilizando una partición de **59 GB** (`nvme0n1p3`), dejando aproximadamente **400 GB de espacio sin asignar** ni utilizar.

```bash
NAME        MAJ:MIN RM   SIZE RO TYPE MOUNTPOINTS
nvme0n1     259:0    0 465.8G  0 disk
├─nvme0n1p3 259:3    0    59G  0 part  <-- Solo 59GB usados
│ ├─pve-data...      0    20G  0 lvm   <-- Solo 20GB para VMs
```

## 4.3. Solución Implementada: Expansión del LVM
Para recuperar el espacio no utilizado y asignarlo al almacenamiento de máquinas virtuales sin reinstalar el servidor, realizamos el siguiente procedimiento de "cirugía en caliente" sobre las particiones.

### Paso 1: Copia de Seguridad

Antes de modificar la tabla de particiones, se realizaron **Backups** de las VMs existentes (`vma.zst`) en el almacenamiento `local` y se descargaron a un equipo externo para garantizar la integridad de los datos.

### Paso 2: Instalación de Herramientas
Instalamos `parted` para gestionar las particiones desde la línea de comandos:

```bash
apt update && apt install parted -y
```

### Paso 3: Redimensionado de la Partición Física
Se expandió la partición 3 para que ocupara el 100% del espacio libre del disco NVMe:

```bash
parted /dev/nvme0n1 resizepart 3 100%
```

### Paso 4: Actualización del Volumen Físico (LVM)
Notificamos al kernel y al gestor LVM que el tamaño del dispositivo físico había cambiado:

```bash
pvresize /dev/nvme0n1p3
```
*Salida:* `Physical volume "/dev/nvme0n1p3" changed`

### Paso 5: Extensión del Thin Pool
Finalmente, asignamos todo el nuevo espacio libre al volumen lógico `data` (donde se alojan los discos de las VMs):

```bash
lvextend -l +100%FREE /dev/pve/data
```
*Salida:* `Logical volume pve/data successfully resized.`

## 4.4. Verificación de Resultados
Tras la operación, se volvió a ejecutar `lsblk` y `lvs` para confirmar la nueva capacidad.

* **Capacidad anterior:** 20 GB.
* **Nueva capacidad:** **433 GB**.
* **Estado:** El sistema pasó de estar saturado al 100% a tener una ocupación real inferior al **4%**.
![](../../imágenes/px_conf/px_conf_22.png)

Esto permitió realizar la clonación completa (**Full Clone**) del Nodo 2 y encender simultáneamente toda la infraestructura (DNS + Web1 + Web2) sin problemas de rendimiento.
