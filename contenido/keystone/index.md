---
layout: blog
tittle: Keystone y RabbitMQ
menu:
  - Índice
---

En este apartado configuraremos en alta disponibilidad RabbitMQ y el componente de Openstack de identidad Keystone en nuestros nodos controladores Hades y Zeus.

Lo primero en ambos nodos editamos el ficher **/etc/hosts**:

+ Zeus:

~~~
127.0.0.1       localhost
127.0.1.1       zeus
192.168.100.12  zeus
192.168.100.13  hades
~~~

+ Hades:

~~~
127.0.0.1       localhost
127.0.1.1       hades
192.168.100.12  zeus
192.168.100.13  hades
~~~

## RABBITMQ

RabbitMQ es un software de negociación de mensajes. Implemente el estándar AMQP (Advanced Message Queuing Protocol)

El proyecto RabbitMQ consta de diferentes partes:

+ El servidor de intercambio RabbitMQ en sí mismo
+ Pasarelas para los protocolos HTTP, XMPP y STOMP.
+ Bibliotecas de clientes para Java y el framework .NET. (Bibliotecas similares para otros lenguajes se encuentran disponibles por parte de otros proveedores).
+ El plugin Shovel (pala) que se encarga de copiar (replicar) mensajes desde un corredor de mensajes a otros

###INSTALACIÓN Y CONFIGURACIÓN

En ambos nodos instalamos RabbitMQ y ntp:

~~~
apt-get install ntp rabbitmq-server
~~~

A continuación paramos el servicio de rabbit en ambos nodos:

~~~
service rabbitmq-server stop
~~~

Ahora necesitamos copiar por scp de Zeus a Hades una "cookie" de RabbitMQ, para ello necesitamos que en Hades este habilitado el usuario root para conectarnos:

~~~
passwd root
~~~

Ahora desde Zeus copiamos el fichero:

~~~
scp /var/lib/rabbitmq/.erlang.cookie root@192.168.1.36:/var/lib/rabbitmq/.erlang.cookie
~~~

Ahora iniciamos el servicio de rabbit en los dos nodos:

~~~
service rabbitmq-server start
~~~

En Hades configuramos Rabbit con las siguientes instrucciones:

~~~
root@hades:/home/usuario# rabbitmqctl stop_app
Stopping node rabbit@hades ...
...done.
root@hades:/home/usuario# rabbitmqctl join_cluster rabbit@zeus
Clustering node rabbit@hades with rabbit@zeus ...
...done.
root@hades:/home/usuario# rabbitmqctl start_app
Starting node rabbit@hades ...
...done.
root@hades:/home/usuario# rabbitmqctl cluster_status
Cluster status of node rabbit@hades ...
[{nodes,[{disc,[rabbit@hades,rabbit@zeus]}]},
 {running_nodes,[rabbit@zeus,rabbit@hades]},
 {partitions,[]}]
...done.
~~~

En el último comando vemos el estado de nuestro cluster de RabbitMQ.

## KEYSTONE

Keystone es el servicio de identidad de Openstack, actúa como un sistema de autenticación de usuarios asignados a proyectos que pueden acceder. Es compatible con distintas formas de autenticación(usuario, contraseña, tokens)

###INSTALACIÓN Y CONFIGURACIÓN

Instalamos los paquetes necesarios:

~~~
apt-get install keystone python-mysqldb
~~~

En ambos nodos debemos configurar Keystone en el fichero **/etc/keystone/keystone.conf**:

~~~
[DEFAULT]
admin_token=S3CR3TT0K3N
...
rabbit_hosts=192.168.100.12:5672,192.168.100.13:5672
...
[database]
connection = mysql://keystone:asdasd@192.168.1.150/keystone
~~~

Como observamos la base de datos la apuntamos a nuestra IP virtual.

A continuación debemos crear la base de datos para keystone y el usuario correspondiente desde cualquier nodo:

~~~
root@zeus:/home/usuario# mysql -h 192.168.1.150 -u root -p
Enter password: 
Welcome to the MariaDB monitor.  Commands end with ; or \g.
Your MariaDB connection id is 79
Server version: 10.0.17-MariaDB-1~trusty-wsrep mariadb.org binary distribution, wsrep_25.10.r4144

Copyright (c) 2000, 2015, Oracle, MariaDB Corporation Ab and others.

Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.

MariaDB [(none)]> create database keystoe;
Query OK, 1 row affected (0.07 sec)

MariaDB [(none)]> drop database keystoe;
Query OK, 0 rows affected (0.04 sec)

MariaDB [(none)]> create database keystone;
Query OK, 1 row affected (0.05 sec)

MariaDB [(none)]> grant all on keystone.* to keystone@'%' identified by 'asdasd';
Query OK, 0 rows affected (0.04 sec)

