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

Below is a list of the most commonly requested IF-MIB OIDs, their YANG models, containers and leafs, and any caveats about the data.  All the OID data is available through MDT, but there are some gotchas.  For example, telemetry and SNMP might return the values with different units or SNMP might return an enum where telemetry returns a string, etc.

| MIB    | OID     | Yang-Path                                                      | YANG Leaf   |
|--------|---------|----------------------------------------------------------------|-------------|
| IF-MIB | ifAlias | Cisco-IOS-XR-pfi-im-cmd-oper:interfaces/interface-xr/interface | description |
|IF-MIB|ifHCInBroadcastPkts|Cisco-IOS-XR-infra-statsd-oper:infra-statistics/interfaces/interface/latest/generic-counters|broadcast-packets-received|
|IF-MIB|ifHCInMulticastPkts|Cisco-IOS-XR-infra-statsd-oper:infra-statistics/interfaces/interface/latest/generic-counters|multicast-packets-received|