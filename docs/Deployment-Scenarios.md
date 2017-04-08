# Example deployment scenarios

There are a four basic deployment scenarios that are supported by this playbook. In the first two (shown below) we'll walk through the deployment of Cassandra to a single node and the deployment of a multi-node Cassandra cluster using a static inventory file. The third scenario will show how the same multi-node Cassandra cluster deployment shown in the second scenario could be performed using the dynamic inventory scripts for both AWS and OpenStack instead of a static inventory file. Finally, we'll walk through the process of "growing" an existing cluster by adding nodes to it. This combination of scenarios should show how most of the configuraiton parameters shown above can be used.

## Scenario #1: deploying Cassandra to a single node
While this is the simplest of the deployment scenarios that are supported by this playbook, it is more than likely that deployment of Cassandra to a single node is really only only useful for very simple test environments. Even the most basic (default) Cassandra deployments that are typically shown in online examples of how to deploy Cassandra are two-node deployments.  Nevertheless, we will start our discussion with this deployment scenario since it is the simplest. If we want to deploy Cassandra to a single node with the IP address "192.168.34.12", we could simply create a very simple inventory file that looks something like the following:

```bash
$ cat single-node-inventory

192.168.34.12 ansible_ssh_user='cloud-user' ansible_ssh_private_key_file='keys/test_node_private_key'

$ 
```

Note that in this example inventory file the `ansible_ssh_host` and `ansible_ssh_port` will take their default values since they aren't specified for our host in this very simple static inventory file. Once we've built our static inventory file, we can then deploy Cassandra to our single node by running an `ansible-playbook` command that looks something like this:

```bash
$ ansible-playbook -i single-node-inventory -e "{ host_inventory: ['192.168.34.12'] }" site.yml
```

This will download the Apache Cassandra distribution file from the default download server defined in the [defaults/main.yml](../defaults/main.yml) file, unpack that gzipped tarfile into the `/opt/apache-cassandra` directory on that host, and install Cassandra on that node and configure that node as a single-node Cassandra "cluster", using the default configuration parameters that are defined in the [vars/cassandra.yml](../vars/cassandra.yml) and [defaults/main.yml](../defaults/main.yml) files.

## Scenario #2: deploying a multi-node Cassandra cluster
If you are using this playbook to deploy a multi-node Cassandra cluster, then the configuration becomes a bit more complicated. The Cassandra cluster actually consists of nodes with two different roles, a set of one or more seed nodes any new nodes will need to contact when joining the cluster and a larger set of non-seed nodes. In addition to needing to know which nodes are seed nodes and which are non-seed nodes, all nodes in the cluster need to be configured similarly and all of them need to agree (as part of that configuration) on what the name of the cluster is. It is in this second scenario that support for a *local variables file* in the `dn-cassandra` role becomes important.

Let's assume that we are deploying Cassandra to a cluster of six nodes (three seed nodes and three non-seed nodes) and, furthermore, let's assume that we're going to be using a static inventory file to control this deployment. The static inventory file that we will be using for this example looks like this:

```bash
$ cat test-cluster-inventory
# example inventory file for a clustered deployment

192.168.34.72 ansible_ssh_host=192.168.34.72 ansible_ssh_port=22 ansible_ssh_user='cloud-user' ansible_ssh_private_key_file='keys/test_cluster_private_key'
192.168.34.73 ansible_ssh_host=192.168.34.73 ansible_ssh_port=22 ansible_ssh_user='cloud-user' ansible_ssh_private_key_file='keys/test_cluster_private_key'
192.168.34.74 ansible_ssh_host=192.168.34.74 ansible_ssh_port=22 ansible_ssh_user='cloud-user' ansible_ssh_private_key_file='keys/test_cluster_private_key'
192.168.34.75 ansible_ssh_host=192.168.34.75 ansible_ssh_port=22 ansible_ssh_user='cloud-user' ansible_ssh_private_key_file='keys/test_cluster_private_key'
192.168.34.76 ansible_ssh_host=192.168.34.76 ansible_ssh_port=22 ansible_ssh_user='cloud-user' ansible_ssh_private_key_file='keys/test_cluster_private_key'
192.168.34.77 ansible_ssh_host=192.168.34.77 ansible_ssh_port=22 ansible_ssh_user='cloud-user' ansible_ssh_private_key_file='keys/test_cluster_private_key'
192.168.34.78 ansible_ssh_host=192.168.34.78 ansible_ssh_port=22 ansible_ssh_user='cloud-user' ansible_ssh_private_key_file='keys/test_cluster_private_key'
192.168.34.79 ansible_ssh_host=192.168.34.79 ansible_ssh_port=22 ansible_ssh_user='cloud-user' ansible_ssh_private_key_file='keys/test_cluster_private_key'
192.168.34.80 ansible_ssh_host=192.168.34.80 ansible_ssh_port=22 ansible_ssh_user='cloud-user' ansible_ssh_private_key_file='keys/test_cluster_private_key'

$
```

The playbook defined in the [site.yml](../site.yml) file in this repository supports the deployment of a cluster using this static inventory file, but the deployment is actually broken up into two separate `ansible-playbook` runs:

* In the first playbook run, Cassandra is deployed to the three seed nodes and those nodes are configured to work with each other as part a cluster
* In the second playbook run, Cassandra is deployed to the three non-seed nodes, and those nodes are configured to talk to the seed nodes in order to join the cluster

When the two `ansible-playbook` runs are complete, we will have a set of six nodes that are all working together as a cluster, distributing the data and the workload across all of the nodes in the cluster. For purposes of this example, let's assume that we want to setup the first three nodes listed in the `test-cluster-inventory` static inventory file (above) as seed nodes and the next three nodes in that inventory file as non-seed nodes. In that case, we'd run a command that looks something like this in order to deploy our initial set of three seed nodes:

```bash
$ ansible-playbook -i test-cluster-inventory -e "{ \
      host_inventory: ['192.168.34.72', '192.168.34.73', '192.168.34.74'], \
      cloud: vagrant, cassandra_seed_nodes: ['192.168.34.72', '192.168.34.73', '192.168.34.74'], \
      data_iface: eth0, api_iface: eth1, \
      cassandra_url: 'http://192.168.34.254/apache-cassandra/apache-cassandra-3.10-bin.tar.gz', \
      yum_repo_url: 'http://192.168.34.254/centos', \
      cassandra_data_dir: '/data', start_cassandra: true, cassandra_jvm_heaps_size: 1G \
    }" site.yml
```

Once that playbook run is complete, we can SSH into one of our nodes and run the `nodetool` command to view the status of each of the nodes our cluster:

```bash
$ /opt/apache-cassandra/bin/nodetool status
Datacenter: dc1
===============
Status=Up/Down
|/ State=Normal/Leaving/Joining/Moving
--  Address        Load       Tokens       Owns (effective)  Host ID                               Rack
UN  192.168.34.72  99.6 KiB   32           73.9%             6685ca6a-bf8d-41d1-924b-187279a076a6  rack1
UN  192.168.34.73  65.86 KiB  32           62.0%             3e556a60-b440-4f2f-9047-aa70eb7b387c  rack1
UN  192.168.34.74  65.86 KiB  32           64.1%             6746f789-71c0-40c9-9564-d705e9b43732  rack1

$
```

with our initial set of three seed nodes up and running, we can now run the second `ansible-playbook` command to deploy Cassandra to our non-seed nodes and add them to the existing cluster:

```bash
$ ansible-playbook -i test-cluster-inventory -e "{ \
      host_inventory: ['192.168.34.75', '192.168.34.76', '192.168.34.77'], \
      cloud: vagrant, cassandra_seed_nodes: ['192.168.34.72', '192.168.34.73', '192.168.34.74'], \
      data_iface: eth0, api_iface: eth1, \
      cassandra_url: 'http://192.168.34.254/apache-cassandra/apache-cassandra-3.10-bin.tar.gz', \
      yum_repo_url: 'http://192.168.34.254/centos', \
      cassandra_data_dir: '/data', start_cassandra: true, cassandra_jvm_heaps_size: 1G \
    }" site.yml
```

When this second playbook run is complete, we can then SSH into any of the boxes in our cluster (which now consists of the initial set of three seed nodes and the new set of three non-seed nodes that we just added) and run the same `nodetool` command to check the status of all of the nodes in our cluster again:

```bash
$ /opt/apache-cassandra/bin/nodetool status
Datacenter: dc1
===============
Status=Up/Down
|/ State=Normal/Leaving/Joining/Moving
--  Address        Load       Tokens       Owns (effective)  Host ID                               Rack
UN  192.168.34.72  99.6 KiB   32           29.2%             6685ca6a-bf8d-41d1-924b-187279a076a6  rack1
UN  192.168.34.73  65.86 KiB  32           31.0%             3e556a60-b440-4f2f-9047-aa70eb7b387c  rack1
UN  192.168.34.74  65.86 KiB  32           30.6%             6746f789-71c0-40c9-9564-d705e9b43732  rack1
UN  192.168.34.75  99.61 KiB  32           41.6%             01bbe5f2-fac3-443f-97dd-7ce170ecde6c  rack1
UN  192.168.34.76  99.6 KiB   32           37.1%             01216722-d2a6-4bfa-ae6e-b76481470828  rack1
UN  192.168.34.77  99.61 KiB  32           30.5%             f4d3ac7d-0ec8-4b47-8811-c7fb09e8a04b  rack1

$
```

This pair of `ansible-playbook` commands deployed Cassandra to all six of our nodes and configured them as a single cluster. Of course, additional nodes could be added to this cluster in a subsequent playbook run; a topic that we will cover in the last scenario (below). 

## Scenario #3: deploying a Cassandra cluster via dynamic inventory
In this section we will repeat the multi-node cluster deployment that we just showed in the previous scenario, but we will use the dynamic inventory scripts provided in the [common-utils](../common-utils) submodule to control the deployment of our Cassandra cluster to an AWS or OpenStack environment rather than relying on a static inventory file.

To accomplish this, the we have to:

* Tag the seed and non-seed instances in the AWS or OpenStack environment that we will be configuring as a cluster with the appropriate `Tenant`, `Project`, `Domain`, and `Application` tags. Note that for all of the nodes we are targeting in our playbook runs, we will assign an `Application` tag of `cassandra`; the seed nodes will be assigned a `Role` tag of `seed`
* Once all of the nodes that will make up our cluster had been tagged appropriately, we can run `ansible-playbook` command similar to the first `ansible-playbook` command shown in the previous scenario; this will deploy Cassandra to the initial set of (seed) nodes. It is important to note that in this `ansible-playbook` command, the non-seed nodes must be included in the `skip_nodes_list` value passed into the `ansible-playbook` command. This will ensure that the deployment will only target the seed nodes in the cluster; without this argument, the non-seed nodes would not be targeted for deployment and the playbook would deploy Cassandra to the non-seed nodes, instructing them to talk to the non-existent seed nodes in order to join a non-existent Cassandra cluster
* Once the seed nodes are up and running, the same `ansible-playbook` command that we just ran to deploy Cassandra to those seed nodes will be re-run, but this time without listing the non-seed nodes in the `skip_nodes_list`; this will deploy Cassandra to the remaining (non-seed) nodes in the cluster and instruct them to talk to the seed nodes in order to join the cluster.

As was the case with the static inventory example described in the previous scenario, it takes two passes to deploy our cluster. In the first pass we deploy Cassandra to the seed nodes, while in the second we deploy Cassandra to the non-seed nodes. When the two passes are complete, we have a working Cassandra cluster with a mix of seed and non-seed nodes.

