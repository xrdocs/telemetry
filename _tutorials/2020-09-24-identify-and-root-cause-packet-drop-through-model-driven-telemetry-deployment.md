---
published: false
date: '2020-09-24 19:00 +0200'
title: Identify and Root Cause Packet Drop Through Model Driven Telemetry Deployment
author: Fred Cuiller
excerpt: Network Consulting Engineer
position: hidden
---
{% include toc icon="table" title="Identify and Root Cause Packet Drop Through Model Driven Telemetry Deployment" %} 

## Introduction

As part of Cisco Customer Experience group, we are often involved in critical and very technical customer escalations. This blog post aims to describe how Model Driven Telemetry was used to identify and root cause packet drop in a large service provider production environment.

## Context

Earlier this year Cisco TAC was contacted by one of our customers to report transient packet loss, ultimately impacting their IPTV service during peak hour. Packet loss was narrowed down to ASR 9000 routers and dropped packets were identified as Early Fast Discard (EFD) at Network Processor (NP) level. This is a high-level representation view of a Cisco ASR9000 3rd generation linecard in a 8x100G configuration:

![asr9k-8x100G-high-level]({{site.baseurl}}/images/asr9k-8x100G-high-level.png){: .align-center}

If we zoom inside a Network Processor, we can see EFD can happen in very early stages in ingress:




