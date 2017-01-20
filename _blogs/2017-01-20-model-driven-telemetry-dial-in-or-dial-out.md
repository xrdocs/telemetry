---
published: false
date: '2017-01-20 09:35 -0700'
title: 'Model-Driven Telemetry: Dial-In or Dial-Out ?'
---
## Transport Options

In one of my first tutorials on [configuring Model-Driven Telemetry (MDT)](https://xrdocs.github.io/telemetry/tutorials/2016-07-21-configuring-model-driven-telemetry-mdt/), I somewhat blithely referred to three options for transport: TCP Dial-Out, gRPC Dial-Out and gRPC Dial-In.  It's all well and good to know how to configure each one, but what's the difference and which one should you choose?  This blog tackles those questions.

## Dial-Out

When we say "dial-out" and "dial-in," we are speaking from the router's perspective.  So when you configure dial-out, the router "dials out" to the collector.  In other words, the router sends the SYN packet in the TCP handshake.

![Dial-Out.png]({{site.baseurl}}/images/Dial-Out.png)

Anyone who has had to modify ACLs to enable a new SNMP manager to connect to the network can appreciate the value of the dial-out option.  Since the router initiates the connection, you don't have to worry about opening up ports for inbound management traffic.

Dial-out was the first option that we implemented and is still commonly used.  MDT enables the router to _push_ monitoring data, so it seems intuitive that the router should also initiate the session.

### TCP Dial-Out
Our first dial-out protocol was also the simplest: plain old TCP.  Open up a raw TCP socket on your collector and the router will complete the standard three-way handshake and start pushing telemetry data across the session.  No fancy programming libraries are required on the collector -- in python it's a simple matter of a "bind" to the port -- and you're off and running.  MDT Dial-Out inherits all the goodness of TCP (reliable delivery, fragmentation, re-ordering, etc) without having to invent a new protocol or define new mechanisms.  It's a great place to start if you're configuring MDT for the first time.

### gRPC Dial-Out
TCP is a great transport option, but it is not a general purpose RPC mechanism.  For that, we turned to [gRPC](http://www.grpc.io/).  gRPC runs on top of HTTP/2 and provides additional functionality around security, 

