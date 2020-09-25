---
published: true
date: '2020-09-24 19:00 +0200'
title: Root Cause Packet Drop Through Model Driven Telemetry Deployment
author: Frederic Cuiller
position: hidden
excerpt: >-
  How Model Driven Telemetry was used to identify and root cause packet drop in
  a large service provider production environment.
---
{% include toc icon="table" title="Identify and Root Cause Packet Drop Through Model Driven Telemetry Deployment" %} 

## Introduction

As part of Cisco Customer Experience group, we are often involved in critical and very technical customer escalations. This blog post aims to describe how Model Driven Telemetry was used to identify and root cause packet drop in a large service provider production environment.

## Context

Earlier this year Cisco TAC was contacted by one of our customers to report transient packet loss, ultimately impacting their IPTV service during peak hour. Packet loss was narrowed down to ASR 9000 routers and dropped packets were identified as Early Fast Discard (EFD) at Network Processor (NP) level. This is a high-level representation view of a Cisco ASR9000 3rd generation linecard in a 8x100G configuration:

![asr9k-8x100G-high-level]({{site.baseurl}}/images/asr9k-8x100G-high-level.png){: .align-center}

If we zoom inside a Network Processor, we can see EFD can happen in very early stages in ingress:

![asr9k-8x100G-NP-efd]({{site.baseurl}}/images/asr9k-8x100G-NP-efd.png){: .align-center}

More information about ASR 9000 Early Fast Discard (EFD) can be found in B[RKSPG-2904 Cisco Live presentation](https://www.ciscolive.com/c/dam/r/ciscolive/emea/docs/2016/pdf/BRKSPG-2904.pdf).

<div class="highlighter-rouge">
<pre class="highlight">
<code>
RP/0/RP0/CPU0:ASR9000#sh controllers np fast-drop all location 0/10/CPU0
Fri Feb 14 18:00:06.419 FRANCE
		Node: 0/10/CPU0:
----------------------------------------------------------------
 snip
 -------------------------------------------------------------
All fast drop counters for NP 1:
 HundredGigE0/10/0/3:[Priority1]               0
 HundredGigE0/10/0/3:[Priority2]               0
 <mark>HundredGigE0/10/0/3:[Priority3]               247</mark>
 HundredGigE0/10/0/2:[Priority1]               0
 HundredGigE0/10/0/2:[Priority2]               0
 <mark>HundredGigE0/10/0/2:[Priority3]               23451515</mark>
 -------------------------------------------------------------
</code>
</pre>
</div>

Source of EFD was unknown and could be caused by several factors. An engineering Cisco IOS-XR SMU was developed to manually monitor and record average and peak RFD (Receive Frame Descriptor) utilization. As ratio between average and peak values was big (x12), we were able to demonstrate existence of bursts in this network.
