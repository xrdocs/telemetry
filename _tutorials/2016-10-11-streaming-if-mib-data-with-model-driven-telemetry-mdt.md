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

Below is a list of the most commonly requested IF-MIB OIDs, their YANG models, containers and leafs.  All the OID data is available through MDT, but there are some gotchas.  For example, telemetry and SNMP might return the values with different units or SNMP might return an enum where telemetry returns a string, etc.

| OID     | Yang-Path                                                      | YANG Leaf  | Notes |
|---------|----------------------------------------------------------------|------------|  |
| ifAlias | Cisco-IOS-XR-pfi-im-cmd-oper:interfaces/interface-xr/interface | description|  |
|ifHCInBroadcastPkts|Cisco-IOS-XR-infra-statsd-oper:infra-statistics/interfaces/interface/latest/generic-counters|broadcast-packets-received|  |
|ifHCInMulticastPkts|Cisco-IOS-XR-infra-statsd-oper:infra-statistics/interfaces/interface/latest/generic-counters|multicast-packets-received|  |
|ifHCInUcastPkts|Cisco-IOS-XR-infra-statsd-oper:infra-statistics/interfaces/interface/latest/generic-counters|packets-received|  |
|ifHCOutBroadcastPkts|Cisco-IOS-XR-infra-statsd-oper:infra-statistics/interfaces/interface/latest/generic-counters|broadcast-packets-sent|  |
|ifHCOutMulticastPkts|Cisco-IOS-XR-infra-statsd-oper:infra-statistics/interfaces/interface/latest/generic-counters|multicast-packets-sent|  |
|ifHCOutUcastPkts|Cisco-IOS-XR-infra-statsd-oper:infra-statistics/interfaces/interface/latest/generic-counters|packets-sent|  |
|ifIndex|Cisco-IOS-XR-snmp-agent-oper:snmp/interface-indexes/|if-index|  |
|ifLastChange|Cisco-IOS-XR-pfi-im-cmd-oper:interfaces/interface-xr/interface|last-state-transition-time| last-state-transition-time is the elapsed time since last state change while ifLastChange is the sysUpTime value of the last state change |
|ifOutDiscards| sensor-path Cisco-IOS-XR-pfi-im-cmd-oper:interfaces/interface-xr/interface not resolved|output-drops|  |
|ifOutErrors|Cisco-IOS-XR-infra-statsd-oper:infra-statistics/interfaces/interface/latest/generic-counters|output-errors|  |
|ifStackStatus|Sensor-path : Cisco-IOS-XR-snmp-agent-oper/snmp/|if-stack-status|  |
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
|ifName|Cisco-IOS-XR-infra-statsd-oper:infra-statistics/interfaces/interface/latest/generic-counters|interface-name|  |
|ifOutOctets|Cisco-IOS-XR-infra-statsd-oper:infra-statistics/interfaces/interface/latest/generic-counters|bytes-sent|  |
|ifSpeed|Cisco-IOS-XR-pfi-im-cmd-oper:interfaces/interface-xr/interface|bandwidth|  |