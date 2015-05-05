---
layout: blog
tittle: Entorno de despliegue - Máquinas virtuales para el cloud
menu:
  - Índice
---

En esta tabla e imagen detallamos el nombre de las máquinas virtuales creadas, su función
dentro del cloud, su memoria RAM y su IP, todas ellas son Ubuntu Server 14.04.2 LTS y
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

## Instalación de KVM

Mediante repositorio del sistema:
~~~
:::bash
aptitude update
aptitude upgrade
aptitude install qemu-kvm libvirt-bin bridge-utils virt-manager
~~~

Y con esto tendríamos todos los paquetes necesarios.