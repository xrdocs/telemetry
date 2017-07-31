---
published: true
date: "2016-05-13 16:30 -0600"
title: "Filtering Telemetry for RSVP-TE Auto-Bandwidth Demo"
author: Shelly Cadora
excerpt: An introduction to filtering with streaming telemetry
permalink: /blogs/filtering_autobw
tags: 
  - iosxr
  - telemetry
---


Telemetry recently took center stage at [SDX Demo Friday](https://www.sdxcentral.com/resources/sdn-demofriday/cisco-ios-xr-signalfx-demo-monitoring-your-modern-network/) with a new demo showcasing the RSVP-TE auto-bandwidth feature.  In the demo, the TE tunnel headend streamed data about the output bytes sent per tunnel and the resulting applied auto-bandwidth.  We streamed the data to SignalFX's cloud monitoring system and were able to show nice visualizations of auto-bandwidth in action as well as some cool alerting capabilities.

![AutoBW.jpg]({{site.baseurl}}/images/AutoBW.jpg)


This demo gave us an opportunity to exercise the new filtering capability in IOS XR 6.0.1. The operational data for a TE Tunnel headend is contained in the native path RootOper.MPLS_TE.P2P_P2MPTunnel.TunnelHead({'TunnelName': 'tunnel-te10'}). Streaming this path would result in over 600 lines of output for a single tunnel, with multiple layers of hierarchy.  To filter that data down to a single value, you can use a IncludeField in the policy file as follows: 

```json
{
   "Name":"RSVPTEPolicy",
   "Metadata":{
      "Version":1,
      "Description":"This policy collects auto bw stats",
      "Comment":"This is the first draft"
   },
   "CollectionGroups":{
      "FirstGroup":{
         "Period":10,
         "Paths":{
            "RootOper.MPLS_TE.P2P_P2MPTunnel.TunnelHead({'TunnelName': 'tunnel-te10'})":{
               "IncludeFields":[
                  {
                     "P2PInfo":[
                        {
                           "AutoBandwidthOper":[
                              "LastBandwidthApplied"
                           ]
                        }
                     ]
                  }
               ]
            }
         }
      }
   }
}
```

With this IncludeFilter, the telemetry engine will only encode the latest applied auto-bandwidth value for the specified TE tunnel (which is nested two levels down from the top level of the path).  

Filtering on the router is a big win from a performance perspective.  Obviously, the collector (SignalFX in this case) benefits if it has to process less data.  But the router benefits, too.  Internally, the telemetry process still retrieves the entire "bag" of data (i.e. the 600 lines of stuff) to take advantage of bulk retrieval optimizations.  That's pretty much a fixed cost in terms of CPU utilization.  The good news is that filtering optimizations allow the encoding process to completely skip over everything except the fields you want.  Skipping is much more efficient than processing, which decreases the demand on the CPU.  Moreover, the resulting data stream is much smaller.  Since a large proportion of the telemetry CPU usage is for packet IO, a smaller data stream again means less CPU utilization.  So for maximum efficiency, filter when you can!
