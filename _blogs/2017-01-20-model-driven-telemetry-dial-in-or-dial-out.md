---
published: true
date: '2017-01-20 09:35 -0700'
title: 'Model-Driven Telemetry: Dial-In or Dial-Out ?'
author: Shelly Cadora
excerpt: Discusses the transport options for model-driven telemetry
tags:
  - cisco
position: hidden
---
## Transport Options

In one of my first tutorials on [configuring Model-Driven Telemetry (MDT)](https://xrdocs.github.io/telemetry/tutorials/2016-07-21-configuring-model-driven-telemetry-mdt/), I blithely referred to three options for transport: TCP Dial-Out, gRPC Dial-Out and gRPC Dial-In.  It's all well and good to know how to configure each one, but what's the difference and which one should you choose?  This blog tackles those questions.

## Dial-Out

When we say "dial-out," we are speaking from the router's perspective.  So when you configure dial-out, the router "dials out" to the collector.  In other words, the router sends the SYN packet in the TCP handshake.

![Dial-Out.png ]({{site.baseurl}}/images/Dial-Out.png =200x){: .align-center}

Anyone who has had to modify ACLs to enable a new SNMP manager to connect to the network can appreciate the value of the dial-out option.  Since the router initiates the connection, you don't have to worry about opening up ports for inbound management traffic.

Dial-out was the first option that we implemented and is still commonly used.  MDT enables the router to _push_ monitoring data, so it seems intuitive that the router should also initiate the session.

### TCP Dial-Out
Our first dial-out protocol was also the simplest: plain old TCP.  Open up a raw TCP socket on your collector and the router will complete the standard three-way handshake and start pushing telemetry data across the session.  No fancy programming libraries are required on the collector -- in python it's a simple matter of a "bind" to the port -- and you're off and running.  MDT Dial-Out inherits all the goodness of TCP (reliable delivery, fragmentation, re-ordering, etc) without having to invent a new protocol or define new mechanisms.  It's a great place to start if you're configuring MDT for the first time.

### gRPC Dial-Out
TCP is a great transport option, but it is not a general purpose RPC mechanism.  For that, we turned to [gRPC](http://www.grpc.io/).  gRPC runs on top of HTTP/2 and provides additional features for performance, security, RPC definition, etc.  

One of the main reasons that people enable gRPC Dial-Out is that gRPC allows you to do authentication and encryption via TLS.  If you're worried about sending operational data in the clear and/or you want to protect your collector with certificate-based authentication, enable gRPC with TLS.  

gRPC is not quite as trivial from a developer's perspective, but one of its strengths is the plethora of idiomatic client libraries in multiple programming languages.  Go, Python, Ruby, Java, C developers -- grab your gRPC library from github and you'll be juggling gRPC sessions like a pro.

## Dial-In
The main difference between dial-in and dial-out is who sends that first SYN packet.  With dial-in, the router listens passively on a specified port until the collector "dials-in."  

![Dial-In.png]({{site.baseurl}}/images/Dial-In.png)

After the initial session establishment, everything is exactly the same.  The router still pushes the data off the box at the configured interval.  This is very important!  Don't be fooled by the direction of the SYN packet.  There is no polling mechanism.  

Dial-in often appeals to folks who are looking for a "single channel" to communicate with the network.  These are operators who want a single transport, protocol and RPC for both configuration data and operational data.  Connect to the device, push down new configs (including telemetry subscription configs) and have operational data streamed back -- all within a single, unified channel, all using the same underlying data models.  Sound impossible?  Well, we're already doing it today.

### gRPC Dial-In
The mechanism that provides single-channel access is gRPC.  In addition to secure and efficient transport, gRPC provides bidirectional streams (so the router can still push data even if the collector dials in) and connection multiplexing.  Cisco IOS XR has supported [configuration via gRPC](https://github.com/CiscoDevNet/grpc-getting-started) since 6.0.0 and telemetry over gRPC since 6.1.1.  

Since the collector "dials-in" to the router, there's no need to specify each MDT destination in the configuration.  In fact, since you can configure MDT via gRPC, you don't have to configure telemetry at all.  Just enable the gRPC service on the router, connect your client, and dynamically enable the telemetry subscription you want.

## Decisions, Decisions
So what transport should you use for MDT?  Here's a few quick heuristics:
-If you're looking for a quick and simple solution, try TCP dial-out.  It's simple to configure, there are no new protocols to learn, and you won't have to worry about opening up inbound connections.  
-If you need encryption, go for gRPC Dial-out.  
-If you're already using gRPC for configuration, consider gRPC Dial-In. 

As you deploy MDT, you may find that your transport needs change or evolve.  No problem.  The most important thing to remember is that the push mechanism for telemetry data remains exactly the same, dial-in or dial-out, TCP or gRPC.  No matter what you choose, you'll get the same data, in the same data model, at the same speed.  That's the beauty of Model-Driven Telemetry.
