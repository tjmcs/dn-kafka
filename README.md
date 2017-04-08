# dn-kafka
Playbooks/Roles used to deploy Kakfa; supports deployments of either the [Confluent](https://www.confluent.io/) or [Apache](https://kafka.apache.org/) Kafka distributions.

# Installation
To install kafka using the [site.yml](site.yml) playbook in this repository, first clone the contents of this repository to a local directory using a command like the following:

```bash
$ git clone --recursive https://github.com/Datanexus/dn-kafka
```

That command will pull down the repository and it's dependencies. Currently this playbook's only dependencies are on the [common-roles](https://github.com/Datanexus/common-roles) and [common-utils](https://github.com/Datanexus/common-utils) submodules in this repository. The first provides a set of common roles that are reused across the DataNexus playbooks, while the second provides a similar set of common utilities, including a pair of dynamic inventory scripts that can be used to control deployments made using this playbook in AWS and OpenStack environments.

The only other requirements for using the playbook in this repository are a relatively recent (v2.x) release of Ansible. The easiest way to obtain a recent relese if Ansible is via a `pip install`, which requires that Python and pip are both installed locally. We have performed all of our testing using a recent (2.7.x) version of Python (Python 2); your mileage may vary if you attempt to run the playbook or the attached dynamic inventory scripts under a newer (v3.x) release of Python (Python 3).

# Using this role to deploy Kafka
The [site.yml](site.yml) file at the top-level of this repository supports both single-node Kafka deployments and the deployment of multi-node Kafka clusters. The process of deploying Kafka to these nodes will vary, depending on whether you are managing your inventory dynamically or statically (more on this topic [here](docs/Dynamic-vs-Static-Inventory.md)), whether you are performing a single-node deployment or are deploying a Kafka cluster, and where you are downloading the packages and dependencies from that are needed to run Kafka on those nodes.

We discuss the various deployment scenarios supported by this playbook in [this document](docs/Deployment-Scenarios.md) and discuss how the [Vagrantfile](Vagrantfile) in this repository can be used to deploy Kafka (both single-node deployments and multi-node clusters are supported) to a set of VMs hosted locally in VirtualBox [here](docs/Deployment-via-Vagrant.md).

# Assumptions
It is assumed that this playbook will be run on a recent (systemd-based) version of RHEL or CentOS (RHEL-7.x or CentOS-7.x, for example); no support is provided for other distributions (and the `site.xml` playbook will not run successfully). Furthermore, it is assumed that you are interested in deploying a relatively recent version of Kafka using this playbook (the current default for Confluent is v3.2 but any v3.x release should do, while the current release for Apache is the v0.10.1.0 release that was built using Scala v2.11).

It should also be noted that in order to execute the vagrant commands shown in [this document](docs/Deployment-via-Vagrant.md) locally, recent versions of [Vagrant](https://www.vagrantup.com/) and [VirtualBox](https://www.virtualbox.org) will have to be installed locally. While Vagrant does support management of Virtual Machines deployed via VMware Workstation and/or Fusion with the right (commercial) drivers in place, we have only tested the [Vagrantfile](Vagrantfile) in this repository under VirtualBox using recent (v1.9.x) releases of Vagrant.

