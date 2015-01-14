
# Scenario: Basic implementation of HA routers

This scenario describes a basic implementation of the OpenStack
Networking Highly Available (HA) feature using the ML2
plug-in with Linux bridge. The example configuration creates
one flat external network and VXLAN tenant networks. However, HA
also supports VLAN external networks and GRE tenant networks with
minor modifications to the example configuration.

## Requirements

1. One controller node with one network interface: management.

1. At least two network nodes with three network interfaces: management, instance
   tunnels, and external (typically the Internet). The Linux bridge
   bridge `br-ex` must contain a port on the external interface.

1. At least two compute nodes with two network interfaces: management
   and instance tunnels.

![Neutron HA Router  Scenario - Hardware Requirements](images/HALBopenstack.png "Neutron HA Router Scenario - Hardware Requirements")

![Neutron DVR Scenario - Network Layout](../common/images/networkguide-neutron-dvr-networks.png "Neutron DVR Scenario - Network Layout")

**Warning: Linux bridge HA routers are currently broken when using
L2Population. This means that Linux bridge and VXLAN does not work.

bug - ????

**For best operation, consider using the *master* branch of neutron:**

http://git.openstack.org/cgit/openstack/neutron/

## Prerequisites

1. Controller node

  1. Operational SQL server with `neutron` database and appropriate
     configuration in the neutron-server.conf file.

  1. Operational message queue service with appropriate configuration
     in the neutron-server.conf file.

  1. Operational OpenStack Identity service with appropriate configuration
     in the neutron-server.conf file.

  1. Operational OpenStack Compute controller/management service with
     appropriate configuration to use neutron in the nova.conf file.

  1. Neutron server service, ML2 plug-in, and any dependencies.

1. Network node

  1. Operational OpenStack Identity service with appropriate configuration
     in the neutron-server.conf file.

  1. ML2 plug-in, Linux bridge agent, L3 agent,
     DHCP agent, metadata agent, and any dependencies including the
     `ipset` and `conntrack` utilities.

1. Compute nodes

  1. Operational OpenStack Identity service with appropriate configuration
     in the neutron-server.conf file.

  1. Operational OpenStack Compute controller/management service with
     appropriate configuration to use neutron in the nova.conf file.

  1. ML2 plug-in, Linux bridgeh agent and any dependencies including the `ipset` and
     `conntrack` utilities.

![Neutron DVR Scenario - Service Layout](../common/images/networkguide-neutron-dvr-services.png "Neutron DVR Scenario - Service Layout")

## Architecture

### General

The general HA architecture augments the legacy architecture by
creating additional 'qrouter' namespaces for each router created.
An "HA" network is created and connecting between each qrouter namespace.
A keepalived process is created within each namepspace to manage
which namespace router will be the master and applies the gateway IP
as a VIP into the master namespace. If a failure is detected a new
master is elected and the VIPs are moved into the new master namespace.

![Neutron DVR Scenario - Architecture Overview](../common/images/networkguide-neutron-dvr-general.png "Neutron DVR Scenario - Architecture Overview")

The network node runs the L3 agent, DHCP agent, and metadata agent. HA 
routers can coexist with multiple DHCP agents. DHCP agents can even run
on compute nodes. In addition to `qrouter` namespaces, the L3 agent 
manages SNAT for any instances without a floating IP address and well as
floating IPs within the namespace. The metadata agent handles metadata
operations for instances using tenant networks on HA routers.

![Neutron DVR Scenario - Network Node Overview](../common/images/networkguide-neutron-dvr-network1.png "Neutron DVR Scenario - Network Node Overview")

The compute node runs the L2 Linux bridge agent. Using a separate Linux 
bridge for each network, virtual ethernet pairs connect the VM to the
bridge for a network. A tunnel or VLAN interface is also connected to the
bridge to connect to the data network interface.

![Neutron DVR Scenario - Compute Node Overview](../common/images/networkguide-neutron-dvr-compute1.png "Neutron DVR Scenario - Compute Node Overview")

### Components

The network node contains the following components:

1. Linux bridge agent managing Linux bridges, connectivity among
   them, and interaction with other network components
   such as namespaces  and underlying interfaces.

