# VmWare-LXD 1:N

A different approach is to use a single VM where configure
a single LXD instance to deploy multiple containers and supply
different services.

In these use cases, the VM could expose the services:

  * `with a floating IP` that is exposes over the VM network
    interface and the internal network is hidden. On this case
    it's used the Network Forward feature of LXD.

  * `through the PROXY protocol` to reach an internal service
    without expose a direct access from external network and
    the internal service. In this case normally, it used
    a reverse balancer like Nginx.

  * `through a NAT proxy device` to reach an internal service
    with a specific device proxy resource.

  * `through a proxy device on loopback of the device`
    with a specific device proxy resource.


{{< hint info >}}
The official documentation about *devices proxy* is available
[here](https://linuxcontainers.org/lxd/docs/latest/reference/devices_proxy/).
{{< /hint >}}


## Floating IP from VM iface

Between the features available in LXD and configurable through
`lxd-compose` exists the *Network Forwards* that permit to
define one or more Floating IP address configured in one or more
network interfaces of the node that will be used to define network
flows to redirect in the inside containers.

One of the way to understand how it works it through an example.

If we consider that the requirement is to expose two different TCP
services over two different ports with a single Floating IP:

  * port *8081* for the service of the application A

  * port *8080* for the service of the application B

the graph hereinafter, describe the behavior:

![LXD Compose Vmware Floating IP](../../images/lxdc_vmware_forward.png#vmware-medium)

The IP address *192.168.10.10* is our *floating IP* that is reachable
from the external network. In the example is used a Private Network IP but using a real Public IP is pretty the same.

In particular, the IP address *192.168.10.10* is assigned to the **srv0**
interface as primary ip address or as additional IP address.

```bash
4: srv0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP group default qlen 1000
    link/ether ee:ce:9f:0b:c8:7c brd ff:ff:ff:ff:ff:ff
    inet 192.168.0.92/24 brd 192.168.0.255 scope global noprefixroute srv0
       valid_lft forever preferred_lft forever
```

or

```bash

4: srv0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP group default qlen 1000
    link/ether ee:ce:9f:0b:c8:7c brd ff:ff:ff:ff:ff:ff
    inet 192.168.0.1/24 brd 192.168.0.255 scope global noprefixroute srv0
       valid_lft forever preferred_lft forever
    inet 192.168.0.92/24 scope global secondary srv0
       valid_lft forever preferred_lft forever
```

The *floating IP* is not managed by LXD and must be configured
manually.


The internal network bridge is **mottainai0** and is managed by LXD.

```bash
$ lxc network show mottainai0
config:
  bridge.driver: native
  dns.domain: mottainai.local
  dns.mode: managed
  ipv4.address: 172.18.1.249/23
  ipv4.dhcp: "true"
  ipv4.firewall: "true"
  ipv4.nat: "true"
  ipv6.dhcp: "false"
  ipv6.nat: "false"
description: Network mottainai0 created by lxd-compose
name: mottainai0
type: bridge
used_by:
- /1.0/instances/c1
- /1.0/instances/c2
- /1.0/profiles/net-mottainai0
managed: true
status: Created
locations:
- none
```

In the example it's used the native bridge but it works with OVS
bridge too.


The *Application A* is exposed in the internal container *C2* over
the port *8090* and with the IP 172.18.1.1.

The *Application B* is exposed in the internal container *C1* over
the port *8080* and with the IP 172.18.1.2.

{{< hint info >}}
To define Network Forward rules you need to know the IP addresses
of the target containers. So you could create the containers and
later create/update the forward or using static IP addresses in
the containers.
{{< /hint >}}

`lxd-compose` simplify the management of the Network Forward
more bloated through the `lxc` command.

A network forward listen address must be assigned to an existing
network and for this reason that the `lxd-compose` manage these
resources as extension of the network object.

```yaml
# LXD Compose specs to define Network Forward over a network device.
networks:
- name: "mottainai0"
  type: "bridge"
  config:
    bridge.driver: native
    dns.domain: mottainai.local
    dns.mode: managed
    ipv4.address: 172.18.1.249/23
    ipv4.dhcp: "true"
    ipv4.firewall: "true"
    ipv4.nat: "true"
    ipv6.nat: "false"
    ipv6.dhcp: "false"
  forwards:
    - listen_address: "192.168.10.10"
      ports:
        - protocol: tcp
          # Define a port or a port-range or a list of port.
          listen_port: "8081"
          target_address: "172.18.1.1"
          target_port: "8090"

        - protocol: tcp
          listen_port: "8080"
          target_address: "172.18.1.2"
```

So, after define the network specifications and the forwards rules
in the target *environment* just create them with `lxd-compose`:

```bash
$ lxd-compose  network create myproject mottainai0 --with-forwards -u
Network mottainai0 updated.
Network forwards of the net mottainai0 updated.
```

With the same command it's possible update the existing rules.

Indeed, the commands of `lxc` tool to run to check the configuration are:

```bash
$ lxc network forward list mottainai0
+----------------+-------------------------------------------------------------+------------------------+-------+
| LISTEN ADDRESS |                        DESCRIPTION                          | DEFAULT TARGET ADDRESS | PORTS |
+----------------+-------------------------------------------------------------+------------------------+-------+
| 192.168.10.10  | Network forward for ip 192.168.10.10 created by lxd-compose |                        | 2     |
+----------------+-------------------------------------------------------------+------------------------+-------+
```

And this to see the detail of a specific listen address:

```bash
$ lxc network forward show mottainai0 192.168.10.10
description: Network forward for ip 192.168.10.10 created by lxd-compose
config: {}
ports:
- description: ""
  protocol: tcp
  listen_port: "8081"
  target_port: "8090"
  target_address: 172.18.1.1
- description: ""
  protocol: tcp
  listen_port: "8080"
  target_address: 172.18.1.2
listen_address: 192.168.10.10
location: none
```

Under the hood LXD uses `iptables` or `nftables` to define a DNAT rule that permit to maintain
the origin source IP address when the connection reachs the internal node and a MASQUERADE rule
for the revert flow.

```bash

$ sudo iptables -t nat -L -n
Chain PREROUTING (policy ACCEPT)
target     prot opt source               destination         
DNAT       tcp  --  0.0.0.0/0            192.168.10.10         tcp dpt:8080 /* generated for LXD network-forward mottainai0 */ to:172.18.1.2:8080

```

The use of `target_port` attribute is needed only when the public
listening ports are different from the internal else you
can just define the `listen_port` attribute.

The official LXD documentation of the Network Forward feature
is available [here](https://linuxcontainers.org/lxd/docs/latest/howto/network_forwards/).

## Using PROXY protocol to reach an internal service

LXD permits to define *proxy devices* to allow forwarding network connections
between host and instance. This makes it possible to forward traffic hitting
one of the host’s addresses to an address inside the instance or to do the
reverse and have an address in the instance connect through the host.

Between the different types of connections (udp, tcp, unix) it's possible
forward the connection encapsuled over the PROXY protocol that transmit
the sender information.

Using PROXY protocol like this example:

```yaml
name: "mottainai-https"
description: "Profile for export HTTPS port to Host"
devices:
  https:
    bind: host
    connect: tcp:0.0.0.0:443
    listen: tcp:0.0.0.0:443
    nat: false
    proxy_protocol: true
    type: proxy
```

permits to avoid using a static IP address inside the container.

In this case, it's a LXD process that execute the binding of the port
and proxy the connection inside the container.

```bash
# # From the VM / Host
# netstat -lpn | grep lxd | grep 443
tcp6       0      0 :::443                  :::*                    LISTEN      1229/lxd
```

## Using NAT proxy device to reach an internal service

Always through the use of *device proxy* when the `nat` option
is enable the traffic is forwarded usint NAT than being proxied
through a separate connection, in this case you need ensure that the target
instance has a static IP configured in LXD on its NIC device.

To have a static IP address you need to configure the NIC device
with a configuration for the node similar to this:

```yaml
devices:
  eth0:
    ipv4.address: 172.18.1.1
    name: eth0
    nictype: bridged
    parent: mottainai0
    type: nic
  myservice:
    bind: host
    connect: tcp:172.18.1.1:11000
    listen: tcp:192.168.10.10:11000
    nat: "true"
    proxy_protocol: "false"
    type: proxy
```

Without configure a static IP address for the *eth0* device the bind with NAT fail.

In addition, when `nat` is true you can't use `listen` with `tcp:0.0.0.0` but you need
define the external IP address where LXD will configure the NAT rule.

I think that using this solution it makes sense when you have a very little installation,
else the Network Forward way is better.

Under the hood, using the `nat` generates the configuration of these iptables rules:

```bash
$ sudo iptables -t nat -L -n
Chain PREROUTING (policy ACCEPT)
target     prot opt source               destination         
DNAT       tcp  --  0.0.0.0/0            192.168.10.10         tcp dpt:11000 /* generated for LXD container c2 (myservice) */ to:172.18.1.1:11000

Chain OUTPUT (policy ACCEPT)
target     prot opt source               destination         
DNAT       tcp  --  0.0.0.0/0            192.168.10.10         tcp dpt:11000 /* generated for LXD container c2 (myservice) */ to:172.18.1.1:11000

Chain POSTROUTING (policy ACCEPT)
target     prot opt source               destination         
MASQUERADE  tcp  --  172.18.1.1          172.18.1.1        tcp dpt:11000 /* generated for LXD container c2 (myservice) */

```

## Using proxy device to reach an internal service on localhost

If the requirements of the called service is not identify the calling it's possible define
a proxy device rule that map an external port to a specific container through his loopback
interface.


```yaml
name: "myservice"
description: "Profile for export port 12000 to Host"
devices:
  https:
    bind: host
    connect: tcp:127.0.0.1:12000
    listen: tcp:0.0.0.0:12000
    nat: false
    proxy_protocol: false
    type: proxy
```

In this case an LXD process is configured in binding on the specified port and the traffic
is proxied inside the container in the selected port.

```bash
# # From the VM / Host
# netstat -lpn | grep lxd | grep 12000
tcp6       0      0 :::12000                  :::*                    LISTEN      1229/lxd
```

