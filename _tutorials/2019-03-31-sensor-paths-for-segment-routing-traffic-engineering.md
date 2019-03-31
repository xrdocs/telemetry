---
published: false
date: '2019-03-31 16:39 -0600'
title: Sensor Paths for Segment Routing Traffic Engineering
author: Shelly Cadora
excerpt: Describes sensor-paths for monitoring SR-TE
---
## SR-TE, Meet MDT

Recently, when I was building out a proof of concept in my lab for some new Segment Routing Traffic Engineering (SR-TE) use cases, I found myself wanting a quick way to check if all my policies were coming up and routing traffic.  Model-Driven Telemetry (MDT) can get a lot of data off the routers very quickly.  The only tricky part is identifying the YANG model and container that has the best data for a given use case.  So I wanted to share a couple of the ones that were useful for me for SR-TE.

## SR-TE Policy Status

One high-level Key Performance Indicator (KPI) for SR-TE is the operational state of the SR-TE policies on the headend.  In CLI, a quick way to verify that is with this command:

```
RP/0/RP0/CPU0:iosxrv-1#show segment-routing traffic-eng policy summary
Sun Mar 31 22:55:35.164 UTC

XTC policy database summary:
----------------------------

IPv4 source address: 1.1.1.1
Total configured policies: 6
  Operational: up 6 down 0

RP/0/RP0/CPU0:iosxrv-1#
```

In the output of this command, you can see that all six of the configured policies are up.  This same data can be found in the XR Native YANG model ```Cisco-IOS-XR-infra-xtc-agent-oper.yang```.  That model can have a lot of data, so to narrow it down to just the policy summary data, specify your MDT sensor-path like this:

```sensor-path Cisco-IOS-XR-infra-xtc-agent-oper:xtc/policy-summary```

(For the complete configuration of MDT, see [Configuring Model-Driven Telemetry](https://xrdocs.io/telemetry/tutorials/2016-07-21-configuring-model-driven-telemetry-mdt/) ).

If you use [pipeline](https://xrdocs.io/telemetry/tutorials/2018-03-01-everything-you-need-to-know-about-pipeline/) to collect that data, you can easily direct the data to InfluxDB and display it with Grafana. Just add this entry to your [metrics.json](https://xrdocs.io/telemetry/tutorials/2018-03-01-everything-you-need-to-know-about-pipeline/#pipeline-metricsjson) file in pipeline:

```
        {
                "basepath" : "Cisco-IOS-XR-infra-xtc-agent-oper:xtc/policy-summary",
                "spec" : {
                        "fields" : [
                                {"name" : "configured-total-policy-count"},
                                {"name" : "configured-down-policy-count"},
                                {"name" : "configured-up-policy-count"}
                        ]
                }
        }
```

The resulting graph in Grafana might look something like this as you configure and bring up SR-TE policies:

![SR-TE-Policy-Summary.jpg]({{site.baseurl}}/images/SR-TE-Policy-Summary.jpg)


## SR-TE Topology Summary

Another high-level KPI can be found in the SR-TE topology database statistics.  In CLI, a quick way to verify that is with this command:

```
RP/0/RP0/CPU0:iosxrv-1#show segment-routing traffic-eng ipv4 topology summary
Sun Mar 31 23:24:38.880 UTC

XTC Agent's topology database summary:
--------------------------------

Topology nodes:                8
Prefixes:                      6
  Prefix SIDs:                11
Links:                        20
  Adjacency SIDs:             40

RP/0/RP0/CPU0:iosxrv-1#
```

This same data can be reported via MDT using this sensor-path:
```
sensor-path Cisco-IOS-XR-infra-xtc-agent-oper:xtc/topology-summary
```

Not surprisinging, the metrics.json entry for this sensor-path looks like this:

```
{
                "basepath" : "Cisco-IOS-XR-infra-xtc-agent-oper:xtc/topology-summary",
                "spec" : {
                        "fields" : [
                                {"name" : "links"},
                                {"name" : "nodes"},
                                {"name" : "prefix-sids"},
                                {"name" : "adjacency-sids"},
                                {"name" : "prefixes"}
                        ]
                }
        }
```

Here is the resulting graph when I brought up my 8-node lab topology:

![SR-TE-Topo-Sum.jpg]({{site.baseurl}}/images/SR-TE-Topo-Sum.jpg)

