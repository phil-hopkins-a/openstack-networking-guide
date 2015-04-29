.. highlight: ini
   :linenothreshold: 5

============================================
Scenario: Basic implementation of HA routers
============================================

This scenario describes a basic implementation of the OpenStack
Networking Highly Available (HA) feature using the ML2
plug-in with Linux bridge. The example configuration creates
one flat external network and VXLAN tenant networks. However, HA
also supports VLAN external networks and GRE tenant networks with
minor modifications to the example configuration.

Requirements
~~~~~~~~~~~~

1. One controller node with one network interface: management.

#. At least two network nodes with three network interfaces: management, instance
   tunnels, and external (typically the Internet). The Linux bridge
   bridge `br-ex` must contain a port on the external interface.

#. At least two compute nodes with two network interfaces: management
   and instance tunnels.

.. figure:: ./figures/scenario-l3ha-hw.png
   :alt: Neutron HA Router Scenario - Network Layout

.. warning:: 
    
    Linux bridge HA routers are currently broken when using
    L2Population. This means that Linux bridge and VXLAN does not work.

    bug - https://bugs.launchpad.net/neutron/+bug/1365476, https://bugs.launchpad.net/neutron/+bug/1411752
    
.. note::
   For best operation, consider using the *master* branch of neutron:
   http://git.openstack.org/cgit/openstack/neutron/

Prerequisites
~~~~~~~~~~~~~

Controller node

  * Operational SQL server with `neutron` database and appropriate configuration in the neutron-server.conf file.

  * Operational message queue service with appropriate configuration in the neutron-server.conf file.

  * Operational OpenStack Identity service with appropriate configuration in the neutron-server.conf file.

  * Operational OpenStack Compute controller/management service with appropriate configuration to use neutron in the nova.conf file.

  * Neutron server service, ML2 plug-in, and any dependencies.

Network node

  * Operational OpenStack Identity service with appropriate configuration in the neutron-server.conf file.

  * ML2 plug-in, Linux bridge agent, L3 agent, DHCP agent, metadata agent, and any dependencies including the `ipset` and `conntrack` utilities.

Compute nodes

  * Operational OpenStack Identity service with appropriate configuration in the neutron-server.conf file.

  * Operational OpenStack Compute controller/management service with appropriate configuration to use neutron in the nova.conf file.

  * ML2 plug-in, Linux bridge agent and any dependencies including the `ipset` and `conntrack` utilities.

.. figure:: ./figures/scenario-l3ha-lb-services.png
   :alt: Neutron HA router Scenario - Service Layout

Architecture
~~~~~~~~~~~~

General
~~~~~~~

The general HA architecture augments the legacy architecture by
creating additional 'qrouter' namespaces for each router created.
An "HA" network is created which connects each qrouter namespace.
A keepalived process is created within each namespace. The keepalived
processes will commumicate with one namespace router selected as the master
and the the gateway IP is used as a VIP in the master namespace.
If a failure is detected a new master will be elected and the VIPs 
are moved into the new master namespace.

.. figure:: ./figures/scenario-l3ha-general.png
   :alt: Neutron HA router Scenario - Architecture Overview

The network node runs the L3 agent, DHCP agent, and metadata agent. HA 
routers can coexist with multiple DHCP agents. DHCP agents can even run
on compute nodes. In addition to `qrouter` namespaces, the L3 agent 
manages SNAT for any instances without a floating IP address as well as
floating IPs within the namespace. The metadata agent handles metadata
operations for instances using tenant networks on HA routers.

.. figure:: ./figures/scenario-l3ha-lb-network1.png
   :alt: Neutron HA router Scenario - Network Node Overview

The compute node runs the L2 Linux bridge agent. Using a separate Linux 
bridge for each network, virtual ethernet pairs connect the VM to the
bridge for a network. A tunnel or VLAN interface is also connected to the
bridge to connect to the data network interface.

