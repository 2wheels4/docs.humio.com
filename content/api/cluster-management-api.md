---
title: "Cluster Management API"
weight: 500
---

This page provides information about the HTTP API for managing on-premises installations of Humio.

All requests except the status endpoint require **root-level access**.
See [API token for local root access]({{< ref "root-access.md#root-token" >}}).

You can see the [Cluster Administration Documentation]({{< ref "administration/_index.md" >}})
for more details on how to perform common tasks like adding and removing nodes from
a cluster.

## Available Endpoints

| Endpoint | Method | Description |
| -------- | ------ | ----------- |
| `/api/v1/clusterconfig/members`                                                    | [GET](#list-cluster-members)                                            | List cluster nodes                                   |
| `/api/v1/clusterconfig/members/$NODE_ID`                                           | [GET, PUT](#modifying-a-node-in-your-cluster)                           | Get or modify a node in your cluster                 |
| `/api/v1/clusterconfig/members/$NODE_ID`                                           | [DELETE](#deleting-a-node-from-your-cluster)                            | Deleting a node from your cluster                    |
| `/api/v1/clusterconfig/segments/partitions/setdefaults`                            | [POST](#applying-default-partition-settings)                            | Applying default partition settings                  |
| `/api/v1/clusterconfig/segments/partitions`                                        | [GET, POST](#querying-and-assigning-storage-partitions-to-nodes)        | Querying and assigning storage partitions to nodes   |
| `/api/v1/clusterconfig/segments/partitions/set-replication-defaults`               | [POST](#assigning-default-storage-partitions-to-nodes)                  | Assigning default storage partitions to nodes        |
| `/api/v1/clusterconfig/segments/distribute-evenly`                                 | [POST](#moving-existing-segments-between-nodes)                         | Moving existing segments between nodes               |
| `/api/v1/clusterconfig/segments/prune-replicas`                                    | [POST](#pruning-replicas-when-reducing-replica-setting)                 | Pruning replicas when reducing replica setting       |
| `/api/v1/clusterconfig/segments/distribute-evenly-reshuffle-all`                   | [POST](#moving-existing-segments-between-nodes)                         | Moving existing segments between nodes               |
| `/api/v1/clusterconfig/segments/distribute-evenly-to-host/$NODE_ID`                | [POST](#moving-existing-segments-between-nodes)                         | Moving existing segments between nodes               |
| `/api/v1/clusterconfig/segments/distribute-evenly-from-host/$NODE_ID`              | [POST](#moving-existing-segments-between-nodes)                         | Moving existing segments between nodes               |
| `/api/v1/clusterconfig/ingestpartitions`                                           | [GET, POST](#digest-partitions)                                         | Get/Set digest partitions                            |
| `/api/v1/clusterconfig/ingestpartitions/setdefaults`                               | [POST](#digest-partitions)                                              | Set digest partitions defaults                       |
| `/api/v1/clusterconfig/ingestpartitions/distribute-evenly-from-host/$NODE_ID`      | [POST](#digest-partitions)                                              | Move digest partitions from node                     |
| `/api/v1/clusterconfig/kafka-queues/partition-assignment`                          | [GET, POST](#managing-kafka-queue-settings)                             | Managing kafka queue settings                        |
| `/api/v1/clusterconfig/kafka-queues/partition-assignment/set-replication-defaults` | [POST](#managing-kafka-queue-settings)                                  | Managing kafka queue settings                        |
| `/api/v1/listeners`                                                                | [GET,POST](#adding-a-ingest-listener-endpoint)                          | Add tcp listener (used for Syslog)                   |
| `/api/v1/listeners/$ID`                                                            | [GET,DELETE](#adding-a-ingest-listener-endpoint)                        | Add tcp listener (used for Syslog)                   |
| `/api/v1/repositories/$REPOSITORY_NAME/taggrouping`                                   | [GET,POST](#setup-grouping-of-tags)                                     | Setup grouping of tags                               |
| `/api/v1/repositories/$REPOSITORY_NAME/datasources/$DATASOURCEID/autosharding`       | [GET,POST,DELETE](#configure-auto-sharding-for-high-volume-datasources) | Configure auto-sharding for high-volume datasources. |
| `/api/v1/status`       							     | [GET](#status-endpoint) 						       | Get status and version of node 	 	      |


## List cluster members

```
GET    /api/v1/clusterconfig/members
```

Example:

```shell
curl -H "Authorization: Bearer $TOKEN" "$BASEURL/api/v1/clusterconfig/members"
```

## Adding a node to your cluster

[See: Adding a node]({{< ref "adding-a-node.md" >}})


## Modifying a node in your cluster

You can fetch / re-post the object representing the node in the cluster using GET/PUT requests.
$NODE_ID is the integer-id of the new node.

```
GET    /api/v1/clusterconfig/members/$NODE_ID
PUT    /api/v1/clusterconfig/members/$NODE_ID
```

Example:

```shell
curl -H "Authorization: Bearer $TOKEN" "$BASEURL/api/v1/clusterconfig/members/1" > node-1.json
curl -XPUT -d @node-1.json -H "Content-Type: application/json" -H "Authorization: Bearer $TOKEN" "$BASEURL/api/v1/clusterconfig/members/1"
```

outputs:

```json
{"vhost":1,"uuid":"7q2LwHv6q3C5jmdGj3EYL1n56olAYcQy","internalHostUri":"$BASEURL","displayName":"host-1"}
```

You can edit the fields internalHostUri and displayName in this structure and POST the resulting changes back to the server, preserving the vhost and uuid fields.

## Deleting a node from your cluster

[See: Removing a node]({{< ref "removing-a-node.md" >}})

If the host does not have any segments files, and no assigned partitions, there is no data loss when deleting a node.

```
DELETE    /api/v1/clusterconfig/members/$NODE_ID
```

Example:

```shell
curl -XDELETE -H "Authorization: Bearer $TOKEN" "$BASEURL/api/v1/clusterconfig/members/1"
```

It is possible to drop a host, even if it has data and assigned partitions, by adding the query parameter "accept-data-loss" with the value "true".

{{% notice warning %}}
This silently drops your data.
{{% /notice %}}

Example:

```shell
curl -XDELETE -H "Authorization: Bearer $TOKEN" "$BASEURL/api/v1/clusterconfig/members/1?accept-data-loss=true"
```

## Applying default partition settings

This is a shortcut to getting all members of a cluster to have the same share of the load on both [digest and storage partitions]({{< ref "ingest-flow.md" >}}).

```
POST   /api/v1/clusterconfig/partitions/setdefaults
```

Example:

```shell
curl -XPOST -H "Authorization: Bearer $TOKEN" "$BASEURL/api/v1/clusterconfig/partitions/setdefaults"
```

## Querying and assigning storage partitions to nodes

We recommend you read the [docs section about storage rules]({{< ref "storage-rules.md" >}}).

When a data segments is complete, the server select the host(s) to place the segment on by looking up a segment-related key in the storage partition table.
The partitions map to a set of nodes. All of these nodes are then assigned as owners of the segment, and will start getting their copy shortly after.

You can modify the storage partitions at any time.
Any number of partitions larger than the number of nodes is allowed, but the recommended the number of storage partitions is 24 or similar fairly low number.
There is no gain in having a large number of partitions.

Existing segments are not moved when re-assigning partitions. Partitions only affect segments completed after they are POST'ed.

```
GET    /api/v1/clusterconfig/segments/partitions
POST   /api/v1/clusterconfig/segments/partitions
```

Example:

```shell
curl -H "Authorization: Bearer $TOKEN" "$BASEURL/api/v1/clusterconfig/segments/partitions" > segments-partitions.json
curl -XPOST -d @segments-partitions.json -H "Content-Type: application/json" -H "Authorization: Bearer $TOKEN" "$BASEURL/api/v1/clusterconfig/segments/partitions"
```

## Assigning default storage partitions to nodes

When the set of nodes has been modified, you likely want to make the storage partitions distribute the storage load evenly among the current set of nodes.
The following API allows doing that, while also selecting the number of replicas to use.

Any number of partitions larger than the number of nodes is allowed, but the recommended the number of storage partitions is 24 or similar fairly low number.
There is no gain in having a large number of partitions.

The number of replicas must be at least one, and at most the number of nodes in the cluster. The replicas selects how many nodes should keep a copy of each completed segment.

```
POST   /api/v1/clusterconfig/segments/partitions/set-replication-defaults
```

Example:

```shell
echo '{ "partitionCount": 7, "replicas": 2 }' > settings.json
curl -XPOST -d @settings.json -H "Content-Type: application/json" -H "Authorization: Bearer $TOKEN" "$BASEURL/api/v1/clusterconfig/segments/partitions/set-replication-defaults"
```

## Pruning replicas when reducing replica setting

If the number of replicas has been reduced, existing segments in the cluster do not get their replica count reduced. In order to reduce the number of replicas on existing segments, invoke this:

```
POST   /api/v1/clusterconfig/segments/prune-replicas
```

```shell
curl -XPOST -H "Authorization: Bearer $TOKEN" "$BASEURL/api/v1/clusterconfig/segments/prune-replicas"
```

## Moving existing segments between nodes

There is API for taking the actions moving the existing segments between nodes.

1.  Moving segments so that all nodes have their "fair share" of the segments, as stated in storage partitioning setting, but as mush as possible leaving segments where they are.
    It's also possible to apply the current partitioning scheme to all existing segments, possibly moving every segment to a new node.

1.  It's possible to move all existing segments off a node.
    If that node is not assigned any partitions at all (both storage and ingest kinds), this then relieves the node of all duties, preparing it to be deleted from the cluster.

1.  If a new node is added, and you want it to take its fair share of the current stored data, use the "distribute-evenly-to-host" variant.

```
POST   /api/v1/clusterconfig/segments/distribute-evenly
POST   /api/v1/clusterconfig/segments/distribute-evenly-reshuffle-all
POST   /api/v1/clusterconfig/segments/distribute-evenly-to-host/$NODE_ID
POST   /api/v1/clusterconfig/segments/distribute-evenly-from-host/$NODE_ID
Optional; Add a "percentage=[0..100]" query parameter to only apply the action to a fraction of the full set.
```

Examples:

```shell
curl -XPOST -H "Content-Type: application/json" -H "Authorization: Bearer $TOKEN" "$BASEURL/api/v1/clusterconfig/segments/distribute-evenly"
curl -XPOST -H "Content-Type: application/json" -H "Authorization: Bearer $TOKEN" "$BASEURL/api/v1/clusterconfig/segments/distribute-evenly-reshuffle-all?percentage=3"
curl -XPOST -H "Content-Type: application/json" -H "Authorization: Bearer $TOKEN" "$BASEURL/api/v1/clusterconfig/segments/distribute-evenly-to-host/1"
curl -XPOST -H "Content-Type: application/json" -H "Authorization: Bearer $TOKEN" "$BASEURL/api/v1/clusterconfig/segments/distribute-evenly-from-host/7"
```

## Digest Partitions

These route the incoming data while it is "in progress".

We recommend you read the [docs section about digest rules]({{< ref "digest-rules.md" >}}).

Warning: Do not `POST` to this API unless the cluster is running fine,
with all members connected and active. All digest stops for a few seconds when being applied.

Digest does not start before all nodes are ready, thus if a node is failing, digest does not resume.

1.  GET/POST the setting to hand-edit where each partition goes. You cannot reduce the number of partitions.

1.  Invoke "setdefaults" to distribute the current number of partitions evenly among the known nodes in the cluster

1.  Invoke "distribute-evenly-from-host" to reassign partitions currently assigned to $NODE_ID to the other nodes in the cluster.

```
GET    /api/v1/clusterconfig/ingestpartitions
POST   /api/v1/clusterconfig/ingestpartitions
POST   /api/v1/clusterconfig/ingestpartitions/setdefaults
POST   /api/v1/clusterconfig/ingestpartitions/distribute-evenly-from-host/$NODE_ID
```

Example:

```shell
curl -H "Authorization: Bearer $TOKEN" "$BASEURL/api/v1/clusterconfig/ingestpartitions" > digest-rules.json
curl -XPOST -d @digest-rules.json -H "Content-Type: application/json" -H "Authorization: Bearer $TOKEN" "$BASEURL/api/v1/clusterconfig/ingestpartitions"
```

## Managing Kafka queue settings

The ingest queues are partitions of the Kafka queue named "humio-ingest".
Humio offers an API for editing the Kafka partition to broker assignments this queue.
Note that changes to these settings are applied asynchronously, thus you can get the previous settings, or a mix with the latest settings, for a few seconds after applying a new set.

```
GET    /api/v1/clusterconfig/kafka-queues/partition-assignment
POST   /api/v1/clusterconfig/kafka-queues/partition-assignment
POST   /api/v1/clusterconfig/kafka-queues/partition-assignment/set-replication-defaults
```

Example:

```shell
curl -H "Authorization: Bearer $TOKEN" "$BASEURL/api/v1/clusterconfig/kafka-queues/partition-assignment" > kafka-ingest-partitions.json
curl -XPOST -d @kafka-ingest-partitions.json -H "Content-Type: application/json" -H "Authorization: Bearer $TOKEN" "$BASEURL/api/v1/clusterconfig/kafka-queues/partition-assignment"

echo '{ "partitionCount": 24, "replicas": 2 }' > kafka-ingest-settings.json
curl -XPOST -d @kafka-ingest-settings.json -H "Content-Type: application/json" -H "Authorization: Bearer $TOKEN" "$BASEURL/api/v1/clusterconfig/kafka-queues/partition-assignment/set-replication-defaults"
```

## Adding a ingest listener endpoint

You can ingest events using one of the many [existing integration]({{< ref "integrations/_index.md" >}}) but when your requirements do
not match, perhaps you can supply a stream of events on TCP, separated by line feeds.
This API allows you to create and configure a TCP listener for such events.
Use cases include accepting "rsyslogd forward format" and similar plain-text event streams.

```
GET    /api/v1/listeners
POST   /api/v1/listeners
GET    /api/v1/listeners/$ID
DELETE /api/v1/listeners/$ID
```

If you use [rsyslog for transport of logs](http://www.rsyslog.com/doc/v8-stable/configuration/templates.html#standard-template-for-forwarding-to-a-remote-host-rfc3164-mode)
then this example serves as a starting point:

```
# Example input line on the wire:
<14>2017-08-07T10:57:04.270540-05:00 mgrpc kernel: [   17.920992] Bluetooth: Core ver 2.22
```

Creating a parser accepting rsyslogd forward format: [(How to add a parser)](#create-or-update-parser)

```shell
cat << EOF > create-rsyslogd-rfc3339-parser.json
{ "parser": "^<(?<pri>\\\\d+)>(?<datetimestring>\\\\S+) (?<host>\\\\S*) (?<syslogtag>\\\\S*): ?(?<message>.*)",
  "kind": "regex",
  "parseKeyValues": true,
  "dateTimeFormat": "yyyy-MM-dd'T'HH:mm:ss[.SSSSSS]XXX",
  "dateTimeFields": [ "datetimestring" ]
}
EOF
curl -XPOST \
 -d @create-rsyslogd-rfc3339-parser.json \
 -H "Authorization: Bearer $TOKEN" \
 -H 'Content-Type: application/json' \
 "$BASEURL/api/v1/repositories/$REPOSITORY_NAME/parsers/rsyslogd-rfc3339"
```

Example setting up a listener using the rsyslogd forward format added above:

```shell
cat << EOF > create-rsyslogd-listener.json
{ "listenerPort": 7777,
  "kind": "tcp",
  "dataspaceID": "$REPOSITORY_NAME",
  "parser": "rsyslogd-rfc3339",
  "bindInterface": "0.0.0.0",
  "name": "my rsyslog input",
  "vhost": 1
}
# "bindInterface" is optional. If set, sets local interface to bind on to select network interface.
# "vhost" is optional. If set, only the cluster node with that index binds the port.

EOF
curl -XPOST \
  -d @create-rsyslogd-listener.json \
  -H 'Content-Type: application/json' \
  -H "Authorization: Bearer $TOKEN" \
  '$BASEURL/api/v1/listeners'

curl -H "Authorization: Bearer $TOKEN" "$BASEURL/api/v1/listeners"
curl -H "Authorization: Bearer $TOKEN" "$BASEURL/api/v1/listeners/tcp7777"
```

Listeners also support UDP by setting `kind` to `"udp"`. For UDP, each UDP datagram is
ingested as a single log line (i.e. it is not split by newlines).

It is possible to specify that fields in the incomming events, should be turned into tags.
This can be done by setting `"tagFields": ["fielda", "fieldb"]` when creating a listener. Only use tags like this if you really need it.

To reduce packet loss in bursts of UDP traffic, please increase the maximum allowed receive buffer size for UDP.
Humio will try to increase the buffer to up to 128MB, but will accept whatever the system sets as maximum.

```shell
# To set to 16MB.
sudo sysctl net.core.rmem_max=16777216
```

## Setup grouping of tags

```
GET    /api/v1/repositories/$REPOSITORY_NAME/taggrouping
POST   /api/v1/repositories/$REPOSITORY_NAME/taggrouping
```

Please note that this is a feature for advanced users only.

Humio recommends most users to only use the parser as a tag, in the field `#type`.
This is usually sufficient.

Using more tags may speed up queries on large data volumes, but only works on a bounded value-set for the tag fields.
The speed-up only affects queries prefixed with `#tag=value` pairs that significantly filter out input events.

Tags are the fields with a prefix of `#` that are used internally to do sharding of data into smaller streams
A `datasource` is is created for every unique combination of tag values set by the clients (e.g. data-shippers)
Humio will reject ingested events once a certain number of datasources get created.
The limit is currently 10.000 pr. repository.

For some use cases, such as having the "client IP" from an access log as a tag,
too many different tags will arise.
For such a case, it is necessary to either stop having the field as a tag, or
create a grouping rule on the tag field.
Existing data is not re-written when grouping rules are added or changed.
Changing the grouping rules will thus in it-self create more datasources.

Example: Setting the grouping rules for repository `$REPOSITORY_NAME`
to hash the field `#host` into 8 buckets, and `#client_ip` into 10 buckets.
Note how the field names do not include the `#` prefix in the rules.

```shell
curl $BASEURL/api/v1/repositories/$REPOSITORY_NAME/taggrouping \
  -X POST \
  -H "Authorization: Bearer $TOKEN" \
  -H 'Content-Type: application/json' \
  -d '[ {"field":"host","modulus": 8}, {"field":"client_ip","modulus": 10} ]'
```

Adding a new set of rules using POST replaces the current set.
The previous sets are kept, and if a previous one matches, then the previous one is reused.
The previous rules are kept in the system, but may be deleted by Humio once all datasources referring them has been deleted (through retention settings)

When using grouped tags in the query field, you can expect to get a speed-up of approximately the modulus compared to not including the tags in the query,
provided you use an exact match on the field. If you use a wildcard (`*`) in the value for the grouped tag, the implementation currently scans all
datasources that have a non-empty value for that field and filter the events to only get the results the match the wildcard pattern.

For non-grouped tag fields, using a wildcard at either end of the value string to match is efficient.

Humio also support auto-grouping of tags using the configuration
variables `MAX_DISTINCT_TAG_VALUES` (default is `1000`) and
`TAG_HASHING_BUCKETS` (default is `16`).  Humio checks the number of
distinct values for each key in each tag combination against
`MAX_DISTINCT_TAG_VALUES` at regular intervals.  It this threshold is
exceeded, a new grouping rule is added with the modulus set to the
value set in `TAG_HASHING_BUCKETS`. But only if there is no rule for
that tag key. You can thus configure rules using the API above and
decide the number of buckets there. This is preferable to
auto-detecting, as the auto-detection works after the fact and thus
leaves a large number of unused datasources that will need to get
deleted by retention at some point. The auto-grouping support is meant
as a safety measure to avoid suddenly creating many datasources by
mistake for a single tag key.

IF you happen to read this and is using a hosted Humio instance, please contact support
if you wish to add grouping rules to your repository.

## Importing a repository (not view) from another Humio instance (BETA)

You can import users, dashboards and segments files from another Humio instance.
You need to get a copy of the `/data/humio-data/global-data-snapshot.json` from the origin server.

You also need to copy the segments files that you want to
import. These must be placed in the folder
`/data/humio-data/ready_for_import_dataspaces` using the following
structure:

`/data/humio-data/ready_for_import_dataspaces/dataspace_$ID`

You should copy the files for the repository to the server into
another folder while the copying is happening, and then move it to the
proper name once it's ready. Note the name of the directory uses the
internal ID of the dataspace, which is the directory name in the
source system.

The folder `/data/humio-data/ready_for_import_dataspaces` must be
read+writeable for the humio-user running the server, as it moves the
files to another directory and deletes the imported files when it is
done with them, one at a time.

Example: (Note that you need both NAME and ID of the repository)

```shell
NAME="my-repository-name"
ID="my-repository-id"
sudo mkdir /data/humio-data/ready_for_import_dataspaces
sudo mv /data/from-other/dataspace_$ID /data/humio-data/ready_for_import_dataspaces
sudo chown -R humio /data/humio-data/ready_for_import_dataspaces/
curl -XPOST \
     -H "Content-Type: application/json" \
     -H "Authorization: Bearer $TOKEN" \
     -d @from-other-global-data-snapshot.json \
     "$BASEURL/api/v1/importrepository/$NAME"
```

The `POST` imports the metadata, such as users and dashboards, and moves
the repository folder from
`/data/humio-data/ready_for_import_dataspaces` to
`/data/humio-data/import`. A low-priority background task will then
import the actual segments files from that point on.

You can start using the ingest tokens and other data, that are not
actual log-events as soon as the POST has completed.

You can run the `POST` starting the import of the same repository more
than once. This is useful if you wish to import only a fraction of the
data files at first, but get all the metadata. When you rerun the POST,
the metadata is inserted/updated again, if it no longer matches
only. The new repository files will get copied at that point in time.

## Configure auto-sharding for high-volume datasources

A datasource is ultimately bounded by the volume that one CPU thread can manage
to compress and write to the filesystem. This is typically in the 1-4 TB/day range.
To handle more ingest traffic from a specific datasource, you ned to provide more
variability in the set of tags. But in some cases it may not be possible or desirable to adjust
the set of tags or tagged fields in the client. To solve this case, Humio supports
adding a synthetic tag, that is assigned a random number for each (small bulk) of events.

The API allows `GET`/`POST`/`DELETE` of the settings. `POST` with no arguments
applies a default number of shards, currently _4_.

The API requires root access.

### Examples

```shell
curl -H "Authorization: Bearer $TOKEN" "$BASEURL/api/v1/repositories/$REPOSITORY_NAME/datasources/$DATASOURCEID/autosharding"
curl -XPOST -H "Authorization: Bearer $TOKEN" "$BASEURL/api/v1/repositories/$REPOSITORY_NAME/datasources/$DATASOURCEID/autosharding"
curl -XPOST -H "Authorization: Bearer $TOKEN" "$BASEURL/api/v1/repositories/$REPOSITORY_NAME/datasources/$DATASOURCEID/autosharding?number=7"
curl -XDELETE -H "Authorization: Bearer $TOKEN" "$BASEURL/api/v1/repositories/$REPOSITORY_NAME/datasources/$DATASOURCEID/autosharding"
```

Humio also supports detecting if there is high load on a datasource, and automatically trigger this auto-sharding on the datasources.
You will see this happening on "fast" datasources, typically if more than 2 TB/day is delivered to a single datasource.
The events then get an extra tag, `#humioAutoShard` that is assigned a random integer value.

This is configured through the settings `AUTOSHARDING_TRIGGER_DELAY_MS`, which is compared to the time an event spends in the ingest pipeline inside Humio.
When the delay threshold is exceeded, the number of shards on that datasource (combination of tags) is doubled.
The default value for `AUTOSHARDING_TRIGGER_DELAY_MS` is 5000 ms (5 seconds).
The delay needs to be increasing as well, as noted two times in a row at an interval of `AUTOSHARDING_CHECKINTERVAL_MS` which defaults to 20000 (20 seconds).

The setting `AUTOSHARDING_MAX` controls how many different datasources get created this way for each "real" datasource. Default value is 128. Internally, the number of cores and hosts reading from the ingest queue is also taken into consideration, aiming at not creating more shards than totoal number of cores in the ingest part of the cluster.

## Status endpoint

The status endpoint can be used to check whether the node can be reached and which version it's running.
Useful as e.g. a smoke test after a humio upgrade or as a health check to be used with service discovery tools such as Consul.

Example:

```shell
$ curl -s https://cloud.humio.com/api/v1/status
{"status":"ok","version":"1.1.0--build-2965--sha-581e6ec64"
```
