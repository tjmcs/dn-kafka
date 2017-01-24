# dn-kafka
Playbooks/Roles used to deploy Kakfa

# Installation
To install kafka using the `site.yml` playbook in this repository, first clone the contents of this repository to a local directory using a command like the following:
```bash
$ git clone --recursive https://github.com/Datanexus/dn-kafka
```
That command will pull down the repository and it's submodules (currently the only dependency embedded as a submodule is the dependency on the `https://github.com/Datanexus/common-roles` repository).

# Using this role to deploy Kafka
The `site.yml` file at the top-level of this repository pulls in a set of default values for the parameters that are needed to deploy the Confluent distribution of Kafka from the `vars/kafka.yml` file.  The contents of that file currently look like this:

```yaml
# (c) 2016 DataNexus Inc.  All Rights Reserved
#
# Defaults that are necessary for all deployments of
# kafka
---
application: kafka
# the distribution of Kafka that should be installed (apache or confluent)
kafka_distro: confluent
# the interface Kafka should listen on when running
kafka_iface: eth0
# the following parameters are only used when provisioning an instance
# of the apache distribution, but are uncommented here (regardless) to
# provide reasonable default values when provisioning via Vagrant (where
# the distribution being provisioned may be different from the default)
scala_version: "2.11"
kafka_version: "0.10.1.0"
kafka_url: "https://www-us.apache.org/dist/kafka/{{kafka_version}}/kafka_{{scala_version}}-{{kafka_version}}.tgz"
kafka_dir: "/opt/kafka"
# this value is only used when installing the confluent distribution,
# but is uncommented here so that it can be used if a confluent distribution
# is chosen when provisioning via Vagrant
confluent_version: "3.1"
# these parameters are used for both confluent and apache distributions
kafka_topics: ["metrics", "logs"]
kafka_package_list: ["java-1.8.0-openjdk", "java-1.8.0-openjdk-devel"]
```

This default configuration defines default values for all of the parameters needed to deploy an instance of Kafka (both the Apache and the Confluent Kafka distributions are supported) to a node, including defining reasonable defaults for the network interface the Kafka instance should listen ("eth0" by default) and the topics that should automatically be create every Kafka node (the "metrics" and "logs" topics by default).  To deploy Kafka to a node the IP address "192.168.34.8" using the role in this repository (by default the Confluent Kafka distribution will be used), one would simply run a command that looks like this:

```bash
$ export KAFKA_ADDR="192.168.34.8"
$ ansible-playbook -i "${KAFKA_ADDR}," -e "{host_inventory: ['${KAFKA_ADDR}']}" site.yml
```

## Deploying the Apache Kafka distribution
The easiest way to switch from a Confluent Kafka install to an Apache Kafka install is to simply override the default `kafka_distro` variable on the command-line by using the `ansible-playbook` command's `--extra-vars` flag.  The resulting command would look something like this:

```bash
$ export KAFKA_ADDR="192.168.34.8"
$ export KAFKA_DISTRO="apache"
$ ansible-playbook -i "${KAFKA_ADDR}," -e "{ host_inventory: ['${KAFKA_ADDR}'], \
    kafka_distro: '${KAFKA_DISTRO}' }" site.yml
```

Note that in this case we are overriding the values defined in the `vars/kakfa.yml` file (above) with the values that are necessary to create an Apache Kafka instance on that node.  Alternatively, one could simply modify the `kafka_distro` value defined in the `vars/kafka.yml` file so that an Apache (rather than a Confluent) Kafka distribution was deployed to the node.  To make this process easier, the `vars/kafka.yml` file that is shown above includes all of the settings needed, simply modify the `kafka_distro` value so that the resulting configuration file looks something like this:

```yaml
# (c) 2016 DataNexus Inc.  All Rights Reserved
#
# Defaults that are necessary for all deployments of
# kafka
---
application: kafka
# the distribution of Kafka that should be installed (apache or confluent)
kafka_distro: apache
# the interface Kafka should listen on when running
kafka_iface: eth0
# the following parameters are only used when provisioning an instance
# of the apache distribution, but are uncommented here (regardless) to
# provide reasonable default values when provisioning via Vagrant (where
# the distribution being provisioned may be different from the default)
scala_version: "2.11"
kafka_version: "0.10.1.0"
kafka_url: "https://www-us.apache.org/dist/kafka/{{kafka_version}}/kafka_{{scala_version}}-{{kafka_version}}.tgz"
kafka_dir: "/opt/kafka"
# this value is only used when installing the confluent distribution,
# but is uncommented here so that it can be used if a confluent distribution
# is chosen when provisioning via Vagrant
confluent_version: "3.1"
# these parameters are used for both confluent and apache distributions
kafka_topics: ["metrics", "logs"]
kafka_package_list: ["java-1.8.0-openjdk", "java-1.8.0-openjdk-devel"]
```

With these changes in place, you could then use the same command (the one shown above for the default, Confluent Kafka distribution) to deploy an instance of the Apache Kafka distribution to that same node:

```bash
$ export KAFKA_ADDR="192.168.34.8"
$ ansible-playbook -i "${KAFKA_ADDR}," -e "{host_inventory: ['${KAFKA_ADDR}']}" site.yml
```

It should be noted here that any of the variables shown in the `vars/kafka.yml` file can be overridden on the command-line using the `--extra-vars` command-line flag.  As is the case with any other ansible-playbook run, the values defined on the command-line in this manner will override values defined in any of the `vars` files that are included in the playbook.  Keep in mind, however, that while this mechanism provides a simple method for overriding the underlying values defined in those `vars` files at runtime (without requiring that the underlying `vars` file be modified), the resulting configuration may be harder to track or reproduce over time (in a production deployment, for example) since the values that were set are not maintained under revision control.

# Assumptions
It is assumed that this playbook will be run on a recent (systemd-based) version of RHEL or CentOS (RHEL-7.x or CentOS-7.x, for example); no support is provided for other distributions (and the `site.xml` playbook will not run successfully).  The examples shown above also assume that some (shared-key?) mechanism has been used to provide access to the Kafka host from the Ansible host that the ansible-playbook is being run on (if not, then additional arguments might be required to authenticate with that host from the Ansible host that are not shown in the example `ansible-playbook` commands shown above).

# Deployment via vagrant
A Vagrantfile is included in this repository that can be used to deploy kafka to a VM using the `vagrant` command.  From the top-level directory of this repostory a command like the following will (by default) deploy kafka to a CentOS 7 virtual machine running under VirtualBox (assuming that both vagrant and VirtualBox are installed locally, of course):

```bash
$ vagrant -k="192.168.34.8" -d="apache" up
```

Note that the `-k` (or the corresponding `--kafka-addr`) flag must be used to pass an IP address into the Vagrantfile, and this IP address will be used as the IP address of the kafka server that is created by the vagrant command shown above.  In addition, the user *may* define the distribution of Kafka that they wish to deploy to that node using the `-d` (or the corresponding `--distro`) flag.  Valid values for the distribution are either `confluent` (the default) or `apache`.

If the Vagrantfile in this repository is being used to deploy the Confluent Kafka distribution to the specified node, then those are the only variables that need to be defined (the Confluent distribution is actually installed as a package, so no additional information is needed).  However, if you are deploying the Apache Kafka distribution to a node there are two additional parameters that must be defined for the playbook to succeed:  the `kafka_dir`, and `kafka_url` parameters.  As was mentioned earlier, default values for these two parameters are defined in the `vars/kafka.yml` file, and this file is pulled into the playbook contained in the `site.yml` file used by the Vagrantfile for provisioning.  As such, the Vagrantfile in this repository can be used (out of the box, so to speak) to deploy either the Confluent or Apache Kafka distributions to a node, without requiring editing of either the Vagrantfile or the `vars/kafka.yml` file.

## Additional vagrant deployment options
While the `vagrant up` command that is shown above can be used to easily deploy Kafka to a node, the Vagrantfile included in this distribution also supports separating out the creation of the virtual machine from the provisioning of that virtual machine using the Ansible playbook contained in this repository's `site.yml` file. To create a virtual machine without provisioning it, simply run a command that looks something like this:

```bash
$ vagrant -k="192.168.34.8" up --no-provision
```

This will create a virtual machine with the appropriate IP address ("192.168.34.8"), but will skip the process of provisioning that VM with an instance of the Confluent Kafka distribution using the playbook in the `site.yml` file.  To provision that machine with a Confluent Kafka instance, you would simply run the following command:

```bash
$ vagrant -k="192.168.34.8" provision
```

That command will attach to the named instance (the VM at "192.168.34.8") and run the playbook in this repository's `site.yml` file on that node (resulting in the deployment of an instance of the Confluent Kafka distribution to that node).

It should also be noted here that while the commands shown above will install Kafka with a reasonable default configuration from a standard location, there are two additional command-line parameters that can be used to override the default values that are embedded in the Vagrantfile that is included as part of this distribution when we are deploying an instance of the Apache Kafka distribution:  the `-u` (or corresponding `--url`) flag and the `-p` (or corresponding `--path`) flag.  The `-u` flag can be used to override the default URL that is used to download the Apache Kafka distribution (which points back to the main Apache Kafka distribution site), while the `-p` flag can be used to override the default path (`/opt/kafka`) that that the Apache Kafka gzipped tarfile is unpacked into during the provisioning process.  Both of these flags are silently ignored if an instance of the Confluent distribution is being provisioned since that distribution is installed as a package, not from a gzipped tarfile.

As an example of how these options might be used, the following command will download the gzipped tarfile containing the Apache Kafka distribution from a local web server, rather than downloading it from the main Apache Kafka distribution site, when provisioning the VM with an IP address of `192.168.34.8` with an instance of the Apache Kafka distribution:

```bash
$ vagrant -k="192.168.34.8" -d="apache" -u="https://10.0.2.2/dist/kafka/0.10.1.0/kafka_2.11-0.10.1.0.tgz" provision
```

Obviously, this option could prove to be quite useful in situations were we are deploying the distribution from a datacenter environment (where access to the internet may be restricted, or even unavailable).