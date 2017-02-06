# (c) 2016 DataNexus Inc.  All Rights Reserved
# -*- mode: ruby -*-
# vi: set ft=ruby :

require 'optparse'
require 'resolv'

# monkey-patch that is used to leave unrecognized options in the ARGV
# list so that they can be processed by underlying vagrant command
class OptionParser
  # Like order!, but leave any unrecognized --switches alone
  def order_recognized!(args)
    extra_opts = []
    begin
      order!(args) { |a| extra_opts << a }
    rescue OptionParser::InvalidOption => e
      extra_opts << e.args[0]
      retry
    end
    args[0, 0] = extra_opts
  end
end

options = {}
# define an array containg the list of Kafka distributions supported by the
# underlying playbook used for provisioning
VALID_KAFKA_DISTROS = ['confluent', 'apache']

# vagrant commands that include these commands can be run without specifying
# any IP addresses
no_ip_commands = ['version', 'global-status', '--help', '-h']
# vagrant commands that only work for a single IP address
single_ip_commands = ['status', 'ssh']
# vagrant command arguments that indicate we are provisioning a cluster (if multiple
# nodes are supplied via the `--kafka-list` and `--zookeeper-list` flags)
provisioning_command_args = ['up', 'provision']
not_provisioning_flag = ['--no-provision']

optparse = OptionParser.new do |opts|
  opts.banner    = "Usage: #{opts.program_name} [options]"
  opts.separator "Options"

  options[:node_list] = nil
  opts.on( '-n', '--node-list A1[,A2,...]', 'Node address list (single-node commands/provisioning)' ) do |node_list|
    # while parsing, trim an '=' prefix character off the front of the string if it exists
    # (would occur if the value was passed using an option flag like '-k=192.168.1.1')
    options[:node_list] = node_list.gsub(/^=/,'')
  end

  options[:kafka_list] = nil
  opts.on( '-k', '--kafka-list A1,A2[,...]', 'Kafka address list (multi-node provisioning)' ) do |kafka_list|
    # while parsing, trim an '=' prefix character off the front of the string if it exists
    # (would occur if the value was passed using an option flag like '-k=192.168.1.1')
    options[:kafka_list] = kafka_list.gsub(/^=/,'')
  end

  options[:zookeeper_list] = nil
  opts.on( '-z', '--zookeeper-list A1,A2[,...]', 'Zookeeper address list (multi-node provisioning)' ) do |zookeeper_list|
    # while parsing, trim an '=' prefix character off the front of the string if it exists
    # (would occur if the value was passed using an option flag like '-k=192.168.1.1')
    options[:zookeeper_list] = zookeeper_list.gsub(/^=/,'')
  end

  options[:kafka_only] = false
  opts.on( '-o', '--kafka-only', 'Only provision the kafka nodes (zookeeper nodes should be up)' ) do |kafka_only|
    # while parsing, trim an '=' prefix character off the front of the string if it exists
    # (would occur if the value was passed using an option flag like '-k=192.168.1.1')
    options[:kafka_only] = true
  end

  options[:kafka_distro] = nil
  opts.on( '-d', '--distro DISTRO_NAME', 'Kafka distribution (apache or confluent)') do |kafka_distro|
    # while parsing, trim an '=' prefix character off the front of the string if it exists
    # (would occur if the value was passed using an option flag like '-k=192.168.1.1')
    options[:kafka_distro] = kafka_distro.gsub(/^=/,'')
  end

  options[:kafka_path] = nil
  opts.on( '-p', '--path KAFKA_DIR', 'Installation path (Apache Kafka only)' ) do |kafka_path|
    # while parsing, trim an '=' prefix character off the front of the string if it exists
    # (would occur if the value was passed using an option flag like '-p=/opt/kafka')
    options[:kafka_path] = kafka_path.gsub(/^=/,'')
  end

  options[:kafka_url] = nil
  opts.on( '-u', '--url KAFKA_URL', 'URL for distribution (Apache Kafka only)' ) do |kafka_url|
    # while parsing, trim an '=' prefix character off the front of the string if it exists
    # (would occur if the value was passed using an option flag like '-u=http://localhost/tmp.tgz')
    options[:kafka_url] = kafka_url.gsub(/^=/,'')
  end

  options[:local_kafka_path] = nil
  opts.on( '-l', '--local-kafka-path PATH', 'Directory containing Kafka RPMs (Confluent Kafka only)' ) do |local_kafka_path|
    # while parsing, trim an '=' prefix character off the front of the string if it exists
    # (would occur if the value was passed using an option flag like '-u=http://localhost/tmp.tgz')
    options[:local_kafka_path] = local_kafka_path.gsub(/^=/,'')
  end

  options[:yum_repo_addr] = nil
  opts.on( '-y', '--yum-repo-host REPO_HOST', 'Local yum repository hostname/address' ) do |yum_repo_addr|
    # while parsing, trim an '=' prefix character off the front of the string if it exists
    # (would occur if the value was passed using an option flag like '-y=192.168.1.128')
    options[:yum_repo_addr] = yum_repo_addr.gsub(/^=/,'')
  end

  options[:kafka_log_dir] = nil
  opts.on( '-r', '--remote-log-dir REPO_HOST', 'Local yum repository hostname/address' ) do |kafka_log_dir|
    # while parsing, trim an '=' prefix character off the front of the string if it exists
    # (would occur if the value was passed using an option flag like '-r="/data"')
    options[:kafka_log_dir] = kafka_log_dir.gsub(/^=/,'')
  end

  opts.on_tail( '-h', '--help', 'Display this screen' ) do
    print opts
    exit
  end

