# Managing Connectors

In addition to supporting the deployment of Kafka to one or more target nodes, the playbook in this repository also supports management of topics within a deployed Kafka instance or cluster. Specifically, the playbook in this repository can be used to:

* Add new topics to the cluster
* Remove existing topics from the cluster

We will describe these capabilities more in the sections that follow. The only assumption made is that the target nodes for the playbook run are actually members of a Kafka cluster.

## Adding Topics to the Cluster

The easiest way to control the process of adding topics to a Kafka node or cluster is to create a local variables file. If we create a static configuration file that looks something like this:

```bash
$ cat combined-inventory
# example combined inventory file for clustered deployment

192.168.34.8 ansible_ssh_host=192.168.34.8 ansible_ssh_port=22 ansible_ssh_user='cloud-user' ansible_ssh_private_key_file='keys/kafka_cluster_private_key'
192.168.34.9 ansible_ssh_host=192.168.34.9 ansible_ssh_port=22 ansible_ssh_user='cloud-user' ansible_ssh_private_key_file='keys/kafka_cluster_private_key'
192.168.34.10 ansible_ssh_host=192.168.34.10 ansible_ssh_port=22 ansible_ssh_user='cloud-user' ansible_ssh_private_key_file='keys/kafka_cluster_private_key'
192.168.34.18 ansible_ssh_host=192.168.34.18 ansible_ssh_port=22 ansible_ssh_user='cloud-user' ansible_ssh_private_key_file='keys/zk_cluster_private_key'
192.168.34.19 ansible_ssh_host=192.168.34.19 ansible_ssh_port=22 ansible_ssh_user='cloud-user' ansible_ssh_private_key_file='keys/zk_cluster_private_key'
192.168.34.20 ansible_ssh_host=192.168.34.20 ansible_ssh_port=22 ansible_ssh_user='cloud-user' ansible_ssh_private_key_file='keys/zk_cluster_private_key'

[kafka]
192.168.34.8
192.168.34.9
192.168.34.10

[zookeeper]
192.168.34.18
192.168.34.19
192.168.34.20

```

Then the local variables file that would be used to add two new topics to the cluster might look something like this:

```bash
$ cat local-vars-add-topics.yml
---
inventory_type: static
modify_topics: true
data_iface: eth0
api_iface: eth2
zookeeper_inventory_file: './combined_inventory'
action_hash:
  action: create
  topic_list:
    - test1
    - test2
  repl_factor: 2
  partions: 2
```

The first few parameters (the `inventory_type`, `data_iface`, `api_iface`, and `zookeeper_inventory_file`) that are defined in this local variables file might look familiar to those of you who have used the playbook in this repository to provision a Kafka distribution to one or more target nodes and configure them as a Kafka cluster, but the process of adding new topics to an existing cluster requires that we define a couple of additional parameters. The first is the `modify_topics` flag which, when set to `true` (as it is in this local variables file), ensures that the plays involved in modifying the topics in the cluster will be invoked.