.. figure:: ./figures/scenario-l3ha-lb-compute1.png
   :alt: Neutron HA router Scenario - Compute Node Overview

Components
----------

The network node contains the following components:

* Linux bridge agent managing Linux bridges, connectivity among
   them, and interaction with other network components
   such as namespaces  and underlying interfaces.

* DHCP agent managing the `qdhcp` namespaces.

  1. The `qdhcp` namespaces provide DHCP services for instances using 
     tenant networks on HA routers.

* L3 agent managing the `qrouter` namespaces.

  1. For instances using tenant networks using HA routers, the
     `qrouter` namespaces perform SNAT between tenant and external
     networks. They also route metadata traffic between instances using
     tenant networks on HA routers and the metadata agent.


* Metadata agent handling metadata operations.

  1. The metadata agent handles metadata operations for instances
     using tenant networks using HA routers.

.. figure:: ./figures/scenario-l3ha-lb-network2.png
   :alt: Neutron HA router Scenario - Network Node Components

The compute nodes contain the following components:

* Linux bridge agent managing Linux bridges, connectivity among
   them, and interaction via virtual ethernet pairs with other network 
   components such as namespaces, Linux bridges, and underlying interfaces.

* L3 agent managing the `qrouter` namespace.

  1. For instances using tenant networks on HA routers, the
     `qrouter` namespaces route network traffic among tenant
     networks.

  1. For instances using tenant networks on HA routers, the
     qrouter namespaces perform DNAT and SNAT between tenant and external
     networks.

* Metadata agent handling metadata operations.

  1. The metadata agent handles metadata operations for instances
     using tenant networks on distributed routers.

* Linux bridges handling security groups.

  1. The Networking service uses iptables to manage security groups for
     instances.

.. figure:: ./figures/scenario-l3ha-lb-compute2.png
   :alt: Neutron HA router Scenario - Compute Node Components

Packet Flow through Linux bridge HA router environment
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Packet flow through HA routers is identical to the path used in the Linux bridge using a single router. The master HA router will be the same as the single router. See that section for more details.

HA Failover Operatons
~~~~~~~~~~~~~~~~~~~~~

.. figure:: ./figures/scenario-l3ha-lb-flowfailover1.png
   :alt: Neutron HA router Scenario - Failover operations

Configuration
~~~~~~~~~~~~~

The configuration files on each node, controller, network, compute, are similar with only the local_ip set to the interface on the data network for that node. The crucial settings are indicated as follows:

1. Configure the base options Edit the :file:`/etc/neutron/neutron.conf` file:
   ::
     [DEFAULT]
     verbose = True
     core_plugin = ml2
     service_plugins = router
     allow_overlapping_ips = True

     dhcp_agents_per_network = 2
      
     router_distributed = False
     l3_ha = True
     max_l3_agents_per_router = 3
     min_l3_agents_per_router = 2
     l3_ha_net_cidr = 169.254.192.0/18
     notify_nova_on_port_status_changes = True
     notify_nova_on_port_data_changes = True
     nova_url = http://controller:8774/v2
     nova_region_name = regionOne
     nova_admin_username = NOVA_ADMIN_USERNAME
     nova_admin_tenant_id = NOVA_ADMIN_TENANT_ID
     nova_admin_password =  NOVA_ADMIN_PASSWORD
     nova_admin_auth_url = http://controller:35357/v2.0

   .. note::

      Replace NOVA_ADMIN_USERNAME, NOVA_ADMIN_TENANT_ID, and
      NOVA_ADMIN_PASSWORD with suitable values for your environment.
      
#. Edit the :file:`l3_agent.ini` file:
   ::
      agent_mode = legacy
 
