---
title: Digest Rules
weight: 40
aliases: ["/administration/digest-rules", "/ref/digest-rules"]
related:
  - storage-rule.md
  - ingest-flow.md
---

[Digest nodes]({{< ref "cluster-nodes.md#digest-node" >}}) are responsible for
executing real-time queries and compiling incoming events into segment files.

Whenever parsing is completed new events are placed in a Kafka queue called the
[Digest Queue]({{< ref "ingest-flow.md#parse" >}}).


## Digest Partitions {#partitions}

A cluster will divide data into partitions (or buckets), we cannot know exactly
which partition a given event will be put in -
Partitions are chosen randomly to spread the workload evenly across digest nodes.
These partitions should correlate with the number of kafka `humio-ingest` topic
partitions (which uses a default of 24).

Each partition of that queue must have a node associated to handle the work that
is placed in the partition or they would never be processed and saved.

Humio clusters have a set of rules that associate partitions in the Digest Queue
with the nodes that are responsible for processing events on that queue. These
are called _Digest Rules_.

## Configuring Digest Rules

You can see the current Digest Rules for your own cluster by going to the
Cluster Management Page and selecting the Digest Rules tab on the right-hand side:

{{< figure src="/pages/ingest-flow/digest-rules.png" class="screenshot" caption="The Cluster Management Page showing Digest Rules for a cluster of 3 nodes where each node is assigned to 8 out of 24 digest partitions." >}}

When a node is assigned to at least one digest partition, it is considered to be a Digest Node.

<!-- TODO: Add information about HA -->

__Example Digest Rules__

| Partition ID | Node         |
|--------------|--------------|
| 1            | 1            |
| 2            | 3            |
| 3            | 1            |

The table shows three digest rules. Node `1` will receive 2x more work
than node `3`. This is because `1` is assigned to two partitions while node `3`
is only handling events on partition `2`.

If a node is not assigned to a partition, it will not take part in the digest
phase.

## Increasing Digest Partitions {#increase-digest-partitions}

{{% notice warning %}}
This operation is not reversible. Once you've increased partitions, you cannot
later decrease the number you're using.
{{% /notice %}}

If additional digest nodes are added to a cluster down the road, you will likely
want to increase the number of digest partitions to ensure all digest nodes are
able to consume from at least one partition. To do this, you must first increase the
number of partitions for the kafka `humio-ingest` topic. Increasing the number of
kafka partitions for a topic involves logging into one of your kafka nodes and
running this command:

```shell
/usr/lib/kafka/bin/kafka-topics.sh --alter --zookeeper localhost:2181 --topic humio-ingest --partitions 28
```
{{% notice note %}}
The path to `kafka-topics.sh` may vary depending on installation type. Docker containers
may have this at `/products/kafka/bin`. Ansible-based installs will have it at the above
path. If you installed this by hand, use whichever path kafka is installed to.
{{% /notice %}}

If zookeeper is running on the kafka machine, use localhost for the zookeeper address,
otherwise specify its actual hostname or IP. Replace `28` with whatever total
partition count you're after (note that you're specifying the total and not the number
of partitions being added). After running the command, you should see output that says
"Adding partitions succeeded!".

Next you need to create the digest partitions in Humio using the Cluster Management API.
Connect to a Humio node and run this command to get the current partition layout (be sure
you update the path to the local admin token if necessary):

```shell
curl -v -H "Content-Type: application/json" -H "Authorization: Bearer $(cat /data/humio-data/local-admin-token.txt)" http://127.0.0.1:8080/api/v1/clusterconfig/ingestpartitions
```

You should get JSON output similar to this (the array for hosts will reflect which
nodes you have assigned to each partition in the Humio administration interface):

```json
[{"hosts":[1],"partition":0},{"hosts":[1],"partition":1},{"hosts":[1],"partition":2},{"hosts":[1],"partition":3},{"hosts":[1],"partition":4},{"hosts":[1],"partition":5},{"hosts":[1],"partition":6},{"hosts":[1],"partition":7},{"hosts":[1],"partition":8},{"hosts":[1],"partition":9},{"hosts":[1],"partition":10},{"hosts":[1],"partition":11},{"hosts":[1],"partition":12},{"hosts":[1],"partition":13},{"hosts":[1],"partition":14},{"hosts":[1],"partition":15},{"hosts":[1],"partition":16},{"hosts":[1],"partition":17},{"hosts":[1],"partition":18},{"hosts":[1],"partition":19},{"hosts":[1],"partition":20},{"hosts":[1],"partition":21},{"hosts":[1],"partition":22},{"hosts":[1],"partition":23}]
```

Copy that JSON output and add addition entries to the array for the added partitions. Now
POST that JSON payload back to the endpoint. In the case above where the count is being
increased from 24 to 28, that would look like this:

