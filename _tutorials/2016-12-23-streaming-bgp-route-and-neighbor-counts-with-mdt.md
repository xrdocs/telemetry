---
published: false
date: '2016-12-23 09:45 -0700'
title: Streaming BGP Route and Neighbor Counts with MDT
---
## BGP Performance Indicators

The number of BGP routes and neighbor at any given time can be good, high-level indications network health.  Being able to stream those numbers periodically is a good use of model-driven telemetry (MDT).  The BGP YANG models are large and can be intimidating, so I thought it would be a good idea to show how to drill down for these specific stats. 

### Number of BGP Routes

For CLI fans, the information that I was looking for is often found in the output of "show route ipv4 summary":

```
RP/0/RP0/CPU0:SunC#show route ipv4  summ
Fri Sep 16 23:15:40.570 UTC
Route Source                     Routes     Backup     Deleted     Memory(bytes)
static                           1          0          0           240
local                            4          0          0           960
connected                        3          1          0           960
dagr                             0          0          0           0
bgp 5021                         147        0          0           103040
Total                            8          1          0           2160

RP/0/RP0/CPU0:SunC#
```
 
This data is included in the IOS XR native YANG model called "Cisco-IOS-XR-ip-rib-ipv4-oper.yang".  Now, there is a ton of stuff in that YANG model, including all of the prefixes in the RIB.  That's too much. I just want the summary statistics.  So I needed to filter down to a very specific tree path.  Using pyang to present a tree view of the model, here is the desired path:
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
 
As you can see from the [table below](#oid-yang-table), most of the interface statistics are in the Cisco-IOS-XR-infra-statsd-oper.yang model, with some state parameters in Cisco-IOS-XR-infra-statsd-oper.yang, and a couple SNMP-specific values in Cisco-IOS-XR-snmp-agent-oper.yang.  

Leaving aside the SNMP-specific parameters, here is what the sensor-path configuration in MDT would look like for the IF-MIB:

```
RP/0/RP0/CPU0:SunC(config)#telemetry model-driven
RP/0/RP0/CPU0:SunC(config-model-driven)#sensor-group SGroup1
RP/0/RP0/CPU0:SunC(config-model-driven-snsr-grp)# sensor-path Cisco-IOS-XR-infra-statsd-oper:infra-statistics/interfaces/interface/latest/generic-counters
RP/0/RP0/CPU0:SunC(config-model-driven-snsr-grp)# sensor-path Cisco-IOS-XR-pfi-im-cmd-oper:interfaces/interface-xr/interface
RP/0/RP0/CPU0:SunC(config-model-driven-snsr-grp)# commit
```  

For the complete MDT configuration, see [my configuration tutorial](https://xrdocs.github.io/telemetry/tutorials/2016-07-21-configuring-model-driven-telemetry-mdt/).

With that, you should be streaming all your favorite IF-MIB data at a fraction of the cost of doing an SNMP poll. 

### OID-YANG Table

Below is a table of the most commonly requested IF-MIB OIDs, their corresponding YANG models, containers, leafs and any usage notes.  


| OID     | Yang-Path                                                      | YANG Leaf  | Notes |
|---------|----------------------------------------------------------------|------------|  |
| ifAlias | Cisco-IOS-XR-pfi-im-cmd-oper:interfaces/interface-xr/interface | description|  |
|ifHCInBroadcastPkts|Cisco-IOS-XR-infra-statsd-oper:infra-statistics/interfaces/interface/latest/generic-counters|broadcast-packets-received|  |
|ifHCInMulticastPkts|Cisco-IOS-XR-infra-statsd-oper:infra-statistics/interfaces/interface/latest/generic-counters|multicast-packets-received|  |
|ifHCInUcastPkts|Cisco-IOS-XR-infra-statsd-oper:infra-statistics/interfaces/interface/latest/generic-counters|N/A| Must be calculated: packets-received - multicast-packets-received - broadcast-packets-received |
|ifHCOutBroadcastPkts|Cisco-IOS-XR-infra-statsd-oper:infra-statistics/interfaces/interface/latest/generic-counters|broadcast-packets-sent|  |
|ifHCOutMulticastPkts|Cisco-IOS-XR-infra-statsd-oper:infra-statistics/interfaces/interface/latest/generic-counters|multicast-packets-sent|  |
|ifHCOutUcastPkts|Cisco-IOS-XR-infra-statsd-oper:infra-statistics/interfaces/interface/latest/generic-counters|N/A| Must be calculated: packets-sent - multicast-packets-sent - broadcast-packets-sent  |
|ifIndex|Cisco-IOS-XR-snmp-agent-oper:snmp/interface-indexes/|if-index|  |
|ifLastChange|Cisco-IOS-XR-pfi-im-cmd-oper:interfaces/interface-xr/interface|last-state-transition-time| last-state-transition-time is the elapsed time since last state change while ifLastChange is the sysUpTime value of the last state change |
|ifOutDiscards| Cisco-IOS-XR-infra-statsd-oper:infra-statistics/interfaces/interface/latest/generic-counters|output-drops|  |
|ifOutErrors|Cisco-IOS-XR-infra-statsd-oper:infra-statistics/interfaces/interface/latest/generic-counters|output-errors|  |
|ifStackStatus|Cisco-IOS-XR-snmp-agent-oper/snmp/|if-stack-status|  |
|ifAdminStatus|Cisco-IOS-XR-pfi-im-cmd-oper:interfaces/interface-xr/interface|state|  |
|ifDescr|Cisco-IOS-XR-pfi-im-cmd-oper:interfaces/interface-xr/interface|interface-name|  |
|ifHCInOctets|Cisco-IOS-XR-infra-statsd-oper:infra-statistics/interfaces/interface/latest/generic-counters|bytes-received|  |
|ifHCOutOctets|Cisco-IOS-XR-infra-statsd-oper:infra-statistics/interfaces/interface/latest/generic-counters|bytes-sent|  |
|ifHighSpeed|Cisco-IOS-XR-pfi-im-cmd-oper:interfaces/interface-xr/interface|speed| ifHighSpeed is in Mbps, speed is in kbps |
|ifInErrors|Cisco-IOS-XR-infra-statsd-oper:infra-statistics/interfaces/interface/latest/generic-counters|input-errors|  |
|ifOperStatus|Cisco-IOS-XR-pfi-im-cmd-oper:interfaces/interface-xr/interface|state|  |
|ifPhysAddress|Cisco-IOS-XR-pfi-im-cmd-oper:interfaces/interface-xr/interface|address|  |
|ifType|Cisco-IOS-XR-pfi-im-cmd-oper:interfaces/interface-xr/interface|interface-type|  |
|ifInDiscards|Cisco-IOS-XR-infra-statsd-oper:infra-statistics/interfaces/interface/latest/generic-counters|input-drops|  |
|ifInOctets|Cisco-IOS-XR-infra-statsd-oper:infra-statistics/interfaces/interface/latest/generic-counters|bytes-received|  |
|ifMtu|Cisco-IOS-XR-pfi-im-cmd-oper:interfaces/interface-xr/interface|mtu|  |
|ifName|Cisco-IOS-XR-infra-statsd-oper:infra-statistics/interfaces/interface/latest/generic-counters|interface-name| interface-name format is "HundredGigE0_3_0_0" |
|ifOutOctets|Cisco-IOS-XR-infra-statsd-oper:infra-statistics/interfaces/interface/latest/generic-counters|bytes-sent|  |
|ifSpeed|Cisco-IOS-XR-pfi-im-cmd-oper:interfaces/interface-xr/interface|bandwidth|  |
