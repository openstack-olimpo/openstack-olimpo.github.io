---
layout: blog
tittle: Neutron
menu:
  - Índice
---

En este apartado instalaremos Neutron. Este componente no estará en HA. No se podria mantener dos servicios neutron creando redes virtuales, routers, etc... Sin que hubiera ningún tipo de conflicto. Por ello, hemos decidido instalarlo solo en el controlador Zeus y que se encargue de la gestión de las redes.

## NEUTRON

OpenStack Networking (Neutron, anteriormente Quantum) es un sistema para la gestión de redes y direcciones IP. Asegura que la red no presente el problema del cuello de botella o el factor limitante en un despliegue en la nube y ofrece a los usuarios un autoservicio real, incluso a través de sus configuraciones de red.

Neutron proporciona modelos de redes para diferentes aplicaciones o grupos de usuarios. Los modelos estándar incluyen redes planas o VLAN para la separación de los servidores y el tráfico. Gestiona las direcciones IP, lo que permite direcciones IP estáticas o DHCP reservados. Direcciones IP flotantes permiten que el tráfico se redirija dinámicamente a cualquiera de sus recursos informáticos, que permite redirigir el tráfico durante el mantenimiento o en caso de fracaso. Los usuarios pueden crear sus propias redes, controlar el tráfico y conectar los servidores y los dispositivos a una o más redes. Los administradores pueden aprovechar las redes definidas por software de tecnología (SDN) como OpenFlow para permitir altos niveles de multiempresa y escala masiva. 

###INSTALACIÓN Y CONFIGURACIÓN

Lo primero que haremos será crear la base de datos para neutron:

~~~
create database neutron character set utf8 collate utf8_general_ci;
grant all on neutron.* to neutron@'%' identified by 'asdasd';
flush privileges;
~~~

SCRIPT O A MANO¿?

~~~
keystone user-create --name neutron --pass asdasd --email neutron@olimpo.com

keystone user-role-add --user neutron --tenant service --role admin

keystone service-create --name neutron --type network --description "OpenStack Networking"

keystone endpoint-create \
  --service-id $(keystone service-list | awk '/ network / {print $2}') \
  --publicurl http://192.168.100.12:9696 \
  --adminurl http://192.168.100.12:9696 \
  --internalurl http://192.168.100.12:9696
~~~

Ahora instalaremos los paquetes necesarios:

~~~
apt-get install neutron-server neutron-plugin-ml2 neutron-plugin-openvswitch-agent neutron-l3-agent neutron-dhcp-agent
~~~

Lo primero será descomentar y añadir al fichero **/etc/sysctl.conf** las siguientes ĺineas:

~~~
net.ipv4.ip_forward=1
net.ipv4.conf.all.rp_filter=0
net.ipv4.conf.default.rp_filter=0
net.bridge.bridge-nf-call-arptables=1
net.bridge.bridge-nf-call-iptables=1
net.bridge.bridge-nf-call-ip6tables=1
~~~

Y configuramos el fichero **/etc/neutron/neutron.conf**:

~~~
[DEFAULT]
lock_path = $state_path/lock
core_plugin = neutron.plugins.ml2.plugin.Ml2Plugin
service_plugins = neutron.services.loadbalancer.plugin.LoadBalancerPlugin,neutron.services.metering.metering_plugin.MeteringPlugin,neutron.services.l3_router.l3_router_plugin.L3RouterPlugin
auth_strategy = keystone
allow_overlapping_ips = True
rpc_backend = neutron.openstack.common.rpc.impl_kombu
rabbit_port = 5672
rabbit_hosts = 192.168.100.12,192.168.100.13
rabbit_password = guest
rabbit_userid = guest
notification_driver = neutron.openstack.common.notifier.rpc_notifier
notify_nova_on_port_status_changes = True
notify_nova_on_port_data_changes = True
nova_url = http://192.168.1.150:8774/v2
nova_admin_username = nova
nova_admin_tenant_id = d26f330fee374ab098c46f498383fa0b
nova_admin_password = asdasd
nova_admin_auth_url = http://192.168.1.150:35357/v2.0
[quotas]
[agent]
root_helper = sudo neutron-rootwrap /etc/neutron/rootwrap.conf
[keystone_authtoken]
auth_host = 192.168.1.150
auth_port = 35357
auth_protocol = http
admin_tenant_name = service
admin_user = neutron
admin_password = asdasd
signing_dir = $state_path/keystone-signing
[database]
connection = mysql://neutron:asdasd@192.168.1.150/neutron
[service_providers]
service_provider=LOADBALANCER:Haproxy:neutron.services.loadbalancer.drivers.haproxy.plugin_driver.HaproxyOnHostPluginDriver:default
service_provider=VPN:openswan:neutron.services.vpn.service_drivers.ipsec.IPsecVPNDriver:default
~~~

