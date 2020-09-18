---
id: data-management
title: "Data management"
---

<!--
  ~ Licensed to the Apache Software Foundation (ASF) under one
  ~ or more contributor license agreements.  See the NOTICE file
  ~ distributed with this work for additional information
  ~ regarding copyright ownership.  The ASF licenses this file
  ~ to you under the Apache License, Version 2.0 (the
  ~ "License"); you may not use this file except in compliance
  ~ with the License.  You may obtain a copy of the License at
  ~
  ~   http://www.apache.org/licenses/LICENSE-2.0
  ~
  ~ Unless required by applicable law or agreed to in writing,
  ~ software distributed under the License is distributed on an
  ~ "AS IS" BASIS, WITHOUT WARRANTIES OR CONDITIONS OF ANY
  ~ KIND, either express or implied.  See the License for the
  ~ specific language governing permissions and limitations
  ~ under the License.
  -->




## Schema changes

Schemas for datasources can change at any time and Apache Druid supports different schemas among segments.

### Replacing segments

Druid uniquely
identifies segments using the datasource, interval, version, and partition number. The partition number is only visible in the segment id if
there are multiple segments created for some granularity of time. For example, if you have hourly segments, but you
have more data in an hour than a single segment can hold, you can create multiple segments for the same hour. These segments will share
the same datasource, interval, and version, but have linearly increasing partition numbers.

```
foo_2015-01-01/2015-01-02_v1_0
foo_2015-01-01/2015-01-02_v1_1
foo_2015-01-01/2015-01-02_v1_2
```

In the example segments above, the dataSource = foo, interval = 2015-01-01/2015-01-02, version = v1, partitionNum = 0.
If at some later point in time, you reindex the data with a new schema, the newly created segments will have a higher version id.

```
foo_2015-01-01/2015-01-02_v2_0
foo_2015-01-01/2015-01-02_v2_1
foo_2015-01-01/2015-01-02_v2_2
```

Druid batch indexing (either Hadoop-based or IndexTask-based) guarantees atomic updates on an interval-by-interval basis.
In our example, until all `v2` segments for `2015-01-01/2015-01-02` are loaded in a Druid cluster, queries exclusively use `v1` segments.
Once all `v2` segments are loaded and queryable, all queries ignore `v1` segments and switch to the `v2` segments.
Shortly afterwards, the `v1` segments are unloaded from the cluster.

Note that updates that span multiple segment intervals are only atomic within each interval. They are not atomic across the entire update.
For example, you have segments such as the following:

```
foo_2015-01-01/2015-01-02_v1_0
foo_2015-01-02/2015-01-03_v1_1
foo_2015-01-03/2015-01-04_v1_2
```

`v2` segments will be loaded into the cluster as soon as they are built and replace `v1` segments for the period of time the
segments overlap. Before v2 segments are completely loaded, your cluster may have a mixture of `v1` and `v2` segments.

```
foo_2015-01-01/2015-01-02_v1_0
foo_2015-01-02/2015-01-03_v2_1
foo_2015-01-03/2015-01-04_v1_2
```

In this case, queries may hit a mixture of `v1` and `v2` segments.

### Different schemas among segments

Druid segments for the same datasource may have different schemas. If a string column (dimension) exists in one segment but not
another, queries that involve both segments still work. Queries for the segment missing the dimension will behave as if the dimension has only null values.
Similarly, if one segment has a numeric column (metric) but another does not, queries on the segment missing the
metric will generally "do the right thing". Aggregations over this missing metric behave as if the metric were missing.

<a name="compact"></a>

## Compaction and reindexing

Compaction is a type of overwrite operation that reads an existing set of segments and rewrites them, most commonly to reorganize smaller segments into larger ones, which can improve query efficiency. For more about compaction, see [Compaction](compaction). 

## Adding new data

Druid can insert new data to an existing datasource by appending new segments to existing segment sets. It can also add new data by merging an existing set of segments with new data and overwriting the original set.

