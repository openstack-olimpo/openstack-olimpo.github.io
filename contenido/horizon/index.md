---
layout: blog
tittle: Horizon
menu:
  - Índice
---

En este apartado vamos a configurar **Horizon** en los nodos **controladores**. 

## HORIZON

El Dashboard de OpenStack (Horizont) proporciona a los administradores y usuarios una interfaz gráfica para el acceso, la provisión y automatización de los recursos basados ​​en la nube. El diseño permite que los productos y servicios de terceros, tales como la facturación, el monitoreo y las herramientas de gestión adicionales. El Dashboard es sólo una forma de interactuar con los recursos de OpenStack. Los desarrolladores pueden automatizar el acceso o construir herramientas para gestionar sus recursos mediante la API nativa de OpenStack o la API de compatibilidad EC2.

###INSTALACIÓN Y CONFIGURACIÓN

Lo primero que vamos a realizar es la instalación de los paquetes necesarios y, también, borraremos el paquete con el dashboard para horizon de ubuntu:

~~~
apt-get install apache2 memcached libapache2-mod-wsgi openstack-dashboard
apt-get remove --purge openstack-dashboard-ubuntu-theme
~~~

Ahora necesitamos editar el fichero **/etc/openstack-dashboard/local_settings.py** y cambiaremos cualquier referencia a la IP 127.0.0.1 por nuestra VIP(192.168.1.150)

~~~
...
CACHES = {
   'default': {
       'BACKEND' : 'django.core.cache.backends.memcached.MemcachedCache',
       'LOCATION' : '192.168.1.150:11211',
   }
}
...
OPENSTACK_HOST = "192.168.1.150"
...
~~~

Ahora en el fichero **/etc/memcached.conf** cambiaremos la IP de listening por la IP del controlador en el que estemos en este momento. (En este caso Zeus, 192.168.1.12):

~~~
...
-l 192.168.1.12
...
~~~

Y reiniciamos los servicios:

~~~
service apache2 restart
service memcached restart
~~~

A continuación, como hacemos con todos los anteriores, vamos añadir este recurso a nuestros balanceadores en nuestro fichero **/etc/haproxy/haproxy.cfg**:

~~~
listen dashboard 192.168.1.150:80
        balance  source
        capture  cookie vgnvisitor= len 32
        cookie  SERVERID insert indirect nocache
        mode  http
        option  forwardfor
        option  httpchk
        option  httpclose
        rspidel  ^Set-cookie:\ IP=
        server node1 192.168.1.12:80 cookie control01 check inter 2000 rise 2 fall 5
        server node2 192.168.1.13:80 cookie control02 check inter 2000 rise 2 fall 5

listen memcached 192.168.1.150:11211
        balance source
        option tcpka
        option httpchk
        maxconn 10000
        server node1 192.168.1.12:11211 check inter 2000 rise 2 fall 5
        server node2 192.168.1.13:11211 check inter 2000 rise 2 fall 5
~~~

Y recargamos:

~~~
service haproxy reload
~~~

Después de esto tenemos acceso a nuestro horizon por la urel:

http://192.168.1.150/horizon