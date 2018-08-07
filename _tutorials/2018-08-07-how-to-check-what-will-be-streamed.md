---
published: true
date: '2018-08-07 06:33 -0700'
title: How to check what will be streamed?
author: Viktor Osipchuk
tags:
  - iosxr
  - cisco
  - Telemetry
  - MDT
excerpt: >-
  This post will describe a convenient way of checking data that can be streamed
  from the box using Model Driven Telemetry
---
{% include toc icon="table" title="How to check what will be streamed?" %}
{% include base_path %}

Have you ever asked yourself about how you can check what data will be pushed from your router if you configure one sensor path or another? Usually, you would end up with a four-step approach: 
- step one: find the right YANG model 
- step two: MDT configuration on your router (the destination, sensor path, and subscription configurations)
- step three: install Pipeline and configure [the "dump" mode](https://xrdocs.io/telemetry/tutorials/2018-03-01-everything-you-need-to-know-about-pipeline/#pipeline-out-to-dumptxt), where you will get a text file with all the counters in JSON format. 
- step four: start Pipeline, get data streamed, open the file and, finally, start the analysis. 

Even if today you can do that procedure easily with your eyes closed, it might be not the best way! In this post we will talk about a more elegant way of doing a quick check of data to be streamed from your box. 

## Before we start

Before we start, it is important to understand two things:
1. The special script is available with XR 6.3.1+.
2. This is not an official feature you will find on the cisco.com documentation site. 

In other words, to use this methodology, you need to have a pretty recent XR code, and you should not call TAC in case something doesn't work as expected :)

Okay, with that note in mind, let's move on.

## A quick way to check counters

The approach we will talk about can help you to move from the four-steps to two-steps procedure:
- step one: you still need to know the YANG model
- step two: activate MDT on a router and run a single CLI command

That's it! Let's see how it is done with more details. 

The very first step is to get the needed model. Yes, you still need to know which model you should use. But, it is not that hard to guess that MPLS-TE counters will be within the "Cisco-IOS-XR-mpls-te-oper.yang", right? After you identified your model, just find the path you're interested in. 

The second step is to have the MDT process running on your router. All you need from the MDT perspective is:

```
RP/0/RP0/CPU0:NCS5501#sh run telemetry model-driven 
Mon Aug  6 10:17:38.179 PDT
telemetry model-driven
!
RP/0/RP0/CPU0:NCS5501#
```

Now, run this exec-mode command on your router:
<div class="highlighter-rouge">
<pre class="highlight">
<code>
 RP/0/RP0/CPU0:ios# run mdt_exec -s <i>your_sensor_path</i> -c <i>cadence</i> 
</code>
</pre>
</div>

"-s" is where you paste the sensor path.  
"-c" is where you define the sample interval (in msec). It does support the event-driven behavior in case you specify "0" there (for the models with EDT support). 

To stop the script running, you just need to hit any key (otherwise, you will see counters on the screen every sample interval configured)

Let's do a quick example. Following the [previous post](https://xrdocs.io/telemetry/tutorials/2018-08-01-going-above-16-containers-per-path/), I've decided to stay with the "Cisco-IOS-XR-bundlemgr-oper.yang" model here as well. To make the example short, this path was used: "Cisco-IOS-XR-bundlemgr-oper:bundles-adjacency/nodes/node/brief". Here is how the script was started:

```
RP/0/RP0/CPU0:NCS5501#run mdt_exec -s Cisco-IOS-XR-bundlemgr-oper:bundles-adjacency/nodes/node/brief -c 10000
```

After some time, you will see the output with all the telemetry data on your screen. In my case: 

```
Mon Aug  6 10:30:21.215 PDT
Enter any key to exit...
 Sub_id 200000001, flag 0, len 0
 Sub_id 200000001, flag 4, len 1613
--------
{"node_id_str":"NCS5501","subscription_id_str":"app_TEST_200000001","encoding_path":"Cisco-IOS-XR-bundlemgr-oper:bundles-adjacency/nodes/node/brief","collection_id":4830917,"collection_start_time":1533576631472,"msg_timestamp":1533576631477,"data_json":[{"timestamp":1533576631477,"keys":{"node-name":"0/0/CPU0"},"content":{"bundle-data":[{"interface-name":"Bundle-Ether12","sub-interface-count":0,"member-count":2,"total-weight":2,"sub-interfaces":[]},{"interface-name":"Bundle-Ether13","sub-interface-count":0,"member-count":3,"total-weight":3,"sub-interfaces":[]},{"interface-name":"Bundle-Ether14","sub-interface-count":0,"member-count":2,"total-weight":2,"sub-interfaces":[]},{"interface-name":"Bundle-Ether15","sub-interface-count":0,"member-count":2,"total-weight":2,"sub-interfaces":[]},{"interface-name":"Bundle-Ether16","sub-interface-count":0,"member-count":1,"total-weight":1,"sub-interfaces":[]}]}},{"timestamp":1533576631479,"keys":{"node-name":"0/RP0/CPU0"},"content":{"bundle-data":[{"interface-name":"Bundle-Ether12","sub-interface-count":0,"member-count":2,"total-weight":2,"sub-interfaces":[]},{"interface-name":"Bundle-Ether13","sub-interface-count":0,"member-count":3,"total-weight":3,"sub-interfaces":[]},{"interface-name":"Bundle-Ether14","sub-interface-count":0,"member-count":2,"total-weight":2,"sub-interfaces":[]},{"interface-name":"Bundle-Ether15","sub-interface-count":0,"member-count":2,"total-weight":2,"sub-interfaces":[]},{"interface-name":"Bundle-Ether16","sub-interface-count":0,"member-count":1,"total-weight":1,"sub-interfaces":[]}]}}],"collection_end_time":1533576631479}
--------
```

All the counters on the screen are in JSON format with minimum actions from your side! 
If you want to see in a more human-readable format, you can use any preferred JSON formatter. Let's try the output with the [first JSON formatter found](https://jsonformatter.curiousconcept.com/):

```
{
   "node_id_str":"NCS5501",
   "subscription_id_str":"app_TEST_200000001",
   "encoding_path":"Cisco-IOS-XR-bundlemgr-oper:bundles-adjacency/nodes/node/brief",
   "collection_id":4830917,
   "collection_start_time":1533576631472,
   "msg_timestamp":1533576631477,
   "data_json":[
      {
         "timestamp":1533576631477,
         "keys":{
            "node-name":"0/0/CPU0"
         },
         "content":{
            "bundle-data":[
               {
                  "interface-name":"Bundle-Ether12",
                  "sub-interface-count":0,
                  "member-count":2,
                  "total-weight":2,
                  "sub-interfaces":[

                  ]
               },
               {
                  "interface-name":"Bundle-Ether13",
                  "sub-interface-count":0,
                  "member-count":3,
                  "total-weight":3,
                  "sub-interfaces":[

                  ]
               },
               {
                  "interface-name":"Bundle-Ether14",
                  "sub-interface-count":0,
                  "member-count":2,
                  "total-weight":2,
                  "sub-interfaces":[

                  ]
               },
               {
                  "interface-name":"Bundle-Ether15",
                  "sub-interface-count":0,
                  "member-count":2,
                  "total-weight":2,
                  "sub-interfaces":[

                  ]
               },
               {
                  "interface-name":"Bundle-Ether16",
                  "sub-interface-count":0,
                  "member-count":1,
                  "total-weight":1,
                  "sub-interfaces":[

                  ]
               }
            ]
         }
      },
      {
         "timestamp":1533576631479,
         "keys":{
            "node-name":"0/RP0/CPU0"
         },
         "content":{
            "bundle-data":[
               {
                  "interface-name":"Bundle-Ether12",
                  "sub-interface-count":0,
                  "member-count":2,
                  "total-weight":2,
                  "sub-interfaces":[

                  ]
               },
               {
                  "interface-name":"Bundle-Ether13",
                  "sub-interface-count":0,
                  "member-count":3,
                  "total-weight":3,
                  "sub-interfaces":[

                  ]
               },
               {
                  "interface-name":"Bundle-Ether14",
                  "sub-interface-count":0,
                  "member-count":2,
                  "total-weight":2,
                  "sub-interfaces":[

                  ]
               },
               {
                  "interface-name":"Bundle-Ether15",
                  "sub-interface-count":0,
                  "member-count":2,
                  "total-weight":2,
                  "sub-interfaces":[

                  ]
               },
               {
                  "interface-name":"Bundle-Ether16",
                  "sub-interface-count":0,
                  "member-count":1,
                  "total-weight":1,
                  "sub-interfaces":[

                  ]
               }
            ]
         }
      }
   ],
   "collection_end_time":1533576631479
}
```

And here you go, a more readable and clear view! 
In the next sections a few more details will be covered.

## Checking a path with several containers

It is important to think about the depth of your sensor path. In the [previous post](https://xrdocs.io/telemetry/tutorials/2018-08-01-going-above-16-containers-per-path/) we talked about the depth of paths and how to make Telemetry working even if there are more than 16 containers contained. 
It is expected, that you will want to try to check a big set of counters to see what you can get from one model or another. The probability that you meet 16 containers limit is pretty high. That's why it might be needed for you to configure this command:

```
telemetry model-driven
 max-containers-per-path 0
```
under the MDT configuration. (yes, the internal workflow stays the same in both cases). 

Whenever you configure a path with several containers, each container will be separated for your convenience. It will look like this:

```
RP/0/RP0/CPU0:NCS5501#run mdt_exec -s Cisco-IOS-XR-bundlemgr-oper:bundles-adjacency -c 20000               
Mon Aug  6 09:09:28.821 PDT
Enter any key to exit...
 Sub_id 200000001, flag 0, len 0
 Sub_id 200000001, flag 4, len 1613
--------
{"node_id_str":"NCS5501","subscription_id_str":"app_TEST_200000001","encoding_path":"Cisco-IOS-XR-bundlemgr-oper:bundles-adjacency/nodes/node/brief","collection_id":4790350,"collection_start_time":1533571789058,"msg_timestamp":1533571789063,"data_json":[{"timestamp":1533571789063,"keys":{"node-name":"0/0/CPU0"},"content":{"bundle-data":[{"interface-name":"Bundle-Ether12","sub-interface-count":0,"member-count":2,"total-weight":2,"sub-interfaces":[]},{"interface-name":"Bundle-Ether13","sub-interface-count":0,"member-count":3,"total-weight":3,"sub-interfaces":[]},{"interface-name":"Bundle-Ether14","sub-interface-count":0,"member-count":2,"total-weight":2,"sub-interfaces":[]},{"interface-name":"Bundle-Ether15","sub-interface-count":0,"member-count":2,"total-weight":2,"sub-interfaces":[]},{"interface-name":"Bundle-Ether16","sub-interface-count":0,"member-count":1,"total-weight":1,"sub-interfaces":[]}]}},{"timestamp":1533571789066,"keys":{"node-name":"0/RP0/CPU0"},"content":{"bundle-data":[{"interface-name":"Bundle-Ether12","sub-interface-count":0,"member-count":2,"total-weight":2,"sub-interfaces":[]},{"interface-name":"Bundle-Ether13","sub-interface-count":0,"member-count":3,"total-weight":3,"sub-interfaces":[]},{"interface-name":"Bundle-Ether14","sub-interface-count":0,"member-count":2,"total-weight":2,"sub-interfaces":[]},{"interface-name":"Bundle-Ether15","sub-interface-count":0,"member-count":2,"total-weight":2,"sub-interfaces":[]},{"interface-name":"Bundle-Ether16","sub-interface-count":0,"member-count":1,"total-weight":1,"sub-interfaces":[]}]}}],"collection_end_time":1533571789067}
--------
 Sub_id 200000001, flag 4, len 6022
--------
{"node_id_str":"NCS5501","subscription_id_str":"app_TEST_200000001","encoding_path":"Cisco-IOS-XR-bundlemgr-oper:bundles-adjacency/nodes/node/bundles/bundle/bundle-info","collection_id":4790351,"collection_start_time":1533571789067,"msg_timestamp":1533571789074,"data_json":[{"timestamp":1533571789073,"keys":{"node-name":"0/0/CPU0","bundle-name":"Bundle-Ether12"},"content":{"brief":{"interface-name":"Bundle-Ether12","sub-interface-count":0,"member-count":2,"total-weight":2,"sub-interfaces":[]},"media":"ethernet","max-member-count":64,"load-balance-data":{"type":"default","value":0,"local-link-threshold":65},"avoid-rebalance":false,"members":[{"interface-name":"TenGigE0/0/0/0","link-id":1,"link-order-number":1,"bandwidth":1},{"interface-name":"TenGigE0/0/0/1","link-id":0,"link-order-number":0,"bandwidth":1}],"sub-interfaces":[]}},{"timestamp":1533571789075,"keys":{"node-name":"0/0/CPU0","bundle-name":"Bundle-Ether13"},"content":{"brief":{"interface-name":"Bundle-Ether13","sub-interface-count":0,"member-count":3,"total-weight":3,"sub-interfaces":[]},"media":"ethernet","max-member-count":3,"load-balance-data":{"type":"default","value":0,"local-link-threshold":65},"avoid-rebalance":false,"members":[{"interface-name":"HundredGigE0/0/1/5","link-id":0,"link-order-number":0,"bandwidth":1},{"interface-name":"HundredGigE0/0/1/4","link-id":1,"link-order-number":1,"bandwidth":1},{"interface-name":"HundredGigE0/0/1/3","link-id":2,"link-order-number":2,"bandwidth":1}],"sub-interfaces":[]}},{"timestamp":1533571789076,"keys":{"node-name":"0/0/CPU0","bundle-name":"Bundle-Ether14"},"content":{"brief":{"interface-name":"Bundle-Ether14","sub-interface-count":0,"member-count":2,"total-weight":2,"sub-interfaces":[]},"media":"ethernet","max-member-count":64,"load-balance-data":{"type":"default","value":0,"local-link-threshold":65},"avoid-rebalance":false,"members":[{"interface-name":"HundredGigE0/0/1/2","link-id":0,"link-order-number":0,"bandwidth":1},{"interface-name":"HundredGigE0/0/1/1","link-id":1,"link-order-number":1,"bandwidth":1}],"sub-interfaces":[]}},{"timestamp":1533571789078,"keys":{"node-name":"0/0/CPU0","bundle-name":"Bundle-Ether15"},"content":{"brief":{"interface-name":"Bundle-Ether15","sub-interface-count":0,"member-count":2,"total-weight":2,"sub-interfaces":[]},"media":"ethernet","max-member-count":64,"load-balance-data":{"type":"default","value":0,"local-link-threshold":65},"avoid-rebalance":false,"members":[{"interface-name":"TenGigE0/0/0/2","link-id":1,"link-order-number":1,"bandwidth":1},{"interface-name":"TenGigE0/0/0/3","link-id":0,"link-order-number":0,"bandwidth":1}],"sub-interfaces":[]}},{"timestamp":1533571789079,"keys":{"node-name":"0/0/CPU0","bundle-name":"Bundle-Ether16"},"content":{"brief":{"interface-name":"Bundle-Ether16","sub-interface-count":0,"member-count":1,"total-weight":1,"sub-interfaces":[]},"media":"ethernet","max-member-count":64,"load-balance-data":{"type":"default","value":0,"local-link-threshold":65},"avoid-rebalance":false,"members":[{"interface-name":"HundredGigE0/0/1/0","link-id":0,"link-order-number":0,"bandwidth":1}],"sub-interfaces":[]}},{"timestamp":1533571789083,"keys":{"node-name":"0/RP0/CPU0","bundle-name":"Bundle-Ether12"},"content":{"brief":{"interface-name":"Bundle-Ether12","sub-interface-count":0,"member-count":2,"total-weight":2,"sub-interfaces":[]},"media":"ethernet","max-member-count":64,"load-balance-data":{"type":"default","value":0,"local-link-threshold":65},"avoid-rebalance":false,"members":[{"interface-name":"TenGigE0/0/0/0","link-id":1,"link-order-number":1,"bandwidth":1},{"interface-name":"TenGigE0/0/0/1","link-id":0,"link-order-number":0,"bandwidth":1}],"sub-interfaces":[]}},{"timestamp":1533571789084,"keys":{"node-name":"0/RP0/CPU0","bundle-name":"Bundle-Ether13"},"content":{"brief":{"interface-name":"Bundle-Ether13","sub-interface-count":0,"member-count":3,"total-weight":3,"sub-interfaces":[]},"media":"ethernet","max-member-count":3,"load-balance-data":{"type":"default","value":0,"local-link-threshold":65},"avoid-rebalance":false,"members":[{"interface-name":"HundredGigE0/0/1/5","link-id":0,"link-order-number":0,"bandwidth":1},{"interface-name":"HundredGigE0/0/1/4","link-id":1,"link-order-number":1,"bandwidth":1},{"interface-name":"HundredGigE0/0/1/3","link-id":2,"link-order-number":2,"bandwidth":1}],"sub-interfaces":[]}},{"timestamp":1533571789085,"keys":{"node-name":"0/RP0/CPU0","bundle-name":"Bundle-Ether14"},"content":{"brief":{"interface-name":"Bundle-Ether14","sub-interface-count":0,"member-count":2,"total-weight":2,"sub-interfaces":[]},"media":"ethernet","max-member-count":64,"load-balance-data":{"type":"default","value":0,"local-link-threshold":65},"avoid-rebalance":false,"members":[{"interface-name":"HundredGigE0/0/1/2","link-id":0,"link-order-number":0,"bandwidth":1},{"interface-name":"HundredGigE0/0/1/1","link-id":1,"link-order-number":1,"bandwidth":1}],"sub-interfaces":[]}},{"timestamp":1533571789086,"keys":{"node-name":"0/RP0/CPU0","bundle-name":"Bundle-Ether15"},"content":{"brief":{"interface-name":"Bundle-Ether15","sub-interface-count":0,"member-count":2,"total-weight":2,"sub-interfaces":[]},"media":"ethernet","max-member-count":64,"load-balance-data":{"type":"default","value":0,"local-link-threshold":65},"avoid-rebalance":false,"members":[{"interface-name":"TenGigE0/0/0/2","link-id":1,"link-order-number":1,"bandwidth":1},{"interface-name":"TenGigE0/0/0/3","link-id":0,"link-order-number":0,"bandwidth":1}],"sub-interfaces":[]}},{"timestamp":1533571789087,"keys":{"node-name":"0/RP0/CPU0","bundle-name":"Bundle-Ether16"},"content":{"brief":{"interface-name":"Bundle-Ether16","sub-interface-count":0,"member-count":1,"total-weight":1,"sub-interfaces":[]},"media":"ethernet","max-member-count":64,"load-balance-data":{"type":"default","value":0,"local-link-threshold":65},"avoid-rebalance":false,"members":[{"interface-name":"HundredGigE0/0/1/0","link-id":0,"link-order-number":0,"bandwidth":1}],"sub-interfaces":[]}}],"collection_end_time":1533571789088}
--------

```
Be aware that you will need to wait for some time to see data for large collections and/or paths with many containers. It will be silent while data is prepared to be seen on the screen, be patient.

If you don't see anything for a long period, there might be an issue (wrong sensor path, the feature not configured or just a bug). No additional error notification is provided today, but the plan is to improve that behavior in future (no exact date, it is not an official feature). 
As of today, you check the subscription created to do quick checks if you think something is wrong. Let's see how to do this.

## Is there anything else we can check?

Internally the command works almost the same way as when you configure Telemetry through the main configuration (destination, sensor group, subscription). It means that there should be a subscription. 
Each time you use the script, you will have a dynamic subscription created. 
In the output above you can see this:

```
<<< skipped >>>
 Sub_id 200000001
<<< skipped >>>
```

This is the number of the subscription created. You will need to add "app_TEST_" at the beginning, and the full subscription name is ready. 
While your command is still active (it means you didn't hit any key after the start), you can check the content of the subscription:

```
RP/0/RP0/CPU0:NCS5501# sh telem m subscription app_TEST_200000001 internal 
Mon Aug  6 10:35:23.164 PDT
Subscription:  app_TEST_200000001
-------------
  State:       ACTIVE
  DSCP/Qos marked value: Default
  Sensor groups:
  Id: app_TEST_200000001
    Sample Interval:      10000 ms
    Sensor Path:          Cisco-IOS-XR-bundlemgr-oper:bundles-adjacency/nodes/node/brief
    Sensor Path State:    Resolved

  Destination Groups:
  Group Id: APP_1039
    Encoding:             json
    Transport:            app
    State:                Active
    TLS :                 True
    Total bytes sent:     1613
    Total packets sent:   1
    Last Sent time:       2018-08-06 10:35:19.3012280773 -0700

  Collection Groups:
  ------------------
    Id: 86
    Sample Interval:      10000 ms
    Encoding:             json
    Num of collection:    1
    Collection time:      Min:     6 ms Max:     6 ms
    Total time:           Min:     6 ms Avg:     6 ms Max:     6 ms
    Total Deferred:       0
    Total Send Errors:    0
    Total Send Drops:     0
    Total Other Errors:   0
    No data Instances:    0
    Last Collection Start:2018-08-06 10:35:19.3012274773 -0700
    Last Collection End:  2018-08-06 10:35:19.3012280773 -0700
    Sensor Path:          Cisco-IOS-XR-bundlemgr-oper:bundles-adjacency/nodes/node/brief

      Sysdb Path:     /oper/bm-adj/node/*/brief
      Count:          1 Method: GET Min: 6 ms Avg: 6 ms Max: 6 ms
      Item Count:     2 Status: Active
      Missed Collections:0  send bytes: 1613 packets: 1 dropped bytes: 0
                      success         errors          deferred/drops  
      Gets            2               0               
      List            1               0               
      Datalist        0               0               
      Finddata        1               0               
      GetBulk         0               0               
      Encode                          0               0               
      Send                            0               0               
```

You can collect additional information there, if needed!

## Conclusion

While it is not an official feature, I think it should be pretty helpful for people trying to make their steps with telemetry. Having a possibility to get all the counters in real time on your screen to quickly validate something or just understand the format should make your life easier. More to come soon, stay with us.

