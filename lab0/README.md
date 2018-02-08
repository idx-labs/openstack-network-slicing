# OpenStack DevStack and BGP

This README describes how to setup a DevStack instance and then connect it to a virtual Juniper router, establishing a BGP session using [Neutron Dynamic Routing](https://docs.openstack.org/neutron-dynamic-routing/latest/admin/bgp-speaker.html) to configure a BGP speaker that can announce dynamically created networks.

The two instances will be setup in a single, usually baremetal, KVM node using Libvirt.

## Network Diagram

IP addresses are arbitrary. Feel free to make changes there, otherwise this document will assume what is shown on the diagram is what is in use.

![Network Diagram](img/network-diagram.jpg)

## Requirements

* KVM node with enough resources to run devstack and a Juniper router
* Access to the Juniper vSRX image
* A Xenial virtual machine running on the KVM node

## Setup Networking

We need at least three libvirt networks.

1. Default network - This is created by Libvirt/Virsh, and will be used for management
2. A network for the BGP session to occur over
3. A network that will become the provider network, and where the Neutron routers will have their external gateway/interface.

The DevStack instance will have three interfaces, one on each of these networks. The same will occur for the Juniper vSRX router.

## Deploy Juniper Router

TBD

## Deploy DevStack

### Create Virtual Machine

TBD

### Configure DevStack Networking Post-Deployment

#### Setup Interfaces

DevStack will not configure the interface we intend to use as the BGP session interface.

Ensure ens4 is configured by entering the below into /etc/network/interfaces.d/ens4.cfg.

```
auto ens4
iface ens4 inet static
  address 10.55.0.2
  netmask 255.255.255.0
```

Then bring up the interface.

```
$ ifup ens4
```

Validate the interface.

```
$ ping 10.55.0.2
```

#### BGP DR Agent

Configure a unique router id in `/etc/neutron/bgp_dragent.ini`. For the purposes of this document, we will use the IP address that is being set on the ens4 interface, `10.55.0.2`.

```
$ grep -v "^#\|^$" bgp_dragent.ini
[DEFAULT]
debug = True
verbose = True
[bgp]
bgp_speaker_driver = neutron_dynamic_routing.services.bgp.agent.driver.ryu.driver.RyuBgpDriver
bgp_router_id = 10.55.0.2
```

Restart the Neutron BGP agent.

```
$ systemctl restart devstack@q-dr-agent.service
```

#### DevStack Network Configuration

There are a few things we need to setup on the DevStack instance.

Remove various preconfigured DevStack subnets and configure router.

```
openstack router remove subnet router1 private-subnet
openstack subnet delete private-subnet
openstack subnet pool delete shared-default-subnetpool-v4
openstack router unset --external-gateway router1
openstack subnet delete public-subnet
```

Setup address scopes.

```
neutron address-scope-create --shared public 4
neutron subnetpool-create --pool-prefix 172.24.4.0/24 \
  --address-scope public provider
neutron subnetpool-create --pool-prefix 10.0.0.0/16 \
  --address-scope public --shared selfservice
neutron subnet-create --name provider --subnetpool provider --prefixlen 24 public
neutron subnet-create --name selfservice --subnetpool selfservice  --prefixlen 24 private
```

Configure Neutron router.

```
openstack router set --external-gateway public router1
openstack router add subnet router1 selfservice
```

Setup the BGP speaker.

```
neutron bgp-speaker-create --ip-version 4 \
 --local-as 200 bgp-speaker
neutron bgp-speaker-network-add bgp-speaker public
```

#### Validate Advertised Routes

Show what routes will be announced/advertised.

```
$ neutron bgp-speaker-advertiseroute-list bgp-speaker
+-------------+------------+
| destination | next_hop   |
+-------------+------------+
| 10.0.0.0/24 | 172.24.4.6 |
+-------------+------------+
```

#### Add Peer

Create a peer then add it to the bgp-speaker instance.

```
neutron bgp-peer-create --peer-ip 10.55.0.1 \
  --remote-as 100 bgp-100-200
neutron bgp-speaker-peer-add bgp-speaker bgp-100-200
```

## Todo

* Show configuring Juniper router
* Show deploying KVM instance for DevStack
* Change neutron commands to be openstack commands
* Show announced routes, reachability on Juniper router
* Clean up network diagram, add interface names