#. Configure the ML2 plug-in. Edit the
   :file:`/etc/neutron/plugins/ml2/ml2_conf.ini` file:
   ::
     [ml2]
     type_drivers = flat,vxlan
     tenant_network_types = vxlan
     mechanism_drivers = linuxbridge,l2population

     [ml2_type_vxlan]
     vni_ranges = 1:1000
     vxlan_group = 239.1.1.1

     [securitygroup]
     enable_security_group = True
     enable_ipset = True
     firewall_driver = neutron.agent.linux.iptables_firewall.OVSHybridIptablesFirewallDriver
      
     [ml2_type_vlan]
     network_vlan_ranges = vlan:1:1000

     [ml2_type_gre]
     tunnel_id_ranges = 1:1000
      
     [linuxbridge]

     [l2pop]
     agent_boot_time = 180

     [vxlan]
     enable_vxlan = True
     local_ip = TUNNEL_NETWORK_INTERFACE_IP
     l2_population = True
      
  .. note::
      The first value in the 'tenant_network_types' option becomes the
      default tenant network type when a non-privileged user creates a
      network.

  .. note::
      Adjust the VXLAN tunnel ID range for your environment.

#. Start the following services: 

  1. Controller node:
      * Server
  #. Network node(s):
      * Linux bridge agent
      * L3 agent
      * DHCP agent
      * Metadata agent
  #. Computer node(s):
      * Linux bridge agent


Verify service operation
------------------------

1. Source the administrative tenant credentials.

#. Verify presence and operation of the agents
   ::
     $ neutron agent-list
     +--------------------------------------+--------------------+----------+-------+----------------+---------------------------+
     | id                                   | agent_type         | host     | alive | admin_state_up | binary                    |
     +--------------------------------------+--------------------+----------+-------+----------------+---------------------------+
     | 7856ba29-5447-4392-b2e1-2c236bd5f479 | Metadata agent     | network  | :-)   | True           | neutron-metadata-agent    |
     | 85d5c715-08f6-425d-9efc-73633736bf06 | Linux bridge agent | network2 | :-)   | True           | neutron-linuxbridge-agent |
     | 98d32a4d-1257-4b42-aea4-ad9bd7deea62 | Metadata agent     | network2 | :-)   | True           | neutron-metadata-agent    |
     | b45096a1-7bfa-4816-8b3c-900b752a9c08 | DHCP agent         | network  | :-)   | True           | neutron-dhcp-agent        |
     | d4c45b8e-3b34-4192-80b1-bbdefb110c3f | Linux bridge agent | compute2 | :-)   | True           | neutron-linuxbridge-agent |
     | e5a4e06b-dd9d-4b97-a09a-c8ba07706753 | Linux bridge agent | network  | :-)   | True           | neutron-linuxbridge-agent |
     | e8f8b228-5c3e-4378-b8f5-36b5c41cb3fe | L3 agent           | network2 | :-)   | True           | neutron-l3-agent          |
     | f2d10c26-2136-4e6a-86e5-d22f67ab22d7 | Linux bridge agent | compute  | :-)   | True           | neutron-linuxbridge-agent |
     | f9f94732-08af-4f82-8908-fdcd69ab12e8 | L3 agent           | network  | :-)   | True           | neutron-l3-agent          |
     | fbeebad9-6590-4f78-bb29-7d58ea867878 | DHCP agent         | network2 | :-)   | True           | neutron-dhcp-agent        |
     +--------------------------------------+--------------------+----------+-------+----------------+---------------------------+
  
  
Create initial networks
~~~~~~~~~~~~~~~~~~~~~~~

Use the following example commands as a template to create initial networks
in your environment.

External (flat) network
~~~~~~~~~~~~~~~~~~~~~~~

1. Source the administrative tenant credentials.

