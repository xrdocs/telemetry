---
published: false
date: '2017-04-10 14:19 -0600'
title: 'Using Pipeline: Integrating with InfluxDB'
author: Shelly Cadora
excerpt: Describes how to configure Pipeline to relay telemetry data to InfluxDB.
tags:
  - iosxr
---


## Using Pipeline 

In an [earlier blog](https://xrdocs.github.io/telemetry/tutorials/2016-10-03-pipeline-to-text-tutorial/), I discussed how to configure [Pipeline](http://blogs.cisco.com/sp/introducing-pipeline-a-model-driven-telemetry-collection-service) to write Model-Driven-Telemetry (MDT) data to a plain text file. In this tutorial, I'll describe the Pipeline configuration that enables you to write telemetry data into [InfluxDB](https://www.influxdata.com/), an open source platform for time-series data.


### Preparing the Router

This tutorial assumes that you've already configured your router for model-driven telemetry (MDT) with TCP dial-out using the instructions in [this tutorial](https://xrdocs.github.io/telemetry/tutorials/2016-07-21-configuring-model-driven-telemetry-mdt/). The IP address and port that you specify in the destination-group in the router config should match the IP address and port on which Pipeline is listening.


### Getting Influxdb

This tutorial assumes that you have a working instance of Influxdb with an IP address that is accessible from your Pipeline instance.  Influxdb is available from [github](https://github.com/influxdata/influxdb).  


### Getting Pipeline

Pipeline is available from [github](https://github.com/cisco/bigmuddy-network-telemetry-pipeline).   


### Pipeline.conf

The pipeline.conf file contains all the configuration necessary to get Pipeline running.  
The pipeline configuration is divided up into sections.  Each section is delineated by an arbitrary name enclosed in square brackets.  Each section defines either an input stage or an output stage.

### Configuring the Input Stage for TCP Dial-Out

For this tutorial, I'll use the default pipeline.conf input stage for MDT TCP Dial-Out described in the [TCP to Textfile tutorial](https://xrdocs.github.io/telemetry/tutorials/2016-07-21-configuring-model-driven-telemetry-mdt/).  If you take out all the comments, this reduces to 5 lines in pipeline.conf:

{% capture "output" %}
```
[testbed]
stage = xport_input
type = tcp
encap = st
listen = :5432

```  
{% endcapture %}

<div class="notice--warning">

{{ output | markdownify }}

</div>


This ```[testbed]``` section shown above will work "as is" for MDT with TCP dial-out.  If you want to change the port that Pipeline listens on to something other than "5432", you can edit this section of the pipeline.conf.  Otherwise, we're good to go for the input stage.

​

### Configuring the Output Stage for InfluxDB

To push the data to influxdb, we need "metrics" stage in Pipeline.  The default pipeline.conf file comes with an example metrics stage section called ```[mymetrics]```.  Taking out the comments, the important lines are as follows: 

{% capture "output" %}

```
[mymetrics]
stage = xport_output
type = metrics
file = metrics.json
dump = metricsdump.txt
output = influx
influx = http://10.152.176.74:8086
database = mdt_db
```  
{% endcapture %}
<div class="notice--warning">
{{ output | markdownify }}
</div>

This configuration instructs pipeline to transform the MDT data according to the ```[metrics.json]``` file and pushes it to an influxdb instance at 10.152.176.74 that has a database named ```[mdt_db]```.


### Running Pipeline

Running pipeline is trivial.  Just execute the binary in the bin directory.  Pipeline will use the pipeline.conf file by default.

​

{% capture "output" %}

​

```

scadora@darcy:~/bigmuddy-network-telemetry-pipeline$ bin/pipeline &

[1] 21975

scadora@darcy:~/bigmuddy-network-telemetry-pipeline$ Startup pipeline

Load config from [pipeline.conf], logging in [pipeline.log]

Wait for ^C to shutdown

​

scadora@darcy:~/bigmuddy-network-telemetry-pipeline$

​

```  

{% endcapture %}

​

<div class="notice--warning">

{{ output | markdownify }}

</div>

​

### Seeing the Data

Assuming your router is properly configured, the router should initiate the TCP session to Pipeline and stream the data specified in the sensor-group configuration.  To see the data as it comes in, use the Linux "tail" utility on the file that the ```[inspector]``` stage was configured to write to.

​

{% capture "output" %}

​

```

scadora@darcy:~/bigmuddy-network-telemetry-pipeline$ tail -f dump.txt

​

------- 2017-04-03 20:37:06.763244782 -0700 PDT -------

Summary: GPB(common) Message [172.30.8.53:15457(SunC)/Cisco-IOS-XR-infra-statsd-oper:infra-statistics/interfaces/interface/latest/generic-counters msg len: 5984]

{

    "Source": "172.30.8.53:15457",

    "Telemetry": {

        "node_id_str": "SunC",

        "subscription_id_str": "Sub1",

        "encoding_path": "Cisco-IOS-XR-infra-statsd-oper:infra-statistics/interfaces/interface/latest/generic-counters",

        "collection_id": 712163,

        "collection_start_time": 1491277026499,

        "msg_timestamp": 1491277026499,

        "collection_end_time": 1491277026507

    },

    "Rows": [

        {   

            "Timestamp": 1491277026506,

            "Keys": {

                "interface-name": "MgmtEth0/RP0/CPU0/0"

            },

            "Content": {

                "applique": 0,

                "availability-flag": 0,

                "broadcast-packets-received": 65679,

                "broadcast-packets-sent": 0,

                "bytes-received": 272894774,

                "bytes-sent": 20829696017,

                "carrier-transitions": 1,

                <output snipped for brevity>

​

```  

{% endcapture %}

​

<div class="notice--warning">

{{ output | markdownify }}

</div>

​

### Why Did We Do That Again?

To leverage the real power of telemetry, you need to get the data into an analytics stack like InfluxDB or Prometheus...or to multiple consumers via a pub/sub mechanism like Kafka.  Pipeline can do all that and I'll show you how in future tutorials.

​

But having the power to dump encoded telemetry data into a text file does come in handy, especially when you're setting up telemetry and Pipeline for the first time.  The ```tap``` output module gives you a quick and easy way to validate that the router is sending the data you think it should be sending.  Once that's settled, it's a simple matter of configuring a different output module to send the data some place really useful.

​

Give Pipeline a try and let us know what you think!

​

Prose

    Prose
    About
    Developers
    Language
    Logout

