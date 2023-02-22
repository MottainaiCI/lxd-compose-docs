# VmWare-LXD 1:1

In the [Kata Containers](https://katacontainers.io/) technology it's used
an Hardware Virtualization to supply an additional isolation with a lightweight VM
and individual kernels.

In a similar way the use case describe here try to use a single Linux VM over VmWare
where install a standalone LXD instance and then through `lxd-compose` deploy one or
more services using a Physical vNic that is managed by LXD and added from the VM
to the Container deployed.

![LXD Compose Vmware Stack](../../images/lxdc-vmware-stack1.png#vmware)


{{< hint info >}}
In this scenario to supply the classic HA services it's needed follow
exactly the steps that normally are follow on delivery HA service directly
over VMs. This means deploy multiple VMs with the same services (for example two
nodes for Nginx Server) and eventually using VIPs.
{{< /hint >}}

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
    addr: https://macaronios.mirror.garr.it/images/lxd-images
    protocol: simplestreams
    public: true
  myserver:
    addr: https://mysimplestreams.server.local/lxd-images
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

As described in the schema over the LXD instances is configured a LXD profile
that map the VM vNIC `srv0` of the VM to the container with the same name.
The nictype is `physical` is the type implemented by LXD to support this feature.

*Using native bridge or OpenVswitch bridge attached to a VmWare vNic is
something that requires a specific setup over VmWare because the vSwitch
uses MAC caching and MAC filter that often doesn't permit to have something
that work correctly*. For this reason and because not always the people that
implement and deploy the services are the same that control the VmWare the uses
of the `physical` interface over the container is the best solution to follow
because it's transparent to VmWare. The MAC address assigned to the container is
the same mapped over VmWare. An alternative is to use the port map between the LXD
containers and the VM vNIC but it's another use case describe later.


In the example, it's been used only one network interface to supply services
but there aren't limitations on this. Could be configured multiple vNics at VmWare
level that are assigned to the containers, for example to supply an SSH service
through a management interface directly in the container.

For this solution it's a good idea to prepare a VmWare VM Template that is used
to create new VMs to attach to the service when it's needed to scale or just
upgrade the OS and reduce the offline time with an upgrade over a VM not attached
to the running services.

## Dynamic nodes with the LXD Compose render engine

Before describe how could be possible using the *render engine* to define the
`lxd-compose` specifications to manage the service setup I want remember to the
reader that a specific connection with an LXD instance could be defined only
at *group* level. It's possible that multiple groups could be connected to the
same LXD instance but it's not true that the same group could be used to control
multiple LXD instance at the same time (excludes the use case with LXD Cluster not
covered by this scenario).
Based on this concept a defined group is mapped to a specific VM and on using this scenario
a solution that works pretty well it's the use of the render engine to specify the
list of the remotes (or LXD instance) to reach for a specific service following the
pattern described hereinafter.

The render file will supply the list of the remotes assigned for a specific service,
for example the NGinx servers that exposes the HTTP resources. The render file is
defined directly over the `.lxd-compose.yml` or `--render-values|--render-default`
options.

```bash
$ cat .lxd-compose.yml
general:
  lxd_confdir: ./lxd-conf

render_default_file: ./render/default.yaml
render_values_file:  ./render/prod.yaml

env_dirs:
- ./envs/
```

in this case are defined both *default* render and the *values* render.

The `prod.yaml` file could be configured in this way:

```yaml
release: "22.10"
nginx_nodes:
  - connection: "vm1"
    name: "nginx1"
  - connection: "vm2"
    name: "nginx2"
```

And these could be the options defined in the `default.yaml`:

```yaml
ephemeral: false
privileged_containers: false

```

The connections `vm1` and `vm2` are the remotes described before and defined in the *config.yml*
file.

This could be an example of the LXD Compose specs where is defined the
setup of the Nginx server.

```yaml
# Description: Setup the Nginx Production Service

version: "1"
include_storage_files:
- common/storages/default.yml

include_profiles_files:
- common/profiles/default.yml
- common/profiles/net-phy-srv.yml
- common/profiles/logs-disk.yml
- common/profiles/autostart.yml

projects:
  - name: "nginx-services"
    description: |
      Setup Nginx services.

    include_env_files:
      - myenv/vars/common.yml
# Include the variable file that contains
# the network configurations options.
{{ range $k, $v := .Values.nginx_nodes }}
      - myenv/vars/{{ $v.name }}-net.yml
{{ end }}

    groups:
{{ $groups := .Values.nginx_nodes }}
{{ $ephemeral := .Values.ephemeral }}
{{ range $k, $v := .Values.nginx_nodes }}

      - name: "{{ $v.name }}-group"
        description: "Setup Nginx Frontend Node {{ $v.name }}"
        connection: "{{ $v.connection }}"
        common_profiles:
          - default
          - net-phy-srv
          - logs-disk
        {{- if $privileged_containers }}
          - privileged
        {{ end }}
          - autostart

        ephemeral: {{ $ephemeral }}

        include_hooks_files:
          - common/hooks/systemd-net-static.yml
          - common/hooks/systemd-dns.yml
          - common/hooks/hosts.yml

        nodes:
          - name: {{ $v.name }}
            image_source: "nginx/{{ $.Values.release }}"
            image_remote_server: "myserver"

            hooks:
              - event: post-node-sync
                flags:
                  - config
                commands:
                  - >-
                    systemctl daemon-reload &&
                    systemctl enable nginx &&
                    systemctl restart nginx

            # Generate the nginx configuration file based on template
            # and project variables.
            config_templates:
              - source: nginx/templates/nginx.tmpl
                dst: /tmp/lxd-compose/myinfra/{{ $v.name }}/nginx.conf

            sync_resources:
              - source: /tmp/lxd-compose/myinfra/{{ $v.name }}/nginx.conf
                dst: /etc/nginx/
{{ end }}

```

Some helpful commands to analyze the specs are these:

```bash
$> lxd-compose project list
|     PROJECT NAME  |                               DESCRIPTION                               | # GROUPS |
|-------------------|-------------------------------------------------------------------------|----------|
| nginx-services    | Setup Nginx Services                                                    |        2 |

$> lxd-compose group list nginx-services
|   GROUP NAME   |             DESCRIPTION              | # NODES |
|----------------|--------------------------------------|---------|
| nginx1-group   | Setup Nginx Frontend Node nginx1     |       1 |
| nginx2-group   | Setup Nginx Frontend Node nginx2     |       1 |

```

### Deploy the service

The first time the all containers are down it's possible just deploy all nodes with the `apply` command:

```bash
$> lxd-compose apply nginx-services

```

When you want upgrade a production service it's a good practice replace one node a time in this way:

```bash
$> # Destroy the node nginx1.
$> lxd-compose destroy nginx-services --enable-group nginx1-group

$> # Deploy the new container related to the new release 22.10.01.
$> # The variable release could be passed in input or updated directly on default.yaml
$> lxd-compose apply nginx-services --enable-group nginx1-group --render-env "release=22.10.01"
```

The same could be done in for the second node when the first is up and running.

If you have a slow bandwitdh between the LXD images server could be helpful download the images
over the VMs before execute the upgrade with the `fetch` command:

```bash
$> lxd-compose fetch nginx-services
```

This command will download the LXD images over the configured LXD instances of the selected
project without destroy and/or create containers.

{{< hint info >}}
In the example it's used the image LXD with alias `nginx/<release>`. This image
is created and exposed over HTTPS in a separated step. If this is not available
it's possible to use the images available over Macaroni Simplestreams Server or
Canonical Server and just install packages in the `post-node-creation` phase
to prepare the container with all needed software.
{{< /hint >}}

If it's needed add a new node/VM you need just editing the file `prod.yaml` and the `config.yml`
to add the new node.

```bash
$> cat lxd-conf/config.yml
  ...
  vm3:
    addr: https://vm3.infra:8443
    auth_type: tls
    project: default
    protocol: lxd
    public: false
```

```bash
$> cat render/prod.yml
release: "22.10"
nginx_nodes:
  - connection: "vm1"
    name: "nginx1"
  - connection: "vm2"
    name: "nginx2"
  - connection: "vm3"
    name: "nginx3"
```

Again, if it's needed update the configuration files of all Nginx servers this could be
executed sequentially from `lxd-compose` with the same `apply` command.

In the reported example for every container is assigned a static IP address that is
defined in one *vars* file following this pattern that display the file `nginx1-net.yml`:

```yaml
envs:
  nginx1_resolved:
    - file: dns.conf
      content: |
        [Resolve]
        DNS=1.1.1.1
        FallbackDNS=8.8.8.8
        Domains=
        MulticastDNS=no
        LLMNR=no

  nginx1_hosts:
    # - ip: <ip>
    #   domain: <domain1>,<domain2>
    - ip: "10.10.10.1"
      domain: nginx-backend01
    - ip: "10.10.10.2"
      domain: nginx-backend02

  nginx1_net:
    - file: 01-srv0.network
      content: |
        [Match]
        Name=srv0
        [Network]
        Address=172.18.20.100/24
        Gateway=172.18.20.1
        LLMNR=no
        # Disable binding of port ::58
        IPv6AcceptRA=no


```

The hooks used for the configuration are `systemd-net-static.yml`, `systemd-dns.yml`
and `hosts.yml`.

### Upgrade the OS of the VM

Hereinafter a short description about what are the possibile steps to follow in Production
to replace the VM and move an active container over a new VM where is been upgraded the OS.

Obviously, it's possible upgrade the OS and the LXD instance without replacing the VM
but we will try to share a way that could be used for rollback if something goes wrong.

So, the first step is to clone the existing VM or just create a new VM from a template.
If the choice is cloning, just disable service network from VmWare before boostrap of the VM.

![LXD Compose Vmware VM Upgrade S1](../../images/lxdc-vmware-upgrade-s1.png#vmware)

The new VM with a specific management interface could be upgraded meantime the service
is up and running. Until the network interface srv0 is not assigned to the container is
visible in the VM.

When the new VM is ready the steps to follow are describe in the image hereinafter:


![LXD Compose Vmware VM Upgrade S2](../../images/lxdc-vmware-upgrade-s2.png#vmware)

In short, through the `lxd-compose` tool it's needed destroy the existing container.
Eventually, you can just create a backup of the container with the normal `lxc` tool
or copy it before run the *destroy* command.

```bash
$> # Create the backup of the container for the rollback.
$> lxc copy vm1:nginx1 vm1:nginx-bkp1
$> # Or just stop the existing container.
$> lxc stop vm1:nginx1
```

If the container is stopped with `lxc` command the *destroy* command is not needed.

When the container is stopped or destroyed, the service IP address 172.18.20.100
of the example will be reused in the deploy of the new container over the new VM.
To maintain the same name of the container it's only needed modify the `prod.yml`
file to have `vm1-new` (or any other name choiced) that is the name of the
remote of the new VM.

So, in the `prod.yml` the content become:

```yaml
release: "22.10"
nginx_nodes:
  - connection: "vm1-new"
    name: "nginx1"
  - connection: "vm2"
    name: "nginx2"
  - connection: "vm3"
    name: "nginx3"
```

After this change it's only needed to execute the *apply* command that deploy the
new container:

```bash
$> lxd-compose apply --enable-group nginx1-group
```

If something goes wrong and the user want rollback the previous container it's needed:
shutdown the new VM; if the container is been stopped and not destroyed, just rerun
the *apply* command with the `connection` reconfigured to `vm1`.

If the container is been destroyed and it's been used a production-ready LXD image
the user could just redeploy the new container with the previous `release`.

## Best Practices for Production environments

As all know, today all Linux distribution are `Rolling Release` for different
reasons: the world go ahead very fast, CVE and security issues that are identified
every day requires fast updates, etc. This means that what is deployed at time T0
at the 99% is not installable ad the same way at time T1. So, for production
services it's better to prepare LXD images that will be used on delivery without
execute OS upgraded on container creation.

The upgrades of the container OS must be apply in a testing environment where it's
possible verify that there aren't regression with a new LXD images that when the
QA are passed could be used to upgrade *production* environment.

The [Simplestreams Builder](https://github.com/MottainaiCI/simplestreams-builder/)
could be used to prepare an HTTPS endpoint where expose LXD images used in the
installation over the Simplestreams Protocol. The alternative is to use directly
another LXD Instance as LXD images supplier.

For my experience it works well to try using an `environment` specification with a
single project to define a group of VMs that supply a specific service. This will
simplify the upgrade process with the use of one `group` for every LXD instance
as described in the previous chapter. Users are free to find their correct way.
Having multiple projects defined in the same environment file it works too like
to have a static definition of the groups without using render engine.

Using the `physical` nictype doesn't permit to see the network interfaces from
the VM OS, so it's important to supply on the created LXD images the right tools for
throubleshooting and analyses (tcpdump, ethtool, etc.).

Using a VmWare VM already supply a good isolation but to improve the security
levels using the unprivileged containers is the best choice.

In a production environment often it's important to maintain the logs files of
exposed services for subsequent analyzes. This could be handled through the mount
of the VM path inside the container that is persistent between the upgrade of one
container with another but not if the VM is replaced. But there are different ways
to resolve this efficiently for example through an Rsyslog remote server or over
Vmware using a secondary disk that is detached and attached to the new VM after
the upgrade. In the presented example the profile used for this mission is
[logs-disk](https://github.com/MottainaiCI/lxd-compose-galaxy/blob/master/envs/common/profiles/logs-disk.yml).

{{< hint info >}}
The use of additional API to setup and create a VM through the VmWare API
it's out of scope of this guide. This doesn't mean that is not possible.
`lxd-compose` permits to define hooks `pre-project` related to the node *host* that means
the local shell where lxd-compose is executed. Inside that hook it's possible
execute a bash script or other that prepare the VmWare VM before execute the
delivery of the container.
{{< /hint >}}
