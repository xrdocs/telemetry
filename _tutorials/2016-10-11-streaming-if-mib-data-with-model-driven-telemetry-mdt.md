---
published: true
date: '2016-10-11 11:08 -0600'
title: Streaming IF-MIB Data with Model-Driven Telemetry (MDT)
author: Shelly Cadora
position: hidden
tags:
  - iosxr
---
## Data from the IF-MIB

One of the most commonly polled MIBs is the Interfaces MIB (IF-MIB).  Pretty much everyone needs to know how many packets and bytes were sent and received on a given interface.  So it's not surprising that one of the first questions we get is how to get the IF-MIB data from MDT. 

Below is a list of the most commonly requested IF-MIB OIDs, their corresponding YANG models, containers, leafs and any usage notes.  


| OID     | Yang-Path                                                      | YANG Leaf  | Notes |
|---------|----------------------------------------------------------------|------------|  |
| ifAlias | Cisco-IOS-XR-pfi-im-cmd-oper:interfaces/interface-xr/interface | description|  |
|ifHCInBroadcastPkts|Cisco-IOS-XR-infra-statsd-oper:infra-statistics/interfaces/interface/latest/generic-counters|broadcast-packets-received|  |
|ifHCInMulticastPkts|Cisco-IOS-XR-infra-statsd-oper:infra-statistics/interfaces/interface/latest/generic-counters|multicast-packets-received|  |
|ifHCInUcastPkts|Cisco-IOS-XR-infra-statsd-oper:infra-statistics/interfaces/interface/latest/generic-counters|N/A| Must be calcuated: packets-received - multicast-packets-received - broadcast-packets-received |
|ifHCOutBroadcastPkts|Cisco-IOS-XR-infra-statsd-oper:infra-statistics/interfaces/interface/latest/generic-counters|broadcast-packets-sent|  |
|ifHCOutMulticastPkts|Cisco-IOS-XR-infra-statsd-oper:infra-statistics/interfaces/interface/latest/generic-counters|multicast-packets-sent|  |
|ifHCOutUcastPkts|Cisco-IOS-XR-infra-statsd-oper:infra-statistics/interfaces/interface/latest/generic-counters|N/A| Must be calcuated: packets-sent - multicast-packets-sent - broadcast-packets-sent  |
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


As you can see, most of the interface statistics are in the Cisco-IOS-XR-infra-statsd-oper.yang model, with some state parameters in Cisco-IOS-XR-infra-statsd-oper, and a couple SNMP-specific values in Cisco-IOS-XR-snmp-agent-oper.  

### MDT Configuration for IF-MIB equivalence

Leaving aside the SNMP-specific parameters, here is what the sensor-path configuration in MDT would look like for the IF-MIB:

```
RP/0/RP0/CPU0:SunC(config)#telemetry model-driven
RP/0/RP0/CPU0:SunC(config-model-driven)#sensor-group SGroup1
RP/0/RP0/CPU0:SunC(config-model-driven-snsr-grp)# sensor-path Cisco-IOS-XR-infra-statsd-oper:infra-statistics/interfaces/interface/latest/generic-counters
RP/0/RP0/CPU0:SunC(config-model-driven-snsr-grp)# sensor-path Cisco-IOS-XR-pfi-im-cmd-oper:interfaces/interface-xr/interface
RP/0/RP0/CPU0:SunC(config-model-driven-snsr-grp)# commit
```  

For the complete MDT configuration, see [my tutorial](https://xrdocs.github.io/telemetry/tutorials/2016-07-21-configuring-model-driven-telemetry-mdt/).