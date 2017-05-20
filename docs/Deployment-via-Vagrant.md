# Deployment via Vagrant
A [Vagrantfile](../Vagrantfile) is included in this repository that can be used to deploy Cassandra locally (to one or more VMs hosted under [VirtualBox](https://www.virtualbox.org/)) using [Vagrant](https://www.vagrantup.com/).  From the top-level directory of this repository a command like the following will (by default) deploy Cassandra to a single CentOS 7 virtual machine running under VirtualBox:

```bash
$ vagrant -c="192.168.34.70" up
```

Note that the `-c, --cassandra-list` flag must be used to pass an IP address (or a comma-separated list of IP addresses) into the [Vagrantfile](../Vagrantfile). In the example shown above, we are performing a single-node deployment of Cassandra, and that node will be configured as its own seed node (the default seed, if one is not specified, is the localhost or `127.0.0.1`).

When we are performing a multi-node deployment, then the seed nodes for the cluster must also be defined, as in this example:

```bash
vagrant -c="192.168.34.72,192.168.34.73,192.168.34.74,192.168.34.75,192.168.34.76,192.168.34.77" \
    -s="192.168.34.72,192.168.34.73,192.168.34.74" up
```

This command will create a six-node Cassandra cluster, where the first three nodes in the list of Cassandra IP addresses (passed in using the `-c, --cassandra-list` flag) are configured as seed nodes (the list of seed nodes is passed in using the `-s, --seed-nodes` flag). It should be noted here that if any of the IP addresses listed in the list of seed nodes does not also appear in the list of Cassandra IP addresses, an error will be thrown. Similarly, if there is more than one Cassandra IP address listed and a list of seed nodes is not provided, an error will be thrown by the `vagrant` command.

In terms of how it all works, the [Vagrantfile](../Vagrantfile) is written in such a way that the following sequence of events occurs when the `vagrant ... up` command shown above is run:

1. All of the virtual machines in the cluster (the addresses in the `-c, --cassandra-list`) are created
1. Cassandra is deployed to the seed nodes (the nodes with addresses in the `-s, --seed-nodes` list) using the Ansible playbook in the [provision-cassandra.yml](../provision-cassandra.yml) file in this repository; the list of seed nodes that was passed in using the `-s, --seed-nodes` flag are used to set the `seeds` configuration parameter in the `cassandra.yaml` file for each of these Cassandra instances
1. The `cassandra` service is started on all of the (seed) nodes that were just provisioned
1. Cassandra then is deployed to the non-seed nodes using the same Ansible playbook; as was the case in the previous provisioning pass, the list of seed nodes that was passed in using the `-s, --seed-nodes` flag are used to set the `seeds` configuration parameter in the `cassandra.yaml` file for each of these Cassandra instances
1. Finally, the `cassandra` service is started on all of the (non-seed) nodes that were just provisioned

Once the first provisioning pass in the [Vagrantfile](../Vagrantfile) is complete (step 2), the `cassandra` service starts automatically (step 3). At that time, we can run the `nodetool` command on one of our new seed nodes to prove that the initial cluster of three seed nodes is up and running:

```bash
[vagrant@localhost ~]$ /opt/apache-cassandra/bin/nodetool status
Datacenter: dc1
===============
Status=Up/Down
|/ State=Normal/Leaving/Joining/Moving
--  Address        Load       Tokens       Owns (effective)  Host ID                               Rack
UN  192.168.34.72  100.79 KiB  32           70.5%             592ac186-9c23-414b-b503-554bccc2a925  rack1
UN  192.168.34.73  99.62 KiB  32           65.0%             a8e54091-9f51-4b9b-b222-d89141144af0  rack1
UN  192.168.34.74  113.92 KiB  32           64.5%             dbf126b8-6fd1-4d37-a400-907ffdecf183  rack1

[vagrant@localhost ~]$
```

Once the second provisioning pass is complete (step 4) and the `cassandra` service has been started on the non-seed nodes (step 5), those nodes are added to the existing cluster. When those two steps are complete, we can run the same `nodestatus` command that we ran above (on any of the nodes that make up our cluster) to show the status of each of the machines in the complete cluster:

```bash
[vagrant@localhost ~]$ /opt/apache-cassandra/bin/nodetool status
Datacenter: dc1
===============
Status=Up/Down
|/ State=Normal/Leaving/Joining/Moving
--  Address        Load       Tokens       Owns (effective)  Host ID                               Rack
UN  192.168.34.72  100.79 KiB  32           27.3%             592ac186-9c23-414b-b503-554bccc2a925  rack1
UN  192.168.34.73  65.87 KiB  32           37.8%             a8e54091-9f51-4b9b-b222-d89141144af0  rack1
UN  192.168.34.74  83.32 KiB  32           38.1%             dbf126b8-6fd1-4d37-a400-907ffdecf183  rack1
UN  192.168.34.75  99.6 KiB   32           32.3%             c83a1088-5a1a-4c6a-ab04-e470d0671065  rack1
UN  192.168.34.76  65.86 KiB  32           35.5%             20f3a247-0e85-4f5e-9c7b-dbe86db703bd  rack1
UN  192.168.34.77  99.61 KiB  32           29.0%             c4b232bd-e32d-4e64-b58c-6dc473804de0  rack1

[vagrant@localhost ~]$
```

So, to recap, by using a single `vagrant ... up` command we were able to quickly spin up a cluster consisting of of six Cassandra nodes (three seed nodes and three non-seed nodes), and a similar `vagrant ... up` command could be used to build a cluster consisting of any number of seed and non-seed nodes.

## Separating instance creation from provisioning
While the `vagrant up` commands that are shown above can be used to easily deploy Cassandra to a single node or to build a Cassandra cluster consisting of multiple nodes, the [Vagrantfile](../Vagrantfile) included in this distribution also supports separating out the creation of the virtual machine from the provisioning of that virtual machine using the Ansible playbook contained in this repository's [provision-cassandra.yml](../provision-cassandra.yml) file.

To create a set of virtual machines that we plan on using to build a Cassandra cluster without provisioning Cassandra to those machines, simply run a command similar to the following:

```bash
$ vagrant -c="192.168.34.72,192.168.34.73,192.168.34.74,192.168.34.75,192.168.34.76,192.168.34.77" \
    up --no-provision
```

This will create a set of six virtual machines with the appropriate IP addresses ("192.168.34.72", "192.168.34.72", etc.), but will skip the process of provisioning those VMs with an instance of Cassandra. Note that when you are creating the virtual machines but skipping the provisioning step it is not necessary to provide the list of seed nodes using the `-s, --seed-nodes` flag.

To provision the machines that were created above and configure those machines as a Cassandra cluster, we simply need to run a command like the following:

```bash
$ vagrant -c="192.168.34.72,192.168.34.73,192.168.34.74,192.168.34.75,192.168.34.76,192.168.34.77" \
    -s="192.168.34.72,192.168.34.73,192.168.34.74" provision
```

That command will attach to the named instances and run the playbook in this repository's [provision-cassandra.yml](../provision-cassandra.yml) file on those node (first on the seed nodes, then on the non-seed nodes), resulting in a Cassandra cluster consisting of the nodes that were created in the `vagrant ... up --no-provision` command that was shown, above.

## Additional vagrant deployment options
While the commands shown above will install Cassandra with a reasonable, default configuration from a standard location, there are additional command-line parameters that can be used to override the default values that are embedded in the [vars/cassandra.yml](../vars/cassandra.yml) and [defaults/main.yml](../defaults/main.yml) files. Here is a complete list of the command-line flags that can be included in any `vagrant ... up` or `vagrant ... provision` command:

* **`-c, --cassandra-list`**: the Cassandra address list; this is the list of nodes that will be created and provisioned, either by a single `vagrant ... up` command or by a `vagrant ... up --no-provision` command followed by a `vagrant ... provision` command; this command-line flag **must** be provided for almost every `vagrant` command supported by the [Vagrantfile](../Vagrantfile) in this repository
* **`-s, --seed-nodes`**: a comma-separated list of the nodes in the Cassandra address list that should be deployed as seed nodes for the cluster being built; this argument **must** be provided for any `vagrant` commands that involve provisioning of the instances that make up a Cassandra cluster
* **`-u, --url`**: the URL that the Apache Cassandra distribution should be downloaded from. This flag can be useful in situations where there is limited (or no) internet access; in this situation the user may wish to download the distribution from a local web server rather than from the standard Cassandra download site
* **`-o, --home`**: the path to the directory that the Cassandra distribution should be unpacked into during the provisioning process; this defaults to the `/opt/apache-cassandra` directory if not specified
* **`-d, --data`**: the path to the directory where Cassandra will store it's data; this defaults to `/data` if not specified
* **`-v, --version`**: the version of Cassandra that should be downloaded and installed; this parameter is only useful when downloading Cassandra from the standard Cassandra download site
* **`-n, --cluster-name`**: the name of the cluster that should be created; defaults to 'Test Cluster' if not specified
* **`-t, --trickle-fsync`**: if this command-line flag is included on the command-line, the `trickle-fsync` property will be enabled on the provisioned nodes
* **`-l, --listen-method`**: the listen method for the node; should be set to either `"address: IP_ADDRESS"` or `"interface: ADAPTOR"`, and defaults to `"address: IP_ADDRESS"` if not specified
* **`-r, --rpc-method`**: the RPC listen method for the node; should be set to either `"address: IP_ADDRESS"` or `"interface: ADAPTOR"`, and defaults to `"address: IP_ADDRESS"` if not specified
* **`-y, --yum-url`**: the local YUM repository URL that should be used when installing packages during the node provisioning process. This can be useful when installing Cassandra onto CentOS-based VMs in situations where there is limited (or no) internet access; in this situation the user might want to install packages from a local YUM repository instead of the standard CentOS mirrors. It should be noted here that this parameter is not used when installing Cassandra on RHEL-based VMs; in such VMs this option will be silently ignored if set
* **`-f, --local-vars-file`**: the *local variables file* that should be used when deploying the cluster. A local variables file can be used to maintain the configuration parameter definitions that should be used for a given Cassandra cluster deployment, and values in this file will override any values that are either embedded in the [vars/cassandra.yml](../vars/cassandra.yml) and [defaults/main.yml](../defaults/main.yml) files as defaults or passed into the `ansible-playbook` command as extra variables
* **`-c, --clear-proxy-settings`**: if set, this command-line flag will cause the playbook to clear any proxy settings that may have been set on the machines being provisioned in a previous ansible-playbook run. This is useful for situations where an HTTP proxy might have been set incorrectly and, in a subsequent playbook run, we need to clear those settings before attempting to install any packages or download the Cassandra distribution without an HTTP proxy

As an example of how these options might be used, the following command will download the gzipped tarfile containing the Apache Cassandra distribution from a local web server, rather than downloading it from the main Apache distribution site, and override the default configuration parameter definitions in this repository with those defined in the `projectx.yml` file (a project-specific *local variables file*) when provisioning the machines created by the `vagrant ... up --no-provision` command shown above:

```bash
$ vagrant -c="192.168.34.72,192.168.34.73,192.168.34.74,192.168.34.75,192.168.34.76,192.168.34.77" \
    -s="192.168.34.72,192.168.34.73,192.168.34.74" \
    -u="https://10.0.2.2/apache-cassandra-3.10-bin.tar.gz" \
    -f="projectx.yml" provision
```

Obviously, while the list of command-line parameters shown above cannot easily be extended to support all of the configuration options that can be set for any given Cassandra deployment, it is entirely possible to include any of the configuration parameters that can be need to be set in a *local variables file*, then pass them in as is shown above using the `-f, --local-vars-file` command-line option to the `vagrant ... up` or `vagrant ... provision` command (like the example `vagrant ... provision` command that is shown, above).
