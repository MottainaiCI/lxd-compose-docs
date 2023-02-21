---
bookCollapseSection: true
title: "Packaging Projects"
type: docs
---

# Packaging Projects

In order to organize your projects for a Production environment
it's possible that the team that executes the installation is not
the same one that has prepared the specs. Again, the production
environment is for security reasons without a direct connection
with the SCM repositories where are available the templates to
use for the configuration file generation.

Another example is that the templates of the configuration
the file of the modules implemented is strictly related to a
specific version and the lxd-compose specs must follow this
relationship for use.

It's here that the new command `pack` and `unpack` try to help
the workflow operative that permits to have a way to share
independent the lxd-compose specifications to a Production
Engineer Team and follow the installation and upgrade process
when the separation of duties doesn't permit to have access directly
to source repositories.

In particular, the `pack` command permits to the creation of a
tarball of the lxd-compose specs and all the files and templates
used for the system setup. It considers some best practices to
follow that simplify the paths remapping of the sources used.


Hereinafter, an easy example that describes a possible use case.

## Example

In the example we consider to have two different git repositories
for the `lxd-compose` specs and the developed module where it's also
available the template of his configuration file:

In the example, we consider having two different git repositories
for the `lxd-compose` specs and the developed module where it's also
available the template of his configuration file.

* `my-lxd-compose`: the repository of the LXD Compose specs.

* `my-module`: the repository of the developed module.

The two repositories are organized in this way:

```

$# tree
.
├── lxd-compose
│   ├── envs
│   │   ├── common
│   │   │   ├── hooks
│   │   │   │   ├── hosts.yml
│   │   │   │   ├── luet-packages.yml
│   │   │   │   ├── luet-repositories.yml
│   │   │   │   ├── node-exporter-systemd.yml
│   │   │   │   ├── node-exporter-sysvinit.yml
│   │   │   │   ├── systemd-dns.yml
│   │   │   │   ├── systemd-net-static.yml
│   │   │   │   ├── systemd-net.yml
│   │   │   │   ├── ubuntu-setup.yml
│   │   │   │   └── yum-setup.yml
│   │   │   ├── networks
│   │   │   │   ├── mottainai0.yml
│   │   │   │   └── ovs0.yml
│   │   │   ├── profiles
│   │   │   │   ├── default.yml
│   │   │   │   ├── docker-xfs-fs.yml
│   │   │   │   ├── docker.yml
│   │   │   │   ├── flavor-big.yml
│   │   │   │   ├── flavor-medium.yml
│   │   │   │   ├── flavor-thin.yml
│   │   │   │   ├── logs-disk.yml
│   │   │   │   ├── loop.yml
│   │   │   │   ├── lxd-socket-proxy.yml
│   │   │   │   ├── lxd-socket.yml
│   │   │   │   ├── lxd-vm.yml
│   │   │   │   ├── net-mottainai0.yml
│   │   │   │   ├── net-phy-mgmt.yml
│   │   │   │   ├── net-phy-srv.yml
│   │   │   │   ├── privileged.yml
│   │   │   │   └── zfs.yml
│   │   │   └── storages
│   │   │       ├── btrfs-loopback.yml
│   │   │       ├── btrfs-source.yml
│   │   │       ├── ceph.yml
│   │   │       ├── dir-source.yml
│   │   │       ├── lvm-loopback.yml
│   │   │       ├── lvm-source.yml
│   │   │       ├── zfs-loopback.yml
│   │   │       └── zfs-source.yml
│   │   └── myproject
│   │       ├── my.yml
│   │       └── vars
│   │           └── common.yaml
│   ├── lxd-conf
│   │   ├── client.crt
│   │   ├── client.key
│   │   ├── config.yml
│   │   └── servercerts
│   │       ├── 127.0.0.1.crt
│   └── render
│       └── default.yaml
└── my-module
    ├── conf.tmpl
    └── mymodule.sh

13 directories, 45 files

```

This the `lxd-compose` config file `lxd-compose/.lxd-compose.yml`:

```yaml

general:
    lxd_confdir: ./lxd-conf
logging:
    enable_logfile: false
    level: info
    enable_emoji: true
    color: true
    runtime_cmds_output: true
    cmds_output: true
env_dirs:
    - envs/myproject/
render_default_file: render/default.yaml

```

Between the different best practices a good choice when the
templates are available in different repositories is to use
a *render environment* as a prefix path. The content of this
variable will be used for the pack renaming later.

This the content of the file `render/default.yaml`:

