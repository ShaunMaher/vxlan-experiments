# VXLAN, BGP, LXC and KVM
## Overview
I'm trying to make work and really understand how to use GoBGP, Bagpipe-BGP and
Linux's built in bridging capabilities to connect more than two VM/Container
hosts so the VMs/Containers all share the same layer 2 domain using VXLAN.

This is what I understand so far (and it might not be correct)...

VXLAN encapsulates ethernet frames within UDP packets.  These UDP packates can
be transmitted over a normal layer 3 IP network (even the internet though
encryption isn't included by default).  Two end points on a public or private is
simple to set up because you can simply add the remote node's IP to each VXLAN
interface and they know where to find each other.

For more than two end points you need a way of handling BUM packets such as ARP.
This is the part I'm still getting my head around.  There are several answers:
* Setup a multicast domain - I don't want any dependencies on switch smartness
  or proprietry/expensive routers.  Ok, I don't even if those are required but
  I'm aiming for simple and possibly over the internet or IPSec or similar and I
  don't want to complicate things.
* Unicast
 * Unicast with static flooding - each host maintains a list of remote host IPs
   that BUM packets are sent to.  Each time you add a host, you need to tell all
   other hosts about this new host.
 * Unicast with static L2 entries - similar to the above but you are also
   manually maintaining a list of local and remote container/VM MAC addresses
   and the IPs of the hosts they are on.  More efficient than the above because
   less traffic duplicated and sent to all BUM hosts.
 * Unicast with static L3 entries: Similar to the above although this sees a
   list of all MACs and configured on each host.  No BUM hosts involved.  More
   entries to manually keep synchronised though.  This might be a good design
   for a network overseen by an IPAM where the list on each host can be
   maintined by some sort of agent using the IPAM as the source of truth.
 * Unicast with dynamic L3 entries: MACs and IPs are kept in a central
   repository and an agent updates the entries on each host.

We are going to setup the "Unicast with dynamic L3 entries" option using GoBGP
as the central repository of end points and MAC addresses and bagpipe-bgp as the
agent that updates the L3 entries.

At the time of writing, most of the online documentation assumes that:
* You're using OpenStack, Docker or Kubernetes which do this stuff internally
  so you don't need to know how the internal parts work.  bagpipe-bgp is now a
  part of the OpenStack software suite but it doen't rely on it.
* You're going to use Cumulus Quagga or one of it's forks or derivitives.  As
  far as I can tell, this seems to provide a software switch and router like
  environment with a Cisco like command line interface.  This would be great if
  we were building a linux based switch but it seems like overkill for this
  task, at least at these early stages.

Most of the above also adds a fair amount of overhead.  If you have 3+ machines
and want to run a mix of containers and Qemu/KVM/Libvirt VMs, I don't think you
should need most of the features the above offer.  If we can get this working
with one container (or two for HA) with one piece of software installed and one
agent on each host, we have just what we need and no more.  Also, we have learnt
the basics so we can step up to one of the above suites and understand what they
are doing underneith.

## References
https://vincent.bernat.im/en/blog/2017-vxlan-linux
https://vincent.bernat.im/en/blog/2017-vxlan-bgp-evpn
http://murat1985.github.io/kubernetes/cni/2016/05/15/bagpipe-gobgp.html
https://docs.openstack.org/networking-bagpipe/latest/user/bagpipe-bgp.html
https://blog.scottlowe.org/2013/09/04/introducing-linux-network-namespaces/

## Create a container to run GoBGPD
Alternatives to consider:
* Install on existing pfSense router either within a jail or not.  Gain some of
  pfSense's built in HA features, etc.  I couldn't find any examples of
  OpenBGPD (available in pfSense as a package) used as a route reflector.  More
  Googling!
```
lxc init ubuntu:16.04 bgp1-clm
lxc config set bgp1-clm raw.lxc "lxc.aa_allow_incomplete = 1"
lxc start bgp1-clm
```

### Install prerequisites
```
sudo apt install build-essential
```

### Install the latest Go
```
wget https://redirector.gvt1.com/edgedl/go/go1.9.2.linux-arm64.tar.gz
sudo tar -C /usr/local -xzf go1.9.2.linux-arm64.tar.gz
```

### Use Go to install GoBGP
```
/usr/local/go/bin/go get github.com/osrg/gobgp/gobgpd
/usr/local/go/bin/go get github.com/osrg/gobgp/gobgp
```

### Basic configuration
My initial two host machines are 172.30.1.247 and 172.30.1.245 and my BGP
container is 172.30.1.204.

* Can we use DNS names?
* Can we wrap a script to populate from some centralised list?
* Hosts can be added after startup via command line tool but may not be
  persistant

```
[global.config]
  as = 64512
  router-id = "172.30.1.204"

[[neighbors]]
[neighbors.config]
  neighbor-address = "172.30.1.247"
  peer-as = 64512
[neighbors.route-reflector.config]
  route-reflector-client = true
  route-reflector-cluster-id = "172.30.1.204"
[[neighbors.afi-safis]]
[neighbors.afi-safis.config]
  afi-safi-name = "l2vpn-evpn"

[[neighbors]]
[neighbors.config]
  neighbor-address = "172.30.1.245"
  peer-as = 64512
[neighbors.route-reflector.config]
  route-reflector-client = true
  route-reflector-cluster-id = "172.30.1.204"
[[neighbors.afi-safis]]
[neighbors.afi-safis.config]
  afi-safi-name = "l2vpn-evpn"
```

## Install bagpipe-bgp on the host nodes
### For ubuntu version later than 17.10
```
sudo apt install python-networking-bagpipe
```

### Older versions of Ubuntu
```
sudo apt update
sudo apt-get install zlib1g-dev python-pip
```

```
sudo -H pip install networking-bagpipe==7.0.0
sudo mkdir /etc/bagpipe-bgp
sudo vim /etc/bagpipe-bgp/bgp.conf
```

```
[BGP]
local_address=172.30.1.247
peers=172.30.1.204
my_as=64512
enable_rtc=True
[API]
api_host=localhost
api_port=8082
[DATAPLANE_DRIVER_IPVPN]
dataplane_driver = DummyDataplaneDriver
mpls_interface=eth1
ovsbr_interfaces_mtu=4000
[DATAPLANE_DRIVER_EVPN]
dataplane_driver = linux_vxlan.LinuxVXLANDataplaneDriver
ovsbr_interfaces_mtu=4000
```

## First test
To do a basic test to ensure the parts are working we can setup a namespace on
each of our hosts and then make sure they can ping each other.
bagpipe-rest-attach can do everything for us: set up the namespace, set up the
vxlan interface, etc.

On one host:
```
sudo badpipe-bgp
sudo bagpipe-rest-attach --attach --port netns --ip 10.1.1.1 --network-type evpn --vpn-instance-id test2 --rt 64512:100 --vni 100
```

On another host:
```
sudo bagpipe-rest-attach --attach --port netns --ip 10.1.1.2 --network-type evpn --vpn-instance-id test2 --rt 64512:100 --vni 100
```

Now, on the first host, ping the second from within the "test2" network
namespace:
```
ip netns exec test2 ping 10.1.1.2
```

## Utilizing linux bridging
**Note:** None of the following worked!  Work in progress.

We are going to create a VXLAN with VNI (somewhat like a vlan ID) of 100:
```
sudo ip link add vxlan100 type vxlan id 100 dstport 4789 local 172.30.1.247 nolearning
sudo brctl addbr br100
sudo brctl addif br100 vxlan100
sudo brctl stp br100 off
sudo ip link set up br100
sudo ip link set up vxlan100

on e6410 for isr-vxlan VM:
sudo bagpipe-rest-attach --attach --port br100 --vni=100 --network-type evpn --rt 64512:100 --ip 192.168.100.254 --mac 52:54:00:c0:b8:9d

on e6510 for freebsd11_i386
sudo bagpipe-rest-attach --attach --port br100 --vni=100 --network-type evpn --rt 64512:100 --ip 192.168.100.2 --mac 52:54:00:e5:5f:74



sudo bagpipe-rest-attach --attach --port br100 --vni=100 --network-type evpn --rt 64512:100 --ip 192.168.100.0/24 --mac 52:54:00:99:99:99
```

## Integrating with KVM and LXC
### This works for adding a container to an vxlan
* Find the randomly generated veth device name for the container which is generated when the container starts:
```
lxc info <container name> | grep veth
```
* You need to know the IP the container has/will have (rules out DHCP?)
* You need to know the MAC address of the interface INSIDE the container.  This will be different to the MAC address of the veth device outside the container:
```
lxc config show <container name> --expanded | grep 'volatile.*.hwaddr'
```
* Unless there is a way to tell lxc to create the veth devices but not attach
  them to a bridge, your veth device will be attached to a bridge (create a
  consistantly named dummy bridge?).  You need to disconnect it from that bridge:
```
sudo brctl delif <bridge name> <veth device name>
```
* Now have bagpipe connect itself to (better explanation?) the interface:
```
bagpipe-rest-attach --attach --port veth6GX5SM --ip 10.1.1.3/24 --network-type evpn --vpn-instance-id vxlan100 --rt 64512:100 --mac 00:16:3e:6a:cf:ef --vni 100
```

This will probably also work for the veth devices attached to qemu VMs.

### Still to be worked out:
* Turn the above into hook scripts?
* DHCP can't be used?  We need to know the IP before we can connect to the vxlan and we can't DHCP until we're connected to the vxlan.
 * Use 0.0.0.0 on startup, then run again when DHCP address assigned (nope, didn't work)?
 * DHCP relay somewhere?
 * Use "dhtest" tool to generate a dummy DHCP request and inject it directly into the vxlan interface to a known DHCP server (i.e. not broadcast which doesn't work)?
* If DHCP is simply a no-go, store desired IP address in container/VM metadata so accessible to hook scripts.
* How can we mix in IPSec/OpenVPN for remote devices?
 * Proof-of-concept device: VM on laptop at remote site connecting to the L2 network at home site.
 * Laptop connected to home site via VPN for routed L3 traffic

### A gateway onto a non-vxlan:
On one host (more, each host?):
* Create a bridge that contains your physical network port (or bond or vlan, etc) TODO: network-manager version:
```
sudo brctl addbr brLAN
sudo brctl addif brLAN eno1
```
* Move the IP address from the physical network port to the bridge
* Use bagpipe-rest-attach to setup the basics of the vxlan interface:
```
sudo bagpipe-rest-attach --attach --port <physical network port> --ip 172.30.1.247/16 --network-type evpn --vpn-instance-id=vxlan100 --rt 64512:100 --mac 5c:26:0a:56:94:10 --vni 100
```
* Move the vxlan interface that was created from the bridge that was created to your bridge:
```
sudo brctl delif evpn---vxlan100 vxlan--vxlan100
sudo brctl addif brLAN vxlan--vxlan100
```
* Call the bagpipe rest api directly to tell it about the bridge (does this do anything?  It doesn't seem to monitor the bridge for MAC addresses.  I looked at the code and I see nothing for monitoring a bridge):
```
wget -q -O- --post-data='{"import_rt": ["64512:100"], "lb_consistent_hash_order": 0, "vpn_type": "evpn", "vni": 0, "vpn_instance_id": "vxlan100", "ip_address": "172.30.1.247/16", "export_rt": ["64512:100"], "local_port": { "linuxif": "eno1"}, "linuxbr": "brLAN", "advertise_subnet": false, "attract_traffic": {}, "gateway_ip": "172.30.0.2", "mac_address": "5c:26:0a:56:94:10", "readvertise": null}' 'http://127.0.0.1:8082/attach_localport
```
* You still need to tell GoBGP (via bagpipe) about all of the IPs/MACs that are accessible via the bridge (better design pending).  To add the IP and MAC for our gateway:
```
wget -q -O- --post-data='{"import_rt": ["64512:100"], "lb_consistent_hash_order": 0, "vpn_type": "evpn", "vni": 0, "vpn_instance_id": "vxlan100", "ip_address": "172.30.0.2/16", "export_rt": ["64512:100"], "local_port": {"linuxif": "eno1"}, "linuxbr": "brLAN", "advertise_subnet": false, "attract_traffic": {}, "gateway_ip": "172.30.255.254", "mac_address": "00:01:c0:06:d7:68", "readvertise": null}' 'http://127.0.0.1:8082/attach_localport'
```

#### Still more notes
* Read this: https://stgraber.org/2016/10/27/network-management-with-lxd-2-3/
* So, it looks like bagpipe does nothing special on bridge interfaces that would make it harvest IPs/MACS and send to GoBGP.  It wouldn't be too difficult to create something and maybe have it tap into l2miss and l3miss but I had hoped that bagpipe would be doing this.
 * l2miss and l3miss enable, edit line ~125 of linux_vxlan.py
* You can manually allow a guest to access a remote host that is not published via BGP with the following commands on the guest's host:
```
sudo ip neighbor replace 172.30.1.245 lladdr 18:3d:a2:10:98:44 dev vxlan--vxlan100 nud permanent
sudo bridge fdb replace 18:3d:a2:10:98:44 dst 172.30.1.247 dev vxlan--vxlan100 permanent
```
 * 172.30.1.245: Remote host we're granting access to
 * 18:3d:a2:10:98:44: MAC address of remote host
 * 172.30.1.247: Our VXLAN to LAN gateway's IP
* Have I simply missed something that would make normal ARP work?
 * There is code for configuring BUM endpoints on line ~293 of linux_vxlan.py
* If we're going to be writing something ourselves, can we:
 * If the host is on the L2 network that the guests will be connecting to, give them a direct path to the network.  We might be able to do this by having each host have it's own as namespace, having it advertise all routes to the network via itself to it's private as namespace and all routes to it's guests to a global as namespace.  It then subscribes to it's private namespace and the global namespace.

#### Adding a dummy interface with network manager
```
nmcli connection add type dummy ifname dummy0 con-name dummy0 autoconnect yes save yes
nmcli connection modify dummy0 ipv4.method static ipv4.addresses 172.31.5.1/25
```

#### Adding a bridge with a dummy interface as it's only member (for connecting VMs/containers to later)
```
nmcli connection add type bridge stp no ifname brDUMMY con-name brDUMMY
nmcli connection add type bridge-slave master brDUMMY slave-type bridge ifname dummy0 con-name dummy0 autoconnect yes save yes
```
OK.  At this point you need to cheat a little bit.  I couldn't find a way to
make nmcli add a bridge slave with the "dummy" type.  The above added an
ethernet interface which can't come up because there is no real ethernet
hardware associated with it.  For now, you can cheat by manually editing the
config file:
```
sudo vim /etc/NetworkManager/system-connections/dummy0
```
Change
```
type=ethernet
```
to
```
type=dummy
```

Restart NetworkManager to have the change take effect:
```
sudo service network-manager restart
```