end

# default to using the confluent distro if no distro was specified
options[:kafka_distro] = "confluent"
begin
  optparse.order_recognized!(ARGV)
rescue SystemExit => e
  exit
rescue Exception => e
  print "ERROR: could not parse command (#{e.message})\n"
  print optparse
  exit 1
end

# check remaining arguments to see if the command requires
# an IP address (or not)
ip_required = (ARGV & no_ip_commands).empty?
# check the remaining arguments to see if we're provisioning or not
provisioning_command = !((ARGV & provisioning_command_args).empty?) && (ARGV & not_provisioning_flag).empty?
# and to see if multiple IP addresses are supported (or not) for the
# command being invoked
single_ip_command = !((ARGV & single_ip_commands).empty?)

if options[:kafka_url] && !(options[:kafka_url] =~ URI::regexp)
  print "ERROR; input kafka URL '#{options[:kafka_url]}' is not a valid URL\n"
  exit 3
end

if options[:local_kafka_path] && !File.directory?(options[:local_kafka_path])
  print "ERROR; input local Kafka path '#{options[:local_kafka_path]}' is not a local directory\n"
  exit 3
end

if !(VALID_KAFKA_DISTROS.include?(options[:kafka_distro]))
    print "ERROR; Unrecognized kafka_distro value specified; valid values are #{VALID_KAFKA_DISTROS}\n"
    exit 4
end

