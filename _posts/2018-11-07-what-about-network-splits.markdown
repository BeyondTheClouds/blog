---
layout: post
title: What about network splits
date: 2018-11-07
author: Marie Delavergne, Ronan-Alexandre Cherrueau, Adrien Lebre
categories: [OpenStack, network partitions]
tags: [OpenStack, network partitions, network splits, edge]
---


# Living on the edge

Modern applications in the realm of the internet, such as the internet of things, compel the development for edge infrastructures. The concept of edge computing is to distribute the computation on a large number of edge devices, closer to the places where the data are collected.
We define this as hundreds of auto-managed and geo-distributed micro data center of dozens of servers. Since they are distributed all over the globe, one can expect to have different latency and bandwith across the network. But what can also be expected and have nonetheless unexpected consequences are network partitions that could happen over such a distributed network.

A partition occurs when the link to a resource is severed and this resource becomes isolated from the others. This resource can be a node from a database, a compute node, an entire data center, etc.

The goal of this study is to shed some light on the behavior of OpenStack when network partition occurs during


# Base Openstack configuration

# Network topology

<figure>
<img src='{{ "assets/what-about-network-splits/network_topology.svg" | absolute_url }}' alt="Network topology">
	<figcaption style="text-align:center"><span class="figure-number">Figure 1: </span>Network topology</figcaption>
</figure>



# Results

<figure>
<img src='{{ "assets/what-about-network-splits/L2_full.svg" | absolute_url }}' alt="L2 full">
<figcaption style="text-align:center"><span class="figure-number">Figure 2: </span>L2 Full</figcaption>
</figure>

| Domain | Colocation | Ping type   |  Source         | Destination    | Result |
| ------ | ---------- | ----------- | --------------- | -------------- | ------ |
| L2     | full       | East-West   |  VM1 (C1-Nw1)   | VM2 (C2-Nw1)   | <span style="color:red"><span style="color:red">X</span></span>      |
| L2     | full       | East-West   |  VM2 (C2-Nw1)   | VM1 (C1-Nw1)   | <span style="color:green">V </span>     |
| L2     | full       | North-South |  VM1 (C1-Nw1)   | 8.8.8.8        | <span style="color:red">X</span>      |
| L2     | full       | North-South |  VM2 (C2-Nw1)   | 8.8.8.8        | <span style="color:green">V </span>     |

<figure>
<img src='{{ "assets/what-about-network-splits/L2_dense.svg" | absolute_url }}' alt="L2 dense">
<figcaption style="text-align:center"><span class="figure-number">Figure 3: </span>L3 Dense</figcaption>
</figure>


| Domain | Colocation | Ping type   |  Source         | Destination    | Result |
| ------ | ---------- | ----------- | --------------- | -------------- | ------ |
| L2     | dense      | East-West   |  VM1 (C1-Nw1)   | VM2 (C1-Nw1)   | <span style="color:red">X</span>      |
| L2     | dense      | East-West   |  VM2 (C1-Nw1)   | VM1 (C1-Nw1)   | <span style="color:red">X</span>      |
| L2     | dense      | North-South |  VM1 (C1-Nw1)   | 8.8.8.8        | <span style="color:red">X</span>      |
| L2     | dense      | North-South |  VM2 (C1-Nw1)   | 8.8.8.8        | <span style="color:red">X</span>      |

<figure>
<img src='{{ "assets/what-about-network-splits/L3_full.svg" | absolute_url }}' alt="L3 full">
<figcaption style="text-align:center"><span class="figure-number">Figure 4: </span>L3 Full</figcaption>
</figure>


| Domain | Colocation | Ping type   |  Source         | Destination    | Result |
| ------ | ---------- | ----------- | --------------- | -------------- | ------ |
| L3     | full       | East-West   |  VM1 (C1-Nw1)   | VM2 (C2-Nw2)   | <span style="color:red">X</span>      |
| L3     | full       | East-West   |  VM2 (C2-Nw2)   | VM1 (C1-Nw1)   | <span style="color:red">X</span>      |
| L3     | full       | North-South |  VM1 (C1-Nw1)   | 8.8.8.8        | <span style="color:red">X</span>      |
| L3     | full       | North-South |  VM2 (C2-Nw1)   | 8.8.8.8        | <span style="color:green">V </span>     |


<figure>
<img src='{{ "assets/what-about-network-splits/L3_dense.svg" | absolute_url }}' alt="L3 dense">
<figcaption style="text-align:center"><span class="figure-number">Figure 5: </span>L3 Dense</figcaption>
</figure>

| Domain | Colocation | Ping type   |  Source         | Destination    | Result |
| ------ | ---------- | ----------- | --------------- | -------------- | ------ |
| L3     | dense      | East-West   |  VM1 (C1-Nw1)   | VM2 (C1-Nw2)   | <span style="color:red">X</span>      |
| L3     | dense      | East-West   |  VM2 (C1-Nw2)   | VM1 (C1-Nw1)   | <span style="color:red">X</span>      |
| L3     | dense      | North-South |  VM1 (C1-Nw1)   | 8.8.8.8        | <span style="color:red">X</span>      |
| L3     | dense      | North-South |  VM2 (C1-Nw2)   | 8.8.8.8        | <span style="color:red">X</span>      |
