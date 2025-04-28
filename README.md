CURSO DE PROXMOX VE

MANUAL PRÁCTICO DEL ESTUDIANTE

## **1\. Instalación de Proxmox VE**

### **Requisitos:**

* USB bootable con Proxmox VE ISO  
* PC o servidor compatible

### **Pasos:**

1. **Descarga ISO de Proxmox VE:**  
    Ve a https://www.proxmox.com/en/downloads  
    👉 Clic en *Download Proxmox VE ISO Installer*  
2. **Crear USB bootable:**  
    Usa [Rufus](https://rufus.ie) o balenaEtcher  
    👉 Selecciona ISO  
    👉 Selecciona USB  
    👉 Clic en *Start*  
3. **Instalar Proxmox:**

   * Inserta USB y arranca desde él  
   * 👉 Clic en *Install Proxmox VE*  
   * Acepta los términos  
   * Selecciona disco  
   * Configura red, contraseña y zona horaria  
   * Finaliza y reinicia

## **2\. Configuración inicial y acceso web**

1. **Conéctate a la red LAN** con IP estática durante la instalación.  
2. Abre navegador desde otro equipo: 👉 Navega a: `https://<IP-de-Proxmox>:8006`  
3. Accede con:  
   1. **Usuario:** `root`  
   2. **Contraseña:** la que configuraste  
   3. **Realm:** `pam`  
4. Si ves advertencia SSL 👉 Clic en *Avanzado \> Aceptar riesgo*

## **3\. Configurar red (bridge)**

1. **Clic en el nodo \-\> System \-\> Network**  
2. Clic en **Create \-\> Linux Bridge**  
3. Llenar información:

   1. **Name:** vmbr0  
   2. **IPv4:** 192.168.1.11/24 (ejemplo)  
   3. **Gateway:** 192.168.1.1 (ejemplo)  
   4. **`Bridge ports:`** ens1 (poner el nombre de la interfaz de red a usar)  
   5. **`Advanced -> MTU:`** `1450`  
   6. `Aceptar`  
   7. `Clic en Apply Configuration`

## **4\. Crear una máquina virtual desde ISO**

### ***Subir ISO:***

1. *En la interfaz web: 👉 Clic en Nodo (expandir submenús)*  
    *👉 Clic en un volumen (ej local) \> ISO Images*  
    *👉 Clic en Upload o en Download from URL*
    * Pegar la URL: [https://cdimage.debian.org/debian-cd/current/amd64/iso-cd/debian-12.10.0-amd64-netinst.iso](https://cdimage.debian.org/debian-cd/current/amd64/iso-cd/debian-12.10.0-amd64-netinst.iso)  
    *👉 Selecciona ISO y súbela*

***Crear VM:***

1. *👉 Clic en Create VM*  
2. ***General:** Nombre: `mi-linux-iso`*  
3. ***OS:** ISO image: Selecciona la que subiste*  
4. ***System**: Dejar por defecto*  
5. ***Disks**: Bus/Device: Virtio Block, Storage: local (ejemplo), Disk size: 4 (ejemplo)*  
6. ***CPU:** Sockets 1 , Cores 2, Type: host*  
7. ***Memory**: 2048 (2GB de RAM)*  
8. ***Network**: Bridge: vmbr0, MTU 1450*  
9. ***Confirm:** enable after created, Finish*  
10.  *Abrir la Consola y seguir la instalación manual*

## **5\. Crear una máquina virtual con cloud init y un script**

### ***Entrar a la consola del nodo:***

1. *Clic en el **Nodo \-\> Shell***  
2. *Crear un nuevo script con **nano template.sh***  
3. *Pegar el siguiente script y guardarlo*

```sh
# crea template con cloud init
#SE PUEDE USAR PARA DEBIAN .qcow2 O UBUNTU .img
# installing libguestfs-tools only required once, prior to first run
apt update -y
apt install libguestfs-tools -y

# download a debian cloud-init disk image:
wget https://cloud.debian.org/images/cloud/bookworm/20241125-1942/debian-12-genericcloud-amd64-20241125-1942.qcow2

virt-customize -a "debian-12-genericcloud-amd64-20241125-1942.qcow2" --install qemu-guest-agent --run-command 'ln -sf /usr/share/zoneinfo/America/La_Paz /etc/localtime' --run-command 'echo "America/La_Paz" > /etc/timezone'
qemu-img resize debian-12-genericcloud-amd64-20241125-1942.qcow2 2G


qm create 4002 --name "debian-12-qcow2-template" --memory 2048 --cores 2 --net0 virtio,bridge=vmbr0

qm importdisk 4002 debian-12-genericcloud-amd64-20241125-1942.qcow2 local
qm set 4002 --virtio0 local:4002/vm-4002-disk-0.raw
qm set 4002 --ide2 local:cloudinit
qm set 4002 --boot c --bootdisk virtio0
qm set 4002 --serial0 socket --vga serial0
qm set 4002 --agent enabled=1
#qm template 4002

#rm debian-12-genericcloud-amd64-20241125-1942.qcow2

```

4. *Ejecutar el script con **bash template.sh** y esperar que se ejecute completamente.*  
5. *En la interfaz web ya debería verse una nueva VM creada con el nombre **debian-12-qcow2-template**.*

***Configurar VM***

1. *👉 Clic en la VM recién creada.*  
2. ***Hardware:***   
   ***Processors** \-\> Type: host*  
   ***Network** \-\> Advanced \-\> MTU: 1450*  
3. ***Cloud init:***   
   ***User**: mi-usuario*  
   ***Password:** mi-password*  
   ***DNS servers:** 8.8.8.8*  
   ***Upgrade packages:** No (lo recomendado es Yes)*

***Iniciar la VM***

1. *👉 Clic en la VM recién creada.*  
2. *Clic en **Start***  
3. *Clic en **Console***  
4. *Una vez iniciado el sistema loguearse con el usuario y contraseña.*

## **6\. Crear un template de VM y Clonar VMs**

***Crear un template***

1. *👉 Clic en la VM **debian-12-qcow2-template**.*  
2. *Clic en **Shutdown.***  
3. *👉 Clic derecho en la VM **debian-12-qcow2-template** \-\> Convert to template*  
4. *La VM **debian-12-qcow2-template** se convertirá en un template o plantilla reutilizable.*

***Crear VMs a partir del template***

1. *Clic derecho en el nuevo template **debian-12-qcow2-template \-\>** Clone*   
2. *Configurar **Name:** mi-nueva-vm, **Mode:** full clone.*  
3. *Aparecerá una nueva VM, la cual podemos configurar e iniciar.*

## **7\. Crear un cluster PROXMOX**

*Previamente abrir las interfaces web de los 3 proxmox y loguearse.*

1. `https://<IP-de-Proxmox-1>:8006`  
2. `https://<IP-de-Proxmox-2>:8006`  
3. `https://<IP-de-Proxmox-3>:8006`

***En Proxmox-1***

1. *👉 Clic en la **Datacenter \-\> Cluster \-\> Create Cluster***  
2. *Llenar los datos: **Cluster Name:** mi-cluster (ejemplo)*  
3. *Clic en **Create***  
4. *Clic en **Join information \-\> Copy information***

***En Proxmox 2 y 3***

1. *Previamente configurar la **Red (Bridge)** como en el apartado 3, con la ip de cada proxmox.*  
2. *👉 Clic en la **Datacenter \-\> Cluster \-\> Join Cluster** y pegar el contenido anterior*  
3. *Llenar los datos: **Password:** (root password de proxmox1)*  
4. *Desplegar menú: Link **0:** (seleccionar la interfaz a usar)*  
5. *Clic en **Join***

***En el cluster***

1. *Al ingresar a cualquiera de las interfaces web ya puede administrarse los 3 nodos proxmox del cluster.*

## **8\. Configurar un almacenamiento compartido**

*Los 3 nodos tienen conectado un disco en común, el cual vamos a configurar como nuevo volumen de almacenamiento compartido del clúster.*

***En todos los nodos proxmox***

1. *Ingresar a la **consola** del nodo.*  
2. *Ejecutar el comando para listar los discos conectados: **lsblk***  
3. *Identificar el disco adicional (el que no es del sistema operativo)*  
4. *Crear un volumen físico con el disco a usar, con el comando: **pvcreate /dev/vdb***

***En el nodo 1***

1. *Ingresar a la **consola** del nodo.*  
2. *Ejecutar el comando para crear un volume group: **vgcreate vol1 /dev/vdb***  
3. *Clic en **Datacenter \-\> Storage \-\> Add \-\> LVM***  
4. *Llenar los datos: **ID:** vol1, **Volume group:** vol1, **Shared:** marcar*  
5. *Clic en **Add.***

## **9\. Migración de VMs entre nodos proxmox**

*Para que las Vms puedan habitar cualquier nodo del cluster, los discos de la Vm deben estar ubicado en el volumen de almacenamiento compartido.*

1. *Crear una nueva Vm clonando el template **debian-12-qcow2-template**.*  
2. *Seleccionar las opciones de clonación: **Mode:** Full clone, **Target Storage:** vol01*  
3. *Configurar e iniciar la nueva VM creada (configurar una IP única en la red asignada)*  
4. *Clic derecho en la VM, **Migrate***  
5. *Seleccionar **Target Node** con el nuevo nodo al que se migra la VM.*

## **10\. Configurar HA (alta disponibilidad)**

***Crear grupo de alta disponibilidad***

1. *Clic en **Datacenter \-\> HA \-\> Groups \-\> Create***  
2. *Llenar los datos **ID:** grupo-nodos*  
3. *Tickear todos los nodos que pertenecerán al grupo HA.*  
4. *Clic en **Create.***

***Añadir VMs al grupo HA***

*Se deben añadir VMs que tienen sus discos en almacenamiento compartido.*

1. *Clic en **Datacenter \-\> HA \-\> Add***  
2. *Seleccionar la VM y configurar **Group:** grupo nodos, **Request State:** started (se el estado deseado de la VM es encendido)*  
3. *Clic en **Add.***

## **11\. Simulación de fallos**

*Simularemos una desconexión de un nodo del cluster.*

1. *Migrar la VM configurada en el apartado 11 al nodo 3\.*  
2. *Ingresar a la consola del nodo 3, **Nodo \-\> Shell***  
3. *Simular una desconexión del cluster con el comando: **ifdown vmbr0***  
4. *Esperar unos minutos, la VM que estaba en el nodo 3 debería migrarse automáticamente a otro nodo e iniciarse.*  
5. *El nodo 3 aparecerá desconectado, y se reiniciará.*

## **12\. Configurar Firewall**

Para poder usar el firewall de proxmox en las VMs, se debe habilitar el mismo a varios niveles: datacenter, nodos, VMs e interfaz de red.  
***Nivel datacenter***

1. *Clic en **Datacenter \-\> Firewall \-\> Options \-\> Input Policy:** ACCEPT (esto para que el firewall no bloquee todo por defecto)*  
2. *Clic en **Datacenter \-\> Firewall \-\> Options \-\> Firewall:** Yes (Habilitar el firewall nivel de datacenter).*

***Nivel nodo***

1. *En todos los nodos verificar que el firewall esté activo*  
2. *Clic en **Nodo \-\> Firewall \-\> Options \-\> Firewall:** Yes.*

***Nivel VM y su interfaz de red***

1. *Clic en la **VM \-\>  Firewall \-\> Options \-\> Input Policy:** ACCEPT*  
2. *Clic en la **VM \-\> Firewall \-\> Options \-\> Firewall:** Yes (Habilita firewall a nivel de VM)*  
3. *Clic en la **VM \-\> Hardware \-\> Network Device \-\> Firewall:** Yes (Habilita firewall a nivel de la interfaz de red de la VM)*

***Configurar reglas de firewall en la VM***

1. *Clic en la **VM \-\>  Firewall \-\> Add \-\>***   
- ***Direction:** in*  
- ***Action:** REJECT*  
- ***Protocol:** TCP*  
- ***Dest. port:** 22*  
1. *Clic en **Add***  
2. *Clic en **Enable***

   

