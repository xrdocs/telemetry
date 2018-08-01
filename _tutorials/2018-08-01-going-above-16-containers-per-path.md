---
published: true
date: '2018-08-01 11:17 -0700'
title: Going Above 16 Containers Per Path
author: Viktor Osipchuk
excerpt: >-
  This post covers how you can remove the internal check of the containers
  number in the sensor paths that you configure in IOS XR Telemetry 
tags:
  - iosxr
  - cisco
  - Telemetry
  - MDT
---
{% include toc icon="table" title="Going Above 16 Containers Per Path" %}
{% include base_path %}

Imagine a situation when you want to get many counters from some specific sensor path, but, somehow, you don't see anything on your collector because the router doesn't push counters out. Everything seems to be resolved, but you don't like a line in the show output that tells you that there are too many containers. 
This short post will lead you through this situation as well as explains what you can do to have your data pushed out. 

## A Short Intro to Sensor Paths Resolution in IOS XR

As you know, IOS XR resolves YANG models down to the container level. It means that you can specify your exact ("bottom") container or you can specify a less strict sensor path, and the router should resolve down to all the containers after you apply the configuration. 

Let's see an example of how it works before moving forward. 

Throughout this post, we will work with the ["Cisco-IOS-XR-bundlemgr-oper.yang"](https://github.com/YangModels/yang/blob/master/vendor/cisco/xr/632/Cisco-IOS-XR-bundlemgr-oper.yang) yang model (just a random model). 

Let's say that you're interested in this specific path: *Cisco-IOS-XR-bundlemgr-oper:bundle-information/bfd-counters/bfd-counters-bundles/bfd-counters-bundle/bfd-counters-bundle-children-members/bfd-counters-bundle-children-member*. This path contains these counters:

<div class="highlighter-rouge">
<pre class="highlight">
<code>
VOSIPCHU-M-C1GV:632 vosipchu$ pyang -f tree Cisco-IOS-XR-bundlemgr-oper.yang --tree-path bundle-information/bfd-counters/bfd-counters-bundles/bfd-counters-bundle/bfd-counters-bundle-children-members/bfd-counters-bundle-children-member
module: Cisco-IOS-XR-bundlemgr-oper
   +--ro bundle-information
      +--ro bfd-counters
         +--ro bfd-counters-bundles
            +--ro bfd-counters-bundle* [bundle-interface]
               +--ro bfd-counters-bundle-children-members
                  +--ro bfd-counters-bundle-children-member* [member-interface]
                     +--ro member-interface                  string
                     +--ro member-name?                      string
                     +--ro last-time-cleared?                uint64
                     +--ro starting?                         uint32
                     +--ro up?                               uint32
                     +--ro down?                             uint32
                     +--ro neighbor-unconfigured?            uint32
                     +--ro start-timeouts?                   uint32
                     +--ro neighbor-unconfigured-timeouts?   uint32
                     +--ro time-since-cleared?               uint64
</code>
</pre>
</div>

If you try to apply this configuration on your router, you will get exactly that path resolved:

```
RP/0/RP0/CPU0:NCS5501_top(config)#sh conf
Wed Aug  1 08:46:27.232 PDT
Building configuration...
!! IOS XR Configuration version = 6.3.2
telemetry model-driven
 sensor-group bundles-single
  sensor-path Cisco-IOS-XR-bundlemgr-oper:bundle-information/bfd-counters/bfd-counters-bundles/bfd-counters-bundle/bfd-counters-bundle-children-members/bfd-counters-bundle-children-member
 !
!
end

RP/0/RP0/CPU0:NCS5501_top(config)#commit 
Wed Aug  1 08:46:29.805 PDT
RP/0/RP0/CPU0:NCS5501_top(config)#
RP/0/RP0/CPU0:NCS5501_top#sh tele model-driven sensor-group bundles-single internal 
Wed Aug  1 08:46:42.638 PDT
  Sensor Group Id:bundles-single
    Sensor Path:        Cisco-IOS-XR-bundlemgr-oper:bundle-information/bfd-counters/bfd-counters-bundles/bfd-counters-bundle/bfd-counters-bundle-children-members/bfd-counters-bundle-children-member
    Sensor Path State:  Resolved
      Sysdb Path:       /oper/bmd/gl/bfd-counts-mbr/bundle/<bundlemgr_bmd_oper_BFDCountersBundle_bundle>/children/<bundlemgr_bmd_oper_BFDCountersBundleChildrenMember_member>
       Yang Path:       Cisco-IOS-XR-bundlemgr-oper:bundle-information/bfd-counters/bfd-counters-bundles/bfd-counters-bundle/bfd-counters-bundle-children-members/bfd-counters-bundle-children-member
```

If you decide to specify a more generic path, say, *Cisco-IOS-XR-bundlemgr-oper:bundle-information/bfd-counters/bfd-counters-bundles*, then, it will be mapped to all containers within. 
Here is how it looks through pyang:

<div class="highlighter-rouge">
<pre class="highlight">
<code>
VOSIPCHU-M-C1GV:632 vosipchu$ pyang -f tree Cisco-IOS-XR-bundlemgr-oper.yang --tree-path bundle-information/bfd-counters/bfd-counters-bundles
module: Cisco-IOS-XR-bundlemgr-oper
   +--ro bundle-information
      +--ro bfd-counters
         +--ro bfd-counters-bundles
            +--ro bfd-counters-bundle* [bundle-interface]
               +--ro bfd-counters-bundle-descendant
               |  +--ro bundle-name
               |  |  +--ro item-name?   string
               |  +--ro bfd-counter*
               |     +--ro member-name?                      string
               |     +--ro last-time-cleared?                uint64
               |     +--ro starting?                         uint32
               |     +--ro up?                               uint32
               |     +--ro down?                             uint32
               |     +--ro neighbor-unconfigured?            uint32
               |     +--ro start-timeouts?                   uint32
               |     +--ro neighbor-unconfigured-timeouts?   uint32
               |     +--ro time-since-cleared?               uint64
               +--ro bfd-counters-bundle-children-members
               |  +--ro bfd-counters-bundle-children-member* [member-interface]
               |     +--ro member-interface                  string
               |     +--ro member-name?                      string
               |     +--ro last-time-cleared?                uint64
               |     +--ro starting?                         uint32
               |     +--ro up?                               uint32
               |     +--ro down?                             uint32
               |     +--ro neighbor-unconfigured?            uint32
               |     +--ro start-timeouts?                   uint32
               |     +--ro neighbor-unconfigured-timeouts?   uint32
               |     +--ro time-since-cleared?               uint64
               +--ro bfd-counters-bundle-item
               |  +--ro item-name?   string
               +--ro bundle-interface                        xr:Interface-name
</code>
</pre>
</div>

To see the containers, it might be easier to [look at the model in the native format](https://github.com/YangModels/yang/blob/master/vendor/cisco/xr/632/Cisco-IOS-XR-bundlemgr-oper.yang#L99-L142):

<div class="highlighter-rouge">
<pre class="highlight">
<code>
container bfd-counters-bundles {
        description
          "Bundle interfaces with BFD counters information";
        list bfd-counters-bundle {
          key "bundle-interface";
          description
            "Bundle interface";
          <span style="color:magenta">container bfd-counters-bundle-descendant {</span>
            description
              "Data for this item and all its members";
            uses BMD-BFD-COUNTER-BAG-MULTIPLE;
          }
          <span style="color:magenta">container bfd-counters-bundle-children-members {</span>
            description
              "Children of bundle with BFD counters
               information";
            list bfd-counters-bundle-children-member {
              key "member-interface";
              description
                "Bundle member item with BFD counters
                 information";
              leaf member-interface {
                type string;
                description
                  "Member interface";
              }
              uses BMD-BFD-COUNTER-BAG;
            }
          }
          <span style="color:magenta">container bfd-counters-bundle-item {</span>
            description
              "Data for this item";
            uses BM-NAME-BAG;
          }
          leaf bundle-interface {
            type xr:Interface-name;
            description
              "Bundle interface";
          }
        }
     }
</code>
</pre>
</div>

It is easy to see that our generic sensor path is mapped to three containers. Let's check how it looks like in IOS XR:

```
RP/0/RP0/CPU0:NCS5501_top(config)#sh conf
Wed Aug  1 09:07:48.242 PDT
Building configuration...
!! IOS XR Configuration version = 6.3.2
telemetry model-driven
 sensor-group bundles-three
  sensor-path Cisco-IOS-XR-bundlemgr-oper:bundle-information/bfd-counters/bfd-counters-bundles
 !
!
end

RP/0/RP0/CPU0:NCS5501_top(config)#commit
Wed Aug  1 09:07:49.827 PDT
RP/0/RP0/CPU0:NCS5501_top(config)#
RP/0/RP0/CPU0:NCS5501_top#sh tele m sensor-group bundles-three internal 
Wed Aug  1 09:08:05.145 PDT
  Sensor Group Id:bundles-three
    Sensor Path:        Cisco-IOS-XR-bundlemgr-oper:bundle-information/bfd-counters/bfd-counters-bundles
    Sensor Path State:  Resolved
      Sysdb Path:       /oper/bmd/gl/bfd-counts-mbr/bundle/<bundlemgr_bmd_oper_BFDCountersBundle_bundle>/descendant_data
        Yang Path:       Cisco-IOS-XR-bundlemgr-oper:bundle-information/bfd-counters/bfd-counters-bundles/bfd-counters-bundle/bfd-counters-bundle-descendant
      Sysdb Path:       /oper/bmd/gl/bfd-counts-mbr/bundle/<bundlemgr_bmd_oper_BFDCountersBundle_bundle>/children/<bundlemgr_bmd_oper_BFDCountersBundleChildrenMember_member>
        Yang Path:       Cisco-IOS-XR-bundlemgr-oper:bundle-information/bfd-counters/bfd-counters-bundles/bfd-counters-bundle/bfd-counters-bundle-children-members/bfd-counters-bundle-children-member
      Sysdb Path:       /oper/bmd/gl/bfd-counts-mbr/bundle/<bundlemgr_bmd_oper_BFDCountersBundle_bundle>/item_data
        Yang Path:       Cisco-IOS-XR-bundlemgr-oper:bundle-information/bfd-counters/bfd-counters-bundles/bfd-counters-bundle/bfd-counters-bundle-item
```

And confirmed, our generic sensor path was resolved to three different YANG paths. Those resolved YANG paths are exactly the containers we saw before.

## Default behavior

Let's imagine a situation when you want to stream many counters from many different containers within a YANG model. Just because you need this or you want to check all the counters to select a set of containers you want to leave in your configuration. 
What will you do? Right, you will configure a generic sensor path, as before. As an example, you want to configure this path: *Cisco-IOS-XR-bundlemgr-oper:bundle-information*. 

Here is how it will look like on the router:

```
RP/0/RP0/CPU0:NCS5501_top(config)#sh conf
Wed Aug  1 09:14:49.595 PDT
Building configuration...
!! IOS XR Configuration version = 6.3.2
telemetry model-driven
 sensor-group bundles
  sensor-path Cisco-IOS-XR-bundlemgr-oper:bundle-information
 !
!
end

RP/0/RP0/CPU0:NCS5501_top(config)#
RP/0/RP0/CPU0:NCS5501_top(config)#
RP/0/RP0/CPU0:NCS5501_top(config)#
RP/0/RP0/CPU0:NCS5501_top(config)#commit
Wed Aug  1 09:14:52.939 PDT
RP/0/RP0/CPU0:NCS5501_top(config)#
RP/0/RP0/CPU0:NCS5501_top#sh tele m sensor-group bundles internal 
Wed Aug  1 09:15:00.779 PDT
  Sensor Group Id:bundles
    Sensor Path:        Cisco-IOS-XR-bundlemgr-oper:bundle-information
    Sensor Path State:  Not Resolved
      Status:           Resolved to too many containers, max : 16
      Sysdb Path:       /oper/bmd/gl/bfd-counts-mbr/bundle/<bundlemgr_bmd_oper_BFDCountersBundle_bundle>/descendant_data
       Yang Path:       Cisco-IOS-XR-bundlemgr-oper:bundle-information/bfd-counters/bfd-counters-bundles/bfd-counters-bundle/bfd-counters-bundle-descendant
      Sysdb Path:       /oper/bmd/gl/bfd-counts-mbr/bundle/<bundlemgr_bmd_oper_BFDCountersBundle_bundle>/children/<bundlemgr_bmd_oper_BFDCountersBundleChildrenMember_member>
       Yang Path:       Cisco-IOS-XR-bundlemgr-oper:bundle-information/bfd-counters/bfd-counters-bundles/bfd-counters-bundle/bfd-counters-bundle-children-members/bfd-counters-bundle-children-member
      Sysdb Path:       /oper/bmd/gl/bfd-counts-mbr/bundle/<bundlemgr_bmd_oper_BFDCountersBundle_bundle>/item_data
       Yang Path:       Cisco-IOS-XR-bundlemgr-oper:bundle-information/bfd-counters/bfd-counters-bundles/bfd-counters-bundle/bfd-counters-bundle-item
      Sysdb Path:       /oper/bmd/gl/bfd-counts-mbr/member/<bundlemgr_bmd_oper_BFDCountersMember_member>/item_data
       Yang Path:       Cisco-IOS-XR-bundlemgr-oper:bundle-information/bfd-counters/bfd-counters-members/bfd-counters-member/bfd-counters-member-item
      Sysdb Path:       /oper/bmd/gl/scheduled-actions/bundle/<bundlemgr_bmd_oper_ScheduledActionsBundle_bundle>/item_data
       Yang Path:       Cisco-IOS-XR-bundlemgr-oper:bundle-information/scheduled-actions/scheduled-actions-bundles/scheduled-actions-bundle/scheduled-actions-bundle-item
      Sysdb Path:       /oper/bmd/gl/bundle/bundle/<bundlemgr_bmd_oper_BundleBundle_bundle>/descendant_data
       Yang Path:       Cisco-IOS-XR-bundlemgr-oper:bundle-information/bundle/bundle-bundles/bundle-bundle/bundle-bundle-descendant
      Sysdb Path:       /oper/bmd/gl/bundle/bundle/<bundlemgr_bmd_oper_BundleBundle_bundle>/children/<bundlemgr_bmd_oper_BundleBundleChildrenMember_member>
       Yang Path:       Cisco-IOS-XR-bundlemgr-oper:bundle-information/bundle/bundle-bundles/bundle-bundle/bundle-bundle-children-members/bundle-bundle-children-member
      Sysdb Path:       /oper/bmd/gl/bundle/bundle/<bundlemgr_bmd_oper_BundleBundle_bundle>/item_data
       Yang Path:       Cisco-IOS-XR-bundlemgr-oper:bundle-information/bundle/bundle-bundles/bundle-bundle/bundle-bundle-item
      Sysdb Path:       /oper/bmd/gl/bundle/member/<bundlemgr_bmd_oper_BundleMember_member>/ancestor_data
       Yang Path:       Cisco-IOS-XR-bundlemgr-oper:bundle-information/bundle/bundle-members/bundle-member/bundle-member-ancestor
      Sysdb Path:       /oper/bmd/gl/bundle/member/<bundlemgr_bmd_oper_BundleMember_member>/item_data
       Yang Path:       Cisco-IOS-XR-bundlemgr-oper:bundle-information/bundle/bundle-members/bundle-member/bundle-member-item
      Sysdb Path:       /oper/bmd/gl/events-rg/member/<bundlemgr_bmd_oper_EventsRGMember_member>/ancestor_data
       Yang Path:       Cisco-IOS-XR-bundlemgr-oper:bundle-information/events-rg/events-rg-members/events-rg-member/events-rg-member-ancestor
      Sysdb Path:       /oper/bmd/gl/events-rg/rg/<bundlemgr_bmd_oper_EventsRG_ICCPGroup_ig>/item_data
       Yang Path:       Cisco-IOS-XR-bundlemgr-oper:bundle-information/events-rg/events-rg-iccp-groups/events-rg-iccp-group/events-rg-bundle-item-iccp-group
      Sysdb Path:       /oper/bmd/gl/events-rg/bundle/<bundlemgr_bmd_oper_EventsRGBundle_bundle>/ancestor_data
       Yang Path:       Cisco-IOS-XR-bundlemgr-oper:bundle-information/events-rg/events-rg-bundles/events-rg-bundle/events-rg-bundle-ancestor
      Sysdb Path:       /oper/bmd/gl/lacp/bundle/<bundlemgr_bmd_oper_LACPBundle_bundle>/item_data
       Yang Path:       Cisco-IOS-XR-bundlemgr-oper:bundle-information/lacp/lacp-bundles/lacp-bundle/lacp-bundle-item
      Sysdb Path:       /oper/bmd/gl/lacp/bundle/<bundlemgr_bmd_oper_LACPBundle_bundle>/descendant_data
       Yang Path:       Cisco-IOS-XR-bundlemgr-oper:bundle-information/lacp/lacp-bundles/lacp-bundle/lacp-bundle-descendant
      Sysdb Path:       /oper/bmd/gl/lacp/bundle/<bundlemgr_bmd_oper_LACPBundle_bundle>/children/<bundlemgr_bmd_oper_LACPBundleChildrenMember_member>
       Yang Path:       Cisco-IOS-XR-bundlemgr-oper:bundle-information/lacp/lacp-bundles/lacp-bundle/lacp-bundle-children-members/lacp-bundle-children-member
      Sysdb Path:       /oper/bmd/gl/lacp/member/<bundlemgr_bmd_oper_LACPMember_member>/ancestor_data
       Yang Path:       Cisco-IOS-XR-bundlemgr-oper:bundle-information/lacp/lacp-members/lacp-member/lacp-member-ancestor
      Sysdb Path:       /oper/bmd/gl/lacp/member/<bundlemgr_bmd_oper_LACPMember_member>/item_data
       Yang Path:       Cisco-IOS-XR-bundlemgr-oper:bundle-information/lacp/lacp-members/lacp-member/lacp-member-item
      Sysdb Path:       /oper/bmd/gl/mlacp-counters-bdl/rg/<bundlemgr_mlacp_bdl_ctrs_oper_ICCPGroup_ig>/item_data
       Yang Path:       Cisco-IOS-XR-bundlemgr-oper:bundle-information/mlacp-bundle-counters/iccp-groups/iccp-group/iccp-group-item
      Sysdb Path:       /oper/bmd/gl/mlacp-counters-bdl/bundle/<bundlemgr_mlacp_bdl_ctrs_oper_Bundle_bundle>/item_data
       Yang Path:       Cisco-IOS-XR-bundlemgr-oper:bundle-information/mlacp-bundle-counters/bundles/bundle/bundle-item
      Sysdb Path:       /oper/bmd/gl/mlacp-counters-bdl/node/<bundlemgr_mlacp_bdl_ctrs_oper_Node_node>/item_data
       Yang Path:       Cisco-IOS-XR-bundlemgr-oper:bundle-information/mlacp-bundle-counters/nodes/node/node-item
      Sysdb Path:       /oper/bmd/gl/protect/bundle/<bundlemgr_bmd_oper_ProtectBundle_bundle>/item_data
       Yang Path:       Cisco-IOS-XR-bundlemgr-oper:bundle-information/protect/protect-bundles/protect-bundle/protect-bundle-item
      Sysdb Path:       /oper/bmd/gl/mlacp-brief/bundle/<bundlemgr_bmd_oper_mLACPBundleBrief_bundle>/item_data
       Yang Path:       Cisco-IOS-XR-bundlemgr-oper:bundle-information/mlacp-brief/mlacp-bundle-briefs/mlacp-bundle-brief/mlacp-bundle-item-brief
      Sysdb Path:       /oper/bmd/gl/mlacp-brief/rg/<bundlemgr_bmd_oper_mLACPBrief_ICCPGroup_ig>/item_data
       Yang Path:       Cisco-IOS-XR-bundlemgr-oper:bundle-information/mlacp-brief/mlacp-brief-iccp-groups/mlacp-brief-iccp-group/mlacp-brief-iccp-group-item
      Sysdb Path:       /oper/bmd/gl/mlacp/bundle/<bundlemgr_bmd_oper_mLACPBundle_bundle>/item_data
       Yang Path:       Cisco-IOS-XR-bundlemgr-oper:bundle-information/mlacp/mlacp-bundles/mlacp-bundle/mlacp-bundle-item
      Sysdb Path:       /oper/bmd/gl/mlacp/rg/<bundlemgr_bmd_oper_mLACP_ICCPGroup_ig>/item_data
       Yang Path:       Cisco-IOS-XR-bundlemgr-oper:bundle-information/mlacp/mlacp-iccp-groups/mlacp-iccp-group/mlacp-iccp-group-item
      Sysdb Path:       /oper/bmd/gl/mac-allocation/global/item_data
       Yang Path:       Cisco-IOS-XR-bundlemgr-oper:bundle-information/mac-allocation/mac-allocation-global/mac-allocation-global-item
      Sysdb Path:       /oper/bmd/gl/events/member/<bundlemgr_bmd_oper_EventsMember_member>/item_data
       Yang Path:       Cisco-IOS-XR-bundlemgr-oper:bundle-information/events/events-members/events-member/events-member-item
      Sysdb Path:       /oper/bmd/gl/events/member/<bundlemgr_bmd_oper_EventsMember_member>/ancestor_data
       Yang Path:       Cisco-IOS-XR-bundlemgr-oper:bundle-information/events/events-members/events-member/events-member-ancestor
      Sysdb Path:       /oper/bmd/gl/events/bundle/<bundlemgr_bmd_oper_EventsBundle_bundle>/ancestor_data
       Yang Path:       Cisco-IOS-XR-bundlemgr-oper:bundle-information/events/events-bundles/events-bundle/events-bundle-ancestor
      Sysdb Path:       /oper/bmd/gl/events/bundle/<bundlemgr_bmd_oper_EventsBundle_bundle>/item_data
       Yang Path:       Cisco-IOS-XR-bundlemgr-oper:bundle-information/events/events-bundles/events-bundle/events-bundle-item
      Sysdb Path:       /oper/bmd/gl/events/bundle/<bundlemgr_bmd_oper_EventsBundle_bundle>/descendant_data
       Yang Path:       Cisco-IOS-XR-bundlemgr-oper:bundle-information/events/events-bundles/events-bundle/events-bundle-descendant
      Sysdb Path:       /oper/bmd/gl/events/bundle/<bundlemgr_bmd_oper_EventsBundle_bundle>/children/<bundlemgr_bmd_oper_EventsBundleChildrenMember_member>
       Yang Path:       Cisco-IOS-XR-bundlemgr-oper:bundle-information/events/events-bundles/events-bundle/events-bundle-children-members/events-bundle-children-member
      Sysdb Path:       /oper/bmd/gl/events-bdl/member/<bundlemgr_bmd_oper_EventsBDLMember_member>/ancestor_data
       Yang Path:       Cisco-IOS-XR-bundlemgr-oper:bundle-information/events-bdl/events-bdl-members/events-bdl-member/events-bdl-member-ancestor
      Sysdb Path:       /oper/bmd/gl/events-bdl/bundle/<bundlemgr_bmd_oper_EventsBDLBundle_bundle>/item_data
       Yang Path:       Cisco-IOS-XR-bundlemgr-oper:bundle-information/events-bdl/events-bdl-bundles/events-bdl-bundle/events-bdl-bundle-item
      Sysdb Path:       /oper/bmd/gl/events-bdl/rg/<bundlemgr_bmd_oper_EventsBDL_ICCPGroup_ig>/descendant_data
       Yang Path:       Cisco-IOS-XR-bundlemgr-oper:bundle-information/events-bdl/events-bdl-iccp-groups/events-bdl-iccp-group/events-bdl-bundle-descendant-iccp-group
      Sysdb Path:       /oper/bmd/gl/bundle-brief/bundle/<bundlemgr_bmd_oper_BundleBrief_bundle>/item_data
       Yang Path:       Cisco-IOS-XR-bundlemgr-oper:bundle-information/bundle-briefs/bundle-brief/bundle-brief-item
      Sysdb Path:       /oper/bmd/gl/events-mbr/bundle/<bundlemgr_bmd_oper_EventsMBRBundle_bundle>/children/<bundlemgr_bmd_oper_EventsMBRBundleChildrenMember_member>
       Yang Path:       Cisco-IOS-XR-bundlemgr-oper:bundle-information/events-mbr/events-mbr-bundles/events-mbr-bundle/events-mbr-bundle-children-members/events-mbr-bundle-children-member
      Sysdb Path:       /oper/bmd/gl/events-mbr/bundle/<bundlemgr_bmd_oper_EventsMBRBundle_bundle>/descendant_data
       Yang Path:       Cisco-IOS-XR-bundlemgr-oper:bundle-information/events-mbr/events-mbr-bundles/events-mbr-bundle/events-mbr-bundle-descendant
      Sysdb Path:       /oper/bmd/gl/events-mbr/member/<bundlemgr_bmd_oper_EventsMBRMember_member>/item_data
       Yang Path:       Cisco-IOS-XR-bundlemgr-oper:bundle-information/events-mbr/events-mbr-members/events-mbr-member/events-mbr-member-item
      Sysdb Path:       /oper/bmd/gl/events-mbr/rg/<bundlemgr_bmd_oper_EventsMBR_ICCPGroup_ig>/children/<bundlemgr_bmd_oper_EventsMBRBundleChildrenMember_ICCPGroup_member>
       Yang Path:       Cisco-IOS-XR-bundlemgr-oper:bundle-information/events-mbr/events-mbr-iccp-groups/events-mbr-iccp-group/events-mbr-bundle-children-member-iccp-groups/events-mbr-bundle-children-member-iccp-group
      Sysdb Path:       /oper/bmd/gl/events-mbr/rg/<bundlemgr_bmd_oper_EventsMBR_ICCPGroup_ig>/descendant_data
       Yang Path:       Cisco-IOS-XR-bundlemgr-oper:bundle-information/events-mbr/events-mbr-iccp-groups/events-mbr-iccp-group/events-mbr-bundle-descendant-iccp-group
      Sysdb Path:       /oper/bmd/gl/mlacp-counters-rg/rg/<bundlemgr_mlacp_rg_ctrs_oper_ICCPGroup_ig>/ancestor_data
       Yang Path:       Cisco-IOS-XR-bundlemgr-oper:bundle-information/mlacp-iccp-group-counters/iccp-groups/iccp-group/iccp-group-ancestor-bundle
      Sysdb Path:       /oper/bmd/gl/mlacp-counters-rg/rg/<bundlemgr_mlacp_rg_ctrs_oper_ICCPGroup_ig>/item_data
       Yang Path:       Cisco-IOS-XR-bundlemgr-oper:bundle-information/mlacp-iccp-group-counters/iccp-groups/iccp-group/iccp-group-item
      Sysdb Path:       /oper/bmd/gl/system-id/global/item_data
       Yang Path:       Cisco-IOS-XR-bundlemgr-oper:bundle-information/system-id/system-id-global/system-id-global-item
      Sysdb Path:       /oper/bmd/gl/system-id/rg/<bundlemgr_bmd_oper_SystemID_ICCPGroup_ig>/item_data
       Yang Path:       Cisco-IOS-XR-bundlemgr-oper:bundle-information/system-id/system-id-iccp-groups/system-id-iccp-group/system-id-iccp-group-item
      Sysdb Path:       /oper/bmd/gl/mlacp-counters-mbr/rg/<bundlemgr_mlacp_mbr_ctrs_oper_ICCPGroup_ig>/descendant_data
       Yang Path:       Cisco-IOS-XR-bundlemgr-oper:bundle-information/mlacp-member-counters/iccp-groups/iccp-group/iccp-group-item
      Sysdb Path:       /oper/bmd/gl/mlacp-counters-mbr/member/<bundlemgr_mlacp_mbr_ctrs_oper_Member_member>/item_data
       Yang Path:       Cisco-IOS-XR-bundlemgr-oper:bundle-information/mlacp-member-counters/members/member/member-item
      Sysdb Path:       /oper/bmd/gl/mlacp-counters-mbr/bundle/<bundlemgr_mlacp_mbr_ctrs_oper_Bundle_bundle>/item_data
       Yang Path:       Cisco-IOS-XR-bundlemgr-oper:bundle-information/mlacp-member-counters/bundles/bundle/bundle-item
      Sysdb Path:       /oper/bmd/gl/mlacp-counters-mbr/node/<bundlemgr_mlacp_mbr_ctrs_oper_Node_node>/item_data
       Yang Path:       Cisco-IOS-XR-bundlemgr-oper:bundle-information/mlacp-member-counters/nodes/node/node-item
```

Things seem to be fine, you see many paths, and they seem to be mapped to the internal databases. But the overall sensor path state is "Not Resolved" and if you check your collector, you won't see any data. 

This is because of this status line: "Resolved to too many containers, max : 16". 
The reason is that by default there is a restriction in IOS XR telemetry that only sensor paths containing up to 16 containers will be internally activated to push data out. Partially the reason was to encourage people to be [more specific](https://xrdocs.io/telemetry/tutorials/2018-02-13-filtering-in-telemetry-where-to-apply-and-why/) in what to stream out. Plus, we had the telemetry process single-threaded in 6.1.x. That's why it was decided to add a simple check in the code at the very beginning. 

## Removing the restriction

Starting with 6.2.25 and 6.3.2 (and later) you can remove this check and have models with >16 containers within. 

Here is how you configure this behavior:
<div class="highlighter-rouge">
<pre class="highlight">
<code>
RP/0/RP0/CPU0:NCS5501_top(config)#telemetry model-driven 
RP/0/RP0/CPU0:NCS5501_top(config-model-driven)#max-containers-per-path ?
  <0-1024>  Maximum containers allowed per path, 0 disables the check
</code>
</pre>
</div>

The configuration is simple. All you need is to specify how many countainers you want to be allowed in your sensor paths. 
As always, if you put "0", it will remove the check!

Let's see how it can help to our sensor path:

```
RP/0/RP0/CPU0:NCS5501_top(config)#sh conf
Wed Aug  1 09:31:28.959 PDT
Building configuration...
!! IOS XR Configuration version = 6.3.2
telemetry model-driven
 max-containers-per-path 0
!
end

RP/0/RP0/CPU0:NCS5501_top(config)#commit
Wed Aug  1 09:31:30.782 PDT
RP/0/RP0/CPU0:NCS5501_top(config)#
RP/0/RP0/CPU0:NCS5501_top#sh tele m sensor-group bundles internal 
Wed Aug  1 09:32:43.308 PDT
  Sensor Group Id:bundles
    Sensor Path:        Cisco-IOS-XR-bundlemgr-oper:bundle-information
    Sensor Path State:  Resolved
      Sysdb Path:       /oper/bmd/gl/bfd-counts-mbr/bundle/<bundlemgr_bmd_oper_BFDCountersBundle_bundle>/descendant_data
       Yang Path:       Cisco-IOS-XR-bundlemgr-oper:bundle-information/bfd-counters/bfd-counters-bundles/bfd-counters-bundle/bfd-counters-bundle-descendant
      Sysdb Path:       /oper/bmd/gl/bfd-counts-mbr/bundle/<bundlemgr_bmd_oper_BFDCountersBundle_bundle>/children/<bundlemgr_bmd_oper_BFDCountersBundleChildrenMember_member>
       Yang Path:       Cisco-IOS-XR-bundlemgr-oper:bundle-information/bfd-counters/bfd-counters-bundles/bfd-counters-bundle/bfd-counters-bundle-children-members/bfd-counters-bundle-children-member
      Sysdb Path:       /oper/bmd/gl/bfd-counts-mbr/bundle/<bundlemgr_bmd_oper_BFDCountersBundle_bundle>/item_data
       Yang Path:       Cisco-IOS-XR-bundlemgr-oper:bundle-information/bfd-counters/bfd-counters-bundles/bfd-counters-bundle/bfd-counters-bundle-item
      Sysdb Path:       /oper/bmd/gl/bfd-counts-mbr/member/<bundlemgr_bmd_oper_BFDCountersMember_member>/item_data
       Yang Path:       Cisco-IOS-XR-bundlemgr-oper:bundle-information/bfd-counters/bfd-counters-members/bfd-counters-member/bfd-counters-member-item
      Sysdb Path:       /oper/bmd/gl/scheduled-actions/bundle/<bundlemgr_bmd_oper_ScheduledActionsBundle_bundle>/item_data
       Yang Path:       Cisco-IOS-XR-bundlemgr-oper:bundle-information/scheduled-actions/scheduled-actions-bundles/scheduled-actions-bundle/scheduled-actions-bundle-item
      Sysdb Path:       /oper/bmd/gl/bundle/bundle/<bundlemgr_bmd_oper_BundleBundle_bundle>/descendant_data
       Yang Path:       Cisco-IOS-XR-bundlemgr-oper:bundle-information/bundle/bundle-bundles/bundle-bundle/bundle-bundle-descendant
      Sysdb Path:       /oper/bmd/gl/bundle/bundle/<bundlemgr_bmd_oper_BundleBundle_bundle>/children/<bundlemgr_bmd_oper_BundleBundleChildrenMember_member>
       Yang Path:       Cisco-IOS-XR-bundlemgr-oper:bundle-information/bundle/bundle-bundles/bundle-bundle/bundle-bundle-children-members/bundle-bundle-children-member
      Sysdb Path:       /oper/bmd/gl/bundle/bundle/<bundlemgr_bmd_oper_BundleBundle_bundle>/item_data
       Yang Path:       Cisco-IOS-XR-bundlemgr-oper:bundle-information/bundle/bundle-bundles/bundle-bundle/bundle-bundle-item
      Sysdb Path:       /oper/bmd/gl/bundle/member/<bundlemgr_bmd_oper_BundleMember_member>/ancestor_data
       Yang Path:       Cisco-IOS-XR-bundlemgr-oper:bundle-information/bundle/bundle-members/bundle-member/bundle-member-ancestor
      Sysdb Path:       /oper/bmd/gl/bundle/member/<bundlemgr_bmd_oper_BundleMember_member>/item_data
       Yang Path:       Cisco-IOS-XR-bundlemgr-oper:bundle-information/bundle/bundle-members/bundle-member/bundle-member-item
      Sysdb Path:       /oper/bmd/gl/events-rg/member/<bundlemgr_bmd_oper_EventsRGMember_member>/ancestor_data
       Yang Path:       Cisco-IOS-XR-bundlemgr-oper:bundle-information/events-rg/events-rg-members/events-rg-member/events-rg-member-ancestor
      Sysdb Path:       /oper/bmd/gl/events-rg/rg/<bundlemgr_bmd_oper_EventsRG_ICCPGroup_ig>/item_data
       Yang Path:       Cisco-IOS-XR-bundlemgr-oper:bundle-information/events-rg/events-rg-iccp-groups/events-rg-iccp-group/events-rg-bundle-item-iccp-group
      Sysdb Path:       /oper/bmd/gl/events-rg/bundle/<bundlemgr_bmd_oper_EventsRGBundle_bundle>/ancestor_data
       Yang Path:       Cisco-IOS-XR-bundlemgr-oper:bundle-information/events-rg/events-rg-bundles/events-rg-bundle/events-rg-bundle-ancestor
      Sysdb Path:       /oper/bmd/gl/lacp/bundle/<bundlemgr_bmd_oper_LACPBundle_bundle>/item_data
       Yang Path:       Cisco-IOS-XR-bundlemgr-oper:bundle-information/lacp/lacp-bundles/lacp-bundle/lacp-bundle-item
      Sysdb Path:       /oper/bmd/gl/lacp/bundle/<bundlemgr_bmd_oper_LACPBundle_bundle>/descendant_data
       Yang Path:       Cisco-IOS-XR-bundlemgr-oper:bundle-information/lacp/lacp-bundles/lacp-bundle/lacp-bundle-descendant
      Sysdb Path:       /oper/bmd/gl/lacp/bundle/<bundlemgr_bmd_oper_LACPBundle_bundle>/children/<bundlemgr_bmd_oper_LACPBundleChildrenMember_member>
       Yang Path:       Cisco-IOS-XR-bundlemgr-oper:bundle-information/lacp/lacp-bundles/lacp-bundle/lacp-bundle-children-members/lacp-bundle-children-member
      Sysdb Path:       /oper/bmd/gl/lacp/member/<bundlemgr_bmd_oper_LACPMember_member>/ancestor_data
       Yang Path:       Cisco-IOS-XR-bundlemgr-oper:bundle-information/lacp/lacp-members/lacp-member/lacp-member-ancestor
      Sysdb Path:       /oper/bmd/gl/lacp/member/<bundlemgr_bmd_oper_LACPMember_member>/item_data
       Yang Path:       Cisco-IOS-XR-bundlemgr-oper:bundle-information/lacp/lacp-members/lacp-member/lacp-member-item
      Sysdb Path:       /oper/bmd/gl/mlacp-counters-bdl/rg/<bundlemgr_mlacp_bdl_ctrs_oper_ICCPGroup_ig>/item_data
       Yang Path:       Cisco-IOS-XR-bundlemgr-oper:bundle-information/mlacp-bundle-counters/iccp-groups/iccp-group/iccp-group-item
      Sysdb Path:       /oper/bmd/gl/mlacp-counters-bdl/bundle/<bundlemgr_mlacp_bdl_ctrs_oper_Bundle_bundle>/item_data
       Yang Path:       Cisco-IOS-XR-bundlemgr-oper:bundle-information/mlacp-bundle-counters/bundles/bundle/bundle-item
      Sysdb Path:       /oper/bmd/gl/mlacp-counters-bdl/node/<bundlemgr_mlacp_bdl_ctrs_oper_Node_node>/item_data
       Yang Path:       Cisco-IOS-XR-bundlemgr-oper:bundle-information/mlacp-bundle-counters/nodes/node/node-item
      Sysdb Path:       /oper/bmd/gl/protect/bundle/<bundlemgr_bmd_oper_ProtectBundle_bundle>/item_data
       Yang Path:       Cisco-IOS-XR-bundlemgr-oper:bundle-information/protect/protect-bundles/protect-bundle/protect-bundle-item
      Sysdb Path:       /oper/bmd/gl/mlacp-brief/bundle/<bundlemgr_bmd_oper_mLACPBundleBrief_bundle>/item_data
       Yang Path:       Cisco-IOS-XR-bundlemgr-oper:bundle-information/mlacp-brief/mlacp-bundle-briefs/mlacp-bundle-brief/mlacp-bundle-item-brief
      Sysdb Path:       /oper/bmd/gl/mlacp-brief/rg/<bundlemgr_bmd_oper_mLACPBrief_ICCPGroup_ig>/item_data
       Yang Path:       Cisco-IOS-XR-bundlemgr-oper:bundle-information/mlacp-brief/mlacp-brief-iccp-groups/mlacp-brief-iccp-group/mlacp-brief-iccp-group-item
      Sysdb Path:       /oper/bmd/gl/mlacp/bundle/<bundlemgr_bmd_oper_mLACPBundle_bundle>/item_data
       Yang Path:       Cisco-IOS-XR-bundlemgr-oper:bundle-information/mlacp/mlacp-bundles/mlacp-bundle/mlacp-bundle-item
      Sysdb Path:       /oper/bmd/gl/mlacp/rg/<bundlemgr_bmd_oper_mLACP_ICCPGroup_ig>/item_data
       Yang Path:       Cisco-IOS-XR-bundlemgr-oper:bundle-information/mlacp/mlacp-iccp-groups/mlacp-iccp-group/mlacp-iccp-group-item
      Sysdb Path:       /oper/bmd/gl/mac-allocation/global/item_data
       Yang Path:       Cisco-IOS-XR-bundlemgr-oper:bundle-information/mac-allocation/mac-allocation-global/mac-allocation-global-item
      Sysdb Path:       /oper/bmd/gl/events/member/<bundlemgr_bmd_oper_EventsMember_member>/item_data
       Yang Path:       Cisco-IOS-XR-bundlemgr-oper:bundle-information/events/events-members/events-member/events-member-item
      Sysdb Path:       /oper/bmd/gl/events/member/<bundlemgr_bmd_oper_EventsMember_member>/ancestor_data
       Yang Path:       Cisco-IOS-XR-bundlemgr-oper:bundle-information/events/events-members/events-member/events-member-ancestor
      Sysdb Path:       /oper/bmd/gl/events/bundle/<bundlemgr_bmd_oper_EventsBundle_bundle>/ancestor_data
       Yang Path:       Cisco-IOS-XR-bundlemgr-oper:bundle-information/events/events-bundles/events-bundle/events-bundle-ancestor
      Sysdb Path:       /oper/bmd/gl/events/bundle/<bundlemgr_bmd_oper_EventsBundle_bundle>/item_data
       Yang Path:       Cisco-IOS-XR-bundlemgr-oper:bundle-information/events/events-bundles/events-bundle/events-bundle-item
      Sysdb Path:       /oper/bmd/gl/events/bundle/<bundlemgr_bmd_oper_EventsBundle_bundle>/descendant_data
       Yang Path:       Cisco-IOS-XR-bundlemgr-oper:bundle-information/events/events-bundles/events-bundle/events-bundle-descendant
      Sysdb Path:       /oper/bmd/gl/events/bundle/<bundlemgr_bmd_oper_EventsBundle_bundle>/children/<bundlemgr_bmd_oper_EventsBundleChildrenMember_member>
       Yang Path:       Cisco-IOS-XR-bundlemgr-oper:bundle-information/events/events-bundles/events-bundle/events-bundle-children-members/events-bundle-children-member
      Sysdb Path:       /oper/bmd/gl/events-bdl/member/<bundlemgr_bmd_oper_EventsBDLMember_member>/ancestor_data
       Yang Path:       Cisco-IOS-XR-bundlemgr-oper:bundle-information/events-bdl/events-bdl-members/events-bdl-member/events-bdl-member-ancestor
      Sysdb Path:       /oper/bmd/gl/events-bdl/bundle/<bundlemgr_bmd_oper_EventsBDLBundle_bundle>/item_data
       Yang Path:       Cisco-IOS-XR-bundlemgr-oper:bundle-information/events-bdl/events-bdl-bundles/events-bdl-bundle/events-bdl-bundle-item
      Sysdb Path:       /oper/bmd/gl/events-bdl/rg/<bundlemgr_bmd_oper_EventsBDL_ICCPGroup_ig>/descendant_data
       Yang Path:       Cisco-IOS-XR-bundlemgr-oper:bundle-information/events-bdl/events-bdl-iccp-groups/events-bdl-iccp-group/events-bdl-bundle-descendant-iccp-group
      Sysdb Path:       /oper/bmd/gl/bundle-brief/bundle/<bundlemgr_bmd_oper_BundleBrief_bundle>/item_data
       Yang Path:       Cisco-IOS-XR-bundlemgr-oper:bundle-information/bundle-briefs/bundle-brief/bundle-brief-item
      Sysdb Path:       /oper/bmd/gl/events-mbr/bundle/<bundlemgr_bmd_oper_EventsMBRBundle_bundle>/children/<bundlemgr_bmd_oper_EventsMBRBundleChildrenMember_member>
       Yang Path:       Cisco-IOS-XR-bundlemgr-oper:bundle-information/events-mbr/events-mbr-bundles/events-mbr-bundle/events-mbr-bundle-children-members/events-mbr-bundle-children-member
      Sysdb Path:       /oper/bmd/gl/events-mbr/bundle/<bundlemgr_bmd_oper_EventsMBRBundle_bundle>/descendant_data
       Yang Path:       Cisco-IOS-XR-bundlemgr-oper:bundle-information/events-mbr/events-mbr-bundles/events-mbr-bundle/events-mbr-bundle-descendant
      Sysdb Path:       /oper/bmd/gl/events-mbr/member/<bundlemgr_bmd_oper_EventsMBRMember_member>/item_data
       Yang Path:       Cisco-IOS-XR-bundlemgr-oper:bundle-information/events-mbr/events-mbr-members/events-mbr-member/events-mbr-member-item
      Sysdb Path:       /oper/bmd/gl/events-mbr/rg/<bundlemgr_bmd_oper_EventsMBR_ICCPGroup_ig>/children/<bundlemgr_bmd_oper_EventsMBRBundleChildrenMember_ICCPGroup_member>
       Yang Path:       Cisco-IOS-XR-bundlemgr-oper:bundle-information/events-mbr/events-mbr-iccp-groups/events-mbr-iccp-group/events-mbr-bundle-children-member-iccp-groups/events-mbr-bundle-children-member-iccp-group
      Sysdb Path:       /oper/bmd/gl/events-mbr/rg/<bundlemgr_bmd_oper_EventsMBR_ICCPGroup_ig>/descendant_data
       Yang Path:       Cisco-IOS-XR-bundlemgr-oper:bundle-information/events-mbr/events-mbr-iccp-groups/events-mbr-iccp-group/events-mbr-bundle-descendant-iccp-group
      Sysdb Path:       /oper/bmd/gl/mlacp-counters-rg/rg/<bundlemgr_mlacp_rg_ctrs_oper_ICCPGroup_ig>/ancestor_data
       Yang Path:       Cisco-IOS-XR-bundlemgr-oper:bundle-information/mlacp-iccp-group-counters/iccp-groups/iccp-group/iccp-group-ancestor-bundle
      Sysdb Path:       /oper/bmd/gl/mlacp-counters-rg/rg/<bundlemgr_mlacp_rg_ctrs_oper_ICCPGroup_ig>/item_data
       Yang Path:       Cisco-IOS-XR-bundlemgr-oper:bundle-information/mlacp-iccp-group-counters/iccp-groups/iccp-group/iccp-group-item
      Sysdb Path:       /oper/bmd/gl/system-id/global/item_data
       Yang Path:       Cisco-IOS-XR-bundlemgr-oper:bundle-information/system-id/system-id-global/system-id-global-item
      Sysdb Path:       /oper/bmd/gl/system-id/rg/<bundlemgr_bmd_oper_SystemID_ICCPGroup_ig>/item_data
       Yang Path:       Cisco-IOS-XR-bundlemgr-oper:bundle-information/system-id/system-id-iccp-groups/system-id-iccp-group/system-id-iccp-group-item
      Sysdb Path:       /oper/bmd/gl/mlacp-counters-mbr/rg/<bundlemgr_mlacp_mbr_ctrs_oper_ICCPGroup_ig>/descendant_data
       Yang Path:       Cisco-IOS-XR-bundlemgr-oper:bundle-information/mlacp-member-counters/iccp-groups/iccp-group/iccp-group-item
      Sysdb Path:       /oper/bmd/gl/mlacp-counters-mbr/member/<bundlemgr_mlacp_mbr_ctrs_oper_Member_member>/item_data
       Yang Path:       Cisco-IOS-XR-bundlemgr-oper:bundle-information/mlacp-member-counters/members/member/member-item
      Sysdb Path:       /oper/bmd/gl/mlacp-counters-mbr/bundle/<bundlemgr_mlacp_mbr_ctrs_oper_Bundle_bundle>/item_data
       Yang Path:       Cisco-IOS-XR-bundlemgr-oper:bundle-information/mlacp-member-counters/bundles/bundle/bundle-item
      Sysdb Path:       /oper/bmd/gl/mlacp-counters-mbr/node/<bundlemgr_mlacp_mbr_ctrs_oper_Node_node>/item_data
       Yang Path:       Cisco-IOS-XR-bundlemgr-oper:bundle-information/mlacp-member-counters/nodes/node/node-item
```

Voila! We don't see any restriction, and we're good to go!

## Conclusion

The cli knob described in this post might be helpful in your telemetry testing. You can quickly get all the counters pushed from some specific model to check how it will look like. Whenever it comes to your prod network, it still might be better to be more strict and exact in what you want to stream. It will be better for the router, for network bandwidth and your collector(s). 

