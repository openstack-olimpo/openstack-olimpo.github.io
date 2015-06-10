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


create database neutron character set utf8 collate utf8_general_ci;
grant all on neutron.* to neutron@'%' identified by 'asdasd';
flush privileges;

keystone user-create --name neutron --pass asdasd --email neutron@olimpo.com

keystone user-role-add --user neutron --tenant service --role admin

keystone service-create --name neutron --type network --description "OpenStack Networking"

keystone endpoint-create \
  --service-id $(keystone service-list | awk '/ network / {print $2}') \
  --publicurl http://192.168.100.12:9696 \
  --adminurl http://192.168.100.12:9696 \
  --internalurl http://192.168.100.12:9696

apt-get install neutron-server neutron-plugin-ml2

fichero
---
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
---

/etc/neutron/plugins/ml2/ml2_conf.ini

---
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
---


/etc/nova/nova.conf (2 nodos)

---
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
---

(2 nodos)
service nova-api restart
service nova-scheduler restart
service nova-conductor restart

(Zeus)
service neutron-server restart



Añadir /etc/sysctl.conf (los 3 primeros están comentado)s

net.ipv4.ip_forward=1
net.ipv4.conf.all.rp_filter=0
net.ipv4.conf.default.rp_filter=0
net.bridge.bridge-nf-call-arptables=1
net.bridge.bridge-nf-call-iptables=1
net.bridge.bridge-nf-call-ip6tables=1

sysctl -p

apt-get install neutron-plugin-ml2 neutron-plugin-openvswitch-agent neutron-l3-agent neutron-dhcp-agent


/etc/neutron/l3_agent.ini

interface_driver = neutron.agent.linux.interface.OVSInterfaceDriver
use_namespaces = True

/etc/neutron/metadata_agent.ini

auth_url = http://{{ controller_ip }}:5000/v2.0
auth_region = {{ region }}
admin_tenant_name = service
admin_user = neutron
admin_password = {{ neutron_identity_password }}

nova_metadata_ip = {{ controller_ip }}
metadata_proxy_shared_secret = {{ shared_secret }}



/etc/nova/nova.conf (2 nodos)

service_neutron_metadata_proxy = true
neutron_metadata_proxy_shared_secret = asdasd

service nova-api restart

/etc/neutron/plugins/ml2/ml2_conf.ini

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



service openvswitch-switch restart

ovs-vsctl add-br br-int
ovs-vsctl add-br br-ex
ovs-vsctl add-port br-ex eth1

service neutron-plugin-openvswitch-agent restart
service neutron-l3-agent restart
service neutron-dhcp-agent restart
service neutron-metadata-agent restart

-----------
Ejemplo con palabras en **Negrita**.

>Ejemplo de texto en letra cursiva

Ejemmplo de insercion imagen:
![nombre](ubicacion de la imagen)

Ejemplo de enlace:
[texto](url)

Ejemplo de tabla:
|NOMBRE MV|FUNCIÓN|MEMORIA RAM|IP|
|:---:|------|------|------|
|**HERA**|Proxy 1|256 MB|192.168.100.10|
|**AFRODITA**|Proxy 2|256 MB|192.168.100.11|
|**ZEUS**|Controlador 1|1 GB|192.168.100.12|
|**HADES**|Controlador 2|1 GB|192.168.100.13|
|**POSEIDON**|Ceph 1|1 GB|192.168.100.14|
|**APOLO**|Ceph 2|1 GB|192.168.100.15|
|**ARTEMISA**|Ceph 3|1 GB|192.168.100.16|
|**ARES**|Computación 1|2 GB|192.168.100.17|
|**ATENEA**|Computación 1|2 GB|192.168.100.18|

## TITULO MAYOR - SECCION

### SUBSECCION

Ejemplo de lista:

+ item 1
+ item 2
+ item 3

Ejemplo para poner codigo:
~~~
aptitude update
aptitude upgrade
aptitude install qemu-kvm libvirt-bin bridge-utils virt-manager
~~~