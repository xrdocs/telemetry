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

| MIB    | OID     | Yang-Path                                                      | YANG Leaf   |
|--------|---------|----------------------------------------------------------------|-------------|
| IF-MIB | ifAlias | Cisco-IOS-XR-pfi-im-cmd-oper:interfaces/interface-xr/interface | description |
|IF-MIB|ifHCInBroadcastPkts|Cisco-IOS-XR-infra-statsd-oper:infra-statistics/interfaces/interface/latest/generic-counters|broadcast-packets-received|
|IF-MIB|ifHCInMulticastPkts|Cisco-IOS-XR-infra-statsd-oper:infra-statistics/interfaces/interface/latest/generic-counters|multicast-packets-received|
|IF-MIB|ifHCInUcastPkts|Cisco-IOS-XR-infra-statsd-oper:infra-statistics/interfaces/interface/latest/generic-counters|packets-received|
|IF-MIB|ifHCOutBroadcastPkts|Cisco-IOS-XR-infra-statsd-oper:infra-statistics/interfaces/interface/latest/generic-counters|broadcast-packets-sent|
|IF-MIB|ifHCOutMulticastPkts|Cisco-IOS-XR-infra-statsd-oper:infra-statistics/interfaces/interface/latest/generic-counters|multicast-packets-sent|
|IF-MIB|ifHCOutUcastPkts|Cisco-IOS-XR-infra-statsd-oper:infra-statistics/interfaces/interface/latest/generic-counters|packets-sent|
|IF-MIB|ifIndex|" |
|Sensor-path: Cisco-IOS-XR-snmp-agent-oper:snmp/interface-indexes/|
|"|if-index|
|IF-MIB|ifLastChange|"|
|Sensor-path: Cisco-IOS-XR-pfi-im-cmd-oper:interfaces/interface-xr/interface|
|"|last-state-transition-time|
|IF-MIB|ifOutDiscards| sensor-path Cisco-IOS-XR-pfi-im-cmd-oper:interfaces/interface-xr/interface not resolved|output-drops|
|IF-MIB|ifOutErrors||output-errors|
|IF-MIB|ifStackStatus|Sensor-path : Cisco-IOS-XR-snmp-agent-oper/snmp/|if-stack-status|
|IF-MIB|ifAdminStatus|Cisco-IOS-XR-pfi-im-cmd-oper:interfaces/interface-xr/interface|state|
|IF-MIB|ifDescr|Cisco-IOS-XR-pfi-im-cmd-oper:interfaces/interface-xr/interface|interface-name|
|IF-MIB|ifHCInOctets|Cisco-IOS-XR-infra-statsd-oper:infra-statistics/interfaces/interface/latest/generic-counters|bytes-received|
|IF-MIB|ifHCOutOctets|Cisco-IOS-XR-infra-statsd-oper:infra-statistics/interfaces/interface/latest/generic-counters|bytes-sent|
|IF-MIB|ifHighSpeed|Cisco-IOS-XR-pfi-im-cmd-oper:interfaces/interface-xr/interface|speed|
|IF-MIB|ifInErrors|Cisco-IOS-XR-infra-statsd-oper:infra-statistics/interfaces/interface/latest/generic-counters|input-errors|
|IF-MIB|ifOperStatus|Cisco-IOS-XR-pfi-im-cmd-oper:interfaces/interface-xr/interface|state|
|IF-MIB|ifPhysAddress|Cisco-IOS-XR-pfi-im-cmd-oper:interfaces/interface-xr/interface|address|
|IF-MIB|ifType|Cisco-IOS-XR-pfi-im-cmd-oper:interfaces/interface-xr/interface|interface-type|
|IF-MIB|ifInDiscards|Cisco-IOS-XR-infra-statsd-oper:infra-statistics/interfaces/interface/latest/generic-counters|input-drops|
|IF-MIB|ifInOctets|Cisco-IOS-XR-infra-statsd-oper:infra-statistics/interfaces/interface/latest/generic-counters|bytes-received|
|IF-MIB|ifMtu|Cisco-IOS-XR-pfi-im-cmd-oper:interfaces/interface-xr/interface|mtu|
|IF-MIB|ifName|Cisco-IOS-XR-infra-statsd-oper:infra-statistics/interfaces/interface/latest/generic-counters|interface-name|
|IF-MIB|ifOutOctets|Cisco-IOS-XR-infra-statsd-oper:infra-statistics/interfaces/interface/latest/generic-counters|bytes-sent|
|IF-MIB|ifSpeed|Cisco-IOS-XR-pfi-im-cmd-oper:interfaces/interface-xr/interface|bandwidth|