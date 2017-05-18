# Example deployment scenarios

There are a four basic deployment scenarios that are supported by this playbook. In the first two (shown below) we'll walk through the deployment of Cassandra to a single node and the deployment of a multi-node Cassandra cluster using a static inventory file. The third scenario will show how the same multi-node Cassandra cluster deployment shown in the second scenario could be performed using the dynamic inventory scripts for both AWS and OpenStack instead of a static inventory file. Finally, we'll walk through the process of "growing" an existing cluster by adding nodes to it.

## Scenario #1: deploying Cassandra to a single node
While this is the simplest of the deployment scenarios that are supported by this playbook, it is more than likely that deployment of Cassandra to a single node is really only only useful for very simple test environments. Even the most basic (default) Cassandra deployments that are typically shown in online examples of how to deploy Cassandra are two-node deployments.  Nevertheless, we will start our discussion with this deployment scenario since it is the simplest.

If we want to deploy Cassandra to a single node with the IP address "192.168.34.70", we could simply create a very simple inventory file that looks something like the following:

```bash
$ cat single-node-inventory

192.168.34.70 ansible_ssh_user='cloud-user' ansible_ssh_private_key_file='keys/test_node_private_key'

[cassandra]
192.168.34.70

$ 
```

Note that in this example inventory file the `ansible_ssh_host` and `ansible_ssh_port` will take their default values since they aren't specified for our host in this very simple static inventory file. Once we've built our static inventory file, we can then deploy Cassandra to our single node by running an `ansible-playbook` command that looks something like this:

```bash
$ ansible-playbook -i single-node-inventory provision-cassandra.yml
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

[cassandra_seed]
192.168.34.72
192.168.34.73
192.168.34.74

[cassandra]
192.168.34.75
192.168.34.76
192.168.34.77

$
```

Note that in this case, the host groups needed for the playbook run (the `cassandra_seed` and `cassandra` host groups) are actually defined (init-file style) at the bottom of the static inventory file. If a static inventory file is used with the playbooks in this repository, we assume that this will always be the case (there is no attempt made to modify the host groups defined in a static inventory file that is passed into the playbook).

The playbook defined in the [provision-cassandra.yml](../provision-cassandra.yml) file in this repository will deploy Cassandra to to all of the hosts in the clsuter defined in the static inventory file shown above in a single `ansible-playbook` run, but the deployment is actually broken up into two separate plays:

* In the first play in the playbook, Cassandra is deployed to the three seed nodes and those nodes are configured to work with each other as part a cluster
* In the second play in the playbook, Cassandra is deployed to the three non-seed nodes, and those nodes are configured to talk to the seed nodes in order to join the cluster

When the `ansible-playbook` run is complete, we will have a set of six nodes that are all working together as a cluster, distributing the data and the workload managed by the cluster between them. To deploy our cluster using the static inventory file shown above, we'd run a command that looks something like this:

```bash
$ ansible-playbook -i test-cluster-inventory -e "{ \
      data_iface: eth0, api_iface: eth1, \
      cassandra_url: 'http://192.168.34.254/apache-cassandra/apache-cassandra-3.10-bin.tar.gz', \
      yum_repo_url: 'http://192.168.34.254/centos', \
      cassandra_data_dir: '/data', start_cassandra: true \
    }" provision-cassandra.yml
```

Alternatively, rather than passing all of those arguments in on the command-line as extra variables, we can make use of the *local variables file* support that is built into this playbook and construct a YAML file that looks something like this containing the configuration parameters that are being used for this deployment:

```yaml
data_iface: eth0
api_iface: eth1
cassandra_url: 'http://192.168.34.254/apache-cassandra/apache-cassandra-3.10-bin.tar.gz'
yum_repo_url: 'http://192.168.34.254/centos'
cassandra_data_dir: '/data'
```

