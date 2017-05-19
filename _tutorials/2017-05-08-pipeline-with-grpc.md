---
published: true
date: '2017-05-08 11:03 -0600'
title: Pipeline with gRPC
author: Shelly Cadora
excerpt: Describes how to use Model-Driven Telemetry with Pipeline and gRPC
tags:
  - iosxr
  - telemetry
  - gRPC
  - MDT
  - pipeline
postion: hidden
---
In previous tutorials, I've shown how to use [Pipeline](http://blogs.cisco.com/sp/introducing-pipeline-a-model-driven-telemetry-collection-service) to dump Model Driven Telemetry (MDT) data into a [text file](https://xrdocs.github.io/telemetry/tutorials/2016-10-03-pipeline-to-text-tutorial/) and into [InfluxDB](https://xrdocs.github.io/telemetry/tutorials/2017-04-10-using-pipeline-integrating-with-influxdb/).  In each case, I configured the router to transport MDT data to Pipeline using TCP.  In this tutorial, I'll cover a few additional steps that are required to use Pipeline with [gRPC](http://www.grpc.io/).  I'll focus on only the changes needed in the router and Pipeline input stage configs here, so be sure to consult the other Pipeline tutorials for important info about install, output stage, etc.

If you're going to use gRPC, the first thing to decide is whether you're going to [dial out](#dialout) from the router or [dial in](#dialin) to the router.  

{% capture "output" %}
If you don't know the difference between dialin and dialout or need help chosing, check out [my blog](https://xrdocs.github.io/telemetry/blogs/2017-01-20-model-driven-telemetry-dial-in-or-dial-out/) for some guidance. 
{% endcapture %}
<div class="notice--warning">
{{ output | markdownify }}
</div>

Once you've made that decision, go the appropriate section of this tutorial.  

[DIALIN](#dialout) [DIALOUT](#dialin)
{: .text-justify}

For each section, there will be some "common" router and Pipeline config setps and well as some specific steps you need depending on whether or not you enable TLS.


# gRPC Dialout<a name="dialout"></a>
For gRPC Dialout, the subscription and sensor-group config are the same whether you use TLS or not, so I'll re-use those parts of the MDT router config from the [gRPC dialout example](https://xrdocs.github.io/telemetry/tutorials/2016-07-21-configuring-model-driven-telemetry-mdt/#grpc-dial-out).  It will look like this:

```
telemetry model-driven
 sensor-group SGroup2
  sensor-path Cisco-IOS-XR-nto-misc-oper:memory-summary/nodes/node/summary
 !
 subscription Sub2
  sensor-group-id SGroup2 sample-interval 30000
  destination-id DGroup2
``` 

Now the big decision is whether to use [TLS](#dialout-tls) or [not](#dialout-no-tls). This impacts the destination-group in the router config and the ingress stage of the Pipeline input stage as you'll see below.

## gRPC Dialout Without TLS<a name="dialout-no-tls"></a>
If you don't use TLS, your MDT data won't be encrypted.  On the other hand, it's easy to configure. So if you're new to MDT and gRPC, this might be a good starting place.

### Router Config: no-tls

The gRPC config for the router is contained in the MDT destination-group.  Here is a destination-group for gRPC dialout without TLS:

```
telemetry model-driven
 destination-group DGroup2
  address family ipv4 172.30.8.4 port 57500
   encoding self-describing-gpb
   protocol grpc no-tls
```

Add that to the subscription and sensor-path configuration in the [commmon router config above](#dialout) and your router config for gRPC dialout with TLS is done.

### Pipeline.conf: tls = false

You can use the ```[gRPCDIalout]``` input stage in the default pipeline.conf from github.  Just uncomment the 6 lines shown below.

```
$ grep -A25 "gRPCDialout" pipeline.conf | grep -v -e '^#' -e '^$'
[gRPCDialout]
 stage = xport_input
 type = grpc
 encap = gpb
 listen = :57500
 tls = false
```

If you now run pipeline with the debug option, you should see these lines when Pipeline starts and the router (at 172.30.8.53) connects:
```
$ bin/pipeline -config pipeline.conf -log= -debug | grep gRPC
INFO[2017-05-08 11:25:50.046573] gRPC starting block                           encap=gpb name=grpcdialout server=:57500 tag=pipeline type="pipeline is SERVER"
INFO[2017-05-08 11:25:50.046902] gRPC: Start accepting dialout sessions        encap=gpb name=grpcdialout server=:57500 tag=pipeline type="pipeline is SERVER"
INFO[2017-05-08 11:26:03.572534] gRPC: Receiving dialout stream                encap=gpb name=grpcdialout peer="172.30.8.53:61857" server=:57500 tag=pipeline type="pipeline is SERVER"
```

And that's it.  You're done.  Telemetry data is streaming into pipeline and you can do with it what you want.  You can stop reading now unless you want to experiment with TLS.

## gRPC Dialout With TLS<a name="dialout-tls"></a>
In a dialout scenario, the router is the "client" and Pipeline is the "server."  Therefore, in the TLS handshake, Pipeline will need to send a certificate to authenticate itself to the router.  The router validates Pipeline's certificate using the public certificate of the Root Certificate Authority (CA) that signed it and then generates sesssion keys to encrypt the session.

To make this all work, you need the following:
1. A Root CA certificate
2. A Pipeline certificate signed by the Root CA
3. A copy of the Root CA certificate on the router

For the purpose of this tutorial, I will use [openssl](https://www.openssl.org/) (an open-source TLS toolkit) for the root CA and Pipeline certificate.  If your organization has an existing PKI, you can skip the first two steps and just copy the Root CA certificate to the router.

### Certificates for TLS Dialout

#### 1. The rootCA Key and Certificate
For simplicity, I'll generate the rootCA on the same server that I am running Pipeline.  First, create a rootCA key-pair (may require sudo):
<div class="highlighter-rouge">
<pre class="highlight">
<code>
scadora@darcy:/etc/ssl/certs$ <b>openssl genrsa -out rootCA.key 2048</b>
Generating RSA private key, 2048 bit long modulus
...........+++
................................+++
e is 65537 (0x10001)
scadora@darcy:/etc/ssl/certs$
</code>
</pre>
</div>

Now use that key to self-sign the rootCA certificate.  It will ask you a bunch of questions that you can fill out as you want (I just used all defaults):
<div class="highlighter-rouge">
<pre class="highlight">
<code>
scadora@darcy:/etc/ssl/certs$ <b>sudo openssl req -x509 -new -nodes -key rootCA.key -sha256 -days 1024 -extensions v3_ca -config ../openssl.cnf -out rootCA.pem </b>
You are about to be asked to enter information that will be incorporated
into your certificate request.
What you are about to enter is what is called a Distinguished Name or a DN.
There are quite a few fields but you can leave some blank
For some fields there will be a default value,
If you enter '.', the field will be left blank.
-----
Country Name (2 letter code) [AU]:
State or Province Name (full name) [Some-State]:
Locality Name (eg, city) []:
Organization Name (eg, company) [Internet Widgits Pty Ltd]:
Organizational Unit Name (eg, section) []:
Common Name (e.g. server FQDN or YOUR name) []:
Email Address []:
scadora@darcy:/etc/ssl/certs$
</code>
</pre>
</div>

You should now have a rootCA certificate called rootCA.pem.

#### 2. The Pipeline Certificate<a name="pipelinecert"></a>
First, create a key pair for Pipeline.  In this case, I've called it "darcy.key" since darcy is the name of the server on which I am running Pipeline.

<div class="highlighter-rouge">
<pre class="highlight">
<code>
scadora@darcy:/etc/ssl/certs$  <b>sudo openssl genrsa -out darcy.key 2048</b>
Generating RSA private key, 2048 bit long modulus
................+++
..+++
e is 65537 (0x10001)
scadora@darcy:/etc/ssl/certs$
</code>
</pre>
</div>
Next, create a Certificate Signing Request (CSR) using the key you just generated.  In the following, I use all the defaults except for the Common Name, which I set as darcy.cisco.com:

<div class="highlighter-rouge">
<pre class="highlight">
<code>
scadora@darcy:/etc/ssl/certs$ <b>openssl req -new -key darcy.key -out darcy.csr</b>
You are about to be asked to enter information that will be incorporated
into your certificate request.
What you are about to enter is what is called a Distinguished Name or a DN.
There are quite a few fields but you can leave some blank
For some fields there will be a default value,
If you enter '.', the field will be left blank.
-----
Country Name (2 letter code) [AU]:
State or Province Name (full name) [Some-State]:
Locality Name (eg, city) []:
Organization Name (eg, company) [Internet Widgits Pty Ltd]:
Organizational Unit Name (eg, section) []:
Common Name (e.g. server FQDN or YOUR name) []:darcy.cisco.example
Email Address []:

Please enter the following 'extra' attributes
to be sent with your certificate request
A challenge password []:
An optional company name []:
scadora@darcy:/etc/ssl/certs$
</code>
</pre>
</div>

Finally, use your rootCA certificate to sign the CSR ("darcy.csr") you just generated and create a certificate for Pipeline ("darcy.pem"):

<div class="highlighter-rouge">
<pre class="highlight">
<code>
scadora@darcy:/etc/ssl/certs$ <b>openssl x509 -req -in darcy.csr -CA rootCA.pem -CAkey rootCA.key -CAcreateserial -out darcy.pem -days 500 -sha256</b>
Signature ok
subject=/C=AU/ST=Some-State/O=Internet Widgits Pty Ltd/CN=darcy.cisco.example
Getting CA Private Key
scadora@darcy:/etc/ssl/certs$
</code>
</pre>
</div>

{% capture "output" %}
Note: Some people issue certificates with a Common Name set to the IP address of the server instead of a FQDN.  Should you do this for the Pipeline certificate, bear in mind that the certificate will also need to have a Subject Alternative Name section that explicitly lists all valid IP addresses.  If you see the following message in your grpc trace, this could be your problem.

```RP/0/RP0/CPU0:SunC#show grpc trace ems
Tue May 16 19:35:44.792 UTC
3 wrapping entries (141632 possible, 320 allocated, 0 filtered, 3 total)
May 16 19:35:40.240 ems/grpc 0/RP0/CPU0 t26842 EMS-GRPC: grpc: Conn.resetTransport failed to create client transport: connection error: desc = "transport: x509: cannot validate certificate for 172.30.8.4 because it doesn't contain any IP SANs"
```

For more info on certificates with IP Addresses, take a look at [this discussion](https://serverfault.com/questions/611120/failed-tls-handshake-does-not-contain-any-ip-sans).
{% endcapture %}
<div class="notice--warning">
{{ output | markdownify }}
</div>

#### 3. Copy rootCA Certificate to the router

For the router to validate Pipeline's certificate, it needs to have a copy of the rootCA certificate in /misc/config/grpc/dialout/dialout.pem (the filename is important!).  Here is how to scp the rootCA.pem to the appropriate file and directory:
```
RP/0/RP0/CPU0:SunC#bash
Tue May 16 18:06:04.592 UTC
[xr-vm_node0_RP0_CPU0:~]$ scp scadora@172.30.8.4://etc/ssl/certs/rootCA.pem /misc/config/grpc/dialout/dialout.pem
rootCA.pem                                    100% 1204     1.2KB/s   00:00
[xr-vm_node0_RP0_CPU0:~]$
```

### Configuring the Router for gRPC Dialout with TLS
In addition to the common sensor-group and subscription configuration that we configured on the router at the beginning of the [Dialout section](dialout), we also need a destination-group. TLS is the default for MDT for gRPC, so the destination-group config just looks like this:

```
telemetry model-driven
 destination-group DGroup2
  address family ipv4 172.30.8.4 port 57500
   encoding self-describing-gpb
   protocol grpc 
```

### Configuring Pipeline for tls=true
We can use the ```[gRPCDIalout]``` input stage in the default pipeline.conf.  The only change is the last 4 lines where we enable tls and set the pem, key and servername to the values corresponding to the [Pipeline certificate we generated earlier](#pipelinecert). 

<div class="highlighter-rouge">
<pre class="highlight">
<code>
scadora@darcy:~/bigmuddy-network-telemetry-pipeline$ grep -A30 "gRPCDialout" pipeline.conf | grep -v -e '^#' -e '^$'
[gRPCDialout]
 stage = xport_input
 type = grpc
 encap = gpb
 listen = :57500
 <b>tls = true
 tls_pem = /etc/ssl/certs/darcy.pem
 tls_key = /etc/ssl/certs/darcy.key
 tls_servername = darcy.cisco.example</b>
</code>
</pre>
</div>

And that's it.  Run pipeline as usual and you'll see the router connect:

```
scadora@darcy:~/bigmuddy-network-telemetry-pipeline$ sudo bin/pipeline -config pipeline.conf -log= -debug | grep gRPC
INFO[2017-05-16 13:00:40.976896] gRPC starting block                           encap=gpb name=grpcdialout server=:57500 tag=pipeline type="pipeline is SERVER"
INFO[2017-05-16 13:00:40.977505] gRPC: Start accepting dialout sessions        encap=gpb name=grpcdialout server=:57500 tag=pipeline type="pipeline is SERVER"
INFO[2017-05-16 13:01:09.775514] gRPC: Receiving dialout stream                encap=gpb name=grpcdialout peer="172.30.8.53:59865" server=:57500 tag=pipeline type="pipeline is SERVER"
```

On the router, the grpc trace will show you the server name (darcy.cisco.example) of the received certificate.

```
RP/0/RP0/CPU0:SunC#show grpc trace ems
Tue May 16 20:01:15.059 UTC
2 wrapping entries (141632 possible, 320 allocated, 0 filtered, 2 total)
May 16 20:01:06.757 ems/conf 0/RP0/CPU0 t26859 EMS-CONF:emsd_is_active_role get proc role (1)
May 16 20:01:06.759 ems/info 0/RP0/CPU0 t26843 EMS_INFO: nsDialerCheckAddTLSOption:322 mdtDialout: TLS pem: /misc/config/grpc/dialout/dialout.pem, Server host: darcy.cisco.example
RP/0/RP0/CPU0:SunC#
```

That's it.  You're done with gRPC Dialout with TLS.  No need to read further.

# gRPC Dialin<a name="dialin"></a>

In a dialin scenario, Pipeline sends the TCP SYN packet and acts as the "client" in the gRPC session and TLS handshake (if you're configuring TLS).

## Common Router Config for gRPC DialIn<a name="router-dialin"></a>
For this part of the tutorial, I'll re-use the MDT router config from the [gRPC dialin example](https://xrdocs.github.io/telemetry/tutorials/2016-07-21-configuring-model-driven-telemetry-mdt/#grpc-dial-in)  It should look like this:

```
grpc
 port 57500
!
telemetry model-driven
 sensor-group SGroup3
  sensor-path openconfig-interfaces:interfaces/interface
 !
 subscription Sub3
  sensor-group-id SGroup3 sample-interval 30000
``` 

Note that there is no destination-group for dialin.  

## Common Dialin Credentials<a name="router-creds"></a> 
Regardless of whether you use TLS or not, Pipeline will have to provide a username and password when it first connects to the router. 

On the router side, you need to configure a username and password that Pipeline can use when it dials in.  If you're just doing a quick test in the lab, assign the user to one of these default usergroups: sysadmin, netadmin, or root-lr.  For example:

```
username mdt
 group sysadmin
 secret 5 $1$kAbv$xNk9KA.mIC7K2wfdpGjzk1
```

If you want to be more restrictive, here is the minimal taskgroup that you will need to support telemetry for gRPC dialin:

```
taskgroup mdt-grpc
 task read li
 task read acl
 task read cdp
 task read eem
 task read boot
 task read diag
 task read ipv4
 task read ipv6
 task read snmp
 task read vpdn
 task read crypto
 task read system
 task read logging
 task read fault-mgr
 task read interface
 task read ext-access
 task read filesystem
 task read tty-access
 task read config-mgmt
 task read ip-services
 task read host-services
 task read basic-services
 task read config-services
!
usergroup mdt-grpc
 taskgroup mdt-grpc
!
username mdt
 group mdt-grpc
 secret 5 $1$kAbv$xNk9KA.mIC7K2wfdpGjzk1
```

Next, you get to decide if you want to use [TLS](#grpc-in-tls) or [not](#grpc-in-no-tls).

### gRPC Dialin Without TLS<a name="grpc-in-no-tls"></a>

If you don't use TLS, your MDT data won't be encrypted.  On the other hand, there's less fiddling with certificates. So if you're trying to get gRPC dialin to work for the first time, this might be a good starting place. There's nothing you need to add to the router config for this beyond the [common router config](#router-dialin) and [credentials](#router-creds) we did above. You just need to configure and run Pipeline as shown below. 

 You can use the ```[mymdtrouter]``` input stage in the default pipeline.conf.  Just uncomment the 8 lines shown below, changing the server line to match your router's IP address and configured gRPC port:

<div class="highlighter-rouge">
<pre class="highlight">
<code>
$ grep -A48 "mymdtrouter" pipeline.conf | grep -v -e '^#' -e '^$'
 [mymdtrouter]
 stage = xport_input
 type = grpc
 encoding = gpbkv
 encap = gpb
 <b>server = 172.30.8.53:57500</b>
 subscriptions = Sub3
 tls = false
</code>
</pre>
</div>

Note that the subscription is "Sub3", which matches the subscription in the router configuration [above](#router-dialin).  Pipeline will request the pre-configured subscription from the router when it connects.

When you run pipeline, you will be prompted for a username and password.  This is the username and password that you configured on the router [above](#router-creds).

```
$ bin/pipeline -config pipeline.conf
Startup pipeline
Load config from [pipeline.conf], logging in [pipeline.log]

CRYPT Client [mymdtrouter],[172.30.8.53:57500]
 Enter username: mdt
 Enter password:
Wait for ^C to shutdown
```

If you don't want to have to manually enter the username and password each time you run Pipeline, check out the section below on [secure password storage in pipeline](#secure-passwords).
{: .notice--warning}

To verify that the connection is established, check that the subscription Destination Group State is Active. Also note that the Destination Group Id has been dynamically created (since we don't configure a destination-group on the router for dialin) and begins with "DialIn_."

<div class="highlighter-rouge">
<pre class="highlight">
<code>
RP/0/RP0/CPU0:SunC#show telemetry model sub Sub3
Thu May 18 20:46:38.658 UTC
Subscription:  Sub3
-------------
  State:       ACTIVE
  Sensor groups:
  Id: SGroup3
    Sample Interval:      30000 ms
    Sensor Path:          openconfig-interfaces:interfaces/interface
    Sensor Path State:    Resolved

  Destination Groups:
  <mark>Group Id: DialIn_1019</mark>
    Destination IP:       172.30.8.4
    Destination Port:     48667
    Encoding:             self-describing-gpb
    Transport:            dialin
    <mark>State:                Active</mark>
    No TLS
    Total bytes sent:     5723
    Total packets sent:   4
    Last Sent time:       2017-05-18 20:46:28.2143492698 +0000
</code>
</pre>
</div>

That's it, you're done.  No need to read the next section unless you want to do TLS.  

### gRPC Dialin With TLS<a name="grpc-in-tls"></a>
In a dialin scenario, Pipeline acts as the "client" in the TLS handshake.  Therefore, the router will need to send a certificate to authenticate itself to Pipeline. 

There are a couple ways to go about creating the router certificate. If you already have a root CA, you can issue a certificate for the router.  However, because this tutorial is far too long already, I'm going to take the easy way out and use a self-signed certificate.

#### Router Certificate and Config for gRPC TLS DialIn<a name="router-dialin-tls"></a>

The first thing we have to do is enable the gRPC service on the router for TLS by adding "tls" to the grpc config on the router:

```
grpc
 port 57500
 tls
```

Once you do this, the router automatically generates a self-signed cert called "ems.pem":
<div class="highlighter-rouge">
<pre class="highlight">
<code>
RP/0/RP0/CPU0:SunC#bash
Thu May 18 23:05:51.266 UTC

[xr-vm_node0_RP0_CPU0:~]$cd /misc/config/grpc
[xr-vm_node0_RP0_CPU0:/misc/config/grpc]$ls
dialout  ems.key  <mark>ems.pem</mark>
[xr-vm_node0_RP0_CPU0:/misc/config/grpc]$
</code>
</pre>
</div>

You can use standard openssl commands to view the cert:

<div class="highlighter-rouge">
<pre class="highlight">
<code>
[xr-vm_node0_RP0_CPU0:/misc/config/grpc]$<b>openssl x509 -noout -text -in ems.pem</b>
Certificate:
    Data:
        Version: 3 (0x2)
        Serial Number: 1789 (0x6fd)
    Signature Algorithm: sha512WithRSAEncryption
        Issuer: C=US, ST=CA, L=San Jose/street=3700 Cisco Way/postalCode=95134, O=Cisco Systems, Inc., OU=CSG, CN=ems.cisco.com/serialNumber=949DF85F746
        Validity
            Not Before: May 18 22:49:51 2017 GMT
            Not After : May 18 22:49:51 2037 GMT
        Subject: C=US, ST=CA, L=San Jose/street=3700 Cisco Way/postalCode=95134, O=Cisco Systems, Inc., OU=CSG, <mark>CN=ems.cisco.com</mark>/serialNumber=949DF85F746
        Subject Public Key Info:
            Public Key Algorithm: rsaEncryption
                Public-Key: (2048 bit)
                Modulus:
                    00:cf:89:9e:7a:14:5f:6a:f4:8a:75:ce:69:07:00:
                    38:2d:5d:1f:71:f5:cb:69:37:3b:6d:9b:20:ab:47:
                    9e:2b:b6:4b:be:30:1e:54:81:76:4f:61:91:de:4e:
                    47:80:b2:6d:0c:f3:2a:69:be:85:67:ca:a3:80:7f:
                    bc:40:2e:63:5d:c9:ec:a4:fc:60:ae:b2:10:2a:f9:
                    de:02:11:50:5a:1e:43:c9:3a:95:6b:9f:fa:3d:f4:
                    db:1f:a9:6d:bd:7b:0b:d8:87:64:08:26:2b:54:82:
                    42:2f:a2:7e:36:64:9b:42:9a:ff:bf:19:25:2f:42:
                    3e:9d:94:af:fc:ea:62:ef:ec:57:20:57:d9:39:c5:
                    bd:77:5c:a9:01:76:e1:2c:69:67:6f:b7:30:f8:f8:
                    2c:d1:2c:25:de:66:46:fb:49:30:a7:c9:9c:14:b0:
                    70:f4:3f:b2:62:8c:5c:c6:8f:a2:e3:de:75:c3:c3:
                    e5:72:1f:4e:40:d4:bd:1b:2a:27:19:e7:80:b3:c9:
                    cb:56:4e:5c:99:42:d6:97:23:04:6d:9c:9e:f0:d2:
                    0e:8b:5c:02:09:d1:c8:31:04:23:b4:f1:b4:41:a2:
                    44:b5:16:fd:c3:80:a5:3d:39:26:de:94:2b:db:22:
                    d0:0b:07:92:2d:6a:24:37:d4:db:b2:29:23:f3:00:
                    88:13
                Exponent: 65537 (0x10001)
        X509v3 extensions:
            X509v3 Basic Constraints: critical
            <mark>CA:TRUE</mark>
            X509v3 Subject Key Identifier:
                1C:B6:98:EF:7F:A4:1D:07:0B:6F:73:01:08:E6:0C:8C:97:AC:E0:A2
            X509v3 Authority Key Identifier:
                keyid:1C:B6:98:EF:7F:A4:1D:07:0B:6F:73:01:08:E6:0C:8C:97:AC:E0:A2

    Signature Algorithm: sha512WithRSAEncryption
         60:97:b9:e2:cb:d5:1d:b0:48:d5:68:fa:aa:a6:36:de:e3:64:
         1f:6a:7f:4b:3e:9c:42:e8:59:23:26:14:c1:b1:0e:f3:17:d5:
         34:71:4c:79:6f:f7:62:94:21:a2:d7:d4:99:cc:9c:f2:29:2a:
         38:79:19:83:fd:a9:16:df:0e:35:55:5e:11:b5:b7:3f:e6:10:
         0d:71:c7:3d:2d:9b:41:44:09:9b:b3:98:64:ab:9e:33:f3:08:
         a9:f0:6b:62:93:18:7e:ff:14:a7:ea:c2:c3:3b:ed:a6:b3:69:
         25:07:04:41:23:82:c6:12:23:6d:e0:14:80:7c:10:dd:ea:06:
         8e:e6:78:f5:42:a0:3e:21:81:7d:48:29:18:29:0a:ef:ce:a1:
         7c:38:7b:e8:17:44:db:24:37:ba:1c:53:6d:9d:6f:d2:5c:2a:
         69:b5:11:13:4d:7c:cc:3d:44:d2:96:fa:71:41:3a:b6:ab:6e:
         e7:b1:ff:53:db:e8:95:5c:67:68:51:80:ab:24:e0:7e:8e:fe:
         e1:af:36:8c:bc:b2:3a:69:3f:33:bc:b6:36:25:ad:78:49:d1:
         2e:43:6f:f8:80:c3:1c:21:89:cd:da:9f:3d:62:ec:79:1b:b0:
         77:0d:96:c8:c8:26:25:0b:94:ae:21:14:d1:1b:e0:f7:11:af:
         61:ce:13:74
[xr-vm_node0_RP0_CPU0:/misc/config/grpc]$
</code>
</pre>
</div>

As you can see, this certificate has been issued for a CN=ems.cisco.com and is a CA certificate.

Since this is also the CA cert (it's self-signed), we'll transfer it to the server running Pipeline.<a name="copy-router-cert"></a>

```
[xr-vm_node0_RP0_CPU0:/misc/config/grpc]$ scp ems.pem scadora@172.30.8.4:
scadora@172.30.8.4's password:
ems.pem                                       100% 1513     1.5KB/s   00:00    
[xr-vm_node0_RP0_CPU0:/misc/config/grpc]$
```

#### Pipeline for gRPC Dialin with TLS

All that's left is to configure pipeline.conf for TLS.

You can use the ```[mymdtrouter]``` input stage in the default pipeline.conf.  Uncomment the 10 lines shown below and do the following:

- change the server line to match your router's IP address and configured gRPC port.
- set tls to "true"
- set tls_pem to the full path and filename of the ems.pem file you copied from the router [above](#copy-router-cert).
- set tls_pem to the CN of the router's certificate ("ems.cisco.com")

<div class="highlighter-rouge">
<pre class="highlight">
<code>
$ grep -A48 "mymdtrouter" pipeline.conf | grep -v -e '^#' -e '^$'
[mymdtrouter]
 stage = xport_input
 type = grpc
 encoding = gpbkv
 encap = gpb
 <b>server = 172.30.8.53:57500</b>
 subscriptions = Sub3
 <b>tls = true
 tls_pem = /home/scadora/ems.pem
 tls_servername = ems.cisco.com</b>
</code>
</pre>
</div>

Note that the subscription is "Sub3", which matches the subscription in the router configuration [above](#router-dialin).  Pipeline will request the pre-configured subscription from the router when it connects.

When you run pipeline, you will be prompted for a username and password.  This is the username and password that you configured on the router [above](#router-creds).

```
$ bin/pipeline -config pipeline.conf
Startup pipeline
Load config from [pipeline.conf], logging in [pipeline.log]

CRYPT Client [mymdtrouter],[172.30.8.53:57500]
 Enter username: mdt
 Enter password:
Wait for ^C to shutdown
```

If you don't want to have to manually enter the username and password each time you run Pipeline, check out the section below on [secure password storage in pipeline](#secure-passwords).
{: .notice--warning}

To verify that the connection is established, check that the subscription Destination Group State is Active. Also note that the Destination Group Id has been dynamically created (since we don't configure a destination-group on the router for dialin) and beings with "DialIn_."

<div class="highlighter-rouge">
<pre class="highlight">
<code>
RP/0/RP0/CPU0:SunC#show telemetry model-driven subscription Sub3
Fri May 19 16:48:22.396 UTC
Subscription:  Sub3
-------------
  State:       ACTIVE
  Sensor groups:
  Id: SGroup3
    Sample Interval:      30000 ms
    Sensor Path:          openconfig-interfaces:interfaces/interface
    Sensor Path State:    Resolved

  Destination Groups:
  <mark>Group Id: DialIn_1030</mark>
    Destination IP:       172.30.8.4
    Destination Port:     57590
    Encoding:             self-describing-gpb
    Transport:            dialin
    <mark>State:                Active</mark>
    No TLS
    Total bytes sent:     11446
    Total packets sent:   8
    Last Sent time:       2017-05-19 16:48:12.1233215666 +0000
</code>
</pre>
</div>

That's it, you're done.  Have fun with your telemetry data!


### Appendix: Secure Password Storage for Dialin<a name="secure-passwords"></a>
Because Pipeline cares about your security, it won't let you store unencrypted router passwords in pipeline.conf.  If you dislike being prompted for a password every time you run it or you want to run pipeline in the background, you can have pipeline encrypt the password using the -pem option and store it in a new file as follows:

```
$ bin/pipeline -config pipeline.conf -pem ~/.ssh/id_rsa
Startup pipeline
Load config from [pipeline.conf], logging in [pipeline.log]

CRYPT Client [mymdtrouter],[172.30.8.53:57500]
 Enter username: mdt
 Enter password:
Generating sample config...A new configuration file [pipeline.conf_REWRITTEN] has been written including user name and encrypted password.
In future, you can run pipeline non-interactively.
Do remember to run pipeline with '-pem /home/scadora/.ssh/id_rsa -config pipeline.conf_REWRITTEN' options.
Wait for ^C to shutdown
```

If you take a look at the ```[mymdtrouter]``` stage in pipeline.conf_REWRITTEN, you'll see that the username and encrypted password are included:

```
$ grep -A10 "mymdtrouter" pipeline.conf_REWRITTEN | grep -v -e '^#' -e '^$'
[mymdtrouter]
subscriptions=Sub3
password=PZbS/IG4O+2lsok3xxBjQZwJ5CFyraixl//qdNy67IRMM1YMLlWqbbGHUXVGM1pX0HfKf7JU1beRivkOcwyANPff4hVmF5b7Ne1SBxnKS4VqSU+AMCN/e+FFHFrCA24m0ywTYB/Dt2PJZaUCQmYzxTwa71+Vxc7lHe2dtovH/DGutQfvRa2On6aHeqiQfMbBcEeKqwya4jtmexS11Dt1ai1QXqWgn2WiggvWTGcldANO4Nfkl4vICguVlrVEfNv16qNoPB/HerTNCuGLlBR0EBhxGPxCJteexAxadt68whG4UP/teTiD2qFZ2UFXCRnpnPvpic9LIZIaF4PgNg9AGw==
server=172.30.8.53:57500
type=grpc
encoding=gpbkv
encap=gpb
tls=false
username=mdt
stage=xport_input
```

Now you can run pipeline with the rewritten .conf file and it won't prompt you for the username again:

```
$ sudo bin/pipeline -config pipeline.conf_REWRITTEN -pem ~/.ssh/id_rsa
Startup pipeline
Load config from [pipeline.conf_REWRITTEN], logging in [pipeline.log]
Wait for ^C to shutdown
```
