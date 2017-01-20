---
published: false
date: '2017-01-20 09:35 -0700'
title: 'Model-Driven Telemetry: Dial-In or Dial-Out ?'
---
## Transport Options

In one of my first tutorials on [configuring Model-Driven Telemetry](https://xrdocs.github.io/telemetry/tutorials/2016-07-21-configuring-model-driven-telemetry-mdt/), I somewhat blithely referred to three options for transport: TCP Dial-Out, gRPC Dial-Out and gRPC Dial-In.  It's all well and good to know how to configure each one, but what's the difference and which one should you choose?  This blog tackles those questions.

## Dial-Out

The terms "dial-out" and "dial-in" are both meant from the perspective of the router.  So when you configure dial-out, the router "dials out" to the collector.  In other words, the router sends the SYN packet in the TCP handshake
