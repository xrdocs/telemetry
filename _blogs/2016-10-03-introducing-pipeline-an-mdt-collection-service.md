---
published: false
date: '2016-10-03 14:27 -0600'
title: 'Introducing Pipeline: An MDT Collection Service'
---
## Collection

With model-driven telemetry, routers can stream out large amounts of operational data in a highly efficient, easily consumable way.  But what do you do with the data once it's off the box?  You could, if you're code-inclined, whip up a 

This week, we opensourced Pipeline, a lightweight collection service that provides the first step in scaleable data collection.  

Pipeline is a lightweight, open-source, multi-function collection service (written in Go) that can input telemetry data from any XR release starting with 6.0.1 (in any encoding and over any transport) and output the telemetry data in different ways (to a text file, Kafka bus, Prometheus, etc).  Pipeline is _not_ a complete big data analytics stack.  Think of it as the first layer in a scalable, modular analytics architecture.  Depending on your use case, that architecure would also include separate components for big data storage, stream processing, analysis, alerting and visualization.  Big data platforms in open source include (among many) [PNDA](http://pnda.io/guide) and the [Prometheus](https://prometheus.io/) eco-system.  Pipeline's function is to processes the raw telemetry data from the network and transforms it into a format that can be leveraged by systems like these.