```shell
curl -v -H "Content-Type: application/json" -H "Authorization: Bearer $(cat /data/humio-data/local-admin-token.txt)" http://127.0.0.1:8080/api/v1/clusterconfig/ingestpartitions -d '[{"hosts":[1],"partition":0},{"hosts":[1],"partition":1},{"hosts":[1],"partition":2},{"hosts":[1],"partition":3},{"hosts":[1],"partition":4},{"hosts":[1],"partition":5},{"hosts":[1],"partition":6},{"hosts":[1],"partition":7},{"hosts":[1],"partition":8},{"hosts":[1],"partition":9},{"hosts":[1],"partition":10},{"hosts":[1],"partition":11},{"hosts":[1],"partition":12},{"hosts":[1],"partition":13},{"hosts":[1],"partition":14},{"hosts":[1],"partition":15},{"hosts":[1],"partition":16},{"hosts":[1],"partition":17},{"hosts":[1],"partition":18},{"hosts":[1],"partition":19},{"hosts":[1],"partition":20},{"hosts":[1],"partition":21},{"hosts":[1],"partition":22},{"hosts":[1],"partition":23},{"hosts":[1],"partition":24},{"hosts":[1],"partition":25},{"hosts":[1],"partition":26},{"hosts":[1],"partition":27}]'
```

You can confirm that the new digest partitions are created by checking the Digest rules
in the Cluster Nodes area of the Administration area in Humio.

## Removing a Digest Node {#removal}

When [removing a digest node]({{< ref "removing-a-node.md" >}}) it is important
that you first assign another cluster node to take the work in any digest partitions
that the node is assigned to. If you do not do this, there will be no one to process
the incoming events and it will stack up, and in the worst case data might get lost.

## High Availability

Humio has High-Availability support for digest nodes.

The short version: Run a cluster of at least 3 nodes. Make sure to
list 2 independent nodes on each partition, both for digest partitions
and for storage partitions, then optionally apply those new storage
rules to all existing segments to allow searching existing data too if
a node should stop working. If any data is present on only one node,
then queries will get warnings saying "Segment files missing, partial
results". Make your http-load-balancer know about more tha one node in
the cluster, preferably all of them. Make sure you cluster has the CPU
power required to handle the fail-over: It should be running at well
below 50% utilization normally.

This feature preumes that you Kafka cluster is properly configured for
high-availability: All topics in use by Humio needs to have at least 2
replica on independent server nodes.

### How High Availability in Humio works

The design aims for high performance during normal operation at the
cost of a penalty to be paid when a fail-over situation happens.

Turn on high availability on ingest by configuring all digest
partitions to have two or more nodes listed. The first live node in
the list will handle that partition during normal operation. If the
primary nodes stops, then the next one in the list will take on the
responsibility of handling the digest on that partition. If the
primary returns at some point, then the primary will resume
responsibility for that partition. This happend independently for all
digest partitions.

To get proper results in all live queries, all live queries get
restarted when a node stops

The node that then takes on the digest processing for has to start
quite a way back in time on the digest queue, processing all events
(again probably) that have not yet been written to segments replicated
sufficiently to other nodes. Care is taken to ensure that events only
get included once in a query, regardless of the re-processing in
digest, including not duplicating them once the failed node returns
and has those events as well.

The fail-over process can take quite while to complete, depending on
how much data has been ingested during the latest 30 minutes or
so. The cluster typically needs to start 30 minutes back in time on a
fail-over. While it does so, the event latency of events coming in
will be higher than normal on the partitions that have been
temporarily reassigned, starting at up to 1800 - 2000 seconds. The
cluster thus needs to have sufficient ample CPU ressources to catch up
from such a latency fairly quickly. To aid in that process, queries
get limited to 50% of their normal CPU time while the digest latency
is high.

Fail-over makes the cluster start much further back in time compared
to a controlled handover between two live nodes. When reassigning
partitions with the nodes active, the nodes cooperate to reduce the
number of events being re-processed to a minimum, making the latency
in ingested events during that typically stay below 10 seconds.

The decision to go with the 30 minutes as target failover cost is a
compromise between two conflicting concerns. If the value is raised
further, the fail-over will take even longer, which is usually not
desirable. But raising it reduces the number of small segments
generated, since segments get flushed after at most that amount of
time. Reducing to 30 minute interval to say 5 minutes would make the
fail-over happoen much faster, but at the cost of normal operation, as
there would be 12 * 24 = 288 segments in each datasource per day,
compared to the 2 * 24 = 48 with the current value. The cost of having
all these extra segments would slow down normal operation somewhat.

There are a number of configuration parameters to control the
tradeoffs. Here they are listed with their defaults.

```
# How long can a mini-segment stay open. How long back is a fail-over likely to go?
FLUSH_BLOCK_SECONDS=1800

# Desired number of blocks (each ~1MB before compression) in a final segment after merge
BLOCKS_PER_SEGMENT=2000

# How long cn the same semgent be the final target of all the minisegments being produced?
MAX_HOURS_SEGMENT_OPEN=24
```
