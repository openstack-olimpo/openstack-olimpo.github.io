---
layout: blog
tittle: Nova - Nodos controladores
menu:
  - Índice
---

En este apartado vamos a configurar **Nova** en los nodos **controladores** llamados Zeus y Hades. Más adelante lo configuraremos en los nodos de computación que serán los que levanten las instancias.

## NOVA

Nova es un controlador de estructura cloud computing, que es la parte principal de un sistema de IaaS. Está diseñado para gestionar y automatizar los pools de los recursos del equipo y puede trabajar con tecnologías ampliamente disponibles de virtualización. KVM y Xen son las opciones disponibles para la tecnología de hipervisor, junto con la tecnología Hyper-V, la tecnología vSphere de VMware y la tecnología de contenedores Linux como LXC.

###INSTALACIÓN Y CONFIGURACIÓN

Lo primero será instalar los paquetes necesarios en ambos nodos:

~~~
apt-get install nova-api nova-cert nova-conductor nova-consoleauth nova-novncproxy nova-scheduler python-novaclient
~~~

Ahora debemos editar el fichero **/etc/nova/nova.conf** también en ambos nodos:

~~~
[DEFAULT]
dhcpbridge_flagfile=/etc/nova/nova.conf
dhcpbridge=/usr/bin/nova-dhcpbridge
logdir=/var/log/nova
state_path=/var/lib/nova
lock_path=/var/lock/nova
force_dhcp_release=True
iscsi_helper=tgtadm
libvirt_use_virtio_for_bridges=True
connection_type=libvirt
root_helper=sudo nova-rootwrap /etc/nova/rootwrap.conf
verbose=True
ec2_private_dns_show_ip=True
api_paste_config=/etc/nova/api-paste.ini
volumes_path=/var/lib/nova/volumes
enabled_apis=ec2,osapi_compute,metadata

rpc_backend = nova.openstack.common.rpc.impl_kombu
rabbit_hosts = 192.168.100.12,192.168.100.13
my_ip = 192.168.1.12

vncserver_listen = 192.168.1.12
vncserver_proxyclient_address = 192.168.1.150
auth_strategy = keystone

[database]
connection = mysql://nova:asdasd@192.168.1.150/nova

[keystone_authtoken]
auth_uri = http://192.168.1.150:5000
auth_host = 192.168.1.150
auth_port = 35357
auth_protocol = http
admin_tenant_name = service
admin_user = nova
admin_password = asdasd
~~~
\* Advertencia: La linea my_ip debemos colocar la IP de cada nodo en su propio fichero.

Ahora desde cualquier nodo vamos a crear la base de datos para nova:

~~~
create database nova character set utf8 collate utf8_general_ci;
grant all on nova.* to nova@'%' identified by 'asdasd';
flush privileges;
exit;
~~~

Ahora desde Zeus poblamos esta base de datos:

~~~
nova-manage db sync
~~~

Y reiniciamos los servicios pertinentes en ambos nodos:

~~~
service nova-api restart
service nova-cert restart
service nova-consoleauth restart
service nova-scheduler restart
service nova-conductor restart
service nova-novncproxy restart
~~~

En este punto ya tenemos configurado nova en los nodos controladores. Ahora en nuestros proxys Hera y Afrodita vamos a añadir los siguientes recursos como hicimos con los anteriores.

Para ello, añadimos al fichero **/etc/haproxy/haproxy.cfg** los siguientes apartados:

~~~
listen nova_osapi 192.168.1.150:8774
        balance source
        option tcpka
        option httpchk
        maxconn 10000
        server zeus 192.168.100.12:8774 check inter 2000 rise 2 fall 5
        server hades 192.168.100.13:8774 check inter 2000 rise 2 fall 5

listen nova_metadata 192.168.1.150:8775
        balance source
        option tcpka
        option httpchk
        maxconn 10000
        server zeus 192.168.100.12:8775 check inter 2000 rise 2 fall 5
        server hades 192.168.100.13:8775 check inter 2000 rise 2 fall 5

listen novnc 192.168.1.150:6080
        balance source
        option tcpka
        maxconn 10000
        server zeus 192.168.100.12:6080 check inter 2000 rise 2 fall 5
        server hades 192.168.100.13:6080 check inter 2000 rise 2 fall 5
~~~

Y reiniciamos el servicio en ambos nodos todo esto:

~~~
service haproxy restart
~~~

Llegados a este punto podemos comprobar como nova esta funcionando, por ejemplo, con (Aunque no tenemos ninguna instancia creada todavía):

~~~
root@zeus:~# nova list
+----+------+--------+------------+-------------+----------+
| ID | Name | Status | Task State | Power State | Networks |
+----+------+--------+------------+-------------+----------+
+----+------+--------+------------+-------------+----------+
~~~

Y ademas en haproxy nos muestra nuestras api funcionando:

![STATS](img/novacontrolador_stats.png)