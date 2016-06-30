---
published: false
date: '2016-06-30 20:40 +0800'
title: Copying Telemetry Policy Files in Vagrant IOS XRv 6.1.1
author: Richard Wade
excerpt: >-
  How to copy IOS XR policy files from the host Linux filesystem to the IOS XRv
  image running in a Vagrant box.
tags:
  - vagrant
  - iosxr
  - linux
  - Telemetry
---
This tutorial describes how to copy IOS XR policy files from your host filesystem to the IOS XR filesystem, where the target is IOS XRv running as a Vagrant box. Firstly, we should check which ports we should use for interaction with the IOS XRv Vagrant box. We do this using vagrant in the host operating system.

<div class="highlighter-rouge">
<pre class="highlight">
<code>
user@host:~/vagrant/iosxrv> <mark>vagrant port</mark>
The forwarded ports for the machine are listed below. Please note that these values may differ from values configured in the Vagrantfile if the provider supports automatic port collision detection and resolution.

    22 (guest) => 2223 (host)
    57722 (guest) => 2222 (host)
</code>
</pre>
</div>

We can see that port 22 of the IOS XRv instance has been mapped to host port 2223. However, we need to interact with the IOS XR Linux operating system, which has SSH listening on port 57722. Vagrant has mapped port 57722 to port 2222 of the host operating system. We therefore need to copy our policy file to port 2222 as follows:

<div class="highlighter-rouge">
<pre class="highlight">
<code>
user@host:~/vagrant/iosxrv⟫ ls -l Test.policy 
-rw-rw-r-- 1 user group 412 Jun 29 18:22 Test.policy
user@host:~/vagrant/iosxrv⟫ <mark>scp -P 2222 Test.policy vagrant@localhost:/misc/app_host/scratch/</mark>
vagrant@localhost's password: 
Test.policy                                                  100%  412     0.4KB/s   00:00  
</code>
</pre>
</div>

Due to filesystem security policy in IOS XR, incoming files must be copied to a directory which is writable by the vagrant user. /misc/app_host/scratch is suitable for this and is writable by the vagrant user. In order to install the policy file in IOS XR's running configuration, we must copy it to the telemetry policy directory at /telemetry/policies/:

<div class="highlighter-rouge">
<pre class="highlight">
<code>
user@host:~/vagrant/iosxrv> <mark>ssh -p 2223 vagrant@localhost</mark>
vagrant@localhost's password: 

RP/0/RP0/CPU0:ios#run
Wed Jun 29 08:37:35.302 UTC
[xr-vm_node0_RP0_CPU0:~]$<mark>cd /misc/app_host/scratch/</mark>
[xr-vm_node0_RP0_CPU0:/misc/app_host/scratch]$ls -l
total 4
-rw-r--r-- 1 vagrant vagrant 412 Jun 29 08:23 Test.policy
[xr-vm_node0_RP0_CPU0:/misc/app_host/scratch]$<mark>cp Test.policy /telemetry/policies/</mark>
[xr-vm_node0_RP0_CPU0:/misc/app_host/scratch]$ls -l /telemetry/policies/
total 4
-rwxr-xr-x 1 root root 412 Jun 29 08:37 Test.policy
[xr-vm_node0_RP0_CPU0:/misc/app_host/scratch]$
</code>
</pre>
</div>

Now, if we return to the IOS XR command line interface and show the configured telemetry policies, we see that the uploaded policy has been installed:

<div class="highlighter-rouge">
<pre class="highlight">
<code>
RP/0/RP0/CPU0:ios#<mark>show telemetry policies</mark>
Wed Jun 29 08:44:37.055 UTC

cpu
  Filename:             None
  Status:               Inactive

memory
  Filename:             None
  Status:               Inactive

robot
  Filename:             None
  Status:               Inactive

Test
  Filename:             Test.policy
  Version:              25
  Description:          This is a sample policy
  Status:               Inactive
  CollectionGroup: FirstGroup
    Cadence:              30.0s
    Total collections:    0
    Latest collection:    Never
    Min total time:       0.000s
    Max total time:       0.000s
    Avg total time:       0.000s
    Collection errors:    0
    Missed collections:   0
    +----------------------------------------------+---------+---------+------+
    | Path                                         | Avg (s) | Max (s) | Err  |
    +----------------------------------------------+---------+---------+------+
    | RootOper.InfraStatistics.Interface(*).Latest |   0.000 |   0.000 |    0 |
    +----------------------------------------------+---------+---------+------+


RP/0/RP0/CPU0:ios#
</code>
</pre>
</div>
