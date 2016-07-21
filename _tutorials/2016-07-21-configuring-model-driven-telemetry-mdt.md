---
published: true
date: '2016-07-21 09:06 -0600'
title: Configuring Model-Driven Telemetry (MDT)
position: hidden
author: Shelly Cadora
excerpt: Quick Start for MDT Configuration
tags:
  - iosxr
  - Telemetry
---

## Important Background (aka TL;DR)

Before configuring Model-Driven Telemetry, you should understand the different options that are available for encoding and transport and pick the combination that works for you.  Here's a quick summary:  
-**Transport:** The router can deliver telemetry data either across using TCP or [gRPC](https://www.google.com/url?sa=t&rct=j&q=&esrc=s&source=web&cd=1&cad=rja&uact=8&ved=0ahUKEwiSv9Tl7ITOAhUTzGMKHb-ICh0QFggeMAA&url=http%3A%2F%2Fwww.grpc.io%2F&usg=AFQjCNHCJm5rnsywES2mSmRVZrlyDB3Ebw&sig2=ovceQ1KnEIWKo7364lLhUg) over HTTP/2.  Some people will prefer the simplicity of a raw TCP socket, others will appreciate the optional TLS encyption that gRPC brings.  
-**Session Initiation:** There are two options for initiating a telemetry session.  The router can "dial-out" to the collector or the collector can "dial-in" to the router.  Regardless of which side initiates the session, the router always streams the data to the collector at the requested intervals. TCP supports "dial-out" while gRPC supports both "dial-in" and "dial-out."  
-**Encoding:** The router can deliver telemetry data in two different flavors of Google Protocol Buffers: [Compact and Self-Describing GPB](http://blogs.cisco.com/sp/streaming-telemetry-with-google-protocol-buffers).  Compact GPB is the most efficient encoding but requires a unique .proto for each YANG model that is streamed.  Self-describing GPB is less efficient but it uses a single .proto file to decode all YANG models because the keys are passed as strings in the .proto.  

## Using TCP Dial-Out
With the TCP Dial-Out method, the router initiates a TCP session to the collector and sends whatever data is specified by the sensor-group in the subscription.
 
### TCP Dial-Out Router Config
There are three steps to configuring the router for telemetry with TCP dial-out:create a destination-group, create a sensor-group, create a subscription.
 
#### Step 1: Create a destination-group
The destination-group specifies the destination address, port, encoding and transport that the router should use to send out telemetry data.  In this case, we configure the router to send telemetry via tcp, encoding as self-describing gpb, to 172.30.8.4 port 5432.  
```
RP/0/RP0/CPU0:SunC(config)#telemetry model-driven  

RP/0/RP0/CPU0:SunC(config-model-driven)# destination-group DGroup1  

RP/0/RP0/CPU0:SunC(config-model-driven-dest)#  address family ipv4 172.30.8.4 port 5432  

RP/0/RP0/CPU0:SunC(config-model-driven-dest-addr)#   encoding self-describing-gpb  
RP/0/RP0/CPU0:SunC(config-model-driven-dest-addr)#   protocol tcp  
RP/0/RP0/CPU0:SunC(config-model-driven-dest-addr)# commit  
```  

#### Step 2: Create a sensor-group
The sensor-group specifies a list of YANG models which are to be streamed.  The sensor path below represents the YANG model for interfaces statistics:
```
RP/0/RP0/CPU0:SunC(config)#telemetry model-driven   
RP/0/RP0/CPU0:SunC(config-model-driven)#sensor-group SGroup1  
RP/0/RP0/CPU0:SunC(config-model-driven-snsr-grp)# sensor-path Cisco-IOS-XR-infra-statsd-oper:infra-statistics/interfaces/interface/latest/generic-counters  
RP/0/RP0/CPU0:SunC(config-model-driven-snsr-grp)# commit  
```
#### Step 3: Create a subscription  
The subscription associates a destination-group with a sensor-group and sets the streaming interval.  The following configuration associates the sensor-group and destination created above with a streaming interval of 30 seconds.  
```
RP/0/RP0/CPU0:SunC(config)telemetry model-driven  
RP/0/RP0/CPU0:SunC(config-model-driven)#subscription Sub1  
RP/0/RP0/CPU0:SunC(config-model-driven-subs)#sensor-group-id SGroup1 sample-interval 30000  
RP/0/RP0/CPU0:SunC(config-model-driven-subs)#destination-id DGroup1  
RP/0/RP0/CPU0:SunC(config-mdt-subscription)# commit  
```

#### All Together Now
Here's the entire configuration for TCP dial-out with GPB encoding in one shot:  

```
telemetry model-driven  
 destination-group DGroup1  
   address family ipv4 172.30.8.4 port 5432  
   encoding self-describing-gpb  
   protocol tcp
  !
 !
 sensor-group SGroup1
  sensor-path Cisco-IOS-XR-infra-statsd-oper:infra-statistics/interfaces/interface/latest/generic-counters
 !  
 subscription Sub1  
  sensor-group-id SGroup1 sample-interval 30000  
  destination-id DGroup1   
``` 

#### Validation
Use the following command to verify that you have correctly configured the router for TCP dial-out.
```
RP/0/RP0/CPU0:SunC#show telemetry model-driven subscription
Thu Jul 21 15:42:27.751 UTC
Subscription:  Sub1                     State: ACTIVE
-------------
  Sensor groups:
  Id                Interval(ms)        State
  SGroup1           30000               Resolved

  Destination Groups:
  Id                Encoding            Transport   State   Port    IP
  DGroup1           self-describing-gpb tcp         Active  5432    172.30.8.4
```


## Using gRPC Dial-Out
With the gRPC Dial-Out method, the router initiates a gRPC session to the collector and sends whatever data is specified by the sensor-group in the subscription.
 
### gRPC Dial-Out Router Config
The steps to configure gRPC dial-out are the same as TCP dial-out: create a destination-group, create a sensor-group, create a subscription.

#### Step 1: Create a destination-group
The destination-group specifies the destination address, port, encoding and transport that the router should use to send out telemetry data.  In this case, we configure the router to send telemetry via gRPC, encoding as self-describing gpb, to 172.30.8.4 port 57500.  
  
RP/0/RP0/CPU0:SunC(config)#telemetry model-driven  
RP/0/RP0/CPU0:SunC(config-model-driven)# destination-group DGroup2  
RP/0/RP0/CPU0:SunC(config-model-driven-dest)#  address family ipv4 172.30.8.4 port 57500  
RP/0/RP0/CPU0:SunC(config-model-driven-dest-addr)#   encoding self-describing-gpb  
RP/0/RP0/CPU0:SunC(config-model-driven-dest-addr)#   protocol grpc  
RP/0/RP0/CPU0:SunC(config-model-driven-dest-addr)# commit  
   
#### Step 2: Create a sensor-group
The sensor-group specifies a list of YANG models which are to be streamed.  The sensor path below represents the YANG model for summarized memory statistics:
  
RP/0/RP0/CPU0:SunC(config)#telemetry model-driven   
RP/0/RP0/CPU0:SunC(config-model-driven)#sensor-group SGroup2  
RP/0/RP0/CPU0:SunC(config-model-driven-snsr-grp)# sensor-path Cisco-IOS-XR-nto-misc-oper:memory-summary/nodes/node/summary  
RP/0/RP0/CPU0:SunC(config-model-driven-snsr-grp)# commit  
  
#### Step 3: Create a subscription
The subscription associates a destination-group with a sensor-group and sets the streaming interval.  The following configuration associates the sensor-group and destination created above with a streaming interval of 30 seconds.  
RP/0/RP0/CPU0:SunC(config)telemetry model-driven  
RP/0/RP0/CPU0:SunC(config-model-driven)#subscription Sub2  
RP/0/RP0/CPU0:SunC(config-model-driven-subs)#sensor-group-id SGroup2 sample-interval 30000  
RP/0/RP0/CPU0:SunC(config-model-driven-subs)#destination-id DGroup2  
RP/0/RP0/CPU0:SunC(config-mdt-subscription)# commit  

#### All Together Now
Here's the entire configuration for gRPC dial-out with GPB encoding in one shot:  
```
telemetry model-driven
 destination-group DGroup2
  address family ipv4 172.30.8.4 port 57500
   encoding self-describing-gpb
   protocol grpc
  !
 !
 sensor-group SGroup2
  sensor-path Cisco-IOS-XR-nto-misc-oper:memory-summary/nodes/node/summary
 !
 subscription Sub2
  sensor-group-id SGroup2 sample-interval 30000
  destination-id DGroup2
``` 

#### Validation
Use the following command to verify that you have correctly configured the router for gRPC dial-out.
```
RP/0/RP0/CPU0:SunC#show telemetry model-driven subscription
Thu Jul 21 21:14:08.636 UTC
Subscription:  Sub2                     State: ACTIVE
-------------
  Sensor groups:
  Id                Interval(ms)        State
  SGroup2           30000               Resolved

  Destination Groups:
  Id                Encoding            Transport   State   Port    IP
  DGroup2           self-describing-gpb grpc        NA      57500   172.30.8.4
```

## Using gRPC Dial-In
With the gRPC Dial-In method, the collector initiates a gRPC session to the collector.  The router sends whatever data is specified by the sensor-group in the subscription requested by the collector.
 
### gRPC Dial-In Router Config
There are three steps to configure gRPC dial-out: enable gRPC, create a sensor-group, create a subscription.

#### Step 1: Enable gRPC
The following configuration enables the router's gRPC server to accept incoming connections from the collector.  
  
RP/0/RP0/CPU0:SunC(config)#grpc  
RP/0/RP0/CPU0:SunC(config-grpc)#port 57500  
RP/0/RP0/CPU0:SunC(config-grpc)#commit  
   
#### Step 2: Create a sensor-group
The sensor-group specifies a list of YANG models which are to be streamed.  The sensor path below represents the OpenConfig YANG model for interfaces:
  
RP/0/RP0/CPU0:SunC(config)#telemetry model-driven   
RP/0/RP0/CPU0:SunC(config-model-driven)#sensor-group SGroup3  
RP/0/RP0/CPU0:SunC(config-model-driven-snsr-grp)# sensor-path openconfig-interfaces:interfaces/interface  
RP/0/RP0/CPU0:SunC(config-model-driven-snsr-grp)# commit  
  
#### Step 3: Create a subscription
The subscription associates a sensor-group with the streaming interval.  No destination group is required because the collector will be dialing in.  The collector will need to request subscription "Sub3" when it connects.    
RP/0/RP0/CPU0:SunC(config)telemetry model-driven  
RP/0/RP0/CPU0:SunC(config-model-driven)#subscription Sub3  
RP/0/RP0/CPU0:SunC(config-model-driven-subs)#sensor-group-id SGroup3 sample-interval 30000  
RP/0/RP0/CPU0:SunC(config-mdt-subscription)# commit  

#### All Together Now
Here's the entire configuration for gRPC dial-in in one shot:  
```
grpc
 port 57500
!
telemetry model-driven
 sensor-group SGroup3
  sensor-path openconfig-interfaces:interfaces/interface
 !
 subscription Sub3
  sensor-group-id SGroup3 sample-interval 30000

``` 

#### Validation
Use the following command to verify that you have correctly configured the router for gRPC dial-in.
```
RP/0/RP0/CPU0:SunC#show telemetry model-driven subscription Sub3
Thu Jul 21 21:32:45.365 UTC
Subscription:  Sub3
-------------
  State:       ACTIVE
  Sensor groups:
  Id: SGroup3
    Sample Interval:      30000 ms
    Sensor Path:          openconfig-interfaces:interfaces/interface
    Sensor Path State:    Resolved

  Destination Groups:
  Group Id: DialIn_1002
    Destination IP:       172.30.8.4
    Destination Port:     44841
    Encoding:             self-describing-gpb
    Transport:            dialin
    State:                Active
    Total bytes sent:     13909
    Total packets sent:   14
    Last Sent time:       2016-07-21 21:32:25.231964501 +0000

  Collection Groups:
  ------------------
    Id: 2
    Sample Interval:      30000 ms
    Encoding:             self-describing-gpb
    Num of collection:    7
    Collection time:      Min:    32 ms Max:    39 ms
    Total time:           Min:    34 ms Avg:    37 ms Max:    40 ms
    Total Deferred:       0
    Total Send Errors:    0
    Total Send Drops:     0
    Total Other Errors:   0
    Last Collection Start:2016-07-21 21:32:25.231930501 +0000
    Last Collection End:  2016-07-21 21:32:25.231969501 +0000
    Sensor Path:          openconfig-interfaces:interfaces/interface

```
