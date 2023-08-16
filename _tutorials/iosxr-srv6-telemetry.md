---
published: true
date: '2023-08-16 15:03 +0200'
title: Let's get started with SRv6 Telemetry
author: Frederic Cuiller
tags:
  - iosxr
  - SRv6
  - Telemetry
position: hidden
excerpt: >-
  In this article, let’s look at some interesting data to stream using
  Model-Driven Telemetry in the context of SRv6.
---
# Introduction

In this article, let’s look at some of the data that I found interesting to stream using Model-Driven Telemetry at the beginning of my SRv6 journey.

# Leveraging Streaming Telemetry for SRv6

The networks of today widely differs between the domains. The end-to-end provisioning is lengthy & complex because of different management teams, manual operations, and heterogeneous underlay/overlay networks. That’s why Segment Routing v6 has been introduced, to provide a scalable, simple, and unified technology for the transport, services and programmability across the Datacenter, Core and Metro domains. It is fully standardized, it solves many design and scalability issues.  

For a customer training, I had to migrate my lab from SR-MPLS to SRv6 uSID architecture. Thanks to Model-Driven Telemetry and the vastness of IOS-XR operational data models, following concerns can be adressed:

##### Provides real-time visibility about SRv6 usage

SRv6 YANG models are ready to use for data collection. It allows us to collect SRv6 statistics about our network. Further details will be provided later in this article.

##### Successful migration

In general, one of the most crucial tasks before and after a migration is to do “pre/post sanity checks”. This means collecting all the necessary and relevant KPIs by executing show commands on all the devices.  

Our use case involves transitioning from SR-MPLS to SRv6. It is a major mind shift and it relates to the transport network. This is not a routine task, but a critical one that demands a certain level of trust. Telemetry can make things easier, by collecting and streaming the necessary data to improve its usability over time.  

The following KPIs can be gathered:
- ISISv4/v6 neighbors
- ISISv4/v6 routes
- BGP VPNv4/v6 sessions
- L2VPN stats
- Hardware stats
- Interface traffic

##### Accurate network planning

Capacity planning can be a challenging task. One way to achieve this is by tracking statistics for your devices over several months or years. Telemetry can also help to be proactive in identifying scale issues that may have a business impact.

##### Better understanding of hardware resources mapping

Determining the NP utilization for a service or the number of entries consumed during the setup of a L3VPN service can be difficult. Getting visibility at hardware level is critical to better handle scale constraints.

# XR CLI command to YANG Model

# YANG data modeling

# Verifying the streamed data

# Conclusion

# Additional Resources


