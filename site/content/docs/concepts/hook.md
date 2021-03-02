---
title: Hooks
type: docs
---

# Hooks

The *hooks* are the operations that could be done to configure the target
services.

Inside the hook you could define:

  * `event`: identify the type of the hook

  * `node`: an optional attribute that define the name of the container
    where to run the commands. There is a special node `host` that is
    used to run the commands on the host where is running *lxd-compose*

  * `commands`: an array of commands executed in the target node

  * `out2var`: an optional attribute to use when the stdout of the last
    command must be saved as a variable.

  * `err2var`: an optional attribute to use when the stderr of the last
    command must be saved as a variable.

  * `entrypoint`: an optional attribute to override the entrypoint
    used to run the commands.

  * `flags`: an optional list of string that permit to assign a special
    label to an hook that could be used to exclude the hook or for filter
    hooks.

{{< hint warning >}}
If a command defined in a hook fails, the chain is interrupted and lxd-compose exiting
with error.
{{< /hint >}}

### Types of events

| Event Type | Description |
| ---------- | ----------- |
| *pre-project* | Hook executed before the deployment phase of the selected project/ |
| *pre-group* | Hook executed before the deployment of a group/ |
| *pre-node-creation*  | Hook executed before the creation of a node |
| *post-node-creation* | Hook executed after the creation of a node |
| *pre-node-sync* | Hook executed before the sync phase to a node. Also if there aren't files to sync. |
| *post-node-sync* | Hook executed after the sync phase to a node. Also, if there aren't files to sync. |
| *post-group* | Hook executed after that all nodes of the group are been creation and/or updated. |
| *post-project* | Hook executed at the end of the deployment phase of the selected project.

The hooks with type *pre-node-creation* and *post-node-creation* are executed only
if the container is not present and it's created by *lxd-compose*.

The hooks of the type *pre-project* or *post-project* can be defined only
at project level.

To reduce the verbosity of the YAML, *lxd-compose* permits to define hooks used by
multiple groups or node at different levels.

The hooks defined to an upper level are merged with the hooks of the bottom level and executed
in the defined order.

| Level | Available Event Types (in the execution order) |
| ----- | --------------------- |
| project | *pre-project*<br> *pre-group*<br> *pre-node-creation*<br> *post-node-creation*<br> *pre-node-sync*<br> *post-node-sync*<br> *post-group*<br> *post-project* |
| group | *pre-group*<br> *pre-node-creation*<br> *post-node-creation*<br> *pre-node-sync*<br> *post-node-sync*<br> *post-group* |
| node | *pre-node-creation*<br> *post-node-creation*<br> *pre-node-sync*<br> *post-node-sync* |

### Hooks defined at project level for all nodes

```yaml
# Hooks at project level
hooks:
  - event: pre-project
    # Special node value to execute commands in the host running lxd-compose.
    # The use of sysctl on host it make sense only for local LXD instances.
    node: "host"
    commands:
      - sysctl -w vm.max_map_count=262144
    flags:
      - host_commands
  - event: post-node-creation
    commands:
      - ACCEPT_LICENSE=\* equo update && equo upgrade
    flags:
      - update_container
```

This hook is executed after the creation of all nodes of all groups of the project.
The `node` attribute is not needed in this case.

The flag **update_container** permits to exclude the hook on deploy phase.

```shell
# This run the hook on all nodes
$> lxd-compose apply myproject
# This exclude the hooks with flag update_container
$> lxd-compose apply myproject --disable-flag update_container
# This execute only the hook with the flag update_container
$> lxd-compose apply myproject --enable-flag update_container
```

If it's used the `--enable-flag` option and the hook doesn't contain flags then it's skipped.


### Hooks defined at group level for all nodes or for specific nodes

In a similar way of the hooks defined at project level the hooks of type `pre-node-creation`,
`post-node-creation`, `pre-node-sync` and `post-node-sync` are applied to all nodes.

A possible use case could be that to use the `post-group` event to run infrastructure tests
in the last group of the project.

```yaml
# List of commands executed just after the creation of the
# container.
hooks:
  - event: post-node-creation
    # Install the python unittest2 package in all nodes.
    commands:
      - ACCEPT_LICENSE=\* equo i dev-python/unittest2

  - event: post-group
    node: "node1"
    commands:
      - rm /tmp/foo.log

  - event: post-group
    node: "node2"
    # The run-tests.sh generate traffic/actions to node1.
    commands:
      - /root/run-tests.sh

  - event: post-group
    node: "node1"
    commands:
      - sleep 5 # to manage transmission delays during test
      - /root/analyze-result.sh
```

The `node` field could be related also to node not defined in the project. In this case lxd-compose uses
the `connection` of the group where the hook is defined.


### Hooks defined at node level

Also in the hooks defined at node level it's possible to execute hooks on node `host`, for example,
an hook that to store the password automatically generated by the single MySQL instance that could be
needed in additional operations and/or for manual tasks.

Hereinafter, an example:

```yaml
hooks:
  - event: post-node-creation
    commands:
      - cat /var/log/mysqld.log | grep temporary | awk '{ print $NF }'
    out2var: "mysql_temporary_pwd"
  - event: post-node-creation
    node: "host"
    commands:
      - node=$(echo ${node} | jq '.name') && echo "${mysql_temporary_pwd}" > ./secrets/${node}.mysql.pwd
```

