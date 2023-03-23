---
layout: default
title: Input Format vs Output Format
description: "Part 2: Input Format vs Output Format"
parent: Part 2
nav_order: 2
---
# Input Format vs Output Format
{: .no_toc }

## Table of contents
{: .no_toc .text-delta }

1. TOC
{:toc}

---

The InfluxDB input format is line protocol. The InfluxDB output format is Annotated CSV. InfluxDB users are sometimes initially confused because using line protocol can be similar to writing a single table, but then when they query the same data, the query returns multiple tables in CSV. This is because as the data formatted in line protocol is ingested by InfluxDB it is seperated into series. Understanding this subtle relationship between the ingestion format and time series is important for good schema design and for using InfluxDB optimally.   


## Line Protocol

The smallest allowed  line of [line protocol](https://docs.influxdata.com/influxdb/cloud/reference/syntax/line-protocol/) includes a measurement, a single field, and a value for that field. A measurement is the highest grouping of data inside a bucket. A field is  a part of that measurement and  defines a single point. Fields are separated by measurements by a space. So, the smallest readable line looks like:


```js
measurement1 field1=1i
```



### Adding multiple fields

You need to supply at least one field, but you can supply as many as you need in a single line, separated by a comma:


```js
measurement1 field1=1i,field2=1,field3="a"
```


Line protocol is very compact, but each of the fields will be in their own series when stored in the database.  More on that in [From Series to Tables on Disk](). 


### Types explained

The types of a field value can be an integer, a float, or a string. By default a number will be interpreted as a float. The addition of the  “i” after each number for my_field tells InfluxDB that I wanted those to be integers. Using quotes ensures that InfluxDB knows I want a string type.


### Type conflicts   

Once you have created a series, you cannot change the type field type of the series. InfluxDB will reject the the following write:


```js
measurement1 field1=1,field2="1",field3="a"
```


After having written this line: 
```js
measurement1 field1=1i,field2=1,field3="a"
```

Notice how the field2 field value has been changed from a string to a float. 


### Adding tags

Tags are another kind of data that you can add to a line in line protocol. Tags are useful because they are automatically indexed by InfluxDB. Using and querying for tags allows you to significantly improve your query performance. Additionally, tags are useful for categorizing your queries. 

Remember, a series is defined by a measurement(s), tag sets(s), and field key(s). Tags are defined after the measurement name, and separated from it by a comma. I can add a tag to the previous line protocol as such:


```js
measurement1,tag1=tagvalue1 field1=1i,field2=1,field3="a" 1626118680000000000
measurement1,tag1=tagvalue2 field1=2i,field2=2,field3="b" 1626118740000000000
```


The introduction of the tag key, “tag1”, with 2 different tag values, “tagvalue1” and “tagvalue2” produces 6 series–3 series come from the different fields for each of the 2 tag values. 


### Adding timestamps 

In cases where a timestamp is omitted from line protocol, InfluxDB will add a timestamp based on the current server time at the time of the write.

Letting InfluxDB automatically supply a timestamp is very convenient, but is likely too imprecise for your application, so you will typically supply a timestamp as well. InfluxDB expects timestamps to be [unix timestamps](https://docs.influxdata.com/influxdb/v2.0/reference/syntax/line-protocol/#timestamp). InfluxDB also expects the timestamps to be in nanosecond resolution, but the write API allows you to define lower resolution precision if needed. 

The timestamp comes at the end of a line of line protocol and is separated from the last field value by a space. Supplying a timestamp to the above line protocol would look like this:


```js
measurement1 field1=1i,field2=1,field3="a" 1626118680000000000
measurement1 field1=2i,field2=2,field3="b" 1626118740000000000
```



### Note on Timestamp Precision

The native resolution of time stamps in InfluxDB is nanoseconds. In InfluxDB, the unit of resolution (for example nanosecond, microsecond, millisecond) is called the “precision” of the time stamp. For context:



* There are 1,000 nanoseconds in a microsecond
* There are 1,000,000 nanoseconds in a millisecond
* There are 1,000,000,000 nanoseconds in a second

This is important to keep in mind while constructing your application, because many systems do not handle nanosecond resolution, so it is necessary to convert between them. 

Note that InfluxDB tools do allow you to define the precision of your timestamps, so when writing, you can allow InfluxDB to handle the conversion for you. This will be covered in Part 3.


### Overwriting points

You can overwrite points in InfluxDB when you write data with the same series and same timestamp. For example, if you wrote this line of line protocol to InfluxDB: 


```js
measurement1 field1=2i,field2=2,field3="b" 1626118740000000000
```


You would then overwrite the 3 points by writing this line next. 


```js
measurement1 field1=10i,field2=10,field3="overwritten" 1626118740000000000
<more examples that include partial overwrites, subsets of tags>
```



## Annotated CSV 

[Annotated CSV](https://docs.influxdata.com/influxdb/cloud/reference/syntax/annotated-csv/) is the output format for InfluxDB. The annotated CSV output matches the InfluxDB persistence format **with simple queries or when you don’t apply additional transformations to your data**. Annotated CSV result is a stream of tables returned by Flux where each table represents a series **for simple queries only**. For example if you wrote this line protocol line to InfluxDB:


```js
measurement1,tag1=tagvalue1 field1=1i
```


You would return the following full annotated CSV output when querying for all fields and tags (of which there are none) within the measurement: 


```
#group,false,false,true,true,false,false,true,true,true
#datatype,string,long,dateTime:RFC3339,dateTime:RFC3339,dateTime:RFC3339,long,string,string,string
#default,_result,,,,,,,,
,result,table,_start,_stop,_time,_value,_field,_measurement,tag1
,,0,rfc3339time1,rfc3339time2,2021-08-17T21:23:39.000000000Z,1,field1,Measurement1,tagvalue1
```


Remember, that line of line protocol produces 1 series which is why one table is included in the annotated CSV output. 


### Raw CSV vs Annotated CSV

The first thing to notice about the output format of annotated CSV is that it resembles the CSV format that you’re familiar with. To easily distinguish between the two we’ll refer to CSV as Raw CSV. Unlike raw CSV, annotated CSV contains the following rows:



* Header Row
* Records Rows
* Annotation Rows


### Header Row

The header row is similar to any header row in a CSV. It describes the column names for your time series data.The header row is found below the 3 Annotation rows.  Our header row is: 


```
,result,table,_start,_stop,_time,_value,_field,_measurement,tag1
```


Some of these headers are an intuitive translation from line protocol while others are not. The headers are: 



* `result`. This column includes the name of the result as specified by the query. If no name is provided it will be empty. 
* `table`. This column contains a unique ID for each table in the annotated CSV output result. stream. In the example above we are only writing and querying for 1 series so the value of that column is set to 0 for that table. 
* `_start.`This column includes the start time for the time range over which you queried your data for. Specifying a range is necessary for any Flux query. 
* `_stop.`This column includes the stop time for the time range over which you queried your data for. Specifying a range is necessary for any Flux query.
* `_time.`This column is the timestamp for your time series. The value of this column is either added upon write or included in the line protocol explicitly. 
* `_value`. This column contains the field values for the corresponding field keys in the same row under the `_field `column. 
* `_field`. This column contains the field keys. 
* `_measurement`. This column contains the name of the measurement. 
* `tag1`. This column contains the tag value for our `tag1` tag key. 

For the rest of this section you can ignore the values for the `_start, _stop, `and `result` columns. Those values are assigned by the user during query execution. For now, just focus on understanding the similarities between your line protocol input format and the resulting annotated CSV.  


### Record Rows

The records row(s) is directly below the header row. These rows contain our time series data.  Each row contains one record. A record is a tuple of named values.  Each table contains at least one record. When querying for your time series without adding any additional Flux transformations, a table


### Records vs Points

Remember, a point is a datapoint from a series at a specific timestamp. A record can be a point but not always. For example, you could use Flux to drop all tag, field, and measurement columns. At this point the records in the annotated CSV wouldn’t reflect a point but rather time series data instead. 


### Annotation Rows

The annotation rows are found in the first three lines of the full annotated CSV output. Annotations include metadata about your annotated CSV. When querying InfluxDB with the API, you can specify which headers you want to include in your annotated CSV output. All 3 headers are included by default when you export the results of your query through the InfluxDB UI. 

The 3 annotations are: 



1. ​​#group: A boolean that indicates the column is part of the [group key](https://v2.docs.influxdata.com/v2.0/query-data/flux/group-data/#group-keys). A group key is a list of columns for which every row in the table has the same value.  A column is part of the group key if its ​​#group annotation is set to ​​true. 

    **Important Note:** The exception is for the table column. The group key for the table is set to false because users can’t directly change the table number. The table record will always be the same across rows even though the group key is set to false. 

2. #datatype: Describes the type of data or which line protocol element the column represents. 
3. #default: The value to use for rows with an empty value. 

If annotations confuse you, don’t worry. The importance of annotations will become apparent in subsequent sections of this book, specifically around understanding Flux. Additionally it’s worth mentioning that the​​ #group annotation is the most important annotation for using Flux successfully. For now, be aware that InfluxDB applies a default group key to your data so that the tables in your annotated CSV output will each represent a single series for simple queries. This default application of group keys is the result of the way that series are stored on disk, described in the following section. 


## From Series to Tables on Disk 

Each series is stored as a table on disk. Data gets written to InfluxDB into buckets. When data is written to a bucket it is added to an appropriate table, or a new table is created if needed. Each table has exactly one measurement, one field, and a unique set of tag values. Indexes are created or amended on write to enable quickly finding these tables when queried. Additionally, all rows in all tables are indexed by time.

Some people consider InfluxDB to be a “schemaless” database. However, that is not really accurate. More accurately, InfluxDB is a “schema on write.” That is to say, InfluxDB does not require a schema to be defined beforehand nor enforce a schema on writes. Instead, Influxdb builds a schema implicitly based on the writes you do make.

While schema on write is a true convenience for developers, you should be cognizant of the schema that you are implicitly creating as you write data. A poorly designed schema can have a negative impact on things like ease of querying, performance, and cardinality.

In this section we’ll learn about how line protocol produces series which get stored as tables on disk. 


### Adding Fields 

In this section we’ll break down how the line protocol is converted to series and written as tables in InfluxDB. Let’s review the line protocol examples above: 


```js
measurement1 field1=1i,field2=1,field3="a"
measurement1 field1=1i,field2=2,field3="b"
```


When written, this will form 3 different time series with 2 points in each series, assuming each line was  written one minute apart. The data will then be persisted in three separate tables:


<table>
  <tr>
   <td>_measurement
   </td>
   <td>_field
   </td>
   <td>_value
   </td>
   <td>_time
   </td>
  </tr>
  <tr>
   <td>measurement1
   </td>
   <td>field1
   </td>
   <td>1i
   </td>
   <td>2021-07-12T19:38:00.000000000Z
   </td>
  </tr>
  <tr>
   <td>measurement1
   </td>
   <td>field1
   </td>
   <td>1i
   </td>
   <td>2021-07-12T19:39:00.000000000Z
   </td>
  </tr>
</table>



<table>
  <tr>
   <td>_measurement
   </td>
   <td>_field
   </td>
   <td>_value
   </td>
   <td>_time
   </td>
  </tr>
  <tr>
   <td>measurement1
   </td>
   <td>field2
   </td>
   <td>1
   </td>
   <td>2021-07-12T19:38:00.000000000Z
   </td>
  </tr>
  <tr>
   <td>measurement1
   </td>
   <td>field2
   </td>
   <td>2
   </td>
   <td>2021-07-12T19:39:00.000000000Z
   </td>
  </tr>
</table>



<table>
  <tr>
   <td>_measurement
   </td>
   <td>_field
   </td>
   <td>_value
   </td>
   <td>_time
   </td>
  </tr>
  <tr>
   <td>measurement1
   </td>
   <td>field3
   </td>
   <td>a
   </td>
   <td>2021-07-12T19:38:00.000000000Z
   </td>
  </tr>
  <tr>
   <td>measurement1
   </td>
   <td>field3
   </td>
   <td>b
   </td>
   <td>2021-07-12T19:39:00.000000000Z
   </td>
  </tr>
</table>


This example provides the first insights into how the input format and the persistence/output format differ. Each series in InfluxDB is persisted with exactly one field, though line protocol allows writing with multiple fields. This is a critically important concept to understand, so it is worth repeating. **The InfluxDB input format is different from the InfluxDB persistence and output format**.

These tables represent the minimum data roles that can exist in a series. A series must have at a minimum:



* A measurement name
* A field key
* A value for the field
* A time

These are represented in the database with a leading “_”. The leading underscore conveys that these are “slots” in the series that are enforced by the storage engine, as well as helps to avoid naming conflicts with potential tag names.

When simply querying for this data, without adding any additional Flux transformations, the annotated CSV output looks like:


```
#group,false,false,true,true,false,false,true,true
#datatype,string,long,dateTime:RFC3339,dateTime:RFC3339,dateTime:RFC3339,long,string,string
#default,_result,,,,,,,
,result,table,_start,_stop,_time,_value,_field,_measurement
,,0,rfc3339time1,rfc3339time2,2021-07-12T19:39:00.000000000Z,1,field1,measurement1
,,0,rfc3339time1,rfc3339time2,2021-07-12T19:38:00.00000000Z,1,field1,measurement1

#group,false,false,true,true,false,false,true,true
#datatype,string,long,dateTime:RFC3339,dateTime:RFC3339,dateTime:RFC3339,string,string,string
#default,_result,,,,,,,
,result,table,_start,_stop,_time,_value,_field,_measurement
,,1,rfc3339time1,rfc3339time2,2021-07-12T19:39:00.000000000Z,a,field3,measurement1
,,1,rfc3339time1,rfc3339time2,2021-07-12T19:38:00.00000000Z,b,field3,measurement1

#group,false,false,true,true,false,false,true,true
#datatype,string,long,dateTime:RFC3339,dateTime:RFC3339,dateTime:RFC3339,double,string,string
#default,_result,,,,,,,
,result,table,_start,_stop,_time,_value,_field,_measurement
,,2,rfc3339time1,rfc3339time2,2021-07-12T19:39:00.000000000Z,1,field2,measurement1
,,2,rfc3339time1,rfc3339time2,2021-07-12T19:38:00.000000000Z,2,field2,measurement1
```


Notice how the resulting annotated CSV contains 3 tables in the output. This is evident by the row separation and also by the value of the `table` column in the last stream of the table which is equal to 2 (remember annotated CSV counts the table results from 0). Group keys have been added to the data to produce these tables so that each table represents a series by default. Remember a column is part of a group key if all of the values in that column are identical within a single table. For example, the `time `and` value `columns are assigned a `#group `annotation of ​​`false.` Setting the `#group `annotation to ​​`false` allows the different timestamps and field values of points across a single series to be included in the same table. 


### Adding Tags 

Let’s review the line protocol example above with an added tag:


```js
measurement1,tag1="tagvalue1" field1=1i,field2=1,field3="a" 1626118680000000000
measurement1,tag1="tagvalue2" field1=2i,field2=2,field3="b" 1626118740000000000
```


The introduction of tag1 produces the following 6 series:


<table>
  <tr>
   <td>_measurement
   </td>
   <td>tag1
   </td>
   <td>_field
   </td>
   <td>_value
   </td>
   <td>_time
   </td>
  </tr>
  <tr>
   <td>measurement1
   </td>
   <td>tagvalue1
   </td>
   <td>field1
   </td>
   <td>1i
   </td>
   <td>2021-07-12T19:38:00.000000000Z
   </td>
  </tr>
</table>



<table>
  <tr>
   <td>_measurement
   </td>
   <td>tag1
   </td>
   <td>_field
   </td>
   <td>_value
   </td>
   <td>_time
   </td>
  </tr>
  <tr>
   <td>measurement1
   </td>
   <td>tagvalue1
   </td>
   <td>field2
   </td>
   <td>1
   </td>
   <td>2021-07-12T19:38:00.000000000Z
   </td>
  </tr>
</table>



<table>
  <tr>
   <td>_measurement
   </td>
   <td>tag1
   </td>
   <td>_field
   </td>
   <td>_value
   </td>
   <td>_time
   </td>
  </tr>
  <tr>
   <td>measurement1
   </td>
   <td>tagvalue1
   </td>
   <td>field3
   </td>
   <td>a
   </td>
   <td>2021-07-12T19:38:00.000000000Z
   </td>
  </tr>
</table>



<table>
  <tr>
   <td>_measurement
   </td>
   <td>tag1
   </td>
   <td>_field
   </td>
   <td>_value
   </td>
   <td>_time
   </td>
  </tr>
  <tr>
   <td>measurement1
   </td>
   <td>tagvalue2
   </td>
   <td>field1
   </td>
   <td>2i
   </td>
   <td>2021-07-12T19:39:00.000000000Z
   </td>
  </tr>
</table>



<table>
  <tr>
   <td>_measurement
   </td>
   <td>tag1
   </td>
   <td>_field
   </td>
   <td>_value
   </td>
   <td>_time
   </td>
  </tr>
  <tr>
   <td>measurement1
   </td>
   <td>tagvalue2
   </td>
   <td>field2
   </td>
   <td>2
   </td>
   <td>2021-07-12T19:39:00.000000000Z
   </td>
  </tr>
</table>



<table>
  <tr>
   <td>_measurement
   </td>
   <td>tag1
   </td>
   <td>_field
   </td>
   <td>_value
   </td>
   <td>_time
   </td>
  </tr>
  <tr>
   <td>measurement1
   </td>
   <td>tagvalue2
   </td>
   <td>field3
   </td>
   <td>b
   </td>
   <td>2021-07-12T19:39:00.000000000Z
   </td>
  </tr>
</table>


Ensure that you take the time to study and understand the relationship between the line protocol and the resulting tables as they are represented on disk by the storage engine. This relationship is critically important to achieving the most effective schema and querying for your application. 

When simply querying for this data, without adding any additional Flux transformations the annotated CSV output looks like:


```
#group,false,false,true,true,false,false,true,true,true
#datatype,string,long,dateTime:RFC3339,dateTime:RFC3339,dateTime:RFC3339,string,string,string,string
#default,_result,,,,,,,,
,result,table,_start,_stop,_time,_value,_field,_measurement,tag1
,,0,rfc3339time1,rfc3339time2,2021-07-12T19:39:000000000Z,b,field3,measurement1,tagvalue2

#group,false,false,true,true,false,false,true,true,true
#datatype,string,long,dateTime:RFC3339,dateTime:RFC3339,dateTime:RFC3339,double,string,string,string
#default,_result,,,,,,,,
,result,table,_start,_stop,_time,_value,_field,_measurement,tag1
,,1,rfc3339time1,rfc3339time2,2021-07-12T19:38:000000000Z,1,field2,measurement1,tagvalue1

#group,false,false,true,true,false,false,true,true,true
#datatype,string,long,dateTime:RFC3339,dateTime:RFC3339,dateTime:RFC3339,long,string,string,string
#default,_result,,,,,,,,
,result,table,_start,_stop,_time,_value,_field,_measurement,tag1
,,2,2021-01-18T20:59:37Z,2021-08-17T19:59:37.097Z,2021-07-12T19:38:000000000Z,1,field1,measurement1,tagvalue1

#group,false,false,true,true,false,false,true,true,true
#datatype,string,long,dateTime:RFC3339,dateTime:RFC3339,dateTime:RFC3339,string,string,string,string
#default,_result,,,,,,,,
,result,table,_start,_stop,_time,_value,_field,_measurement,tag1
,,3,rfc3339time1,rfc3339time2,2021-07-12T19:38:000000000Z,a,field3,measurement1,tagvalue1

#group,false,false,true,true,false,false,true,true,true
#datatype,string,long,dateTime:RFC3339,dateTime:RFC3339,dateTime:RFC3339,long,string,string,string
#default,_result,,,,,,,,
,result,table,_start,_stop,_time,_value,_field,_measurement,tag1
,,4,rfc3339time1,rfc3339time2,2021-07-12T19:39:000000000Z,2,field1,measurement1,tagvalue2

#group,false,false,true,true,false,false,true,true,true
#datatype,string,long,dateTime:RFC3339,dateTime:RFC3339,dateTime:RFC3339,double,string,string,string
#default,_result,,,,,,,,
,result,table,_start,_stop,_time,_value,_field,_measurement,tag1
,,5,rfc3339time1,rfc3339time2,2021-07-12T19:39:000000000Z,2,field2,measurement1,tagvalue2
```


Notice how the resulting annotated CSV contains 6 tables in the output. This is evident by the row separation and also by the value of the `table` column in the last stream of the table which is equal to 5 (remember annotated CSV counts the table results from 0). Group keys have been added to the data to produce these tables so that each table represents a series by default. Remember a column is part of a group key if all of the values in that column are identical within a single table. For example, the `time `column is assigned a `#group `annotation of ​​`false.` Setting the `#group `annotation to ​​`false` allows the different timestamps of points across a single series to be included in the same table. Conversely, the `_measurement `column is assigned a `#group `annotation of ​​`true.` Setting the `#group `annotation to ​​`true` enforces that all of the records in that table have the same measurement value. Remember, a series is identified by a unique combination of measurements, tag sets, and fields. If a table is to represent a single series, the table must contain records with the same measurement, tag sets, and fields across all of the rows. 

**Important note:** You can use Flux to manipulate the group keys and the resulting number of tables in the the output Annotated CSV table stream. We’ll learn about how to do this in later chapters. 

For now, let's focus on understanding how line protocol results in different series. We can extend the example by adding an additional tag, but in this case, note that there is only a single tag value for `tag2`:


```js
measurement1,tag1="tagvalue1",tag2="tagvalue3" field1=1i,field2=1,field3="a" 1626118680000000000
measurement1,tag1="tagvalue2",tag2="tagvalue3" field1=2i,field2=2,field3="b" 1626118740000000000
```


Again, each series is identified by their unique tag keys, tag values, and field key combinations.  Because a series is defined in part by a unique set of tag _values_, in this case, the introduction of tag2 does not change the table count in the underlying data model. When the introduction of a tag does not change the table count in the underlying data model, the tag is referred to as a [dependent tag](https://docs.influxdata.com/influxdb/cloud/reference/glossary/#series-cardinality):


<table>
  <tr>
   <td>_measurement
   </td>
   <td>tag1
   </td>
   <td>tag2
   </td>
   <td>_field
   </td>
   <td>_value
   </td>
   <td>_time
   </td>
  </tr>
  <tr>
   <td>measurement1
   </td>
   <td>tagvalue1
   </td>
   <td>tagvalue3
   </td>
   <td>field1
   </td>
   <td>1i
   </td>
   <td>2021-07-12T19:38:00.000Z
   </td>
  </tr>
</table>



<table>
  <tr>
   <td>_measurement
   </td>
   <td>tag1
   </td>
   <td>tag2
   </td>
   <td>_field
   </td>
   <td>_value
   </td>
   <td>_time
   </td>
  </tr>
  <tr>
   <td>measurement1
   </td>
   <td>tagvalue1
   </td>
   <td>tagvalue3
   </td>
   <td>field2
   </td>
   <td>1
   </td>
   <td>2021-07-12T19:38:00.000Z
   </td>
  </tr>
</table>



<table>
  <tr>
   <td>_measurement
   </td>
   <td>tag1
   </td>
   <td>tag2
   </td>
   <td>_field
   </td>
   <td>_value
   </td>
   <td>_time
   </td>
  </tr>
  <tr>
   <td>measurement1
   </td>
   <td>tagvalue1
   </td>
   <td>tagvalue3
   </td>
   <td>field3
   </td>
   <td>a
   </td>
   <td>2021-07-12T19:38:00.000Z
   </td>
  </tr>
</table>



<table>
  <tr>
   <td>_measurement
   </td>
   <td>tag1
   </td>
   <td>tag2
   </td>
   <td>_field
   </td>
   <td>_value
   </td>
   <td>_time
   </td>
  </tr>
  <tr>
   <td>measurement1
   </td>
   <td>tagvalue2
   </td>
   <td>tagvalue3
   </td>
   <td>field1
   </td>
   <td>2i
   </td>
   <td>2021-07-12T19:39:00.000Z
   </td>
  </tr>
</table>



<table>
  <tr>
   <td>_measurement
   </td>
   <td>tag1
   </td>
   <td>tag2
   </td>
   <td>_field
   </td>
   <td>_value
   </td>
   <td>_time
   </td>
  </tr>
  <tr>
   <td>measurement1
   </td>
   <td>tagvalue2
   </td>
   <td>tagvalue3
   </td>
   <td>field2
   </td>
   <td>2
   </td>
   <td>2021-07-12T19:39:00.000Z
   </td>
  </tr>
</table>



<table>
  <tr>
   <td>_measurement
   </td>
   <td>tag1
   </td>
   <td>tag2
   </td>
   <td>_field
   </td>
   <td>_value
   </td>
   <td>_time
   </td>
  </tr>
  <tr>
   <td>measurement1
   </td>
   <td>tagvalue2
   </td>
   <td>tagvalue3
   </td>
   <td>field3
   </td>
   <td>b
   </td>
   <td>2021-07-12T19:39:00.000Z
   </td>
  </tr>
</table>


To demonstrate the impact of combinations of tag values on the creation of time series, here are three lines of line protocol, but with only one field:


```js
measurement1,tag1="tagvalue1",tag2="tagvalue4" field1=1i 1626118620000000000
measurement1,tag1="tagvalue2",tag2="tagvalue5" field1=2i 1626118680000000000
measurement1,tag1="tagvalue3",tag2="tagvalue6" field1=3i 1626118740000000000
```


In those 3 lines there are 3 unique combinations of tag values and the single field, so, despite the presence of six total tag values, there are only 3 series created:


<table>
  <tr>
   <td>_measurement
   </td>
   <td>tag1
   </td>
   <td>tag2
   </td>
   <td>_field
   </td>
   <td>_value
   </td>
   <td>_time
   </td>
  </tr>
  <tr>
   <td>measurement1
   </td>
   <td>tagvalue1
   </td>
   <td>tagvalue4
   </td>
   <td>field1
   </td>
   <td>1i
   </td>
   <td>2021-07-12T19:37:00.000Z
   </td>
  </tr>
</table>



<table>
  <tr>
   <td>_measurement
   </td>
   <td>tag1
   </td>
   <td>tag2
   </td>
   <td>_field
   </td>
   <td>_value
   </td>
   <td>_time
   </td>
  </tr>
  <tr>
   <td>measurement1
   </td>
   <td>tagvalue2
   </td>
   <td>tagvalue5
   </td>
   <td>field1
   </td>
   <td>2i
   </td>
   <td>2021-07-12T19:38:00.000Z
   </td>
  </tr>
</table>



<table>
  <tr>
   <td>_measurement
   </td>
   <td>tag1
   </td>
   <td>tag2
   </td>
   <td>_field
   </td>
   <td>_value
   </td>
   <td>_time
   </td>
  </tr>
  <tr>
   <td>measurement1
   </td>
   <td>tagvalue3
   </td>
   <td>tagvalue6
   </td>
   <td>field1
   </td>
   <td>3i
   </td>
   <td>2021-07-12T19:39:00.000Z
   </td>
  </tr>
</table>


In this example, there are only 2 unique combinations of tag values:


```js
measurement1,tag1="tagvalue1",tag2="tagvalue4" field1=1i 1626118620000000000
measurement1,tag1="tagvalue1",tag2="tagvalue4" field1=2i 1626118680000000000
measurement1,tag1="tagvalue2",tag2="tagvalue4" field1=3i 1626118740000000000
```


As a result, the first series contains two points because those two points have the same combination of field name and tag values, whereas the third point has a different set of tag values.


<table>
  <tr>
   <td>_measurement
   </td>
   <td>tag1
   </td>
   <td>tag2
   </td>
   <td>_field
   </td>
   <td>_value
   </td>
   <td>_time
   </td>
  </tr>
  <tr>
   <td>measurement1
   </td>
   <td>tagvalue1
   </td>
   <td>tagvalue4
   </td>
   <td>field1
   </td>
   <td>1i
   </td>
   <td>2021-07-12T19:37:00.000Z
   </td>
  </tr>
  <tr>
   <td>measurement1
   </td>
   <td>tagvalue1
   </td>
   <td>tagvalue4
   </td>
   <td>field1
   </td>
   <td>2i
   </td>
   <td>2021-07-12T19:38:00.000Z
   </td>
  </tr>
</table>

<table>
  <tr>
   <td>_measurement

   </td>
   <td>tag1

   </td>
   <td>tag2

   </td>
   <td>_field

   </td>
   <td>_value

   </td>
   <td>_time

   </td>
  </tr>
  <tr>
   <td>measurement1

   </td>
   <td>tagvalue3

   </td>
   <td>tagvalue4

   </td>
   <td>field1

   </td>
   <td>3i

   </td>
   <td>2021-07-12T19:39:00.000Z

   </td>
  </tr>
 </table>

## Real World Data

There is an excellent [repository of semi-live line protocol data](https://github.com/influxdata/influxdb2-sample-data) maintained by InfluxData. This is generally intended as sample data to help you get started exploring InfluxDB. Currently, 4 datasets are kept up to date there:



1. [Air Sensor Data](https://docs.influxdata.com/influxdb/v2.0/reference/sample-data/#air-sensor-sample-data): This dataset includes a single tag, which is an id for the specific air quality sensor that is reporting  3 fields, temperature, humidity, and carbon monoxide levels. 
2. [Bird Migration Data](https://docs.influxdata.com/influxdb/v2.0/reference/sample-data/#bird-migration-sample-data): This is geo-spatial dataset represents migratory movements of birds. It is tagged to aid geo-spacial querying.
3. [NOAA National Buoy Center Data](https://docs.influxdata.com/influxdb/v2.0/reference/sample-data/#noaa-ndbc-data): This dataset  provides the latest observations from the NOAA NDBC network of buoys. It contains a large number of tags and fields.
4. [USGS Earthquake Data](https://docs.influxdata.com/influxdb/v2.0/reference/sample-data/#usgs-earthquake-data). The United States Geological Survey (USGS) earthquake dataset contains seismic activity data. This is a very large dataset, and contains even more tags and fields.

While you can simply copy the Flux from any of the real world sample datasets into the **Script Editor** in the **Data Explorer** and visualize the data. I recommend creating a bucket and using the to() function to write the data to that bucket, as described in [Write and Query Sample Data]({{site.url}}/docs/part-1/introduction-to-influxdb/#write-and-query-sample-data) in Part 1. Writing the data to a bucket in InfluxDB allows you to use Flux to explore the schema of your dataset. 

**Important Note:** You might hit your series cardinality limit for Free Tier accounts if you write the larger datasets to InfluxDB. I recommend just writing the  [Air Sensor Data](https://docs.influxdata.com/influxdb/v2.0/reference/sample-data/#air-sensor-sample-data) if you’re using the Free Tier account. 

**Important Note:** This section recommends that you use the InfluxDB UI to write data to InfluxDB only because writing data with other tools hasn’t been covered yet. We recommend writing data with the CLI or VS Code. If you prefer developing in with those tools look at the 


### Exploring the Real Word Data Schema with Flux 

Let us turn our attention to 2 real world data sets:  [Air Sensor Data](https://docs.influxdata.com/influxdb/v2.0/reference/sample-data/#air-sensor-sample-data) and the [NOAA National Buoy Center Data](https://docs.influxdata.com/influxdb/v2.0/reference/sample-data/#noaa-ndbc-data). We’ll use Flux to get an understanding of our schema. Then we’ll run through some exercises to ensure our understanding of the relationship between the line protocol input format and the InfluxDB data model as persisted on disk by the storage engine. 

We’ll start by focusing on the Air sensor dataset, as it is the simplest dataset. The a Air sensor dataset contains:



* 1 measurement: airSensors
* 3 fields: 
    * co
    * humidity
    * temperature 
* 1 tag: sensor_id		
    * 8 sensor_id tag values

As you can see, the fields are the actual data, in this case all of type float, where the tag is metadata, defining which sensor produced the data.

[Explore your data schema with Flux](https://docs.influxdata.com/influxdb/cloud/query-data/flux/explore-schema/) to obtain the number of measurements, tag keys, tag values, and field keys in your data by using the schema package. 

To get the number of fields in the airSensors measurement from the Air sensor sample dataset, run the following Flux query in your preferred tool (CLI, VS Code, InfluxDB UI):


```js
import "influxdata/influxdb/schema"

schema.measurementFieldKeys(
  bucket: "Air sensor sample dataset",
  measurement: "airSensors"
)
|> count()
```


 

To get the number of tag keys in the airSensors measurement from the Air sensor sample dataset, run the following Flux query in your preferred tool (CLI, VS Code, InfluxDB UI):


```js
import "influxdata/influxdb/schema"

schema.measurementTagKeys(
  bucket: "Air sensor sample dataset",
  measurement: "airSensors"
)
|> count()
```


To get the number of tag values for the sensor_id tag key in the airSensors measurement from the Air sensor sample dataset, use the following Flux query (CLI, VS Code, InfluxDB UI):


```js
import "influxdata/influxdb/schema"

schema.measurementTagValues(
  bucket: "Air sensor sample dataset",
  tag: "sensor_id",
  measurement: "example-measurement"
)
|> count()
```


We can repeat the same approach for the [NOAA National Buoy Center Data](https://docs.influxdata.com/influxdb/v2.0/reference/sample-data/#noaa-ndbc-data). We find that the [NOAA National Buoy Center Data](https://docs.influxdata.com/influxdb/v2.0/reference/sample-data/#noaa-ndbc-data) has the following schema: 



* 1 measurement: ndbc
* 21 fields:
    * air_temp_degc
    * avg _wave_period_sec
    * dewpoint_temp_degc
    * dominate_wave _period_sec
    * gust_speed_mps
    * lat
    * lon
    * pressure_temdancy_hpa
    * sea_level_pressure_hpa
    * sea_surface_temp_degc
    * significant_Wave_height_m
    * station_currents
    * station_dart
    * station_elev
    * sation_met
    * station_visibility_mei
    * station_waterquality
    * water_level_ft
    * wave_dir_degt
    * wind_dir_degt
    * wind_spead_mps
* 5 tag keys:
    * station_id
        * 113 station_id tag values
    * station_name
        * 828
    * station_owner
        * 57 station_owner tag values
    * station_pgm
        * 6 station_pgm tag values
    * station_type
        * 4 station_type tag values

Again, note that the 21 fields are all the different kinds of data that might be collected by a weather station, whereas the 5 tags contain metadata about which stations collected it.


### Exercises with Real World Data

Now that we understand the schema of the 2 real world data sets, [Air Sensor Data](https://docs.influxdata.com/influxdb/v2.0/reference/sample-data/#air-sensor-sample-data) and the [NOAA National Buoy Center Data](https://docs.influxdata.com/influxdb/v2.0/reference/sample-data/#noaa-ndbc-data), try to answer the following questions to test your understanding of schema design, line protocol, and 

Question 1: How many series will be created by the Air Sensor Data given the schema above? 

Answer 1: (1 sensor_id tag x 8 unique tag values) x (3 fields) = 8 x 3 = **24**

Question 2: How many series will be created by the NOAA National Buoy Center Data given the schema above (assuming no tags are dependent tags)? 

Answer 2: (1 station_id tag x 113 unique tag values) x (1 station_name tag x 828 unique tag values) x (1 station_owner tag x 57 unique tag values) x (1 station_pgm tag x 6 unique tag values) x (1 station_type tag x 47 unique tag values) x (21 fields) = 113 x 828 x 56 x 6 x 47 x 21 = **31028816448**

Question 3: How would the following line protocol form the Air sensor sample dataset be organized into tables on disk? And how many points are in each series?  


```js
airSensors,sensor_id=TLM0100 temperature=71.17615703642676,humidity=35.12940716174776,co=0.5024058630839136 1626537623000000000
airSensors,sensor_id=TLM0101 temperature=71.80350992863588,humidity=34.864121891949736,co=0.4925449578765155 1626537623000000000
airSensors,sensor_id=TLM0102 temperature=72.02673296407973,humidity=34.91147650009415,co=0.4941631223400505 1626537623000000000
airSensors,sensor_id=TLM0103 temperature=71.34822444566278,humidity=35.19576623496297,co=0.4046734235304059 1626537623000000000
airSensors,sensor_id=TLM0200 temperature=73.57230556533555,humidity=35.77102288427073,co=0.5317633226995193 1626537623000000000
airSensors,sensor_id=TLM0201 temperature=72.57230556533555,humidity=35.17327249047271,co=0.5000439017037601 1626537623000000000
airSensors,sensor_id=TLM0202 temperature=75.28582430811852,humidity=35.668729783597556,co=0.48071553398947864 1626537623000000000
airSensors,sensor_id=TLM0203 temperature=74.75927935923579,humidity=35.89268792033798,co=0.4089308476612381 1626537623000000000
airSensors,sensor_id=TLM0100 temperature=71.2194835668512,humidity=35.12891266051405,co=0.4958773037139102 1626537633000000000
airSensors,sensor_id=TLM0101 temperature=71.78232293801005,humidity=34.88621453634278,co=0.5074032895942003 1626537633000000000
airSensors,sensor_id=TLM0102 temperature=72.07101160147653,humidity=34.938529830668536,co=0.5102716855442547 1626537633000000000
airSensors,sensor_id=TLM0103 temperature=71.32889101333731,humidity=35.21581883021604,co=0.4245915521103036 1626537633000000000
airSensors,sensor_id=TLM0200 temperature=73.55081075397399,humidity=35.74330537831752,co=0.5435288991742965 1626537633000000000
airSensors,sensor_id=TLM0201 temperature=74.06284877512215,humidity=35.17611147751894,co=0.4813785832360323 1626537633000000000
airSensors,sensor_id=TLM0202 temperature=75.29425020175684,humidity=35.64366062740866,co=0.4911462705616819 1626537633000000000
airSensors,sensor_id=TLM0203 temperature=74.77142594525142,humidity=35.941017361190255,co=0.42797647488504065 1626537633000000000
```


Answer 3: 

The line protocol would result in the following series and tables on disk. Each series contains two points from the line protocol above. 


<table>
  <tr>
   <td>_measurement
   </td>
   <td>sensor_id
   </td>
   <td>_field
   </td>
   <td>_value
   </td>
   <td>_time
   </td>
  </tr>
  <tr>
   <td>airSensors
   </td>
   <td>TLM0100
   </td>
   <td>temperature
   </td>
   <td>73.57230556533555
   </td>
   <td>rfc3339time1
   </td>
  </tr>
  <tr>
   <td>airSensors
   </td>
   <td>TLM0100
   </td>
   <td>temperature
   </td>
   <td>71.2194835668512
   </td>
   <td>rfc3339time2
   </td>
  </tr>
</table>



<table>
  <tr>
   <td>_measurement
   </td>
   <td>sensor_id
   </td>
   <td>_field
   </td>
   <td>_value
   </td>
   <td>_time
   </td>
  </tr>
  <tr>
   <td>airSensors
   </td>
   <td>TLM0101
   </td>
   <td>temperature
   </td>
   <td>72.02673296407973
   </td>
   <td>rfc3339time1
   </td>
  </tr>
  <tr>
   <td>airSensors
   </td>
   <td>TLM0101
   </td>
   <td>temperature
   </td>
   <td>71.78232293801005
   </td>
   <td>rfc3339time2
   </td>
  </tr>
</table>



<table>
  <tr>
   <td>_measurement
   </td>
   <td>sensor_id
   </td>
   <td>_field
   </td>
   <td>_value
   </td>
   <td>_time
   </td>
  </tr>
  <tr>
   <td>airSensors
   </td>
   <td>TLM0102
   </td>
   <td>temperature
   </td>
   <td>73.57230556533555
   </td>
   <td>rfc3339time1
   </td>
  </tr>
  <tr>
   <td>airSensors
   </td>
   <td>TLM0102
   </td>
   <td>temperature
   </td>
   <td>72.07101160147653
   </td>
   <td>rfc3339time2
   </td>
  </tr>
</table>



<table>
  <tr>
   <td>_measurement
   </td>
   <td>sensor_id
   </td>
   <td>_field
   </td>
   <td>_value
   </td>
   <td>_time
   </td>
  </tr>
  <tr>
   <td>airSensors
   </td>
   <td>TLM0103
   </td>
   <td>temperature
   </td>
   <td>71.34822444566278
   </td>
   <td>rfc3339time1
   </td>
  </tr>
  <tr>
   <td>airSensors
   </td>
   <td>TLM0103
   </td>
   <td>temperature
   </td>
   <td>71.32889101333731
   </td>
   <td>rfc3339time2
   </td>
  </tr>
</table>



<table>
  <tr>
   <td>_measurement
   </td>
   <td>sensor_id
   </td>
   <td>_field
   </td>
   <td>_value
   </td>
   <td>_time
   </td>
  </tr>
  <tr>
   <td>airSensors
   </td>
   <td>TLM0200
   </td>
   <td>temperature
   </td>
   <td>73.57230556533555
   </td>
   <td>rfc3339time1
   </td>
  </tr>
  <tr>
   <td>airSensors
   </td>
   <td>TLM0200
   </td>
   <td>temperature
   </td>
   <td>73.55081075397399
   </td>
   <td>rfc3339time2
   </td>
  </tr>
</table>



<table>
  <tr>
   <td>_measurement
   </td>
   <td>sensor_id
   </td>
   <td>_field
   </td>
   <td>_value
   </td>
   <td>_time
   </td>
  </tr>
  <tr>
   <td>airSensors
   </td>
   <td>TLM0201
   </td>
   <td>temperature
   </td>
   <td>73.57230556521233
   </td>
   <td>rfc3339time1
   </td>
  </tr>
  <tr>
   <td>airSensors
   </td>
   <td>TLM0201
   </td>
   <td>temperature
   </td>
   <td>74.06284877512215
   </td>
   <td>rfc3339time2
   </td>
  </tr>
</table>



<table>
  <tr>
   <td>_measurement
   </td>
   <td>sensor_id
   </td>
   <td>_field
   </td>
   <td>_value
   </td>
   <td>_time
   </td>
  </tr>
  <tr>
   <td>airSensors
   </td>
   <td>TLM0202
   </td>
   <td>temperature
   </td>
   <td>75.28582430811852
   </td>
   <td>rfc3339time1
   </td>
  </tr>
  <tr>
   <td>airSensors
   </td>
   <td>TLM0202
   </td>
   <td>temperature
   </td>
   <td>75.29425020175684
   </td>
   <td>rfc3339time2
   </td>
  </tr>
</table>



<table>
  <tr>
   <td>_measurement
   </td>
   <td>sensor_id
   </td>
   <td>_field
   </td>
   <td>_value
   </td>
   <td>_time
   </td>
  </tr>
  <tr>
   <td>airSensors
   </td>
   <td>TLM0203
   </td>
   <td>temperature
   </td>
   <td>74.75927935923579
   </td>
   <td>rfc3339time1
   </td>
  </tr>
  <tr>
   <td>airSensors
   </td>
   <td>TLM0203
   </td>
   <td>temperature
   </td>
   <td>74.77142594525142
   </td>
   <td>rfc3339time2
   </td>
  </tr>
</table>



<table>
  <tr>
   <td>_measurement
   </td>
   <td>sensor_id
   </td>
   <td>_field
   </td>
   <td>_value
   </td>
   <td>_time
   </td>
  </tr>
  <tr>
   <td>airSensors
   </td>
   <td>TLM0100
   </td>
   <td>humidity
   </td>
   <td>35.12940716174776
   </td>
   <td>rfc3339time1
   </td>
  </tr>
  <tr>
   <td>airSensors
   </td>
   <td>TLM0100
   </td>
   <td>humidity
   </td>
   <td>35.12891266051405
   </td>
   <td>rfc3339time2
   </td>
  </tr>
</table>



<table>
  <tr>
   <td>_measurement
   </td>
   <td>sensor_id
   </td>
   <td>_field
   </td>
   <td>_value
   </td>
   <td>_time
   </td>
  </tr>
  <tr>
   <td>airSensors
   </td>
   <td>TLM0101
   </td>
   <td>humidity
   </td>
   <td>34.864121891949736
   </td>
   <td>rfc3339time1
   </td>
  </tr>
  <tr>
   <td>airSensors
   </td>
   <td>TLM0101
   </td>
   <td>humidity
   </td>
   <td>34.88621453634278
   </td>
   <td>rfc3339time2
   </td>
  </tr>
</table>



<table>
  <tr>
   <td>_measurement
   </td>
   <td>sensor_id
   </td>
   <td>_field
   </td>
   <td>_value
   </td>
   <td>_time
   </td>
  </tr>
  <tr>
   <td>airSensors
   </td>
   <td>TLM0102
   </td>
   <td>humidity
   </td>
   <td>34.91147650009415
   </td>
   <td>rfc3339time1
   </td>
  </tr>
  <tr>
   <td>airSensors
   </td>
   <td>TLM0102
   </td>
   <td>humidity
   </td>
   <td>34.938529830668536
   </td>
   <td>rfc3339time2
   </td>
  </tr>
</table>



<table>
  <tr>
   <td>_measurement
   </td>
   <td>sensor_id
   </td>
   <td>_field
   </td>
   <td>_value
   </td>
   <td>_time
   </td>
  </tr>
  <tr>
   <td>airSensors
   </td>
   <td>TLM0103
   </td>
   <td>humidity
   </td>
   <td>35.19576623496297
   </td>
   <td>rfc3339time1
   </td>
  </tr>
  <tr>
   <td>airSensors
   </td>
   <td>TLM0103
   </td>
   <td>humidity
   </td>
   <td>35.21581883021604
   </td>
   <td>rfc3339time2
   </td>
  </tr>
</table>



<table>
  <tr>
   <td>_measurement
   </td>
   <td>sensor_id
   </td>
   <td>_field
   </td>
   <td>_value
   </td>
   <td>_time
   </td>
  </tr>
  <tr>
   <td>airSensors
   </td>
   <td>TLM0200
   </td>
   <td>humidity
   </td>
   <td>35.77102288427073
   </td>
   <td>rfc3339time1
   </td>
  </tr>
  <tr>
   <td>airSensors
   </td>
   <td>TLM0200
   </td>
   <td>humidity
   </td>
   <td>35.74330537831752
   </td>
   <td>rfc3339time2
   </td>
  </tr>
</table>



<table>
  <tr>
   <td>_measurement
   </td>
   <td>sensor_id
   </td>
   <td>_field
   </td>
   <td>_value
   </td>
   <td>_time
   </td>
  </tr>
  <tr>
   <td>airSensors
   </td>
   <td>TLM0201
   </td>
   <td>humidity
   </td>
   <td>35.17327249047271
   </td>
   <td>rfc3339time1
   </td>
  </tr>
  <tr>
   <td>airSensors
   </td>
   <td>TLM0201
   </td>
   <td>humidity
   </td>
   <td>35.17611147751894
   </td>
   <td>rfc3339time2
   </td>
  </tr>
</table>



<table>
  <tr>
   <td>_measurement
   </td>
   <td>sensor_id
   </td>
   <td>_field
   </td>
   <td>_value
   </td>
   <td>_time
   </td>
  </tr>
  <tr>
   <td>airSensors
   </td>
   <td>TLM0202
   </td>
   <td>humidity
   </td>
   <td>35.668729783597556
   </td>
   <td>rfc3339time1
   </td>
  </tr>
  <tr>
   <td>airSensors
   </td>
   <td>TLM0202
   </td>
   <td>humidity
   </td>
   <td>35.64366062740866
   </td>
   <td>rfc3339time2
   </td>
  </tr>
</table>



<table>
  <tr>
   <td>_measurement
   </td>
   <td>sensor_id
   </td>
   <td>_field
   </td>
   <td>_value
   </td>
   <td>_time
   </td>
  </tr>
  <tr>
   <td>airSensors
   </td>
   <td>TLM0203
   </td>
   <td>humidity
   </td>
   <td>35.89268792033798
   </td>
   <td>rfc3339time1
   </td>
  </tr>
  <tr>
   <td>airSensors
   </td>
   <td>TLM0203
   </td>
   <td>humidity
   </td>
   <td>35.941017361190255
   </td>
   <td>rfc3339time2
   </td>
  </tr>
</table>



<table>
  <tr>
   <td>_measurement
   </td>
   <td>sensor_id
   </td>
   <td>_field
   </td>
   <td>_value
   </td>
   <td>_time
   </td>
  </tr>
  <tr>
   <td>airSensors
   </td>
   <td>TLM0100
   </td>
   <td>co
   </td>
   <td>0.4925449578765155
   </td>
   <td>rfc3339time1
   </td>
  </tr>
  <tr>
   <td>airSensors
   </td>
   <td>TLM0100
   </td>
   <td>co
   </td>
   <td>0.495877303713910
   </td>
   <td>rfc3339time2
   </td>
  </tr>
</table>



<table>
  <tr>
   <td>_measurement
   </td>
   <td>sensor_id
   </td>
   <td>_field
   </td>
   <td>_value
   </td>
   <td>_time
   </td>
  </tr>
  <tr>
   <td>airSensors
   </td>
   <td>TLM0101
   </td>
   <td>co
   </td>
   <td>0.4925449578765155
   </td>
   <td>rfc3339time1
   </td>
  </tr>
  <tr>
   <td>airSensors
   </td>
   <td>TLM0101
   </td>
   <td>co
   </td>
   <td>0.5074032895942003
   </td>
   <td>rfc3339time2
   </td>
  </tr>
</table>



<table>
  <tr>
   <td>_measurement
   </td>
   <td>sensor_id
   </td>
   <td>_field
   </td>
   <td>_value
   </td>
   <td>_time
   </td>
  </tr>
  <tr>
   <td>airSensors
   </td>
   <td>TLM0102
   </td>
   <td>co
   </td>
   <td>0.4941631223400505
   </td>
   <td>rfc3339time1
   </td>
  </tr>
  <tr>
   <td>airSensors
   </td>
   <td>TLM0102
   </td>
   <td>co
   </td>
   <td>0.5102716855442547
   </td>
   <td>rfc3339time2
   </td>
  </tr>
</table>



<table>
  <tr>
   <td>_measurement
   </td>
   <td>sensor_id
   </td>
   <td>_field
   </td>
   <td>_value
   </td>
   <td>_time
   </td>
  </tr>
  <tr>
   <td>airSensors
   </td>
   <td>TLM0103
   </td>
   <td>co
   </td>
   <td>0.4046734235304059
   </td>
   <td>rfc3339time1
   </td>
  </tr>
  <tr>
   <td>airSensors
   </td>
   <td>TLM0103
   </td>
   <td>co
   </td>
   <td>0.4245915521103036
   </td>
   <td>rfc3339time2
   </td>
  </tr>
</table>



<table>
  <tr>
   <td>_measurement
   </td>
   <td>sensor_id
   </td>
   <td>_field
   </td>
   <td>_value
   </td>
   <td>_time
   </td>
  </tr>
  <tr>
   <td>airSensors
   </td>
   <td>TLM0200
   </td>
   <td>co
   </td>
   <td>0.5317633226995193
   </td>
   <td>rfc3339time1
   </td>
  </tr>
  <tr>
   <td>airSensors
   </td>
   <td>TLM0200
   </td>
   <td>co
   </td>
   <td>0.5435288991742965
   </td>
   <td>rfc3339time2
   </td>
  </tr>
</table>



<table>
  <tr>
   <td>_measurement
   </td>
   <td>sensor_id
   </td>
   <td>_field
   </td>
   <td>_value
   </td>
   <td>_time
   </td>
  </tr>
  <tr>
   <td>airSensors
   </td>
   <td>TLM0201
   </td>
   <td>co
   </td>
   <td>0.5000439017037601
   </td>
   <td>rfc3339time1
   </td>
  </tr>
  <tr>
   <td>airSensors
   </td>
   <td>TLM0201
   </td>
   <td>co
   </td>
   <td>0.4813785832360323
   </td>
   <td>rfc3339time2
   </td>
  </tr>
</table>



<table>
  <tr>
   <td>_measurement
   </td>
   <td>sensor_id
   </td>
   <td>_field
   </td>
   <td>_value
   </td>
   <td>_time
   </td>
  </tr>
  <tr>
   <td>airSensors
   </td>
   <td>TLM0202
   </td>
   <td>co
   </td>
   <td>0.48071553398947864
   </td>
   <td>rfc3339time1
   </td>
  </tr>
  <tr>
   <td>airSensors
   </td>
   <td>TLM0202
   </td>
   <td>co
   </td>
   <td>0.4911462705616819
   </td>
   <td>rfc3339time2
   </td>
  </tr>
</table>

<table>
  <tr>
   <td>_measurement

   </td>
   <td>sensor_id

   </td>
   <td>_field

   </td>
   <td>_value

   </td>
   <td>_time

   </td>
  </tr>
  <tr>
   <td>airSensors

   </td>
   <td>TLM0203

   </td>
   <td>co

   </td>
   <td>0.4089308476612381

   </td>
   <td>rfc3339time1

   </td>
  </tr>
  <tr>
   <td>airSensors

   </td>
   <td>TLM0203

   </td>
   <td>co

   </td>
   <td>0.42797647488504065

   </td>
   <td>rfc3339time2

   </td>
  </tr>
</table>

[Next Section]({{site.url}}/docs/part-2/designing-your-schema){: .btn .btn-purple}

## Further Reading
1.  [TL;DR InfluxDB Tech Tips – How to Interpret an Annotated CSV](https://www.influxdata.com/blog/tldr-tech-tips-how-to-interpret-an-annotated-csv/)
2.  [Line Protocol](https://docs.influxdata.com/influxdb/v2.1/reference/syntax/line-protocol/)
