---
title:  "Increasing the number of tokens in Cassandra 2.2"
date:   2017-10-14 00:00:00 -0600
categories: cassandra vnodes tokens
---
# Overview
I wrote a blog post a few months ago detailing my [experience with Cassandra and tokens]({% post_url 2017-08-04-vnodes-in-cassandra %}). This is a follow-up post on the outcome of that issue. Note: In these posts, I'm using tokens and vnodes interchangeably. The setting in Cassandra I'm referring to is `num_tokens` in `cassandra.yaml`. In the previous post, I discussed a cluster that had been built a few years ago, and was built with the number of tokens at `12`. This setting was causing issues because the data wasn't spread evenly across the nodes. There weren't enough virtual nodes in order for the data to be uniformly distributed.

This issue started becoming more prevalent - and causing more noticeable issues, especially during peak performance testing. During tests, some nodes would be >90% utilized, and other nodes sitting relatively idle. Here is a quick example of the behavior we were seeing:
![alt text](/images/loadavg.png "Load average of Cassandra nodes")

You can see 3 of the nodes are under heavy load - but the rest are fine. Usually hot spots are caused by a data modeling issue. We spent a lot of time reviewing the data model, and eventually came to the conclusion the small number of vnodes was causing the issue.

It became clear we needed a fix - the question was, what? We either needed to increase the number of tokens in a live cluster, or find another option. We looked in to upgrading to Cassandra 3 - they have a new [token allocation algorithm](https://www.datastax.com/dev/blog/token-allocation-algorithm) that is much better at distribution. However, we decided this was too much risk before the holiday season (our most important time by far), and we weren't 100% sure an upgrade would rebalance the existing tokens. We decided to move forward with increasing the number of vnodes (rather than upgrading.)

# Process
The most important thing during this change was the cluster had to stay up, healthy, and fast. Everything we did needed to be deliberate and safe, as we have production traffic flowing through. Here are the options we came up with:
1. Build a new cluster, and mirror the data to it
2. Decommission a node, delete all the data, and rejoin it with a higher number of tokens
3. Decommission a node, delete all the data, and rejoin it with a higher number of tokens, but in a *new* Cassandra datacenter

The first option we considered was building a second cluster and mirroring all our data. This was thrown out pretty quickly for a couple reasons:
* Mirroring the data was not as easy as expected. Keeping this cluster in sync with the live one added more risk than we were willing to take in this scenario
* We either needed to poach nodes from the live cluster to use in the new cluster, or provision additional servers to make the cluster

We decided we did not want to be constrained by either of those problems. The second option sounded more promising, but after experimenting it was clear it wouldn't work. The reason is because num_tokens is used to control data distribution. If you had a cluster with 9 nodes at `num_tokens: 12` and 1 node at `num_tokens: 256` - the node at 256 would own almost all of the data because it has more token ranges than all the other nodes combined. You can see this in a simple test, just going from num_tokens: 12 -> 24:
```
--  Address       Load       Tokens       Owns (effective)  Host ID                               Rack
UN  192.168.1.1   7.29 GB    12           12.9%             9f4b145d-1f1a-4f19-8772-894cdd7687b2  dc1
UN  192.168.1.2   16.44 GB   12           26.3%             cdaea723-926b-4583-98ab-1b9307f65cf6  dc1
UN  192.168.1.3   24.29 GB   24           60.9%             1d7793da-14a7-4a29-9d93-c7829925e62d  dc1
```

A node with double tokens owns double what the next highest does. A node with such a large increase to 256 would own much more. This would make our issue even worse short-term, and could potentially cause the entire cluster to fail when that single node couldn't handle the traffic.

The third option turned out to be the best. We would decommission the nodes one-by-one, and slowly add them to a new Cassandra datacenter. This would allow us to separate the changes we were making from the live datacenter. Luckily our cluster was idle enough during normal days we could stand to lose a few nodes to the new datacenter before flipping over.

Here's a high-level overview of the process we followed:
1. `nodetool decommission` on the node we want to convert
2. Delete all of the node data, including data directory, commit log, and saved_caches.
3. Change the datacenter in the relevant property file (we use GossipingPropertyFile snitch, so `cassandra-rackdc.properties`) to the new datacenter. We called our new datacenter $OLDDC-1 (not very creative, but it works)
4. Bootstrap the node as a "new" node - it won't stream any data at first because we haven't update the keyspace replication factor yet.
  * note: we decided not to change the replication factor until a few nodes existed because we've seen issues with the driver in the past when it tries to calculate a token range for non-existent datacenters. These issues have since been fixed, but we wanted to be as safe as possible.
5. Repeat this process on 2 more nodes, for a total of 3 in the new datacenter. Your status should look something like:

```
Datacenter: DC
================
--  Address       Load       Tokens       Owns    Host ID                               Rack
UN  10.86.1.1   307.16 GB  12           ?       b28f5ba3-be36-4f97-a408-d858091315b9  a
UN  10.86.1.2   437.23 GB  12           ?       ce21bce1-8e75-493a-8184-a7d1284e25bd  a
UN  10.86.1.3   360.73 GB  12           ?       e90e3c24-98de-43aa-98c9-47daa6944199  a
UN  10.86.1.4   350.62 GB  12           ?       aa6c6f8f-6119-413f-a1d6-e5920128936e  a

Datacenter: DC-1
================
Status=Up/Down
|/ State=Normal/Leaving/Joining/Moving
--  Address       Load       Tokens       Owns    Host ID                               Rack
UN  10.86.1.5   241.27 GB  256          ?       3c3d4959-7181-4e96-9ee3-f3448d481e92  a
UN  10.86.1.6   284.46 GB  256          ?       7728303f-9c26-4fba-b82f-d32327836d92  a
UN  10.86.1.7   298.06 GB  256          ?       40922f30-d0fc-44c8-927c-bf69cdcf5e95  a
```

6. alter the replication on the keyspace(s) you want to replicate to this new datacenter.
  * make sure to remember the `system_auth` keyspace.
7. Run `nodetool rebuild` (repair would work too) on these 3 nodes
  * make sure this finishes and they get their full range of data

Once these steps are completed, it's just a matter of converting a large enough number of nodes to support the traffic in the new datacenter. Once that is done, a change needs to be made on the application side to point to the new datacenter. Note: the [DCAwareRoundRobinPolicy](http://docs.datastax.com/en/drivers/java/2.2/com/datastax/driver/core/policies/DCAwareRoundRobinPolicy.html) will need to be used in order to change what datacenter traffic is going to. Once traffic is switched to the new DC, the rest of the nodes in the old datacenter can be decommissioned and converted.

# Results
We recently finished this on our production cluster of 24 nodes. We have 2 datacenters, 12 nodes each. We switched app traffic to the new datacenter once we got to 6 nodes. Everything went very smoothly. The biggest issue we had is the decommission/bootstrap of nodes took longer than expected, but that was easily worked around. We were able to increase that somewhat by bumping up stream throughput on the nodes as well.

Here are some of the results we saw:
* data is much more evenly distributed. Here is a load graph from a recent perf run:
![alt text](/images/loadavg-updated.png "Load average of Cassandra nodes after vnode change")
* we were able to scale much higher because we are not bottlenecked by single nodes anymore
* we're now able to hit our peak performance numbers :)
