---
published: false
date: '2016-07-21 09:06 -0600'
title: Configuring Model-Driven Telemetry (MDT)
---
## Important Background

Before configuring Model-Driven Telemetry, you should understand the different options that are available for encoding and transport and pick the combination that works for you.  Here's a quick summary:  
-**Transport:** The router can deliver telemetry data either across using TCP or [gRPC](https://www.google.com/url?sa=t&rct=j&q=&esrc=s&source=web&cd=1&cad=rja&uact=8&ved=0ahUKEwiSv9Tl7ITOAhUTzGMKHb-ICh0QFggeMAA&url=http%3A%2F%2Fwww.grpc.io%2F&usg=AFQjCNHCJm5rnsywES2mSmRVZrlyDB3Ebw&sig2=ovceQ1KnEIWKo7364lLhUg) over HTTP/2.  Some people will prefer the simplicity of a raw TCP socket, others will appreciate the optional TLS encyption that gRPC brings.  
-**Session Initiation:** There are two options for initiating a telemetry session.  The router can "dial-out" to the collector or the collector can "dial-in" to the router.  Regardless of which side initiates the session, the router always streams the data to the collector at the requested intervals. TCP supports "dial-out" while gRPC supports both "dial-in" and "dial-out."  
-**Encoding:** The router can deliver telemetry data in two different flavors of Google Protocol Buffers: [Compact and Self-Describing GPB](http://blogs.cisco.com/sp/streaming-telemetry-with-google-protocol-buffers).  Compact GPB is the most efficient encoding but requires a unique .proto for each YANG model that is streamed.  Self-describing GPB is less efficient but it uses a single .proto file to decode all YANG models because the keys are passed as strings in the .proto.  

### Using TCP Dial-Out
With the TCP Dial-Out method, the router initiates a TCP session to the collector and sends whatever data is specified by the sensor-group in the subscription.
 
#### TCP Dial-Out Router Config
There are three steps to configuring the router for telemetry with TCP dial-out:
create a destination-group
create a sensor-group
create a subscription
 
##### Step 1: Create a destination-group
The destination-group specifies the destination address, port, encoding and transport that the router should use to send out telemetry data.  In this case, we configure the router to send telemetry via tcp, encoding as self-describing gpb, to 172.30.8.4 port 5432.
RP/0/RP0/CPU0:SunC(config)#telemetry model-driven
RP/0/RP0/CPU0:SunC(config-model-driven)# destination-group 1
RP/0/RP0/CPU0:SunC(config-model-driven-dest)#  address family ipv4 172.30.8.4 port 5432
RP/0/RP0/CPU0:SunC(config-model-driven-dest-addr)#   encoding self-describing-gpb
RP/0/RP0/CPU0:SunC(config-model-driven-dest-addr)#   protocol tcp
RP/0/RP0/CPU0:SunC(config-model-driven-dest-addr)# commit
 
##### Step 2: Create a sensor-group
The sensor-group specifies a list of YANG models which are to be streamed.  The sensor path below represents the YANG model for interfaces statistics:
RP/0/RP0/CPU0:SunC(config)#telemetry model-driven 
RP/0/RP0/CPU0:SunC(config-mdt)# sensor-group 1
RP/0/RP0/CPU0:SunC(config-mdt-sensor-group)#  sensor-path Cisco-IOS-XR-infra-statsd-oper:infra-statistics/interfaces/interface/latest/generic-counters
RP/0/RP0/CPU0:SunC(config-mdt-sensor-group)# commit
 _
##### Step 3: Create a subscription
The subscription associates a destination-group with a sensor-group and sets the streaming interval.  The following configuration associates the sensor-group and destination created above with a streaming interval of 30 seconds.  
RP/0/RP0/CPU0:SunC(config)telemetry model-driven
RP/0/RP0/CPU0:SunC(config-mdt)# subscription 1
RP/0/RP0/CPU0:SunC(config-mdt-subscription)#  sensor-group-id 1 sample-interval 30000
RP/0/RP0/CPU0:SunC(config-mdt-subscription)#  destination-id 1
RP/0/RP0/CPU0:SunC(config-mdt-subscription)# commit
 
#### Validation
Use the following command to verify that you have correctly configured the router for TCP dial-out.
RP/0/RP0/CPU0:SunC#show telemetry model-driven subscription
Thu Jun 16 22:20:41.949 UTC
Subscription:  1                        State: ACTIVE
-------------
  Sensor groups:
  Id                Interval(ms)        State
  1                 10000               Resolved
 
  Destination Groups:
  Id                Encoding            Transport   State   Port    IP
  1                 self-describing-gpb tcp         Active  5432    172.30.8.4
 
RP/0/RP0/CPU0:SunC#