#. Create the external network:
   ::
      $ neutron net-create ext-net --router:external True \
        --provider:physical_network external --provider:network_type flat
      Created a new network:
      +---------------------------+--------------------------------------+
      | Field                     | Value                                |
      +---------------------------+--------------------------------------+
      | admin_state_up            | True                                 |
      | id                        | 5266fcbc-d429-4b21-8544-6170d1691826 |
      | name                      | ext-net                              |
      | provider:network_type     | flat                                 |
      | provider:physical_network | external                             |
      | provider:segmentation_id  |                                      |
      | router:external           | True                                 |
      | shared                    | False                                |
      | status                    | ACTIVE                               |
      | subnets                   |                                      |
      | tenant_id                 | 96393622940e47728b6dcdb2ef405f50     |
      +---------------------------+--------------------------------------+

#. Create a subnet on the external network:
   ::
      $ neutron subnet-create ext-net --name ext-subnet \
        --allocation-pool start=203.0.113.101,end=203.0.113.200 \
        --disable-dhcp --gateway 203.0.113.1 203.0.113.0/24
      Created a new subnet:
      +-------------------+----------------------------------------------------+
      | Field             | Value                                              |
      +-------------------+----------------------------------------------------+
      | allocation_pools  | {"start": "203.0.113.101", "end": "203.0.113.200"} |
      | cidr              | 203.0.113.0/24                                     |
      | dns_nameservers   |                                                    |
      | enable_dhcp       | False                                              |
      | gateway_ip        | 203.0.113.1                                        |
      | host_routes       |                                                    |
      | id                | b32e0efc-8cc3-43ff-9899-873b94df0db1               |
      | ip_version        | 4                                                  |
      | ipv6_address_mode |                                                    |
      | ipv6_ra_mode      |                                                    |
      | name              | ext-subnet                                         |
      | network_id        | 5266fcbc-d429-4b21-8544-6170d1691826               |
      | tenant_id         | 96393622940e47728b6dcdb2ef405f50                   |
      +-------------------+----------------------------------------------------+

Tenant (VXLAN) network
----------------------
1. Source the regular tenant credentials.

#. Create a tenant network:
   ::
     $ neutron net-create private
     Created a new network:
     +---------------------------+--------------------------------------+
     | Field                     | Value                                |
     +---------------------------+--------------------------------------+
     | admin_state_up            | True                                 |
     | id                        | d990778b-49ea-4beb-9336-6ea2248edf7d |
     | name                      | private                              |
     | provider:network_type     | vxlan                                |
     | provider:physical_network |                                      |
     | provider:segmentation_id  | 100                                  |
     | router:external           | False                                |
     | shared                    | False                                |
     | status                    | ACTIVE                               |
     | subnets                   |                                      |
     | tenant_id                 | f8207c03fd1e4b4aaf123efea4662819     |
     +---------------------------+--------------------------------------+
   
#. Create a subnet on the tenant network:
   ::
     $ neutron subnet-create --name private-subnet private 10.1.0.0/28
     Created a new subnet:
     +-------------------+-------------------------------------------+
     | Field             | Value                                     |
     +-------------------+-------------------------------------------+
     | allocation_pools  | {"start": "10.1.0.2", "end": "10.1.0.14"} |
     | cidr              | 10.1.0.0/28                               |
     | dns_nameservers   |                                           |
     | enable_dhcp       | True                                      |
     | gateway_ip        | 10.1.0.1                                  |
     | host_routes       |                                           |
     | id                | b7fe4e86-65d5-4e88-8266-88795ae4ac53      |
     | ip_version        | 4                                         |
     | ipv6_address_mode |                                           |
     | ipv6_ra_mode      |                                           |
     | name              | private-subnet                            |
     | network_id        | d990778b-49ea-4beb-9336-6ea2248edf7d      |
     | tenant_id         | f8207c03fd1e4b4aaf123efea4662819          |
     +-------------------+-------------------------------------------+

#. Create a tenant HA router:
   ::
     $ neutron router-create MyRouter --distributed False --ha True
     Created a new router:
     +-----------------------+--------------------------------------+
     | Field                 | Value                                |
     +-----------------------+--------------------------------------+
     | admin_state_up        | True                                 |
     | distributed           | False                                |
     | external_gateway_info |                                      |
     | ha                    | True                                 |
     | id                    | 557bf478-6afe-48af-872f-63513f7e9b92 |
     | name                  | MyRouter                             |
     | routes                |                                      |
     | status                | ACTIVE                               |
     | tenant_id             | f8207c03fd1e4b4aaf123efea4662819     |
     +-----------------------+--------------------------------------+