```yaml
#-------------------------------------------#
# General params
#-------------------------------------------#
connection: "local"
ephemeral: true
default_ubuntu_image: "ubuntu/22.04"
default_ubuntu_lts_image: "ubuntu/18.04"
default_internal_domain: "mottainai.local"

source_base_dir: "../../.."
```

The **source_base_dir** is the render env used in the project
for the compilation of the module file.

Obviously, it's better to use the same tree level for all
projects and environments to improve readability.

So, if we have a very simple module/bashing script
that just has a source file like this:

```bash
$> cat my-module/conf.tmpl
#!/bin/bash

export msg="{{ .message }}"
```

That is imported in the main script file:

```bash
$ cat my-module/mymodule.sh
#!/bin/bash

# The conf.sh file is generated by lxd-compose from conf.tmpl.
source /etc/conf.sh

for ((i=0; i<5;i++)); do
  echo $msg
done
```

The **message** variable is defined in the var file:

```bash
$> cat lxd-compose/envs/myproject/vars/common.yaml 
envs:
  message: "W LXD Compose"
```

In particular, the file *conf.tmpl* is the file compiled by
`lxd-compose` to generate the *conf.sh* file imported by the
script *mymodule.sh*. The both files are then synced in the
container like described in this environment specs:

```yaml
$ cat lxd-compose/envs/myproject/my.yml
# Author: Daniele Rondina, geaaru@funtoo.org

version: "1"

template_engine:
  engine: "mottainai"

include_profiles_files:
- ../common/profiles/net-mottainai0.yml
- ../common/profiles/default.yml
- ../common/profiles/flavor-medium.yml
- ../common/profiles/loop.yml
- ../common/profiles/docker.yml
- ../common/profiles/privileged.yml

include_networks_files:
- ../common/networks/mottainai0.yml
- ../common/networks/ovs0.yml

include_storage_files:
- ../common/storages/dir-source.yml
- ../common/storages/btrfs-source.yml

pack_extra:
  files:
    - {{ .Values.source_base_dir }}/my-module/mymodule.sh
  rename:
    # The packth is related to the lxd-compose directory and the
    # environment file basedir.
    - source: ../my-module/mymodule.sh
      dest: sources/my-module/mymodule.sh

projects:

  - name: "myproject"
    description: |
      Testing project for pack command.

    include_env_files:
      - vars/common.yaml

    vars:
      - envs:
          LUET_NOLOCK: "true"
          LUET_YES: "true"

          luet_packages:
            - net-tools
            - bash

    groups:
      - name: "my-module-service"
        description: "Start container for running my-module."
        include_hooks_files:
          - ../common/hooks/luet-packages.yml

        common_profiles:
          - default
          - net-mottainai0

        # Create the environment container as ephemeral or not.
        ephemeral: true
        connection: "{{ .Values.connection }}"

        nodes:
          - name: mymodule1
            image_source: "macaroni/terragon-dumplings"
            image_remote_server: "macaroni"

            hooks:
              - event: post-node-sync
                commands:
                  # Running mymodule
                  - bash /mymodule.sh

            config_templates:
              - source: {{ .Values.source_base_dir }}/my-module/conf.tmpl
                dst: /tmp/lxd-compose/mymodule1/etc/conf.sh

            sync_resources:
              - source: {{ .Values.source_base_dir }}/my-module/mymodule.sh
                dst: /
              - source: /tmp/lxd-compose/mymodule1/etc/conf.sh
                dst: /etc/

```

As visible in the specs the *source_base_dir* render variable is used
as a prefix path in the `config_templates` and `sync_resources` sections.

{{< hint warning >}}
The `pack` command just includes automatically all files defined in the
*config_templates*, profiles, groups files, variable files, networks,
and commands but NOT the additional files added in the *sync_resources*
section.
This is because often it contains files generated from templates.
If there is a sync of an extra file this file must be defined in the
`pack_extra` section to be included in the tarball.
This means that when there is a static file to inject in the tarball
this path must be renamed to `sources` directory created automatically.
If this files are stored under the lxd-compose project the rename is
not neeeded.
{{< /hint >}}

When the project is ready for Production we can generate the tarball
with this command:

```bash
$> lxd-compose pack --source-common-path "../../.." --to /tmp/lxd-compose-myproject.tar.gz myproject
🏭 Processing project myproject with env file envs/myproject/my.yml.
Template ../my-module/conf.tmpl -> sources/my-module/conf.tmpl
Tarball /tmp/lxd-compose-myproject.tar.gz generated.
The source dir to use is: ../../sources
```

The option `--source-common-path` normally is set with the same
value defined in the render env for the variable *source_base_dir*.

The tarball `/tmp/lxd-compose-project.tar.gz` is ready for the installation!

So, in a fresh filesystem it's possible *unpack* the tarball with this
command:

```bash
$> lxd-compose unpack --render-file render/default.yaml --render-env "source_base_dir=../../sources" /tmp/lxd-compose-myproject.tar.gz
Render file render/default.yaml updated correctly.
Operation completed.

```

That for the example generates this tree:

```
<lxd-compose-demo>#  tree
.
├── envs
│   ├── common
│   │   ├── hooks
│   │   │   └── luet-packages.yml
│   │   ├── networks
│   │   │   ├── mottainai0.yml
│   │   │   └── ovs0.yml
│   │   ├── profiles
│   │   │   ├── default.yml
│   │   │   ├── docker.yml
│   │   │   ├── flavor-medium.yml
│   │   │   ├── loop.yml
│   │   │   ├── net-mottainai0.yml
│   │   │   └── privileged.yml
│   │   └── storages
│   │       ├── btrfs-source.yml
│   │       └── dir-source.yml
│   └── myproject
│       ├── my.yml
│       └── vars
│           └── common.yaml
├── lxd-conf
│   ├── client.crt
│   ├── client.key
│   ├── config.yml
│   └── servercerts
│       └── 127.0.0.1.crt
├── render
│   └── default.yaml
└── sources
    └── my-module
        ├── conf.tmpl
        └── mymodule.sh

13 directories, 20 files

```

As is visible, in the tarball files are automatically injected all files
used by the selected project like for example the hook file
*envs/common/luet-packages.yml* but not the file *envs/common/luet-repositories.yml*.

At the same time, the *pack* command injects the `lxd-conf` directory if it's been
defined in the lxd-compose config file.

After, the *unpack* the tree is ready for the deploy:

```bash

$> lxd-compose a myproject --env sleep=3
Apply project myproject
Searching image: macaroni/terragon-dumplings
For image macaroni/terragon-dumplings found fingerprint 109aec564696c4757cf0036eca5311c6f447dbf9c2ef4afc2e8db43f7c8fe98b
>>> Creating container mymodule1... - 🏭 
>>> [mymodule1] - [started] 💣 
>>> [mymodule1] - [ -n "${sleep}" ] && sleep ${sleep} ; if [ -e /etc/os-release ] ; then ubuntu=$(cat /etc/os-release | grep ID| grep ubuntu | wc -l) ; else ubuntu="0" ; fi && luet repo update && luet i -y utils/jq utils/yq && luet i -y $(echo ${luet_packages} | jq '.[]' -r) && luet cleanup --purge-repos ; - ☕ 
🏠 Repository:              geaaru-repo-index Revision:   3 - 2023-02-07 14:36:23 +0000 UTC
🏠 Repository:               mottainai-stable Revision:  67 - 2023-02-20 18:13:58 +0000 UTC
🏠 Repository:               macaroni-commons Revision: 117 - 2023-01-08 09:28:23 +0000 UTC
🏠 Repository:              macaroni-terragon Revision: 143 - 2023-02-07 09:11:27 +0000 UTC

...

Resolve finalizers...
🚀 Luet 0.33.0-geaaru-g4e8db62fb8d2b25df7652f5001353fcde8893197 2023-02-19 07:42:11 UTC - go1.20.1
🏠 Repository:              geaaru-repo-index Revision:   3 - 2023-02-07 14:36:23 +0000 UTC
🏠 Repository:               macaroni-commons Revision: 117 - 2023-01-08 09:28:23 +0000 UTC
🏠 Repository:              macaroni-terragon Revision: 143 - 2023-02-07 09:11:27 +0000 UTC
🏠 Repository:               mottainai-stable Revision:  67 - 2023-02-20 18:13:58 +0000 UTC
🚧  warning sys-apps/net-tools already installed.
🚧  warning app-shells/bash already installed.
🚧  warning No packages to install.
Cleaned:  17 packages.
Repos Cleaned:  4
>>> [mymodule1] Compile 1 resources... 🍦
>>> [mymodule1] - [ 1/ 1] /tmp/lxd-compose/mymodule1/etc/conf.sh ✔
>>> [mymodule1] Syncing 2 resources... - 🚌
>>> [mymodule1] - [ 1/ 2] / - ✔
>>> [mymodule1] - [ 2/ 2] /etc/ - ✔
>>> [mymodule1] - bash /mymodule.sh - ☕
W LXD Compose
W LXD Compose
W LXD Compose
W LXD Compose
W LXD Compose
All done.
```