The second parameter that may be unfamiliar to some of you is the `action_hash` that is defined at the end of this file. When adding new topics, this `action_hash` is used to define the `action` that should be taken (adding topics in this example), the `topic_list` for that action (in this case we'll be adding a `test1` and `test2` topic in a single playbook run), and a couple of additional parameters that must be defined when creating new topics (the replication factor or `repl_factor` and the number of `partitions` for the topics that we are adding). With these parameters defined in our local variables file, adding these two topics to our cluster via an `ansible-playbook` run is as simple as the following command:

```bash
$ ansible-playbook -i combined_inventory -e "{ \
    host_inventory: ['192.168.34.8','192.168.34.9','192.168.34.10'], \
    local_vars_file: 'local-vars-add-topics.yml' \
}" site.yml
```

During the playbook run, Ansible will connect to the target nodes in the cluster and run the necessary commands to create the requested topics in the cluster. When that playbook run completes, the list of topics available in the cluster will include those that were defined in the local variables file shown above.

## Removing Topics from the Cluster

The process of removing topics from the cluster is a bit more involved. To successfully remove topics from the cluster, we must:

* use the `kafka-topics` command (or the equivalent `kafka-topics.sh` shell script if the Apache Kafka distribution was used to build our cluster instead of the Confluent Kafka distribution) to remove the topics in question
* shut down the `kafka` service on each of the nodes in our cluster
* backup the directories that were used to store the meta-data and data associated with these topics on the nodes that make up the Kafka cluster in the `kafka_data_dir` on each of these nodes if the user requested that a backup be made before the topics are removed; this decision is based on the value of the `backup_topics` flag defined in the `action_hash` (if a value for that flag is not specified in the `action_hash` used for a particular playbook run, then no backup is made)
* remove the directories that were used to store the meta-data and data associated with these topics on the nodes that make up our Kafka cluster from the `kafka_data_dir` on each of these nodes
* connect to the Zookeeper instance or ensemble that the Kafka instance or cluster is integrated with and remove the meta-data associated with these topics from that instance or ensemble

Once all of those steps have been successfully completed we can restart the `kafka` service on each of the nodes in our cluster. Obviously, this is an involved process that requires some downtime on the part of our cluster (however brief it may be), so we recommend against removing topics from the cluster unless such removal is absolutely necessary. It should also be noted here that there is no attempt made, at any time during this processs, to determine whether there are any clients listening on the topics that are being removed. It is up to you, as a user, to ensure that such processes are shut down smoothly before any attempt is made to remove the topics that those processes depend on from the cluster.

With that description of the process in mind, removing the topics that we created in our example, above, is a relatively simple matter. First, we need to create a local variables file that includes a couple of additional paramters:


```bash
$ cat local-vars-remove-topics.yml
---
inventory_type: static
modify_topics: true
data_iface: eth0
api_iface: eth2
zookeeper_inventory_file: './combined_inventory'
kafka_data_dir: '/data'
remote_zookeeper_dir: '/opt/zookeeper'
action_hash:
  action: delete
  topic_list:
    - test1
    - test2
  backup_topics: true
```

As you can see (by comparing with our previous example), we've added definitions for the following pair of parameters:

* the `kafka_data_dir`: the value defined for this parmeter must be the same as the value that was used for this parameter when the Kafka cluster was created; it's value points to the directory that the Kafka cluster nodes use to store their meta-data and data, including the data associated with the topics they are managing; if this parameter is unspecified a default value of `/var/lib` is actually used)
* the `remote_zookeeper_dir`: the value defined for this parmeter must be the same as the `zookeeper_dir` value that was used when the Zookeeper node or ensemble associated with the Kafka instance or cluster was created; it's value points to the directory where the Zookeeper distribution was installed on the Zookeeper node or nodes and it is assumed that the `zkCli.sh` command that is used to remove the topic-related metadata from the Zookeeper node or ensemble can be found in the `{{remote_zookeeper_dir}}/bin` directory on the corresponding Zookeeper node or nodes

With this local variables file constructed, removing the two topics defined in that file is as simple as running the following `ansible-playbook` command:

```bash
$ ansible-playbook -i combined_inventory -e "{ \
    host_inventory: ['192.168.34.8','192.168.34.9','192.168.34.10'], \
    local_vars_file: 'local-vars-remove-topics.yml' \
}" site.yml
```

This command will remove the topics in question from the Kafka cluster, backup the topic-related directories on each of the target nodes (since the `backup_topics` flag was set to `true`), remove those topic-related directories from the target nodes, and remove the meta-data associated with those topics from the associated Zookeeper node or nodes. When the playbook run is complete, the topics listed in the `topic_list` of the `action_hash` defined in the local variables file shown above will no longer appear in the list of topics being managed by the Kafka instance or cluster.