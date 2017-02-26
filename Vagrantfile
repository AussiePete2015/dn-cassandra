# (c) 2017 DataNexus Inc.  All Rights Reserved
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
no_ip_commands = ['version', 'global-status', '--help', '-h']

optparse = OptionParser.new do |opts|
  opts.banner    = "Usage: #{opts.program_name} [options]"
  opts.separator "Options"

  options[:cassandra_addr] = nil
  opts.on( '-c', '--cassandra-addr IP_ADDR', 'IP address of the Cassandra server' ) do |cassandra_addr|
    # while parsing, trim an '=' prefix character off the front of the string if it exists
    # (would occur if the value was passed using an option flag like '-c=192.168.1.1')
    options[:cassandra_addr] = cassandra_addr.gsub(/^=/,'')
  end

  options[:cassandra_home_dir] = nil
  opts.on( '-o', '--home PATH', 'Path where the distribution should be installed' ) do |cassandra_home_dir|
    # while parsing, trim an '=' prefix character off the front of the string if it exists
    # (would occur if the value was passed using an option flag like '-h=/opt/apache-cassandra')
    options[:cassandra_home_dir] = cassandra_home_dir.gsub(/^=/,'')
  end

  options[:cassandra_data_dir] = nil
  opts.on( '-d', '--data PATH', 'Path where Cassandra will store its data' ) do |cassandra_data_dir|
    # while parsing, trim an '=' prefix character off the front of the string if it exists
    # (would occur if the value was passed using an option flag like '-d=/data')
    options[:cassandra_home_dir] = cassandra_home_dir.gsub(/^=/,'')
  end

  options[:cassandra_url] = nil
  opts.on( '-u', '--url URL', 'URL the distribution should be downloaded from' ) do |cassandra_url|
    # while parsing, trim an '=' prefix character off the front of the string if it exists
    # (would occur if the value was passed using an option flag like '-u=http://localhost/tmp.tgz')
    options[:cassandra_url] = cassandra_url.gsub(/^=/,'')
  end

  options[:cassandra_dist_url] = nil
  opts.on( '-i', '--dist-url URL', 'URL of the distribution site including directory path where the binary can be downloaded from' ) do |cassandra_dist_url|
    # while parsing, trim an '=' prefix character off the front of the string if it exists
    # (would occur if the value was passed using an option flag like '-i=http://localhost:8080/dist/path')
    options[:cassandra_dist_url] = cassandra_dist_url.gsub(/^=/,'')
  end

  options[:cassandra_bin_name] = nil
  opts.on( '-b', '--binary-name NAME', 'Name of the binary to download' ) do |cassandra_bin_name|
    # while parsing, trim an '=' prefix character off the front of the string if it exists
    # (would occur if the value was passed using an option flag like '-b=apache-cassandra.tar.gz')
    options[:cassandra_bin_name] = cassandra_bin_name.gsub(/^=/,'')
  end

  options[:cassandra_version] = nil
  opts.on( '-v', '--version VERSION', 'Cassandra version to install' ) do |cassandra_version|
    # while parsing, trim an '=' prefix character off the front of the string if it exists
    # (would occur if the value was passed using an option flag like '-v=3.10')
    options[:cassandra_version] = cassandra_version.gsub(/^=/,'')
  end

  options[:cassandra_cluster_name] = nil
  opts.on( '-n', '--cluster-name NAME', 'Name of the Cassandra cluster the node will be part of' ) do |cassandra_cluster_name|
    # while parsing, trim an '=' prefix character off the front of the string if it exists
    # (would occur if the value was passed using an option flag like '-n=Test Cluster')
    options[:cassandra_cluster_name] = cassandra_cluster_name.gsub(/^=/,'')
  end

  options[:cassandra_seed_nodes] = nil
  opts.on( '-s', '--seed-nodes NODES', 'Comma separated string containing the seed nodes in the cluster' ) do |cassandra_seed_nodes|
    # while parsing, trim an '=' prefix character off the front of the string if it exists
    # (would occur if the value was passed using an option flag like '-s=127.0.01')
    options[:cassandra_seed_nodes] = cassandra_seed_nodes.gsub(/^=/,'')
  end

  options[:cassandra_trickle_fsync] = nil
  opts.on( '-t', '--trickle-fsync', 'Sepcifies whether to enable tickle-fsync on the node' ) do |cassandra_trickle_fsync|
    options[:cassandra_trickle_fsync] = true
  end

  options[:cassandra_listen_comms_method] = nil
  opts.on( '-l', '--listen-method METHOD', 'Set the method for the node to listen on; either \"address: IP_ADDRES\" or \"interface: ADAPTOR\"' ) do |cassandra_listen_comms_method|
    # while parsing, trim an '=' prefix character off the front of the string if it exists
    # (would occur if the value was passed using an option flag like '-l=address: 127.0.0.1')
    options[:cassandra_listen_comms_method] = cassandra_listen_comms_method.gsub(/^=/,'')
  end

  options[:cassandra_rpc_comms_method] = nil
  opts.on( '-r', '--rpc-method METHOD', 'Set the RPC method for the node to listen on; either \"address: IP_ADDRES\" or \"interface: ADAPTOR\"' ) do |cassandra_rpc_comms_method|
    # while parsing, trim an '=' prefix character off the front of the string if it exists
    # (would occur if the value was passed using an option flag like '-l=address: 127.0.0.1')
    options[:cassandra_rpc_comms_method] = cassandra_rpc_comms_method.gsub(/^=/,'')
  end

  opts.on_tail( '-h', '--help', 'Display this screen' ) do
    print opts
    exit
  end