#. Add the tenant subnet interface to the router:
   ::
     neutron router-interface-add MyRouter private-subnet
     Added interface 4cb8f7ea-28f2-4fe1-91f7-1c2823994fc4 to router MyRouter.

#. Set the router gateway to the external network:
   ::
     $ neutron router-gateway-set MyRouter public
     Set gateway for router MyRouter

#. Namespaces created on the network nodes:
   ::
     $ ip netns
     qrouter-744e386d-03de-4993-8ab2-3b55b78a22e2
     qdhcp-4bc242e0-97c4-4791-908d-7c471fc10ad1
     qdhcp-d990778b-49ea-4beb-9336-6ea2248edf7d


HA router functional description
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

The network implementation in this example uses Linux Bridge as the ML2 agent and VXLAN as the network segmentation technology. Refer to the network node illustration for for help in understanding the following discussion.

Upon creation of a network, router namespaces are built, with the number of routers namespaces built per network determined by the settings for max_l3_agents_per_router and min_l3_agents_per_router. Each tenant is limited to a total of 255 HA routers so the max L3 routers variable should not be a large number. These namespaces are created on different network nodes running an L3 agent with a L3 router within each namespace. The neutron scheduler, running on the controller node, will determine which network nodes will be selected to receive the router namespaces. As shown in the illustration, a keepalived and a conntrackd process will be created to control which router namespace has the router IPs, as these can exist on only one of the routers.

Verify Operation
~~~~~~~~~~~~~~~~

Show networks and verify the creation of the HA network:
   ::
   
     $ neutron net-list
     +--------------------------------------+----------------------------------------------------+-------------------------------------------------------+
     | id                                   | name                                               | subnets                                               |
     +--------------------------------------+----------------------------------------------------+-------------------------------------------------------+
     | 4bc242e0-97c4-4791-908d-7c471fc10ad1 | private1                                           | cc605c67-3e0b-4127-9eb2-4e4d0e5e589d 10.2.0.0/28      |
     | b304e495-b80d-4dd7-9345-5455302397a7 | HA network tenant f8207c03fd1e4b4aaf123efea4662819 | bbb53715-f4e9-4ce3-bf2b-44b2aed2f4ef 169.254.192.0/18 |
     | d990778b-49ea-4beb-9336-6ea2248edf7d | private                                            | b7fe4e86-65d5-4e88-8266-88795ae4ac53 10.1.0.0/28      |
     | fde31a29-3e23-470d-bc9d-6218375dca4f | public                                             | 2e1d865a-ef56-41e9-aa31-63fb8a591003 172.16.0.0/24    |
     +--------------------------------------+----------------------------------------------------+-------------------------------------------------------+
1. On the network nodes, verify creation of the ``qrouter`` and ``qdhcp``
   namespaces:

Network node 1:
   ::

     $ ip netns
     qrouter-744e386d-03de-4993-8ab2-3b55b78a22e2
     qdhcp-4bc242e0-97c4-4791-908d-7c471fc10ad1
     qdhcp-d990778b-49ea-4beb-9336-6ea2248edf7d

Network node 2:
   ::

     $ ip netns
     qrouter-744e386d-03de-4993-8ab2-3b55b78a22e2
     qdhcp-4bc242e0-97c4-4791-908d-7c471fc10ad1
     qdhcp-d990778b-49ea-4beb-9336-6ea2248edf7d


   .. note::
      Both ``qrouter`` namespaces should use the same UUID.

   .. note::
      The ``qdhcp`` namespaces might not appear until launching an instance.


