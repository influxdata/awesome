---
layout: default
title: Flux API Invokable Scripts and Parameterized Queries
parent: Part 3
nav_order: 5
---

# Deletes
{: .no_toc }

## Table of contents
{: .no_toc .text-delta }

1. TOC
{:toc}

---

InfluxDB Cloud supports API Invokable Scripts in Flux. API Invokable Scripts with parameterized queries that are hosted and invoked by the InfluxDB Cloud server so that application developers can add custom functionality to their time series applications without having to introduce query resources into their client-side services and code. A parameterized query enables you to supply arguments which are then inserted into the Flux query for it to be executed. By supporting runtime parameters, it is possible to generalize these functions so that custom results may be returned for each user. 


### Parameterized Queries

For the most basic example, we’ll pass in a bucket name as an argument to our parameterized Flux query. The Flux script looks like this:


```js
from(bucket:params.mybucket) 
|> range(start: -7d) 
|> limit(n:2)","params":{"mybucket":"telegraf"}
```


The Flux engine will replace params.mybucket with the bucket name that we want to query. We specify the value of the mybucket  parameter at the end of the Flux query request payload with "params":{"mybucket":"telegraf"} to query the "telegraf" bucket. Since this query must be executed with the API, we convert the query into JSON to execute the following cURL request:


```
curl -X POST \
'https://us-west-2-1.aws.cloud2.influxdata.com/api/v2/query?orgID=<myOrgID>' \
  -H 'authorization: Token <myToken>' \
  -H 'content-type: application/json' \
  -d '{"query":"from(bucket:params.mybucket) |> range(start: -7d) |> limit(n:2)","params":{"mybucket":"telegraf"}}'
```


That’s all there is to it! Now you can start constructing parameterized queries to secure your IoT application and help prevent injection attacks. Parameterized queries also make updating a query to reflect a new bucket, filter or timestamp easy and encourage code reuse. 


#### Typing Parameterized Queries

Using parameterized Flux queries is pretty straightforward. Parameterized Flux queries support parameters of int, float, and string types. However, Flux itself supports more types, such as duration and others. Therefore, you must make sure to correctly type date parameters. For example if you want to make a parameter a timestamp, you must convert that value into a duration with the duration() function. The body of your request should look like this:


```
{"query":"from(bucket:\"telegraf\") |> range(start: duration(v : params.mystart)) |> limit(n:2)","params":{"mystart":"-7d"}}
```



### Flux API Invokable Scripts

You can create an API Invokable Script and an associated function resource with the InfluxDB v2 API. An associated function resource contains the following: name, id, description, orgID, script, language, url, createdAt, and updatedAt. These features help the developer identify their API Invokable Script and its associated metadata. The function id is used to manually invoke the function upon request.

To create a function with an associated function resource, use the api/v2/functions endpoint. In this example, our parameterized query returns the last two data points from the bucket of our choice.


```
curl -X 'POST' \
  'https://us-west-2-1.aws.cloud2.influxdata.com/api/v2/functions' \
  -H 'accept: application/json' \
  -H 'Content-Type: application/json' \
  -d '{
  "name": "myFirstAPIInvokableScripts",
  "description": "a API Invokable Script that gathers the last point from a bucket",
  "orgID": "<myOrgID>",
  "script": "from(bucket:params.mybucket) \
|> range(start: -7d) \
|> limit(n:2)",
  "language": "flux"
}'
```


The function id is returned in the body of the response when you create an API Invokable Script. Alternatively, you can also retrieve the function id and all the other associated function resources by listing the function. Then you can use list the function with:


```
curl -X 'GET' \
  'https://us-west-2-1.aws.cloud2.influxdata.com/api/v2/functions?orgID=<myOrgID>' \
  -H 'accept: application/json'
```



#### Invoking a Script with Parameterized Queries

Once you have the function id, you can manually invoke the API Invokable Script and supply the parameters. In this case, we’ll supply the bucket name to the mybucket parameter. Our request looks like this:


```
curl -X 'POST' \
  'https://us-west-2-1.aws.cloud2.influxdata.com/api/v2/functions/<functionID>/invoke' \
  -H 'accept: application/json' \
  -H 'Content-Type: application/json' \
  -d '{
  "params": {"mybucket":"telegraf"}
}'
