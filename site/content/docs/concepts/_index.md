---
bookCollapseSection: true
title: "Concepts"
weight: 20
type: docs
---

# Concepts

Inside the **lxd-compose** world you can create different *environments*
(or *infrastructures*) that are described by different YAML file read by
**lxd-compose** on the directories defined on the parameter `env_dirs`
of the configuration file.

An *environment* contains one or more *project*. A *project* describe
a target service or a subset of the services supplied from an infrastructure.
For example, a *project* could contains all services in the External DMZ
like balancer, proxy, etc.

A *project* contains one or more *groups*. A *group* describes the list of
nodes that supply a service. For example, a *group* could contain all
nodes supply Nginx reverse balancer. A *group* communicate with a single
LXD instance or Cluster defined on `connection` attribute.

A *groups* contains one or more *nodes*. A *node* is an LXD container (or in the
near future also VM because LXD now supports QEMU integration).

A *node* is based on an image (available from official [Canonical](https://us.images.linuxcontainers.org/) service or any LXD Server supply images or yet from a Simplestreams
server build with [simplestreams-builder](https://github.com/MottainaiCI/simplestreams-builder) tool.

A *node* is created with a list of LXD *profiles* that are used to configure
networking, CPU limit, memory limit, host path mounted inside the container, etc.

In the deploy workflow **lxd-compose** generates optionally files to inject
in the container through jinja2 engine (it uses `j2cli` tool) or directly through
Golang's template engine based on the [MottainaiCI](https://github.com/MottainaiCI/mottainai-server) code. These files are then pushed inside the container.

The steps to configure the container are called *hooks* and they are characterized
by shell commands execute inside the container or in the host where is run **lxd-compose**
and where the environment is initialized with all the *variables* of the project.
These *hooks* could be executed in different phases of the deployment and based
on the choice of the flags.


{{< hint info >}}
At the moment, the execution of hooks and the creation of the containers
is sequential but in the next releases will be integrate a
Directed Acyclic Graph (DAG) management of the hooks/tasks for
parallel deploy or complex scenarios.
{{< /hint >}}