deployment_addr_hash = {}
# if we're provisioning, then either the `--node-list` flag should be provided
# and only contain a single node or the `--kafka_list` and `--zookeeper-list` flags
# must both be provided and contain multiple nodes that define a valid definition for
# the kafka and zookeeper cluster's we're provisioning (respectively)
if provisioning_command
  if options[:node_list]
    node_addr_array = options[:node_list].split(',').map { |elem| elem.strip }.reject { |elem| elem.empty? }
    not_ip_addr_list = node_addr_array.select { |addr| !(addr =~ Resolv::IPv4::Regex) }
    # if any of the values in the node list are not valid IP addresses, print an error and exit
    if node_addr_array.size > 1
      print "ERROR; only single-node provisioning is supported via the `--node-list` flag\n"
      print "       for multi-node provisioning the `--kafka-list` and `--zookeeper-list`\n"
      print "       flags should be used instead\n"
      exit 5
    elsif not_ip_addr_list.size > 0
      print "ERROR; input node IP address #{not_ip_addr_list} is not a valid IP address\n"
      exit 2
    end
    deployment_addr_hash['zookeeper'] = node_addr_array
  elsif options[:kafka_list] && !options[:zookeeper_list]
    print "ERROR; for multi-node Kafka provisioning, both the `--kafka-list` and `--zookeeper-list`\n"
    print "       flags must be used to define the kafka and zookeeper clusters (respectively)\n"
    exit 5
  else
    # if we're provisioning, we can provision the Zookeeper nodes separately from
    # the Kafka nodes but must supply the IP address for the nodes in the Zookeeper
    # cluster if we're provisioning the Kafka nodes; long story short, the Kafka node
    # IP addresses are optional but the zookeeper node IP addresses are required
    # for any multi-node cluster provisioning we may be provisioning
    kafka_addr_array = []
    if options[:kafka_list]
      # check the input kafka address list to ensure that it contains only IP addresses
      # that it defines a valid cluster of kafka nodes (only clusters made up of an odd
      # number of members between three and seven are allowed)
      VALID_CLUSTER_SIZES = [3, 5, 7]
      kafka_addr_array = options[:kafka_list].split(',').map { |elem| elem.strip }.reject { |elem| elem.empty? }
      not_ip_addr_list = kafka_addr_array.select { |addr| !(addr =~ Resolv::IPv4::Regex) }
      if not_ip_addr_list.size > 0
        # if some of the values are not valid IP addresses, print an error and exit
        if not_ip_addr_list.size == 1
          print "ERROR; input kafka IP address #{not_ip_addr_list} is not a valid IP address\n"
          exit 2
        else
          print "ERROR; input kafka IP addresses #{not_ip_addr_list} are not valid IP addresses\n"
          exit 2
        end
      elsif !VALID_CLUSTER_SIZES.include?(kafka_addr_array.size)
        print "ERROR; only a Kafka cluster with an odd number of elements between 3 and 7 is\n"
        print "       supported; the defined Kafka cluster (#{kafka_addr_array})\n"
        print "       contains #{kafka_addr_array.size} elements\n"
        exit 5
      end
      deployment_addr_hash['kafka'] = kafka_addr_array
    end
    # for a multi-node Kafka cluster, we **must** have a zookeeper cluster that contains
    # three nodes (any other topology is not supported, so an error is thrown)
    zookeeper_addr_array = options[:zookeeper_list].split(',').map { |elem| elem.strip }.reject { |elem| elem.empty? }
    not_ip_addr_list = zookeeper_addr_array.select { |addr| !(addr =~ Resolv::IPv4::Regex) }
    if not_ip_addr_list.size > 0
      # if some of the values are not valid IP addresses, print an error and exit
      if not_ip_addr_list.size == 1
        print "ERROR; input zookeeper IP address #{not_ip_addr_list} is not a valid IP address\n"
        exit 2
      else
        print "ERROR; input zookeeper IP addresses #{not_ip_addr_list} are not valid IP addresses\n"
        exit 2
      end
    elsif zookeeper_addr_array.size != 3
      print "ERROR; only a zookeeper cluster with three elements is supported for multi-node\n"
      print "       kafka deployments; requested cluster #{zookeeper_addr_array}\n"
      print "       contains #{zookeeper_addr_array.size} elements\n"
      exit 5
    end
    # if we're here, then we're performing a multi-node kafka deployment with a three-node
    # zookeeper cluster; we need to make sure that the machines we're deploying kafka and zookeeper
    # to are not the same machines (the zookeeper cluster must be on a separate set of machines
    # from the kafka cluster)
    same_addr_list = zookeeper_addr_array & kafka_addr_array
    if same_addr_list.size > 0
      print "ERROR; the zookeeper cluster and kafka cluster cannot be deployed to the same\n"
      print "       machines; requested clusters overlap for the machines at #{same_addr_list}\n"
      exit 7
    end
    deployment_addr_hash['zookeeper'] = zookeeper_addr_array
  end