Druid does not support single-record updates by primary key.

<a name="update"></a>

## Updating existing data

Once you ingest some data in a dataSource for an interval and create Apache Druid segments, you might want to make changes to
the ingested data. There are several ways this can be done.

### Using lookups

If you have a dimension where values need to be updated frequently, try first using [lookups](../querying/lookups.md). A
classic use case of lookups is when you have an ID dimension stored in a Druid segment, and want to map the ID dimension to a
human-readable String value that may need to be updated periodically.

### Reingesting data

If lookup-based techniques are not sufficient, you will need to reingest data into Druid for the time chunks that you
want to update. This can be done using one of the [batch ingestion methods](../ingestion/index.md#batch) in overwrite mode (the
default mode). It can also be done using [streaming ingestion](../ingestion/index.md#streaming), provided you drop data for the
relevant time chunks first.

If you do the reingestion in batch mode, Druid's atomic update mechanism means that queries will flip seamlessly from
the old data to the new data.

We recommend keeping a copy of your raw data around in case you ever need to reingest it.

### With Hadoop-based ingestion

This section assumes the reader understands how to do batch ingestion using Hadoop. See
[Hadoop batch ingestion](../ingestion/hadoop.md) for more information. Hadoop batch-ingestion can be used for reindexing and delta ingestion.

Druid uses an `inputSpec` in the `ioConfig` to know where the data to be ingested is located and how to read it.
For simple Hadoop batch ingestion, `static` or `granularity` spec types allow you to read data stored in deep storage.

There are other types of `inputSpec` to enable reindexing and delta ingestion.

### Reindexing with Native Batch Ingestion

This section assumes the reader understands how to do batch ingestion without Hadoop using [native batch indexing](../ingestion/native-batch.md),
which uses an `inputSource` to know where and how to read the input data. The [`DruidInputSource`](../ingestion/native-batch.md#druid-input-source)
can be used to read data from segments inside Druid. Note that IndexTask is to be used for prototyping purposes only as
it has to do all processing inside a single process and can't scale. Please use Hadoop batch ingestion for production
scenarios dealing with more than 1GB of data.

<a name="delete"></a>

## Deleting data

Druid supports permanent deletion of segments that are in an "unused" state (see the
[Segment lifecycle](../design/architecture.html#segment-lifecycle) section of the Architecture page).

The Kill Task deletes unused segments within a specified interval from metadata storage and deep storage.

For more information, please see [Kill Task](../ingestion/tasks.html#kill).

Permanent deletion of a segment in Apache Druid has two steps:

1. The segment must first be marked as "unused". This occurs when a segment is dropped by retention rules, and when a user manually disables a segment through the Coordinator API.
2. After segments have been marked as "unused", a Kill Task will delete any "unused" segments from Druid's metadata store as well as deep storage.

For documentation on retention rules, please see [Data Retention](../operations/rule-configuration.md).

For documentation on disabling segments using the Coordinator API, please see the
[Coordinator Datasources API](../operations/api-reference.html#coordinator-datasources) reference.

A data deletion tutorial is available at [Tutorial: Deleting data](../tutorials/tutorial-delete-data.md)

## Kill Task

Kill tasks delete all information about a segment and removes it from deep storage. Segments to kill must be unused (used==0) in the Druid segment table. The available grammar is:

```json
{
    "type": "kill",
    "id": <task_id>,
    "dataSource": <task_datasource>,
    "interval" : <all_segments_in_this_interval_will_die!>,
    "context": <task context>
}
```

## Retention

Druid supports retention rules, which are used to define intervals of time where data should be preserved, and intervals where data should be discarded.

Druid also supports separating Historical processes into tiers, and the retention rules can be configured to assign data for specific intervals to specific tiers.

These features are useful for performance/cost management; a common use case is separating Historical processes into a "hot" tier and a "cold" tier.

For more information, please see [Load rules](../operations/rule-configuration.md).
