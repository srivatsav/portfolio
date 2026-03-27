---
title: "Taming the Table API in Flink"
date: 2023-06-18T00:00:00+05:30
draft: false
tags: ["flink", "streaming", "table-api", "data-engineering"]
---

*Originally published on [Medium](https://medium.com/@vatsav.gs/taming-the-table-api-in-flink-4b025e55140a)*

In this article I will try to cover few aspects of Table API in Apache Flink. The goal of this article is not to give any generic solutions about table API but it's more about the problems that I had encountered while working on a very specific use case @Phenom Inc.

**NOTE**: All the solutions or the problems that I would refer to further in this article were encountered on flink version 1.16.1

First things first, the Table API allows one to work with data in a familiar SQL-like language, which makes it easier for both SQL-savvy folks and those who prefer a more programmatic approach. There are two different runtime environments for table API in Flink:

- `TableEnvironment`
- `StreamTableEnvironment`

The major dissimilarity between both is that StreamTableEnvironment seamlessly integrates with Flink's DataStream API. So any unbounded / bounded data stream can literally be a table as simple as:

```java
private static final StreamExecutionEnvironment STREAM_ENVIRONMENT =
    StreamExecutionEnvironment.getExecutionEnvironment();
    
private static final StreamTableEnvironment STREAM_TABLE_ENVIRONMENT =
    StreamTableEnvironment.create(STREAM_ENVIRONMENT);

DataStream<Row> tooComplexStream = ____________________________________;

Table table = STREAM_TABLE_ENVIRONMENT
    .fromDataStream(tooComplexStream).as(schema);
```

Doing this creates a table object in the runtime environment and returns a reference of the table object. But can you access it already and perform queries / data manipulations? NO, to do any sort of operation in SQL you have to give a table name and so is the case with Flink Table too. So, the table that got created should be registered with a name as:

```java
STREAM_TABLE_ENVIRONMENT.createTemporaryView("simplified_data", table);
```

This registers the table with a name `simplified_data`.

With `createTemporaryView`, you can explore, query, and transform your data using a temporary logical view, without the need to persist the intermediate results.

## Schema Definition

In order to be able to create a table in Flink you definitely need a schema similar to SQL table. Now, there are multiple ways to define a schema in Flink Table API.