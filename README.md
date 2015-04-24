
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

![Neutron HA Router  Scenario - Network Layout](images/scenario-l3ha-hw.png "Neutron HA Router Scenario - Network Layout")


**Warning: Linux bridge HA routers are currently broken when using
L2Population. This means that Linux bridge and VXLAN does not work.

bug - https://bugs.launchpad.net/neutron/+bug/1365476, https://bugs.launchpad.net/neutron/+bug/1411752

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

  1. ML2 plug-in, Linux bridge agent and any dependencies including the `ipset` and
     `conntrack` utilities.

![Neutron HA router Scenario - Service Layout](images/scenario-l3ha-lb-services.png "Neutron HA router Scenario - Service Layout")

## Architecture

### General

The general HA architecture augments the legacy architecture by
creating additional 'qrouter' namespaces for each router created.
An "HA" network is created which connects each qrouter namespace.
A keepalived process is created within each namespace. The keepalived
processes will commumicate with one namespace router selected as the master
and the the gateway IP is used as a VIP in the master namespace.
If a failure is detected a new master will be elected and the VIPs 
are moved into the new master namespace.

![Neutron HA router Scenario - Architecture Overview](./images/scenario-l3ha-general.png "Neutron HA router Scenario - Architecture Overview")

The network node runs the L3 agent, DHCP agent, and metadata agent. HA 
routers can coexist with multiple DHCP agents. DHCP agents can even run
on compute nodes. In addition to `qrouter` namespaces, the L3 agent 
manages SNAT for any instances without a floating IP address as well as
floating IPs within the namespace. The metadata agent handles metadata
operations for instances using tenant networks on HA routers.

![Neutron HA router Scenario - Network Node Overview](./images/scenario-l3ha-lb-network2.png "Neutron HA router Scenario - Network Node Overview")

The compute node runs the L2 Linux bridge agent. Using a separate Linux 
bridge for each network, virtual ethernet pairs connect the VM to the
bridge for a network. A tunnel or VLAN interface is also connected to the
bridge to connect to the data network interface.

![Neutron HA router Scenario - Compute Node Overview](./images/scenario-l3ha-lb-compute2.png "Neutron HA router Scenario - Compute Node Overview")

### Components

The network node contains the following components:

1. Linux bridge agent managing Linux bridges, connectivity among
   them, and interaction with other network components
   such as namespaces  and underlying interfaces.

1. DHCP agent managing the `qdhcp` namespaces.

  1. The `qdhcp` namespaces provide DHCP services for instances using 
     tenant networks on HA routers.

1. L3 agent managing the `qrouter` namespaces.

  1. For instances using tenant networks using HA routers, the
     `qrouter` namespaces perform SNAT between tenant and external
     networks. They also route metadata traffic between instances using
     tenant networks on HA routers and the metadata agent.


1. Metadata agent handling metadata operations.

  1. The metadata agent handles metadata operations for instances
     using tenant networks using HA routers.

![Neutron HA router Scenario - Network Node Components](../common/images/networkguide-neutron-dvr-network2.png "Neutron HA router Scenario - Network Node Components")

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

The configuration files on each node, controller, network, compute, are similar with only the local_ip set to the interface on the data network for that node. The crucial settings are indicated as follows:

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
      * Linux bridge agent
      * L3 agent
      * DHCP agent
      * Metadata agent
   3. Computer node(s):
      * Linux bridge agent
      

##Create networks and VMs:

1. check the agents:
```
neutron agent-list
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
```

###Create the networks and router:

   1. Tenant VXLAN network
      1. Source regular tenant credentials
      1. Create the tenant network

    ```
   neutron net-create private
   Created a new network:
neutron net-create private
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
   ```
   1. Create a subnet on the tenant network:
   ```
   neutron subnet-create --name private-subnet private 10.1.0.0/28
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
   ```

