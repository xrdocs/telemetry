---
published: true
date: '2016-12-23 09:45 -0700'
title: Streaming BGP Route and Neighbor Counts with MDT
excerpt: >-
  Describes how to configure sensor-paths to stream BGP route counts and
  neighbor counts
tags:
  - iosxr
author: Shelly Cadora
position: hidden
---
## BGP Performance Indicators

The number of BGP routes and neighbor at any given time can be good, high-level indicators of network health.  Being able to stream those numbers periodically is a good use of model-driven telemetry (MDT).  The BGP YANG models are large and can be intimidating, so this tutorial shows how to drill down for these specific stats. 

### Number of BGP Routes

For CLI fans, the information that I was looking for is often found in the output of "show route ipv4 summary":

```
RP/0/RP0/CPU0:SunC#show route ipv4 summary
Fri Dec 23 16:48:59.988 UTC
Route Source                     Routes     Backup     Deleted     Memory(bytes)
local                            3          0          0           720
connected                        2          1          0           720
static                           1          0          0           240
dagr                             0          0          0           0
bgp 1                            5          1          0           1440
isis 1                           1          1          0           480
Total                            12         3          0           3600

RP/0/RP0/CPU0:SunC#
```
 
This data is included in the IOS XR native YANG model called "Cisco-IOS-XR-ip-rib-ipv4-oper.yang".  Now, there is a ton of stuff in that YANG model, including all of the prefixes in the RIB.  That's too much. I just want the summary statistics.  So I needed to filter down to a very specific tree path.  Using [pyang](https://github.com/mbj4668/pyang) to present a tree view of the model, here is the desired path:

```
$ pyang -f tree Cisco-IOS-XR-ip-rib-ipv4-oper.yang --tree-path rib/vrfs/vrf/afs/af/safs/saf/ip-rib-route-table-names/ip-rib-route-table-name/protocol/bgp/as/information

module: Cisco-IOS-XR-ip-rib-ipv4-oper
   +--ro rib
      +--ro vrfs
         +--ro vrf* [vrf-name]
            +--ro afs
               +--ro af* [af-name]
                  +--ro safs
                     +--ro saf* [saf-name]
                        +--ro ip-rib-route-table-names
                           +--ro ip-rib-route-table-name* [route-table-name]
                              +--ro protocol
                                 +--ro bgp
                                    +--ro as* [as]
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
$
```

If you did a NETCONF <get> operation on this subtree, the data would be returned encoded in XML like this:

```
<?xml version="1.0"?>
<rpc-reply message-id="urn:uuid:7aa4d7d8-4638-40ed-bb87-93b1403e0baa" xmlns:nc="urn:ietf:params:xml:ns:netconf:base:1.0" xmlns="urn:ietf:params:xml:ns:netconf:base:1.0">
 <data>
  <rib xmlns="http://cisco.com/ns/yang/Cisco-IOS-XR-ip-rib-ipv4-oper">
   <vrfs>
    <vrf>
     <vrf-name>default</vrf-name>
     <afs>
      <af>
       <af-name>IPv4</af-name>
       <safs>
        <saf>
         <saf-name>Unicast</saf-name>
         <ip-rib-route-table-names>
          <ip-rib-route-table-name>
           <route-table-name>default</route-table-name>
           <protocol>
            <bgp>
             <as>
              <as>1</as>
              <information>
               <protocol-names>bgp</protocol-names>
               <instance>1</instance>
               <version>0</version>
               <redistribution-client-count>0</redistribution-client-count>
               <protocol-clients-count>1</protocol-clients-count>
               <routes-counts>6</routes-counts>
               <active-routes-count>5</active-routes-count>
               <deleted-routes-count>0</deleted-routes-count>
               <paths-count>6</paths-count>
               <protocol-route-memory>1440</protocol-route-memory>
               <backup-routes-count>1</backup-routes-count>
              </information>
             </as>
            </bgp>
           </protocol>
          </ip-rib-route-table-name>
         </ip-rib-route-table-names>
        </saf>
       </safs>
      </af>
     </afs>
    </vrf>
   </vrfs>
  </rib>
 </data>
</rpc-reply>
```

To get this same data encoded in Google Protocol Buffers and streamed using MDT, just configure a sensor path as follows:

To get that data via MDT,  configure the sensor path like this:

```
telemetry model-driven
  sensor-group SGroup1
   sensor-path Cisco-IOS-XR-ip-rib-ipv4-oper:rib/vrfs/vrf/afs/af/safs/saf/ip-rib-route-table-names/ip-rib-route-table-name/protocol/bgp/as/information
```
 
Notice that the subtree filter (everything after **Cisco-IOS-XR-ip-rib-ipv4-oper:** in the sensor-path) is exactly the same as the argument I passed to the --tree-path filter in pyang.  That's a handy tip for constructing sensor-paths in general!

### Number of BGP Neighbors

For BGP neighbor counts, the model you want is Cisco-IOS-XR-ipv4-bgp-oper.yang.  Again, this is a very juicy model, so it's best to winnow it down to the nearest subtree:

```
$ pyang -f tree Cisco-IOS-XR-ipv4-bgp-oper.yang --tree-path bgp/instances/instance/instance-active/default-vrf/process-info/global
module: Cisco-IOS-XR-ipv4-bgp-oper
   +--ro bgp
      +--ro instances
         +--ro instance* [instance-name]
            +--ro instance-active
               +--ro default-vrf
                  +--ro process-info
                     +--ro global
                        +--ro process-instance-node?                string
                        +--ro restart-count?                        uint32
                        +--ro path-attributes-entry-count?          uint32
                        +--ro path-attribute-memory?                uint32
                        +--ro as-path-entry-count?                  uint32
                        +--ro as-path-entries-memory?               uint32
                        +--ro community-entry-count?                uint32
                        +--ro community-memory?                     uint32
                        +--ro extended-community-entry-count?       uint32
                        +--ro extended-community-memory?            uint32
                        +--ro pe-distinguisher-label-entry-count?   uint32
                        +--ro pe-distinguisher-label-memory?        uint32
                        +--ro pta-entry-count?                      uint32
                        +--ro pta-memory?                           uint32
                        +--ro ribrnh-entry-count?                   uint32
                        +--ro ribrnh-memory?                        uint32
                        +--ro ppmp-entry-count?                     uint32
                        +--ro ppmp-memory?                          uint32
                        +--ro route-reflectors?                     uint32
                        +--ro route-reflector-memory?               uint32
                        +--ro nexthop-count?                        uint32
                        +--ro nexthop-memory?                       uint32
                        +--ro local-as?                             uint32
                        +--ro total-vrf-count?                      uint32
                        +--ro neighbors-count-total?                uint32
                        +--ro established-neighbors-count-total?    uint32
                        +--ro sn-num-non-dflt-vrf-nbrs?             uint32
                        +--ro sn-num-non-dflt-vrf-nbrs-estab?       uint32
                        +--ro pool-size*                            uint32
                        +--ro pool-alloc-count*                     uint32
                        +--ro pool-free-count*                      uint32
                        +--ro msg-log-pool-size*                    uint32
                        +--ro msg-log-pool-alloc-count*             uint32
                        +--ro msg-log-pool-free-count*              uint32
                        ```

Depending on whether you're interested in all neighbors or just neighbors in the established state, you can grab neighbors-count-total or established-neighbors-count-total from that list.

To get that data via MDT,  configure the sensor path like this:

```
telemetry model-driven
  sensor-group SGroup1
   sensor-path Cisco-IOS-XR-ipv4-bgp-oper:bgp/instances/instance/instance-active/default-vrf/process-info
```