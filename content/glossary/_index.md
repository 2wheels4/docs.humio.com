---
title: "Glossary"
weight: 10
---
This section introduces the core concepts of Humio. It also describes how Humio organizes data.

## Main Concepts

### Data Sources

A Data Source is a set of Events that have the same [Tags](#tags).
Humio divides each [Respository](#repos) into more than one Data Source.

Humio creates Data Sources automatically when it encounters a new combination of Tags. Users cannot create Data Sources directly.

Data Sources are the smallest unit of data that you can delete.
You cannot delete individual Events in a Data Source beyond expiration.<!--GRW: I'm not sure what 'beyond expiration' means. -->

Humio represents each Data Source internally as a dedicated directory within the Repository directory.

{{% notice note %}}
We recommend that you do not create more than 1,000 separate Tags, or combinations of Tags.

If you need more combinations, then we recommend that you use attributes on individual Events to differentiate them, and let you select them separately.
{{% /notice %}}

### Repositories {#repos}

Humio organizes data into [Repositories]({{< relref "getting_started/repositories.md" >}}).
Each Repository has its own set of users, and a single directory on disk.

When you set up data quotas and retention policies, you configure them for each Repository.

{{% notice note %}}
Queries cannot span more than one Repository. But you can create
[Views]({{< relref "getting_started/views.md" >}})
that do span multiple Repositories to achieve a cross-repostitory search.
See [Repositories]({{< relref "getting_started/repositories.md" >}}).
{{% /notice %}}

### Events {#events}

Events are data items that represent a particular message, incident, or logging item from a system. They are the most important data type in Humio.

Each Event contains a timestamp and a set of key/value attributes.

Humio represents the original text of the Event in the attribute `@rawstring`.
You can configure a parser to extract the attributes and timestamp of Events from the raw text entries.

For JSON data, you can specify what the `@rawstring` represents. By default, this is the original JSON data string.
The timestamp of an Event is represented in the `@timestamp` attribute

Events also have a special `@repo` meta-field that denotes the repository the event comes from.
This is useful in cross-repository searches when using [Views]({{< relref "getting_started/views.md" >}}).

Events can also have [Tags](#tags) associated with them.
The Data Source manages and stores Tags related to Events. This means that Tags do not add to the storage requirements of individual Events.

### Tags {#tags}

Humio saves data in Data Sources. You can provide a set of Tags to specify which Data Source the data is saved in.  
You can add Tags to [Events](#events) that you ingest into Humio.
Tags provide an important way to speed up searching. They allow Humio to select which Data Sources to search through.     
For example, you can add Tags to Events that represent host names, file names, service names, or the kind of service.  
Tags can be configured in [parsers](/sending_logs_to_humio/parsers/parsing/) or specified in the APIs for data ingestion.

{{% notice note %}}
Tags are an advanced option in Humio. It can be used to separate data into different datasources and thereby improve query performance.
**If in doubt, start out by not specifying any tags.**

We, at Humio (support@humio.com), are ready to help you in using tags.
{{% /notice %}}

In Humio tags always start with a #. When turning a field into a tag it will be
prepended with `#`.
If fields start with an `@` or `_` , the character is removed before prepending
the `#`.

{{% notice warning %}}
You should use Tags for the aspects that you want to search for most often.

Do not create more distinct dynamic Tags than you need. This reduces system
performance and increases resource usage.

You should set dynamic values, such as names that include dates, as Event
attributes, not Tags. Attributes are individual key/values that are associated
with an individual Event.
{{% /notice %}}

### Users

You can configure Humio to run either with or without user authentication.
If user authentication is disabled, then everyone with access to the site can
access everything.

When you run Humio with authentication enabled, each Repository has its own set of users.
Humio identifies users by their email address. It validates each email address
using an OAuth identity provider - either Google, Github, or Bitbucket. Humio
can also check usernames and password using your local LDAP service.

There are three levels of users: 'normal', 'administrator', and 'root':

* Normal users can only access and query data, including managing dashboards an dsaved queries
* Administrators can also add and remove other users to a Repository and make them administrators of the Repository.
* 'Root' users can add Repositories and create new root users.

  You can create your initial users with 'Root' acces through the HTTP API. See [how to gain root access using root access token](/operation/installation/authentication/#root-token)

You can manage Users and their rights using the 'Repository' web page in Humio. Root users (apart from the initial one) can get added through the 'Administration' page when you are logged in as a root user.

{{% notice note %}}
You can add the same user ID to more than one Repository.
{{% /notice %}}


## Query Concepts

### Aggregate Queries
_Aggregate queries_ are queries that join the Events into a new structure of Events with attributes.

A query becomes an _aggregate query_ if it uses an aggregate function like `sum()`, `count()` or `avg()`. See [functions](/searching_logs/query_functions/) for more information.

For example, the query `count()` takes a stream of Events as its input, and produces one Event containing a `count` attribute.

<!--
The final result af an _aggregate query_ is not ready until the query has completed, although it is still possible to get a partial result.
In contrast _filter queries_ can start streaming the response as soon as Events pass through the 'filter'
-->


### Filter Queries
_Filter queries_, or _non-aggregate queries_, are queries that only filter
Events, or add or remove attributes on each Event.

These queries can only contain filters and transformation functions
(see [functions](/searching_logs/query_functions/))


### Live Queries

Live queries provide a way to run searches that are continuously
updated as new Events arrive. Live queries are important for creating dashboards,
and many other uses.

Humio uses an efficient streaming implementation to provide this feature.

In a live query, the time interval is a time window relative to 'now', such
as 'the last 5 minutes' or 'the last day'.

Humio sets the {{% function "groupBy" %}} attribute of a live query automatically.
It bases the grouping on buckets that each represent a part of the given time interval.

Aggregate queries run live inside each bucket as Events arrive. Whenever the current
response is selected, it runs the aggregations for the query across the buckets.

{{% notice note %}}
Humio purges live queries if no client has checked their status for 60 minutes.
{{% /notice %}}

### Query Boundaries

The term *query boundary* means those aspects of a query that
Humio uses to select the data that it scans to produce the query result.

The query boundary consists of a time interval and a set of Tags.

In the Humio interface, you can set the time interval using a special selector.
The Tags should preferably be at the start of the query string.
For example, `#tagname=value`.
