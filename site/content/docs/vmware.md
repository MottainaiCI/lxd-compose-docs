# LXD Compose & VMWare

The mission of this section is try to describe some use cases about using
LXD Compose over a VMWare stack to supply production-ready services and
help the systemist and developers life.

In particular, over my work experiences I saw a lot of different requirements
exposed by my Clients. Just to help the reader I will try to describe in addition
of the behavior of the specific use case what are the pro & cons.

One of these scenaries follow the idea of the [Kata Containers](https://katacontainers.io/)
where are used the VMWare VMs as nodes where install LXD standalone instances
where `lxd-compose` could be configured to delivery the services over 1 container for 1 VM.
Hereinafter, this use case is called `Vmware-LXD 1:1`.

## VMware-LXD 1:1

In the [Kata Containers](https://katacontainers.io/) technology it's used
an Hardware Virtualization to supply an additional isolation with a lightweight VM
and individual kernels.

In a similar way the use case describe here try to use a single Linux VM over VmWare
where install a standalone LXD instance and then through `lxd-compose` deploy one or
more services using a Physical vNic that is managed by LXD and added from the VM
to the Container deployed.

![LXD Compose Vmware Stack](../../images/lxdc-vmware-stack1.png#vmware)


As visible in the image every VM is configured with two differents vNICs.

A `management vNic/Iface` that is only available over the Host/VM
and it's used to communicate with LXD HTTPS API (normally over the 8443 port)
and/or for SSH access (VM packages upgrades, maintenance, etc.).
It's a good idea to reach this interface only over a private VPN.

A `service vNic/iface` that is used over the container to supply services
configured in the container.

To ensure a more clean setup of the host a best practices is to rename the network's
interface with a name more oriented with the target infrastructure. In the example,
it's used the name `srv0` defined in the LXD profile assigned to the container to
deploy. The same could be done for the management interface with a name like `man0`.

To rename the network interfaces it's needed create an udev rule based on the MAC
addresses assigned from VmWare to the VM. For example, editing the file
`/etc/udev/rules.d/70-persistent-net.rules`:

```
SUBSYSTEM=="net", ACTION=="add", DRIVERS=="?*", ATTR{address}=="XX:XX:XX:XX:XX:XX", ATTR{dev_id}=="0x0", ATTR{type}=="1", KERNEL=="eth*", NAME="man0"
SUBSYSTEM=="net", ACTION=="add", DRIVERS=="?*", ATTR{address}=="XX:XX:XX:XX:XX:YY", ATTR{dev_id}=="0x0", ATTR{type}=="1", KERNEL=="eth*", NAME="srv0"
```

and then without the needed of reboot the VM just to execute:

```bash
$> # Ensure that the selected interface are down
$> ip link set eth0 down
$> ip link set eth1 down
$> udevadm control -R && udevadm trigger --action=add -v -s net
```

When the VMs are all configured with LXD reachable over HTTPS the `lxd-compose`
tool will reach every nodes through the different *remotes*. The file
`lxd-conf/config.yml` will be configured in this way:

```
default-remote: local
remotes:
  images:
    addr: https://images.linuxcontainers.org
    protocol: simplestreams
    public: true
  macaroni:
    addr: https://images.macaroni.funtoo.org/lxd-images
    protocol: simplestreams
    public: true
  local:
    addr: unix://
    public: false
  local-snapd:
    addr: unix:///var/snap/lxd/common/lxd/unix.socket
    public: false
  vm1:
    addr: https://vm1.infra:8443
    auth_type: tls
    project: default
    protocol: lxd
    public: false
  vm2:
    addr: https://vm2.infra:8443
    auth_type: tls
    project: default
    protocol: lxd
    public: false
aliases: {}

```

The remote `vm1` and `vm2` are the remotes of the VMs where deploy the
target services.

{{< hint info >}}
In this scenario to supply the classic HA services it's needed follow
exactly the steps that normally are follow on delivery HA service directly
over VMs. This means deploy multiple VMs with the same services (for example two
nodes for Nginx Server) and eventually using VIPs.
{{< /hint >}}


### Best Practices for Production environments
