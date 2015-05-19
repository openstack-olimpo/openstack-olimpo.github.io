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

Una vez editada cada máquina con la RAM que describimos anteriormente, debemos cambiar
en cada máquina los ficheros **/etc/hosts, /etc/hostname y /etc/network/interfaces** para
editar las características de la plantilla:

/etc/hosts:

~~~
IP 		nombre_mv
~~~

/etc/hostname:

~~~
nombre_mv
~~~

/etc/network/interfaces:

~~~
auto eth0
iface eth0 inet static
		address 192.168.100.11
		netmask 255.255.255.0
		network 192.168.100.0
		broadcast 192.168.100.255
		gateway 192.168.100.1
		# dns-* options are implemented by the resolvconf package...
		dns-nameservers 192.168.100.1
~~~

Sincronizamos las fecha y horas de todas las máquinas con un servidor ntp de internet:

~~~
ntpdate 3.es.pool.ntp.org
~~~

Hacemos una snapshot de cada máquina con:

~~~
root@olimpo:/home/usuario/Snapshots# virsh start maquina
root@olimpo:/home/usuario/Snapshots# virsh save maquina nombre_snapshot
root@olimpo:/home/usuario/Snapshots# ls
afrodita_0 ares_0 atenea_0 zeus_0
apolo_0 artemisa_0 hades_0 hera_0 poseidon_0
~~~

Ya tenemos nuestras máquinas listas para iniciar la instalación de Openstack.