The keepalived processes for each router communicate with each other through an HA network which is also created at this time. The HA network name will use take the form ha-<tennant UUID> and can be seen by running neutron net-list. An HA port is generated for each router namespace along with a veth pair on the network nodes hosting the router namespace, where one veth member, with the name ha-<left most 11 characters of the port UUID>, placed into the router namespace and the other veth pair member, with the name tap<left most 11 characters of the port UUID>, placed into a Linux bridge, named brg<Left most 11 chars of the HA network UUID>. A VXLAN interface using the HA network segmentation ID is added to the Linux bridge to complete the communication path. The interface within the router namespace is assigned the IP range of 169.254.???.???/24, where the third octet of the IP is unique to each tenant and the forth octet unique to each ha interface.  The keepalived processes within each router namespace will communicate with each other using vrrp and elect a master router. The master router then adds all of the router VIPs (gateway IPs and external IP) to its interfaces and all other routers are placed into backup mode.

#. Show network node qrouter namespace on the master node:
   ::
     $ ip netns exec qrouter-744e386d-03de-4993-8ab2-3b55b78a22e2 ip a
     1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default 
         link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
         inet 127.0.0.1/8 scope host lo
            valid_lft forever preferred_lft forever
         inet6 ::1/128 scope host 
            valid_lft forever preferred_lft forever
     2: ha-0d039391-92: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP group default qlen 1000
         link/ether fa:16:3e:d9:c0:7c brd ff:ff:ff:ff:ff:ff
         inet 169.254.192.6/18 brd 169.254.255.255 scope global ha-0d039391-92
            valid_lft forever preferred_lft forever
         inet6 fe80::f816:3eff:fed9:c07c/64 scope link 
            valid_lft forever preferred_lft forever
     3: qr-670e2e87-5f: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP group default qlen 1000
         link/ether fa:16:3e:70:69:40 brd ff:ff:ff:ff:ff:ff
         inet6 fe80::f816:3eff:fe70:6940/64 scope link 
            valid_lft forever preferred_lft forever
     4: qr-158c1d10-c5: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP group default qlen 1000
         link/ether fa:16:3e:c4:7a:4b brd ff:ff:ff:ff:ff:ff
         inet6 fe80::f816:3eff:fec4:7a4b/64 scope link 
            valid_lft forever preferred_lft forever
     5: qg-a41a7d54-94: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP group default qlen 1000
         link/ether fa:16:3e:c9:fc:13 brd ff:ff:ff:ff:ff:ff
         inet6 fe80::f816:3eff:fec9:fc13/64 scope link 
            valid_lft forever preferred_lft forever

#. Show network node qrouter namespace on the backup node:
   ::
     $ ip netns exec qrouter-557bf478-6afe-48af-872f-63513f7e9b92 ip a
     1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default 
         link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
         inet 127.0.0.1/8 scope host lo
            valid_lft forever preferred_lft forever
         inet6 ::1/128 scope host 
            valid_lft forever preferred_lft forever
     2: ha-602e7d30-71: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP group default qlen 1000
         link/ether fa:16:3e:dd:8b:30 brd ff:ff:ff:ff:ff:ff
         inet 169.254.192.3/18 brd 169.254.255.255 scope global ha-602e7d30-71
            valid_lft forever preferred_lft forever
         inet6 fe80::f816:3eff:fedd:8b30/64 scope link 
            valid_lft forever preferred_lft forever
     3: qr-4cb8f7ea-28: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP group default qlen 1000
         link/ether fa:16:3e:c9:74:0c brd ff:ff:ff:ff:ff:ff
         inet6 fe80::f816:3eff:fec9:740c/64 scope link 
            valid_lft forever preferred_lft forever
     4: qr-df9c2f7b-37: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP group default qlen 1000
         link/ether fa:16:3e:87:60:5e brd ff:ff:ff:ff:ff:ff
         inet6 fe80::f816:3eff:fe87:605e/64 scope link 
            valid_lft forever preferred_lft forever
     5: qg-ad2929f6-dd: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP group default qlen 1000
         link/ether fa:16:3e:58:2e:10 brd ff:ff:ff:ff:ff:ff
         inet6 fe80::f816:3eff:fe58:2e10/64 scope link 
            valid_lft forever preferred_lft forever

