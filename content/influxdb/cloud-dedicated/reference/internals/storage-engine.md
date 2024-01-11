---
title: InfluxDB v3 storage engine architecture
description: >
  The InfluxDB v3 storage engine is a real-time, columnar database optimized for
  time series data that supports infinite tag cardinality, real-time queries,
  and is optimized to reduce storage cost.
weight: 103
menu:
  influxdb_cloud_dedicated:
    name: Storage engine architecture
    parent: InfluxDB internals
influxdb/cloud-dedicated/tags: [storage, internals]
related:
  - /influxdb/cloud-dedicated/admin/custom-partitions/
---

The InfluxDB v3 storage engine is a real-time, columnar database optimized for
time series data built in [Rust](https://www.rust-lang.org/) on top of
[Apache Arrow](https://arrow.apache.org/) and
[DataFusion](https://arrow.apache.org/datafusion/user-guide/introduction.html).
It supports infinite tag cardinality (number of unique tag values), real-time
queries, and is optimized to reduce storage cost.

- [Storage engine diagram](#storage-engine-diagram)
- [Storage engine components](#storage-engine-components)
  - [Ingester](#ingester)
  - [Querier](#querier)
  - [Catalog](#catalog)
  - [Object store](#object-store)
  - [Compactor](#compactor)
- [Scaling strategies](#scaling-strategies)
  - [Vertical scaling](#vertical-scaling)
  - [Horizontal scaling](#horizontal-scaling)

## Storage engine diagram

{{< svg "/static/svgs/v3-storage-architecture.svg" >}}

## Storage engine components

- [Ingester](#ingester)
- [Querier](#querier)
- [Catalog](#catalog)
- [Object store](#object-store)
- [Compactor](#compactor)

### Ingester

The Ingester processes line protocol submitted with write requests and persists
time series data to the [Object store](#object-store). In this process, the
Ingester does the following:

- Queries the [Catalog](#catalog) to identify where data should be persisted and
  to ensure the schema of the line protocol is compatible with the schema of
  persisted data.
- Identifies any write errors and generates a response for the client that made
  the write request.
- Processes line protocol and persists time series data to the
  [Object store](#object-store) in Apache Parquet format. Each Parquet file
  represents a _partition_--a logical grouping of data.
- Makes yet-to-be-persisted data available to [Queriers](#querier) to ensure
  leading edge data is included in query results.
- Maintains a short-term write-ahead log (WAL) to prevent data loss in case of a
  service interruption.

##### Ingester scaling strategies

The Ingester can be scaled both [vertically](#vertical-scaling) and
[horizontally](#horizontal-scaling).
Horizontal scaling increases write throughput and is typically the most
effective scaling strategy for the Ingester.

### Querier

The querier handles query requests and returns query results to the client that
mad the query request.
It supports both SQL and InfluxQL through
[Apache Arrow DataFusion](https://arrow.apache.org/datafusion/user-guide/introduction.html).

#### Query life cycle

At query time, the querier:

1.  Receives the query request and builds a query plan.
2.  Queries the [Ingester](#ingester) to:

    - ensure the schema assumed by the query plan matches the schema of written data.
    - include recently written, yet-to-be-persisted data in query results.

3.  Queries the [Catalog](#catalog) to identify the physical location of partitions
    that contain the queried data.
4.  Reads partition Parquet files from the [Object store](#object-store)
    containing the queried data and scans each row to find those that match the
    logic defined in the query plan.
5.  Returns the data specified in the query and performs any additional
    operations specified in the query plan.
6.  Returns the query result to the client.

##### Querier scaling strategies

The Querier can be scaled both [vertically](#vertical-scaling) and
[horizontally](#horizontal-scaling).
Horizontal scaling increases query throughput to handle more concurrent queries.
Vertical scaling improves the queriers ability to process computation-heavy queries.

### Catalog

The Catalog is a PostgreSQL-compatible relational database that stores metadata
related to your time series data including schema information and physical
locations of partitions in the [Object store](#object-store).
It fulfills the following roles:

- Provides information about the schema of written data.
- Tells the [Ingester](#ingester) what partitions to persist data to.
- Tells the [Querier](#querier) what partitions contain the queried data.

##### Catalog scaling strategies

Scaling strategies available for the Catalog depend on the PostgreSQL-compatible
database used to run the catalog. All support [vertical scaling](#vertical-scaling).
Most support [horizontal scaling](#horizontal-scaling) for redundancy and failover. 

### Object store

The Object store contains time series data in [Apache Parquet](https://parquet.apache.org/) format.
Each parquet file represents a partition.
By default, InfluxDB partitions data by measurement and day, but you can
[customize the partitioning strategy](/influxdb/cloud-dedicated/admin/custom-partitions/).

##### Object store scaling strategies

Scaling strategies available for the Object store depend on the underlying
object storage services used to run the object store.
Most support [horizontal scaling](#horizontal-scaling) for redundancy, failover,
and increased capacity.

### Compactor

The Compactor processes and compresses partitions in the [Object store](#object-store)
to continually optimize storage.
It then updates the [Catalog](#catalog) with locations of compacted data.

##### Compactor scaling strategies

The Compactor can be scaled both [vertically](#vertical-scaling) and
[horizontally](#horizontal-scaling).
Because compaction is a compute-heavy process, vertical scaling (especially
increasing the available CPU) is the most effective scaling strategy for the Compactor.
Horizontal scaling increases compaction throughput, but not as well as vertical scaling.

---

## Scaling strategies

The following scaling strategies can be applied to components of the InfluxDB v3
storage architecture.

{{% note %}}
<!-- Cloud Dedicated-specific -->
For information about scaling your {{< product-name >}} infrastructure,
[contact InfluxData support](https://support.influxdata.com).
{{% /note %}}

### Vertical scaling

Vertical scaling (also known as "scaling up") involves increasing the resources
(such as RAM or CPU) available to a process or system.
Vertical scaling is typically used to handle resource-intensive tasks that
require more processing power.

{{< html-diagram/scaling-strategy "vertical" >}}

### Horizontal scaling

Horizontal scaling (also known as "scaling out") involves increasing the number of
nodes or processes available to perform a given task. 
Horizontal scaling is typically used to increase the amount of workload or
throughput a system can manage, but also provides additional redundancy and failover.

{{< html-diagram/scaling-strategy "horizontal" >}}