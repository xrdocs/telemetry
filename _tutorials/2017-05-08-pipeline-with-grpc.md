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

In previous tutorials, I've shown how to use [Pipeline](http://blogs.cisco.com/sp/introducing-pipeline-a-model-driven-telemetry-collection-service) to dump Model Driven Telemetry (MDT) data into a [text file](https://xrdocs.github.io/telemetry/tutorials/2016-10-03-pipeline-to-text-tutorial/) and into [InfluxDB](https://xrdocs.github.io/telemetry/tutorials/2017-04-10-using-pipeline-integrating-with-influxdb/).  In each case, I configured the router to transport MDT data to Pipeline using TCP.  In this tutorial, I'll cover a few additional steps that are required to use Pipeline with [gRPC](http://www.grpc.io/).  I'll focus on only the changes needed in the router and Pipeline input stage configs here, so be sure to consult the other Pipeline tutorialis for important info about install, egress stage, etc.

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

## gRPC Dialout Without TLS
If you don't use TLS, your MDT data won't be encrypted and the server will have no way to authenticate the collector's identity before pushing data to it.  On the other hand, it's easy to configure. So if you're new to MDT and gRPC, this might be a good starting place.

### Router Config: no-tls

The gRPC config for the router is contained in the MDT destination-group.  Here is a destination-group for gRPC dialout without TLS:

```
telemetry model-driven
 destination-group DGroup2
  address family ipv4 172.30.8.4 port 57500
   encoding self-describing-gpb
   protocol grpc no-tls
```

### Pipeline.conf: tls = false

You can use the ```[gRPCDIalout]``` input stage in the default pipeline.conf.  Just uncomment the 6 lines shown below.

```
$ grep -A25 "gRPCDialout" pipeline.conf | grep -v -e '^#' -e '^$'
[gRPCDialout]
 stage = xport_input
 type = grpc
 encap = gpb
 listen = :57500
 tls = false
```

If you now run pipeline with the debug option, you should see these lines when Pipeline starts and the router (at 172.30.8.53) connects:
```
$ bin/pipeline -config pipeline.conf -log= -debug | grep gRPC
INFO[2017-05-08 11:25:50.046573] gRPC starting block                           encap=gpb name=grpcdialout server=:57500 tag=pipeline type="pipeline is SERVER"
INFO[2017-05-08 11:25:50.046902] gRPC: Start accepting dialout sessions        encap=gpb name=grpcdialout server=:57500 tag=pipeline type="pipeline is SERVER"
INFO[2017-05-08 11:26:03.572534] gRPC: Receiving dialout stream                encap=gpb name=grpcdialout peer="172.30.8.53:61857" server=:57500 tag=pipeline type="pipeline is SERVER"
```

## With TLS

# gRPC Dialin
[dial-in](https://xrdocs.github.io/telemetry/tutorials/2016-07-21-configuring-model-driven-telemetry-mdt/#grpc-dial-in)

### Without TLS

### With TLS