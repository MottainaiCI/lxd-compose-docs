---
title: Nodes
type: docs
---

# Nodes

The `node` identify the container created to a specific LXD instance defined by
the `connection` attribute available at group level.

{{< hint warning >}}
The name of the node must be unique inside all loaded environments.
In the real the limitation is mandatory only for the same LXD instance, but
`lxd-compose validate` at the moment doesn't permit to use the same name
between different groups or projects.
{{< /hint >}}

In particular, inside the node you could define:

  * `name`: the name of the container used inside the LXD container instance

  * `name_prefix`: an optional attribute to define the prefix to use on the
    name of the node. Normally, this is used at runtime to run test unites
    and it's set by the CLI.

  * `image_source`: the name of the image to use for create the container.

  * `image_remote_server`: an optional attribute to define the name of the
    remote where to search the used source image. By default it's used
    Canonical `images` server if *P2P mode* is disabled.

  * `labels`: an optional attribute to define key values field of the specific
    node that are then set as environment variables of the hooks executed for
    the node.

  * `source_dir`: an optional attribute to define the source directory
    to use in the join with the configuration files to use with the template
    engine or for the files/directories to sync.

  * `entrypoint`: an optional attribute to define the entrypoint to use
    on run command inside the container. By default it's used `/bin/bash -c`.

  * `hooks`: an optional attribute to define the list of hooks to apply.

  * `config_templates`: an optional attribute to define the list of
    configuration files to generate through the template engine.

  * `sync_resources`: an optional attribute to define the list of
    files or directories to sync from the host where is running *lxd-compose*
    to the container.

An example of a node section:

```yaml
        nodes:
          - &mongors1
            name: mongo-rs1
            image_source: "ubuntu/18.04"
            # By deafult it use remote images"
            image_remote_server: "images"

            entrypoint:
              - "/bin/bash"
              - "-c"

            # Define the list of LXD Profile to use in additional
            # to group profiles for create the containers
            #profiles:
            #  - privileged

            # List of commands executed just after the creation of the
            # container.
            hooks:

              - event: post-node-creation
                commands:
                  # DHCP seems slow
                  - sleep 5
                  - apt-get update
                  - apt-get upgrade -y
                  - apt-get install wget gpg ca-certificates jq -y
                  - |
                    wget -q -O /usr/bin/yq \
                    https://github.com/mikefarah/yq/releases/download/3.4.1/yq_linux_amd64
                  - chmod a+x /usr/bin/yq
                  - apt-get clean

              - event: post-node-creation
                commands:
                  - apt-get install -y $(echo ${packages} | jq '.[]' -r)
                  - apt-get clean

            sync_resources:
            # The path files is under the directory of the
            # environment file.
              - source: files/mongo-setup.sh
                dst: /tmp/mongo-setup.sh

          - <<: *mongors1
            name: mongo-rs2

          - <<: *mongors1
            name: mongo-rs3
```

{{< hint infi >}}
On setup multiple nodes of the same group with the same configuration options
you can to use the YAML ancors and to reduce the specification.
{{< /hint >}}
