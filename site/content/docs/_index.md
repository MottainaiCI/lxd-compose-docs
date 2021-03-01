---
title: "LXD Compose"
type: docs
---

# Why LXD Compose?

While the world trying to move every service over to the cloud at 
the PAAS level and there are a lot of scenarios that vary in relation 
to the old school with VMs (Vmware, Openstack, QEMU, etc.).

In these scenarios, the use of LXD technology allows for the maintenance of multi-services
for containers and applications that are OS or distro specific. (for example: requiring or avoiding SystemD in their scope).

The use of **lxd-compose** with the tool [simplestreams-builder](https://github.com/MottainaiCI/simplestreams-builder) for the building of LXD images ensures a method to cleanly reproduce preparing
the container's golden images for a production environment.
No more worries about rolling updates for the OS used.

In the same way, embedded have a way to deploy services in a
reproducible manner. For example, to use Banana PI or Raspberry PI 
at home for Home entertainment, home automation, etc. How many
times did you lose your SD card with the configurations?

It's here that **lxd-compose** wants to help people on tracing their configuration
and the workflow to follow on setup their infrastructure/services. In particular,
also to share this workflow with the community.

In the past, I tried to resolve these issues through Ansible with
the project [FreeRadius Tasks](https://github.com/geaaru/freeradius-tasks) but it's
too verbose and few dynamic, it requires dependencies and a lot of RAM.

So, in summary, these are the core targets of the **lxd-compose**:

{{< columns >}}

## Setup LXD instances

LXD has so many features and options that are often configured manually after you
have installed the instance. These configurations could be applied at the container
level (which in this case are lost when you destroy the container) or at the profile level.
Personally, I suggest using profiles for this job and indeed **lxd-compose** tries
to share a way to register these profiles used on your configuration service and
create these profiles from the specs. The same applies for network devices.

<--->

## Automation

There are a lot of configurations and/or operations to do in a VM or in a container
that are repetitive.
These steps could be written in a more readable way thanks to the YAML syntax anchors or
through [Helm engine](https://helm.sh/docs/chart_template_guide/).

<--->

## Tracing installation steps

How often do you write an Installation Guide of the software written for your clients?
To ensure that these steps are always valid and correct in a world where the software
changes every minute it's a bit hard. To write a specification that could be to test
in a CD/CI pipeline reduce the effort of your team.

{{< /columns >}}

{{< columns >}}

## Update configurations

**lxd-compose** automatically checks if the container of the project is already
present and applies only the hooks related to the configuration. This permits you
to update the configuration easily without destroying and creating again the container.

<--->

## Infrastructure less

**lxd-compose** doesn't require an external service or database to deploy your
services. For example like `etcd` for `k8s`. You need only a node were to run
lxd-compose that reaches one or more LXD instances.

<--->

## Developers's friend

Often it's not easy for a developer to test code when other developers
are, at the same time, working on modules of the same infrastructure.
**lxd-compose** works to supply a method to quickly, reliably, and repetitively
deploy an infrastructure for testing and for syncing code directly in
the container used by the developer for testing code.

{{< /columns >}}


