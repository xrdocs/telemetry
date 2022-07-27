---
published: true
date: '2022-07-27 11:21 -0600'
title: Monitoring BNG with Model-Driven Telemetry
author: Shelly Cadora
excerpt: This tutorial describes sensor-paths for basic BNG monitoring
tags:
  - iosxr
position: hidden
---
## A New Way To Monitor A Venerable Technology

Broadband Network Gateway (BNG) is a well-established technology for establishing and managing subscriber sessions, thus enabling service providers to offer differentiated services for their customers.  But even mature technologies can learn new tricks!  In this blog, I will describe the sensor-paths that allow you to monitor BNG with model-driven telemetry

## Active Sessions

When monitoring a BNG device, one of simplest but most important KPIs is the number of activated sessions. In IOS XR, you can use this CLI to get that information:

<div class="highlighter-rouge">
<pre class="highlight">
<code>
RP/0/RSP0/CPU0:R1#show subscriber session all summary
Wed Jul 27 19:19:09.440 UTC

Session Summary Information for all nodes

                Type            PPPoE           IPSub           IPSub
                                                (DHCP)          (PKT)
                ====            =====           ======          =====

Session Counts by State:
        initializing            0               0               0
          connecting            0               0               0
           connected            0               0               0
           activated            0               121             0
                idle            0               0               0
       disconnecting            0               0               0
                 end            0               0               0
              Total:            0               121             0


Session Counts by Address-Family/LAC:
         in progress            0               0               0
           ipv4-only            0               121             0
           ipv6-only            0               0               0
     dual-partial-up            0               0               0
             dual-up            0               0               0
                 lac            0               0               0
              Total:            0               121             0
RP/0/RSP0/CPU0:R1#
</code>
</pre>
</div>

The CLI returns the number of session counts organized by state or address-family, then by subscriber type (PPPoE, DHCP or IP packet).  As you can see from this output, the device current has 121 Activated IPv4 DHCP sessions.

### The Data Model for Subscriber Summary Data

This same data can be retrieved from the [Cisco-IOS-XR-iedge4710-oper.yang](https://github.com/YangModels/yang/blob/main/vendor/cisco/xr/761/Cisco-IOS-XR-iedge4710-oper.yang) data model.  Here is the relevant part of the model in tree format:

```
pyang -f tree Cisco-IOS-XR-iedge4710-oper.yang --tree-path subscriber/session/nodes/node/summary 

module: Cisco-IOS-XR-iedge4710-oper 
  +--ro subscriber 
     +--ro session 
        +--ro nodes 
           +--ro node* [node-name] 
              +--ro summary 
                 +--ro state-xr 
                 |  +--ro pppoe 
                 |  |  +--ro initialized-sessions?     uint32 
                 |  |  +--ro connecting-sessions?      uint32 
                 |  |  +--ro connected-sessions?       uint32 
                 |  |  +--ro activated-sessions?       uint32 
                 |  |  +--ro idle-sessions?            uint32 
                 |  |  +--ro disconnecting-sessions?   uint32 
                 |  |  +--ro end-sessions?             uint32 
                 |  +--ro ip-subscriber-dhcp 
                 |  |  +--ro initialized-sessions?     uint32 
                 |  |  +--ro connecting-sessions?      uint32 
                 |  |  +--ro connected-sessions?       uint32 
                 |  |  +--ro activated-sessions?       uint32 
                 |  |  +--ro idle-sessions?            uint32 
                 |  |  +--ro disconnecting-sessions?   uint32 
                 |  |  +--ro end-sessions?             uint32 
                 |  +--ro ip-subscriber-packet 
                 |     +--ro initialized-sessions?     uint32 
                 |     +--ro connecting-sessions?      uint32 
                 |     +--ro connected-sessions?       uint32 
                 |     +--ro activated-sessions?       uint32 
                 |     +--ro idle-sessions?            uint32 
                 |     +--ro disconnecting-sessions?   uint32 
                 |     +--ro end-sessions?             uint32 
                 +--ro address-family-xr 
                    +--ro pppoe 
                    |  +--ro in-progress-sessions?    uint32 
                    |  +--ro ipv4-only-sessions?      uint32 
                    |  +--ro ipv6-only-sessions?      uint32 
                    |  +--ro dual-part-up-sessions?   uint32 
                    |  +--ro dual-up-sessions?        uint32 
                    |  +--ro lac-sessions?            uint32 
                    +--ro ip-subscriber-dhcp 
                    |  +--ro in-progress-sessions?    uint32 
                    |  +--ro ipv4-only-sessions?      uint32 
                    |  +--ro ipv6-only-sessions?      uint32 
                    |  +--ro dual-part-up-sessions?   uint32 
                    |  +--ro dual-up-sessions?        uint32 
                    |  +--ro lac-sessions?            uint32 
                    +--ro ip-subscriber-packet 
                       +--ro in-progress-sessions?    uint32 
                       +--ro ipv4-only-sessions?      uint32 
                       +--ro ipv6-only-sessions?      uint32 
                       +--ro dual-part-up-sessions?   uint32 
                       +--ro dual-up-sessions?        uint32 
                       +--ro lac-sessions?            uint32
```
                      
This particular YANG model follows the CLI pretty closely, reporting the session numbers per subscriber type, by state and address-family.

### The Config for Subscriber Summary Data

```
telemetry model-driven
 destination-group TELEGRAF
  address-family ipv4 172.24.78.221 port 57000
   encoding self-describing-gpb
   protocol tcp
  !
 !
 sensor-group TEST_GROUP1
  sensor-path Cisco-IOS-XR-iedge4710-oper:subscriber/session/nodes/node/summary
 !
 subscription TEST_SUB
  sensor-group-id TEST_GROUP1 sample-interval 10000
  destination-id TELEGRAF
```

### Displaying Subscriber Summary Data

What you do with the data when you get it will depend on your collection infrastructure.  In my lab, I'm using a simple, open-source collection stack of Telegraf, InfluxDB and Grafana.  Just as an example, here is the Flux query I configured in Grafana to retrieve just the IPv4 DHCP sessions from InfluxDB:

```from(bucket: "mybucket")
  |> range(start: v.timeRangeStart, stop:v.timeRangeStop)
  |> filter(fn: (r) =>
    r._measurement == "Cisco-IOS-XR-iedge4710-oper:subscriber/session/nodes/node/summary" and
    r._field == "address_family_xr/ip_subscriber_dhcp/ipv4_only_sessions"
  )
 ```
 
 And that is enough to get me my first BNG monitoring panel:
 ![Grafana panel showing IPv4 DHCP activated sessions]({{site.baseurl}}/images/GrafanaBNGActivatedSessions.jpg)

## DHCP KPIs

You may also be interested in keeping track of DHCP statistics.  That data is reported in different models for IPv4 and IPv6.

### DHCPv4

For information on IPv4 DHCP Discovers, Offers, Requests, ACKS, NAKS and Release, use this sensor-path:
```Cisco-IOS-XR-ipv4-dhcpd-oper:ipv4-dhcpd/nodes/node/proxy/vrfs/vrf/statistics```

### DHCPv6
for DHCPv6 Solicits, Advertises, Requests and Replies use this sensor path:
```Cisco-IOS-XR-ipv6-new-dhcpv6d-oper:dhcpv6/nodes/node/proxy/vrfs/vrf/statistics```

### DHCP Proxy
For details on DHCP Proxy Binding, use this sensor path:
```Cisco-IOS-XR-ipv4-dhcpd-oper:ipv4-dhcpd/nodes/node/proxy/binding/summary```

