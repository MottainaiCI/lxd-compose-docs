---
title: Groups
type: docs
---

# Groups

The `group` is used to define for example a list of nodes that supply a particular service. There isn't a real limitation on this but only common/best practices. **lxd-compose** hasn't limit to use multiple groups with only one node instead of a single group with multiple nodes.

{{< hint warning >}}
The name of the group must be unique inside all loaded environments. This permits
to use runtime options to disable/enable groups on deploy a project.
{{< /hint >}}

In particular, inside the group you could define:

  * `name`: an unique identifier of the group. I suggest to avoid spaces.

  * `description`: an user friendly description of the group mission.

  * `connection`: the LXD remote/LXD instance to use on manage the LXD containers
    of the group.

  * `common_profiles`: an optional attribute to define the list of the LXD
    profiles to assign at the LXD containers of the group.

  * `ephemeral`: define if the containers must be created as ephemeral or not.

  * `nodes`: contains the list of the nodes of the group

  * `node_prefix`: an optional attribute to define the prefix to use on
    the name of the nodes. Normally, this is used at runtime to run test units
    and it's set by the CLI.

  * `hooks`: an optional attribute to define the list of hooks to apply.

  * `config_templates`: an optional attribute to define the list of
    configuration files to generate through the template engine.


***

To show the list of groups available in a project:

```shell
$> # command execute on lxd-compose-galaxy repository
$> lxd-compose group list mottainai-server-services
-  mottainai-database
-  mottainai-broker
-  mottainai-server
```

***

An example of a group section:

```yaml
    groups:
      - name: "proxy1"
        description: "Nginx Proxy"

        connection: "local"

        # Define the list of LXD Profile to use
        # for create the containers
        common_profiles:
          - default
          - net-local

        # Create the environment container as ephemeral or not.
        ephemeral: false

        nodes:
# ...
```
