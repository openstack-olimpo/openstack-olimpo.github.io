---
layout: blog
tittle: Ceph
menu:
  - Índice
---

En esta sección instalaremos y configuraremos un **cluster de almacenamiento con Ceph**. Esta configuración la realizaremos sobre los tres nodos que preparamos para Ceph: Poseidon, Apolo y Artemisa. Estos nodos tienen configurado un segundo disco cada uno (/dev/vdb). Estos discos son volumenes lógicos del disco físico SSD de la máquina anfitriona.

Lo primero, debemos editar el fichero **/etc/hosts** en cada nodo:

+ Poseidon:

~~~
127.0.0.1       localhost
192.168.100.14	poseidon
192.168.100.15  apolo
192.168.100.16  artemisa
~~~

+ Apolo:

~~~
127.0.0.1       localhost
192.168.100.14	poseidon
192.168.100.15  apolo
192.168.100.16  artemisa
~~~

+ Artemisa:

~~~
127.0.0.1       localhost
192.168.100.14	poseidon
192.168.100.15  apolo
192.168.100.16  artemisa
~~~

## CEPH

Ceph es un sistema de almacenamiento de objetos distribuido y también sistema de ficheros diseñado para brindar un excelente rendimiento, fiablidad y escalabilidad. Esta diseñado por la comunidad. Ceph es software libre y gratuito, y siempre lo será.

### INSTALACIÓN Y CONFIGURACIÓN

Ceph incluye una herramienta de despliegue llamada **ceph-deploy** que nos facilita la instalación desde un nodo administrador al resto de nodos. Actualmente, la versión de Ceph estable se denomina **Hammer** y es la que usaremos. Añadiremos el repo y la key asociada. Para ello en el nodo que seleccionemos como admin ejecutamos (En nuestro caso Poseidon):

~~~
wget -q -O- 'https://ceph.com/git/?p=ceph.git;a=blob_plain;f=keys/release.asc' | sudo apt-key add -
echo deb http://ceph.com/debian-hammer/ $(lsb_release -sc) main | sudo tee /etc/apt/sources.list.d/ceph.list
sudo apt-get update && sudo apt-get install ceph-deploy
~~~

Una vez completado, definiremos nuestro nuevo cluster. Crearemos un directorio y definiremos los nodos de los que dispondrá nuestro cluster y cuál será nuestro nodo monitor (Poseidon):

~~~
mkdir ceph-cluster
cd ceph-cluster
ceph-deploy new poseidon
~~~

Antes de seguir debemos editar el fichero **ceph.conf** de esta carpeta y modificar estas dos líneas:

~~~
mon_host = 192.168.100.14
public_network = 192.168.100.0/24
~~~

Y continuamos con las siguientes instrucciones:

~~~
ceph-deploy install poseidon apolo artemisa
ceph-deploy --overwrite-conf mon create-initial
ceph-deploy osd prepare poseidon:/dev/vdb apolo:/dev/vdb artemisa:/dev/vdb
ceph-deploy osd activate poseidon:/dev/vdb1 apolo:/dev/vdb1 artemisa:/dev/vdb1
chmod +r /etc/ceph/ceph.client.admin.keyring
scp /etc/ceph/ceph.client.admin.keyring apolo:/etc/ceph
scp /var/lib/ceph/bootstrap-osd/ceph.keyring apolo:/var/lib/ceph/bootstrap-osd
scp /etc/ceph/ceph.client.admin.keyring artemisa:/etc/ceph
scp /var/lib/ceph/bootstrap-osd/ceph.keyring artemisa:/var/lib/ceph/bootstrap-osd
~~~

Y ya podemos comprobar el estado de nuestro cluster:

~~~
root@poseidon:/home/usuario/ceph_cluster# ceph health
HEALTH_OK
~~~

Por último vamos a crear el datastore que utilizaremos para glance y cinder:

~~~
ceph osd pool create datastore 150
~~~

Lo configuramos con 150 "placements groups", esto es invisible para el usuario de ceph y tiene relación con el como se almacenan los objetos en el cluster. En un entorno de producción sería mejor aplicar más de estos grupos.

Y vemos como efectivamente tenemos nuestro cluster optimo para su uso:

~~~
root@poseidon:/home/usuario/ceph_cluster# ceph -w
    cluster d63cfea6-113e-42be-9e42-94fa8ae009f1
     health HEALTH_OK
     monmap e1: 1 mons at {poseidon=192.168.100.14:6789/0}
            election epoch 1, quorum 0 poseidon
     osdmap e25: 3 osds: 3 up, 3 in
      pgmap v151: 214 pgs, 2 pools, 459 MB data, 65 objects
            1492 MB used, 38410 MB / 39902 MB avail
                 214 active+clean

2015-06-06 13:02:20.319501 mon.0 [INF] pgmap v151: 214 pgs: 214 active+clean; 459 MB data, 1492 MB used, 38410 MB / 39902 MB avail
~~~