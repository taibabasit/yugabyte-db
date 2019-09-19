---
title: Change data capture (CDC)
linkTitle: CDC architecture
description: Change data capture (CDC) architecture
beta: /faq/product/#what-is-the-definition-of-the-beta-feature-tag
menu:
  latest:
    parent: change-data-capture
    identifier: cdc-architecture
    weight: 651
type: page
isTocNested: true
showAsideToc: true
---

This topic includes an overview of change data capture (CDC), use cases, the CDC process architecture, and how to use CDC with Apache Kafka.

## Overview

Change data capture (CDC) refers to technology for ensuring that any changes in data (inserts, updates, and deletions) are identified, captured, and automatically applied to another data repository instance or made available for consumption by applications and other tools. CDC is useful in a number of scenarios, such as:

### Microservice-oriented architectures

Some microservices require a stream of changes to the data and using CDC in Yugabyte DB can provide consumable data changes to CDC subscribers.

### Asynchronous replication to remote systems

Remote systems may subscribe to a stream of data changes and then transform and consume the changes. Maintaining separate database instances for transactional and reporting purposes can be used to manage workload performance.

### Multiple data center strategies

Maintaining multiple data centers enables enterprises to provide:

- High availability (HA) — Redundant systems helps ensure that your operations virtually never fail
- Geo-redundancy — Geographically dispersed servers provide resiliency against catastrophic events and natural disasters

Two data center (2DC), or dual data center, deployments are a common use of CDC that allows efficient management of two Yugabyte universes that are geographically separated. For more information, see [Deploy to two data centers](../two-data-centers).

### Compliance and auditing

Auditing and compliance requirements can require you to use CDC to maintain records of data changes.

{{< note title="Note" >}}

In the design overview below, the terms "data center", "cluster", and "universe" are used interchangeably. We assume here that each YB universe is deployed in a single data-center.

{{< /note >}}

## Process Architecture

```
                          ╔═══════════════════════════════════════════╗
                          ║  Node #1                                  ║
                          ║  ╔════════════════╗ ╔══════════════════╗  ║
                          ║  ║    YB-Master   ║ ║    YB-TServer    ║  ║  CDC Service is stateless
    CDC Streams metadata  ║  ║  (Stores CDC   ║ ║  ╔═════════════╗ ║  ║           |
    replicated with Raft  ║  ║   metadata)    ║ ║  ║ CDC Service ║ ║  ║<----------'
             .----------->║  ║                ║ ║  ╚═════════════╝ ║  ║
             |            ║  ╚════════════════╝ ╚══════════════════╝  ║
             |            ╚═══════════════════════════════════════════╝
             |
             |
             |_______________________________________________.
             |                                               |
             V                                               V
  ╔═══════════════════════════════════════════╗    ╔═══════════════════════════════════════════╗
  ║  Node #2                                  ║    ║  Node #3                                  ║
  ║  ╔════════════════╗ ╔══════════════════╗  ║    ║  ╔════════════════╗ ╔══════════════════╗  ║
  ║  ║    YB-Master   ║ ║    YB-TServer    ║  ║    ║  ║    YB-Master   ║ ║    YB-TServer    ║  ║
  ║  ║  (Stores CDC   ║ ║  ╔═════════════╗ ║  ║    ║  ║  (Stores CDC   ║ ║  ╔═════════════╗ ║  ║
  ║  ║   metadata)    ║ ║  ║ CDC Service ║ ║  ║    ║  ║   metadata)    ║ ║  ║ CDC Service ║ ║  ║
  ║  ║                ║ ║  ╚═════════════╝ ║  ║    ║  ║                ║ ║  ╚═════════════╝ ║  ║
  ║  ╚════════════════╝ ╚══════════════════╝  ║    ║  ╚════════════════╝ ╚══════════════════╝  ║
  ╚═══════════════════════════════════════════╝    ╚═══════════════════════════════════════════╝

```

### CDC streams

Creating a new CDC stream on a table returns a stream UUID. The CDC Service stores information about all streams in the system table `cdc_streams`. The schema for this table looks like this:

```
cdc_streams {
stream_id  text,
params     map<text, text>,
primary key (stream_id)
}
```

### CDC subscribers

Along with creating a CDC stream, a CDC subscriber is also created for all existing tablets of the stream. A new subscriber entry is created in the `cdc_subscribers` table. The schema for this table is:

