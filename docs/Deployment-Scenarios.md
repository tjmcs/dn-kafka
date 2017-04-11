# Example deployment scenarios

There are a three basic deployment scenarios that are supported by this playbook. In the first two scenarios (shown below) we'll walk through the deployment of Kafka to a single node and the deployment of a multi-node Kafka cluster using a static inventory file, discussing the differences between the deployment mechanisms used for the two distributions supported by this playbook (the [Confluent Kafka](https://www.confluent.io/) distribution and the [Apache Kafka](https://kafka.apache.org/) distribution) as we go. Finally, in the third scenario, we will show how the same multi-node Kafka cluster deployment shown in the second scenario could be performed using the dynamic inventory scripts for both AWS and OpenStack instead of a static inventory file.

## Scenario #1: deploying Kafka to a single node
While this is the simplest of the deployment scenarios that are supported by this playbook, it is more than likely that deployment of Kafka to a single node is really only only useful for very small workloads or deployments of simple test environments. Nevertheless, we will start our discussion with this deployment scenario since it is the simplest.

If we want to deploy the either of the two supported Kafka distributions to a single node with the IP address "192.168.34.2", we would start by creating a very simple inventory file that looks something like the following:

```bash
$ cat single-node-inventory

192.168.34.2 ansible_ssh_user='cloud-user' ansible_ssh_private_key_file='keys/test_node_private_key'

$ 
```

Note that in this inventory file the `ansible_ssh_host` and `ansible_ssh_port` parameters will take their default values since they aren't specified for the single host entry in the inventory file.

Once we've built our static inventory file, we need to decide which Kafka distribution we want to deploy to our node. For purposes of this example, let's assume that we want to deploy the Confluent distribution (the default). To deploy the Confluent Kafka distribution to our single node, we would simply run an `ansible-playbook` command that looks something like this:

```bash
$ ansible-playbook -i single-node-inventory -e "{ host_inventory: ['192.168.34.2'] }" site.yml
```

This will install the Confluent Kafka distribution as a set of packages, using the default package repository that is defined in the [templates/confluent-repo.j2](../templates/confluent-repo.j2) template, and configure the node as a single-node Kafka instance (using the default configuration parameters that are defined in the [vars/kafka.yml](../vars/kafka.yml) file). It should be noted here that When deploying a single-node Kafka instance, regardless of the distribution, an instance of Zookeeper will also be deployed locally on the same node. The Kafka instance will then be configured to work with that bundled Zookeeper instance.

