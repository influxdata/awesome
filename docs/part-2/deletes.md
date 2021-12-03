---
layout: default
title: Deletes
parent: Part 2
nav_order: 6
---

# Deletes
{: .no_toc }

## Table of contents
{: .no_toc .text-delta }

1. TOC
{:toc}

---
## Deletes

As a Time Series database, InfluxDB is not optimized for interactive deletes. Typically, data is deleted by aging out due to a bucket's retention policy. As such, delete operations in InfluxDB are computationally expensive and slow compared to writes and queries. It is possible to delete data that has not aged out due to retention policies, but one must be aware of the performance implications.

You can delete time series data with either:

1.  The InfluxDB CLI command [influx delete](https://docs.influxdata.com/influxdb/cloud/write-data/delete-data/).
2.  The InfluxDB v2 API [delete endpoint](https://docs.influxdata.com/influxdb/cloud/api/#operation/PostDelete). 

Use the InfluxDB CLI to delete data from a bucket within a specific time range, measurement, and tags. For example, if we wanted to delete some sensor data from the Air  Sensor sample dataset we could use the following command: 


```
influx delete --bucket 'Air sensor sample dataset' \
  --start 'rfc3339time1' \
  --stop 'rfc3339time2' \
  --predicate '_measurement="airSensors" AND sensor_id="TM0100"'
```


Remember the `rfc3339time` timestamps are in  the form of` %Y-%m-%dT%H:%M:%SZ,` with an example being: `2020-03-01T00:00:00Z`. After making the delete, the following query returns no results: \



```js
from(bucket: "Air sensor sample dataset")
  |> range(start: rfc33391, stop: rfc33392)
  |> filter(fn: (r) => r._measurement == "airSensors")
  |> filter(fn: (r) => r.sensor_id == "TM0100")
```


Similarly, you can use the InfluxDB v2 API to delete the same data with: 


```
curl -i -XPOST '<yourInflxDBURL>api/v2/delete?orgID=<yourOrgID>bucket=<yourBucketID>\
  --header 'Authorization: Token <yourToken>' \
  --data-raw  '{
"start": "rfc3339time1",
"stop": "rfc3339time2",
"predicate": '_measurement=\"airSensors\" and sensor_id=\"TM0100\"'
}'
```


Remember you’ll want to replace `<yourInflxDBURL>` with either:



* The [InfluxDB OSS UR](https://docs.influxdata.com/influxdb/v2.0/reference/urls/) 
    * <code>[http://localhost:8086/](http://localhost:8086/)</code>  by default
* The [InfluxDB Cloud URL](https://docs.influxdata.com/influxdb/cloud/reference/regions/) from your correct region and provider
    * <code>https://us-west-2-1.aws.cloud2.influxdata.com/</code> for example

Please review Part 1, to learn about how to gather your <code><yourBucketID></code> and <code><yourOrgID></code>. 

### Deleting Buckets
As mentioned above, the most common way to delete data is to simply allow the data to age out due to retention policites. However, their may be times when you wish to delete a large amount of data. This can be achieved by deleting a whole bucket. If you need to retain some of the data in the bucket, you can use the following procedure:
  1. Create a new bucket, with an arbitrary name.
  2. Create a query to retrieve the data that you would like to keep, and copy that into the new bucket.
  3. Delete the original bucket.
  4. Rename the new bucket to the original bucket name, so that existing queries continue working (assuming that the queries use the bucket name and no the bucket id).
  
  
[Next Section]({{site.baseurl}}/docs/part-2/optimizing-flux-performance){: .btn .btn-purple}
  