###External (flat) network
   1. Source the adminstrative tenant credentials.
   1. Create the external network:
   
   ```
   neutron net-create --shared public --router:external=True --provider:network_type flat --provider:physical_network physnet1
   Created a new network:
   +---------------------------+--------------------------------------+
   | Field                     | Value                                |
   +---------------------------+--------------------------------------+
   | admin_state_up            | True                                 |
   | id                        | 230a5d21-9ccc-447e-9d7e-3a86d4e2294d |
   | name                      | public                               |
   | provider:network_type     | flat                                 |
   | provider:physical_network | physnet1                             |
   | provider:segmentation_id  |                                      |
   | router:external           | True                                 |
   | shared                    | True                                 |
   | status                    | ACTIVE                               |
   | subnets                   |                                      |
   | tenant_id                 | f8207c03fd1e4b4aaf123efea4662819     |
   +---------------------------+--------------------------------------+
   ```
   1. Create a subnet on the external network:
   
   ```
   neutron subnet-create --name public-subnet public  --allocation-pool start=172.16.0.32,end=172.16.0.64 --gateway=172.16.0.5 --enable_dhcp=False 172.16.0.0/24
   Created a new subnet:
   +-------------------+------------------------------------------------+
   | Field             | Value                                          |
   +-------------------+------------------------------------------------+
   | allocation_pools  | {"start": "172.16.0.32", "end": "172.16.0.64"} |
   | cidr              | 172.16.0.0/24                                  |
   | dns_nameservers   |                                                |
   | enable_dhcp       | False                                          |
   | gateway_ip        | 172.16.0.5                                     |
   | host_routes       |                                                |
   | id                | 3bf2ed2a-1cab-4fbe-b4a5-636c687e2fc3           |
   | ip_version        | 4                                              |
   | ipv6_address_mode |                                                |
   | ipv6_ra_mode      |                                                |
   | name              | public-subnet                                  |
   | network_id        | 230a5d21-9ccc-447e-9d7e-3a86d4e2294d           |
   | tenant_id         | f8207c03fd1e4b4aaf123efea4662819               |
   +-------------------+------------------------------------------------+
   ```
   1. Create an HA router:
   
   ```
   neutron router-create MyRouter --distributed False --ha True
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
   ```
   1. Add the private-subnet interface to the router:

   ```
   neutron router-interface-add MyRouter private-subnet
   Added interface 4cb8f7ea-28f2-4fe1-91f7-1c2823994fc4 to router MyRouter.
   ```
   1. Set the router gateway to the external network:

   ```
   neutron router-gateway-set MyRouter public
   Set gateway for router MyRouter
   ```

   1. Boot a VMsk:

   ```
   nova boot --image cirros-qcow2 --flavor 1 --nic net-id=d990778b-49ea-4beb-9336-6ea2248edf7d One
   +--------------------------------------+-----------------------------------------------------+
   | Property                             | Value                                               |
   +--------------------------------------+-----------------------------------------------------+
   | OS-DCF:diskConfig                    | MANUAL                                              |
   | OS-EXT-AZ:availability_zone          | nova                                                |
   | OS-EXT-SRV-ATTR:host                 | -                                                   |
   | OS-EXT-SRV-ATTR:hypervisor_hostname  | -                                                   |
   | OS-EXT-SRV-ATTR:instance_name        | instance-00000025                                   |
   | OS-EXT-STS:power_state               | 0                                                   |
   | OS-EXT-STS:task_state                | scheduling                                          |
   | OS-EXT-STS:vm_state                  | building                                            |
   | OS-SRV-USG:launched_at               | -                                                   |
   | OS-SRV-USG:terminated_at             | -                                                   |
   | accessIPv4                           |                                                     |
   | accessIPv6                           |                                                     |
   | adminPass                            | VnwmSkCfrM4A                                        |
   | config_drive                         |                                                     |
   | created                              | 2015-02-03T18:07:45Z                                |
   | flavor                               | m1.tiny (1)                                         |
   | hostId                               |                                                     |
   | id                                   | b1500828-32c6-4850-a904-c0a0f75288f2                |
   | image                                | cirros-qcow2 (394495f2-45e1-4dbd-8666-6e1f49c259d2) |
   | key_name                             | -                                                   |
   | metadata                             | {}                                                  |
   | name                                 | One                                                 |
   | os-extended-volumes:volumes_attached | []                                                  |
   | progress                             | 0                                                   |
   | security_groups                      | default                                             |
   | status                               | BUILD                                               |
   | tenant_id                            | f8207c03fd1e4b4aaf123efea4662819                    |
   | updated                              | 2015-02-03T18:07:46Z                                |
   | user_id                              | a5bad7856f3e47908b17ab64004c6def                    |
   +--------------------------------------+-----------------------------------------------------+

   ```
   1. Namespaces created on the network nodes:
   ```
   ip netns
   qrouter-744e386d-03de-4993-8ab2-3b55b78a22e2
   qdhcp-4bc242e0-97c4-4791-908d-7c471fc10ad1
   qdhcp-d990778b-49ea-4beb-9336-6ea2248edf7d
   ```

