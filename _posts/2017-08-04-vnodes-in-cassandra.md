---
title:  "vnodes between datacenters in Cassandra"
date:   2017-08-04 00:00:00 -0600
categories: cassandra vnodes TIL
---
Since I promised to write about what I learn in my day-to-day work, here's something I ran in to recently while expanding a Cassandra cluster. Here's the scenario:

We have a Cassandra cluster at Target (version 2.2.9) that consists of two datacenters, `dc1` and `dc2`. These datacenters have been up and running in production for a few years, and each dc has 12 [vnodes](https://www.datastax.com/dev/blog/virtual-nodes-in-cassandra-1-2) (this is the `num_tokens` yaml value in the `cassandra.yaml` configuration file.) This number is important. Recently, we were tasked with adding another datacenter, let's call it `dc3.` Most of our other clusters at Target use the default of `256` for the number of vnodes value. This is where the mistake (and learning) came in. I was under the impression that the vnodes value really only mattered internally to each datacenter. In the past, I have used different values here for 2 reasons:
* partition more data on larger (more RAM, CPU) nodes
* partition less data on smaller nodes

I was under the assumption that between the datacenters the number of vnodes didn't come in to play. That turned out to be wrong.

When we tried to bootstrap the third datacenter, we did it using the default number of vnodes, `256`. The node would start, join the cluster, stream some data, and finish like normal. The problem was - it would only stream about 5% of the data. It turns out this was because the node saw the other datacenter having only 12 vnodes, and assumed that node only "owned" 5% of the data. The catch was that it didn't try to go out and get the rest of the data from anywhere (even if it might "fail"), it just finished and marked itself fully "up."

It seems that in order to bootstrap a node with a larger number of vnodes than exists the cluster, you need to do one of two things:
1. Increase the number of vnodes on the existing nodes, following [these painful steps.](https://stackoverflow.com/questions/32416642/cassandra-vnodes-can-i-lower-the-number-on-slower-nodes-and-expect-rebalancing) This probably isn't feasible for a production cluster.
2. Add the new datacenter with the max number of vnodes equal to the max number of vnodes in the other datacenters. For example. in this case, I couldn't use a `num_tokens` value above `12`, since the other datacenters maximum value is `12`.

In the future, we won't be bringing up any clusters with a smaller number than `256`, and we'll make sure to think about this when building new clusters that span multiple datacenters. The default is good in almost every case, and you can go smaller from there. It seems like it's much harder to increase once you've set `num_tokens`.

**Caveat:** I couldn't find any Jira tickets covering this, but it may already be fixed in newer versions. I am continuing to research this issue, but for now I could not find anything.

---
### Further reading:
* [Virtual Nodes](http://docs.datastax.com/en/cassandra/3.0/cassandra/architecture/archDataDistributeVnodesUsing.html)
* [How data is distributed across a cluster](http://docs.datastax.com/en/cassandra/3.0/cassandra/architecture/archDataDistributeDistribute.html)
