---
layout: blog
tittle: Glance sobre Ceph
menu:
  - Índice
---

En este apartado configuraremos Glance sobre Ceph.


Glance es el componente de Openstack que almacena imágenes. Las imágenes almacenadas se pueden utilizar como una plantilla. También se puede utilizar para almacenar y catalogar copias de seguridad.



Lo primero que vamos a realizar es la instalación de algunos paquetes en **Zeus y Hades** (nodos controladores).

~~~
wget -q -O- 'https://ceph.com/git/?p=ceph.git;a=blob_plain;f=keys/release.asc' | sudo apt-key add -
echo deb http://ceph.com/debian-hammer/ $(lsb_release -sc) main | sudo tee /etc/apt/sources.list.d/ceph.list
sudo apt-get update && sudo apt-get install ceph-common python-ceph glance python-glanceclient
~~~


Una vez que esté instalado en ambos nodos vamos a volver a nuestro cluster Ceph y crear un usuario para que lo usen los nodos controladores:

~~~
ceph auth get-or-create client.olimpo mon 'allow r' osd 'allow class-read object_prefix rbd_children, allow rwx pool=datastore'

ceph auth get-or-create client.olimpo | ssh 192.168.100.12 sudo tee /etc/ceph/ceph.client.olimpo.keyring
ceph auth get-or-create client.olimpo | ssh 192.168.100.13 sudo tee /etc/ceph/ceph.client.olimpo.keyring
ssh 192.168.100.12 sudo chmod +r /etc/ceph/ceph.client.olimpo.keyring
ssh 192.168.100.13 sudo chmod +r /etc/ceph/ceph.client.olimpo.keyring
~~~

Con esto crearemos un usuario olimpo y pasamos la key a nuestros nodos controladores.


Ahora pasamos la configuración de Ceph a los nodos controladores

~~~
ssh 192.168.100.12 sudo tee /etc/ceph/ceph.conf < /etc/ceph/ceph.conf
ssh 192.168.100.13 sudo tee /etc/ceph/ceph.conf < /etc/ceph/ceph.conf
~~~


Una vez pasada la configuración vamos a modificar la configuración de Ceph:

**/etc/glance/glance-api.conf**

~~~
[DEFAULT]
default_store = rbd
show_image_direct_url = True
registry_host = 192.168.1.150
registry_port = 9191
registry_client_protocol = http
auth_strategy = keystone
rabbit_hosts = 192.168.100.12,192.168.100.13
rabbit_port = 5672
rabbit_use_ssl = false
rabbit_userid = guest
rabbit_password = guest
rabbit_virtual_host = /
rabbit_notification_exchange = glance
rabbit_notification_topic = notifications
rabbit_durable_queues = False
rbd_store_ceph_conf = /etc/ceph/ceph.conf
rbd_store_user = olimpo
rbd_store_pool = datastore

[database]
backend = sqlalchemy
connection = mysql://glance:asdasd@192.168.1.150/glance

[keystone_authtoken]
auth_host = 192.168.1.150
auth_port = 35357
auth_protocol = http
admin_tenant_name = service
admin_user = glance
admin_password = asdasd
~~~

**/etc/glance/glance-registry.conf**

~~~
[database]
backend = sqlalchemy
connection = mysql://glance:asdasd@192.168.1.150/glance

[keystone_authtoken]
auth_host = 192.168.1.150
auth_port = 35357
auth_protocol = http
admin_tenant_name = service
admin_user = glance
admin_password = asdasd
~~~

**/etc/glance/glance-cache.conf**

~~~
registry_host = 192.168.1.150
~~~


Una vez finalizada la configuración de glance vamos a crear la base de datos:

~~~
mysql -h 192.168.1.150 -u root -p
create database glance character set utf8 collate utf8_general_ci;
grant all on glance.* to glance@'%' identified by 'asdasd';
flush privileges;
exit
~~~

Reiniciamos los servicios de glance y poblamos la base de datos **sólo en Zeus** (primer nodo controlador):

~~~
service glance-api restart
service glance-registry restart
glance-manage db_sync
~~~

Y finalmente reiniciamos también en Hades (segundo nodo controlador):

~~~
service glance-api restart
service glance-registry restart
~~~


En este punto ya tendríamos configurado Glance sobre Ceph, ahora vamos a añadirlo al haproxy:

En Hera y Afrodita añadimos lo siguiente:


**/etc/haproxy/haproxy.cfg**

~~~
listen glance-api 192.168.1.150:9292
        balance source
        option tcpka
        option httpchk
        maxconn 10000
        server zeus 192.168.100.12:9292 check inter 2000 rise 2 fall 5
        server hades 192.168.1.13:9292 check inter 2000 rise 2 fall 5

listen glance-registry 192.168.1.150:9191
        balance source
        option tcpka
        option httpchk
        maxconn 10000
        server zeus 192.168.100.12:9191 check inter 2000 rise 2 fall 5
        server hades 192.168.100.13:9191 check inter 2000 rise 2 fall 5
~~~

Y reiniciamos el servicio en ambas máquinas:

~~~
service haproxy reload**
~~~

Para comprobar que funciona correctamente vamos a añadir una imagen:

~~~
 wget http://download.cirros-cloud.net/0.3.4/cirros-0.3.4-x86_64-disk.img
 glance image-create --name CirrOS-3.4 --is-public=true --disk-format=qcow2 --container-format=bare < cirros-0.3.4-x86_64-disk.img
~~~

 Y comprobamos que se ha añadido correctamente:

~~~
+--------------------------------------+-----------------+-------------+------------------+-----------+--------+
| ID                                   | Name            | Disk Format | Container Format | Size      | Status |
+--------------------------------------+-----------------+-------------+------------------+-----------+--------+
| 7b15fbd6-bbf3-4a13-81be-85f2384e63c5 | CirrOS-3.4      | qcow2       | bare             | 13287936  | active |
| 51d93ce4-5f27-432e-a53d-77959bac82e6 | Debian-8-Jessie | qcow2       | bare             | 468853760 | active |
+--------------------------------------+-----------------+-------------+------------------+-----------+--------+
~~~

Hemos añadido también una imagen Debian Jessie para comprobar el espacio usado de Ceph:

~~~
root@poseidon:/home/usuario# ceph -w
    cluster d63cfea6-113e-42be-9e42-94fa8ae009f1
     health HEALTH_OK
     monmap e1: 1 mons at {poseidon=192.168.100.14:6789/0}
            election epoch 1, quorum 0 poseidon
     osdmap e25: 3 osds: 3 up, 3 in
      pgmap v151: 214 pgs, 2 pools, 459 MB data, 65 objects
            **1492 MB used, 38410 MB / 39902 MB avail**
                 214 active+clean
~~~

Podemos comprobar que ya tenemos 1492 MB usados habiendo añadido dos imagenes a Glance.


Finalmente comprobamos nuestro panel web de haproxy:

![STATS](img/glance_stats.png)