and then we can pass in the *local variables file* as an argument to the `ansible-playbook` command; assuming the YAML file shown above was in the current working directory and was named `test-cluster-deployment-params.yml`, the resulting command would look somethin like this:

```bash
$ ansible-playbook -i test-cluster-inventory -e "{ \
      local_vars_file: 'test-cluster-deployment-params.yml', \
      start_cassandra: true \
    }" provision-cassandra.yml
```

As an aside, it should be noted here that the [provision-cassandra.yml](../provision-cassandra.yml) playbook includes a [shebang](https://en.wikipedia.org/wiki/Shebang_(Unix)) line at the beginning of the playbook file. As such, the playbook can be executed directly as a shell script (rather than using the file as the final input to an `ansible-playbook` command). This means that the command that was shown above could also be run as:

```bash
$ ./provision-cassandra.yml -i test-cluster-inventory -e "{ \
      local_vars_file: 'test-cluster-deployment-params.yml', \
      start_cassandra: true \
    }"
```

This form is available as a replacement for any of the `ansible-playbook` commands that we show here; which form you use will likely be a matter of personal preference (since both accomplish the same thing).

Once that playbook run is complete, we can SSH into one of our nodes and run the `nodetool` command to view the status of each of the nodes our cluster:

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

As you can quite clearly see, the `ansible-playbook` command shown aqbove deployed Cassandra to all six of our nodes and configured them as a single cluster. Of course, additional nodes could be added to this cluster in a subsequent playbook run; a topic that we will cover in the last scenario (below). 

## Scenario #3: deploying a Cassandra cluster via dynamic inventory
In this section we will repeat the multi-node cluster deployment that we just showed in the previous scenario, but we will use the `build-app-host-groups` role that is provided in the [common-roles](../common-roles) submodule to control the deployment of our Cassandra cluster to an AWS or OpenStack environment rather than relying on a static inventory file.

To accomplish this, the we have to:

* Tag the seed and non-seed instances in the AWS or OpenStack environment that we will be configuring as a cluster with the appropriate `Tenant`, `Project`, `Domain`, and `Application` tags. Note that for all of the nodes we are targeting in our playbook runs, we will assign an `Application` tag of `cassandra`; the seed nodes will be assigned a `Role` tag of `seed` and a `Role` tag will not be assigned to the non-seed nodes
* Once all of the nodes that will make up our cluster had been tagged appropriately, we can run an `ansible-playbook` command similar to the `ansible-playbook` command shown in the previous scenario; this will deploy Cassandra to the both the seed and non-seed nodes (in that order).

As was the case with the static inventory example described in the previous scenario, it the deployment of Cassandra to our seed and non-seed nodes takes place in two separate plays that are both triggered by a single `ansible-playbook` command; when that `ansible-playbook` command is complete, we have a working Cassandra cluster with a mix of seed and non-seed nodes.

In terms of what the commands look like, lets assume for this example that we've tagged our seed nodes with the following VM tags:

* **Tenant**: labs
* **Project**: projectx
* **Domain**: preprod
* **Application**: cassandra
* **Role**: seed

and the non-seed nodes have been assigned the same set of `Tenant`, `Project`, `Domain`, and `Application` tags, but are not tagged with a `Role` tag. The `ansible-playbook` command used to deploy our Cassandra cluster in an OpenStack environment would then look something like this:

```bash
$ ansible-playbook -e "{ \
        application: cassandra, cloud: osp, \
        tenant: labs, project: projectx, domain: preprod, \
        private_key_path: './keys', data_iface: eth0, api_iface: eth1, \
        cassandra_data_dir: '/data', start_cassandra: true \
    }" provision-cassandra.yml
```

In an AWS environment, the command used would look quite similar:

```bash
$ AWS_PROFILE=datanexus_west ansible-playbook -e "{ \
        application: cassandra, cloud: aws, \
        tenant: labs, project: projectx, domain: preprod, \
        private_key_path: './keys', data_iface: eth0, api_iface: eth1, \
        cassandra_data_dir: '/data', start_cassandra: true \
    }" provision-cassandra.yml
```

As you can see, these two commands only in terms of the environment variable defined at the beginning of the command-line used to provision to the AWS environment (`AWS_PROFILE=datanexus_west`) and the value defined for the `cloud` variable (`osp` versus `aws`). In both cases the result would be a set of nodes deployed as a Cassandra cluster, with the number of nodes and their roles in the cluster determined (completely) by the tags that were assigned to them.

## Scenario #4: adding nodes to a multi-node Cassandra cluster
When adding nodes to an existing Cassandra cluster, we must be careful of several things:

* We don't want to redeploy Cassandra to the existing nodes in the cluster, only to the new nodes we are adding
* We want to make sure the nodes we are adding to the cluster are configured properly to join that cluster, but we don't want the `cassandra` service started automatically as part of the playbook run.
    * Once we finish provisioning the nodes that we want to add to the cluster, we will have to start the `cassandra` service separately on each of those nodes; waiting for the cluster to rebalance it's workload after the `cassandra` service on each node has been started before we attempt to add any additional nodes to the cluster.
    * This means that the nodes have to be added to the cluster individually, and that there is an indeterminate wait between when each node is added (depending on how long it takes the cluster to redistribute the data it is managing); in reality, this means the process of adding each node to the cluster (by starting its `cassandra` service) must be managed manually, so the `start_cassandra` parameter used in the playbook runs that were shown above must be set to `false` (its default value)
* We will want to make sure the nodes being added are set to `auto_bootstrap` when they start (so that they reach out to the seed nodes to be sure they are configured correctly before joining the cluster). This parameter is set to `true` by default if it is not specified in the `cassandra.yaml` file, so this is merely a matter of making sure we don't set this parameter to `false` (as might have been done when building the initial cluster, before any data was added to it).

In the case of adding (non-seed) nodes to an existing cluster, the process is relatively simple and basically looks like one of the provisioning scenarios that were shown above. The only differences are:

* we need to ensure that if the `auto_bootstrap` parameter is defined (either as an extra variable or in a *local inventory file* it is set to `true` (it is `true` by default)
* we need to ensure that if the `start_cassandra` parameter is defined (either as an extra variable or in a *local inventory file* it is set to `false` (it is `false` by default)
* we need to make sure that the configuration parameters passed into the playbook run, either as extra variables or in a *local inventory file*, are the same values that were used for those parameters when we deployed the initial cluster

To make matters simpler (and ensure that there is no danger of reprovisioning the nodes in the exiting cluster when attempting to add new nodes to it), we have actually separated out the plays that are used to add nodes to an existing cluster into a separate playbook (the [add-nodes.yml](./add-nodes.yml) file in this repository).

As was mentioned, above, it is critical that the same configuration parameters be passed in during the process of adding new nodes to the cluster as were passed in when building the cluster initially. Cassandra is not very tolerant of differences in configuration between members of a cluster, so we will want to avoid those situations. The easiest way to manage this is to use a *local inventory file* to manage the configuration parameters that are used for a given cluster, then pass in that file as an argument to the `ansible-playbook` command that you are running to add nodes to that cluster. That said, in the examples we show (below) we will define the configuration parameters that were set to non-default values in the previous playbook runs as extra variables that are passed into the `ansible-playbook` command on the command-line for clarity.

To provide a couple of examples of how this process of growing a cluster works, we would first like to walk through the process of adding three new nodes to the existing Cassandra cluster that was created using the `test-cluster-inventory` (static) inventory file, above. The first step would be to edit the static inventory file and add the three new nodes to the `cassandra` host group, then save the resulting file. The host groups defined in the `test-cluster-inventory` file shown above would look like this after those edits:

```
[cassandra_seed]
192.168.34.72
192.168.34.73
192.168.34.74

[cassandra]
192.168.34.75
192.168.34.76
192.168.34.77
192.168.34.78
192.168.34.79
192.168.34.80
```

(note that we have only shown the tail of that file; the hosts defined at the start of the file would remain the same). With the new static inventory file in place, the playbook command that we would run to add the three additional nodes to our cluster would look something like this:

```bash
$ ansible-playbook -i test-cluster-inventory -e "{ \
      data_iface: eth0, api_iface: eth1, \
      cassandra_url: 'http://192.168.34.254/apache-cassandra/apache-cassandra-3.10-bin.tar.gz', \
      yum_repo_url: 'http://192.168.34.254/centos', cassandra_data_dir: '/data', \
      auto_bootstrap: true, start_cassandra: false \
    }" add-nodes.yml
```

As you can see, this is essentially the same command we ran previously to provision our cluster initially in the static inventory scenario. The only changes to the previous command are that we are using a different playbook (the [add-nodes.yml](../add-nodes.yml) playbook instead of the [provision-cassandra.yml](../provision-cassandra.yml) playbook), the `start_cassandra` variable has been set to `false` (that is the default value, but we often set it here anyway just to ensure that the `cassandra` service won't be set to start at the end of the playbook run because of a value set elsewhere) and the `auto_bootstrap` variable has been set to `true` (so the new nodes that we are adding will explicitly be set to `auto_bootstrap` when they are started; again this is the default but we have specified it explicitly here).

To add new nodes to an existing Cassandra cluster in an AWS or OpenStack environment, we would simply create the new nodes we want to add in that environment and tag them appropriately (using the same `Tenant`, `Application`, `Project`, and `Domain` tags that we used when creating our initial cluster). With those new machines tagged appropriately, the command used to add a new set of nodes to an existing cluster in an OpenStack environment would look something like this:

```bash
$ ansible-playbook -e "{ \
        application: cassandra, cloud: osp, \
        tenant: labs, project: projectx, domain: preprod, \
        private_key_path: './keys', data_iface: eth0, api_iface: eth1, \
        cassandra_data_dir: '/data', \
        auto_bootstrap: true, start_cassandra: false \
    }" add-nodes.yml
```

The only difference when adding nodes to an AWS environment would be the environment variable that needs to be set at the beginning of the command-line (eg. `AWS_PROFILE=datanexus_west`) and the cloud value that we define within the extra variables that are passed into that `ansible-playbook` command (`aws` instead of `osp`):

```bash
$ AWS_PROFILE=datanexus_west ansible-playbook -e "{ \
        application: cassandra, cloud: aws, \
        tenant: labs, project: projectx, domain: preprod, \
        private_key_path: './keys', data_iface: eth0, api_iface: eth1, \
        cassandra_data_dir: '/data', \
        auto_bootstrap: true, start_cassandra: false \
    }" add-nodes.yml
```

As was the case with the static inventory example shown above, the command shown here for adding new nodes to an existing cluster in an AWS or OpenStack cloud (using tags and dynamic inventory) is essentially the same command that was used when deploying the initial cluster, but we are using a different playbook (the [add-nodes.yml](../add-nodes.yml) playbook instead of the [provision-cassandra.yml](../provision-cassandra.yml) playbook), we have set the `start_cassandra` variable to `false` (so that the `cassandra` service will not start automatically when the playbook run is complete), and we have set the `auto_bootstrap` variable to `true` (so the new nodes we are adding will explicitly be set to `auto_bootstrap` when they are started).

It should be noted that the playbook associated with this role does not currently support the process of adding new seed nodes to an existing cluster, only the process of adding non-seed nodes. Adding a new seed node (or nodes) involves modifying the `cassandra.yaml` configuration on every node in the cluster in order to add the new seed node (or nodes) to the list of seeds that is defined there. This requires that each node be taken offline, it's configuration modified, then brought back online. That process is hard (at best) to automate, so we have made no effort to do so in the current version of this playbook.