end

begin
  optparse.order_recognized!(ARGV)
rescue SystemExit
  exit
rescue Exception => e
  print "ERROR: could not parse command (#{e.message})\n"
  print optparse
  exit 1
end

# check remaining arguments to see if the command requires
# an IP address (or not)
ip_required = (ARGV & no_ip_commands).empty?

if ip_required && !options[:cassandra_addr]
  print "ERROR; server IP address must be supplied for vagrant commands\n"
  print optparse
  exit 1
elsif options[:cassandra_addr] && !(options[:cassandra_addr] =~ Resolv::IPv4::Regex)
  print "ERROR; input server IP address '#{options[:cassandra_addr]}' is not a valid IP address"
  exit 2
end

if options[:cassandra_url] && !(options[:cassandra_url] =~ URI::regexp)
  print "ERROR; input Cassandra URL '#{options[:cassandra_url]}' is not a valid URL\n"
  exit 3
end

# All Vagrant configuration is done below. The "2" in Vagrant.configure
# configures the configuration version (we support older styles for
# backwards compatibility). Please don't change it unless you know what
# you're doing.
proxy = ENV['http_proxy'] || ""
no_proxy = ENV['no_proxy'] || ""
proxy_username = ENV['proxy_username'] || ""
proxy_password = ENV['proxy_password'] || ""
Vagrant.configure("2") do |config|
  if Vagrant.has_plugin?("vagrant-proxyconf")
    if $proxy
      config.proxy.http               = $proxy
      config.proxy.no_proxy           = "localhost,127.0.0.1"
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
  config.vm.network "private_network", ip: "#{options[:cassandra_addr]}"

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

  config.vm.provider "virtualbox" do |vb|
    # Customize the amount of memory on the VM:
    vb.memory = "8192"
  end

  # Define a Vagrant Push strategy for pushing to Atlas. Other push strategies
  # such as FTP and Heroku are also available. See the documentation at
  # https://docs.vagrantup.com/v2/push/atlas.html for more information.
  # config.push.define "atlas" do |push|
  #   push.app = "YOUR_ATLAS_USERNAME/YOUR_APPLICATION_NAME"
  # end

  config.vm.define "#{options[:cassandra_addr]}"
  cassandra_addr_array = "#{options[:cassandra_addr]}".split(/,\w*/)

  config.vm.provision "ansible" do |ansible|
    ansible.playbook = "site.yml"
    ansible.extra_vars = {
      proxy_env: {
        http_proxy: proxy,
        no_proxy: no_proxy,
        proxy_username: proxy_username,
        proxy_password: proxy_password
      },
      host_inventory: cassandra_addr_array,
      cassandra_jvm_heaps_size: "2G",
      cassandra_swap_mode: "on",
      cassandra_trickle_fsync: false
    }

    # if defined, set the 'extra_vars[:cassandra_url]' value to the value that was passed in on
    # the command-line (eg. "https://10.0.2.2/apache-cassandra-3.10-bin.tar.gz")
    if options[:cassandra_url]
      # Obtain the binary name and pass it in as well so the package can be removed after it is installed
      host_url, match, bin_name = options[:cassandra_url].rpartition('/')
      ansible.extra_vars[:cassandra_bin_url] = "#{options[:cassandra_url]}"
      ansible.extra_vars[:cassandra_bin_name] = "#{bin_name}"
      options[:cassandra_bin_name] = bin_name
    end

    # if defined, set the 'extra_vars[:cassandra_dist_url]' value to the value that was passed in on
    # the command-line (eg. "http://localhost:8080/dist/path/")
    if options[:cassandra_dist_url]
      ansible.extra_vars[:cassandra_dist_url] = "#{options[:cassandra_dist_url]}"
    end

    # if defined, set the 'extra_vars[:cassandra_bin_name]' value to the value that was passed in on
    # the command-line (eg. "apache-cassandra.tar.gz")
    if options[:cassandra_bin_name]
      ansible.extra_vars[:cassandra_bin_name] = "#{options[:cassandra_bin_name]}"
    end

    # if defined, set the 'extra_vars[:cassandra_version]' value to the value that was passed in on
    # the command-line (eg. "3.10")
    if options[:cassandra_version]
      ansible.extra_vars[:cassandra_version] = "#{options[:cassandra_version]}"
    end

    # if defined, set the 'extra_vars[:cassandra_home_dir]' value to the value that was passed in on
    # the command-line (eg. "/opt/apache-cassandra")
    if options[:cassandra_home_dir]
      ansible.extra_vars[:cassandra_home_dir] = "#{options[:cassandra_home_dir]}"
    end

    # if defined, set the 'extra_vars[:cassandra_data_dir]' value to the value that was passed in on
    # the command-line (eg. "/opt/apache-cassandra")
    if options[:cassandra_data_dir]
      ansible.extra_vars[:cassandra_data_dir] = "#{options[:cassandra_data_dir]}"
    end

    # if defined, set the 'extra_vars[:cassandra_cluster_name]' value to the value that was passed in on
    # the command-line (eg. "Test Cluster")
    if options[:cassandra_cluster_name]
      ansible.extra_vars[:cassandra_cluster_name] = "#{options[:cassandra_cluster_name]}"
    end

    # if defined, set the 'extra_vars[:cassandra_seed_nodes]' value to the value that was passed in on
    # the command-line (eg. "127.0.0.1")
    if options[:cassandra_seed_nodes]
      ansible.extra_vars[:cassandra_seed_nodes] = "#{options[:cassandra_seed_nodes]}"
    end

    # if defined, set the 'extra_vars[:cassandra_trickle_fsync]' to true
    if options[:cassandra_trickle_fsync]
      ansible.extra_vars[:cassandra_trickle_fsync] = true
    end

    # if defined, set the 'extra_vars[:cassandra_listen_comms_method]' value to the value that was passed in on
    # the command-line (eg. "address: 127.0.0.1")
    if options[:cassandra_listen_comms_method]
      ansible.extra_vars[:cassandra_listen_comms_method] = "#{options[:cassandra_listen_comms_method]}"
    end

    # if defined, set the 'extra_vars[:cassandra_rpc_comms_method]' value to the value that was passed in on
    # the command-line (eg. "address: 127.0.0.1")
    if options[:cassandra_rpc_comms_method]
      ansible.extra_vars[:cassandra_rpc_comms_method] = "#{options[:cassandra_rpc_comms_method]}"
    end
  end

end
