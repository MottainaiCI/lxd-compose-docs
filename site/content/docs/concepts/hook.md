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

  * `commands: an array of commands executed in the target node

  * `out2var`: an optional attribute to use when the stdout of the last
    command must be saved as a variable.

  * `err2var`: an optional attribute to use when the stderr of the last
    command must be saved as a variable.

  * `entrypoint`: an optional attribute to override the entrypoint
    used to run the commands.

  * `flags`: an optional list of string that permit to assign a special
    label to an hook that could be used to exclude the hook or for filter
    hooks.


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


