---
layout: default
title: InfluxDB Data Model
description: "Part 2: The InfluxDB Data Model"
parent: Part 2
nav_order: 1
---

# InfluxDB Data Model
{: .no_toc }

## Table of contents
{: .no_toc .text-delta }

1. TOC
{:toc}

---
The goal of this section is to provide the reader with a firm foundation in the InfluxDB data model, specifically:

* Understanding the InfluxDB input format (line protocol).
* Understanding the InfluxDB output format (annotated CSV).
* The relationships between these two formats.
* Understanding how the Influxdb storage engine persists data in a table on disk


## InfluxDB Data Elements


### Buckets

All data in InfluxDB gets written to a bucket. A [bucket](https://docs.influxdata.com/influxdb/cloud/organizations/buckets/) is a container that can hold points for as many measurements as you like. Buckets have some important properties:



1. They can be named whatever you want (within reason).
2. You can create tokens that can control read and write permissions for a bucket, scoped only to a specific bucket.
3. You must set a retention period on a bucket, upon creation. A [retention period](https://docs.influxdata.com/influxdb/cloud/reference/glossary/#retention-period) determines how long InfluxDB will store your time series data within that bucket. Retention periods are critical for time series database management.  They provide users with a convenient solution for automatic expiration of old, useless data which enables them to focus on the recent, valuable data instead while reducing their storage bills. 

These topics will be covered in detail in a later section, for now, it is enough to know that measurements are stored in and read from a bucket.


### Measurements

A [measurement](https://docs.influxdata.com/influxdb/v2.0/reference/syntax/line-protocol/#measurement) is the highest level of data structure within a bucket. InfluxDB accepts one measurement per point. Use a measurement to organize similar data.  In some ways, you can think of it as analogous to a table in a traditional database. Measurements are also indexed, which enables you to query data within a measurement more quickly when you filter for a specific measurement. Measurements must be a string type.  Measurement names cannot begin with an underscore.

To further understand measurements,  let’s imagine you’re building a weather app and writing temperature data across multiple cities to InfluxDB. For this time series use case, you might create a measurement named “air_temperature”. 


### Tag Sets

A [tag set](https://docs.influxdata.com/influxdb/v2.0/reference/syntax/line-protocol/#tag-set) consists of key-value pairs, where the values are always strings. Tags are essentially metadata, typically encoding information about the source of the data. 

Imagine you’re building a weather app and writing temperature data across multiple cities to InfluxDB. For this time series use case, you might add a tag key to your data called “location” where the “location” tag key contains tag values for the cities weather you’re monitoring. This way you can easily query for all of the temperature data for “Los Angeles”, for example, by filtering for the “location” tag key with the “Los Angeles” tag value. 

Critically, tags are indexed, so they are an important element of designing for query performance. However, tags are also optional. Tag keys cannot begin with an underscore, because InfluxDB reserves leading underscores for its own purposes. 


### Field Sets

A [field set](https://docs.influxdata.com/influxdb/v2.0/reference/syntax/line-protocol/#field-set) consists of key-value pairs, where the values can be strings, integers, or floats. Fields are the actual data to store, visualize, use for computations, etc...

Building on the weather app example, the temperature readings would be an example of a field.

Field sets are required for InfluxDB. This is where you store the value of your time series data. Field values can be of either an integer, float, or string type. Since field values are almost always different, field keys are frequently referred to as just fields. Fields are not indexed. Field also cannot begin with an underscore. 


## Series and Points

A series is defined by the unique combination of measurement, tag set(s), and fields. A point is a datapoint from a series at a specific timestamp. If you rewrite a duplicate point with identical measurement, tag set, field, and timestamp values, this write will overwrite your previous point. If you try to write a new point to the same series but you change the field type, this write will fail. 


## Assumptions and Conventions 

Before diving into more nuanced and technical topics around the InfluxDB Data Model, let’s take a moment to establish some baseline assumptions and conventions.  


### Conventions Used In This Book

Chapters in this book will generally introduce concepts using abstract and simplified examples, and then follow with detailed examples using real world data.


### Meta Syntax for Examples

For the abstract and simplified examples, we will use names in the form of:


```
attributeorvaluen
```


Where “attributeorvalue” refers a column name of a table or a value in a point, and “n” is a number simply differentiating multiple identifiers of the same role. Roles can comprise any of the following:



* Measurement
* Tag
* Tag Value
* Field

Samples will also generally include:



* Field Values
* Timestamps

Field values will be represented by actual values. Timestamps are in the following timestamp formats:



* Unix: The unix timestamp is a way to track time as a running total of seconds–i.e. `1465839830100400200`
* RFC3339: A standard for date and time representation from a Request For Comment document by the Internet Engineering Task Force (IETF)–​​i.e. `2019-08-28T22:00:000000000Z`
* Relative Duration: -`1h`
* Duration: `1h`

Timestamps will be represented by actual values or by `unixtime1` or `rfc3339time1` . In case we want to refer to another timestamp in the same example, we will use `unixtime2` or `rfc3339time2`.

To refer to a measurement in an example, we will use: `measurement1`. In case we want to refer to another measurement in the same example, we will use `measurement2`, and so forth.

An example of line protocol (explained in depth later) then, may look like:


```js
measurement1,tag1=tagvalule1,tag2=tagvalue2 field1=1i,field2=2 1628858104
```


From time to time an example may be focused on understanding a type. In such cases, we will use the form “atype” where “type” is the data type under focus. For example, if we are discussing that field names are always strings. we may say:


```js
r._field == "astring"
```


Or if we are discussing type conflicts, we may say:


```js
aint == afloat
```


Instead of an example with specific values such as:


```js
1i == 1.0
```

[Next Section]({{site.url}}/docs/part-2/input-format-vs-output-format){: .btn .btn-purple}

