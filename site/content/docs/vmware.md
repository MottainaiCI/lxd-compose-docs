# LXD Compose & VmWare

The mission of this section is try to describe some use cases about using
LXD Compose over a VMWare stack to supply production-ready services and
help the systemist and developers life.

In particular, over my work experiences I saw a lot of different requirements
exposed by my Clients. Just to help the reader I will try to describe in addition
of the behavior of the specific use case what are the pro & cons.

The described use cases are:

[**Vmware-LXD 1:1**]({{< relref "/docs/vmware-1-1" >}}): this scenario follow
the idea of the [Kata Containers](https://katacontainers.io/) where are used
the VMWare VMs as nodes where install LXD standalone instances where `lxd-compose` 
could be configured to delivery the services over 1 container for 1 VM.

[**Vmware-LXD 1:N**]({{< relref "/docs/vmware-1-n-forward" >}}): this scenario
it's maybe the more used where inside a VM and a single LXD instance are
started more of one container and with different means are exposed service
over specific ports, etc.