1. DHCP agent managing the `qdhcp` namespaces.

  1. The `dhcp` namespaces provide DHCP services for instances using 
     tenant networks on HA routers.

1. L3 agent managing the `qrouter` namespaces.

  1. For instances using tenant networks using HA routers, the
     `qrouter` namespaces perform SNAT between tenant and external
     networks. They also route metadata traffic between instances using
     tenant networks on HA routers and the metadata agent.


1. Metadata agent handling metadata operations.

  1. The metadata agent handles metadata operations for instances
     using tenant networks using HA routers.

![Neutron DVR Scenario - Network Node Components](../common/images/networkguide-neutron-dvr-network2.png "Neutron DVR Scenario - Network Node Components")

The compute nodes contain the following components:

1. Linux bridge agent managing Linux bridges, connectivity among
   them, and interaction via virtual ethernet pairs with other network 
   components such as namespaces, Linux bridges, and underlying interfaces.

1. L3 agent managing the `qrouter` namespace.

  1. For instances using tenant networks on HA routers, the
     `qrouter` namespaces route network traffic among tenant
     networks.

  1. For instances using tenant networks on HA routers, the
     qrouter namespaces perform DNAT and SNAT between tenant and external
     networks.

1. Metadata agent handling metadata operations.

  1. The metadata agent handles metadata operations for instances
     using tenant networks on distributed routers.

1. Linux bridges handling security groups.

  1. The Networking service uses iptables to manage security groups for
     instances.


System environment:

![Neutron HA Router on Linux bridge Environment](images/netlha.png "Neutron HA Router on Linux bridge Environment")

## Configuration:

The configuration files on each node are similar with only the local_ip set to the interface on the data network. The crutial settings are indicated as follows:

1. Configure the base options

   1. Edit the /etc/neutron/neutron.conf file:

    ```
    router_distributed = False
    l3_ha = True
    max_l3_agents_per_router = 3
    min_l3_agents_per_router = 2
    l3_ha_net_cidr = 169.254.192.0/18
    ```

   2. Edit the l3_agent.ini file:
 
    ```
    agent_mode = legacy
    ```

 
1. Configure the ML2 plug-in.

   1. Edit the /etc/neutron/plugins/ml2/ml2_conf.ini file:
 
    ```
    [linuxbridge]

    [l2pop]
    agent_boot_time = 180

    [vxlan]
    enable_vxlan = True
    local_ip = TUNNEL_NETWORK_INTERFACE_IP
    l2_population = True
    ```
 
 
  Note: Replace NOVA_ADMIN_USERNAME, NOVA_ADMIN_TENANT_ID, and
  NOVA_ADMIN_PASSWORD with suitable values for your environment.


1. Start the following services
   1. Controller node:
      * Server
   2. Network node(s):
      * Open vSwitch
      * Open vSwitch agent
      * L3 agent
      * DHCP agent
      * Metadata agent
   3. Computer node(s):
      * Open vSwitch
      * Open vSwitch agent

##Create the routers and gateway:

    ```
    neutron net-create private
    neutron subnet-create --name private-subnet private 10.1.0.0/28
    neutron net-create private1
    neutron subnet-create --name private1-subnet private1 10.2.0.0/28
    neutron net-create --shared public --router:external=True
    neutron net-create --shared public --router:external=True --provider:network_type flat --provider:physical_network phy-br-ex
    neutron subnet-create --name public-subnet public  --allocation-pool start=172.16.0.32,end=172.16.0.64 --gateway=172.16.0.5 --enable_dhcp=False 172.16.0.0/24
    neutron router-create MyRouter --distributed False --ha True
    neutron router-interface-add MyRouter private-subnet
    neutron router-interface-add MyRouter private1-subnet
    neutron router-gateway-set MyRouter public
    ```

##Boot two VMs, one on each network:

    ```
    nova boot --image cirros-qcow --flavor 1 --nic net-id=<UUID of private network> One
    nova boot --image cirros-qcow --flavor 1 --nic net-id=<UUID of private1 network> Two
    ```
    








