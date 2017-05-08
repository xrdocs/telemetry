---
published: false
date: '2017-05-08 11:03 -0600'
title: Pipeline with gRPC
author: Shelly Cadora
excerpt: Descibres how to use Model-Driven Telemetry with Pipeline and gRPC
tags:
  - iosxr
  - telemetry
  - gRPC
  - MDT
  - pipeline
---
# Using Pipeline with gRPC

In previous tutorials, I've shown how to use [Pipeline](http://blogs.cisco.com/sp/introducing-pipeline-a-model-driven-telemetry-collection-service) to dump Model Driven Telemetry (MDT) data into a [text file](https://xrdocs.github.io/telemetry/tutorials/2016-10-03-pipeline-to-text-tutorial/) and [InfluxDB](https://xrdocs.github.io/telemetry/tutorials/2017-04-10-using-pipeline-integrating-with-influxdb/).  In each case, I configured the router to transport MDT data to Pipeline using TCP.  In this tutorial, I'll cover a few additional steps that are required to use Pipeline with [gRPC](http://www.grpc.io/).

# gRPC Dialout
For gRPC dial-out, the big decision is whether to use TLS or not. This impacts the destination-group in the router config and the ingress stage of the Pipeline input stage.

Aside the destination-group config, I'll re-use the rest of the MDT router config from the [gRPC dial-out example](https://xrdocs.github.io/telemetry/tutorials/2016-07-21-configuring-model-driven-telemetry-mdt/#grpc-dial-out).  It should look something like this:

```
telemetry model-driven
 sensor-group SGroup2
  sensor-path Cisco-IOS-XR-nto-misc-oper:memory-summary/nodes/node/summary
 !
 subscription Sub2
  sensor-group-id SGroup2 sample-interval 30000
  destination-id DGroup2
``` 

## Without TLS


## With TLS

# gRPC Dialin
[dial-in](https://xrdocs.github.io/telemetry/tutorials/2016-07-21-configuring-model-driven-telemetry-mdt/#grpc-dial-in)

### Without TLS

### With TLS