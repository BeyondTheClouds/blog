---
layout: post
title: What about network splits
date: 2018-11-07
author: Marie Delavergne, Ronan-Alexandre Cherrueau
categories: [OpenStack, network partitions]
tags: [OpenStack, network partitions, network splits, edge]
---


# Living on the edge #

Modern applications in the realm of the internet, such as the internet of things, compel the development for edge infrastructures. The concept of edge computing is to distribute the computation on a large number of edge devices, closer to the places where the data are collected.
We define this as hundreds of auto-managed and geo-distributed micro data center of dozens of servers. Since they are distributed all over the globe, one can expect to have different latency and bandwith across the network. But what can also be expected and have nonetheless unexpected consequences are network partitions that could happen over such a distributed network.

A partition occurs when the link to a resource is severed and this resource becomes isolated from the others. This resource can be a node from a database, a compute node, an entire data center, etc. When this split happens, we can expected either that this isolated part executions can differ from the main partition, which puts it in an incoherent state, or that it can become simply unavailable.

To build an edge infrastructure without reinventing the wheel, the Discovery initiative investigates the use of OpenStack. OpenStack is an open-source cloud computing infrastructure resource manager widely used, and more and more in the context of edge computing. Openstack is built of two types of nodes which constitutes the data plane on one side and the control plane on the other. The data plane
nodes are able to fulfill typical XaaS needs, such as computation, storage, network, etc. The control nodes are required to process the incoming requests which will probably require to communicate with the data plane. Since all these services are distributed across different nodes, we can expect to get a partition between them at some point.

The goal of this study is to shed some light on the behavior of OpenStack when network partition occurs between a control and a compute node.

# Experimental protocol #
## Base configuration ##

