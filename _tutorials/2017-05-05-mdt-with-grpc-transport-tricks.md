---
published: true
date: '2017-05-05 14:39 -0600'
title: 'MDT with gRPC: Transport Tricks'
author: Shelly Cadora
excerpt: >-
  Describes issues you may encounter when using MDT with gRPC over
  non-management interfaces.
position: hidden
---
## gRPC

In previous tutorials, I've covered how to configure a router for Model-Driven Telemetry (MDT) with gRPC [dial-out](https://xrdocs.github.io/telemetry/tutorials/2016-07-21-configuring-model-driven-telemetry-mdt/#grpc-dial-out) and [dial-in](https://xrdocs.github.io/telemetry/tutorials/2016-07-21-configuring-model-driven-telemetry-mdt/#grpc-dial-in).  In this tutorial, I'll discuss some advanced topics and gotchas that you might encounter as you work with gRPC.

Note that you may not need to read this blog!  It is entirely likely that your gRPC connection will "just work."  But for the few corner cases where it doesn't, you might have to read this.  For example, if you can get TCP dial-out to work but not gRPC dial-out, keep reading...

## What is gRPC again?

(gRPC)[http://www.grpc.io/] is an open source RPC framework that leverages HTTP/2 as a transport.  Compared to simple TCP transport, gRPC brings two important features to MDT: 1) Optional encryption with TLS and 2) Support for "dial-in" (from collector to router).

Now bear with me for a moment, as this next bit get a little complicated.  If you are familiar with the 64 bit (IOS XR software architecture)[https://xrdocs.github.io/application-hosting/blogs/2016-06-28-xr-app-hosting-architecture-quick-look/], you may already be aware that IOS XR runs in a container on top of a Linux kernel.  The gRPC server used by MDT lives in the IOS XR container (it's part of the IOS XR Linux shell) but it is not part of the XR Control Plane proper.  This means that gRPC uses the Linux networking stack (not the XR networking stack).  And this is where problems can happen.

## What Does gRPC see?

To see the world from gRPC's perspective, drop into the XR Linux shell and take a look at the routes there:

```
RP/0/RP0/CPU0:SunC#bash
Fri May  5 21:22:04.749 UTC

[xr-vm_node0_RP0_CPU0:~]$netstat -r
Kernel IP routing table
Destination     Gateway         Genmask         Flags   MSS Window  irtt Iface
10.30.111.0     *               255.255.255.0   U         0 0          0 Mg0_RP0_CPU0_0
[xr-vm_node0_RP0_CPU0:~]$
```

From this output, you can see that there is only a single route out the management interface.  If your gRPC collector lives on that subnet, the gRPC process will be able to find it.  So that's one reason some lucky people don't have to read this blog.  But if your collector is reachable through some other port, gRPC doesn't know how to get there.  One symptom is that your subscription will stay in the dreaded "Not Active" state:

```
RP/0/RP0/CPU0:SunC#show telem model destination DGroup1 | include State
Fri May  5 22:16:48.995 UTC
    State:                Not Active
```

If you are doing gRPC dial-out, you will see this trace:
```
RP/0/RP0/CPU0:SunC#show grpc trace ems
Fri May  5 21:36:58.868 UTC
May  5 21:36:57.774 ems/grpc 0/RP0/CPU0 t19523 EMS-GRPC: grpc: Conn.resetTransport failed to create client transport: connection error: desc = "transport: dial tcp 172.30.8.4:5432: connect: network is unreachable"; Reconnecting to "172.30.8.4:5432"
May  5 21:38:01.626 ems/grpc 0/RP0/CPU0 t13628 EMS-GRPC: Failed to dial 172.30.8.4:5432: grpc: timed out trying to connect; please retry.
RP/0/RP0/CPU0:SunC#
```

## Just Tell Me How to Fix It

One way to fix this for both dial-in and dial-out is by configuring a Third-Party App (TPA) source address.  (Configuring the TPA sets a src-hint)[https://xrdocs.github.io/application-hosting/tutorials/2016-06-16-xr-toolbox-part-4-bring-your-own-container-lxc-app/#set-the-src-hint-for-application-traffic] for Linux applications, so that originating traffic from the applications can be tied to any reachable IP of XR.

```
RP/0/RP0/CPU0:SunC(config)#tpa address-family ipv4 update-source GigabitEthernet 0/0/0/0
```

By doing this, we automatically get a default route in the Linux shell (the fwdintf takes the traffic back to XR for routing):

```
[xr-vm_node0_RP0_CPU0:~]$netstat -r
Kernel IP routing table
Destination     Gateway         Genmask         Flags   MSS Window  irtt Iface
default         *               0.0.0.0         U         0 0          0 fwdintf
10.30.111.0     *               255.255.255.0   U         0 0          0 Mg0_RP0_CPU0_0
[xr-vm_node0_RP0_CPU0:~]$
```

So now, gRPC has a route back to the collector.  That's all you need for dial-in.

For dial-out knows to use GigabitEthernet0/0/0/0 as a source address. You can see that here:

```
[xr-vm_node0_RP0_CPU0:~]$ip route
default dev fwdintf  scope link  src 172.30.8.53
10.30.111.0/24 dev Mg0_RP0_CPU0_0  proto kernel  scope link  src 10.30.111.9
[xr-vm_node0_RP0_CPU0:~]$
```

See that "src 172.30.8.53" ?  That's the source address that gRPC will use when sending MDT traffic in dial-out mode.

## I Didn't Configure TPA But It Still Works, So There!

So some lucky people who didn't configure TPA can still get gRPC to work!  Doesn't seem fair, does it?  Well, the reason is that they have a Loopback (any Loopback except Loopback 1 which is reserved -- read (this)[https://xrdocs.github.io/application-hosting/blogs/2016-06-28-xr-app-hosting-architecture-quick-look/] for the gory details) configured.  When a Loopback interface is configured, you also get a default route in the Linux stack:

```
RP/0/RP0/CPU0:SunC(config)#no tpa
RP/0/RP0/CPU0:SunC(config)#int loop0
RP/0/RP0/CPU0:SunC(config-if)#ipv4 address 5.5.5.5/32
RP/0/RP0/CPU0:SunC(config-if)#commit
RP/0/RP0/CPU0:SunC(config-if)#end
RP/0/RP0/CPU0:SunC#bash
Fri May  5 22:01:59.141 UTC

[xr-vm_node0_RP0_CPU0:~]$netstat -r
Kernel IP routing table
Destination     Gateway         Genmask         Flags   MSS Window  irtt Iface
default         *               0.0.0.0         U         0 0          0 fwdintf
10.30.111.0     *               255.255.255.0   U         0 0          0 Mg0_RP0_CPU0_0

```

That's good for dial-in, but what about dial-out?  Well, it still *might* work. Check out the src address below:

```
[xr-vm_node0_RP0_CPU0:~]$ip route
default dev fwdintf  scope link  src 5.5.5.5
10.30.111.0/24 dev Mg0_RP0_CPU0_0  proto kernel  scope link  src 10.30.111.9
[xr-vm_node0_RP0_CPU0:~]$
```

Traffic sent to the collector will have a source address of 5.5.5.5.  If your collector has a route back to 5.5.5.5 (e.g. you're distributing your loopback addresses in your IGP), then great.  If not, then the collector will drop the packet and you'll need the TPA config.

## Conclusion

I hope you didn't have to read this tutorial at all.  But if you did and even if you glazed over the bits about the Linux networking stack and XR Linux shell, just remember this: either configure a TPA source address or a routable Loopback and gRPC will work.