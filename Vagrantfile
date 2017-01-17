# -*- mode: ruby -*-
# vi: set ft=ruby :

require 'optparse'
require 'resolv'

options = {}
valid_kafka_distros = ['confluent', 'apache']

optparse = OptionParser.new do |opts|
  opts.banner    = "Usage: #{opts.program_name} [options]"
  opts.separator "Options"

  options[:kafka_addr] = nil
  opts.on( '-k', '--kafka-addr IP_ADDR', 'IP_ADDR of the kafka server' ) do |kafka_addr|
    # while parsing, trim an '=' prefix character off the front of the string if it exists
    # (would occur if the value was passed using an option flag like '-k=192.168.1.1')
    options[:kafka_addr] = kafka_addr.gsub(/^=/,'')
  end

  options[:kafka_distro] = nil
  opts.on( '-d', '--distro DISTRO_NAME', 'Name of the kafka distribution' ) do |kafka_distro|
    # while parsing, trim an '=' prefix character off the front of the string if it exists
    # (would occur if the value was passed using an option flag like '-k=192.168.1.1')
    options[:kafka_distro] = kafka_distro.gsub(/^=/,'')
  end

  opts.on_tail( '-h', '--help', 'Display this screen' ) do
    print opts
    exit
  end

end

# default to using the confluent distro if no distro was specified
options[:kafka_distro] = "confluent"
begin
  optparse.parse!
rescue SystemExit
  ;
rescue Exception => e
  print "ERROR: could not parse command (#{e.message})\n"
  print optparse
  exit 1
end

if !options[:kafka_addr]
  print "ERROR; kafka IP address must be supplied for vagrant commands\n"
  print optparse
  exit 1
elsif !(options[:kafka_addr] =~ Resolv::IPv4::Regex)
  print "ERROR; input kafka IP address '#{options[:kafka_addr]}' is not a valid IP address"
  exit 2
end

if !(valid_kafka_distros.include?(options[:kafka_distro]))
    print "ERROR; Unrecognized kafka_distro value specified; valid values are #{valid_kafka_distros}\n"
    exit 3
end

# All Vagrant configuration is done below. The "2" in Vagrant.configure
# configures the configuration version (we support older styles for
# backwards compatibility). Please don't change it unless you know what
# you're doing.
proxy = ENV['http_proxy'] || ""
no_proxy = ENV['no_proxy'] || ""
Vagrant.configure("2") do |config|
  if Vagrant.has_plugin?("vagrant-proxyconf")
    if $proxy
      config.proxy.http               = $proxy
      config.vm.box_download_insecure = true
      config.vm.box_check_update      = false
    end
    if $no_proxy
      config.proxy.no_proxy           = $no_proxy
    end
  end
  # The most common configuration options are documented and commented below.
  # For a complete reference, please see the online documentation at
  # https://docs.vagrantup.com.

  # Every Vagrant development environment requires a box. You can search for
  # boxes at https://atlas.hashicorp.com/search.
  config.vm.box = "centos/7"

  # Disable automatic box update checking. If you disable this, then
  # boxes will only be checked for updates when the user runs
  # `vagrant box outdated`. This is not recommended.
  # config.vm.box_check_update = false

  # Create a forwarded port mapping which allows access to a specific port
  # within the machine from a port on the host machine. In the example below,
  # accessing "localhost:8080" will access port 80 on the guest machine.
  # config.vm.network "forwarded_port", guest: 80, host: 8080

  # Create a private network, which allows host-only access to the machine
  # using a specific IP.
  config.vm.network "private_network", ip: "#{options[:kafka_addr]}"

  # Create a public network, which generally matched to bridged network.
  # Bridged networks make the machine appear as another physical device on
  # your network.
  # config.vm.network "public_network"

  # Share an additional folder to the guest VM. The first argument is
  # the path on the host to the actual folder. The second argument is
  # the path on the guest to mount the folder. And the optional third
  # argument is a set of non-required options.
  # config.vm.synced_folder "../data", "/vagrant_data"
  config.vm.synced_folder ".", "/vagrant"

  # Provider-specific configuration so you can fine-tune various
  # backing providers for Vagrant. These expose provider-specific options.
  # Example for VirtualBox:
  #
  # config.vm.provider "virtualbox" do |vb|
  #   # Display the VirtualBox GUI when booting the machine
  #   vb.gui = true
  #
  #   # Customize the amount of memory on the VM:
  #   vb.memory = "1024"
  # end
  #
  # View the documentation for the provider you are using for more
  # information on available options.

  # Define a Vagrant Push strategy for pushing to Atlas. Other push strategies
  # such as FTP and Heroku are also available. See the documentation at
  # https://docs.vagrantup.com/v2/push/atlas.html for more information.
  # config.push.define "atlas" do |push|
  #   push.app = "YOUR_ATLAS_USERNAME/YOUR_APPLICATION_NAME"
  # end

  config.vm.define "#{options[:kafka_addr]}"
  kafka_addr_array = "#{options[:kafka_addr]}".split(/,\w*/)
  
  config.vm.provision "ansible" do |ansible|
    ansible.playbook = "site.yml"
    ansible.extra_vars = {
      proxy_env: { http_proxy: proxy, no_proxy: no_proxy },
      # kafka_url: "https://www-us.apache.org/dist/kafka/0.10.1.0/kafka_2.11-0.10.1.0.tgz",
      kafka_url: "https://10.0.2.2/dist/kafka/0.10.1.0/kafka_2.11-0.10.1.0.tgz",
      kafka_addr: "#{options[:kafka_addr]}",
      kafka_distro: "#{options[:kafka_distro]}",
      kafka_dir: "/opt/kafka",
      kafka_topics: ["metrics", "logs"],
      confluent_version: "3.1",
      kafka_package_list: ["java-1.8.0-openjdk", "java-1.8.0-openjdk-devel"],
      host_inventory: kafka_addr_array
    }
  end

end