To make the required tests, we used [Enos](https://github.com/BeyondTheClouds/enos), a tool previously developed by the Discovery Initiative, and deployed on [Grid'5000](https://www.grid5000.fr/mediawiki/index.php/Grid5000:Home), a testbed dedicated to research.  The platform gives access to approximately 1000 machines grouped in 30 clusters geographically distributed in 8 sites. This study uses the [paravance cluster](https://www.grid5000.fr/mediawiki/index.php/Rennes:Hardware#paravance) composed of 72 nodes with each:
- **CPU:** Intel Xeon E5-2630 v3 (Haswell, 2.40GHz, 2 CPUs/node, 8 cores/CPU)
- **Memory:** 128 GB
- **Network:**
  - eth0/eno1, Ethernet, configured rate: 10 Gbps, model: Intel 82599ES 10-Gigabit SFI/SFP+ Network Connection, driver: ixgbe
  - eth1/eno2, Ethernet, configured rate: 10 Gbps, model: Intel 82599ES 10-Gigabit SFI/SFP+ Network Connection, driver: ixgbe

Enos enables us to get the resources, deploy the required topology and configure OpenStack according to our needs. We used the following topology:
```python
topology:
  grp1:
    paravance:
      control: 1
      network: 1
  grp2:
    paravance:
      compute: 1
  grp3:
    paravance:
      compute: 2
```
We thus created 3 different groups of 5 paravance servers, connected by two virtual LANs and plugged to each nodes using two different NICs. This servers are 3 compute nodes, one to handle Neutron (network) and one control to manage everything (Glance, Keystone, MariaDB, Horizon, etc.), as shown on Figure<a href="#os_topo_base">1</a>.

Every VM we booted were placed on the compute nodes, numbered from 1 to 3. As seen on the figure and the topology, the nodes are divided into 3 groups: grp1 for the control and the network nodes, grp1 for the compute node which will be isolated and the grp3 acting as compute nodes to make the experiment as well as control samples.

<figure id="os_topo_base">
<img src='{{ "assets/what-about-network-splits/openstack_topology_base.svg" | absolute_url }}' alt="openstack base topology">
<figcaption style="text-align:center"><span class="figure-number">Figure 1: </span>Openstack base topology</figcaption>
</figure>

Enos deployed everything as displayed in the previous figure. It created two VLans, each one associated to a node's NIC.

## OpenStack topology ##

We then used a [heat template file]({{ "assets/what-about-network-splits/heat.hot" | absolute_url }}) to deploy 6 VMs distributed equally across the computes (i.e. 2 VMs per compute) and across 2 private networks (i.e. 3 VMs per network).
<figure id="net_topo">
<img src='{{ "assets/what-about-network-splits/network_topology.svg" | absolute_url }}' alt="Network topology">
	<figcaption style="text-align:center"><span class="figure-number">Figure 2: </span>Network topology</figcaption>
</figure>

The figure [2](#net_topo) presents the network topology in a similar way to Horizon dashboard. We can see that VMs 1, 3 and 5 are in the first network (Nw1) and the 3 others in the second network (Nw2). The yellow/purple/orange represents the compute on which the VMs are located. Thus, VMs 1 and 2 are collocated and VM 3 and 6 are separated from each other. To sum up the positioning of VMs across computes, groups and network:

| VM  | Compute | Group | Network |
|:---:| :---:   | :---: | :---:   |
| 1   | 1       | 2     | 1       |
| 2   | 2       | 2     | 2       |
| 3   | 2       | 3     | 1       |
| 4   | 2       | 3     | 2       |
| 5   | 3       | 3     | 1       |
| 6   | 3       | 3     | 2       |

This architecture allows us to test communications between two networks, two computes and to the outside.
We consider three types of communications:
  * **L2/L3**: In **L2** configuration, everything happens in the same network, whereas in **L3** the messages goes from a network to another.
  * **Full/Dense**: This is about the positioning of the VMs towards the compute. In **dense** mode, the VMs are on the same compute. In **full mode**, they are scattered on two different computes.
  * **East-West/North-South**: Whether the VMs talk to each other (**East-West**) or to an external address, e.g. 8.8.8.8 (**North-South**)

## Cutting the edge ##

We then applied some tc rules using `enos tc` from Enos (these lines are simply put in the configuration file of Enos):
```python
network_constraints:
  enable: true
  default_delay: 0ms
  default_rate: 10gbit
  default_loss: 0%
  constraints:
    - src: grp1
      dst: grp2
      loss: 100%
      symetric: true
      network: 'network_interface'
```
These rules use traffic control to drop all messages (`loss:100%`) on the nodes of grp1 going go or coming from the ip adresses of grp2 nodes and vice-versa (`symetric:true`). This only concerns the `network_interface` vLAN, which means it is only for one NIC (used by OpenStack).

We can nonetheless represent the topology depicted in Figure [3](#os_topo), as far as OpenStack is concerned:
<figure id="os_topo">
<img src='{{ "assets/what-about-network-splits/openstack_topology.svg" | absolute_url }}' alt="openstack topology">
<figcaption style="text-align:center"><span class="figure-number">Figure 3: </span>Openstack topology</figcaption>
</figure>



# Results #

We deployed our topology and pinged the VMs from one another (East-West) or the public network (North-South) to ensure everything worked as intended. We then cut the network using `enos tc`. This had several consequences:  TODO
1. On the horizon dashboard and through the command `openstack TODO`, compute1 became down after a few seconds
2. The VMs became unattainable through the command `openstack server ssh`
3. The VMs were not marked as down or shutoff on the horizon dashboard TODO: command line

We will now detail more about the results.


## A bit of pinging ##

### L3 Full East-West/North-South ###

<figure id="L3_full">
<img src='{{ "assets/what-about-network-splits/L3_full.svg" | absolute_url }}' alt="L3 full">
<figcaption style="text-align:center"><span class="figure-number">Figure 6: </span>L3 Full</figcaption>
</figure>


| Domain | Colocation | Ping type   |  Source         | Destination    | Result |
| ------ | ---------- | ----------- | --------------- | -------------- | ------ |
| L3     | full       | East-West   |  VM1 (C1-Nw1)   | VM4 (C2-Nw2)   | <span style="color:red">X</span>      |
| L3     | full       | East-West   |  VM4 (C2-Nw2)   | VM1 (C1-Nw1)   | <span style="color:red">X</span>      |
| L3     | full       | North-South |  VM1 (C1-Nw1)   | 8.8.8.8        | <span style="color:red">X</span>      |
| L3     | full       | North-South |  VM4 (C2-Nw1)   | 8.8.8.8        | <span style="color:green">V </span>     |

As expected, compute1 is unreachable so no communication were possible through the VMs.


### L3 Dense East-West/North-South ###

<figure id="L3_dense">
<img src='{{ "assets/what-about-network-splits/L3_dense.svg" | absolute_url }}' alt="L3 dense">
<figcaption style="text-align:center"><span class="figure-number">Figure 7: </span>L3 Dense</figcaption>
</figure>

| Domain | Colocation | Ping type   |  Source         | Destination    | Result |
| ------ | ---------- | ----------- | --------------- | -------------- | ------ |
| L3     | dense      | East-West   |  VM1 (C1-Nw1)   | VM2 (C1-Nw2)   | <span style="color:red">X</span>      |
| L3     | dense      | East-West   |  VM2 (C1-Nw2)   | VM1 (C1-Nw1)   | <span style="color:red">X</span>      |
| L3     | dense      | North-South |  VM1 (C1-Nw1)   | 8.8.8.8        | <span style="color:red">X</span>      |
| L3     | dense      | North-South |  VM2 (C1-Nw2)   | 8.8.8.8        | <span style="color:red">X</span>      |

As previously, since both VMs are on the unreachable compute, we can not get any ping.


### L2 Full East-West/North-South ###

<figure id="L2_full">
<img src='{{ "assets/what-about-network-splits/L2_full.svg" | absolute_url }}' alt="L2 full">
<figcaption style="text-align:center"><span class="figure-number">Figure 4: </span>L2 Full</figcaption>
</figure>

| Domain | Colocation | Ping type   |  Source         | Destination    | Result |
| ------ | ---------- | ----------- | --------------- | -------------- | ------ |
| L2     | full       | East-West   |  VM1 (C1-Nw1)   | VM3 (C2-Nw1)   | <span style="color:red"><span style="color:red">X</span></span>      |
| L2     | full       | East-West   |  VM3 (C2-Nw1)   | VM1 (C1-Nw1)   | <span style="color:green">V </span>     |
| L2     | full       | North-South |  VM1 (C1-Nw1)   | 8.8.8.8        | <span style="color:red">X</span>      |
| L2     | full       | North-South |  VM3 (C2-Nw1)   | 8.8.8.8        | <span style="color:green">V </span>     |

TODO check
Here the VM on the compute2 (VM3) seems to be able to ping the one on compute1 (VM1).

### L2 Dense East-West/North-South ###

This had to be done using a different topology as we did not have a compute that had two VMs on the same network. The principle remains entirely the same.

<figure id="L2_dense">
<img src='{{ "assets/what-about-network-splits/L2_dense.svg" | absolute_url }}' alt="L2 dense">
<figcaption style="text-align:center"><span class="figure-number">Figure 5: </span>L3 Dense</figcaption>
</figure>


| Domain | Colocation | Ping type   |  Source         | Destination    | Result |
| ------ | ---------- | ----------- | --------------- | -------------- | ------ |
| L2     | dense      | East-West   |  VM1 (C1-Nw1)   | VM2 (C1-Nw1)   | <span style="color:red">X</span>      |
| L2     | dense      | East-West   |  VM2 (C1-Nw1)   | VM1 (C1-Nw1)   | <span style="color:red">X</span>      |
| L2     | dense      | North-South |  VM1 (C1-Nw1)   | 8.8.8.8        | <span style="color:red">X</span>      |
| L2     | dense      | North-South |  VM2 (C1-Nw1)   | 8.8.8.8        | <span style="color:red">X</span>      |

As for the previous dense experiments, nothing can be done since we can not reach the VMs on compute1.
