---
published: true
date: '2018-01-17 08:51 -0800'
title: Sample intervals in telemetry configuration
author: Viktor Osipchuk
excerpt: Sample intervals in telemetry configuration
tags:
  - iosxr
  - telemetry
  - MDT
  - automation
---

{% include toc icon="table" title="Sample intervals in telemetry configuration" %}
{% include base_path %}

We've got several similar questions recently about sample intervals in MDT configuration on IOS-XR routers. A basic MDT configuration is pretty clear and you can find a number of [documents](https://www.cisco.com/c/en/us/td/docs/iosxr/ncs5500/telemetry/b-telemetry-cg-ncs5500-62x/b-telemetry-cg-ncs5500-62x_chapter_010.html) and [posts](https://xrdocs.github.io/telemetry/tutorials/2016-07-21-configuring-model-driven-telemetry-mdt/) explaining how to do it. But this is a minimum configuration and there are many additional features coming with new releases. The goal is this document is to explain how "sample-interval" works internally and options you have today.

## Sample-interval overview

As a short reminder, "sample-interval" is used inside the subscription level within Model-Driven Telemetry configuration on an IOS-XR router:
<div class="highlighter-rouge">
<pre class="highlight">
<code>   	
telemetry model-driven
  subscription SUB1
    sensor-group-id SGROUP1 sample-interval 10000
    destination-id DGROUP1</code>
</pre>
</div>

The general meaning is how fast you want configured sensor-path (the defined data) to be pushed out of the box. But how does this work internally? 

Here is a simple schematic view of how "sample-interval" works:

![](https://github.com/xrdocs/xrdocs-images/blob/gh-pages/assets/tutorial-images/vosipchu/sample-intervals/Sample-interval2.png?raw=true)

Let's say that at To your configuration was applied and the first collection just started. The collection by itself needs some time to gather information and put it on the wire. Define this period of time as Tc.  Going this way, at [To + Tc] moment your system will be ready to start counting down your configured "sample-interval" [Ts].  Your next collection will start its internal calls right after [To + Tc + Ts], or, speaking more general, instead of having your configured amount of time, you will observe total time of the collection plus the "sample-interval" itself. 

You can feel that behavior analyzing the logs on your collector. In the example below “sample-interval” was configured to be 20 seconds (20000 msec):
<div class="highlighter-rouge">
<pre class="highlight">
<code>
{'collection_id': 150, 'node_id_str': 'R1', 'msg_timestamp': <mark>1511451973975</mark>, 'collection_start_time': <mark>1511451973975</mark>, 'collection_end_time': <mark>1511451974094</mark>, 'encoding_path': 'Cisco-IOS-XR-infra-statsd-oper:infra-statistics/interfaces/interface/latest/protocols/protocol', 'subscription_id_str': 'sub_testsampleinterval'}
 
{'collection_id': 151, 'node_id_str': 'R1', 'msg_timestamp': <mark>1511451994094</mark>, 'collection_start_time': 1511451994094, 'collection_end_time': 1511451994209, 'encoding_path': 'Cisco-IOS-XR-infra-statsd-oper:infra-statistics/interfaces/interface/latest/protocols/protocol', 'subscription_id_str': 'sub_ testsampleinterval'}
</code>
</pre>
</div>
If you do a simple math with the "msg_timestamp" values from both entries you will find out, that the difference is 20119 (msec). This number is slightly bigger than the configured "sample-interval", so, let's find out the time needed for the first collection by subtracting "collection_start_time" from "collection_end_time". The difference is exactly 119 (msec). This simple exercise confirms the behavior explained above. 


## Strict-timer overview

Starting with IOS-XR 6.2.2 you can add a "strict-timer" command under the subscription:

<div class="highlighter-rouge">
<pre class="highlight">
<code>   	
telemetry model-driven
  subscription SUB1
    sensor-group-id SGROUP1 sample-interval 10000
    sensor-group-id SGROUP1 strict-timer
    destination-id DGROUP1</code>
</pre>
</div>

This command is not used alone, but together with "sample-interval". The question comes, how does this influence our behavior?

Let's have a look at this picture:

![](https://github.com/xrdocs/xrdocs-images/blob/gh-pages/assets/tutorial-images/vosipchu/sample-intervals/Strict-timer.png?raw=true)


The system starts at the same To , when you got your configuration applied. The collection needs the same amount of time, Tc.  But instead of waiting for the [T0 + TC] moment your router will start counting down your configured "sample-interval" right at the moment To.  This way, your next collection will start exactly at [To + Ts], or, using other words, the next collection will happen exactly when you expect it to start (according to the configured "sample-interval").

Here are the logs from the same system, but running with "strict-timer" applied this time:

<div class="highlighter-rouge">
<pre class="highlight">
<code>

{'collection_id': 365, 'node_id_str': 'R1', 'msg_timestamp': <mark>1511551935943</mark>, 'collection_start_time': <mark>1511551935943</mark>, 'collection_end_time': <mark>1511551936062</mark>, 'encoding_path': 'Cisco-IOS-XR-infra-statsd-oper:infra-statistics/interfaces/interface/latest/protocols/protocol', 'subscription_id_str': 'sub_testsampleinterval'}
 
{'collection_id': 366, 'node_id_str': 'R1', 'msg_timestamp': <mark>1511551955943</mark>, 'collection_start_time': 1511551955943, 'collection_end_time': 1511551956062, 'encoding_path': 'Cisco-IOS-XR-infra-statsd-oper:infra-statistics/interfaces/interface/latest/protocols/protocol', 'subscription_id_str': 'sub_ testsampleinterval'}

</code>
</pre>
</div>

Following the same math operations as above, you can see that difference between "msg_timestamp" values in both entries is exactly 20000 (msec), as it was configured (and each collection takes the same 119msec to complete).


## What is a missed collection?

A question might arise, like, what will happen if the collection time is longer than the configured sample interval? Such a situation is possible when you applied wrong sample interval (a very aggressive one), or when due to some reason collection/encoding/sending took more time than usual. Before going into explanation of behavior, let's see how one can check missed collections:

<div class="highlighter-rouge">
<pre class="highlight">
<code>

RP/0/RSP0/CPU0:ASR9k#sh telemetry model-driven subscription xrdocs-test <b>internal</b>
Tue Jan 15 21:01:00.141 PST 
Subscription:  xrdocs-test

<i>skipped</i>   

    Sensor Path:          Cisco-IOS-XR-wdsysmon-fd-oper:system-monitoring/cpu-utilization
 
      Sysdb Path:     /oper/wdsysmon_fd/gl/*
      Count:          59 Method: DATALIST Min: 19 ms Avg: 21 ms Max: 29 ms
      Item Count:     144
     <mark>Missed Collections:57</mark>  send bytes: 2048671 packets: 38 dropped bytes: 0
                      success         errors          deferred/drops 
      Gets            0               0              
      List            0               0              
      Datalist        39              0              
      Finddata        0               0              
      GetBulk         0               0              
      Encode                          0               0              
      Send                            0               0           

<i>skipped</i>

</code>
</pre>
</div>

The output from "show telemetry model-driven subscription <subs_id> <b>internal</b>" will give you a ton of information, including the number of missed collections. (We will explain each line of the output in one of our next posts!). 


### Missed collections with "sample-interval"

As it was described at the very beginning, for the collections without "strict-timer" configured, as soon as current collection stops, the system will start a new "sample-interval" timer countdown. 

![](https://github.com/xrdocs/xrdocs-images/blob/gh-pages/assets/tutorial-images/vosipchu/sample-intervals/missed-sample.png?raw=true)

If the collection (includes also encoding and sending) took more time than the "sample-interval" configured, then the IOS-XR router will skip sending <b>any</b> information out at this moment (but it will push the data out right when it is ready) and will update its internal "missed collections" counter (+1 missed collection, based on "sample-interval" timer). Right after the collection is done, a new sample interval will start its count down process. 

### Missed collections with "strict-timer"

"Strict-timer" adds more deterministic behavior to the process of pushing data out. And this stays the same with situations where collection time takes longer than the configured "sample-interval":

![](https://github.com/xrdocs/xrdocs-images/blob/gh-pages/assets/tutorial-images/vosipchu/sample-intervals/missed-strict.png?raw=true)
 
As soon as the configured "sample-interval" ends, the router will update its internal counter (+1 missed collection) and will <b>not</b> send anything out (only when information is ready). As with "sample-interval" case, it is very important that your router doesn't send stale information, or even a string of zeros, as it might affect your monitoring and/or automation tools. 
The difference with "sample-interval" case is that here the countdown for the next "sample-interval" will start right at the moment when the first timer ends. If you have a consistent behavior with long collection times, you should expect that the "window", between the moment when the previous collection stops and the start of the following collection, will be shrinking over time. 

## Conclusion

There is no "golden" rule on how to proceed with your configuration of sample intervals. There are telemetry customers that use each way. If you have collections that take several msec to collect the data, encode it and send to your collector, probably there is not much difference. If you have collections with big total time, then make sure your sampling is aligned with that and you don't have missed collections over time. A good point for "strict-timer" could be an easier way to implement some kind of automation, as you will have more deterministic behavior of your telemetry.