#HA router functional description 

The network implementation in this example uses Linux Bridge as the ML2 agent and VXLAN as the network segmentation technology. Refer to the network node illustration for for help in understanding the following discussion.

Upon creation of a network, router namespaces are built, with the number of routers namespaces built per network determined by the settings for max_l3_agents_per_router and min_l3_agents_per_router. Each tenant is limited to a total of 255 HA routers so the max L3 routers variable should not be a large number. These namespaces are created on different network nodes running an L3 agent with a L3 router within each namespace. The neutron scheduler, running on the controller node, will determine which network nodes will be selected to receive the router namespaces. As shown in the illustration, a keepalived and a conntrackd process will be created to control which router namespace has the router IPs, as these can exist on only one of the routers.

Show networks:
```
neutron net-list
+--------------------------------------+----------------------------------------------------+-------------------------------------------------------+
| id                                   | name                                               | subnets                                               |
+--------------------------------------+----------------------------------------------------+-------------------------------------------------------+
| 4bc242e0-97c4-4791-908d-7c471fc10ad1 | private1                                           | cc605c67-3e0b-4127-9eb2-4e4d0e5e589d 10.2.0.0/28      |
| b304e495-b80d-4dd7-9345-5455302397a7 | HA network tenant f8207c03fd1e4b4aaf123efea4662819 | bbb53715-f4e9-4ce3-bf2b-44b2aed2f4ef 169.254.192.0/18 |
| d990778b-49ea-4beb-9336-6ea2248edf7d | private                                            | b7fe4e86-65d5-4e88-8266-88795ae4ac53 10.1.0.0/28      |
| fde31a29-3e23-470d-bc9d-6218375dca4f | public                                             | 2e1d865a-ef56-41e9-aa31-63fb8a591003 172.16.0.0/24    |
+--------------------------------------+----------------------------------------------------+-------------------------------------------------------+
```


The keepalived processes for each router communicate with each other through an HA network which is also created at this time. The HA network name will use take the form ha-<tennant UUID> and can be seen by running neutron net-list. An HA port is generated for each router namespace along with a veth pair on the network nodes hosting the router namespace, where one veth member, with the name ha-<left most 11 characters of the port UUID>, placed into the router namespace and the other veth pair member, with the name tap<left most 11 characters of the port UUID>, placed into a Linux bridge, named brg<Left most 11 chars of the HA network UUID>. A VXLAN interface using the HA network segmentation ID is added to the Linux bridge to complete the communication path. The interface within the router namespace is assigned the IP range of 169.254.???.???/24, where the third octet of the IP is unique to each tenant and the forth octet unique to each ha interface.  The keepalived processes within each router namespace will communicate with each other using vrrp and elect a master router. The master router then adds all of the router VIPs (gateway IPs and external IP) to its interfaces and all other routers are placed into backup mode.

