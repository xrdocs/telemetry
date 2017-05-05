---
published: false
date: '2017-05-05 14:39 -0600'
title: 'MDT with gRPC: Transport Tricks'
author: Shelly Cadora
---
## gRPC

In previous tutorials, I've covered how to configure a router for Model-Driven Telemetry (MDT) with gRPC [dial-out](https://xrdocs.github.io/telemetry/tutorials/2016-07-21-configuring-model-driven-telemetry-mdt/#grpc-dial-out) and [dial-in](https://xrdocs.github.io/telemetry/tutorials/2016-07-21-configuring-model-driven-telemetry-mdt/#grpc-dial-in).  In this tutorial, I'll discuss some advanced topics and gotchas that you might encounter as you work with gRPC.

Note that you may not need to read this blog!  It is entirely likely that your gRPC connection will "just work."  But for the few corner cases where it doesn't, you might have to read this.  For example, if you can get TCP dial-out to work but not gRPC dial-out, keep reading...

## What is gRPC again?

(gRPC)[http://www.grpc.io/] is an open source RPC framework that leverages HTTP/2 as a transport.  Compared to simple TCP transport, gRPC brings two important features to MDT: 1) Optional encryption with TLS and 2) Support for "dial-in" (from collector to router).

Now bear with me for a moment, as this next bit get a little complicated.  If you are familiar with the 64 bit (IOS XR software architecture)[https://xrdocs.github.io/application-hosting/blogs/2016-06-28-xr-app-hosting-architecture-quick-look/], you may already be aware that IOS XR runs in a container on top of a Linux kernel.  The gRPC process used by MDT lives in the IOS XR container (it's part of the IOS XR Linux shell) but it is not part of the XR Control Plane proper.  This means that gRPC uses the global-vrf namespace.

