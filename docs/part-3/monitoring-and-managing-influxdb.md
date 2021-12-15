---
layout: default
title: Monitoring and Managing InfluxDB
parent: Part 3
nav_order: 6
---

# Monitoring and Managing InfluxDB
{: .no_toc }

## Table of contents
{: .no_toc .text-delta }

1. TOC
{:toc}

---

# The Operational Monitoring Template 

So you’re using InfluxDB Cloud, and you’re writing millions of metrics to your account. You’re also running a variety of downsampling and data transformation tasks. Whether you’re building an [IoT application on top of InfluxDB](https://github.com/bonitoo-io/influxdata-iot-petstore) or monitoring your production environment with InfluxDB, your time series operations are finally running smoothly. You want to keep it that way. You might be a [Free Plan Cloud user](https://www.influxdata.com/influxdb-cloud-pricing/) or a [Usage-Based Plan user](https://www.influxdata.com/blog/influxdb-cloud-pricing-control-transparency/), but either way, you need visibility into your instance size to manage resources and costs. Use the [InfluxDB Operational Monitoring Template](https://github.com/influxdata/community-templates/tree/master/influxdb2_operational_monitoring) to protect against runaway series cardinality and ensure that your tasks are successful.


## Install and Use the Operational Monitoring Template

In case you’re new to InfluxData, [InfluxDB Templates](https://docs.influxdata.com/influxdb/v2.0/influxdb-templates/) are a preconfigured and shareable collection of dashboards, tasks, alerts, Telegraf configurations, and more. You apply them by copying the URL of a template of interest from the [community templates](https://github.com/influxdata/community-templates) and pasting it into the UI. Today, we’ll apply the Operational Monitoring Template to our InfluxDB Cloud account to monitor our cardinality and task execution.

![install operational monitoring template]({{site.url}}/assets/images/part-3/monitoring-and-managing-influxdb/1-install-operational-monitoring-template.jpg "image_tooltip")

An example of how to apply a template through the UI. Navigate to the Templates tab on the Settings page and paste the link to the YAML or JSON of the template you want to apply.

The Operational Monitoring Template consists of the following resources:



    1. 1 [Bucket](https://docs.influxdata.com/influxdb/v2.0/organizations/buckets/): cardinality
    2. 1 [Task](https://docs.influxdata.com/influxdb/v2.0/process-data/): cardinality_by_bucket that runs every hour. This task is responsible for calculating the cardinality of all of your buckets and writing the calculation to the cardinality bucket.
    3. 1 [Label](https://docs.influxdata.com/influxdb/v2.0/visualize-data/labels/): operational_monitoring. This label helps you find the resources that are part of this template more easily within the UI.
    4. 2 [Variables](https://docs.influxdata.com/influxdb/v2.0/visualize-data/variables/): bucket and measurement.
    5. 2 [Dashboards](https://docs.influxdata.com/influxdb/v2.0/visualize-data/dashboards/):
        * Task Summary Dashboard. This dashboard displays information about your task runs, successes and failures.
        * Cardinality Explorer. This dashboard helps you visualize your series cardinality for all of the buckets. Drill down into your measurement cardinality to [solve your runaway cardinality](https://www.influxdata.com/blog/solving-runaway-series-cardinality-when-using-influxdb/) problem.


## Understanding the Task Summary dashboard

Both the Task Summary dashboard and the Cardinality Explorer dashboard can be used to monitor your specific task execution status and series growth. The Task Summary dashboard looks like this:


![task summary dashboard]({{site.url}}/assets/images/part-3/monitoring-and-managing-influxdb/2-task-summary-dashboard-1.jpg "image_tooltip")
An example of the Task Summary Dashboard. It provides information about the success of task runs for the past hour controlled by the time range dropdown configuration (pink square).

The first three cells of this dashboard allow you to easily evaluate the success of all of your tasks for the specified time range. From this screenshot we see that we have 80 failed task runs. The cell in the second row, allows us to easily sort by the most successful and failed runs per task within the time range. It provides us with the error rate and task ID.


![task summary dashboard]({{site.url}}/assets/images/part-3/monitoring-and-managing-influxdb/3-task-summary-dashboard-2.jpg "image_tooltip")
More of the Task Summary Dashboard. It provides information about the success of task runs for the past hour.

The dashboard also includes an Error List cell. This cell contains information about all of the failed runs, complete with the individual errorMessages for each runID. You can use this cell to determine why a task is failing. Below the Error Rates Over Time allows you to easily determine whether you have an increasing number of failed tasks. A linear line indicates that the same set of tasks are failing whereas an exponential line would suggest that an increasing number of tasks are failing.

![task summary dashboard]({{site.url}}/assets/images/part-3/monitoring-and-managing-influxdb/4-task-summary-dashboard-3.jpg "image_tooltip")
More of the Task Summary Dashboard. It provides information about the success of task runs for the past hour.

The last cell of the Task Summary Dashboard, Last Successful Run Per Task helps you identify when a task failed so you can more easily debug your task. Now that we’ve reviewed the features of the Task Summary Dashboard, let’s examine the Cardinality Explorer dashboard.


## Supplement the Template with your own Task Alerts

If you are running Tasks that are mission-critical to your time series use case, I recommend setting up Alerts to monitor when those Tasks are failing. Use the Task Summary Dashboard to help you find the taskID associated with the task you want to alert on. Now we need to convert our status into a numeric value in order to create a threshold alert for failed tasks. Next, navigate to the Data Explorer. Use the [map()](https://docs.influxdata.com/influxdb/v2.0/reference/flux/stdlib/built-in/transformations/map/) function to convert the status into a boolean with the following Flux script:


```
from(bucket: "_tasks")
|> range(start: v.timeRangeStart, stop: v.timeRangeStop)
|> filter(fn: (r) => r["taskID"] == "05ff8e19b32dc000")
|> map(fn: (r) => ({ r with status_bool: if r.status == "success" then 1 else 0}))
```


Click the **Save As** button, to convert this to an [Alert](https://docs.influxdata.com/influxdb/v2.0/monitor-alert/). Then create a threshold alert and alert if the value of status_bool is below 1.


![task alert]({{site.url}}/assets/images/part-3/monitoring-and-managing-influxdb/5-task-alert.jpg "image_tooltip")



## Understanding the Cardinality Explorer Dashboard

The Cardinality Explorer Dashboard provides the cardinality per bucket in your InfluxDB instance.


![cardinality explorer dashboard]({{site.url}}/assets/images/part-3/monitoring-and-managing-influxdb/6-cardinality-explorer-dashboard-1.jpg "image_tooltip")
An example of the Cardinality Explorer Dashboard. It provides information about the cardinality of your InfluxDB instance. Labels allow you to visualize metrics with a dropdown configuration (orange).

The variables at the top of the dashboard allow you to easily switch between buckets and measurements.



* The graph in the first cell, Cardinality by Bucket, allows you to monitor the cardinality of all of your buckets at a glance. We can see that our cardinality is remaining within the same range. The fact that the cardinality across all buckets has a small standard deviation is a good sign. It allows us to verify that we don’t have a series cardinality problem. It might be advantageous to create [additional alerts](https://docs.influxdata.com/influxdb/v2.0/monitor-alert/) on top of this bucket to notify you when your cardinality exceeds an expected standard deviation. Dips in the cardinality are likely evidence of successful data expiration or perhaps a [data deletion](https://docs.influxdata.com/influxdb/v2.0/write-data/delete-data/). Spikes in the cardinality could be due to momentary overlap between new data ingested and old data about to be evicted. It’s also likely that the spikes are due to an important event.


![cardinality explorer dashboard]({{site.url}}/assets/images/part-3/monitoring-and-managing-influxdb/7-cardinality-explorer-dashboard-2.jpg "image_tooltip")




* The second cell, Top Buckets, provides a table view of the largest buckets and their cardinality.
* The third cell, Cardinality by Measurement, allows you to drill into specific measurements for each bucket, selected by the bucket and measurement variables. This is especially useful for finding the source of your runaway series cardinality, if you were to experience it.
* Use the final cell (not pictured), Cardinality by Tag, to find unbounded tags. It’s worth noting that the Flux script to execute these calculations is fairly clever.

In the next section, we’ll learn about how the cardinality_by_bucket task works to help us understand how these cells produce this cardinality data as the queries follow the same logic as the task.


## The cardinality task explained

The cardinality task, cardinality_by_bucket, consists of the following Flux code:


```js
import "influxdata/influxdb"
        buckets()
        	|> map(fn: (r) => {
        		cardinality = influxdb.cardinality(bucket: r.name, start: -task.every)
        			|> findRecord(idx: 0, fn: (key) =>
        				(true))
        		return {
        			_time: now(),
        			_measurement: "buckets",
        			bucket: r.name,
        			_field: "cardinality",
        			_value: cardinality._value,
        		}
        	})
        	|> to(bucket: "cardinality")
```


The [Flux influxdata/influxdb](https://docs.influxdata.com/influxdb/v2.0/reference/flux/stdlib/influxdb/) package currently contains one Flux function, [influxdb.cardinality()](https://docs.influxdata.com/influxdb/v2.0/reference/flux/stdlib/influxdb/cardinality/). This function returns the series cardinality of data within a specific bucket. The [buckets()](https://docs.influxdata.com/influxdb/v2.0/reference/flux/stdlib/built-in/inputs/buckets/) function returns a list of buckets. The [map()](https://docs.influxdata.com/influxdb/v2.0/reference/flux/stdlib/built-in/transformations/map/) function here is effectively “iterating” through the list of bucket names. It uses [findRecord()](https://docs.influxdata.com/influxdb/v2.0/reference/flux/stdlib/built-in/transformations/stream-table/findrecord/) in a [custom function](https://docs.influxdata.com/influxdb/v2.0/query-data/flux/custom-functions/#function-definition-structure), `cardinality`, to return the series cardinality of each bucket. The return statement creates a new table with the corresponding  r.name for the field key, and `cardinality._value` for the field value. This data is then written to our “cardinality” bucket.


## Supplement the Template with your own Cardinality Alerts

To gain the full benefits from this template, I recommend setting up a cardinality threshold alert based off of the cardinality bucket. The fastest way to create an alert is through the UI. Navigate to the **Alerts** tab in the UI. First query for your cardinality. Although it’s hard to visualize our cardinality data in the graph visualization, the default visualization type for the **Alerts** UI, I can hover over the points or view the data in the **Raw Data View** to understand my data better.


![cardinality alert]({{site.url}}/assets/images/part-3/monitoring-and-managing-influxdb/8-cardinality-alert-define-query.jpg "image_tooltip")


I notice that my cardinality data can be split into roughly three ranges. Next, I will configure alerts around those ranges. If the cardinality in my buckets crosses one of those three thresholds, I will receive an Alert.


![cardinality alert]({{site.url}}/assets/images/part-3/monitoring-and-managing-influxdb/9-cardinality-alert-thresholds.jpg "image_tooltip")


This type of Alert can help you guard against reaching a cardinality limit in InfluxDB Cloud. Specifically, free tier and Usage-Based Plan users want to be alerted when they approach 80% their cardinality limit of 10,000 and initial cardinality limit of 1M, respectively. You might also consider creating a task that sums the cardinality across all buckets and alerting on the total cardinality as well.

If you notice that your cardinality is growing significantly, you can decide whether or not you need to request a higher cardinality limit or whether you need to downsample historical data. To request a higher cardinality limit, click the **Upgrade Now** button on the top right of the InfluxDB UI.


## Initial data delay for the Cardinality Explorer Dashboard

Please note that the Cardinality Explorer dashboard will be empty until the `cardinality_by_bucket` task has run. It runs on an interval of 1hr, so it might take a while before that dashboard is populated with meaningful data. However, if you need to visualize your cardinality data immediately, you can repurpose the script in the cardinality_by_bucket task, and select the time range you wish to gather cardinality data from. Then you can write the cardinality data to the cardinality bucket with the [to()](https://docs.influxdata.com/influxdb/cloud/reference/flux/stdlib/built-in/outputs/to/) function if you desire. Use the [subDuration()](https://docs.influxdata.com/influxdb/cloud/query-data/flux/manipulate-timestamps/#subtract-a-duration-from-a-timestamp) function to create the correct timestamp.


```js
import "influxdata/influxdb"
import "experimental"

time = experimental.subDuration(
  d: 7d,
  from: now(),
)
buckets()
	|> map(fn: (r) => {
		cardinality = influxdb.cardinality(bucket: r.name, start: -7d)
			|> findRecord(idx: 0, fn: (key) =>
				(true))
		return {
			_time: time,
			_measurement: "buckets",
			bucket: r.name,
			_field: "cardinality",
			_value: cardinality._value,
		}
	})
```

# The InfluxDB Cloud Usage Template

Use the [InfluxDB Cloud Usage Template](https://www.influxdata.com/influxdb-templates/influxdb-cloud-usage-dashboard/) to gain visibility into your usage and take proactive action before you hit your limits. In this section, we’ll apply the InfluxDB Cloud Usage Template to our InfluxDB Cloud account to monitor our usage and rate-limiting events. Rate-limiting events occur when a user exceeds the usage limits of their Cloud account. Use the following command to apply the InfluxDB Cloud Usage Template through the CLI:


```
influx apply --file https://raw.githubusercontent.com/influxdata/community-templates/master/usage_dashboard/usage_dashboard.yml
```


Alternatively, you can use the UI to apply a template. Navigate to the **Templates** tab on the Settings page and paste the link to the YAML or JSON of the template you want to apply.


![alt_text]({{site.url}}/assets/images/part-3/monitoring-and-managing-influxdb/10-usage-dashboard.png "image_tooltip")


The InfluxDB Cloud Usage Template consists of the following resources:



* 1 [Dashboard](https://docs.influxdata.com/influxdb/v2.0/visualize-data/dashboards/): Usage Dashboard. This dashboard enables you to track your InfluxDB Cloud data usage and limit events.
* 1 [Task](https://docs.influxdata.com/influxdb/v2.0/process-data/): Cardinality Limit Alert (usage dashboard) that runs every hour. This task is responsible for determining whether or not your cardinality has exceeded the cardinality limit associated with your Cloud account.
* 1 [Label](https://docs.influxdata.com/influxdb/v2.0/visualize-data/labels/): usage_dashboard. This label helps you find the resources that are part of this template more easily within the UI and through the API.


## The Usage Dashboard explained

The InfluxDB Cloud Usage Template contains the Usage Dashboard which gives you visibility into your Data In, Query Count, Storage, Data Out, and Rate-Limiting Events.


![alt_text]({{site.url}}/assets/images/part-3/monitoring-and-managing-influxdb/11-usage-dashboard.png "image_tooltip")


The cells in the dashboard display the following information:



* The Data In (Graph) visualizes the total bytes written to your InfluxDB Org through the [/write endpoint](https://docs.influxdata.com/influxdb/v2.0/api/#operation/PostWrite).
* The Query Count (Graph) visualizes the number of queries executed.
* The Storage (By Bucket) visualizes the number of total bytes in each bucket and all buckets overall.
* The Data Out (Graph) visualizes the total bytes queried from your InfluxDB organization through the [/query endpoint](https://docs.influxdata.com/influxdb/v2.0/api/#operation/PostQuery).
* The Rate Limit Events visualizes the number of times limits has been reached
* The Rate Limit Events are split into event types in the bottom right cells to display the number of write limit, query limit, and cardinality limit events as well as the total cardinality limit for your organization.

All of these visualizations automatically aggregate the data to 1hr resolution. Essentially, this dashboard supplements the data that exists in the Usage page in the InfluxDB UI to give you even more insights into your usage.


![alt_text]({{site.url}}/assets/images/part-3/monitoring-and-managing-influxdb/12-usage-page.png "image_tooltip")
_The Usage page in the InfluxDB UI contains general information about your usage._


## Understanding the Flux behind the Usage Dashboard

In order to fully understand the InfluxDB Cloud Usage Template, you must understand the Flux behind the Usage Dashboard. Let’s take a look at the query that produces the Data In (Graph) visualization in the first cell.


```
import "math"
import "experimental/usage"


usage.from(
start: v.timeRangeStart,
stop: v.timeRangeStop,
)
|> filter(fn: (r) =>
r._measurement == "http_request"
and (r.endpoint == "/api/v2/write" or r.endpoint == "/write")
and r._field == "req_bytes"
)
|> group()
|> keep(columns: ["_value", "_field", "_time"])
|> fill(column: "_value", value: 0)
|> map(fn: (r) =>
({r with
write_mb: math.round(x: float(v: r._value) / 10000.0) / 100.0
}))
```


First we import all the relative packages to execute the Flux query that generates the visualization we desire. The Flux math package provides basic mathematical functions and constants. The Flux experimental/usage package provides functions for collecting usage and usage limit data related to your InfluxDB Cloud account. All of the cells, dashboards, and tasks in the InfluxDB Cloud Usage Template use the Flux experimental/usage package. The [usage.from()](https://docs.influxdata.com/influxdb/cloud/reference/flux/stdlib/experimental/usage/from/) function returns usage data. We filter the usage data for the “req_bytes” field from any writes. Then we group write data, fill any empty rows with 0, and calculate the number of megabytes and store the value in a new column, “write_mb”.


## Exploring the experimental/usage Flux Package

The experimental/usage package contains two functions: [usage.from()](https://docs.influxdata.com/influxdb/cloud/reference/flux/stdlib/experimental/usage/from/) and [usage.limits()](https://docs.influxdata.com/influxdb/cloud/reference/flux/stdlib/experimental/usage/limits/).

The [usage.from()](https://docs.influxdata.com/influxdb/cloud/reference/flux/stdlib/experimental/usage/from/) function maps to the [org/{orgID}/usage endpoint](https://docs.influxdata.com/influxdb/v2.0/api/#operation/GetOrgsID) in the InfluxDB v2 API. The [usage.from()](https://docs.influxdata.com/influxdb/cloud/reference/flux/stdlib/experimental/usage/from/) function returns usage data for your InfluxDB Organization. The [usage.from()](https://docs.influxdata.com/influxdb/cloud/reference/flux/stdlib/experimental/usage/from/) function requires that you specify the `start` and `stop` parameters. By default, the function returns aggregated usage data as an hourly sum. This default downsampling enables you to query usage data over long periods of time efficiently. If you want to view raw high-resolution data, query your data over a short time range (a few hours) and set `raw:true`. By default the  [usage.from()](https://docs.influxdata.com/influxdb/cloud/reference/flux/stdlib/experimental/usage/from/) returns usage data from the InfluxDB Org you’re currently using. To query an outside InfluxDB Cloud Org, supply the `host`, `orgID`, and `token` parameters. The [usage.from()](https://docs.influxdata.com/influxdb/cloud/reference/flux/stdlib/experimental/usage/from/) returns usage data with the following schema:



* _time: the time of the usage event  (`raw:true`) or downsampled usage data (`raw:false`).
* _bucket_id: the bucket id for each bucket specific usage, like the storage_usage_bucket_bytes measurement — tag.
* _org_id: the org id for which usage data is being queried from — tag.
* _endpoint: the InfluxDB v2 API endpoint from where usage data is being collected— tag
* _status: the HTTP status codes — a tag.
* _measurement: the measurements.
    * http_request
    * storage_usage_buckets_bytes
    * query _count
    * events
* _field: the field keys
    * resp_bytes: the number of bytes in one response or `raw:true`. In the downsampled data, `raw:false` is summed over an hour for each endpoint.
    * gauge: the number of bytes in the bucket indicated by the bucket_id tag at the moment of time indicated by _time. If your bucket has an infinite retention policy, then the sum of your req_bytes from when you started writing data to the bucket would equal your gauge value.
    * req_bytes: the number of bytes in one request for `raw:true`. In the downsampled data, `raw:false` is summed over an hour for each endpoint.
    * event_type_limited_cardinality: the number of cardinality limit events.
    * event_type_limited_query: the number of query limit events.
    * event_type_limited_write: the number of write limit events.

The [usage.limits()](https://docs.influxdata.com/influxdb/cloud/reference/flux/stdlib/experimental/usage/limits/) function maps to the [org/{orgID}/limits endpoint](https://docs.influxdata.com/influxdb/v2.0/api/#operation/GetOrgsID) in the InfluxDB v2 API. The [usage.limits()](https://docs.influxdata.com/influxdb/cloud/reference/flux/stdlib/experimental/usage/limits/) returns a record, specifically a json object, of the usage limits associated with your InfluxDB Cloud Org. To return the record in either VS Code or the InfluxDB UI, I suggest using the [array.from](https://docs.influxdata.com/influxdb/cloud/reference/flux/stdlib/array/from/) function like so:


```
import "experimental/usage"
import "array"
limits = usage.limits()
array.from(rows: [{
    orgID: limits.orgID,
    wrte_rate: limits.rate.writeKBs,
    query_rate: limits.rate.readKBs,
    bucket: limits.bucket.maxBuckets,
    task: limits.maxTasks,
    dashboard: limits.dashboard.maxDashboards,
    check: limits.check.maxChecks,
    notificationRule: limits.notificationRule.maxNotifications,
}])
```


The Flux above is essentially converting the json output to Annotated CSV with the use of the [array.from](https://docs.influxdata.com/influxdb/cloud/reference/flux/stdlib/array/from/) function. Here is an example of the json object that [usage.limits()](https://docs.influxdata.com/influxdb/cloud/reference/flux/stdlib/experimental/usage/limits/) returns to help you understand where the nested json keys are coming from.


```
{
  orgID: "123",
  rate: {
    readKBs: 1000,
    concurrentReadRequests: 0,
    writeKBs: 17,
    concurrentWriteRequests: 0,
    cardinality: 10000
  },
  bucket: {
    maxBuckets: 2,
    maxRetentionDuration: 2592000000000000
  },
  task: {
    maxTasks: 5
  },
  dashboard: {
    maxDashboards: 5
  },
  check: {
    maxChecks: 2
  },
  notificationRule: {
    maxNotifications: 2,
    blockedNotificationRules: "comma, delimited, list"
  },
  notificationEndpoint: {
    blockedNotificationEndpoints: "comma, delimited, list"
  }
}
```



## Using the Cardinality Limit Alert Task

The Cardinality Limit Alert Task is responsible for determining when you have reached or exceeded your cardinality limits and sending an alert to a Slack endpoint. In order to make this task functional and send alerts to your Slack, you must edit the task and supply the webhook. You can either edit the task directly through the UI, or you can use the CLI and the Flux extension for Visual Studio code. For this one-line edit, I recommend using the UI.


![alt_text]({{site.url}}/assets/images/part-3/monitoring-and-managing-influxdb/13-cardinality-alert.png  "image_tooltip")
_Navigate to the Tasks page, click on the Cardinality Limit Alert task to edit, edit line 7 and provide your Slack webhook, then hit save to make the Cardinality Limit Alert task functional._

​​ 

[Next Section]({{site.url}}/docs/part-3/vs-code-integration){: .btn .btn-purple}