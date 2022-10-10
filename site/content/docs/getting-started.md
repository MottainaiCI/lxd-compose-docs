---
title: "Getting Started"
type: docs
---

# Getting Started

## Prerequisites

`lxd-compose` doesn't require libraries or tools, just an LXD instance to call.

## Get LXD Compose

`lxd-compose` is available as [Funtoo Macaroni OS](https://www.macaroni.funtoo.org/) package and installable in every Linux
distro through [luet](https://luet-lab.github.io/docs/) tool with these steps:

#### Installation

```bash
$> curl https://raw.githubusercontent.com/geaaru/luet/geaaru/contrib/config/get_luet_root.sh | sh # Install luet on your system
$> sudo luet install -y app-emulation/lxd-compose # Install lxd-compose binary
$> sudo luet cleanup
```

#### Upgrade

```bash
$> sudo luet upgrade
```


## Create environment tree

There isn't a specific directory tree required to create an `lxd-compose`
project but hereinafter is explained the best practices to start.

```shell

$> mkdir myproject && cd myproject
$> mkdir -p envs/files  # Create directory for static files (optional)
$> mkdir -p envs/groups # Create directory for containers groups (optional)
$> mkdir -p envs/vars   # Create directory for environment variables (optional)

```

The next step is create the `lxd-compose` configuration file. This file is
using YAML format and is automatically read if it's called `.lxd-compose.yml`.

```shell

$> echo "
general:
  debug: false

logging:
  level: \"info\"
  runtime_cmds_output: true

# Define the directories list where load environments.
env_dirs:
- ./envs
" > .lxd-compose.yml

```

Now, it's time to create the project file `./envs/myproject.yml`:

```yaml
version: "1"

template_engine:
  engine: "mottainai"

projects:

  - name: "myproject1"
    description: "My First Project"

    # A fast way to define environments for template
    vars:
      - envs:
          my_var: "value1"
          obj:
            key: "xxx"
            foo: "baa"

    groups:
      - name: "group1"
        description: "Group 1"

        # Define the LXD Remote to use and where
        # create the environment.
        connection: "local"
        # Define the list of LXD Profile to use
        # for create the containers
        common_profiles:
          - default

        # Create the environment container as ephemeral or not.
        ephemeral: true

        nodes:
          - name: node1
            image_source: "alpine/3.12"

            entrypoint:
              - "/bin/sh"
              - "-c"

            # List of commands executed just after the creation of the
            # container.
            hooks:

              - event: post-node-creation
                commands:
                  - echo "Run container command ${my_var}"

              # Print node json
              - event: post-node-creation
                commands:
                  - apk add curl
                  - curl --no-pregress-meter https://raw.githubusercontent.com/geaaru/luet/geaaru/contrib/config/get_luet_root.sh | sh
                  - luet install utils/jq
                  - echo "${node}" | jq

```

And finally, deploy the project:

```shell
$> lxd-compose apply myproject1
Apply project myproject1
Searching image: alpine/3.12
For image alpine/3.12 found fingerprint 57a2e180cbb8b3daf21a51681232e88d145ebd8955a62ada48f876be86bcd093
Try to download image 57a2e180cbb8b3daf21a51681232e88d145ebd8955a62ada48f876be86bcd093 from remote images...
>>> Creating container node1... - 🏭             
>>> [node1] - [started] 💣                
   🏡  - echo "Run container command ${my_var}"
Run container command value1
>>> [node1] - apk add curl - ☕ 
fetch http://dl-cdn.alpinelinux.org/alpine/v3.12/main/x86_64/APKINDEX.tar.gz
fetch http://dl-cdn.alpinelinux.org/alpine/v3.12/community/x86_64/APKINDEX.tar.gz
(1/4) Installing ca-certificates (20191127-r4)
(2/4) Installing nghttp2-libs (1.41.0-r0)
(3/4) Installing libcurl (7.69.1-r3)
(4/4) Installing curl (7.69.1-r3)
...
```

The container is been created and all hooks are been executed.

```shell
$> lxc list
+--------------------------+---------+----------------------+-----------------------------------------------+-----------------------+-----------+
|           NAME           |  STATE  |         IPV4         |                     IPV6                      |         TYPE          | SNAPSHOTS |
+--------------------------+---------+----------------------+-----------------------------------------------+-----------------------+-----------+
| node1                    | RUNNING | 172.18.10.171 (eth0) | fd42:380a:f674:76f3:216:3eff:fef0:52f8 (eth0) | CONTAINER (EPHEMERAL) | 0         |
+--------------------------+---------+----------------------+-----------------------------------------------+-----------------------+-----------+
```

Congratulations! Your first project has been deployed!
