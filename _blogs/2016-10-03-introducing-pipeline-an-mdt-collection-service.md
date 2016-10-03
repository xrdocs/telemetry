---
published: false
date: '2016-10-03 14:27 -0600'
title: 'Introducing Pipeline: An MDT Collection Service'
---
## The Other Half of The Story
With model-driven telemetry (MDT), routers can stream out large amounts of operational data in a highly efficient, easily consumable way.  But getting data off the box is only half the story.  You have to have something on the other end to collect and transform the raw data in preparation for storage and analysis.  MDT uses standard transports, RPCs and encodings, so theoretically it wouldn't be too hard to whip up your own collector using standard libraries and packages.  Luckily, you don't have to start from scratch.  This week, we open-sourced Pipeline, a lightweight collection service that provides the first step in scalable data collection.  

## Input, Transform, Output
Pipeline is a flexible, multi-function collection service that is written in Go.  It can ingest telemetry data from any XR release starting from 6.0.0.  Pipeline's input stages support raw UDP and TCP, as well as gRPC dial-in and dial-out capability. For encoding, Pipeline can consume JSON, compact GPB and self-describing GPB.  On the output side, Pipeline can write the telemetry data to a text file as a JSON object, push the data to a Kafka bus and/or format it for consumption by open source stacks.  Pipeline can easily be extended to include other output stages and we encourage contributions from anyone who wants to get involved.

## What It's Not
It's important to understand that Pipeline is _not_ a complete big data analytics stack.  Think of it as the first layer in a scalable, modular analytics architecture.  Depending on your use case, that architecure would also include separate components for big data storage, stream processing, analysis, alerting and visualization.  Big data platforms in open source include (among many) [PNDA](http://pnda.io/guide) and the [Prometheus](https://prometheus.io/) eco-system.  Pipeline's function is to process the raw telemetry data from the network and transform it into a format that can be leveraged by systems like these.  
