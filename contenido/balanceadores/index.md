---
layout: blog
tittle: Balanceador de carga redundante - HAProxy & KeepAlived
menu:
  - Índice
---

Lo primero que haremos será configurar Haproxy para que actué como balanceador de carga y
Keepalived para que compruebe el estado de nuestra IP virtual (Nuestro recurso en cluster). La
IP que hemos elegido será la 192.168.1.150 (Más tarde servirá para entrar a Horizon).

## HAPROXY

HAProxy es una solución gratuita , muy rápido y fiable que ofrece alta disponibilidad, balanceo
de carga y proxy para TCP y aplicaciones basadas en HTTP . Es especialmente adecuado para
los sitios web de alto tráfico y bastante visitados del mundo. Con los años se ha convertido en
el equilibrador de carga de código abierto estándar por defecto , ahora se incluye con la
mayoría de las principales distribuciones de Linux , y con frecuencia se despliega por defecto
en plataformas de cloud.

Su modo de funcionamiento hace que su integración en arquitecturas existentes sea muy fácil
y sin riesgo , sin dejar de ofrecer la posibilidad de no exponer los servidores web frágiles a la
red , como a continuación:

![HAPROXY](img/haproxy.png)

## KEEPALIVED

Keepalived es un software de enrutamiento escrito en C. El objetivo principal de este proyecto
es proporcionar instalaciones simples y robustas para balanceadores de carga y de alta
disponibilidad para los sistemas Linux y las infraestructuras basadas en Linux. Keepalived
implementa un conjunto de "fichas" para mantener y administrar grupo de servidores en
equilibrio de carga de acuerdo a su "salud" de forma dinámica y adaptativa. Por otra parte, la
alta disponibilidad se consigue mediante el protocolo VRRP. VRRP es un pilar fundamental para
la conmutación por error.

Keepalived es software libre; puedes redistribuirlo y / o modificarlo bajo los términos de la
Licencia Pública General GNU publicada por la Fundación para el Software Libre; ya sea la
versión 2 de la Licencia, o (a su elección) cualquier versión posterior.

## INSTALACIÓN Y CONFIGURACIÓN

Los dos servidores de Proxy son Hera y Afrodita (Ver tabla), estos dos servidores poseen una
interfaz en modo puente para acceder de manera "pública".

La configuración que vamos a realizar se debe de hacer en los dos nodos.

Lo primero será configurar la interfaz de red puente. La añadimos en virt-manager en remoto y
la añadimos en **/etc/network/interfaces**:

~~~
auto eth1
iface eth1 inet static
		address 192.168.1.110
		netmask 255.255.255.0
		gateway 192.168.1.1
		dns-nameserver 8.8.8.8
~~~

Y levantamos la interfaz:

~~~
ifup eth1
~~~

>**ERROR**: Nos encontramos que cuando encendemos las máquinas quedan esperando la
configuración de red desde el servidor DHCP (Cuando tienen configuradas una IP estática),
demorando su arranque bastante tiempo.

>**SOLUCIÓN**: Encontramos la solución:
>http://www.linuxquestions.org/questions/linux-server-73/ubuntu-14-04-incredibly-slow-
boot-without-network-4175510497/

>Procedemos a comentar en el fichero /etc/init/failsafe.conf las dos líneas siguientes:
~~~
$PLYMOUTH message --text="Waiting for network configuration..." || :
sleep 40
$PLYMOUTH message --text="Waiting up to 60 more seconds for network
configuration..." || :
sleep 59
$PLYMOUTH message --text="Booting system without full network
configuration..." || :
~~~

>Y así obtenemos un tiempo de inicio aceptable.
