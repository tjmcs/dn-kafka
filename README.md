# dn-kafka
Playbooks/Roles used to deploy Kakfa

# Installation
To install kafka using the `site.yml` playbook in this repository, first clone the contents of this repository to a local directory using a command like the following:
```bash
$ git clone --recursive https://github.com/Datanexus/dn-kafka
```
That command will pull down the repository and it's submodules (currently the only dependency embedded as a submodule is the dependency on the `https://github.com/Datanexus/common-roles` repository).

# Use
To run the included playbook, change directories to the `dn-kafka` subdirectory and run a set of commands that look something like the following (the commands shown here will install the most recent version of the Apache Kafka distribution from the main apache website, for example, onto a machine with at the IP address "192.168.34.8"):
```bash
$ export KAFKA_URL="https://www-us.apache.org/dist/kafka/0.10.1.0/kafka_2.11-0.10.1.0.tgz"
$ export KAFKA_ADDR="192.168.34.8"
$ export KAFKA_DISTRO="apache"
$ export KAFKA_DIR="/opt/kafka"
$ export KAFKA_TOPICS='["metrics", "logs"]'
$ echo "[all]\n${KAFKA_ADDR}" > hosts
$ ansible-playbook site.yml --inventory-file hosts --extra-vars "kafka_url=${KAFKA_URL} \
    kafka_addr=${KAFKA_ADDR} kafka_distro=${KAFKA_DISTRO} kafka_dir=${KAFKA_DIR} \
    kafka_topics=${KAFKA_TOPICS}"
```
The `kafka_topics` variable is optional (if a value for this variable is not provided, then no topics will be created); all other variables shown in this example must be defined for the playbook to run successfully.

If, on the other hand, you wanted to install the Confluent Kafka distribution onto that same node, a set of commands like the following would be used instead:
```bash
$ export KAFKA_ADDR="192.168.34.8"
$ export KAFKA_DISTRO="confluent"
$ export CONFLUENT_VER="3.1
$ export KAFKA_TOPICS='["metrics", "logs"]'
$ echo "[all]\n${KAFKA_ADDR}" > hosts
$ ansible-playbook site.yml --inventory-file hosts --extra-vars "kafka_addr=${KAFKA_ADDR} \
    kafka_distro=${KAFKA_DISTRO} confluent_version=${CONFLUENT_VER} \
    kafka_topics=${KAFKA_TOPICS}"
```
As was the case with the Apache Kafka deployment (above), the `kafka_topics` variable is optional (if a value for this variable is not provided, then no topics will be created); all other variables shown in this example must be defined for the playbook to run successfully.

# Assumptions
It is assumed that this playbook will be run on a recent (systemd-based) version of RHEL or CentOS (RHEL-7.x or CentOS-7.x, for example); no support is provided for other distributions (and the `site.xml` playbook will not run successfully).

# Deployment via vagrant
The included Vagrantfile can be used to deploy kafka to a VM using `Vagrant`.  From the top-level directory of this repostory a command like the following will (by default) deploy kafka to a CentOS 7 virtual machine running under VirtualBox (assuming that both vagrant and VirtualBox are installed locally, of course):
```bash
$ VAGRANT_DEFAULT_PROVIDER=virtualbox vagrant -k="192.168.34.8" -d="apache" up
```
Note that the `-k` (or the corresponding `--kafka-addr`) flag must be used to pass an IP address into the Vagrantfile (this IP address will be used as the IP address of the kafka server that is created by the vagrant command shown above) and the `-d` (or the corresponding `--distro`) flag must be used to pass in the kafka distribution that should be installed (valid values for the distribution are either `apache` or `confluent`).  The Vagrantfile also includes a definition for a `kafka_url` variable that must be updated to point to the gzipped tarfile containing the Apache Kafka distribution if that is the distribution being installed (this URL is not necessary if you are installing the `confluent` distribution, since that distribution can be installed directly as a system package using the `yum` command).