Show network node qrouter namespace on the master node:
```
ip netns exec qrouter-744e386d-03de-4993-8ab2-3b55b78a22e2 ip a
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
```
Show network node qrouter namespace on the backup node:
```
ip netns exec qrouter-557bf478-6afe-48af-872f-63513f7e9b92 ip a
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
```
Network node Linux bridges:
```
brctl show
bridge name     bridge id               STP enabled     interfaces
brq4bc242e0-97          8000.2eefb68bea30       no              tap158c1d10-c5
                                                        tap4310e4c7-de
                                                        vxlan-101
brqb304e495-b8          8000.921bf69da9dd       no              tap0d039391-92
                                                        vxlan-102
brqd990778b-49          8000.1a5ce98d92e2       no              tap670e2e87-5f
                                                        tapc5914ae6-d1
                                                        vxlan-100
brqfde31a29-3e          8000.eebd5cd87645       no              eth2
                                                        tapa41a7d54-94
```

VRRP communication from one netwoek node to the other:
```
ip netns exec qrouter-744e386d-03de-4993-8ab2-3b55b78a22e2 tcpdump -e -n -vvv -l -i ha-0d039391-92
tcpdump: listening on ha-0d039391-92, link-type EN10MB (Ethernet), capture size 65535 bytes
16:00:39.994393 fa:16:3e:d9:c0:7c > 01:00:5e:00:00:12, ethertype IPv4 (0x0800), length 54: (tos 0xc0, ttl 255, id 36898, offset 0, flags [none], proto VRRP (112), length 40)
    169.254.192.6 > 224.0.0.18: vrrp 169.254.192.6 > 224.0.0.18: VRRPv2, Advertisement, vrid 1, prio 50, authtype none, intvl 2s, length 20, addrs: 10.1.0.1
16:00:41.995826 fa:16:3e:d9:c0:7c > 01:00:5e:00:00:12, ethertype IPv4 (0x0800), length 54: (tos 0xc0, ttl 255, id 36899, offset 0, flags [none], proto VRRP (112), length 40)
    169.254.192.6 > 224.0.0.18: vrrp 169.254.192.6 > 224.0.0.18: VRRPv2, Advertisement, vrid 1, prio 50, authtype none, intvl 2s, length 20, addrs: 10.1.0.1
16:00:43.997403 fa:16:3e:d9:c0:7c > 01:00:5e:00:00:12, ethertype IPv4 (0x0800), length 54: (tos 0xc0, ttl 255, id 36900, offset 0, flags [none], proto VRRP (112), length 40)
    169.254.192.6 > 224.0.0.18: vrrp 169.254.192.6 > 224.0.0.18: VRRPv2, Advertisement, vrid 1, prio 50, authtype none, intvl 2s, length 20, addrs: 10.1.0.1
16:00:45.998820 fa:16:3e:d9:c0:7c > 01:00:5e:00:00:12, ethertype IPv4 (0x0800), length 54: (tos 0xc0, ttl 255, id 36901, offset 0, flags [none], proto VRRP (112), length 40)
    169.254.192.6 > 224.0.0.18: vrrp 169.254.192.6 > 224.0.0.18: VRRPv2, Advertisement, vrid 1, prio 50, authtype none, intvl 2s, length 20, addrs: 10.1.0.1
```

The keepalived processes for a set of HA routers then monitor each other using VRRP multicasts. If the master router fails, it is detected due to a loss of its VRRP multicasts, a new master router will be elected and the VIPs are moved onto the new master router. When a failure occurs the conntrackd processes ensure that any existing TCP connection states exist on all of the backup routers so that the connections migrate smoothly over to the new master router preventing connection loss.


#Packet Flow through Linux bridge HA router environment 

Packet flow through HA routers is identical to the path used in the Linux bridge using a single router. The master HA router will be the same as the single router. See that section for more details.