As an aside, if we wanted to make the two `ansible-playbook` commands for our two passes identical, we could modify the sequence of events sketched out, above, to look more like this:

* First, the seed instances in the AWS or OpenStack environment that make up the cluster that we are deploying would be tagged with an `Application` tag of `cassandra` and a `Role` tag of `seed`; the non-seed nodes **would be left untagged** for the moment (i.e. neither an `Application` tag nor a `Role` tag would be assigned to these nodes)
* Once the seed nodes had been tagged appropriately, we could run our `ansible-playbook` command. Note that in this sequence of events there would be no need to add a `skip_nodes_list` value to the `ansible-playbook` command used for the first pass of our deployment since there would be no matching non-seed nodes that need to be skipped (only the seed nodes would be found by the [build-app-host-groups](../common-roles/build-app-host-groups) role based on the tags that were passed into the `ansible-playbook` command).
* After the first pass playbook run was complete, the non-seed nodes would be tagged with an `Application` tag of `cassandra`, and a second `ansible-playbook` command (identical to the first) would be run. In that playbook run, the [build-app-host-groups](../common-roles/build-app-host-groups) role would find a mix of both seed and non-seed nodes, and the playbook would deploy Cassandra to the non-seed nodes, configuring them to talk to the seed nodes in order to join the cluster.

While the effect of this second approach would be identical to the first, one may be preferred over the other depending on the environment you are deploying Cassandra into. For example, if you don't have administrative access to the nodes in order to tag them yourself, it might be easier to use the first approach since you would only have to ask an administrator to tag the machines once. On the other hand, if you can't easily determine the IP addresses for the non-seed nodes ahead of time, then the second approach mignt be easier since you wouldn't have to know the IP addresses of the machines that should be skipped during the first run. Both scenarios are sketched out here so that you can choose the one that best suits your needs.

In terms of what the commands look like, lets assume for this example that we've tagged our seed nodes with the following VM tags:

* **Tenant**: labs
* **Project**: projectx
* **Domain**: preprod
* **Application**: cassandra
* **Role**: seed

and the non-seed nodes have been assigned the same set of `Tenant`, `Project`, `Domain`, and `Application` tags. The `ansible-playbook` command used to deploy Cassandra to our initial set of seed nodes in an OpenStack environment would then look something like this (assuming that the IP addresses for our three non-seed nodes were '10.0.1.15', '10.0.1.16', and '10.0.1.17'):

```bash
$ ansible-playbook -i common-utils/inventory/osp/openstack -e "{ \
        host_inventory: 'meta-Application_cassandra:&meta-Cloud_osp:&meta-Tenant_labs:&meta-Project_projectx:&meta-Domain_preprod', \
        skip_nodes_list: ['10.0.1.15', '10.0.1.16', '10.0.1.17'], \
        application: cassandra, cloud: osp, tenant: labs, project: projectx, domain: preprod, \
        ansible_user: cloud-user, private_key_path: './keys', data_iface: eth0, api_iface: eth1, \
        cassandra_data_dir: '/data', start_cassandra: true \
    }" site.yml
```

