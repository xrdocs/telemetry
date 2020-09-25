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

More information about ASR 9000 Early Fast Discard (EFD) can be found in [BRKSPG-2904 Cisco Live presentation](https://www.ciscolive.com/c/dam/r/ciscolive/emea/docs/2016/pdf/BRKSPG-2904.pdf).

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

## Leveraging Model Driven Telemetry in production

While a manual and offline analysis through enhanced np_perf SMU was handy to prove bursts presence, this was not enough to identify when exactly they appeared, how big they were and how often they would repeat. Those events were very short and intense and could not be caught with traditional monitoring techniques which often provide 5min resolution. Moreover, we were missing critical information not available through regular SNMP MIB which would help us identify the source of those bursts.
For all those reasons and after several weeks without progress, we decided to leverage Model Driven Telemetry.

## Bringing up a cloud-hosted telemetry stack

To collect and digest telemetry data from routers at scale, infrastructure is required. As we had to move fast, it was decided to implement a telemetry collection stack in a secured cloud environment. We decided to use the popular Telegraf – InfluxDB – Grafana stack. This topic has already been covered and I will only focus on key implementation details:
- IPv6 only: cloud-hosted telemetry stack is reachable via IPv6 only. 
- Export is done through TCP. It was not possible to use gRPC given IOS-XR software release used in this environment.
- /etc/telegraf/telegraf.conf file is updated to add tags and avoid some values being overwritten
<div class="highlighter-rouge">
<pre class="highlight">
<code>
[[inputs.cisco_telemetry_mdt]]

# # Cisco model-driven telemetry (MDT) input plugin for IOS XR, IOS XE and NX-OS platforms
[[inputs.cisco_telemetry_mdt]]
#  ## Telemetry transport can be "tcp" or "grpc".  TLS is only supported when
#  ## using the grpc transport.
transport = "tcp"
#  ## Address and port to host telemetry listener
service_address = ":57500"

#  ## Define (for certain nested telemetry measurements with embedded tags) which fields are tags
embedded_tags = ["Cisco-IOS-XR-asr9k-np-oper:hardware-module-np/nodes/node/nps/np/counters/np_counters/counter-index", "Cisco-IOS-XR-qos-ma-oper:qos/interface-table/interface/input/service-policy-names/service-policy-instance/statistics/class-stats/class-name", "Cisco-IOS-XR-qos-ma-oper:qos/interface-table/interface/output/service-policy-names/service-policy-instance/statistics/class-stats/class-name", "Cisco-IOS-XR-asr9k-np-oper:hardware-module-np/nodes/node/nps/np/fast-drop/np_fast_drop/interface-name", "Cisco-IOS-XR-asr9k-np-oper:hardware-module-np/nodes/node/nps/np/counters/np-counter/counter-name", "Cisco-IOS-XR-asr9k-fsi-oper:fabric-stats/nodes/node/statses/stats/stats-table/fsi_stats/counter-name", "Cisco-IOS-XR-clns-isis-oper:isis/instances/instance/statistics-global/per_area_data/per_topology_data/id/af-name"]
name_suffix = "_tcp"
</code>
</pre>
</div>

- Collector has been hardened through nftable filters implementation and fail2ban installation.
- Grafana interface has been secured with Let’s Encrypt certificate.
- InfluxDB data retention policy has been initially configured to 3 days and later on extended to 5 days. We were not interested in getting full history as routers were monitored in real time and a review was performed on a daily basis. 

## Router configuration

Model Driven Telemetry configuration is done in 3 steps.   

First, we need to declare what we want to export through sensor-group. We were interested in collecting low level router KPI (NP counters, fabric counters), traffic KPI (QoS counters, interface statistics, etc.) and also some control plan statistics (ISIS, OSPF, PIM, RIB). Those counters will be used to make data correlation.

<div class="highlighter-rouge">
<pre class="highlight">
<code>
telemetry model-driven
sensor-group NP-COUNTERS
  sensor-path Cisco-IOS-XR-asr9k-np-oper:hardware-module-np/nodes/node/nps/np/fast-drop
  sensor-path Cisco-IOS-XR-asr9k-np-oper:hardware-module-np/nodes/node/nps/np/load-utilization
  sensor-path Cisco-IOS-XR-asr9k-np-oper:hardware-module-np/nodes/node/nps/np/counters/np-counter
 !
 sensor-group PIM-COUNTERS
  sensor-path Cisco-IOS-XR-ipv4-pim-oper:pim/active/default-context/traffic-counters
 !
 sensor-group QOS-COUNTERS
  sensor-path Cisco-IOS-XR-qos-ma-oper:qos/interface-table/interface/input/service-policy-names/service-policy-instance/statistics
  sensor-path Cisco-IOS-XR-qos-ma-oper:qos/interface-table/interface/output/service-policy-names/service-policy-instance/statistics
 !
 sensor-group RIB-COUNTERS
  sensor-path Cisco-IOS-XR-ip-rib-ipv4-oper:rib/rib-stats
 !
 sensor-group ISIS-COUNTERS
  sensor-path Cisco-IOS-XR-clns-isis-oper:isis/instances/instance/statistics-global
 !
 sensor-group OSPF-COUNTERS
  sensor-path Cisco-IOS-XR-ipv4-ospf-oper:ospf/processes/process/default-vrf/process-information/process-areas/process-area
 !
 sensor-group FABRIC-COUNTERS
  sensor-path Cisco-IOS-XR-asr9k-fsi-oper:fabric-stats/nodes/node
  sensor-path Cisco-IOS-XR-asr9k-xbar-oper:cross-bar-stats/nodes/node/cross-bar-table
 !
 sensor-group ETHERNET-COUNTERS
  sensor-path Cisco-IOS-XR-drivers-media-eth-oper:ethernet-interface/statistics/statistic
</code>
</pre>
</div>

Then we need to define how often counters are collected. As we were troubleshooting traffic bursts, data had to be streamed at very high frequency. We decided to use 10s resolution for control-plan and interface statistics, 30s for NP and fabric counters as number of counters was huge.

<div class="highlighter-rouge">
<pre class="highlight">
<code>
telemetry model-driven
subscription SUB_MDT
  sensor-group-id NP-COUNTERS sample-interval 30000
  sensor-group-id PIM-COUNTERS sample-interval 10000
  sensor-group-id QOS-COUNTERS sample-interval 30000
  sensor-group-id RIB-COUNTERS sample-interval 10000
  sensor-group-id ISIS-COUNTERS sample-interval 10000
  sensor-group-id OSPF-COUNTERS sample-interval 10000
  sensor-group-id FABRIC-COUNTERS sample-interval 30000
  sensor-group-id ETHERNET-COUNTERS sample-interval 10000
</code>
</pre>
</div>

Last, we need to describe how counters are streamed. TCP was used instead of gRPC as monitored device did not support this last one. Encoding is GPB-KV (self-describing-gpb) to accommodate Telegraf Cisco Model-Driven Telemetry (MDT) input plugin.

<div class="highlighter-rouge">
<pre class="highlight">
<code>
telemetry model-driven
 destination-group CISCO-MDT
  address-family ipv6 2001:db8::1337 port 57500
   encoding self-describing-gpb
   protocol tcp
subscription SUB_MDT
  destination-id CISCO-MDT
  source-interface Loopback6
</code>
</pre>
</div>

Sensors must be in resolved state:

<div class="highlighter-rouge">
<pre class="highlight">
<code>
RP/0/RP0/CPU0:ASR9000#sh telemetry model-driven subscription SUB_MDT
Mon Sep 21 17:00:22.930 CEST
Subscription:  SUB_MDT
-------------
  State:       ACTIVE
  Source Interface:       Loopback6(Up 0x60000000)
  Sensor groups:
  Id: NP-COUNTERS
    Sample Interval:      30000 ms
    Sensor Path:          Cisco-IOS-XR-asr9k-np-oper:hardware-module-np/nodes/node/nps/np/fast-drop
    <mark>Sensor Path State:    Resolved</mark>
    Sensor Path:          Cisco-IOS-XR-asr9k-np-oper:hardware-module-np/nodes/node/nps/np/load-utilization
    Sensor Path State:    Resolved
    Sensor Path:          Cisco-IOS-XR-asr9k-np-oper:hardware-module-np/nodes/node/nps/np/counters/np-counter
    Sensor Path State:    Resolved
...
</code>
</pre>
</div>

Routers will establish TCP session with the collector and start streaming telemetry:
<div class="highlighter-rouge">
<pre class="highlight">
<code>
RP/0/RP0/CPU0:ASR9000#sh telemetry model-driven destination CISCO-MDT
Mon Sep 21 17:01:29.610 CEST
  Destination Group:  CISCO-MDT
  -----------------
    Destination IP:       2001:db8::1337
    Destination Port:     57500
    Subscription:         SUB_MDT
    <mark>State:                Active</mark>
    Encoding:             self-describing-gpb
    Transport:            tcp
    No TLS
    Total bytes sent:     183646872388
    Total packets sent:   2767164
    Last Sent time:       2020-09-21 17:01:27.536123923 +0200

    Collection Groups:
    ------------------
      Id: 78
      Sample Interval:      10000 ms
    Encoding:             self-describing-gpb
      Num of collection:    285252
      Collection time:      Min:    70 ms Max:  9046 ms
      Total time:           Min:  1683 ms Max: 1084745 ms Avg:  2087 ms
      Total Deferred:       0
      Total Send Errors:    0
      Total Send Drops:     0
      Total Other Errors:   0
    No data Instances:    0
      Last Collection Start:2020-09-21 17:01:25.533967923 +0200
      Last Collection End:  2020-09-21 17:01:27.536164923 +0200
      Sensor Path:          Cisco-IOS-XR-drivers-media-eth-oper:ethernet-interface/statistics/statistic
</code>
</pre>
</div>

## First results

Model Driven Telemetry rapidly helped us gaining visibility into EFD, especially timestamps and volume:

![first-EFD]({{site.baseurl}}/images/first-EFD.png){: .align-center}

Early in our analysis we were also able to detect strange traffic pattern, like sudden traffic increase in both directions for all packet size:

![packet-size]({{site.baseurl}}/images/packet-size.png){: .align-center}

While NP load model might not be available in older software versions, it was possible to get an idea adding what was transmitted to the 100G wire and what was parsed from the ethernet port through specific NP counters monitoring:

![NP-sum]({{site.baseurl}}/images/NP-sum.png){: .align-center}

When NP load sensor was supported, we could have a real-time view of all ASR9000 NP load present in all linecards belonging to a chassis and see if capacity was exceeded:

![NP-load]({{site.baseurl}}/images/NP-load.png){: .align-center}

Last, we leveraged QoS statistics to see if bursts were caused by a particular class of traffic:

![QoS-stats]({{site.baseurl}}/images/QoS-stats.png){: .align-center}







