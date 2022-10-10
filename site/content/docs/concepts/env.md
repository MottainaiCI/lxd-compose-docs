---
title: Environements
type: docs
---

# Environments

The *environments* are characterized of from these properties or elements:

  * `version`: describe the version of the specifications. At the moment
    the only version supported is "1".

  * `template_engine`: under the *template_engine* section is configured
    the engine used for create project files. The available engines are
    `jinja2` (it uses `j2cli` tool and jinja2 framework like for Ansible)
    and `mottainai` that it uses Golang's template render.
    For the `jinja2` template is possible set additional options for the
    `j2cli` through `opts` field.

  * `profiles`: the list of LXD profiles used by the environment
    that could be added and/or updated on the LXD instances used
    by the projects.

  * `networks`: the list of LXD networks used by the environment
    that could be added and/or updated on the LXD instances used
    by the projects.

  * `commands`: the list of commands defined related to the projects
    of the environment. It's helpful to have a register of the more useful
    commands to run on a running system for backup, validation, etc.
    The commands are aliases of `apply` command where it's possible
    define flags, additional hooks, etc.

  * `storages`: the list of LXD storages used by the environment
    that could be added and/or updated on the LXD instances used
    by the projects.

  * `projects`: the projects to deploy.

**lxd-compose** read all files under the directories defined on the paramater `env_dirs`
of the configuration file and load the specification of all projects in memory before
run commands.

The `profiles`, `networks`, `storages` and `commands` are all loaded and rendered
through the LXD Compose render engine that permit to customize the entities without
create multiple time the same resource but with few differences.

Hereinafter, an extract of the configuration file available on [LXD Compose Galaxy](https://github.com/MottainaiCI/lxd-compose-galaxy/blob/master/.lxd-compose):

```yaml
general:
  debug: false
#  lxd_confdir: ./lxd-conf
  push_progressbar: false
logging:
  level: "info"
# Define the directories list where load
# environments.
env_dirs:
- ./envs/nginx
- ./envs/mottainai-server
```

where is been defined the directories where **lxd-compose** the files
`envs/{nginx,mottainai-server}/*.yml` or `.yaml` files.

The environment's files could be a pure YAML file or template for
the [Helm engine](https://helm.sh/docs/chart_template_guide/);
in this case, you need to define the render values file from CLI or
from the configuration file.

For example you can define the source image used by a node inside a group
in this way:

```
nodes:
  - name: "node1"
    image_source: "alpine/{{ .Values.alpine_version }}"
    image_remote_server: "images"
```

and to test your services with all available version of *alpine* images on define different render files
like this:

```yaml
# file: alpine3_11.yml
alpine_version: "3.11"
```

```yaml
# file: alpine3_10.yml
alpine_version: "3.10"
```

At this point you can run your project in this way:

```shell
$> lxd-compose apply myproject --render-values alpine3_11.yml
$> lxd-compose destroy myproject
$> lxd-compose apply myproject --render-values alpine3_10.yml
```

In alternative you can set the default render file inside the config:

```yaml
# file: .lxd-compose.yml
render_default_file: alpine3_11.yml
```
and then override the value only when it's needed:

```shell
$> lxd-compose apply myproject
$> lxd-compose destroy myproject
$> lxd-compose apply myproject --render-values alpine3_10.yml
```

{{< hint info >}}
In general, the *render engine* is used to generate the environment's files at runtime,
instead the *template engine* defined inside the environment is used as template engine
for the files to use inside the deploy workflow.
{{< /hint >}}

{{< hint warning >}}
It’s a good practice avoid to use group names equal across different projects or nodes
with equals names because inside the project it’s possible to define a hook to execute
a command to an external node of the project. The lxd-compose validate command blocks
duplicate at the moment.
{{< /hint >}}


#### Profiles


Inside the environment's files could be defined the LXD profiles:

```yaml
# Define the list of LXD Profiles used by all projects.
# This profiles are not mandatory. An user could create
# his profiles without to use this list.
profiles:
- name: "mottainai-https"
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

This is section is used only for tracing the profiles needed by the infrastructure.
It is possible create and/or update profiles through the `lxd-compose profile` subcommand.

The definition of the profiles could be inline over the environment YAML or with external files
through the `include_profiles_files` attribute.

#### Networks

In a similar way, inside an environment file it's possible define the list of
network device or bridge used by the LXD instances.

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

To show all possible configurations for both `networks` and `profiles` there is the
[LXD documentation](https://lxd.readthedocs.io/en/latest/configuration/). **lxd-compose** maps
the API configurations directly.

Some examples are available on [LXD Compose Galaxy](https://github.com/MottainaiCI/lxd-compose-galaxy/tree/master/envs/common/networks).

The definition of the networks could be inline over the environment YAML or with external files
through the `include_networks_files` attribute.

#### Commands

The commands have different missions:

1. permit to define and register maintenance tasks and/or particolar hooks to run over
   existing container of already deployed projects. An example could the task that update
   the lencrypt certificate of existing HTTP service.

2. permit to deploy a specific project with customization (different vars files, flags, etc.).
   For example, the task to build LXD images over LXD Compose Galaxy is a single project that
   supply different commands as shortcuts for build the different LXD images.

Inside the environment file the commands could be defined inline:

```yaml
commands:
  - name: mottainai-proxy-update-cerbot
    description: |
      Update letencrypt certificate
      on mottainai Proxy.

      NOTE: the container must be already created.

    project: mottainai-server-services
    apply_alias: true
    enable_groups:
      - mottainai-proxy1
    enable_flags:
      - certbot_standalone
```

or through includes:

```yaml
include_commands_files:
- commands/certbot.yml
- commands/backup-certbot.yml
```

Obviously, using `include_commands_files` permit to reuse the same command over multiple projects.

#### Storages

Inside LXD there are different way to setup the LXD storage: btrfs, zfs, lvm, loopback, etc.

The storage is the main element when an LXD instance is configured. This is the reason why
it's important to trace the configurations option used over a specific remote.

The [LXD Compose Galaxy](https://github.com/MottainaiCI/lxd-compose-galaxy/tree/master/envs/common/storages) has already a good list of possible configuration that could be used by the users in
their projects.

The storage specs could be defined inside the environment YAML inline or as included
files through the `include_storages_files` attribute.