MariaDB [(none)]> flush privileges;
Query OK, 0 rows affected (0.00 sec)

MariaDB [(none)]> exit;
Bye
~~~

Ahora en Zeus, debemos reiniciar el servicio y poblar la estructura de la base de datos de keystone:

~~~
service keystone restart
keystone-manage db_sync
~~~

Comprobamos que se han creado todas las tablas:

~~~
MariaDB [(none)]> use keystone;
Reading table information for completion of table and column names
You can turn off this feature to get a quicker startup with -A

Database changed
MariaDB [keystone]> show tables
    -> ;
+-----------------------+
| Tables_in_keystone    |
+-----------------------+
| assignment            |
| credential            |
| domain                |
| endpoint              |
| group                 |
| migrate_version       |
| policy                |
| project               |
| region                |
| role                  |
| service               |
| token                 |
| trust                 |
| trust_role            |
| user                  |
| user_group_membership |
+-----------------------+
16 rows in set (0.00 sec)
~~~

Cuando instalamos Keystone en cada nodo, nos crea un certificado autofirmado para firmar los tokens de autorización. Debemos hacer coincidir estos certificados pasando de Zeus a Hades el suyo:

~~~
scp -r /etc/keystone/ssl root@192.168.1.13:/etc/keystone
~~~

Ahora reiniciamos Keystone en Hades:

~~~
service keystone restart
~~~

Ya tenemos nuestros dos nodos configurados. A continuación debemos irnos a nuestros Proxys Hera y Afrodita y editar el fichero **/etc/haproxy/haproxy.cfg** para que podamos ver el estado de nuestros endpoint de keystone:

~~~
listen keystone_admin 192.168.1.150:35357
        balance source
        option tcpka
        option httpchk
        maxconn 10000
        server zeus 192.168.100.12:35357 check inter 2000 rise 2 fall 5
        server hades 192.168.100.13:35357 check inter 2000 rise 2 fall 5

listen keystone_api 192.168.1.150:5000
        balance source
        option tcpka
        option httpchk
        maxconn 10000
        server zeus 192.168.100.12:5000 check inter 2000 rise 2 fall 5
        server hades 192.168.100.13:5000 check inter 2000 rise 2 fall 5
~~~

Y hacemos un reload de HAproxy:

~~~
service haproxy reload
~~~

Y si accedemos a nuestro panel web de HAproxy:

![STATS](img/keystone_stats.png)

Por último debemos poblar Keystone con los usuarios y servicios, tenants, endpoints, etc... Para hacerlo todos de una vez utilizamos un script en Bash:

**Fichero keystone.sh**:

~~~
#!/bin/bash

#VARIABLES
ADMIN_PASSWORD=asdasd
SERVICE_PASSWORD=asdasd
DEMO_PASSWORD=asdasd
export OS_SERVICE_TOKEN=S3CR3TT0K3N
export OS_SERVICE_ENDPOINT="http://192.168.1.150:35357/v2.0"
SERVICE_TENANT_NAME=service
#
KEYSTONE_REGION=regionOne
KEYSTONE_HOST=192.168.1.150

#FUNCION QUE OBTIENE EL ID MAS RECIENTE
function get_field() {
    while read data; do
        if [ "$1" -lt 0 ]; then
            field="(\$(NF$1))"
        else
            field="\$$(($1 + 1))"
        fi
        echo "$data" | awk -F'[ \t]*\\|[ \t]*' "{print $field}"
    done
}

# TENANTS
ADMIN_TENANT=$(keystone tenant-create --name=admin | grep " id " | get_field 2)
DEMO_TENANT=$(keystone tenant-create --name=demo | grep " id " | get_field 2)
SERVICE_TENANT=$(keystone tenant-create --name=$SERVICE_TENANT_NAME | grep " id " | get_field 2)

# USERS
ADMIN_USER=$(keystone user-create --name=admin --pass="$ADMIN_PASSWORD" --email=admin@olimpo.com | grep " id " | get_field 2)
DEMO_USER=$(keystone user-create --name=demo --pass="$DEMO_PASSWORD" --email=demo@olimpo.com --tenant-id=$DEMO_TENANT | grep " id " | get_field 2)
NOVA_USER=$(keystone user-create --name=nova --pass="$SERVICE_PASSWORD" --tenant-id $SERVICE_TENANT --email=nova@olimpo.com | grep " id " | get_field 2)
GLANCE_USER=$(keystone user-create --name=glance --pass="$SERVICE_PASSWORD" --tenant-id $SERVICE_TENANT --email=glance@olimpo.com | grep " id " | get_field 2)
CINDER_USER=$(keystone user-create --name=cinder --pass="$SERVICE_PASSWORD" --tenant-id $SERVICE_TENANT --email=cinder@olimpo.com | grep " id " | get_field 2)
NEUTRON_USER=$(keystone user-create --name=neutron --pass="$SERVICE_PASSWORD" --tenant-id $SERVICE_TENANT --email=neutron@olimpo.com | grep " id " | get_field 2)