once that playbook run was complete, you could simply re-run the same command (without the `skip_nodes_list` to deploy Cassandra to the non-seed nodes:

```bash
$ ansible-playbook -i common-utils/inventory/osp/openstack -e "{ \
        host_inventory: 'meta-Application_cassandra:&meta-Cloud_osp:&meta-Tenant_labs:&meta-Project_projectx:&meta-Domain_preprod', \
        application: cassandra, cloud: osp, tenant: labs, project: projectx, domain: preprod, \
        ansible_user: cloud-user, private_key_path: './keys', data_iface: eth0, api_iface: eth1, \
        cassandra_data_dir: '/data', start_cassandra: true \
    }" site.yml
```

In an AWS environment, the commands look quite similar; the command used for the first pass (provisioning the seed nodes in the cluster) would look something like this (again, assuming that the IP addresses for the three non-seed nodes were '10.10.0.15', '10.10.0.15', and '10.10.0.15'):

```bash
$ ansible-playbook -i common-utils/inventory/aws/ec2 -e "{ \
        host_inventory: 'tag_Application_cassandra:&tag_Cloud_aws:&tag_Tenant_labs:&tag_Project_projectx:&tag_Domain_preprod', \
        skip_nodes_list: ['10.10.0.15', '10.10.0.15', '10.10.0.15'], \
        application: cassandra, cloud: aws, tenant: labs, project: projectx, domain: preprod, \
        ansible_user: cloud-user, private_key_path: './keys', data_iface: eth0, api_iface: eth1, \
        cassandra_data_dir: '/data', start_cassandra: true \
    }" site.yml
```

while the command used for the second pass (provisioning the non-seed nodes) would look something like this:

```bash
$ ansible-playbook -i common-utils/inventory/aws/ec2 -e "{ \
        host_inventory: 'tag_Application_cassandra:&tag_Cloud_aws:&tag_Tenant_labs:&tag_Project_projectx:&tag_Domain_preprod', \
        application: cassandra, cloud: aws, tenant: labs, project: projectx, domain: preprod, \
        ansible_user: cloud-user, private_key_path: './keys', data_iface: eth0, api_iface: eth1, \
        cassandra_data_dir: '/data', start_cassandra: true \
    }" site.yml
```

As you can see, they are basically the same commands that were shown for the OpenStack use case, but they are slightly different in terms of the name of the inventory script passed in using the `-i, --inventory-file` command-line argument, the value passed in for `Cloud` tag (and the value for the associated `cloud` variable), and the prefix used when specifying the tags that should be matched in the `host_inventory` value (`tag_` instead of `meta-`). In both cases the result would be a set of nodes deployed as a Cassandra cluster, with the number of nodes and their roles in the cluster determined (completely) by the tags that were assigned to them.

## Scenario #4: adding nodes to a multi-node Cassandra cluster
When adding nodes to an existing Cassandra cluster, we must be careful of several things:

* We don't want to redeploy Cassandra to the existing nodes in the cluster, only to the new nodes we are adding
* We want to make sure the nodes are configured properly to join the cluster, but we don't want the `cassandra` service started automatically as part of the playbook run. Once the nodes are setup to join the cluster, we'll have to go to each node, individually, and start the `cassandra` service separately.
* In addition, we'll have to wait after adding each node until the cluster has redistributed the data it is managing across the new cluster topology before we move on to start the `cassandra` service on the next we are adding to the cluster. This means that the nodes in the cluster have to be added one-by-one, and there is an indeterminate wait between when each node is added (depending on how long it takes the cluster to redistribute the data it is managing)
* We will want to make sure the nodes being added are set to `auto_bootstrap` when they start (so that they reach out to the seed nodes to be sure they are configured correctly before joining the cluster). This parameter is set to `true` by default if it is not specified in the `cassandra.yaml` file, so this is merely a matter of making sure we don't set this parameter to `false` (as might have been done when building the initial cluster, before any data was added to it).

In the case of adding (non-seed) nodes to an existing cluster, the process is relatively simple and basically looks like the second pass of either of the two cluster-based scenarios that were shown above. The only differences are:

* we need to ensure that if the `auto_bootstrap` parameter is defined (either as an extra variable or in a *local inventory file* it is set to `true` (it is `true` by default)
* we need to ensure that if the `start_cassandra` parameter is defined (either as an extra variable or in a *local inventory file* it is set to `false` (it is `false` by default)
* we need to add a `skip_nodes_list` value to the dynamic inventory scenario to ensure that the playbook doesn't try to redeploy Cassandra to the existing (non-seed) nodes in the cluster that we are adding our new nodes to.  For the static inventory use case, we just need to make sure that the list of `cassandra_seed_nodes` is the same as the list that was used when deploying the initial cluster.

In both the static and dynamic use cases is it critical that the same configuration parameters be passed in during the deployment process that were used when building the initial cluster. Cassandra is not very tolerant of differences in configuration between members of a cluster, so we will want to avoid those situations. The easiest way to manage this is to use a *local inventory file* to manage the configuration parameters that are used for a given cluster, then pass in that file as an argument to the `ansible-playbook` command that you are running to add nodes to that cluster. That said, in the examples we show (below) we will define the configuration parameters that were set to non-default values in the previous playbook runs as extra variables that are passed into the `ansible-playbook` command on the command-line for clarity.

To provide a couple of examples of how this process of growing a cluster works, this command could be used to add three new nodes to the existing Cassandra cluster that was created using the `test-cluster-inventory` (static) inventory file, above:

```bash
$ ansible-playbook -i test-cluster-inventory -e "{ \
      host_inventory: ['192.168.34.78', '192.168.34.79', '192.168.34.80'], \
      cloud: vagrant, cassandra_seed_nodes: ['192.168.34.72', '192.168.34.73', '192.168.34.74'], \
      data_iface: eth0, api_iface: eth1, \
      cassandra_url: 'http://192.168.34.254/apache-cassandra/apache-cassandra-3.10-bin.tar.gz', \
      yum_repo_url: 'http://192.168.34.254/centos', cassandra_data_dir: '/data', \
      auto_bootstrap: true, start_cassandra: false, cassandra_jvm_heaps_size: 1G \
    }" site.yml
```

As you can see, this is essentially the same command we ran previously to add the initial set of three non-seed nodes to our cluster in the static inventory scenario. The only changes to the previous command are that the IP addresses for the nodes being added that are passed in using the `host_inventory` extra variable are different, the `start_cassandra` variable has been set to `false` (that is the default value, but we often set it here anyway just to ensure that the `cassandra` service won't be set to start at the end of the playbook run because of a value set elsewhere), and an `auto_bootstrap: true` declaration has been added so the new nodes will explicitly be set to auto_bootstrap when they are started.

As an example of the dynamic inventory use case, this command could be used to add a new set of nodes to our OpenStack cluster (the number of nodes would depend on the number of non-seed nodes that matched the tags that were passed in):

```bash
$ ansible-playbook -i common-utils/inventory/osp/openstack -e "{ \
        host_inventory: 'meta-Application_cassandra:&meta-Cloud_osp:&meta-Tenant_labs:&meta-Project_projectx:&meta-Domain_preprod', \
        skip_nodes_list: ['10.0.1.15', '10.0.1.16', '10.0.1.17'], \
        application: cassandra, cloud: osp, tenant: labs, project: projectx, domain: preprod, \
        ansible_user: cloud-user, private_key_path: './keys', data_iface: eth0, api_iface: eth1, \
        cassandra_data_dir: '/data', auto_bootstrap: true, start_cassandra: false \
    }" site.yml
```

Note that in this dynamic inventory example we are once again using the `skip_nodes_list`, but this time we're using it to skip the non-seed nodes that we have already deployed Cassandra to. As was the case with the static inventory example shown above, we have also set the `start_cassandra` to `false` and added a `auto_bootstrap: true` declaration, so the new nodes will explicitly be set to auto_bootstrap but the `cassandra` service on those new nodes will not start when the playbook run is complete.

It should be noted that the playbook associated with this role does not currently support the process of adding new seed nodes to an existing cluster, only the process of adding non-seed nodes. Adding a new seed node (or nodes) involves modifying the `cassandra.yaml` configuration on every node in the cluster in order to add the new seed node (or nodes) to the list of seeds that is defined there. This requires that each node be taken offline, it's configuration modified, then brought back online. That process is hard (at best) to automate, so we have made no effort to do so in the current version of this playbook.
