# dn-cassandra
Playbooks/Roles used to deploy Appache Cassandra

# Installation
To install Cassandra using the `site.yml` playbook in this repository, first clone the contents of this repository to a local directory using a command like the following:

```bash
$ git clone --recursive https://github.com/Datanexus/dn-cassandra
```

That command will pull down the repository and it's submodules (currently the only dependency embedded as a submodule is the dependency on the `https://github.com/Datanexus/common-roles` repository).

# Using this role to deploy Cassandra
The `site.yml` file at the top-level of this repository pulls in a set of default values for the parameters that are needed to deploy an instance of the Apache Cassandra distribution to a node from the [vars/cassandra.yml](vars/cassandra.yml) and [defaults/main.yml](defaults/main.yml) files.

This default configuration defines default values for all of the parameters needed to deploy an instance of Cassandra to a node, including defining reasonable defaults for the URL that the Cassandra distribution should be downloaded from, the directory that the gzipped tarfile containing the Cassandra distribution should be unpacked into, and the packages that must be installed on the node for Cassandra to run.  To deploy Cassandra to a node the IP address "192.168.34.12" using the role in this repository, one would simply run a command that looks like this:

```bash
$ export CASSANDRA_ADDR="192.168.34.12"
$ ansible-playbook -i "${CASSANDRA_ADDR}," -e "{ host_inventory: ['${CASSANDRA_ADDR}']}" site.yml
```

This will download the distribution from the main Apache Cassandra download site onto the host with an IP address of 192.168.34.12, unpack the gzipped tarfile that it downlaoded form that site into the `/opt/apache-cassandra` directory on that host, and install/configure the Cassandra server locally on that host.

If installing this playbook on multiple nodes, it may help to download the gzipped tarfile containing the Cassandra distribution from that site once. The distribution can then be placed on an internal webserver and downloaded from the nodes that are being provisioned as Cassandra nodes. This can be accomplished by including a definition for the `cassandra_url` parameter in the extra variables that are passed into the Ansible playbook.  In this case, the above command will look similar to the following command:

```bash
$ export CASSANDRA_URL="https://10.0.2.2/apache-cassandra-3.0.11-bin.tar.gz
$ export CASSANDRA_ADDR="192.168.34.12"
$ ansible-playbook -i "${CASSANDRA_ADDR}," -e "{ host_inventory: ['${CASANDRA_ADDR}'] \
    cassandra_url: '${CASSANDRA_URL}'}" site.yml
```

The above commands assume that the file is available directly from a web-server that is accessible by the Cassandra node being built via the IP address 10.0.2.2.

The path that the Cassandra distribution is unpacked into can be overriden on the command-line in a similar fashion (using the `cassandra_path` variable) if the default values shown in the `vars/cassandra.yml` file (above) proves to be problematic.

# Assumptions
It is assumed that this playbook will be run on a recent (systemd-based) version of RHEL or CentOS (RHEL-7.x or CentOS-7.x, for example); no support is provided for other distributions (and the `site.xml` playbook will not run successfully).  The examples shown above also assume that some mechanism has been used to provide access to the Cassandra host from the Ansible host that the ansible-playbook is being run on. If not, then additional arguments might be required to authenticate with that host from the Ansible host that are not shown in the example `ansible-playbook` commands shown above.

In order to execute the vagrant commands locally, it is assumed that both vagrant and VirtualBox are installed locally.

# Deployment via vagrant
A Vagrantfile is included in this repository that can be used to deploy Cassandra to a VM using the `vagrant` command.  From the top-level directory of this repository a command like the following will (by default) deploy Cassandra to a CentOS 7 virtual machine running under VirtualBox:

```bash
$ vagrant -c="192.168.34.12" up
```

Note that the `-c` (or the corresponding `--cassandra-addr`) flag must be used to pass an IP address into the Vagrantfile. This IP address will be used as the IP address of the Cassandra server that is created by the Vagrant command shown above.

## Additional vagrant deployment options
While the `vagrant up` command that is shown above can be used to easily deploy Cassandra to a node, the Vagrantfile included in this distribution also supports separating out the creation of the virtual machine from the provisioning of that virtual machine using the Ansible playbook contained in this repository's `site.yml` file. To create a virtual machine without provisioning it, simply run a command similar to the following:

```bash
$ vagrant -c="192.168.34.12" up --no-provision
```

This will create a virtual machine with the appropriate IP address ("192.168.34.12"), but will skip the process of provisioning that VM with an instance of Cassandra.  To provision that machine with a Cassandra instance, simply run the following command:

```bash
$ vagrant -c="192.168.34.12" provision
```

That command will attach to the named instance (the VM at "192.168.34.12") and run the playbook in this repository's `site.yml` file on that node resulting in the deployment of an instance the Apache Cassandra distribution to the node.

It should also be noted here that while the commands shown above will install Cassandra with a reasonable default configuration from a standard location, there are two additional command-line parameters that can be used to override the default values that are embedded in the `vars/main.yml` file that is included as part of this repository:  the `-u` (or corresponding `--url`) flag and the `-p` (or corresponding `--path`) flag.  The `-u` flag can be used to override the default URL that is used to download the Apache Cassandra distribution, while the `-p` flag can be used to override the default path (`/opt/apache-cassandra`) that that the Apache Cassandra gzipped tarfile is unpacked into during the provisioning process.

As an example of how these options might be used, the following command will download the gzipped tarfile containing the Apache Cassandra distribution from a local web server, rather than downloading it from the main Apache distribution site and unpack the downladed gzipped tarfile into the `/opt/apache-cassandra` directory when provisioning the VM with an IP address of `192.168.34.12` with an instance of the Apache Cassandra distribution:

```bash
$ vagrant -c="192.168.34.12" -h="/opt/apache-cassandra" -u="https://10.0.2.2/apache-cassandra-3.10-bin.tar.gz" provision
```

This option could prove to be quite useful in situations were we are deploying the distribution from a datacenter environment where access to the internet may be restricted, or even unavailable.