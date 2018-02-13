---
published: true
date: '2018-02-13 13:20 -0800'
title: Filtering in Telemetry. Where to apply and why?
author: Viktor Osipchuk
excerpt: Filtering in Telemetry. Where to apply and why?
tags:
  - iosxr
  - MDT
  - Telemetry
  - Filtering in MDT
  - Streaming Telemetry
---
Let’s continue to explore smaller features available in IOS-XR Model Driven Telemetry, following the previous [post](https://xrdocs.github.io/telemetry/tutorials/2018-01-16-sample-intervals-in-telemetry-configuration), where sample intervals were explained. The goal of this post is to explain what filtering is, how you can use it and couple of benefits coming with filtering.

## Why filtering?

As an engineer, each time someone starts talking about filtering you may start thinking about electric circuits or (radio) signals. In both cases filtering is used to get rid of everything, that is not needed. Like removing of high order harmonics, that can lead to equipment damage (in electric circuits) or improper signal processing and recovery. 

Streaming Telemetry in IOS XR appeared back in 6.0.1 release and there are around 200 Native YANG models available today (with 6.3.1 release). The major goal is to make every counter in a box available to you for offline processing. If to look through the [models]( https://github.com/YangModels/yang/tree/master/vendor/cisco/xr/631), one can see that IOS XR has [mostly] a model per feature or protocol. This is very convenient and one can quickly find the needed model and start exploring it. 
But there is also another side of having a model per feature or protocol. For example, MPLS Traffic Engineering has been available in IOS XR for many years and has a very wide coverage. As the result, MPLS-TE YANG native model has more than 41k leafs of operational data! Correlate this with the number of tunnels you can have and you can see, that the amount of data streamed out could be enormous. 
With modern Streaming Telemetry you can afford this. But the question comes right away – do I really want to get all that data? In most situations, the answer will be “No”. 

## Filtering options

The very first step for you in any configuration should be an exact path definition. If you’re interested in getting summary information about your auto-mesh tunnels, there is absolutely no need to stream all other 41k different counters at all! Find what you need:

<div class="highlighter-rouge">
<pre class="highlight">
<code>
$ pyang -f tree <span style="font-weight: bold">Cisco-IOS-XR-mpls-te-oper.yang</span> --tree-path <span style="font-weight: bold">mpls-te/auto-tunnel/mesh/summary</span>
module: Cisco-IOS-XR-mpls-te-oper
   +--ro mpls-te
      +--ro auto-tunnel
         +--ro mesh
            +--ro summary
             <mark> +--ro auto-mesh-tunnels?        uint32 </mark>
             <mark> +--ro up-auto-mesh-tunnels?     uint32 </mark>
             <mark> +--ro down-auto-mesh-tunnels?   uint32 </mark>
             <mark> +--ro frr-auto-mesh-tunnels?    uint32 </mark>
             <mark> +--ro auto-mesh-groups?         uint32 </mark>
             <mark> +--ro auto-mesh-destinations?   uint32 </mark>
</code>
</pre>
</div>

And it is easy now to get your sensor path based on that information:

<div class="highlighter-rouge">
<pre class="highlight">
<code>
telemetry model-driven
 sensor-group mpls-te
   sensor-path <span style="font-weight: bold">Cisco-IOS-XR-mpls-te-oper:mpls-te/auto-tunnel/mesh/summary</span>
</code>
</pre>
</div>

As soon as you have your sensor path defined, there might be an option to do even more filtering. The first option is to define what you want to insert into your Time Series Database (TSDB) in [Pipeline](https://github.com/cisco/bigmuddy-network-telemetry-pipeline) configuration. “Metrics.json” is a file used in Pipeline to define what you want to be inserted in your TSDB. Everything NOT specified in that file will be dropped. You can find a high-level overview of this behavior with InfluxDB [here](https://xrdocs.github.io/telemetry/tutorials/2017-04-10-using-pipeline-integrating-with-influxdb) and it will be explained deeper in upcoming tutorials. 

Yet another option available for you is to do filtering on the router itself. Starting with IOS XR 6.2.2 release you can configure filtering within your sensor paths. 
Have a look at the popular Interface Stats sensor path:

<div class="highlighter-rouge">
<pre class="highlight">
<code>
$ pyang -f tree <span style="font-weight: bold">Cisco-IOS-XR-infra-statsd-oper.yang</span> --tree-path <span style="font-weight: bold">infra-statistics/interfaces/interface/latest/generic-counters</span>
module: Cisco-IOS-XR-infra-statsd-oper
   +--ro infra-statistics
      +--ro interfaces
        <mark> +--ro interface* [interface-name] </mark>
            +--ro latest
               +--ro generic-counters
                  +--ro packets-received?                    uint64
                  +--ro bytes-received?                      uint64
                  +--ro packets-sent?                        uint64
                  +--ro bytes-sent?                          uint64
                  +--ro multicast-packets-received?          uint64
 	…
</code>
</pre>
</div>

In the highlighted section you can see "interface-name" in square brackets. It shows that all the leafs below will be sent for each interface. And this is exactly the place where you can apply filtering! 
Your router might have a big number of interfaces. If you are interested in 100G interfaces only, or if you don’t care about sub-interfaces, or if you just need to get stats from your core facing ports, you can achieve this with filtering!
There are two ways to apply filtering for interfaces:
- using regex
- specifying an interface

This is an example of how you can ask a router to send you stats for 100G interfaces only:

<div class="highlighter-rouge">
<pre class="highlight">
<code>
sensor-path Cisco-IOS-XR-infra-statsd-oper:infra-statistics/interfaces/<mark>interface[interface-name='HundredGigE*']</mark>/latest/generic-counters
</code>
</pre>
</div>

An asterisk (a "*" symbol) above is used to represent "any" interface that is 100G ("HundredGigE"). This way Streaming Telemetry will push stats for all 100G interfaces on a router.

Specifying an exact interface is straightforward:

<div class="highlighter-rouge">
<pre class="highlight">
<code>
sensor-path Cisco-IOS-XR-infra-statsd-oper:infra-statistics/interfaces/<mark>interface[interface-name='HundredGigE0/0/0/0']</mark>/latest/generic-counters
</code>
</pre>
</div>

As you can see, configuration of filtering is easy and all you need to do, is to specify exactly what you want to stream out (where a model has this choice). 

You can find opportunities for filtering in other models as well:

<div class="highlighter-rouge">
<pre class="highlight">
<code>
$ pyang -f tree <span style="font-weight: bold">Cisco-IOS-XR-ip-rib-ipv4-oper.yang</span> --tree-path <span style="font-weight: bold">rib/vrfs/vrf/afs/af/safs/saf/ip-rib-route-table-names/ip-rib-route-table-name/protocol/bgp/as/information</span>
module: Cisco-IOS-XR-ip-rib-ipv4-oper
   +--ro rib
      +--ro vrfs
        <mark>+--ro vrf* [vrf-name]</mark>
            +--ro afs
              <mark>+--ro af* [af-name]</mark>
                  +--ro safs
                    <mark>+--ro saf* [saf-name]</mark>
                        +--ro ip-rib-route-table-names
                          <mark>+--ro ip-rib-route-table-name* [route-table-name]</mark>
                              +--ro protocol
                                 +--ro bgp
                                   <mark>+--ro as* [as]</mark>
                                       +--ro information
                                          +--ro protocol-names?                string
                                          +--ro instance?                      string
                                          +--ro version?                       uint32
                                          +--ro redistribution-client-count?   uint32
                                          +--ro protocol-clients-count?        uint32
                                          +--ro routes-counts?                 uint32
                                          +--ro active-routes-count?           uint32
                                          +--ro deleted-routes-count?          uint32
                                          +--ro paths-count?                   uint32
                                          +--ro protocol-route-memory?         uint32
                                          +--ro backup-routes-count?           uint32
</code>
</pre>
</div>

As you can see, there are several places where you can apply filtering. 
Usually in such models you will go with filtering for an exact element, like this: 

<div class="highlighter-rouge">
<pre class="highlight">
<code>
sensor-path Cisco-IOS-XR-ip-rib-ipv4-oper:rib/vrfs/<mark>vrf[vrf-name='MDT']</mark>/afs/af/safs/saf/ip-rib-route-table-names/ip-rib-route-table-name/protocol/bgp/as/information
</code>
</pre>
</div>


## Conclusion

As soon as you start trying telemetry, make sure you stream a specific path within a model and not all the counters from that model. It will remove wasting of resources on the router, on the wire and on the collection side. After you specified useful paths, you have 2 ways to fine tune your telemetry experience: filter on the router and filter on the collector. Filtering on the router gives you a benefit to stream data for the needed elements, like interfaces under attention, specific VRFs or address-families. On the collector side you may specify which fields you want to insert into your database. Both ways help with improving resource utilization. Filtering on the router side also brings additional not so obvious benefits to the internal MDT operation. Stay tuned, an MDT internal architecture overview will be provided soon!