#. Network node Linux bridges:
   ::
     $ brctl show
     bridge name     bridge id               STP enabled     interfaces
     brqb304e495-b8          8000.921bf69da9dd       no              tap0d039391-92
                                                          vxlan-102
     brqd990778b-49          8000.1a5ce98d92e2       no              tap670e2e87-5f
                                                          vxlan-100
     brqfde31a29-3e          8000.eebd5cd87645       no               eth2
                                                          tapa41a7d54-94

#. VRRP communication from one network node to the other:
   ::
      $ ip netns exec qrouter-744e386d-03de-4993-8ab2-3b55b78a22e2 tcpdump -e -n -vvv -l -i ha-0d039391-92
      tcpdump: listening on ha-0d039391-92, link-type EN10MB (Ethernet), capture size 65535 bytes
      16:00:39.994393 fa:16:3e:d9:c0:7c > 01:00:5e:00:00:12, ethertype IPv4 (0x0800), length 54: (tos 0xc0, ttl 255, id 36898, offset 0, flags [none], proto VRRP (112), length 40)
          169.254.192.6 > 224.0.0.18: vrrp 169.254.192.6 > 224.0.0.18: VRRPv2, Advertisement, vrid 1, prio 50, authtype none, intvl 2s, length 20, addrs: 10.1.0.1
      16:00:41.995826 fa:16:3e:d9:c0:7c > 01:00:5e:00:00:12, ethertype IPv4 (0x0800), length 54: (tos 0xc0, ttl 255, id 36899, offset 0, flags [none], proto VRRP (112), length 40)
          169.254.192.6 > 224.0.0.18: vrrp 169.254.192.6 > 224.0.0.18: VRRPv2, Advertisement, vrid 1, prio 50, authtype none, intvl 2s, length 20, addrs: 10.1.0.1
      16:00:43.997403 fa:16:3e:d9:c0:7c > 01:00:5e:00:00:12, ethertype IPv4 (0x0800), length 54: (tos 0xc0, ttl 255, id 36900, offset 0, flags [none], proto VRRP (112), length 40)
          169.254.192.6 > 224.0.0.18: vrrp 169.254.192.6 > 224.0.0.18: VRRPv2, Advertisement, vrid 1, prio 50, authtype none, intvl 2s, length 20, addrs: 10.1.0.1
      16:00:45.998820 fa:16:3e:d9:c0:7c > 01:00:5e:00:00:12, ethertype IPv4 (0x0800), length 54: (tos 0xc0, ttl 255, id 36901, offset 0, flags [none], proto VRRP (112), length 40)
          169.254.192.6 > 224.0.0.18: vrrp 169.254.192.6 > 224.0.0.18: VRRPv2, Advertisement, vrid 1, prio 50, authtype none, intvl 2s, length 20, addrs: 10.1.0.1

The keepalived processes for a set of HA routers then monitor each other using VRRP multicasts. If the master router fails, it is detected due to a loss of its VRRP multicasts, a new master router will be elected and the VIPs are moved onto the new master router. When a failure occurs the conntrackd processes ensure that any existing TCP connection states exist on all of the backup routers so that the connections migrate smoothly over to the new master router preventing connection loss.

