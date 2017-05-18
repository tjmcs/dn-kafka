# Managing Topics

The [modfy-topics.yml](../modify-topics.yml) playbook in this repository can be used to modify the topics within a deployed Kafka instance or cluster. Specifically, the playbook in this playbook can be used to:

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
data_iface: eth0
api_iface: eth2
action_hash:
  action: create
  topic_list:
    - test1
    - test2
  repl_factor: 2
  partions: 2
```

The first few parameters (the `inventory_type`, `data_iface`, `api_iface`, and `zookeeper_inventory_file`) that are defined in this local variables file might look familiar to those of you who have used the [provision-kafka.yml](../provision-kafka.yml) playbook to provision Kafka to a set of target nodes and configure them as a Kafka cluster, but the process of adding new topics to an existing cluster requires that we define a couple of additional parameters. These additional parameters are contained in the `action_hash` that is defined at the end of the file shown above. When adding new topics, this `action_hash` is used to define the `action` that should be taken (creating topics in this example), the `topic_list` for that action (in this case we'll be adding a `test1` and `test2` topic in a single playbook run), and any additional parameters that must be defined for the action in question (when creating new topics, we must also specify the replication factor, or `repl_factor`, and the number of `partitions` for the topics that we are adding). With these parameters defined in our local variables file, adding these two topics to our cluster via an `ansible-playbook` run is as simple as the following command:

```bash
$ ansible-playbook -i combined_inventory -e "{ \
    local_vars_file: 'local-vars-add-topics.yml' \
}" modify-topics.yml
```

During the playbook run, Ansible will connect to the target nodes in the cluster and run the necessary commands to create the requested topics in the cluster. When that playbook run completes, the list of topics available in the cluster will include those that were defined in the local variables file shown above.

## Removing Topics from the Cluster

The process of removing topics from the cluster is a bit more involved. To successfully remove topics from the cluster, we must:

* use the `kafka-topics` command (or the equivalent `kafka-topics.sh` shell script if the Apache Kafka distribution was used to build our cluster instead of the Confluent Kafka distribution) to remove the topics in question from the cluster
* for the rest of this process we must iterate over the nodes of our cluster to ensure that only one of the cluster nodes is offline at any given time; this includes:
    * shutting down the `kafka` service on the cluster node in question
    * if this is the first node in the cluster, connecting to the Zookeeper instance or ensemble that the Kafka instance or cluster is integrated with and removing the meta-data associated with these topics from that instance or ensemble
    * if the user requested a that we make a backup the directories that were used to store the meta-data and data associated with these topics before removing them from the cluster, we do so by taking a snapshot (in the form of a bzipped tarfile) of each of the matching topic directories (if any) that we find in the `kafka_data_dir` on the node in question; this decision is based on the value of the `backup_topics` flag defined in the `action_hash` (if a value for that flag is not specified in the `action_hash` used for a particular playbook run, then no backup is made)
    * removing the directories that were used to store the meta-data and data associated with each of these topics (if any) in the `kafka_data_dir` on the node in question
    * restarting the `kafka` service on the node in question
    * pausing for a configurable amount of time (by default, the playbook pauses for 5 seconds between iterations) to ensure that the node is up before we move on to the next node in the cluster
* finally, if the cluster was built using the Apache Kafka distribution (rather than the Confluent Kafka distribution), then the same `kafka-topics.sh` command that was used in the first step of this process is re-run to ensure that the topics are really removed; without this additional command, if the a topic with the same name as the topic we are removing is created at some later time it will come up as a topic that has been _marked for deletion_ in clusters that have been built using the Apache Kafka distribution.

With that description of the process in mind, removing the topics that we created in our example, above, is a relatively simple matter. First, we need to create a local variables file that includes a couple of additional paramters:


```bash
$ cat local-vars-remove-topics.yml
---
data_iface: eth0
api_iface: eth2
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

* the `kafka_data_dir`: the value defined for this parmeter must be the same as the value that was used for this parameter when the Kafka cluster was created; its value points to the directory that the Kafka cluster nodes use to store their meta-data and data, including the data associated with the topics they are managing; if this parameter is unspecified a default value of `/var/lib` is used by default
* the `remote_zookeeper_dir`: the value defined for this parmeter must be the same as the `zookeeper_dir` value that was used when the Zookeeper node or ensemble associated with the Kafka instance or cluster was created; its value points to the directory where the Zookeeper distribution was installed on the Zookeeper node or nodes and it is assumed that the `zkCli.sh` command that is used to remove the topic-related metadata from the Zookeeper node or ensemble can be found in the `{{remote_zookeeper_dir}}/bin` directory on the corresponding Zookeeper node or nodes

In addition, we've changed the `action` in our `action_hash` (from `create` to `delete`) and set the `backup_topics` flag to `true`. The fields that were used to set the replication factor and number of partitions for the listed topics are not necessary for this command so we have not bothered to define them here (if they were defined, they would be silently ignored). With this local variables file constructed, removing the two topics defined in that file is as simple as running the following `ansible-playbook` command:

```bash
$ ansible-playbook -i combined_inventory -e "{ \
    local_vars_file: 'local-vars-remove-topics.yml' \
}" modify-topics.yml
```

This command will remove the topics in question from the Kafka cluster, remove the meta-data associated with those topics from the associated Zookeeper node or nodes, backup the topic-related directories on each of the target nodes (since the `backup_topics` flag was set to `true`), and remove those topic-related directories from the target nodes. When the playbook run is complete, the topics listed in the `topic_list` of the `action_hash` defined in the local variables file shown above will no longer appear in the list of topics being managed by the Kafka instance or cluster.

Finally, it should be noted that, as is the case with the [provision-kafka.yml](../provision-kafka.yml) playbook, the [modfy-topics.yml](../modify-topics.yml) playbook includes a [shebang](https://en.wikipedia.org/wiki/Shebang_(Unix)) line at the beginning of the playbook file. As such, the playbook can be executed directly as a shell script (rather than using the file as the final input to an `ansible-playbook` command). This means that the command that was used above to add topics to the cluster could be rewritten as:

```bash
./modify-topics.yml -i combined_inventory -e "{ \
    local_vars_file: 'local-vars-add-topics.yml' \
}"
```

and the corresponding command to remove those topics from the cluster could be rewritten as:


```bash
./modify-topics.yml -i combined_inventory -e "{ \
    local_vars_file: 'local-vars-remove-topics.yml' \
}"
```

both forms (running the playbook as a shell script and using the playbook as the final input to an `ansible-playbook` command) accomplish the same thing; which form you choose to use is a matter of personal preference.
