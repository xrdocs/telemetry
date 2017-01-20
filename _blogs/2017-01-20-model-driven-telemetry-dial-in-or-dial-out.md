---
published: true
date: '2017-01-20 09:35 -0700'
title: 'Model-Driven Telemetry: Dial-In or Dial-Out ?'
author: Shelly Cadora
excerpt: Discusses the transport options for model-driven telemetry
tags:
  - cisco
  - MDT
position: hidden
---
## Transport Options

In one of my first tutorials on [configuring Model-Driven Telemetry (MDT)](https://xrdocs.github.io/telemetry/tutorials/2016-07-21-configuring-model-driven-telemetry-mdt/), I blithely referred to three options for transport: TCP dial-out, gRPC dial-out and gRPC dial-in.  It's all well and good to know how to configure each one, but what's the difference and which one should you choose?  This blog tackles those questions.

## Dial-Out Vs. Dial-In

When we say "dial-out," we are speaking from the router's perspective: the router "dials out" to the collector.  In other words, the router sends the SYN packet in the TCP handshake.

![DialOut2.png]({{site.baseurl}}/images/DialOut2.png)

Anyone who has had to modify ACLs to enable a new SNMP manager to connect to the network can appreciate the value of the dial-out option.  Since the router initiates the connection, you don't have to worry about opening up ports for inbound management traffic.

With dial-in, on the other hand, the router listens passively on a specified port until the collector "dials-in."  

![DialIn2.png]({{site.baseurl}}/images/DialIn2.png)

After the initial session establishment, the router still pushes the data off the box at the configured interval.  This is very important!  Don't be fooled by the direction of the SYN packet.  There is no polling mechanism in MDT.  

Dial-in appeals to folks who are looking for a "single channel" to communicate with the network.  These are operators who want a single transport and protocol for both configuration data and streaming operational data. Sound impossible?  Well, we're already doing it today.

## TCP Dial-Out
Our first dial-out protocol was also the simplest: plain old TCP.  Open up a raw TCP socket on your collector and the router will complete the standard three-way handshake and start pushing telemetry data across the session.  No fancy programming libraries are required on the collector -- in python it's a simple matter of a "bind" to the port.  TCP dial-out inherits all the goodness of TCP (reliable delivery, fragmentation, re-ordering, etc) without having to invent a new protocol or define new mechanisms.  It's a great place to start if you're configuring MDT for the first time.

## gRPC Dial-Out
One of TCP's great strengths is its simplicity.  But TCP by itself lacks higher-level functions that can enable more secure and sophisticated communication between the router and the collector.  For that, we turned to [gRPC](http://www.grpc.io/).  

gRPC is an open source communication framework built on top of HTTP/2.  It was originally designed by Google to enable efficient, accurate and low-latency communication between clients and servers.  It has many functions beyond those required by MDT.

One of the main reasons that people enable gRPC dial-out is that gRPC allows you to do authentication and encryption via TLS.  If you're worried about sending operational data in the clear and/or you want to protect your collector with certificate-based authentication, enable gRPC with TLS.  

gRPC is not quite as trivial from a developer's perspective, but one of its strengths is the plethora of idiomatic client libraries in multiple programming languages.  Go, Python, Ruby, Java, C developers -- grab your gRPC library from github and you'll be juggling gRPC sessions like a pro.

## gRPC Dial-In
In addition to secure and efficient transport, gRPC provides bidirectional streaming  and connection multiplexing.  This means that you can "dial-in" to a router, push down new configs (including telemetry subscription configs) and have operational data streamed back -- all within a single, unified channel, all using the same underlying data models.  Cisco IOS XR has supported [configuration via gRPC](https://github.com/CiscoDevNet/grpc-getting-started) since 6.0.0 and dial-in telemetry over gRPC since 6.1.1.  

Since the collector "dials-in" to the router, there's no need to specify each MDT destination in the configuration.  In fact, since you can configure MDT via gRPC, you don't have to configure telemetry at all.  Just enable the gRPC service on the router, connect your client, and dynamically enable the telemetry subscription you want.

## Decisions, Decisions
So what transport should you use for MDT?  Here's a few quick heuristics:

- If you're looking for a quick and simple solution, try TCP dial-out.  It's simple to configure, there are no new protocols to learn, and you won't have to worry about opening up inbound connections.  
- If you need encryption, go for gRPC dial-out.  
- If you're already using gRPC for configuration, consider gRPC dial-in.


As you deploy MDT, you may find that your transport needs change or evolve.  No problem.  The most important thing to remember is that the push mechanism for telemetry data remains exactly the same, dial-in or dial-out, TCP or gRPC.  No matter what you choose, you'll get the same data, in the same data model, at the same speed.  That's the beauty of Model-Driven Telemetry.
