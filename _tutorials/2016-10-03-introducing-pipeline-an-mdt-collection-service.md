---
published: false
date: '2016-10-03 12:13 -0600'
title: 'Using Pipeline: TCP to textfile'
author: Shelly Cadora
excerpt: >-
  Describes how to use the open source pipeline utility to collect and transform
  telemetry data for consumption by various applications
---
## Using Pipeline 
In an earlier blog (link) , I introduced Pipeline, a multi-function collection service written in Go.  In this tutorial, I'll cover how to set up Pipeline for the simplest of tasks:  ingesting telemetry data over TCP and writing it to a file as a JSON object.

### Preparing the Router
This tutorial assumes that you've already configured your router for TCP dial-out using the instructions in [this tutorial](https://xrdocs.github.io/telemetry/tutorials/2016-07-21-configuring-model-driven-telemetry-mdt/). The IP address and port that you specify in the destination-group in the router config should match the IP address and port on which Pipeline is listening.

### Getting Pipeline
Pipeline is available from github (link).  

### Pipeline.conf
The pipeline.conf file contains all the configuration necessary to get Pipeline running.  In many cases, the default pipeline.conf can be used with little or no modification. 

The pipeline configuration is divided up into sections.  Each section is delineated by an arbitrary name enclosed in square brackets.  Each section defines either an input stage ("stage = xport_input") or an output stage ("stage = export_output").  Other parameters in the section tell Pipeline what to listen for (in the case of an input stage) or how to output the data (for an output stage).

The easiest way to understand this is to look at a simple example.

### Configuring the Input Stage for TCP Dial-Out
Let's take a look at the TCP dial-out section in the default pipeline.conf.

{% capture "output" %}

```
$cd pipeline
pipeline$ grep -A20 "Example of a TCP dialout" pipeline.conf
# Example of a TCP dialout (router connects to pipeline over TCP) 
#
[testbed]
stage = xport_input
#
# Module type, the type dictates the rest of the options in the section.
# TCP can only be used as xport_iinput currently.
#
type = tcp
#
# Supported encapsulation is 'st' for streaming telemetry header. 
#
encap = st
#
# TCP option dictating which binding to listen on. Can be a host name
# or address and port, or just port.
#
listen = :5432
#
pipeline$
```  
{% endcapture %}

<div class="notice--warning">
{{ output | markdownify }}
</div>

This [testbed] section shown above will work "as is" for MDT with TCP dial-out.  If you want to change the port that Pipeline listens on to something other than "5432", you can edit this section of the pipeline.conf.  Otherwise, we're good to go for the input stage.

### Configuring the Output Stage for Text File
To dump the received data to a file, we need a "tap" stage in Pipeline.  The default pipeline.conf file comes with a tap stage section called [inspector] as you can see below.

{% capture "output" %}

```
$cd pipeline
pipeline$ grep -A20 "Example of a tap stage" pipeline.conf
## Example of a tap stage; dump content to file, or at least count messages
#
[inspector]
stage = xport_output
#
# Module type: tap is only supported in xport_output stage currently.
#
type = tap
#
# File to dump decoded messages
#
file = /data/dump.txt
```  
{% endcapture %}

<div class="notice--warning">
{{ output | markdownify }}
</div>

This [inspector] section shown above will work "as is" for dumping data to a file.  If you want to change the file that Pipeline writes to (default is /data/dump.txt) or write with a different encoding (default is JSON), you can edit this section of the pipeline.conf.  Otherwise, we're good to go for the output stage as well.

### Running Pipeline
Running pipeline is trivial.  Just execute the binary in the bin directory.  Pipeline will use the pipeline.conf file by default.

{% capture "output" %}

```
admin@collector:pipeline$ bin/pipeline -config &
[1] 15411
Startup pipeline
Load config from [pipeline.conf], logging in [pipeline.log]
Wait for ^C to shutdown
admin@collector:pipeline$

```  
{% endcapture %}

<div class="notice--warning">
{{ output | markdownify }}
</div>

### Seeing the Data
Assuming your router is properly configured, the router should initiate the TCP session to Pipeline and stream the data specified in the sensor-group configuration.  To see the data as it comes in, use the Linux "tail" utility on the file that the [inspector] stage was configured to write to.

{% capture "output" %}

```
admin@collector:pipeline$ tail -f dump.txt

------- 2016-07-31 14:43:21.523938034 -0700 PDT -------
Summary: GPB K/V Message [172.16.128.2:19630/172.16.128.2:19630/Cisco-IOS-XR-infra-statsd-oper:infra-statistics/interfaces/interface/latest/generic-counters]
{
    "collection_id": "34",
    "base_path": "Cisco-IOS-XR-infra-statsd-oper:infra-statistics/interfaces/interface/latest/generic-counters",
    "subscription_identifier": "",
    "model_version": "",
    "collection_start_time": "1470001399128",
    "msg_timestamp": "1470001399128",
    "fields": [
        {

```  
{% endcapture %}

<div class="notice--warning">
{{ output | markdownify }}
</div>
