---
published: false
date: '2016-10-03 12:13 -0600'
title: 'Introducing Pipeline: An MDT Collection Service'
author: Shelly Cadora
excerpt: >-
  Describes how to use the open source pipeline utility to collect and transform
  telemetry data for consumption by various applications
---
## Pipeline

In previous tutorials, I've written about several different ways to configure XR to stream model-driven telemetry. In this tutorial, I will focus on collecting that data with Pipeline.  

Pipeline is a lightweight, open-source, multi-function collection service (written in Go) that can input telemetry data from any XR release starting with 6.0.1 (in any encoding and over any transport) and output the telemetry data in different ways (to a text file, Kafka bus, Prometheus, etc).  Pipeline is _not_ a complete big data analytics stack.  Think of it as the first layer in a scalable, modular architecture.  Depending on your use case, that architecure would also include separate components for big data storage, stream processing, and data analysis and visualization.  Pipeline processes the raw telemetry data from the network and transforms it into a format that can be leveraged by the data platform such as [PNDA](http://pnda.io/guide) or the [Prometheus](https://prometheus.io/) eco-system.