Ahora editamos el fichero **/etc/neutron/plugins/ml2/ml2_conf.ini**:

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
[ovs]
local_ip = 192.168.100.12
tunnel_type = gre
enable_tunneling = True
[securitygroup]
firewall_driver = neutron.agent.linux.iptables_firewall.OVSHybridIptablesFirewallDriver
enable_security_group = True
~~~

Editamos también el fichero **/etc/neutron/l3_agent.ini**:

~~~
interface_driver = neutron.agent.linux.interface.OVSInterfaceDriver
use_namespaces = True
~~~

Editamos el fichero **/etc/neutron/dhcp_agent.ini**:

~~~
interface_driver = neutron.agent.linux.interface.OVSInterfaceDriver
dhcp_driver = neutron.agent.linux.dhcp.Dnsmasq
use_namespaces = True
~~~~

Y el fichero **/etc/neutron/metadata_agent.ini**:

~~~
auth_url = http://{{ controller_ip }}:5000/v2.0
auth_region = {{ region }}
admin_tenant_name = service
admin_user = neutron
admin_password = {{ neutron_identity_password }}

nova_metadata_ip = {{ controller_ip }}
metadata_proxy_shared_secret = {{ shared_secret }}
~~~

Reiniciamos openvswitch:

~~~
service openvswitch-switch restart
~~~

Y creamos los puentes necesarios:

~~~
ovs-vsctl add-br br-int
ovs-vsctl add-br br-ex
ovs-vsctl add-port br-ex eth1
~~~

Una vez finalizada la configuración en Zeus, debemos editar el fichero **/etc/nova/nova.conf** en los dos nodos controladores Zeus y Hades:

~~~
network_api_class = nova.network.neutronv2.api.API
neutron_url = http://controller:9696
neutron_auth_strategy = keystone
neutron_admin_tenant_name = service
neutron_admin_username = neutron
neutron_admin_password = NEUTRON_PASS
neutron_admin_auth_url = http://controller:35357/v2.0
linuxnet_interface_driver = nova.network.linux_net.LinuxOVSInterfaceDriver
firewall_driver = nova.virt.firewall.NoopFirewallDriver
security_group_api = neutron
service_neutron_metadata_proxy = true
neutron_metadata_proxy_shared_secret = asdasd
~~~

Y reniciamos en los dos nodos los siguientes servicios:

~~~
service nova-api restart
service nova-scheduler restart
service nova-conductor restart
~~~

Y solo en Zeus:

~~~
service neutron-server restart
service neutron-plugin-openvswitch-agent restart
service neutron-l3-agent restart
service neutron-dhcp-agent restart
service neutron-metadata-agent restart
~~~

Creamos algunas redes para comprobar que neutron funciona correctamente:

~~~
neutron net-create ext-net --shared --router:external=True
~~~

~~~
neutron subnet-create ext-net --name ext-subnet \
  --allocation-pool start=192.168.1.200,end=192.168.1.220 \
  --disable-dhcp --gateway 192.168.1.1 192.168.1.0/24
~~~

~~~
# neutron net-list
+--------------------------------------+--------------+-----------------------------------------------------+
| id                                   | name         | subnets                                             |
+--------------------------------------+--------------+-----------------------------------------------------+
| 1a8e71ec-992f-4869-9ed0-9c26642b9d13 | red de admin | 7c6a81a5-88fd-4ff8-876f-57ba38d3569d 10.0.0.0/24    |
| c2de4862-c291-402e-9ec6-cdd4738f1fba | ext-net      | db372eaf-8118-4b62-aa7f-731d30a8fc03 192.168.1.0/24 |
+--------------------------------------+--------------+-----------------------------------------------------+
~~~

~~~
# neutron router-list
+--------------------------------------+-----------------+-----------------------------------------------------------------------------+
| id                                   | name            | external_gateway_info                                                       |
+--------------------------------------+-----------------+-----------------------------------------------------------------------------+
| 91229ea9-c870-4a9f-b126-82134d4f4e81 | router de admin | {"network_id": "c2de4862-c291-402e-9ec6-cdd4738f1fba", "enable_snat": true} |
+--------------------------------------+-----------------+-----------------------------------------------------------------------------+
~~~