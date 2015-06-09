---
layout: blog
tittle: Neutron
menu:
  - Índice
---

En este apartado instalaremos Neutron. Este componente no estará en HA. No se podria mantener dos servicios neutron creando redes virtuales, routers, etc... Sin que hubiera ningún tipo de conflicto. Por ello, hemos decidido instalarlo solo en el controlador Zeus y que se encargue de la gestión de las redes.

## NEUTRON

OpenStack Networking (Neutron, anteriormente Quantum) es un sistema para la gestión de redes y direcciones IP. Asegura que la red no presente el problema del cuello de botella o el factor limitante en un despliegue en la nube y ofrece a los usuarios un autoservicio real, incluso a través de sus configuraciones de red.

Neutron proporciona modelos de redes para diferentes aplicaciones o grupos de usuarios. Los modelos estándar incluyen redes planas o VLAN para la separación de los servidores y el tráfico. Gestiona las direcciones IP, lo que permite direcciones IP estáticas o DHCP reservados. Direcciones IP flotantes permiten que el tráfico se redirija dinámicamente a cualquiera de sus recursos informáticos, que permite redirigir el tráfico durante el mantenimiento o en caso de fracaso. Los usuarios pueden crear sus propias redes, controlar el tráfico y conectar los servidores y los dispositivos a una o más redes. Los administradores pueden aprovechar las redes definidas por software de tecnología (SDN) como OpenFlow para permitir altos niveles de multiempresa y escala masiva. 

###INSTALACIÓN Y CONFIGURACIÓN



-----------
Ejemplo con palabras en **Negrita**.

>Ejemplo de texto en letra cursiva

Ejemmplo de insercion imagen:
![nombre](ubicacion de la imagen)

Ejemplo de enlace:
[texto](url)

Ejemplo de tabla:
|NOMBRE MV|FUNCIÓN|MEMORIA RAM|IP|
|:---:|------|------|------|
|**HERA**|Proxy 1|256 MB|192.168.100.10|
|**AFRODITA**|Proxy 2|256 MB|192.168.100.11|
|**ZEUS**|Controlador 1|1 GB|192.168.100.12|
|**HADES**|Controlador 2|1 GB|192.168.100.13|
|**POSEIDON**|Ceph 1|1 GB|192.168.100.14|
|**APOLO**|Ceph 2|1 GB|192.168.100.15|
|**ARTEMISA**|Ceph 3|1 GB|192.168.100.16|
|**ARES**|Computación 1|2 GB|192.168.100.17|
|**ATENEA**|Computación 1|2 GB|192.168.100.18|

## TITULO MAYOR - SECCION

### SUBSECCION

Ejemplo de lista:

+ item 1
+ item 2
+ item 3

Ejemplo para poner codigo:
~~~
aptitude update
aptitude upgrade
aptitude install qemu-kvm libvirt-bin bridge-utils virt-manager
~~~