elsif ip_required
  # if we're here, then we're not provisioning but an IP is required for the command;
  # obtain the IP address (or addresses) needed from the value(s) passed in using the
  # `--node-list` flag
  if !options[:node_list]
    print "ERROR; IP address must be supplied (using the `--node-list` flag) for this vagrant command\n"
    exit 1
  end
  node_addr_array = options[:node_list].split(',').map { |elem| elem.strip }.reject { |elem| elem.empty? }
  not_ip_addr_list = node_addr_array.select { |addr| !(addr =~ Resolv::IPv4::Regex) }
  if not_ip_addr_list.size > 0
    # if some of the values are not valid IP addresses, print an error and exit
    if not_ip_addr_list.size == 1
      print "ERROR; input node IP address #{not_ip_addr_list} is not a valid IP address\n"
      exit 2
    else
      print "ERROR; input node IP addresses #{not_ip_addr_list} are not valid IP addresses\n"
      exit 2
    end
  elsif single_ip_command && node_addr_array.size > 1
    print "ERROR; Only a single IP address can be supplied (using the `--node-list` flag) for this command\n"
    exit 1    
  end
  deployment_addr_hash['zookeeper'] = node_addr_array
end

# if a yum repository address was passed in, check and make sure it's a valid
# IPv4 address
if options[:yum_repo_addr] && !(options[:yum_repo_addr] =~ Resolv::IPv4::Regex)
  print "ERROR; input yum repository address '#{options[:yum_repo_addr]}' is not a valid IP address\n"
  exit 6
end