# ROLES
ADMIN_ROLE=$(keystone role-create --name=admin | grep " id " | get_field 2)
MEMBER_ROLE=$(keystone role-create --name=Member | grep " id " | get_field 2)

# ADD ROLES TO USER IN TENANTS
keystone user-role-add --user-id $ADMIN_USER --role-id $ADMIN_ROLE --tenant-id $ADMIN_TENANT
keystone user-role-add --tenant-id $SERVICE_TENANT --user-id $NOVA_USER --role-id $ADMIN_ROLE
keystone user-role-add --tenant-id $SERVICE_TENANT --user-id $GLANCE_USER --role-id $ADMIN_ROLE
keystone user-role-add --tenant-id $SERVICE_TENANT --user-id $CINDER_USER --role-id $ADMIN_ROLE
keystone user-role-add --tenant-id $SERVICE_TENANT --user-id $NEUTRON_USER --role-id $ADMIN_ROLE
keystone user-role-add --tenant-id $DEMO_TENANT --user-id $DEMO_USER --role-id $MEMBER_ROLE

# Create services
COMPUTE_SERVICE=$(keystone service-create --name nova --type compute --description 'OpenStack Compute Service' | grep " id " | get_field 2)
VOLUME_SERVICE=$(keystone service-create --name cinder --type volume --description 'OpenStack Volume Service' | grep " id " | get_field 2)
NETWORKING_SERVICE=$(keystone service-create --name neutron --type network --description 'OpenStack Networking Service' | grep " id " | get_field 2)
IMAGE_SERVICE=$(keystone service-create --name glance --type image --description 'OpenStack Image Service' | grep " id " | get_field 2)
IDENTITY_SERVICE=$(keystone service-create --name keystone --type identity --description 'OpenStack Identity' | grep " id " | get_field 2)

# Create endpoints
keystone endpoint-create --region $KEYSTONE_REGION --service-id $COMPUTE_SERVICE --publicurl 'http://'"$KEYSTONE_HOST"':8774/v2/$(tenant_id)s' --adminurl 'http://'"$KEYSTONE_HOST"':8774/v2/$(tenant_id)s' --internalurl 'http://'"$KEYSTONE_HOST"':8774/v2/$(tenant_id)s'
keystone endpoint-create --region $KEYSTONE_REGION --service-id $VOLUME_SERVICE --publicurl 'http://'"$KEYSTONE_HOST"':8776/v1/$(tenant_id)s' --adminurl 'http://'"$KEYSTONE_HOST"':8776/v1/$(tenant_id)s' --internalurl 'http://'"$KEYSTONE_HOST"':8776/v1/$(tenant_id)s'
keystone endpoint-create --region $KEYSTONE_REGION --service-id $IMAGE_SERVICE --publicurl 'http://'"$KEYSTONE_HOST"':9292' --adminurl 'http://'"$KEYSTONE_HOST"':9292' --internalurl 'http://'"$KEYSTONE_HOST"':9292'
keystone endpoint-create --region $KEYSTONE_REGION --service-id $IDENTITY_SERVICE --publicurl 'http://'"$KEYSTONE_HOST"':5000/v2.0' --adminurl 'http://'"$KEYSTONE_HOST"':35357/v2.0' --internalurl 'http://'"$KEYSTONE_HOST"':5000/v2.0'
keystone endpoint-create --region $KEYSTONE_REGION --service-id $NETWORKING_SERVICE --publicurl 'http://192.168.100.12:9696' --adminurl 'http://192.168.100.12:9696' --internalurl 'http://192.168.100.12:9696'
~~~

Esto lo lanzamos desde Zeus, por ejemplo, y si después lo apagamos y comprobamos en Hades la lista de usuarios de keystone comprobamos que tenemos los datos siempre disponibles:

~~~
root@hades:/home/usuario# keystone user-list
+----------------------------------+--------+---------+-------------------+
|                id                |  name  | enabled |       email       |
+----------------------------------+--------+---------+-------------------+
| 0b26670b9e1b4ef7b1d03722df0109db | admin  |   True  |  admin@olimpo.com |
| 7c80934ddd7849e1b3415a79cf3fd1e6 | cinder |   True  | cinder@olimpo.com |
| 26312174fc934109a88ee94c0ef13cc2 |  demo  |   True  |  demo@olimpo.com  |
| f7d1e36264c244a2bdb28dd2f24fe74d | glance |   True  | glance@olimpo.com |
| 02a85fa306ba41e28b3d2c2432c9a7de |  nova  |   True  |  nova@olimpo.com  |
+----------------------------------+--------+---------+-------------------+
~~~