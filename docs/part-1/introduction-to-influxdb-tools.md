---
layout: default
title: Introduction to InfluxDB Tools
description: "Part 1: Introduction to InfluxDB Tools"
parent: Part 1
nav_order: 2
---

# Introduction to InfluxDB Tools
{: .no_toc }

## Table of contents
{: .no_toc .text-delta }

1. TOC
{:toc}

---
To honor our commitment to developer happiness, InfluxData offers a wide range of tools to make working with time series data easy for all types of developers. 


## Flux and the Task Engine 

[Flux](https://docs.influxdata.com/influxdb/cloud/query-data/get-started/) is a functional query and scripting language. Flux enables you to:

* Transform and analyze data. 
* Write Tasks

Flux has a Javascript inspired syntax that makes it easily composable and readable. [Pipe-forward operators](https://docs.influxdata.com/flux/v0.x/get-started/syntax-basics/) separate function calls and make data transformations flow smoothly. Flux is Open Source, testable, shareable, and contributable. 

The [Task](https://docs.influxdata.com/influxdb/cloud/process-data/manage-tasks/) Engine executes Flux scripts on a schedule. It allows you to: 

* Process data to make visualizations screaming fast
* Get alerted if your data stops writing or breaches certain thresholds
* Periodically call an external service with data from InfluxDB

The Task engine can tackle all the above points with no additional code or operations on your part.


## InfluxDB User Interface

The InfluxDB UI provides a complete interface for working with time series data and InfluxDB. The InfluxDB UI enables you to:

* Build queries to visualize your time series data. You can select from a wide variety of [visualization types](https://docs.influxdata.com/influxdb/cloud/visualize-data/visualization-types/). The InfluxDB UI also supports geotemporal map visualizations. 
* Edit Flux code in the Flux Script Editor.
* Build [dashboards](https://docs.influxdata.com/influxdb/cloud/visualize-data/dashboards/) and [notebooks](https://docs.influxdata.com/influxdb/cloud/notebooks/).
* [Manage multiple users](https://docs.influxdata.com/influxdb/cloud/account-management/multi-user/) in your Organization.
* Build and [manage tasks](https://docs.influxdata.com/influxdb/cloud/process-data/manage-tasks/). Tasks are Flux scripts that run on a schedule. 
* Build checks and notifications. Checks and notifications are specialized types of tasks which enable alert creation. 


## Telegraf 

Have more stringent requirements for your writes? Need batching, retries, and other features? Don’t write this code yourself, just use [Telegraf](https://www.influxdata.com/time-series-platform/telegraf/). Telegraf is InfluxData’s plugin driven collection agent for metric and events.  With over [200 input plugins](https://github.com/influxdata/telegraf/tree/master/plugins/inputs), Telegraf  probably has an input plugin that fits your needs. But even if there isn’t a plugin for your exact use case, you can use telegraf to easily reformat any data type into your preferred output format, be it line protocol, json, csv, etc…. 

Telegraf isn’t just a tool for writing data to a destination data store. The Telegraf processor plugins and aggregator plugins enable you to do so much more with your data than sophisticated collection and writes. For example, you can easily add data transformations with Starlark. You can even use the execd processor plugin which makes Telegraf extensible in any language. It’s no wonder that Telegraf has close to 11,000 stars on github.  


## Awesome CLI

The [InfluxDB CLI](https://docs.influxdata.com/influxdb/cloud/reference/cli/influx/) allows users to interact with InfluxDB with ease. Features like [configuration profiles](https://docs.influxdata.com/influxdb/cloud/sign-up/#step-5-set-up-a-configuration-profile) and [environment variable](https://docs.influxdata.com/influxdb/v2.0/reference/cli/influx/#mapped-environment-variables) support make scripting and controlling InfluxDB a breeze.


## Rest API

All of InfluxDB is wrapped in a [REST API](https://docs.influxdata.com/influxdb/v2.0/api/). This API is highly compatible between OSS and Cloud. The API allows you to write data and query data, of course, but also has everything you need to manage the database, including creating resources like authentication tokens, buckets, users, etc…


## Client Libraries	

You aren’t limited to making REST calls on your own. If you prefer, InfluxDB has [client libraries](https://docs.influxdata.com/influxdb/cloud/api-guide/client-libraries/) written in 13 languages. These Client Libraries enable you to drop InfluxDB functionality into an existing code base with ease or create a new code base.


## VSCode Plugin

If you are a Visual Studio (VS) Code user, then it’s easy to add InfluxDB to your workflow using [Flux VS Code Extension](https://docs.influxdata.com/influxdb/cloud/tools/flux-vscode/).

![vs code extention]({{site.url}}/assets/images/part-1/introduction-to-influxdb-tools/1-vs-code.png)

Using the Flux VS Code Extension to write Flux and run the query. 


## Stacks and Templates

Want to back up or move your whole InfluxDB configuration? Add it to github so you can restore from a backup? Even use a gitops workflow to integrate with your existing CD (continuous deployment) process? InfluxDB supports all of this with Stacks and Templates. 

A Template is a prepackaged InfluxDB configuration that contains multiple InfluxDB resources. Templates include everything from Telegraf configuration, to Dashboards, to Alerts, and more. Use Templates to:



* Get up and running with InfluxDB for a variety of common use cases with a [Community Template](https://github.com/influxdata/community-templates),a community-contributed Templates. 
* Quickly get setup with a new InfluxDB instance
* Backup your Dashboard, Alert, and Task configurations. 

Applying an existing Community Template is as easy as copy and pasting a URL for the Template yaml file into the UI: 

![applying community template]({{site.url}}/assets/images/part-1/introduction-to-influxdb-tools/2-community-template.png)


A [Stack](https://docs.influxdata.com/influxdb/cloud/influxdb-templates/stacks/) is a stateful InfluxDB template that lets you add, update, and remove template resources on your InfluxDB instance. 

[Next Section]({{site.baseurl}}/docs/part-1/setting-up-influxdb){: .btn .btn-purple}

## Further Reading
1. [Flux Specification](https://github.com/influxdata/flux/blob/master/docs/SPEC.md)
2. [Get started with Flux](https://docs.influxdata.com/flux/v0.x/get-started/)
3. [Flux Documentation](https://docs.influxdata.com/flux/v0.x/)
4. [Use Telegraf to write data](https://docs.influxdata.com/influxdb/v2.1/write-data/no-code/use-telegraf/)
5. [TL;DR InfluxDB Tech Tips; Creating Tokens with the InfluxDB API](https://www.influxdata.com/blog/tldr-influxdb-tech-tips-creating-tokens-influxdb-api/)
6. [TL;DR InfluxDB Tech Tips; Creating Buckets with the InfluxDB API](https://www.influxdata.com/blog/tldr-influxdb-tech-tips-creating-buckets-influxdb-api/)