```
cdc_subscribers {
stream_id      text,
subscriber_id  text,
tablet_id      text,
data           map<text, text>,
primary key (stream_id, subscriber_id, tablet_id)
}
```

### CDC service APIs

Every YB-TServer has a `CDC service` that is stateless. The main APIs provided by `CDC Service` are:

- `SetupCDC` API — Sets up a CDC stream for a table.
- `RegisterSubscriber` API — Registers a CDC subscriber that will read changes from some, or all, tablets of the CDC stream.
- `GetChanges` API – Used by CDC subscribers to get the latest set of changes.
- `GetSnapshot` API — Used to bootstrap CDC subscribers and get the current snapshot of the database (typically will be invoked prior to GetChanges)

### Pushing changes to external systems

Each Yugabyte DB's TServer has CDC subscribers (`cdc_subscribers`) that are responsible for getting changes for all tablets for which the TServer is a leader. When a new stream and subscriber are created, the TServer `cdc_subscribers` detects this and starts invoking the `cdc_service.GetChanges` API periodically to get the latest set of changes.

While invoking `GetChanges`, the CDC subscriber needs to pass in a `from_checkpoint` which is the last `OP ID` that it successfully consumed. When the CDC service receives a request of `GetChanges` for a tablet, it reads the changes from the WAL (log cache) starting from `from_checkpoint`, deserializes them and returns those to CDC subscriber. It also records the `from_checkpoint` in `cdc_subscribers` table in the data column. This will be used for bootstrapping fallen subscribers who don’t know the last checkpoint or in case of tablet leader changes.

When `cdc_subscribers` receive the set of changes, they then push these changes out to Kafka.

### CDC guarantees

#### Per-tablet ordered delivery guarantee

All data changes for one row, or multiple rows in the same tablet, will be received in the order in which they occur. Due to the distributed nature of the problem, however, there is no guarantee for the order across tablets.

For example, let us imagine the following scenario:

- Two rows are being updated concurrently.
- These two rows belong to different tablets.
- The first row `row #1` was updated at time `t1` and the second row `row #2` was updated at time `t2`.

In this case, it is possible for CDC to push the later update corresponding to `row #2` change to Kafka before pushing the earlier update, corresponding to `row #1`.

#### At-least-once delivery

Updates for rows will be pushed at least once. With "at-least-once" delivery, you will never lose a message, but might end up being delivered to a CDC consumer more than once. This can happen in case of tablet leader change, where the old leader already pushed changes to Kafka, but the latest pushed `op id` was not updated in `cdc_subscribers` table.

For example, imagine a CDC client has received changes for a row at times t1 and t3. It is possible for the client to receive those updates again.

#### No gaps in change stream

When you have received a change for a row for timestamp `t`, you will not receive a previously unseen change for that row from an earlier timestamp. This guarantees that receiving any change implies that all earlier changes have been received for a row.

## Use change data capture in Yugabyte DB

To use change data capture (CDC) in Yugabyte DB, perform the following steps.

### Create a CDC stream

To create a CDC stream, run the following commands on YSQL or YCQL APIs:

```sql
CREATE CDC
       FOR <namespace>.<table>
       INTO <target>
       WITH <options>;
```

### Drop a CDC stream

To drop an existing CDC stream, run the following command:

```sql
DROP CDC FOR <namespace>.<table>;
```

## Using Kafka as the CDC target

Yugabyte DB supports the use of Apache Kafka as a target for CDC-generated data changes.

### To create a CDC target for Apache Kafka

Run the following CDC command, with any required options, to create a CDC target for Apache Kafka:

```
CREATE CDC FOR my_database.my_table
           INTO KAFKA
           WITH cluster_address = 'http://localhost:9092',
                topic = 'my_table_topic',
                record = AFTER;
```

The CDC options for Kafka include:

| Option name       | Default       | Description   |
| ----------------- | ------------- | ------------- |
| `cluster_address` | -             | The `host:port` of the Kafka target cluster |
| `topic`           | Table name    | The Kafka topic to which the stream is published |
| `record`          | AFTER         | The type of records in the stream. Valid values are `CHANGE` (changed values only), `AFTER` (the entire row after the change is applied), ALL (the before and after value of the row). |