# All Vagrant configuration is done below. The "2" in Vagrant.configure
# configures the configuration version (we support older styles for
# backwards compatibility). Please don't change it unless you know what
# you're doing.
if deployment_addr_hash['zookeeper']
  Vagrant.configure("2") do |config|
    proxy = ENV['http_proxy'] || ""
    no_proxy = ENV['no_proxy'] || ""
    proxy_username = ENV['proxy_username'] || ""
    proxy_password = ENV['proxy_password'] || ""
    if Vagrant.has_plugin?("vagrant-proxyconf")
      if $proxy
        config.proxy.http               = $proxy
        config.vm.box_download_insecure = true
        config.vm.box_check_update      = false
      end
      if $no_proxy
        config.proxy.no_proxy           = $no_proxy
      end
      if $proxy_username
        config.proxy.proxy_username     = $proxy_username
      end
      if $proxy_password
        config.proxy.proxy_password     = $proxy_password
      end
    end

    # Every Vagrant development environment requires a box. You can search for
    # boxes at https://atlas.hashicorp.com/search.
    config.vm.box = "centos/7"
    config.vm.box_check_update = false

    # loop through all of the addresses that were passed in on the command-line
    # and, if we're creating VMs, create a VM for each machine; if we're just
    # provisioning the VMs using an ansible playbook, then wait until the last VM
    # in the loop and trigger the playbook runs for all of the nodes simultaneously
    # using the `site.yml` playbook
    all_addr_array = deployment_addr_hash['zookeeper'] + (deployment_addr_hash['kafka'] || [])
    all_addr_array.each do |machine_addr|
      config.vm.define machine_addr do |machine|
        # setup a private network for this machine
        machine.vm.network "private_network", ip: machine_addr
        # if it's the last node in the list if input addresses, then provision
        # all of the nodes sumultaneously (if the `--no-provision` flag was not
        # set, of course)
        if machine_addr == all_addr_array[-1]
          # if the all_addr_array contains a single element, then deploy both
          # zookeeper and kafka to the same node; otherwise deploy zookeeper to
          # the nodes in the zookeeper list and kafka to the nodes in the
          # kafka list (in that order)
          if all_addr_array.size == 1
            # deploying a single kafka node; in this case set the deployment
            # type to 'zookeeper' (so that the right address array will
            # be pulled out of the deployment_addr_hash, below)
            deployment_types = ['zookeeper']
          elsif !deployment_addr_hash['kafka']
            # if a kafka cluster was not defined, then we're only deploying
            # a zookeeper cluster
            deployment_types = ['zookeeper']
          else
            # deploying a multi-node zookeeper and kafka cluster
            deployment_types = ['zookeeper', 'kafka']
          end
          # loop over our deployment types and provision each in turn
          deployment_types.each do |deployment_type|
            # from the `deployment_addr_hash`, extract the hosts we'll be deploying
            # to based on the `deployment_type` value
            deployment_addr_array = deployment_addr_hash[deployment_type]
            # if it's not a single node deployment and the current deployment_type
            # is zookeeper and the `--kafka-only` flag was set, skip provisioning
            # for these zookeeper nodes
            next if deployment_addr_array.size > 1 && options[:kafka_only] && deployment_type == 'zookeeper'
            # if the array size is one, then we're deploying to a single kafka node;
            # in that case we deploy both kafka and zookeeper to the same node
            if deployment_addr_array.size == 1
              # since we're deploying zookeeper and kafka on the same node,
              # we should set our deployment tags to deploy everything to
              # this single node
              deployment_tags = ['zookeeper', 'kafka']
            else
              # else, we'll be deploying zookeeper, then kafka, on separate
              # sets of nodes, so tag this deployment according to the type
              # of node we're deploying this time through the loop
              deployment_tags = deployment_type
            end
            # now that we know which nodes we're provisioning and what to tags
            # we should be using for those nodes, use the playbook in the `site.yml'
            # file to provision them
            machine.vm.provision "ansible" do |ansible|
              # set the limit to 'all' in order to provision all of machines on the
              # list in a single playbook run
              ansible.limit = "all"
              ansible.playbook = "site.yml"
              ansible.tags = deployment_tags
              ansible.extra_vars = {
                proxy_env: {
                  http_proxy: proxy,
                  no_proxy: no_proxy,
                  proxy_username: proxy_username,
                  proxy_password: proxy_password
                },
                kafka_iface: "eth1",
                kafka_distro: options[:kafka_distro],
                yum_repo_addr: options[:yum_repo_addr],
                local_kafka_package_path: options[:local_kafka_path],
                host_inventory: deployment_addr_array
              }
              # if a kafka log directory was set, then set an extra variable
              # containing the named directory
              if options[:kafka_log_dir]
                ansible.extra_vars[:kafka_log_dir] = options[:kafka_log_dir]
              end
              # if it's an Apache Kafka deployment, then set the `kafka_url` and
              # `kafka_path` extra variables, if values for these parameters were
              # included on the command-line
              if ansible.extra_vars[:kafka_distro] == "apache"
                # if defined, set the 'extra_vars[:kafka_url]' value to the value that was passed in on
                # the command-line (eg. "https://10.0.2.2/dist/kafka/0.10.1.0/kafka_2.11-0.10.1.0.tgz")
                if options[:kafka_url]
                  ansible.extra_vars[:kafka_url] = options[:kafka_url]
                end
                # if defined, set the 'extra_vars[:kafka_dir]' value to the value that was passed in on
                # the command-line (eg. "/opt/kafka")
                if options[:kafka_path]
                  ansible.extra_vars[:kafka_dir] = options[:kafka_path]
                end
              end
              # if more than one zookeeper node was passed in, then pass the values
              # in that hash value through as an extra variable (for use in configuring
              # the zookeeper cluster and/or kafka cluster)
              if deployment_addr_hash['zookeeper'].size > 1
                ansible.extra_vars[:zookeeper_nodes] = deployment_addr_hash['zookeeper']
              end
            end     # end `machine.vm.provision "ansible" do |ansible|`
          end     # end `deployment_tags.each do |deployment_type|`
        end     # end `if kafka_addr == kafka_addr_array[-1]`
      end     # end `config.vm.define machine_addr do |machine|`
    end     # end `all_addr_array.each do |machine_addr|`
  end     # end `Vagrant.configure ("2") do |config|`
end     # end `if kafka_addr_array`
