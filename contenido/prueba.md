## A New Post

Enter text in [Markdown](http://daringfireball.net/projects/markdown/). Use the toolbar above, or click the **?** button for formatting help.

prueba:

`root@olimpo:/home/usuario/Snapshots# virsh start maquina  root@olimpo:/home/usuario/Snapshots# virsh save maquina nombre_snapshot`

`root@olimpo:/home/usuario/Snapshots# ls afrodita_0 ares_0 atenea_0 zeus_0 apolo_0 artemisa_0 hades_0 hera_0 poseidon_0`
~~~
# $PLYMOUTH message --text="Waiting for network configuration..." || :
# sleep 40
# $PLYMOUTH message --text="Waiting up to 60 more seconds for network configuration..." || :
# sleep 59
# $PLYMOUTH message --text="Booting system without full network configuration..." || :
~~~
~~~
auto eth1
iface eth1 inet static 
		address 192.168.1.110 
		netmask 255.255.255.0 
		gateway 192.168.1.1 
		dns-nameserver 8.8.8.8
~~~
~~~
global chroot /var/lib/haproxy user haproxy group haproxy daemon log 192.168.1.110 local0 stats socket /var/lib/haproxy/stats maxconn 4000 defaults log global mode http #option httplog option dontlognull contimeout 5000 clitimeout 50000 srvtimeout 50000 errorfile 400 /etc/haproxy/errors/400.http errorfile 403 /etc/haproxy/errors/403.http errorfile 408 /etc/haproxy/errors/408.http errorfile 500 /etc/haproxy/errors/500.http errorfile 502 /etc/haproxy/errors/502.http errorfile 503 /etc/haproxy/errors/503.http errorfile 504 /etc/haproxy/errors/504.http listen stats 192.168.1.110:80 mode http stats enable stats uri /stats stats realm HAProxy\ Statistics stats auth admin:password
~~~