#. On the controller node, ping the tenant router external network interface
   IP address, typically the lowest IP address in the external network
   subnet allocation range:
   ::

     $ ping -c 4 203.0.113.101
     PING 203.0.113.101 (203.0.113.101) 56(84) bytes of data.
     64 bytes from 203.0.113.101: icmp_req=1 ttl=64 time=0.619 ms
     64 bytes from 203.0.113.101: icmp_req=2 ttl=64 time=0.189 ms
     64 bytes from 203.0.113.101: icmp_req=3 ttl=64 time=0.165 ms
     64 bytes from 203.0.113.101: icmp_req=4 ttl=64 time=0.216 ms

     --- 203.0.113.101 ping statistics ---
     4 packets transmitted, 4 received, 0% packet loss, time 2999ms
     rtt min/avg/max/mdev = 0.165/0.297/0.619/0.187 ms

#. Source the regular tenant credentials.

#. Launch an instance with an interface on the tenant network.

#. Obtain console access to the instance.

   a. Test connectivity to the tenant network router:
      ::

        $ ping -c 4 192.168.1.1
        PING 192.168.1.1 (192.168.1.1) 56(84) bytes of data.
        64 bytes from 192.168.1.1: icmp_req=1 ttl=64 time=0.357 ms
        64 bytes from 192.168.1.1: icmp_req=2 ttl=64 time=0.473 ms
        64 bytes from 192.168.1.1: icmp_req=3 ttl=64 time=0.504 ms
        64 bytes from 192.168.1.1: icmp_req=4 ttl=64 time=0.470 ms

        --- 192.168.1.1 ping statistics ---
        4 packets transmitted, 4 received, 0% packet loss, time 2998ms
        rtt min/avg/max/mdev = 0.357/0.451/0.504/0.055 ms

   #. Test connectivity to the Internet:
      ::

        $ ping -c 4 openstack.org
        PING openstack.org (174.143.194.225) 56(84) bytes of data.
        64 bytes from 174.143.194.225: icmp_req=1 ttl=53 time=17.4 ms
        64 bytes from 174.143.194.225: icmp_req=2 ttl=53 time=17.5 ms
        64 bytes from 174.143.194.225: icmp_req=3 ttl=53 time=17.7 ms
        64 bytes from 174.143.194.225: icmp_req=4 ttl=53 time=17.5 ms

        --- openstack.org ping statistics ---
        4 packets transmitted, 4 received, 0% packet loss, time 3003ms
        rtt min/avg/max/mdev = 17.431/17.575/17.734/0.143 ms

#. Create the appropriate security group rules to allow ping and SSH access
   to the instance.

#. Create a floating IP address:
   ::

     $ neutron floatingip-create ext-net
     Created a new floatingip:
     +---------------------+--------------------------------------+
     | Field               | Value                                |
     +---------------------+--------------------------------------+
     | fixed_ip_address    |                                      |
     | floating_ip_address | 203.0.113.102                        |
     | floating_network_id | 5266fcbc-d429-4b21-8544-6170d1691826 |
     | id                  | 20a6b5dd-1c5c-460e-8a81-8b5cf1739307 |
     | port_id             |                                      |
     | router_id           |                                      |
     | status              | DOWN                                 |
     | tenant_id           | f8207c03fd1e4b4aaf123efea4662819     |
     +---------------------+--------------------------------------+

#. Associate the floating IP address with the instance:
   ::

     $ nova floating-ip-associate demo-instance1 203.0.113.102

#. On the controller node, ping the floating IP address associated with
   the instance:
   ::

     $ ping -c 4 203.0.113.102
     PING 203.0.113.102 (203.0.113.112) 56(84) bytes of data.
     64 bytes from 203.0.113.102: icmp_req=1 ttl=63 time=3.18 ms
     64 bytes from 203.0.113.102: icmp_req=2 ttl=63 time=0.981 ms
     64 bytes from 203.0.113.102: icmp_req=3 ttl=63 time=1.06 ms
     64 bytes from 203.0.113.102: icmp_req=4 ttl=63 time=0.929 ms

     --- 203.0.113.102 ping statistics ---
     4 packets transmitted, 4 received, 0% packet loss, time 3002ms
     rtt min/avg/max/mdev = 0.929/1.539/3.183/0.951 ms