If we wanted to deploy the Apache Kafka distribution to our node instead of the Confluent Kafka distribution, we could simply modify the `kafka_distro` flag that is passed into the playbook (either by modifying the value defined in the [vars/kafka.yml](../vars/kafka.yml) file or by passing in an extra variable that redefines the value from it's default value of `confluent`). If we take the second approach, this is what the `ansible-playbook` command for this simple deployment would look like:

```bash
$ ansible-playbook -i single-node-inventory -e "{ host_inventory: ['192.168.34.2'], \
    kafka_distro: 'apache' }" site.yml
```

This command will download the Apache Kafka distribution from the default URL defined in the [vars/kafka.yml](../vars/kafka.yml) file (the main Apache download site), unpack that distribution file into the `/opt/kafka` directory on that host (the default value for the location to unpack the distribution into), and configure both the bundled `zookeeper` instance and the `kafka` instance on that host, enable those services to start on boot, and (re)start them in that order (first `zookeeper`, then `kafka`).

Regardless of which distribution is deployed, when the playbook run is complete and the `zookeeper` and `kafka` services are up and running on the host, two topics will be created on the node by default, the `metrics` and `logs` topics. The names of the topics created can be modified by changing the `kafka_topics` value that is defined in the [vars/kafka.yml](../vars/kafka.yml) file or by passing a `kafka_topics` list into the `ansible-playbook` command as an extra variable (and, of course, passing in an empty list will result in no topics being created during the playbook run).

## Scenario #2: deploying a multi-node Kafka cluster
If you are using this playbook to deploy a multi-node Kafka cluster, then the configuration becomes a bit more complicated. The playbook assumes that if you are deploying a Kafka cluster you will also want to have a multi-node Zookeeper ensemble associated with that cluster. Furthermore, to ensure that the resulting pair of clusters are relatively tolerant of the failure of a node, we highly recommend that the deployments of these two clusters (the Kafka cluster and the Zookeeper ensemble) be made to separate sets of nodes. For the playbook in this repository, this separation of the Zookeeper ensemble from the Kafka cluster is made by assuming that in deploying a Kafka cluster we will be configuring the nodes of that cluster to work with a multi-node Zookeeper ensemble that has **already been deployed and configured separately** (and we provide a separate role and playbook, both of which are defined in the DataNexus [dn-zookeeper] (https://github.com/DataNexus/dn-zookeeper) repository, that can be used to manage the process of deploying and configuring the necessary Zookeeper ensemble).

So, assuming that we've already deployed a three-node Zookeeper ensemble separately and that we want to deploy a three node Confluent Kafka cluster (other than the difference in the `kafka_distro` value used in the `ansible-playbook` command, the deployment of an Apache Kafka cluster will look exactly the same), let's walk through what the commands that we'll need to run look like. In addition, let's assume that we're going to be using a static inventory file to control our Kafka deployment. The static inventory file that we will be using for this example looks like this:

```bash
$ cat test-cluster-inventory
# example inventory file for a clustered deployment

192.168.34.8 ansible_ssh_host= 192.168.34.8 ansible_ssh_port=22 ansible_ssh_user='cloud-user' ansible_ssh_private_key_file='keys/kafka_cluster_private_key'
192.168.34.9 ansible_ssh_host= 192.168.34.9 ansible_ssh_port=22 ansible_ssh_user='cloud-user' ansible_ssh_private_key_file='keys/kafka_cluster_private_key'
192.168.34.10 ansible_ssh_host= 192.168.34.10 ansible_ssh_port=22 ansible_ssh_user='cloud-user' ansible_ssh_private_key_file='keys/kafka_cluster_private_key'

$
```

To correctly configure our Kafka cluster to talk to the Zookeeper ensemble, the playbook will need to connect to the nodes that make up the associated Zookeeper ensemble and collect information from them, and to do so we'll have to pass in the information that Ansible will need to make those connections to the playbook. We do this by passing in a separate hash of hashes (the `zookeeper_inventory` for the deployment) that maps the same parameters shown above to each of the members of our Zookeeper ensemble. For the purposes of this example, let's assume that our `zookeeper_inventory` hash map looks something like this:

```json
    {
      '192.168.34.18': { ansible_ssh_host: '192.168.34.18', ansible_ssh_port: 22, ansible_ssh_user: 'cloud-user', ansible_ssh_private_key_file: 'keys/zk_cluster_private_key'},
      '192.168.34.19': { ansible_ssh_host: '192.168.34.19', ansible_ssh_port: 22, ansible_ssh_user: 'cloud-user', ansible_ssh_private_key_file: 'keys/zk_cluster_private_key'},
      '192.168.34.20': { ansible_ssh_host: '192.168.34.20', ansible_ssh_port: 22, ansible_ssh_user: 'cloud-user', ansible_ssh_private_key_file: 'keys/zk_cluster_private_key'},
    }
```

To deploy Kafka to the three nodes in our static inventory file, we'd run a command that looks something like this:

```bash
$ ansible-playbook -i test-cluster-inventory -e "{ \
      host_inventory: ['192.168.34.8', '192.168.34.9', '192.168.34.10'], \
      cloud: vagrant, data_iface: eth0, api_iface: eth1, \
      zookeeper_nodes: ['192.168.34.18', '192.168.34.19', '192.168.34.20'], \
      zookeeper_inventory: {
          '192.168.34.18': { ansible_ssh_host: '192.168.34.18', ansible_ssh_port: 22, \
              ansible_ssh_user: 'cloud-user',  \
              ansible_ssh_private_key_file: 'keys/zk_cluster_private_key' }, \
          '192.168.34.19': { ansible_ssh_host: '192.168.34.19', ansible_ssh_port: 22,  \
              ansible_ssh_user: 'cloud-user',  \
              ansible_ssh_private_key_file: 'keys/zk_cluster_private_key' }, \
          '192.168.34.20': { ansible_ssh_host: '192.168.34.20', ansible_ssh_port: 22,  \
              ansible_ssh_user: 'cloud-user',  \
              ansible_ssh_private_key_file: 'keys/zk_cluster_private_key' }, \
      }, \
      kafka_url: 'http://192.168.34.254/confluent/confluent-3.2.0.tar.gz', \
      yum_repo_url: 'http://192.168.34.254/centos', kafka_data_dir: '/data' \
    }" site.yml
```

Alternatively, rather than passing all of those arguments in on the command-line as extra variables, we can make use of the *local variables file* support that is built into this playbook and construct a YAML file that looks something like this containing the configuration parameters that are being used for this deployment:

```yaml
cloud: vagrant
data_iface: eth0
api_iface: eth1
zookeeper_nodes:
    - '192.168.34.18'
    - '192.168.34.19'
    - '192.168.34.20'
zookeeper_inventory:
    '192.168.34.18':
        ansible_ssh_host: '192.168.34.18'
        ansible_ssh_port: 22
        ansible_ssh_user: 'cloud-user'
        ansible_ssh_private_key_file: 'keys/zk_cluster_private_key'
    '192.168.34.19':
        ansible_ssh_host: '192.168.34.19'
        ansible_ssh_port: 22
        ansible_ssh_user: 'cloud-user'
        ansible_ssh_private_key_file: 'keys/zk_cluster_private_key'
    '192.168.34.20':
        ansible_ssh_host: '192.168.34.20'
        ansible_ssh_port: 22
        ansible_ssh_user: 'cloud-user'
        ansible_ssh_private_key_file: 'keys/zk_cluster_private_key'
kafka_url: 'http://192.168.34.254/confluent/confluent-3.2.0.tar.gz'
kafka_data_dir: 'http://192.168.34.254/centos'
solr_data_dir: '/data'
```

and then we can pass in the *local variables file* as an argument to the `ansible-playbook` command; assuming the YAML file shown above was in the current working directory and was named `test-cluster-deployment-params.yml`, the resulting command would look somethin like this:

```bash
$ ansible-playbook -i test-cluster-inventory -e "{ \
      host_inventory: ['192.168.34.8', '192.168.34.9', '192.168.34.10'], \
      local_vars_file: 'test-cluster-deployment-params.yml' \
    }" site.yml
```

Once the playbook run is complete, we can SSH into one of our nodes and run a few simple commands to check and make sure that Kafka has been successfully started and that the requested topics have been created successfully:

```bash
$ systemctl status kafka
● kafka.service - Confluent Kafka server (broker)
   Loaded: loaded (/etc/systemd/system/kafka.service; enabled; vendor preset: disabled)
   Active: active (running) since Fri 2017-04-07 22:08:19 UTC; 24min ago
     Docs: http://docs.confluent.io
 Main PID: 14407 (java)
   CGroup: /system.slice/kafka.service
           └─14407 java -Xmx1G -Xms1G -server -XX:+UseG1GC -XX:MaxGCPauseMillis=20 -XX:InitiatingHeapOccupancyPercent=35 -XX:+DisableExplicitGC -Djava.awt.headless=true -Xloggc:/var/log/k...
$ kafka-topics --zookeeper=192.168.34.18:2181 --list
OpenJDK 64-Bit Server VM warning: If the number of processors is expected to increase from one, then you should configure the number of parallel GC threads appropriately using -XX:ParallelGCThreads=N
_schemas
logs
metrics
$ 
```

Additional checks could be performed (using the command-line tools to add messages to these topics and to read those messages. from another node in the cluster using the same command-line tools, for example), but we will leave those as an exercise for the reader. For more complete monitoring of the cluster status, a tool like LinkedIn's [kafka-monitor](https://github.com/linkedin/kafka-monitor) can be setup and run locally; again we won't show the details of setting up, configuring, and using those sorts of solutions for Kafka cluster monitoring here.

In closing, this section showed how a single `ansible-playbook` command could be used to deploy a three-node Kafka cluster from a static inventory file, provided that there was already a Zookeeper ensemble up and running in the same network. 

## Scenario #3: deploying a Kafka cluster via dynamic inventory
In this section we will repeat the multi-node cluster deployment that we just showed in the previous scenario, but we will use the dynamic inventory scripts provided in the [common-utils](../common-utils) submodule to control the deployment of our Kafka cluster to an AWS or OpenStack environment rather than relying on a static inventory file.

To accomplish this, the we have to:

* Tag the instances in the AWS or OpenStack environment that we will be configuring as a Kafka cluster with the appropriate `Tenant`, `Project`, `Domain`, and `Application` tags. Note that for all of the nodes we are targeting in our playbook runs, we will assign an `Application` tag of `kafka`
* Have already deployed a Zookeeper ensemble into this environment; this ensemble should be tagged with the same  `Tenant`, `Project`, `Domain` tags that were used when tagging the instances that will make up our Kafka cluster (above); obviously the  `Application` for the Zookeeper ensemble nodes would be `zookeeper`
* Once all of the nodes that will make up our cluster had been tagged appropriately, we can run `ansible-playbook` command similar to the `ansible-playbook` command shown in the previous scenario; this will deploy Kafka to the cluster nodes and configure them to talk to the each other through the associated Zookeeper ensemble

In terms of what the commands look like, lets assume for this example that we've tagged our seed nodes with the following VM tags:

* **Tenant**: labs
* **Project**: projectx
* **Domain**: preprod
* **Application**: kafka

The `ansible-playbook` command used to deploy Kafka to our nodes and configure them as a cluster in an OpenStack environment would then look something like this:

```bash
$ ansible-playbook -i common-utils/inventory/osp/openstack -e "{ \
        host_inventory: 'meta-Application_kafka:&meta-Cloud_osp:&meta-Tenant_labs:&meta-Project_projectx:&meta-Domain_preprod', \
        application: kafka, cloud: osp, tenant: labs, project: projectx, domain: preprod, \
        ansible_user: cloud-user, private_key_path: './keys', data_iface: eth0, api_iface: eth1, \
        kafka_data_dir: '/data' \
    }" site.yml
```

The playbook then uses the tags in this playbook run to identify the nodes that make up the associated Zookeeper ensemble (which must be up and running for this playbook command to work) as well as the target nodes for the playbook run, installs the Confluent Kafka distribution on those nodes, and configures them as a new Kafka cluster. 

In an AWS environment, the command would look something like this:

```bash
$ ansible-playbook -i common-utils/inventory/aws/ec2 -e "{ \
        host_inventory: 'tag_Application_kafka:&tag_Cloud_aws:&tag_Tenant_labs:&tag_Project_projectx:&tag_Domain_preprod', \
        application: kafka, cloud: aws, tenant: labs, project: projectx, domain: preprod, \
        ansible_user: cloud-user, private_key_path: './keys', data_iface: eth0, api_iface: eth1, \
        kafka_data_dir: '/data' \
    }" site.yml
```

As you can see, this command is basically the same command that was shown for the OpenStack use case; it only differs slightly in terms of the name of the inventory script passed in using the `-i, --inventory-file` command-line argument, the value passed in for `Cloud` tag (and the value for the associated `cloud` variable), and the prefix used when specifying the tags that should be matched in the `host_inventory` value (`tag_` instead of `meta-`). In both cases the result would be a set of nodes deployed as a Kafka cluster, with the number of nodes in the cluster determined (completely) by the tags that were assigned to the VMs in the cloud environment in question.
