---
layout: blog
tittle: Cinder sobre Ceph
menu:
  - Índice
---

En este apartado configuraremos Cinder sobre Ceph.

## CINDER

Cinder es el componente de Openstack que ofrece dispositivos de bloques.

###INSTALACIÓN Y CONFIGURACIÓN

Lo primero que vamos a realizar es la instalación de algunos paquetes en **Zeus y Hades** (nodos controladores).

~~~
sudo apt-get update && sudo apt-get install cinder-api cinder-scheduler cinder-volume
~~~


Ahora vamos a modificar la configuración de Ceph:

**/etc/cinder/cinder.conf**

~~~
volume_driver=cinder.volume.drivers.rbd.RBDDriver
rbd_pool=datastore
rbd_ceph_conf=/etc/ceph/ceph.conf
rbd_flatten_volume_from_snapshot=false
rbd_max_clone_depth=5
rbd_user=olimpo
glance_api_version=2

rpc_backend = cinder.openstack.common.rpc.impl_kombu
rabbit_hosts = 192.168.100.12,192.168.100.13
rabbit_port = 5672

glance_host = 192.168.1.150

[database]
connection = mysql://cinder:asdasd@192.168.1.150/cinder

[keystone_authtoken]
auth_uri = http://192.168.1.150:5000
auth_host = 192.168.1.150
auth_port = 35357
auth_protocol = http
admin_tenant_name = service
admin_user = cinder
admin_password = asdasd
~~~

Una vez finalizada la configuración de Cinder vamos a crear la base de datos:

~~~
mysql -h 192.168.1.150 -u root -p
create database cinder character set utf8 collate utf8_general_ci;
grant all on cinder.* to cinder@'%' identified by 'asdasd';
flush privileges;
exit
~~~

Reiniciamos los servicios de cinder y poblamos la base de datos **sólo en Zeus** (primer nodo controlador):

~~~
cinder-manage db sync
service cinder-api restart
service cinder-scheduler restart
service cinder-volume restart
~~~

Y finalmente reiniciamos también en Hades (segundo nodo controlador):

~~~
service cinder-api restart
service cinder-scheduler restart
service cinder-volume restart
~~~


En este punto ya tendríamos configurado Cinder sobre Ceph, ahora vamos a añadirlo al haproxy:

En Hera y Afrodita añadimos lo siguiente:


**/etc/haproxy/haproxy.cfg**

~~~
listen cinder_api 192.168.1.150:8776
  balance  source
  option  tcpka
  option  httpchk
  option  tcplog
  server zeus 192.168.100.12:8776  check inter 2000 rise 2 fall 5
  server hades 192.168.100.13:8776  check inter 2000 rise 2 fall 5

~~~

Y reiniciamos el servicio en ambas máquinas:

~~~
service haproxy reload
~~~

Para comprobar que funciona correctamente creando un volumen:

~~~
cinder create --display-name testVolume 1
~~~

 Y comprobamos que se ha creado correctamente:

~~~
root@zeus:~# cinder list
+--------------------------------------+-----------+--------------+------+-------------+----------+-------------+
|                  ID                  |   Status  | Display Name | Size | Volume Type | Bootable | Attached to |
+--------------------------------------+-----------+--------------+------+-------------+----------+-------------+
| 60f27183-fd3b-41bc-94d4-f66dfcac5fe6 | available |  testVolume  |  1   |     None    |  false   |             |
+--------------------------------------+-----------+--------------+------+-------------+----------+-------------+
~~~

Finalmente comprobamos nuestro panel web de haproxy:

![STATS](img/cinder_stats.png)