---
id: compaction
title: "Data compaction"
---

## What is compaction?

Compaction is a type of overwrite operation that reads an existing set of segments, combines them into a new set with larger but fewer segments, and overwrites the original set with the new compacted set, without changing the data that is stored.

For performance reasons, it is sometimes beneficial to compact a set of segments into a set of larger but fewer segments, as there is some per-segment processing and memory overhead in both the ingestion and querying paths.

Compaction tasks merge all segments of the given interval. 

> [Segment Size Optimization](../operations/segment-optimization) provides additional information on why you would want to use compaction to optimize data segment size. 


## Compaction task syntax

The syntax of a compaction task spec is:

```json
{
    "type": "compact",
    "id": <task_id>,
    "dataSource": <task_datasource>,
    "ioConfig": <IO config>,
    "dimensionsSpec" <custom dimensionsSpec>,
    "metricsSpec" <custom metricsSpec>,
    "segmentGranularity": <segment granularity after compaction>,
    "tuningConfig" <parallel indexing task tuningConfig>,
    "context": <task context>
}
```

|Field|Description|Required|
|-----|-----------|--------|
|`type`|Task type. Should be `compact`|Yes|
|`id`|Task id|No|
|`dataSource`|DataSource name to be compacted|Yes|
|`ioConfig`|ioConfig for compaction task. See [Compaction IOConfig](#compaction-ioconfig) for details.|Yes|
|`dimensionsSpec`|Custom dimensionsSpec. Compaction task will use this dimensionsSpec if exist instead of generating one. See below for more details.|No|
|`metricsSpec`|Custom metricsSpec. Compaction task will use this metricsSpec if specified rather than generating one.|No|
|`segmentGranularity`|If this is set, compactionTask will change the segment granularity for the given interval. See `segmentGranularity` of [`granularitySpec`](index.md#granularityspec) for more details. See the below table for the behavior.|No|
|`tuningConfig`|[Parallel indexing task tuningConfig](../ingestion/native-batch.md#tuningconfig)|No|
|`context`|[Task context](../ingestion/tasks.md#context)|No|


An example of compaction task is

```json
{
  "type" : "compact",
  "dataSource" : "wikipedia",
  "ioConfig" : {
    "type": "compact",
    "inputSpec": {
      "type": "interval",
      "interval": "2017-01-01/2018-01-01"
    }
  }
}
```

This compaction task reads _all segments_ of the interval `2017-01-01/2018-01-01` and results in new segments.
Since `segmentGranularity` is null, the original segment granularity will be remained and not changed after compaction.
To control the number of result segments per time chunk, you can set [maxRowsPerSegment](../configuration/index.html#compaction-dynamic-configuration) or [numShards](../ingestion/native-batch.md#tuningconfig).
Please note that you can run multiple compactionTasks at the same time. For example, you can run 12 compactionTasks per month instead of running a single task for the entire year.

A compaction task internally generates an `index` task spec for performing compaction work with some fixed parameters.
For example, its `inputSource` is always the [DruidInputSource](native-batch.md#druid-input-source), and `dimensionsSpec` and `metricsSpec`
include all dimensions and metrics of the input segments by default.

Compaction tasks will exit with a failure status code, without doing anything, if the interval you specify has no
data segments loaded in it (or if the interval you specify is empty).

The output segment can have different metadata from the input segments unless all input segments have the same metadata.

- Dimensions: since Apache Druid supports schema change, the dimensions can be different across segments even if they are a part of the same dataSource.
If the input segments have different dimensions, the output segment basically includes all dimensions of the input segments.
However, even if the input segments have the same set of dimensions, the dimension order or the data type of dimensions can be different. For example, the data type of some dimensions can be
changed from `string` to primitive types, or the order of dimensions can be changed for better locality.
In this case, the dimensions of recent segments precede that of old segments in terms of data types and the ordering.
This is because more recent segments are more likely to have the new desired order and data types. If you want to use
your own ordering and types, you can specify a custom `dimensionsSpec` in the compaction task spec.
- Roll-up: the output segment is rolled up only when `rollup` is set for all input segments.
See [Roll-up](../ingestion/index.html#rollup) for more details.
You can check that your segments are rolled up or not by using [Segment Metadata Queries](../querying/segmentmetadataquery.html#analysistypes).


## Compaction IOConfig

The compaction IOConfig requires specifying `inputSpec` as seen below.

|Field|Description|Required|
|-----|-----------|--------|
|`type`|Task type. Should be `compact`|Yes|
|`inputSpec`|Input specification|Yes|

There are two supported `inputSpec`s for now.

The interval `inputSpec` is:

|Field|Description|Required|
|-----|-----------|--------|
|`type`|Task type. Should be `interval`|Yes|
|`interval`|Interval to compact|Yes|

The segments `inputSpec` is:

|Field|Description|Required|
|-----|-----------|--------|
|`type`|Task type. Should be `segments`|Yes|
|`segments`|A list of segment IDs|Yes|


## Autocompaction

You can configure rules to have the Druid Coordinator service run compaction tasks at regular intervals. [Compacting segments](../design/coordinator.html#compacting-segments) describes how the Coordinator performs compaction.  

Autocompaction can not only optimize data sgement size, it can invoke data repartitioning. Data is typically partitioned by timestamp, but you can set secondary partitioning criteria to further refine and optimize query performance. See [Partitioning](index#partitioning)for more information about why partition. 

When repartitioning via autocompaction, there are several strategies you can employ ...... 


TBD: NEW DOC HERE.

