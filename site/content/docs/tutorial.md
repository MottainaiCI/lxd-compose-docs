---
bookCollapseSection: true
type: docs
---

# Tutorial

Hereinafter, how to create your first project in five steps. Enjoy!

## Setup LXD Environment

### Step 1: Install LXD

{{< tabs "uniqueid" >}}

{{< tab "Ubuntu Linux" >}}

```shell
$> apt-get install lxd-installer
$> apt-get install zfsutils-linux
$> apt-get install curl
```

In Ubuntu LXD is not available as a single package, but it uses *snapd*,
so `lxd-installer` is the installer package of LXD.
The installation of zfsutils is not mandatory but it prepares the
environment if you want to use ZFS with LXD.

Ubuntu supplies LXD through `snapd` service. There isn't a real package.

{{< /tab >}}

{{< tab "Macaroni OS" >}}

This is for *Macaroni Funtoo* release:

```shell
$> luet install -y app-emulation/lxd
```

If you follow the steps described in the `step 1b` lxd-compose manages the installation itself.

{{< /tab >}}

{{< tab "Funtoo Linux" >}}

Install and compile LXD with:

```shell
$> emerge app-emulation/lxd
```

The Funtoo documentaion is available [here](https://www.funtoo.org/LXD).

If you follow the steps described in the `step 1b` lxd-compose manages the installation itself.

{{< /tab >}}

{{< /tabs >}}

Follow the [installation](./getting-started/#installation) steps
to install `lxd-compose` binary.

Enable LXD API in binding (this is not needed if you want to use `local`
remote through unix socket):


#### Step 1a: Setup LXD instance manually

Some initial steps could be done manually for a simple installation.
So, just after that the service is running proceed with the configuration
of the API port binding and password with:

 
```shell
$> lxc config set core.https_address [::]:8443
$> lxc config set core.trust_password mypassword
```

If it's used LXD only with unix socket the previous commands are not needed.

To create containers and download image it's needed create the default storage:


```shell
$> lxc storage create default btrfs size=20GB
```

In the example it used BTRFS loopback mode but there are different options.

{{< hint info >}}
If you setup LXD instance with `lxd init --preseed` you can setup
some configurations directly with a YAML source.
{{< /hint >}}

#### Step 1b: Setup LXD instance through lxd-compose

On both [Macaroni OS](httsp://www.macaroni.funtoo.org) and [Funtoo](https://www.funtoo.org/) it is possible to use `lxd-compose` to setup LXD instance directly with the
project `lxd-setup-and-test` from
[LXD Compose Galaxy Project](https://github.com/MottainaiCI/lxd-compose-galaxy/).

New OS will be integrated in the future.

Hereinafter, the steps to follow.

1. Install `lxd-compose`, `jq` and `yq` tools. Follow these commands or just install from
   your Linux distro the same tools:

   ```shell

    $> sudo su
    $> curl https://raw.githubusercontent.com/geaaru/luet/geaaru/contrib/config/get_luet_root.sh | sh # Install luet on your system
    $> luet install -y app-emulation/lxd-compose utils/jq utils/yq # Install lxd-compose binary
   ```

2. Get the LXD Compose Galaxy project

   ```shell
    $> git clone https://github.com/MottainaiCI/lxd-compose-galaxy.git
    $> cd lxd-compose-galaxy
   ```

3. Choice the storage type between the available visible with this command:

   ```shell

    $> lxd-compose storage list lxd-setup-and-test
    |    STORAGES    | DRIVER |                         DOCUMENTATION                         |
    |----------------|--------|---------------------------------------------------------------|
    | dir-source     | dir    | Directory Storage Pool using existing path /lxd.              |
    |                |        |                                                               |
    |                |        | Using --render-env "storage_source=/mydir" to                 |
    |                |        | override existing /lxd path.                                  |
    |                |        |                                                               |
    | btrfs-source   | btrfs  | BTRFS Storage Pool using existing path or device.             |
    |                |        |                                                               |
    |                |        | Using --render-env "storage_source=/dev/sdX" to override      |
    |                |        | default /lxd path.                                            |
    |                |        |                                                               |
    | btrfs-loopback | btrfs  | BTRFS Storage Pool Loop disk.                                 |
    |                |        |                                                               |
    |                |        | Use --render-env "storage_size=200GB" to override             |
    |                |        | default 150GB size.                                           |
    |                |        |                                                               |
    |                |        | Use --render-env "storage_mount_options=compress=zstd:3"      |
    |                |        | to enable additional btrfs mount options for compression      |
    |                |        | or other.                                                     |
    |                |        |                                                               |
    |                |        | Other compressions options:                                   |
    |                |        |                                                               |
    |                |        | storage_mount_options=rw,relatime,space_cache,compress=zstd:3 |
    |                |        |                                                               |
    | zfs-source     | zfs    | ZFS Storage pool using existing ZFS Pool or dataset           |
    |                |        | or create a new ZFS Zpool on the specified device.            |
    |                |        |                                                               |
    | zfs-loopback   | zfs    | ZFS Loopback Storage pool.                                    |
    |                |        |                                                               |
    |                |        | Use --render-env "zfs_pool_name=pool" to override             |
    |                |        | default ZFS pool name lxd-compose-pool.                       |
    |                |        |                                                               |
    |                |        | Use --render-env "storage_size=XGB" to override               |
    |                |        | default storage size of 100GB                                 |
    |                |        |                                                               |
    | lvm-source     | lvm    | LVM Storage Pool using existing path or device.               |
    |                |        |                                                               |
    |                |        | Using --render-env "storage_source=/dev/sdx" to               |
    |                |        | override existing `lvm` volume group.                         |
    |                |        |                                                               |
    | lvm-loopback   | lvm    | LVM Loopback Storage pool.                                    |
    |                |        |                                                               |
    | ceph-source    | ceph   | Ceph Storage pool.                                            |
    |                |        |                                                               |
    |                |        | NOTE: Not tested.                                             |
    |                |        |                                                               |
   ```

   The description of the storage could not be describe all possibilies render options.
   After the identification of the storage type it's better check the storage specification
   to see all possible options directly on [repository](https://github.com/MottainaiCI/lxd-compose-galaxy/tree/master/envs/common/storages).

4. When the storage is selected it's needed create the render values file with the storage options.
   In the example the storage selected is `btrfs-loopback` with a size of 30GB and with compres:

   ```shell

    $> echo "
    storage_mount_options: compress=zstd:3
    storage_size: 30GB
    storage_name: btrfs-loopback
    storage_render_values: render/values.yaml
    " > render/values.yaml
   ```

   The last row is needed because the lxd-compose specs executes the `lxd-compose storage create`
   command after the setup of LXD instance.

5. Define the configuration file of your LXD instance with the editing of the file `envs/lxd-setup/vars/common.yaml`.

   Between the configuration options it's possible define what are the users and the subuid/subgid to
   use for the management of the unprivileged containers and that will be configured by the execution:

   ```yaml
    # Define the subuid/subguid to create
    lxc_subuids:
      - user: root
        home: /root
        map: 1000000-1065535

    lxc_subgids:
      - user: root
        home: /root
        map: 1000000-1065535
   ```

   And then the LXD instance configuration:

   ```yaml
    # LXD Instance configuration
    lxd_config:
      config:
        # Configure the LXD password
        core.trust_password: mysecret
        # It seems that for cluster it's better define the
        # server interface used.
        core.https_address: "[::]:8443"

        # Configure HTTPS proxy to use
        # core.proxy_https: "http://192.168.10.1:8080"
        # Configure HTTP proxy to use
        # core.proxy_http: "http://192.168.10.1:8080"
        # Configure Proxy Ignore hosts
        # core.proxy_ignore_hosts

        # See https://linuxcontainers.org/lxd/docs/master/server/
        # for all available options.

        # Number of days after which an unused cached remote
        # image will be flushed
        images.remote_cache_expiry: 5

        # Whether to automatically update any image that
        # LXD caches
        # images.auto_update_cached: true

        # Interval in hours at which
        # to look for update to cached images
        # (0 disables it)
        # images.auto_update_interval: 1
   ```

   The selected storage will be configured in the `default` profile of the LXD instance configured.

5. Now it's time to configure your LXD server and to run `lxd-compose` in the host where you
   want to do the setup (so use `scp` if it isn't your node or another tool).

   So, just run this in `Macaroni`:

   ```shell
    $> lxd-compose a lxd-setup-and-test --render-env "os=macaroni" --render-values render/values.yaml
   ```

   or this in `Funtoo`:

   ```shell
    $> lxd-compose a lxd-setup-and-test --render-env "os=funtoo" --render-values render/values.yaml
   ```

   Based on the selected OS the `lxd-compose` will install the packages (with the right PMS) and
   then will configure the LXD instance, it will create the LXD profiles and then it will test the setup.


{{< hint info >}}
You can always setup more of one storage too just using directly the `lxd-compose storage create`
command.
{{< /hint >}}

### Step 2: Configure LXD client

`lxd-compose` uses the LXD's client engine to call LXD API. This means
that you need configure your environment with a `config.yml` that
works with `lxc` too.

In general, the `lxc` client follows this steps on found the configuration
files with the remotes and certificates to use:

  * if it sets `LXD_CONF` variable env that it uses the path defined
    in the variable to search the `config.yml` and to read the
    certificates

  * if `LXD_CONF` variable is not set or the file `$LXD_CONF/config.yml`
    doesn't exist it search under `$HOME/.config/lxc`.

  * if `$HOME/.config/lxc` doesn't exist it creates the directory and
    the file `$HOME/.config/lxc/config.yml` with the default remotes.

In additional, there are some extra variables available that are described
[here](https://lxd.readthedocs.io/en/latest/environment/).

By default `lxc` uses the default remote `local` through LXD unix socket:

```yaml
default-remote: local
remotes:
  images:
    addr: https://images.linuxcontainers.org
    protocol: simplestreams
    public: true
  local:
    addr: unix://
    public: false
```

{{< hint warning >}}
It's not possible to remove the `local` remote, it's automatically added by LXD engine. To disable it (normally for the P2P Mode) you need set the field `lxd_local_disable` at *true* in the *general* section of `lxd-compose` config.
{{< /hint >}}

I think that a best practices to follow with `lxd-compose` (if it isn't used the
*$HOME* configuration file) is to create an `lxd-conf` directory where to register
the needed remotes of the `lxd-compose` projects and the certificates.

Hereinafter, an example of the *tree* created with these steps:

```shell
$> # Create your projects directory
$> mkdir -p $HOME/lxd-compose-projects/my-first-project
$> cd $HOME/lxd-compose-projects/my-first-project

$> # Create lxd-conf directory to use project LXD configuration.
$> mkdir lxd-conf

$> # Force `lxc` client to use project directory
$> # NOTE: Setting LXD_CONF variable is needed only for lxc client,
$> #       lxd-compose permits to avoid this with the specific option
$> #       descibed below.
$> export LXD_CONF=./lxd-conf

$> # Add remote related to the instance to use for my projects.
$> # In this case we use the same node to use lxd-compose and LXD instance.
$> lxc remote add mylxd-instance https://127.0.0.1:8443 --accept-certificate
To start your first instance, try: lxc launch ubuntu:18.04

Generating a client certificate. This may take a minute...
Admin password for mylxd-instance:

$> find lxd-conf/
lxd-conf/
lxd-conf/client.key
lxd-conf/servercerts
lxd-conf/servercerts/mylxd-instance.crt
lxd-conf/config.yml
lxd-conf/client.crt
```

In the example, the *remote* is an external host, if you want to use
directly the *local* remote you just need to run `lxc` one time to create the
right tree. Instead, if you want to use the LXD HTTP API locally you need
to add the remote related to `127.0.0.1` address.

When `lxc` client is been configured you are ready to setup `lxd-compose` config
of the project. You can test it with:
```shell
$> lxc list mylxd-instance:
```

If it works, compliments the step 1 is completed!

#### LXD installed from snapd

LXD available through snapd doesn't expose local unix socket under default path
`/var/lib/lxd/unix.socket` but normally under the path `/var/snap/lxd/common/lxd/unix.socket`.

This means that to use `local` connection it's better to create under the config.yaml an entry like this:

```yaml
  local-snapd:
    addr: unix:///var/snap/lxd/common/lxd/unix.socket
    public: false
```

and then to use `local-snapd` in `connection` option.

{{< hint warning >}}
With LXD >=5.0 that contains new features to create multiple file socket with different permissions
this configuration doesn't work correctly. I will update the documentation correctly soon.
{{< /hint >}}

{{< hint warning >}}
At the moment on Ubuntu the `lxc` command is supplied through snapd and it seems that
that wrapper doesn't propagate the environments variables to real binary. This means that
override `LXD_CONF`  doesn't work. The LXD remotes must be modified only in the snapd path.
Personally, I prefer to use OS where `lxc` is supplied as binary directly.
{{< /hint >}}

Instead, if it's used the HTTPS API this is not needed.

### Step 3: Configure LXD Compose configuration file

By default `lxd-compose` search for the file `$PWD/.lxd-compose.yml` else
it is possible to pass the configuration file path with the `-c|--config`
option.

In this case we override the LXD configuration path so what we need to do
is to create a configuration file like this where we set also the path
where `lxd-compose` search for the environment files.

```shell
$> echo "
general:
  debug: false
  lxd_confdir: ./lxd-conf

logging:
  level: "info"

# Define the directories list where lxd-compose search
# for environments files with .yml or .yaml extension.
env_dirs:
- ./envs" > .lxd-compose.yml
```

### Step 4: Configure LXD profiles and networks

This step is needed only if you want to define custom profiles to use
with your containers and a particolar network configuration.

If your profiles are been defined like your network configuration you can
skip this step.

If you have used the `step 1b` a lot of profiles and a network is already
available and configured.

{{< hint info >}}
Under the LXD Compose Galaxy project, you can find a lot of examples with
ready to use profiles that you can add to your project easily.
{{< /hint >}}

The next step is to create the project environment file:

```shell
$> mkdir envs/
$> echo "
version: \"1\"

# Using mottainai template engine
template_engine:
  engine: \"mottainai\"
" > envs/env1.yml
```

#### Prepare your profiles

Now we can define our profiles inside the file `envs/env1.yml` after the content
created above:

```yaml
profiles:
- name: "privileged"
  config:
    security.privileged: "true"
  description: Privileged profile
  devices:
    fuse:
      path: /dev/fuse
      type: unix-char
    tuntap:
      path: /dev/net/tun
      type: unix-char
    # Comment this if zfs is not available.
    zfs:
      path: /dev/zfs
      type: unix-char

- name: "net-mottainai0"
  description: Net mottainai0
  devices:
    eth0:
      name: eth0
      nictype: bridged
      parent: mottainai0
      type: nic

- name: default
  description: Default Storage
  root:
    path: /
    pool: default
    type: disk

- name: flavor-medium
  description: "flavor with 2GB RAM"
  config:
    limits.memory: 2GB

- name: flavor-big
  description: "flavor with 3GB RAM"
  config:
    limits.memory: 3GB

- name: flavor-thin
  description: "flavor with 500MB RAM"
  config:
    limits.memory: 500MB
```

To configure profiles in the LXD instance you need define at least one project
and one group. We need to add in the file `envs/env1.yml` this section:

```yaml
projects:
- name: "my-first-project"
  description: "This is my fist project."

  groups:
    - name: "my-first-group"
      description: "My first group"

      # Set the remote to use for the group.
      # In this case we use the remote mylxd-instance
      # created in the previous steps.
      connection: "mylxd-instance"

      # We use the ephemeral container for this tutorial.
      ephemeral: true
```

Now we are ready to initialize the LXD instance with our profiles:

```bash
$> lxd-compose profile create my-first-project -a
Profile privileged created correctly.
Profile net-mottainai0 created correctly.
Profile default already created correctly.
Profile flavor-medium created correctly.
Profile flavor-big created correctly.
Profile flavor-thin created correcly.

$> # Update existing profiles
$> lxd-compose profile create my-first-project -a -u
```

#### Prepare your networks

Also the network devices used by LXD could be configured from `lxd-compose`
and prepare the network with the right options used by the project.

In the file `envs/env1.yml` under the `profiles` section add the `networks`
section:

```yaml
networks:
  - name: "mottainai0"
    type: "bridge"
    config:
      bridge.driver: native
      dns.domain: mottainai.local
      dns.mode: managed
      ipv4.address: 172.18.10.1/23
      ipv4.dhcp: "true"
      ipv4.firewall: "true"
      ipv4.nat: "true"
      ipv6.nat: "false"
      ipv6.dhcp: "false"
```

Then we create the network:

```shell
$ lxd-compose network create my-first-project -a
Network mottainai0 created.

# To update configuration existing networks
$ lxd-compose network create my-first-project -a -u
```

You can check the result with:

```shell
$ lxc network list
+-----------------+----------+---------+----------------+---------------------------+-------------------------------------------+---------+
|      NAME       |   TYPE   | MANAGED |      IPV4      |           IPV6            |                DESCRIPTION                | USED BY |
+-----------------+----------+---------+----------------+---------------------------+-------------------------------------------+---------+
| eth0            | physical | NO      |                |                           |                                           | 0       |
+-----------------+----------+---------+----------------+---------------------------+-------------------------------------------+---------+
| eth1            | physical | NO      |                |                           |                                           | 0       |
+-----------------+----------+---------+----------------+---------------------------+-------------------------------------------+---------+
| mottainai0      | bridge   | YES     | 172.18.10.1/23 | fd42:30e5:6279:93f::1/64  | Network mottainai0 created by lxd-compose | 0       |
+-----------------+----------+---------+----------------+---------------------------+-------------------------------------------+---------+

```

There are a lot of options for setup network devices described in the
[LXD Project](https://lxd.readthedocs.io/en/latest/networks/).


### Step 5: Create your first project

We are ready to prepare our first deploy with `lxd-compose`.
In this tutorial, we use single file specs for simplicity.
My suggestion is to use the `include_env_files` and
`include_groups_files` when there are verbose specs.

We need open the file `envs/env1.yml` again and to complete our group specs:

```yaml
- name: "my-first-project"
  description: "This is my fist project."

  # We use online variable
  vars:
    - envs:
        yq_version: "3.4.1"
        ntpd_config: |
          # Pools for Gentoo users
          server 0.gentoo.pool.ntp.org
          server 1.gentoo.pool.ntp.org
          server 2.gentoo.pool.ntp.org
          server 3.gentoo.pool.ntp.org
          restrict default nomodify nopeer noquery limited kod
          restrict 127.0.0.1
          restrict [::1]

  # In this case we have only one group and only
  # one project. Setting hooks here means that
  # are executed by all groups and nodes
  hooks:
    - event: "post-node-creation"
      commands:
        - apt-get update
        - apt-get upgrade -y
        - apt-get install -y htop vim jq ntp wget

    - event: post-node-creation
      commands:
        - sleep 2
        - wget -q -O /usr/bin/yq https://github.com/mikefarah/yq/releases/download/${yq_version}/yq_linux_amd64
        - chmod a+x /usr/bin/yq

  groups:
    - name: "my-first-group"
      description: "My first group"

      # Set the remote to use for the group.
      # In this case we use the remote mylxd-instance
      # created in the previous steps.
      connection: "mylxd-instance"

      # We use the ephemeral container for this tutorial.
      ephemeral: true

      common_profiles:
        - net-mottainai0
        - sdpool

      nodes:
        - name: "node1"
          image_source: "ubuntu/20.10"
          image_remote_server: "images"

          entrypoint:
            - "/bin/bash"
            - "-c"

          hooks:
            - event: "post-node-sync"
              flags:
                - upgrade
              commands:
                - apt-get update && apt-get upgrade -y

            - event: "post-node-sync"
              flags:
                - update_ntpd_config
              commands:
                - |
                  echo "${ntpd_config}" > /etc/ntp.conf
                - cat /etc/ntp.conf
                - systemctl restart ntp

```

Deploy your NTP Service project:

```shell
# Create the container
$> lxd-compose apply my-first-project
Apply project my-first-project
Searching image: ubuntu/20.10
For image ubuntu/20.10 found fingerprint 4aeac41e7f72b587409493fb814522fcf0925ba9c0c7250ab15fb36867c43d85
>>> Creating container node1... - 🏭 
>>> [node1] - [started] 💣 
>>> [node1] - apt-get update - ☕ 
Hit:1 http://archive.ubuntu.com/ubuntu groovy InRelease
Get:2 http://security.ubuntu.com/ubuntu groovy-security InRelease [110 kB]
Get:3 http://archive.ubuntu.com/ubuntu groovy-updates InRelease [115 kB]
Get:4 http://security.ubuntu.com/ubuntu groovy-security/main amd64 Packages [221 kB]
Get:5 http://security.ubuntu.com/ubuntu groovy-security/universe amd64 Packages [57.6 kB]
Get:6 http://archive.ubuntu.com/ubuntu groovy-updates/main amd64 Packages [366 kB]
Get:7 http://archive.ubuntu.com/ubuntu groovy-updates/main Translation-en [94.1 kB]
Get:8 http://archive.ubuntu.com/ubuntu groovy-updates/universe amd64 Packages [138 kB]
Get:9 http://archive.ubuntu.com/ubuntu groovy-updates/universe Translation-en [53.2 kB]
Fetched 1156 kB in 1s (1515 kB/s)
Reading package lists...
>>> [node1] - apt-get upgrade -y - ☕ 
Reading package lists...
Building dependency tree...
Reading state information...
Calculating upgrade...
0 upgraded, 0 newly installed, 0 to remove and 0 not upgraded.
>>> [node1] - apt-get install -y htop vim jq ntp wget - ☕ 
Reading package lists...
Building dependency tree...
Reading state information...
...
>>> [node1] - sleep 2 - ☕ 
>>> [node1] - wget -q -O /usr/bin/yq https://github.com/mikefarah/yq/releases/download/${yq_version}/yq_linux_amd64 - ☕ 
>>> [node1] - chmod a+x /usr/bin/yq - ☕ 
>>> [node1] - apt-get update && apt-get upgrade -y - ☕ 
Hit:1 http://security.ubuntu.com/ubuntu groovy-security InRelease
Hit:2 http://archive.ubuntu.com/ubuntu groovy InRelease
Hit:3 http://archive.ubuntu.com/ubuntu groovy-updates InRelease
Reading package lists...
Reading package lists...
Building dependency tree...
Reading state information...
Calculating upgrade...
0 upgraded, 0 newly installed, 0 to remove and 0 not upgraded.
>>> [node1] - echo "${ntpd_config}" > /etc/ntp.conf
 - ☕ 
>>> [node1] - cat /etc/ntp.conf - ☕ 
# Pools for Gentoo users
server 0.gentoo.pool.ntp.org
server 1.gentoo.pool.ntp.org
server 2.gentoo.pool.ntp.org
server 3.gentoo.pool.ntp.org
restrict default nomodify nopeer noquery limited kod
restrict 127.0.0.1
restrict [::1]

>>> [node1] - systemctl restart ntp - ☕ 
All done.
```

If the node1 is already present `lxd-compose` skips *post-node-creation* hooks:
```shell
$> lxd-compose apply my-first-project 
Apply project my-first-project
>>> [node1] - apt-get update && apt-get upgrade -y - ☕ 
Hit:1 http://archive.ubuntu.com/ubuntu groovy InRelease
Hit:2 http://archive.ubuntu.com/ubuntu groovy-updates InRelease
Hit:3 http://security.ubuntu.com/ubuntu groovy-security InRelease
Reading package lists...
Reading package lists...
Building dependency tree...
Reading state information...
Calculating upgrade...
0 upgraded, 0 newly installed, 0 to remove and 0 not upgraded.
>>> [node1] - echo "${ntpd_config}" > /etc/ntp.conf
 - ☕ 
>>> [node1] - cat /etc/ntp.conf - ☕ 
# Pools for Gentoo users
server 0.gentoo.pool.ntp.org
server 1.gentoo.pool.ntp.org
server 2.gentoo.pool.ntp.org
server 3.gentoo.pool.ntp.org
restrict default nomodify nopeer noquery limited kod
restrict 127.0.0.1
restrict [::1]

>>> [node1] - systemctl restart ntp - ☕ 
All done.
```

And you can skip hooks:
```shell
$> lxd-compose apply my-first-project --disable-flag upgrade
Apply project my-first-project
>>> [node1] - echo "${ntpd_config}" > /etc/ntp.conf
 - ☕ 
>>> [node1] - cat /etc/ntp.conf - ☕ 
# Pools for Gentoo users
server 0.gentoo.pool.ntp.org
server 1.gentoo.pool.ntp.org
server 2.gentoo.pool.ntp.org
server 3.gentoo.pool.ntp.org
restrict default nomodify nopeer noquery limited kod
restrict 127.0.0.1
restrict [::1]

>>> [node1] - systemctl restart ntp - ☕ 
All done.
```

#### Destroy the project

```shell
$> lxd-compose destroy my-first-project
>>> [node1] - [stopped] ✔ 
All done.
```

{{< hint info >}}
Now you are ready to create your project! If it could be something that help
people you can share it with the community
through a PR at [LXD Compose Galaxy](https://github.com/MottainaiCI/lxd-compose-galaxy/).
{{< /hint >}}


