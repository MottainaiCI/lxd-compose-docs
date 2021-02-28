---
title: "LXD Compose"
type: docs
---

# Why LXD Compose?

While the world trying to move every service over the cloud at the PAAS level
there are a lot of scenarios that are little or medium and/or yet related to
the old school with VMs (Vmware, Openstack, QEMU, etc.).

In these scenarios, the use of LXD technology permits to maintain of multi-services
for container and application OS-based (for example that uses SystemD for their scope).

The use of **lxd-compose** with the tool [simplestreams-builder](https://github.com/MottainaiCI/simplestreams-builder) for the building of LXD images ensure a way to prepare container's images
for a production environment where setup services in a reproducible way.
No more worries about rolling updates of the OS used.

In the same way, also in the embedded to have a way to deploy services in a
reproducible it's a wonderful thing. I mean for example to use Banana PI or
Raspberry PI at home for Home entertainment, home automation, etc. How many
times did you lose your SD card with the configurations?

It's here that **lxd-compose** wants to help people on tracing their configuration
and the workflow to follow on setup their infrastructure/services. In particular,
also to share this workflow with the community.

In the past, I tried to resolve these issues through Ansible for example with
the project [FreeRadius Tasks](https://github.com/geaaru/freeradius-tasks) but it's
too verbose and few dynamic, it requires dependencies and a lot of RAM.

So, in summary, these are the core targets of the **lxd-compose**:

{{< columns >}}

## Setup LXD instances

LXD has so many features and options that are often configured manually after that you
have installed the instance. These configurations could be applied at the container
level (in this case are lost when you destroy the container) or at the profile level.
Personally, I suggest using profiles for this job and indeed **lxd-compose** tries
to share a way to register these profiles used on your configuration service and
a way to create these profiles from the specs. The same for network devices.

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
present and applies only the hooks related to the configuration. This permits
to update of easily configuration without destroying and create again the container.

<--->

## Infrastructure less

**lxd-compose** doesn't require an external service or database to deploy your
services. For example like `etcd` for `k8s`. You need only a node were to run
lxd-compose that reaches one or more LXD instances.

<--->

## Developers's friend

Often it's not always easy for a developer to test his code when other developers
of the same time working on modules of the same infrastructure.
**lxd-compose** wants to supply a way to deploy in a fast and reproducible
way an infrastructure to use for testing and for syncing his code directly in
the container used by the developer for testing his code.

{{< /columns >}}

