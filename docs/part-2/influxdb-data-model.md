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

# Introducing the IOx data model

InfluxDB is getting a major upgrade with the new *IOx data model*.

Read this section to gain a firm understanding of the new InfluxDB data model using an IOx bucket, with a particular emphasis on how the *IOx data model* differs from the *TSM data model*.

**Note:** IOx is still in testing. Stay tuned on [influxdata.com](https://www.influxdata.com/) to hear when we officially roll out the new IOx data model to all InfluxDB Cloud production clusters.

## Similarities Between the TSM and IOx Data Model

* [Data elements](#data-elements)
* [Flux compatibility](#flux-compatibility)
* [Line protocol compatibility](#line-protocol-compatibility)

### Data Elements

IOx retains the same data elements that users are accustomed to in InfluxDB, namely:

* Buckets for storing data.
* Measurements for grouping like data together.
* Tag sets, in conjunction with a timestamp, identify a unique row in a table.
* Fields for storing actual data.
* Timestamps, of course, for ordering data by the time dimension.

Current documentation for these elements is available [here](https://awesome.influxdata.com/docs/part-2/influxdb-data-model/#influxdb-data-elements), and will be updated to reflect any changes related to IOx.

### Flux Compatibility

Users who have existing Flux scripts for their application that are working well for them can be reassured that those scripts will continue working unmodified. Throughout the development of IOx such backward compatibility was a consistent focus. However, users should note that with some slight changes to their Flux, to be described later, they will be able to achieve significant performance improvements using IOx.

### Line Protocol Compatibility

Similar to Flux, significant effort has been invested to ensure that InfluxDB Line Protocol compatibility is retained. Therefore, the documentation on line protocol available [here](https://awesome.influxdata.com/docs/part-2/input-format-vs-output-format/#line-protocol) remains relevant with the caveat that users should think about the disk persistence and the output format using Table Flux differently if they want to take full advantage of IOx. This document will pick up the story there, the model for persisting data to disk.

## From Line Protocol to Tables on Disk

In previous versions of InfluxDB, it was most useful to envision the database storing data as a series, with each series in a separate table. The IOx data model is arguably more intuitive, because it builds tables that are returned by Table Flux.

IOx, like TSM before it, is a “schema on write” database engine. This means you are free to write data with new measurements, tags, tag values, and fields after deploying your application, and IOx will accept those writes and persist them as tables. Note that there are some important caveats regarding schema on write, for example, you cannot change the type of a field by merely writing a value with a new type.

A table in IOx is defined by a measurement name. Columns in the table include:

* Tag names
* Field names
* A single time column

Therefore, rows contain:

* Tag values
* Field values
* A single timestamp

Rows are identified by their tag values and time stamp. This becomes relevant to understanding when looking at Upserts.

In the following examples, we'll explore how tables are created on writes in IOx.

### Line Protocol, Fields, and Tables

The simplest line protocol is one measurement and one field with a value. In lieu of providing a timestamp, we can allow the database to add the timestamp by omitting it from the line protocol:

```
measurement1 field1=1i
```

This will be persisted by IOx as a table:

<table>
  <tr>
   <td colspan="2" ><b>Name: measurement1</b>
   </td>
  </tr>
  <tr>
   <td><strong>field1</strong>
   </td>
   <td><strong>time</strong>
   </td>
  </tr>
  <tr>
   <td>1i
   </td>
   <td>timestamp1
   </td>
  </tr>
</table>

As you can see in the above example, the table is defined by the measurement name, and contains a single column, called “field1” and a single row of data. Writing more similar data will grow the table as expected. If we write another line of line protocol:

```
measurement1 field1=2i
```

<table>
  <tr>
   <td colspan="2" ><b>Name: measurement1</b>
   </td>
  </tr>
  <tr>
   <td><strong>field1</strong>
   </td>
   <td><strong>time</strong>
   </td>
  </tr>
  <tr>
   <td>1i
   </td>
   <td>timestamp1
   </td>
  </tr>
  <tr>
   <td>2i
   </td>
   <td>timestamp2
   </td>
  </tr>
</table>

If we write a third line of line protocol, this time with a different field name, IOx will add the field to the table, and null the previous writes.

```
measurement1 field2=3i
```

<table>
  <tr>
   <td colspan="3" ><b>Name: measurement1</b>
   </td>
  </tr>
  <tr>
   <td><strong>field1</strong>
   </td>
   <td><strong>field2</strong>
   </td>
   <td><strong>time</strong>
   </td>
  </tr>
  <tr>
   <td>1i
   </td>
   <td>null
   </td>
   <td>timestamp1
   </td>
  </tr>
  <tr>
   <td>2i
   </td>
   <td>null
   </td>
   <td>timestamp2
   </td>
  </tr>
  <tr>
   <td>null
   </td>
   <td>3i
   </td>
   <td>timestamp3
   </td>
  </tr>
</table>

IOx still supports writing multiple fields in a single line of line protocol, of course, so we can write some line protocol like this:

```
measurement1 field1=4i,field2=4i
```

<table>
  <tr>
   <td colspan="3" ><b>Name: measurement1</b>
   </td>
  </tr>
  <tr>
   <td><strong>field1</strong>
   </td>
   <td><strong>field2</strong>
   </td>
   <td><strong>time</strong>
   </td>
  </tr>
  <tr>
   <td>1i
   </td>
   <td>null
   </td>
   <td>timestamp1
   </td>
  </tr>
  <tr>
   <td>2i
   </td>
   <td>null
   </td>
   <td>timestamp2
   </td>
  </tr>
  <tr>
   <td>null
   </td>
   <td>3i
   </td>
   <td>timestamp3
   </td>
  </tr>
  <tr>
   <td>4i
   </td>
   <td>4i
   </td>
   <td>timestamp4
   </td>
  </tr>
</table>

## Adding Tags

On the surface, it may appear that tags and fields are equivalent in IOx. For example, a simple bit of line protocol with a single tag and field will result in a table with 3 columns, one for the tag, one for the field, and one for the time. Consider the following line protocol and the ensuing table.

```
measurement1,tag1=tagvalue1 field1=1i
```

<table>
  <tr>
   <td colspan="3" ><b>Name: measurement1</b>
   </td>
  </tr>
  <tr>
   <td><strong>field1</strong>
   </td>
   <td><strong>tag1</strong>
   </td>
   <td><strong>time</strong>
   </td>
  </tr>
  <tr>
   <td>1i
   </td>
   <td>tagvalue1
   </td>
   <td>timestamp1
   </td>
  </tr>
</table>

In a departure from the previous data model, adding a new tag value gets added to the same table:

```
measurement1,tag1=tagvalue2 field1=2i
```

<table>
  <tr>
   <td colspan="3" ><b>Name: measurement1</b>
   </td>
  </tr>
  <tr>
   <td><strong>field1</strong>
   </td>
   <td><strong>tag1</strong>
   </td>
   <td><strong>time</strong>
   </td>
  </tr>
  <tr>
   <td>1i
   </td>
   <td>tagvalue1
   </td>
   <td>timestamp1
   </td>
  </tr>
  <tr>
   <td>2i
   </td>
   <td>tagvalue2
   </td>
   <td>timestamp2
   </td>
  </tr>
</table>

Now, if we do a new write, but with a different tag name, similar to adding a field, this will update the table with a new column for the tag, and missing tag values will be set to null.

```
measurement1,tag2=tagvalue3 field1=3i
```

<table>
  <tr>
   <td colspan="4" ><b>Name: measurement1</b>
   </td>
  </tr>
  <tr>
   <td><strong>field1</strong>
   </td>
   <td><strong>tag1</strong>
   </td>
   <td><strong>tag2</strong>
   </td>
   <td><strong>time</strong>
   </td>
  </tr>
  <tr>
   <td>1i
   </td>
   <td>tagvalue1
   </td>
   <td>null
   </td>
   <td>timestamp1
   </td>
  </tr>
  <tr>
   <td>2i
   </td>
   <td>tagvalue2
   </td>
   <td>null
   </td>
   <td>timestamp2
   </td>
  </tr>
  <tr>
   <td>3i
   </td>
   <td>null
   </td>
   <td>tagvalue3
   </td>
   <td>timestamp3
   </td>
  </tr>
</table>

We can continue adding to the measurement1 table in this manner by introducing new tags and fields as needed.

```
measurement1,tag1=tagvalue1,tag2=tagvalue3,tag3=tagvalue4 field1=4i,field2=true
```

<table>
  <tr>
   <td colspan="6" ><b>Name: measurement1</b>
   </td>
  </tr>
  <tr>
   <td><strong>field1</strong>
   </td>
   <td><strong>field2</strong>
   </td>
   <td><strong>tag1</strong>
   </td>
   <td><strong>tag2</strong>
   </td>
   <td><strong>tag3</strong>
   </td>
   <td><strong>time</strong>
   </td>
  </tr>
  <tr>
   <td>1i
   </td>
   <td>null
   </td>
   <td>tagvalue1
   </td>
   <td>null
   </td>
   <td>null
   </td>
   <td>timestamp1
   </td>
  </tr>
  <tr>
   <td>2i
   </td>
   <td>null
   </td>
   <td>tagvalue2
   </td>
   <td>null
   </td>
   <td>null
   </td>
   <td>timestamp2
   </td>
  </tr>
  <tr>
   <td>3i
   </td>
   <td>null
   </td>
   <td>null
   </td>
   <td>tagvalue3
   </td>
   <td>null
   </td>
   <td>timestamp3
   </td>
  </tr>
  <tr>
   <td>4i
   </td>
   <td>true
   </td>
   <td>tagvalue1
   </td>
   <td>tagvalue3
   </td>
   <td>tagvalue4
   </td>
   <td>timestamp4
   </td>
  </tr>
</table>

It’s still possible to write a minimal line of line protocol, and add that to the table:

```
measurement1 field1=1i
```

<table>
  <tr>
   <td colspan="6" ><b>Name: measurement1</b>
   </td>
  </tr>
  <tr>
   <td><strong>field1</strong>
   </td>
   <td><strong>field2</strong>
   </td>
   <td><strong>tag1</strong>
   </td>
   <td><strong>tag2</strong>
   </td>
   <td><strong>tag3</strong>
   </td>
   <td><strong>time</strong>
   </td>
  </tr>
  <tr>
   <td>1i
   </td>
   <td>null
   </td>
   <td>tagvalue1
   </td>
   <td>null
   </td>
   <td>null
   </td>
   <td>timestamp1
   </td>
  </tr>
  <tr>
   <td>2i
   </td>
   <td>null
   </td>
   <td>tagvalue2
   </td>
   <td>null
   </td>
   <td>null
   </td>
   <td>timestamp2
   </td>
  </tr>
  <tr>
   <td>3i
   </td>
   <td>null
   </td>
   <td>null
   </td>
   <td>tagvalue3
   </td>
   <td>null
   </td>
   <td>timestamp3
   </td>
  </tr>
  <tr>
   <td>4i
   </td>
   <td>true
   </td>
   <td>tagvalue1
   </td>
   <td>tagvalue3
   </td>
   <td>tagvalue4
   </td>
   <td>timestamp4
   </td>
  </tr>
  <tr>
   <td>1i
   </td>
   <td>null
   </td>
   <td>null
   </td>
   <td>null
   </td>
   <td>null
   </td>
   <td>timestamp5
   </td>
  </tr>
</table>

## Timestamps

As discussed above, a row is identified by a combination of timestamps and tag values. As such, duplicate timestamps are valid, so long as the tag values are different. For example, we can add multiple rows with timestamp5 by varying the tag values. Consider this line protocol, which has a duplicate timestamp.

```
measurement1,tag1=tagvalue1 field1=1i timestamp5
```

Remembering that a row is defined by its tag values, despite the timestamp and the field being identical, this still represents a new row:

<table>
  <tr>
   <td colspan="6" ><b>Name: measurement1</b>
   </td>
  </tr>
  <tr>
   <td><strong>field1</strong>
   </td>
   <td><strong>field2</strong>
   </td>
   <td><strong>tag1</strong>
   </td>
   <td><strong>tag2</strong>
   </td>
   <td><strong>tag3</strong>
   </td>
   <td><strong>time</strong>
   </td>
  </tr>
  <tr>
   <td>1i
   </td>
   <td>null
   </td>
   <td>tagvalue1
   </td>
   <td>null
   </td>
   <td>null
   </td>
   <td>timestamp1
   </td>
  </tr>
  <tr>
   <td>2i
   </td>
   <td>null
   </td>
   <td>tagvalue2
   </td>
   <td>null
   </td>
   <td>null
   </td>
   <td>timestamp2
   </td>
  </tr>
  <tr>
   <td>3i
   </td>
   <td>null
   </td>
   <td>null
   </td>
   <td>tagvalue3
   </td>
   <td>null
   </td>
   <td>timestamp3
   </td>
  </tr>
  <tr>
   <td>4i
   </td>
   <td>true
   </td>
   <td>tagvalue1
   </td>
   <td>tagvalue3
   </td>
   <td>tagvalue4
   </td>
   <td>timestamp4
   </td>
  </tr>
  <tr>
   <td>1i
   </td>
   <td>null
   </td>
   <td>null
   </td>
   <td>null
   </td>
   <td>null
   </td>
   <td>timestamp5
   </td>
  </tr>
  <tr>
   <td>1i
   </td>
   <td>null
   </td>
   <td>tagvalue1
   </td>
   <td>null
   </td>
   <td>null
   </td>
   <td>timestamp5
   </td>
  </tr>
</table>


When you send a line of line protocol, the write takes into account the **entire** set of tag values. So long as a single tag value is different, even if all the fields and the timestamp are identical, the write will still result in a new row.

For example, the following row is identical to an existing row, except that the value for tag3 is tagvalue5 instead of tagvalue4. Therefore, this will result in a new row being added.


```
measurement1,tag1=tagvalue1,tag2=tagvalue3,tag3=tagvalue5 field1=4i,field2=true timestamp4
```

<table>
  <tr>
   <td colspan="6" ><b>Name: measurement1</b>
   </td>
  </tr>
  <tr>
   <td><strong>field1</strong>
   </td>
   <td><strong>field2</strong>
   </td>
   <td><strong>tag1</strong>
   </td>
   <td><strong>tag2</strong>
   </td>
   <td><strong>tag3</strong>
   </td>
   <td><strong>time</strong>
   </td>
  </tr>
  <tr>
   <td>1i
   </td>
   <td>null
   </td>
   <td>tagvalue1
   </td>
   <td>null
   </td>
   <td>null
   </td>
   <td>timestamp1
   </td>
  </tr>
  <tr>
   <td>2i
   </td>
   <td>null
   </td>
   <td>tagvalue2
   </td>
   <td>null
   </td>
   <td>null
   </td>
   <td>timestamp2
   </td>
  </tr>
  <tr>
   <td>3i
   </td>
   <td>null
   </td>
   <td>null
   </td>
   <td>tagvalue3
   </td>
   <td>null
   </td>
   <td>timestamp3
   </td>
  </tr>
  <tr>
   <td>4i
   </td>
   <td>true
   </td>
   <td>tagvalue1
   </td>
   <td>tagvalue3
   </td>
   <td>tagvalue4
   </td>
   <td>timestamp4
   </td>
  </tr>
  <tr>
   <td>4i
   </td>
   <td>true
   </td>
   <td>tagvalue1
   </td>
   <td>tagvalue3
   </td>
   <td>tagvalue5
   </td>
   <td>timestamp4
   </td>
  </tr>
  <tr>
   <td>1i
   </td>
   <td>null
   </td>
   <td>null
   </td>
   <td>null
   </td>
   <td>null
   </td>
   <td>timestamp5
   </td>
  </tr>
  <tr>
   <td>1i
   </td>
   <td>null
   </td>
   <td>tagvalue1
   </td>
   <td>null
   </td>
   <td>null
   </td>
   <td>timestamp5
   </td>
  </tr>
</table>

## Upserts

It is useful to think of all writes to InfluxDB as Upserts. The term “Upsert” means that the write will “Update or Insert” on write. It will update if it matches an existing record, or will insert a new record if no existing records match.

Because Upserts match on the timestamp and the tag values, it is not possible to update a tag value! Only field values can be updated.

This line of line protocol includes no tags, a duplicate timestamp, and a duplicate field name, but a different field value. So, this matches a timestamp and the tag set (which happens to be empty). Therefore, this will result in an update to the table.


```
measurement1 field1=2i timestamp5
```

<table>
  <tr>
   <td colspan="6" ><b>Name: measurement1</b>
   </td>
  </tr>
  <tr>
   <td><strong>field1</strong>
   </td>
   <td><strong>field2</strong>
   </td>
   <td><strong>tag1</strong>
   </td>
   <td><strong>tag2</strong>
   </td>
   <td><strong>tag3</strong>
   </td>
   <td><strong>time</strong>
   </td>
  </tr>
  <tr>
   <td>1i
   </td>
   <td>null
   </td>
   <td>tagvalue1
   </td>
   <td>null
   </td>
   <td>null
   </td>
   <td>timestamp1
   </td>
  </tr>
  <tr>
   <td>2i
   </td>
   <td>null
   </td>
   <td>tagvalue2
   </td>
   <td>null
   </td>
   <td>null
   </td>
   <td>timestamp2
   </td>
  </tr>
  <tr>
   <td>3i
   </td>
   <td>null
   </td>
   <td>null
   </td>
   <td>tagvalue3
   </td>
   <td>null
   </td>
   <td>timestamp3
   </td>
  </tr>
  <tr>
   <td>4i
   </td>
   <td>true
   </td>
   <td>tagvalue1
   </td>
   <td>tagvalue3
   </td>
   <td>tagvalue4
   </td>
   <td>timestamp4
   </td>
  </tr>
  <tr>
   <td>4i
   </td>
   <td>true
   </td>
   <td>tagvalue1
   </td>
   <td>tagvalue3
   </td>
   <td>tagvalue5
   </td>
   <td>timestamp4
   </td>
  </tr>
  <tr>
   <td>2i
   </td>
   <td>null
   </td>
   <td>null
   </td>
   <td>null
   </td>
   <td>null
   </td>
   <td>timestamp5
   </td>
  </tr>
  <tr>
   <td>1i
   </td>
   <td>null
   </td>
   <td>tagvalue1
   </td>
   <td>null
   </td>
   <td>null
   </td>
   <td>timestamp5
   </td>
  </tr>
</table>


The following line protocol also matches an existing timestamp, and a tag set, so it will result in an update. While only one of the fields is present in the line protocol, the row is still matched, so the field will be updated.


```
measurement1,tag1=tagvalue1,tag2=tagvalue3,tag3=tagvalue5 field2=false timestamp4
```

<table>
  <tr>
   <td colspan="6" ><b>Name: measurement1</b>
   </td>
  </tr>
  <tr>
   <td><strong>field1</strong>
   </td>
   <td><strong>field2</strong>
   </td>
   <td><strong>tag1</strong>
   </td>
   <td><strong>tag2</strong>
   </td>
   <td><strong>tag3</strong>
   </td>
   <td><strong>time</strong>
   </td>
  </tr>
  <tr>
   <td>1i
   </td>
   <td>null
   </td>
   <td>tagvalue1
   </td>
   <td>null
   </td>
   <td>null
   </td>
   <td>timestamp1
   </td>
  </tr>
  <tr>
   <td>2i
   </td>
   <td>null
   </td>
   <td>tagvalue2
   </td>
   <td>null
   </td>
   <td>null
   </td>
   <td>timestamp2
   </td>
  </tr>
  <tr>
   <td>3i
   </td>
   <td>null
   </td>
   <td>null
   </td>
   <td>tagvalue3
   </td>
   <td>null
   </td>
   <td>timestamp3
   </td>
  </tr>
  <tr>
   <td>4i
   </td>
   <td>true
   </td>
   <td>tagvalue1
   </td>
   <td>tagvalue3
   </td>
   <td>tagvalue4
   </td>
   <td>timestamp4
   </td>
  </tr>
  <tr>
   <td>4i
   </td>
   <td>false
   </td>
   <td>tagvalue1
   </td>
   <td>tagvalue3
   </td>
   <td>tagvalue5
   </td>
   <td>timestamp4
   </td>
  </tr>
  <tr>
   <td>2i
   </td>
   <td>null
   </td>
   <td>null
   </td>
   <td>null
   </td>
   <td>null
   </td>
   <td>timestamp5
   </td>
  </tr>
  <tr>
   <td>1i
   </td>
   <td>null
   </td>
   <td>tagvalue1
   </td>
   <td>null
   </td>
   <td>null
   </td>
   <td>timestamp5
   </td>
  </tr>
</table>

A new field can be introduced similarly. The following line protocol matches the timestamp and all tag values for the existing row, so the row will be updated, along with the new field value. As expected, all existing rows without the new field will be set to null for that field.

```
measurement1,tag1=tagvalue1,tag2=tagvalue3,tag3=tagvalue5 field3=0.0 timestamp4
```

<table>
  <tr>
   <td colspan="7" ><b>Name: measurement1</b>
   </td>
  </tr>
  <tr>
   <td><strong>field1</strong>
   </td>
   <td><strong>field2</strong>
   </td>
   <td><strong>field3</strong>
   </td>
   <td><strong>tag1</strong>
   </td>
   <td><strong>tag2</strong>
   </td>
   <td><strong>tag3</strong>
   </td>
   <td><strong>time</strong>
   </td>
  </tr>
  <tr>
   <td>1i
   </td>
   <td>null
   </td>
   <td>null
   </td>
   <td>tagvalue1
   </td>
   <td>null
   </td>
   <td>null
   </td>
   <td>timestamp1
   </td>
  </tr>
  <tr>
   <td>2i
   </td>
   <td>null
   </td>
   <td>null
   </td>
   <td>tagvalue2
   </td>
   <td>null
   </td>
   <td>null
   </td>
   <td>timestamp2
   </td>
  </tr>
  <tr>
   <td>3i
   </td>
   <td>null
   </td>
   <td>null
   </td>
   <td>null
   </td>
   <td>tagvalue3
   </td>
   <td>null
   </td>
   <td>timestamp3
   </td>
  </tr>
  <tr>
   <td>4i
   </td>
   <td>true
   </td>
   <td>null
   </td>
   <td>tagvalue1
   </td>
   <td>tagvalue3
   </td>
   <td>tagvalue4
   </td>
   <td>timestamp4
   </td>
  </tr>
  <tr>
   <td>4i
   </td>
   <td>false
   </td>
   <td>0.0
   </td>
   <td>tagvalue1
   </td>
   <td>tagvalue3
   </td>
   <td>tagvalue5
   </td>
   <td>timestamp4
   </td>
  </tr>
  <tr>
   <td>2i
   </td>
   <td>null
   </td>
   <td>null
   </td>
   <td>null
   </td>
   <td>null
   </td>
   <td>null
   </td>
   <td>timestamp5
   </td>
  </tr>
  <tr>
   <td>1i
   </td>
   <td>null
   </td>
   <td>null
   </td>
   <td>tagvalue1
   </td>
   <td>null
   </td>
   <td>null
   </td>
   <td>timestamp5
   </td>
  </tr>
</table>

[Next Section]({{site.url}}/docs/part-2/input-format-vs-output-format){: .btn .btn-purple}

