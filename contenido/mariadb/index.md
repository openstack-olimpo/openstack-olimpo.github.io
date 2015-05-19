---
layout: blog
tittle: MySQL Alta Disponibilidad con Galera
menu:
  - Índice
---

En este apartado configuraremos MySQL en alta disponibilidad con Galera. Utilizaremos
MariaDB que permite esta integración y no podemos con MySQL Server:


## GALERA

Galera permite una sincronización entre un cluster multi-master para bases de datos MySQL.
Sus ventajas descritas en la página oficial:
World’s most advanced features

+ Synchronous replication
+ Active-active multi-master topology
+ Read and write to any cluster node
+ Automatic membership control, failed nodes drop from the cluster
+ Automatic node joining
+ True parallel replication, on row level
+ Direct client connections, native MySQL look & feel
+ No slave lag
+ No lost transactions
+ Both read and write scalability
+ Smaller client latencies


## INSTALACIÓN Y CONFIGURACIÓN

Lo primero será instalar los paquetes necesarios en ambos nodos (Zeus y Hades en este caso),
añadiendo el repositorio para mariadb:


~~~
apt-get install software-properties-common
apt-key adv --recv-keys --keyserver hkp://keyserver.ubuntu.com:80
0xcbcb082a1bb943db
add-apt-repository 'deb http://ftp.osuosl.org/pub/mariadb/repo/10.0/ubuntu
trusty main'
apt-get update
apt-get install mariadb-galera-server
~~~

Ahora debemos crear el fichero de configuración de Galera llamado
**/etc/mysql/conf.d/cluster.cnf**, en ambos nodos:


~~~
[mysqld]
query_cache_size=0
binlog_format=ROW
default-storage-engine=innodb
innodb_autoinc_lock_mode=2
query_cache_type=0
bind-address=0.0.0.0

# Galera Provider Configuration
wsrep_provider=/usr/lib/galera/libgalera_smm.so
#wsrep_provider_options="gcache.size=32G"

# Galera Cluster Configuration
wsrep_cluster_name="olimpo_cluster"
wsrep_cluster_address="gcomm://192.168.100.12,192.168.100.13"

# Galera Synchronization Congifuration
wsrep_sst_method=rsync
#wsrep_sst_auth=user:pass
# Galera Node Configuration

wsrep_node_address="192.168.100.12"
wsrep_node_name="zeus"
~~~


*Advertencia para el segundo nodo: Debemos cambiar las últimas dos líneas en el segundo
nodo con su configuración específica.

A continuación, en ambos nodos también, debemos comentar en el fichero **/etc/mysql/my.cnf**
la siguiente línea para que no nos falle HAproxy más tarde:


~~~
#bind-address= 127.0.0.1
~~~

Debemos para los servicios de mysql en ambos nodos para la siguiente configuraciones:

~~~
Service mysql stop
~~~


Necesitamos copiar el fichero **/etc/mysql/debian.cnf** para que en ambos nodos sean iguales,
lo pasamos del nodo 1 al nodo 2 (Importante: asegurarse de que los permisos finalmente son
los apropiados):

~~~
scp /etc/mysql/debian.cnf usuario@192.168.100.13:/home/usuario/.
~~~

A continuación debemos iniciar un primer nodo de la siguiente forma:

~~~
service mysql start --wsrep-new-cluster
~~~

Una vez inicializado el primero, iniciamos el segundo de manera normal:

~~~
service mysql start
~~~


Entramos, por ejemplo, en Zeus e introducimos lo siguiente:


~~~
grant all on *.* to root@'%' identified by 'asdasd' with grant option;
insert into mysql.user (Host,User) values ('192.168.100.10','haproxy');
insert into mysql.user (Host,User) values ('192.168.100.11','haproxy');
flush privileges;
exit
~~~

Damos por finalizada la configuración de Galera, debemos irnos a los nodos de HAProxy y
añadir la configuración de nuestro BD Cluster. Además necesitamos instalar el cliente de
mysql, ya que HAproxy comprueba la disponibilidad del sistema logueandose en él. Para ello
instalamos en los nodos Hera y Afrodita:

~~~
Apt-get install mysql-client
~~~


Y añadimos la configuración, como ya hemos comentado, en el fichero
**/etc/haproxy/haproxy.cfg:**

~~~
listen galera 192.168.1.150:3306
	balance source
	mode tcp
	option tcpka
	option mysql-check user haproxy
	server zeus 192.168.100.12:3306 check weight 1
	server hades 192.168.100.13:3306 check weight 1
~~~


Finalmente hacemos un reload del servicio:

~~~
service haproxy reload
~~~

Si quisiéramos comprobarlo podríamos entrar desde el anfitrión o desde los nodos
balanceadores a través de la VIP con la siguiente instrucción:

~~~
mysql -h 192.168.1.150 -u root -p
~~~