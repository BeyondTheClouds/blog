---
layout: post
title: One paper accepted "Scalability and Locality Awareness of Remote Procedure Calls" (CloudCom 2018)
date: 2018-09-27
author: Javier Rojas Balderrama and  Matthieu Simonin
categories: OpenStack CockroachDB
---

{::options parse_block_html="true" /}
<div class="abstract">
Cloud computing depends on communication mechanisms implying location
transparency. Transparency is tied to the cost of ensuring scalability and an
acceptable request responses associated to the locality. Current
implementations, as in the case of OpenStack, mostly follow a centralized
paradigm but they lack the required service agility that can be obtained in
decentralized approaches.
In an edge scenario, the communicating entities of an application can be
dispersed. In this context we focus our study on the inter-process
communication of Openstack when its agents are geo-distributed. More precisely
we are interested in the different Remote Procedure Calls (RPCs)
implementations of OpenStack and their behaviours with regards to three
classical communication patterns: anycast, unicast and multicast. We discuss
how the communication middleware can align with the geo-distribution of the RPC
agents regarding two key factors : scalability and locality. This work
contributes to the growing interest of adapting existing cloud computing
infrastructures to edges models.

</div>
{::options parse_block_html="false" /}
