---
layout: blog
tittle: Entorno de despliegue - Máquinas virtuales para el cloud
menu:
  - Índice
---

En esta tabla e imagen detallamos el nombre de las máquinas virtuales creadas, su función
dentro del cloud, su memoria RAM y su IP, todas ellas son **Ubuntu Server 14.04.2 LTS** y
virtualizadas sobre KVM:

|NOMBRE MV|FUNCIÓN|MEMORIA RAM|IP|
|:---:|------|------|------|
|**HERA**|Proxy 1|256 MB|192.168.100.10|
|**AFRODITA**|Proxy 2|256 MB|192.168.100.11|
|**ZEUS**|Controlador 1|1 GB|192.168.100.12|
|**HADES**|Controlador 2 1|1 GB|192.168.100.13|
|**POSEIDON**|Ceph 1|1 GB|192.168.100.14|
|**APOLO**|Ceph 2|1 GB|192.168.100.15|
|**ARTEMISA**|Ceph 3|1 GB|192.168.100.16|
|**ARES**|Computación 1|2 GB|192.168.100.17|
|**ATENEA**|Computación 1|2 GB|192.168.100.18|

**IMAGEN DEL PROYECTO**

## INSTALACIÓN DE KVM

Mediante repositorio del sistema:

~~~
aptitude update
aptitude upgrade
aptitude install qemu-kvm libvirt-bin bridge-utils virt-manager
~~~

Y con esto tendríamos todos los paquetes necesarios.

## CONFIGURACIÓN Y CREACIÓN DE LAS MÁQUINAS VIRTUALES

Primero crearemos el bridge para conectar las máquinas, para ello editamos el fichero
**/etc/network/interfaces**:

~~~
# The primary network interface
iface eth0 inet static
auto br0
iface br0 inet static
	bridge_ports eth0
	address 192.168.1.100
	netmask 255.255.255.0
	gateway 192.168.1.1
~~~

Creamos una máquina virtual llamada Plantilla con **Ubuntu Server 14.04.2 LTS** y la clonamos
para las 11, todo este proceso lo hicimos con virt-manager:

~~~
root@olimpo:/home/usuario# virsh list --all
Id		Name				State
----------------------------------------------------
-		Afrodita 			shut off
-		Apolo 				shut off
-		Ares 				shut off
-		Artemisa 			shut off
-		Atenea 				shut off
-		Hades 				shut off
-		Hera 				shut off
-		Poseidon 			shut off
-		Zeus 				shut off
~~~