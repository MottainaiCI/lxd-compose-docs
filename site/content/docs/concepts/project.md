---
title: Projects
type: docs
---

# Projects

The `project` is used to define for example a way to identify the deploy of one or more services (inside different
groups).

{{< hint warning >}}
The name of the project must be unique inside all loaded environments.
{{< /hint >}}

In particular, inside the project you could define:

  * `name`: an unique identifier of the project. I suggest to avoid spaces.

  * `description`: an user friendly description of the project mission.

  * `include_env_files`: an optional attribute that permit to define the list
    of files with environment variables loaded by *lxd-compose*.
    If the path is relative this is based on the directory where is present
    the environment file that contains the project.

  * `include_groups_files`: an optional attribute that permit to define the list
    of files where are defined the groups of the project.
    If the path is relative this is based on the directory where is present
    the environment file that contains the project.

  * `include_hooks_files`: an optional attribute that permit to define the list
    of files where are defined the hooks of the project.
    If the path is relative this is based on the directory where is present
    the environment file that contains the project.

  * `vars`: an optional attribute to define inline project variables.

  * `groups`: an optional attribute to define inline groups of the project.

  * `node_prefix`: an optional attribute to define the prefix to use on
    the name of the nodes. Normally, this is used at runtime to run test units
    and it's set by the CLI.

  * `hooks`: an optional attribute to define the list of hooks to apply.

  * `config_templates`: an optional attribute to define the list of
    configuration files to generate through the template engine.

***

To show the list of all available projects:

```shell
$> # command execute on lxd-compose-galaxy repository
$> lxd-compose project list
- sonarqube-ce
- nginx-proxy
- docker-registry-services
- mongo-replica-set
- mottainai-server-services
- luet-runner-amd64
- arm::ubuntu::mottainai-agent
```

***

An example of a project section:

```yaml
projects:

  - name: "nginx-proxy"
    description: |
      Setup NGINX proxy with custom reverse balancer,
      integrated with letencrypt.

    # Include variables files
    include_env_files:
      - vars/main.yml

    # Include groups files
    include_groups_files:
      - groups/grp1.yml

    # Inline variables
    vars:
      - envs:
          mypublic_domain: example1.com
          letencrypt_server: https://acme-v02.api.letsencrypt.org/directory
          letencrypt_email: myemail@example.com

    # Inline groups
    groups:
      - name: "proxy1"
        description: "Nginx Proxy"
        # In this case it's used local remote.
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
