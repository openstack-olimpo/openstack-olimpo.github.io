---
layout: blog
tittle: Nova - Nodos de computación
menu:
  - Índice
---
En este apartado vamos a configurar **Nova** en los nodos de **computación** llamados Ares y Atenea. Serán los encargados de levantar las instancias de nuestro cloud. Necesitarán tener instalados Nova y los agentes "clientes" de Neutron.

## NOVA

Nova es un controlador de estructura cloud computing, que es la parte principal de un sistema de IaaS. Está diseñado para gestionar y automatizar los pools de los recursos del equipo y puede trabajar con tecnologías ampliamente disponibles de virtualización. KVM y Xen son las opciones disponibles para la tecnología de hipervisor, junto con la tecnología Hyper-V, la tecnología vSphere de VMware y la tecnología de contenedores Linux como LXC.

###INSTALACIÓN Y CONFIGURACIÓN

Todos estos pasos los haremos en ambos nodos de computación.

Lo primero que haremos será instalar los paquetes necesarios:

~~~
apt-get install nova-compute-qemu python-guestfs
~~~

Ahora editamos el fichero **/etc/nova/nova.conf**:

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
auth_strategy = keystone
rpc_backend = rabbit
rabbit_hosts = 192.168.100.12,192.168.100.13
rabbit_password = guest
my_ip = 192.168.100.17
vnc_enabled = True
vncserver_listen = 0.0.0.0
vncserver_proxyclient_address = 192.168.100.17
novncproxy_base_url = http://192.168.1.150:6080/vnc_auto.html
glance_host = 192.168.1.150
network_api_class = nova.network.neutronv2.api.API
neutron_url = http://192.168.100.12:9696
neutron_auth_strategy = keystone
neutron_admin_tenant_name = service
neutron_admin_username = neutron
neutron_admin_password = asdasd
neutron_admin_auth_url = http://192.168.100.12:35357/v2.0
linuxnet_interface_driver = nova.network.linux_net.LinuxOVSInterfaceDriver
firewall_driver = nova.virt.firewall.NoopFirewallDriver
security_group_api = neutron
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

Y reiniciamos el servicio:

~~~
service nova-compute restart
~~~

## NEUTRON

OpenStack Networking (Neutron, anteriormente Quantum) es un sistema para la gestión de redes y direcciones IP. Asegura que la red no presente el problema del cuello de botella o el factor limitante en un despliegue en la nube y ofrece a los usuarios un autoservicio real, incluso a través de sus configuraciones de red.

Neutron proporciona modelos de redes para diferentes aplicaciones o grupos de usuarios. Los modelos estándar incluyen redes planas o VLAN para la separación de los servidores y el tráfico. Gestiona las direcciones IP, lo que permite direcciones IP estáticas o DHCP reservados. Direcciones IP flotantes permiten que el tráfico se redirija dinámicamente a cualquiera de sus recursos informáticos, que permite redirigir el tráfico durante el mantenimiento o en caso de fracaso. Los usuarios pueden crear sus propias redes, controlar el tráfico y conectar los servidores y los dispositivos a una o más redes. Los administradores pueden aprovechar las redes definidas por software de tecnología (SDN) como OpenFlow para permitir altos niveles de multiempresa y escala masiva. 


###INSTALACIÓN Y CONFIGURACIÓN

Aquí instalaremos los componentes necesarios de neutron para nuestras máquinas de computación. Todos estos pasos también los haremos en ambos nodos de computación.

Lo primero será editar el fichero **/etc/sysctl.conf** para descomentar y/o añadir las siguientes lineas:

~~~
net.ipv4.conf.all.rp_filter=0
net.ipv4.conf.default.rp_filter=0
net.bridge.bridge-nf-call-arptables=1
net.bridge.bridge-nf-call-iptables=1
net.bridge.bridge-nf-call-ip6tables=1
~~~

Para hacer efectivos los cambios ejecutamos:

~~~
sysctl -p
~~~
A continuación, necesitamos instalar los siguientes paquetes:

~~~
apt-get install neutron-common neutron-plugin-ml2 neutron-plugin-openvswitch-agent
~~~

Una vez instalados, editamos el fichero **/etc/neutron/neutron.conf**:

~~~
[DEFAULT]
state_path = /var/lib/neutron
lock_path = $state_path/lock
core_plugin = ml2
service_plugins = router
auth_strategy = keystone
allow_overlapping_ips = True
rpc_backend = neutron.openstack.common.rpc.impl_kombu
rabbit_password = guest
rabbit_hosts = 192.168.100.12,192.168.100.13
notification_driver = neutron.openstack.common.notifier.rpc_notifier
[quotas]
[agent]
root_helper = sudo /usr/bin/neutron-rootwrap /etc/neutron/rootwrap.conf
[keystone_authtoken]
auth_uri = http://192.168.1.150:5000
auth_host = 192.168.1.150
auth_port = 35357
auth_protocol = http
admin_tenant_name = service
admin_user = neutron
admin_password = asdasd
[database]
connection = mysql://neutron:asdasd@192.168.1.150/neutron
[service_providers]
service_provider=LOADBALANCER:Haproxy:neutron.services.loadbalancer.drivers.haproxy.plugin_driver.HaproxyOnHostPluginDriver:default
service_provider=VPN:openswan:neutron.services.vpn.service_drivers.ipsec.IPsecVPNDriver:default
~~~

También necesitamos editar el fichero **/etc/neutron/plugins/ml2/ml2_conf.ini**:

~~~
[ml2]
type_drivers = gre
tenant_network_types = gre
mechanism_drivers = openvswitch
[ml2_type_flat]
[ml2_type_vlan]
[ml2_type_gre]
tunnel_id_ranges = 1:1000
[ml2_type_vxlan]
[securitygroup]
firewall_driver = neutron.agent.linux.iptables_firewall.OVSHybridIptablesFirewallDriver
enable_security_group = True
[ovs]
local_ip = 192.168.100.17
tunnel_type = gre
enable_tunneling = True
~~~

Por último, reiniciamos todos los servicios implicados:

~~~
service nova-compute restart
service neutron-plugin-openvswitch-agent restart
service openvswitch-switch restart
~~~