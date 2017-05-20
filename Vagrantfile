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
# vagrant commands that include these commands can be run without specifying
# any IP addresses
no_ip_commands = ['version', 'global-status', '--help', '-h']
# vagrant commands that only work for a single IP address
single_ip_commands = ['status', 'ssh']
# vagrant command arguments that indicate we are provisioning a cluster (if multiple
# nodes are supplied via the `--cassandra-list` flag)
provisioning_command_args = ['up', 'provision']
not_provisioning_flag = ['--no-provision']

optparse = OptionParser.new do |opts|
  opts.banner    = "Usage: #{opts.program_name} [options]"
  opts.separator "Options"

  options[:cassandra_list] = nil
  opts.on( '-c', '--cassandra-list A1,A2[,...]', 'Cassandra address list' ) do |cassandra_list|
    # while parsing, trim an '=' prefix character off the front of the string if it exists
    # (would occur if the value was passed using an option flag like '-k=192.168.1.1')
    options[:cassandra_list] = cassandra_list.gsub(/^=/,'')
  end

  options[:cassandra_dir] = nil
  opts.on( '-o', '--home PATH', 'Path where the distribution should be installed' ) do |cassandra_dir|
    # while parsing, trim an '=' prefix character off the front of the string if it exists
    # (would occur if the value was passed using an option flag like '-h=/opt/apache-cassandra')
    options[:cassandra_dir] = cassandra_dir.gsub(/^=/,'')
  end

  options[:cassandra_data_dir] = nil
  opts.on( '-d', '--data PATH', 'Path where Cassandra will store its data' ) do |cassandra_data_dir|
    # while parsing, trim an '=' prefix character off the front of the string if it exists
    # (would occur if the value was passed using an option flag like '-d=/data')
    options[:cassandra_data_dir] = cassandra_data_dir.gsub(/^=/,'')
  end

  options[:cassandra_url] = nil
  opts.on( '-u', '--url URL', 'URL the distribution should be downloaded from' ) do |cassandra_url|
    # while parsing, trim an '=' prefix character off the front of the string if it exists
    # (would occur if the value was passed using an option flag like '-u=http://localhost/tmp.tgz')
    options[:cassandra_url] = cassandra_url.gsub(/^=/,'')
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
  opts.on( '-l', '--listen-method METHOD', 'Set the method for the node to listen on; either "address: IP_ADDRES" or "interface: ADAPTOR"' ) do |cassandra_listen_comms_method|
    # while parsing, trim an '=' prefix character off the front of the string if it exists
    # (would occur if the value was passed using an option flag like '-l=address: 127.0.0.1')
    options[:cassandra_listen_comms_method] = cassandra_listen_comms_method.gsub(/^=/,'')
  end

  options[:cassandra_rpc_comms_method] = nil
  opts.on( '-r', '--rpc-method METHOD', 'Set the RPC method for the node to listen on; either "address: IP_ADDRES" or "interface: ADAPTOR"' ) do |cassandra_rpc_comms_method|
    # while parsing, trim an '=' prefix character off the front of the string if it exists
    # (would occur if the value was passed using an option flag like '-l=address: 127.0.0.1')
    options[:cassandra_rpc_comms_method] = cassandra_rpc_comms_method.gsub(/^=/,'')
  end

  options[:yum_repo_url] = nil
  opts.on( '-y', '--yum-url URL', 'Local yum repository URL' ) do |yum_repo_url|
    # while parsing, trim an '=' prefix character off the front of the string if it exists
    # (would occur if the value was passed using an option flag like '-y=http://192.168.1.128/centos')
    options[:yum_repo_url] = yum_repo_url.gsub(/^=/,'')
  end

  options[:local_vars_file] = nil
  opts.on( '-f', '--local-vars-file FILE', 'Local variables file' ) do |local_vars_file|
    # while parsing, trim an '=' prefix character off the front of the string if it exists
    # (would occur if the value was passed using an option flag like '-f=/tmp/local-vars-file.yml')
    options[:local_vars_file] = local_vars_file.gsub(/^=/,'')
  end

  options[:reset_proxy_settings] = false
  opts.on( '-p', '--clear-proxy-settings', 'Clear existing proxy settings if no proxy is set' ) do |reset_proxy_settings|
    options[:reset_proxy_settings] = true
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
# check the remaining arguments to see if we're provisioning or not
provisioning_command = !((ARGV & provisioning_command_args).empty?) && (ARGV & not_provisioning_flag).empty?
# and to see if multiple IP addresses are supported (or not) for the
# command being invoked
single_ip_command = !((ARGV & single_ip_commands).empty?)

