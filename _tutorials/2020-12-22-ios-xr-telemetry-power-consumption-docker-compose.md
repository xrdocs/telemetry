---
published: false
date: '2020-12-22 15:23 +0100'
title: 'IOS-XR power consumption monitoring: an ephemeral telemetry stack use case'
author: fcuiller
excerpt: >-
  This post will describe how a docker-based ephemeral telemetry stack has been
  built to monitor Cisco IOS-XR device power consumption.
position: hidden
tags:
  - iosxr
---
### Introduction

Power efficiency is a big topic as it can literally help customers save millions. There has been several announcements recently about Cisco’s Silicon One and how it compares to previous generation of chip on this specific point. 
This post will describe how a docker-based ephemeral telemetry stack has been built to monitor Cisco IOS-XR device power consumption.

### Context

As part of an upcoming CPOC (Customer Proof Of Concept) engagement, one of our customers wants to benchmark routers power consumption. Most of time, TCO (Total Cost of Ownership) calculations are made based on estimations. Cisco do provide such numbers through Cisco Power Calculator tool. However, they may vary based on several factors like ambient temperature, traffic load, optics, etc. For a fully loaded ASR 9906 with latest generation of RSP and linecards, there can be a 50% difference between the typical scenario (27C ambient temperature with 50% linerate IMIX traffic) and the worst-case scenario (50/55C)!
For this multi-vendor assessment, apples to apples comparison is required. A power meter or a smart PDU could have been installed, but they require additional hardware to be purchased and installed on the premises. Instead we decided to use Model Driven Telemetry to monitor in real time our device power consumption: it’s already available, free and we only need extra configuration. The last piece is a collector.

### Docker-compose telemetry stack

To receive and consume power data from our routers, I decided to build an ephemeral telemetry stack which can be brought on-demand on a server or a laptop. As CPOC will be delivered over a one-week period, there is no need to build a full-blown and permanent solution.
I reused the same popular components: Telegraf, InfluxDB and Grafana. This time I decided to add Chronograf to quickly see the data stored in InfluxDB. I also leveraged Docker to build this stack and especially Docker Compose. As described in Docker’s documentation:
Compose is a tool for defining and running multi-container Docker applications. With Compose, you use a YAML file to configure your application’s services. Then, with a single command, you create and start all the services from your configuration.”

There was some work done by Jeff Kehres available on Github. I reused it to add Telegraf and I removed the docker volumes: as this stack is ephemeral, persistent storage is not required. Full code and documentation can be found here. Here is the raw docker-compose.yaml file:




