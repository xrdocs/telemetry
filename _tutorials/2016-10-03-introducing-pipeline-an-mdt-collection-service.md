---
published: false
date: '2016-10-03 12:13 -0600'
title: 'Using Pipeline: TCP to textfile'
author: Shelly Cadora
excerpt: >-
  Describes how to use the open source pipeline utility to collect and transform
  telemetry data for consumption by various applications
---
## 
In an earlier blog (link) , I introduced Pipeline, a multi-function collection service written in Go.  In this tutorial, I'll cover how to set up Pipeline for the simplest of tasks:  ingesting telemetry data over TCP and writing it to a file as a JSON object.

### Preparing the Router
This tutorial assumes that you've already configured your router for TCP dial-out using the instructions in [this tutorial](https://xrdocs.github.io/telemetry/tutorials/2016-07-21-configuring-model-driven-telemetry-mdt/). The IP address and port that you specify in the destination-group in the router config should match the IP address and port on which Pipeline is listening.

### Getting Pipeline
Pipeline is available from github (link).  

### Configuring the Input Stage for TCP Dial-Out
The pipeline.conf file contains all the configuration necessary to get Pipeline running.  The default pipeline.conf can be used with minor modifications.

### Configuring the Output Stage for Text File

### Running Pipeline