if options[:cassandra_url] && !(options[:cassandra_url] =~ URI::regexp)
  print "ERROR; input Cassandra URL '#{options[:cassandra_url]}' is not a valid URL\n"
  exit 3
end

# if a yum repository address was passed in, check and make sure it's a valid URL
if options[:yum_repo_url] && !(options[:yum_repo_url] =~ URI::regexp)
  print "ERROR; input yum repository URL '#{options[:yum_repo_url]}' is not a valid URL\n"
  exit 6
end

# if a local variables file was passed in, check and make sure it's a valid filename
if options[:local_vars_file] && !File.file?(options[:local_vars_file])
  print "ERROR; input local variables file '#{options[:local_vars_file]}' is not a local file\n"
  exit 3
end

# if we're provisioning, then the `--cassandra-list` flag must be provided and either contain
# a single node (for single-node deployments) or multiple nodes in a comma-separated list
# (for multi-node deployments) that define a valid cassandra cluster
cassandra_addr_array = []
cassandra_seed_array = []
cassandra_non_seed_array = []
if provisioning_command || ip_required
  if !options[:cassandra_list]
    print "ERROR; IP address must be supplied (using the `-c, --cassandra-list` flag) for this vagrant command\n"
    exit 1
  else
    cassandra_addr_array = options[:cassandra_list].split(',').map { |elem| elem.strip }.reject { |elem| elem.empty? }
    if cassandra_addr_array.size == 1
      if !(cassandra_addr_array[0] =~ Resolv::IPv4::Regex)
        print "ERROR; input Cassandra IP address #{cassandra_addr_array[0]} is not a valid IP address\n"
        exit 2
      end
    elsif !single_ip_command
      # check the input `cassandra_addr_array` to ensure that all of the values passed in are
      # legal IP addresses
      not_ip_addr_list = cassandra_addr_array.select { |addr| !(addr =~ Resolv::IPv4::Regex) }
      if not_ip_addr_list.size > 0
        # if some of the values are not valid IP addresses, print an error and exit
        if not_ip_addr_list.size == 1
          print "ERROR; input Cassandra IP address #{not_ip_addr_list} is not a valid IP address\n"
          exit 2
        else
          print "ERROR; input Cassandra IP addresses #{not_ip_addr_list} are not valid IP addresses\n"
          exit 2
        end
      end
      # if we're provisioning a cluster, then a list of seed nodes is needed
      if provisioning_command && cassandra_addr_array.size > 1 && !options[:cassandra_seed_nodes]
        print "ERROR; List of seed node addresses must be supplied (using the `-s, --seed-nodes` flag) for this vagrant command\n"
        exit 1
      elsif provisioning_command
        cassandra_seed_array = options[:cassandra_seed_nodes].split(',').map { |elem| elem.strip }.reject { |elem| elem.empty? }
        if cassandra_seed_array.size == 1
          if !(cassandra_seed_array[0] =~ Resolv::IPv4::Regex)
            print "ERROR; input Cassandra seed address #{cassandra_seed_array[0]} is not a valid IP address\n"
            exit 2
          end
        else
          # check the input `cassandra_seed_array` to ensure that all of the values passed in are
          # legal IP addresses
          not_ip_addr_list = cassandra_seed_array.select { |addr| !(addr =~ Resolv::IPv4::Regex) }
          if not_ip_addr_list.size > 0
            # if some of the values are not valid IP addresses, print an error and exit
            if not_ip_addr_list.size == 1
              print "ERROR; input Cassandra seed address #{not_ip_addr_list} is not a valid IP address\n"
              exit 2
            else
              print "ERROR; input Cassandra seed addresses #{not_ip_addr_list} are not valid IP addresses\n"
              exit 2
            end
          end
        end
        # make sure that the seed nodes that are passed in are part of the list of cassandra
        # node addresses that were passed in
        not_in_cassandra_array = cassandra_seed_array - cassandra_addr_array
        if not_in_cassandra_array.size == 1
          print "ERROR; input Cassandra seed address #{not_in_cassandra_array} is not in the list of Cassandra addresses\n"
          exit 2
        elsif not_in_cassandra_array.size > 1
          print "ERROR; input Cassandra seed addresses #{not_in_cassandra_array} are not in the list of Cassandra addresses\n"
          exit 2
        end
        cassandra_non_seed_array = cassandra_addr_array - cassandra_seed_array
      end
    end
  end
end

# if we get to here and a list of seed nodes wasn't provided, then we're
# performing a single-node deployment and all of the nodes are "non-seed"
# nodes for the purposes of the playbook that will deploy Cassandra to
# that node
if cassandra_non_seed_array.size == 0
  cassandra_non_seed_array = cassandra_addr_array
end

# All Vagrant configuration is done below. The "2" in Vagrant.configure
# configures the configuration version (we support older styles for
# backwards compatibility). Please don't change it unless you know what
# you're doing.
if cassandra_addr_array.size > 0
  Vagrant.configure("2") do |config|
    options[:proxy] = ENV['http_proxy'] || ""
    options[:no_proxy] = ENV['no_proxy'] || ""
    options[:proxy_username] = ENV['proxy_username'] || ""
    options[:proxy_password] = ENV['proxy_password'] || ""
    if Vagrant.has_plugin?("vagrant-proxyconf")
      if options[:proxy] != ""
        config.proxy.http               = options[:proxy]
        config.proxy.no_proxy           = "localhost,127.0.0.1"
        config.vm.box_download_insecure = true
        config.vm.box_check_update      = false
      end
      if options[:no_proxy] != ""
        config.proxy.no_proxy           = options[:no_proxy]
      end
      if options[:proxy_username] != ""
        config.proxy.proxy_username     = options[:proxy_username]
      end
      if options[:proxy_password] != ""
        config.proxy.proxy_password     = options[:proxy_password]
      end
    end
        # Every Vagrant development environment requires a box. You can search for
    # boxes at https://atlas.hashicorp.com/search.
    config.vm.box = "centos/7"
    config.vm.box_check_update = false

    # loop through all of the addresses in the `cassandra_addr_array` and, if we're
    # creating VMs, create a VM for each machine; if we're just provisioning the
    # VMs using an ansible playbook, then wait until the last VM in the loop and
    # trigger the playbook runs for all of the nodes simultaneously using the
    # `provision-cassandra.yml` playbook
    cassandra_addr_array.each do |machine_addr|
      config.vm.define machine_addr do |machine|
        # disable the default synced folder
        machine.vm.synced_folder ".", "/vagrant", disabled: true
        # Create a two private networks, which each allow host-only access to the machine
        # using a specific IP.
        machine.vm.network "private_network", ip: machine_addr
        split_addr = machine_addr.split('.')
        api_addr = (split_addr[0..1] + [(split_addr[2].to_i + 10).to_s] + [split_addr[3]]).join('.')
        machine.vm.network "private_network", ip: api_addr
        # set the memory for this instance
        machine.vm.provider "virtualbox" do |vb|
          # Customize the amount of memory on the VM:
          # vb.memory = "8192"
          vb.memory = "1536"
       end
        # if it's the last node in the list if input addresses, then provision
        # all of the nodes simultaneously (if the `--no-provision` flag was not
        # set, of course)
        if machine_addr == cassandra_addr_array[-1]
          machine.vm.provision "ansible" do |ansible|
            # now, use the playbook in the `provision-cassandra.yml' file to
            # provision our nodes with Cassandra (and configure them as a
            # cluster if there is more than one node); first, set the limit to
            # 'all' in order to provision all of machines on the list in a
            # single playbook run
            ansible.limit = "all"
            ansible.playbook = "provision-cassandra.yml"
            ansible.groups = {
              cassandra_seed: cassandra_seed_array,
              cassandra: cassandra_non_seed_array
            }
            # then set some extra variables
            ansible.extra_vars = {
              proxy_env: {
                http_proxy: options[:proxy],
                no_proxy: options[:no_proxy],
                proxy_username: options[:proxy_username],
                proxy_password: options[:proxy_password]
              },
              data_iface: "eth1",
              api_iface: "eth2",
              cassandra_jvm_heaps_size: "1G",
              cassandra_swap_mode: "on",
              start_cassandra: true,
              cassandra_trickle_fsync: false
            }

            # if a local yum repositiory  was set, then set an extra variable
            # containing the named repository
            if options[:yum_repo_url]
              ansible.extra_vars[:yum_repo_url] = options[:yum_repo_url]
            end

            # if the flag to reset the proxy settings was set, then set an extra variable
            # containing that value
            if options[:reset_proxy_settings]
              ansible.extra_vars[:reset_proxy_settings] = options[:reset_proxy_settings]
            end

            # if defined, set the 'extra_vars[:cassandra_url]' value to the value that was passed in on
            # the command-line (eg. "https://10.0.2.2/apache-cassandra-3.10-bin.tar.gz")
            if options[:cassandra_url]
              ansible.extra_vars[:cassandra_url] = options[:cassandra_url]
            end

            # if defined, set the 'extra_vars[:cassandra_version]' value to the value that was passed in on
            # the command-line (eg. "3.10")
            if options[:cassandra_version]
              ansible.extra_vars[:cassandra_version] = options[:cassandra_version]
            end

            # if defined, set the 'extra_vars[:cassandra_dir]' value to the value that was passed in on
            # the command-line (eg. "/opt/apache-cassandra")
            if options[:cassandra_dir]
              ansible.extra_vars[:cassandra_dir] = options[:cassandra_dir]
            end

            # if defined, set the 'extra_vars[:cassandra_data_dir]' value to the value that was passed in on
            # the command-line (eg. "/opt/apache-cassandra")
            if options[:cassandra_data_dir]
              ansible.extra_vars[:cassandra_data_dir] = options[:cassandra_data_dir]
            end

            # if defined, set the 'extra_vars[:cassandra_cluster_name]' value to the value that was passed in on
            # the command-line (eg. "Test Cluster")
            if options[:cassandra_cluster_name]
              ansible.extra_vars[:cassandra_cluster_name] = options[:cassandra_cluster_name]
            end

            # if defined, set the 'extra_vars[:local_vars_file]' value to the value that was passed in
            # on the command-line (eg. "/tmp/local-vars-file.yml")
            if options[:local_vars_file]
              ansible.extra_vars[:local_vars_file] = options[:local_vars_file]
            end

            # if defined, set the 'extra_vars[:cassandra_seed_nodes]' value to the value that was passed in on
            # the command-line (eg. "127.0.0.1")
            if cassandra_seed_array.size > 0
              ansible.extra_vars[:cassandra_seed_nodes] = cassandra_seed_array
            end

            # if defined, set the 'extra_vars[:cassandra_trickle_fsync]' to true
            if options[:cassandra_trickle_fsync]
              ansible.extra_vars[:cassandra_trickle_fsync] = true
            end

            # if defined, set the 'extra_vars[:cassandra_listen_comms_method]' value to the value that was passed in on
            # the command-line (eg. "address: 127.0.0.1")
            if options[:cassandra_listen_comms_method]
              ansible.extra_vars[:cassandra_listen_comms_method] = options[:cassandra_listen_comms_method]
            end

            # if defined, set the 'extra_vars[:cassandra_rpc_comms_method]' value to the value that was passed in on
            # the command-line (eg. "address: 127.0.0.1")
            if options[:cassandra_rpc_comms_method]
              ansible.extra_vars[:cassandra_rpc_comms_method] = options[:cassandra_rpc_comms_method]
            end
          end
        end
      end
    end
  end
end
