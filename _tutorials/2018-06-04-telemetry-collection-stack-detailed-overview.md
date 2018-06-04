---
published: true
date: '2018-06-04 16:13 -0700'
title: Telemetry Collection Stack Detailed Overview
author: Viktor Osipchuk
position: hidden
excerpt: >-
  Detailed explanation of the script that installs and configures the IOS XR
  Model Driven Telemetry Stack
tags:
  - iosxr
  - cisco
  - MDT
  - Telemetry
  - Pipeline
  - InfluxDB
  - Grafana
  - Kapacitor
---
{% include toc icon="table" title="IOS XR Telemetry Collection Stack Detailed Overview" %}
{% include base_path %}

In our [previous tutorial](URL) we provided a high-level overview of the IOS XR Telemetry Collection Stack. The goal of the stack is to help engineers to quickly start testing telemetry and not to waste time to research on how to integrate an application A with application B and application C.

While working on this script, attention was given to the fact that people with a very different level of Linux/programming knowledge will use that script. That's why description was added for almost every step there. It might seem not very elegant, but I think it is essential for every person to understand how it works and what will run on his/her server. Also, the decision was to use a single language, with no overcomplication, and not to jump between different tools, as this, again, could bring unnecessary confusion for people who want to understand how it works (obviously, this task could be done in many different ways)

The primary goal of this tutorial is to go through main scripts used in the Telemetry Collection Stack and describe the flow. Hope, this can help people just starting their automation journey.

## The initial script to run

Before jumping to the main code, we should have a look at the initial script used as the [first step](https://github.com/vosipchu/XR_TCS/blob/master/init.sh). The main purpose of this initial script is to add your username to ["/etc/sudoers"](https://www.digitalocean.com/community/tutorials/how-to-edit-the-sudoers-file-on-ubuntu-and-centos) file, as the first step. It will make sure you can run "sudo" without entering the password. Last two lines make sure you can start Pipeline (will be configured in the main script) without entering login/password for InfluxDB (the system will insert credentials automatically every time you start Pipeline).
Throughout all the scripts you will see many "echo" commands. They will print some text on your screen while the code is running. "echo" is enhanced with specific codes to print a colored letter or print a colored background of that line. For example, **echo -e "\e[1;46m IOS XR MDT is awesome \e[0m"** will print text on a green background.

<div class="highlighter-rouge">
<pre class="highlight">
<code>
#!/bin/bash
echo -e "\e[1;46m This is a short script to prep your system for the main one \e[0m";
echo "$USER ALL=(ALL) NOPASSWD: ALL" | sudo EDITOR='tee -a' visudo
ssh-keygen -t rsa -N "" -f ~/.ssh/id_rsa
cp ~/IOSXR-Telemetry-Collection-Stack/Pipeline/id_rsa ~/.ssh/id_rsa
</code>
</pre>
</div>

## The main script to build up the stack

The main code to go through will be the one used [to build the collection stack](https://github.com/vosipchu/XR_TCS/blob/master/IOS-XR-Telemetry-BuildUP-stack.sh). The whole script is divided into several logical segments, let's cover it step by step.
The script starts with collecting all your inputs from the document with variables you need to fill (("BuildUP-help.doc")[https://github.com/vosipchu/XR_TCS/blob/master/BuildUP-help.doc]). A basic check is added to make sure you filled the document with variables correctly. The script then copies your values into variables with meaningful names, to be used throughout the whole script (it is possible to skip assigning of a new variable with a meaningful name, but it was done for better understanding while going through the code).

```
################################################################################
########             Telemetry Collection Build UP script               ########
################################################################################
## This is the script for PIKGP stack installation
## Every step will be described in details
## Feel free to modify it for your personal needs
echo -e "\e[1;46m Welcome to IOS XR MDT collection stack installation script! \e[0m";
echo -e "\e[1;46m It will generate the updates about the progress \e[0m";
echo

## This will insert environment variables you added in the text file
VARS=()
## Going through the variables to add them to a list to be used later
while read STRING; do VARS+=($STRING); done
## A very basic check for following the rules
if [ "${#VARS[*]}"" != 28 ]; then
  echo -e "\e[1;31m Looks like the document with variables was filled with mistakes \e[0m"
  echo -e "\e[1;31m check and run again! \e[0m"
  exit
else
  :
fi
## Creating variables with meaningful names
PROXY=${VARS[1]}
NTP1=${VARS[3]}
NTP2=${VARS[5]}
NTP3=${VARS[7]}
MDT_DURATION=${VARS[9]}
MDT_SHARD=${VARS[11]}
TELEGRAF_DURATION=${VARS[13]}
TELEGRAF_SHARD=${VARS[15]}
SNMP_ROUTER=${VARS[17]}
SNMP_COMMUNITY=${VARS[19]}
SERVER_IP=${VARS[21]}
SLACK_TOKEN=${VARS[23]}
SLACK_CHANNEL=${VARS[25]}
SLACK_USERNAME=${VARS[27]}
GRAFANA_USER=${VARS[29]}
GRAFANA_PASSWORD=${VARS[31]}
```

Next three sections deal with proxy configurations. The script can work behind a proxy, as well as in environments without proxies installed. If you do not have any proxy servers within your environment, you need to comment out [the corresponding line](https://github.com/vosipchu/XR_TCS/blob/master/BuildUP-help.doc#L2) in the helper document (add '#' at the beginning). In that case, all three sections will just be skipped. If not, the script will modify your current session, as well as modify /etc/environment and /etc/apt/apt.conf files to have your proxy configuration active with every next login.

```
################################################################################
########    This section adds proxy config for the current session      ########
################################################################################
if [[ ${PROXY:0:1} == "#" ]]
then
  :
else
  echo -e "\e[1;46m Adding PROXY information \e[0m";
  EXPORT=(http_proxy https_proxy ftp_proxy HTTP_PROXY HTTPS_PROXY FTP_PROXY);
  for i in "${EXPORT[@]}"; do export $i=$PROXY; done
  export no_proxy='localhost,127.0.0.1,localaddress,.localdomain.com'
  export NO_PROXY='localhost,127.0.0.1,localaddress,.localdomain.com'
fi
################################################################################


################################################################################
########       This section adds proxy config for all next logins       ########
################################################################################
if [[ ${PROXY:0:1} == "#" ]]
then
  :
else
  SERVICE=(http_proxy https_proxy ftp_proxy HTTP_PROXY HTTPS_PROXY FTP_PROXY);
  for i in "${SERVICE[@]}"; do echo $i="'$PROXY'" >> /etc/environment; done
  echo 'no_proxy="localhost,127.0.0.1,localaddress,.localdomain.com"' >> /etc/environment;
  echo 'NO_PROXY="localhost,127.0.0.1,localaddress,.localdomain.com"' >> /etc/environment;
fi
################################################################################


################################################################################
########          This section adds proxy config for apt-get            ########
################################################################################
if [[ ${PROXY:0:1} == "#" ]]
then
  :
else
  echo "Acquire::https::Proxy \"$PROXY\";" >> /etc/apt/apt.conf
  sleep 1
  echo -e "\e[1;45m PROXY information was added \e[0m";
fi
################################################################################
```

The next step for the script is to [install a package management system and several libraries](https://github.com/vosipchu/XR_TCS/blob/master/IOS-XR-Telemetry-BuildUP-stack.sh#L88-L103) it will need later on. ["pip"](https://en.wikipedia.org/wiki/Pip_(package_manager)), ["requests"](https://en.wikipedia.org/wiki/Requests_(software)), ["flask"](https://en.wikipedia.org/wiki/Flask_(web_framework)) will be needed for Kapacitor. ["screen"](https://www.poftut.com/linux-screen-tutorial-examples/) will be used together with Pipeline. ["wget"](https://en.wikipedia.org/wiki/Wget) and ["cURL"](https://en.wikipedia.org/wiki/CURL) might or might not be installed initially in your Ubuntu, hence, they are added here just in case. Every step will be printed on your screen.
Specific versions of the libraries will be installed to prevent issues with incompatibilities.

```
################################################################################
########   This section adds helper tools that will be needed later     ########
################################################################################
echo -e "\e[1;32m Installing PIP \e[0m";
apt-get -y install python-pip=8.1.1-2ubuntu0.4
echo -e "\e[1;32m Installing Requests \e[0m";
pip install requests==2.18.4
echo -e "\e[1;32m Installing Flask \e[0m";
pip install flask==1.0.2
echo -e "\e[1;32m Installing screen \e[0m";
apt-get -y install screen=4.3.1-2build1
echo -e "\e[1;32m Installing wget \e[0m";
apt-get -y install wget
echo -e "\e[1;32m Installing cURL \e[0m";
apt-get -y install curl
################################################################################
```

For your convenience the configuration for IOS XR routers is also [provided](https://github.com/vosipchu/XR_TCS/tree/master/Routers).
Initially, it has some generic IP destination address. The script will modify that address to the one your provided (you will need just copy/paste the ready configuration).
The change of the IP address is done using [SED utility](https://linuxconfig.org/learning-linux-commands-sed). The code will search for the generic IP address and change it to the one you provided in the document with variables.

```
################################################################################
########   This section updates Telemetry Destination in MDT configs    ########
################################################################################
echo -e "\e[1;32m Updating Destination in the MDT Configs \e[0m";
sed -i "s/1.2.3.4/$SERVER_IP/" ~/IOSXR-Telemetry-Collection-Stack/Routers/gRPC-ASR9k.config
sed -i "s/1.2.3.4/$SERVER_IP/" ~/IOSXR-Telemetry-Collection-Stack/Routers/gRPC-NCS5500.config
sed -i "s/1.2.3.4/$SERVER_IP/" ~/IOSXR-Telemetry-Collection-Stack/Routers/TCP-ASR9k.config
sed -i "s/1.2.3.4/$SERVER_IP/" ~/IOSXR-Telemetry-Collection-Stack/Routers/TCP-NCS5500.config
################################################################################
```

It is very important to have your time in sync between your routers and the Telemetry Collection Stack. Every telemetry packet will have a timestamp added. InfluxDB will use that timestamp, and when you query the database with Grafana, you will have to specify a range. If you have different time set between the router and the Collection Stack, the data will be stored, but you might not see it in Grafana. It is assumed you have NTP configured on your router. [This part of the code](https://github.com/vosipchu/XR_TCS/blob/master/IOS-XR-Telemetry-BuildUP-stack.sh#L115-L129) adds NTP configuration to the Collection Stack. It will use servers you specified in the [document with variables](https://github.com/vosipchu/XR_TCS/blob/master/BuildUP-help.doc#L3-L8).

```
################################################################################
########     This section configures the NTP service on your server     ########
################################################################################
echo -e "\e[1;46m Configuring NTP service \e[0m";
apt-get -y install ntp=1:4.2.8p4+dfsg-3ubuntu5.8
## Searching for default NTP servers, to comment them out
sleep 2
sed -i 's/pool \([0-3]\|ntp\)/#&/g' /etc/ntp.conf
## Now adding your servers
echo server $NTP1 iburst >> /etc/ntp.conf
echo server $NTP2 iburst >> /etc/ntp.conf
echo server $NTP3 iburst >> /etc/ntp.conf
systemctl restart ntp.service
echo -e "\e[1;45m NTP is installed, check the state later with 'ntpq -p' \e[0m"
################################################################################
```

The first component from the Collection Stack [to be installed](https://github.com/vosipchu/XR_TCS/blob/master/IOS-XR-Telemetry-BuildUP-stack.sh#L132-L202) is InfluxDB.
The code will create a directory (folder) and download the InfluxDB package. After that, the code will check whether the MD5 hash of the file is correct before it proceeds (and will exit if something is wrong). When the MD5 hash is checked, the code will modify InfluxDB configuration file. The primary purpose for the configuration file change is to remove different kind of logging (queries, HTTP, log messages as we don't need all of it), and to disable reporting (sending internal InfluxDB data from your server to the InfluxDB team). SED is used to make changes in the configuration file.
After making the changes, the code will start InfluxDB up and check the state. If it is running, the code will proceed to create databases for Telemetry data ("mdt_db") and one more for SNMP data ("telegraf").
After confirming that databases were created and running, the code will move to the next step.
Creation and checking of databases done using InfluxDB commands sent with cURL.

```
################################################################################
########        This section configures InfluxDB on your server         ########
################################################################################
echo -e "\e[1;46m Configuring InfluxDB, the database for Telemetry \e[0m";
## Create a directory to store data
mkdir -p ~/analytics/influxdb && cd ~/analytics/influxdb/
## Download the package for installation
echo -e "\e[1;32m Downloading InfluxDB package \e[0m";
wget https://dl.influxdata.com/influxdb/releases/influxdb_1.5.1_amd64.deb
## Check the MD5 hash from the download
MD5_Influx=`md5sum -c <<<"5ba6c50dc917dd55ceaef4762d1f876f *influxdb_1.5.1_amd64.deb"`
## A basic notification if things went wrong
MD5_Influx_RESULT=$(echo $MD5_Influx | awk '{ print $2 }' )
if [ "$MD5_Influx_RESULT" == "OK" ];
  then
    echo -e "\e[1;32m MD5 of the file is fine, moving on \e[0m";
  else
    echo -e "\e[1;31m MD5 of the file is wrong, try to start again \e[0m";
    echo -e "\e[1;31m Exiting ... \e[0m";
		exit
fi
## Unpack and install the package after the MD5 check
echo -e "\e[1;32m Unpacking the file and making changes in the config file \e[0m";
dpkg -i influxdb_1.5.1_amd64.deb & > /dev/null
## Make changes in the InfluxDB configuration file
## Disable reporting of usage to the InfluxDB team
sleep 5
sed -i 's/# reporting-disabled = false/reporting-disabled = true/' /etc/influxdb/influxdb.conf
## Disable printing of log messages for the meta service
sed -i 's/# logging-enabled = true/logging-enabled = false/' /etc/influxdb/influxdb.conf
## Disable logging of queries before execution
sed -i 's/# query-log-enabled = true/query-log-enabled = false/' /etc/influxdb/influxdb.conf
## Disable logging of HTTP and Continues Queries
sed -i 's/# log-enabled = true/log-enabled = false/g' /etc/influxdb/influxdb.conf

## Starting InfluxDB
echo -e "\e[1;32m All is good, ready to start InfluxDB \e[0m";
influxd -config /etc/influxdb/influxdb.conf & > /dev/null
## Some time is given to finish the first steps and install the databases
sleep 2
## Check that InfluxDB is running
if pgrep -x "influxd" > /dev/null
then
    echo -e "\e[1;32m InfluxDB is running! \e[0m";
else
    echo -e "\e[1;31m There is something wrong with InfluxDB, check manually \e[0m";
		echo -e "\e[1;31m Exiting ... \e[0m";
		exit;
fi
## Install the databases (for Telemetry and SNMP data)
echo -e "\e[1;32m Adding two databases within InfluxDB for Telemetry \e[0m";
MDT_DB=`echo q=CREATE DATABASE mdt_db WITH DURATION ${MDT_DURATION}h SHARD DURATION ${MDT_SHARD}h`
TELEGRAF_DB=`echo q=CREATE DATABASE telegraf WITH DURATION ${TELEGRAF_DURATION}h SHARD DURATION ${TELEGRAF_SHARD}h`
curl -s -XPOST http://localhost:8086/query --data-urlencode "$MDT_DB" > /dev/null
curl -s -XPOST http://localhost:8086/query --data-urlencode "$TELEGRAF_DB" > /dev/null

## Check that both databases were installed
sleep 3
DB_RESULT1=$( curl -s -XPOST http://localhost:8086/query --data-urlencode "q=show databases" | egrep -o "telegraf" ) > /dev/null
DB_RESULT2=$( curl -s -XPOST http://localhost:8086/query --data-urlencode "q=show databases" | egrep -o "mdt_db" ) > /dev/null
if [ "$DB_RESULT1" == "telegraf" -a "$DB_RESULT2" == "mdt_db" ];
  then
    echo -e "\e[1;32m InfluxDB databases were added and activated! \e[0m";
    echo -e "\e[1;32m Let's move on \e[0m";
  else
          echo -e "\e[1;31m There is something wrong with InfluxDB databases, check manually \e[0m";
          echo -e "\e[1;31m Exiting ... \e[0m";
    exit;
fi
echo -e "\e[1;45m InfluxDB is fully installed, we can move on \e[0m";
################################################################################
```

Our next component to be installed is Telegraf, for SNMP data and server statistics polling. Before moving to that, your server should have SNMP and MIBs installed. To achieve that the code will download and install SNMP and SNMP MIBs. By default, Ubuntu does not accept proprietary MIBs, that's why the code will use "SED" to modify the SNMP configuration file to accept "ALL" MIBs. Finally, the code will add the [four MIBs](https://github.com/vosipchu/XR_TCS/tree/master/SNMP-Telegraf) that are used with Telegraf to poll SNMP counters used in our Telemetry Collection Stack.

```
################################################################################
########    This section configures SNMP for Telegraf on your server    ########
################################################################################
## Install SNMP and Common MIBs
echo -e "\e[1;46m SNMP and MIBs will be installed now (for Telegraf) \e[0m";
apt-get -y install snmp=5.7.3+dfsg-1ubuntu4.1
apt-get -y install snmp-mibs-downloader=1.1
## Change SNMP.CONF to accept proprietary MIBs
sed -i 's/mibs :/mibs \+ALL/' /etc/snmp/snmp.conf
download-mibs > /dev/null
## Just four Cisco MIBs are used in that example, copying them
mkdir -p  ~/.snmp/mibs && cp ~/IOSXR-Telemetry-Collection-Stack/SNMP-Telegraf/* ~/.snmp/mibs/
echo -e "\e[1;45m SNMP and MIBs are installed \e[0m";
################################################################################
```

Right after SNMP is installed, the code will proceed to the installation of the next component. The overall flow of the code is similar to the InfluxDB installation. The code will create a directory for Telegraf, download the package, check the MD5 hash and if it passes the check, the code will install Telegraf to the server. Telegraf stays active after installation, that's why the script will stop it, to modify the configuration file. It will also "disable" the package, for Telegraf not to start running automatically after a reboot. It will be just you who decides when to run and when to stop every component of the stack.
After Telegraf stops, the script will modify the configuration file. As before, "SED" is used. The first step is to disable logging (you, actually, can't disable logging, but you can send log messages to [/dev/null](https://en.wikipedia.org/wiki/Null_device)). The next step is to update the file with SNMP information provided in the document with variables, [IP address of the router, community string name](https://github.com/vosipchu/XR_TCS/blob/master/BuildUP-help.doc#L17-L20). The code needs to specify exact OIDs to collect information from. A configuration of OIDs used in the Collection Stack is pre-configured and copied to the configuration file [from the repo](https://github.com/vosipchu/XR_TCS/blob/master/SNMP-Telegraf/SNMP-MIBS-Telegraf). Required SNMP MIBs are copied to Telegraf.  Finally, the script will start Telegraf up, check that it is running and proceed to the next step.
Feel free to add more OIDs if you want this!

```
################################################################################
########        This section configures Telegraf on your server         ########
################################################################################
echo -e "\e[1;46m Configuring Telegraf, the database for SNMP data \e[0m";
mkdir -p ~/analytics/telegraf && cd ~/analytics/telegraf/
echo -e "\e[1;32m Downloading the Telegraf package \e[0m";
wget https://dl.influxdata.com/telegraf/releases/telegraf_1.5.3-1_amd64.deb
## Check the MD5 hash from the download
MD5_Telegraf=`md5sum -c <<<"f3511698087f43ef270261ba45889162 *telegraf_1.5.3-1_amd64.deb"`
## A basic notification if things went wrong
MD5_Telegraf_RESULT=$(echo $MD5_Telegraf | awk '{ print $2 }' )
if [ "$MD5_Telegraf_RESULT" == "OK" ];
  then
    echo -e "\e[1;32m MD5 of the file is fine, moving on \e[0m";
  else
    echo -e "\e[1;31m MD5 of the file is wrong, try to start again \e[0m";
    echo -e "\e[1;31m Exiting ... \e[0m";
    exit
fi
## Unpack and install the package after MD5 check
echo -e "\e[1;32m Unpacking the file and making changes in the config file \e[0m";
sudo dpkg -i telegraf_1.5.3-1_amd64.deb > /dev/null
## Stopping Telegraf to work with configs
sleep 3
systemctl stop telegraf
## Disabling from running automatically after a reboot
systemctl disable telegraf.service
## Updating telegraf.conf to make sure it works as needed
## Sending all log information to /dev/null
sed -i "s/logfile = \"\"/logfile = \"\/dev\/null\"/" /etc/telegraf/telegraf.conf
## Activating SNMP, adding the IP Address of the router and the community string
sed -i "s/# \[\[inputs\.snmp\]\]/\[\[inputs\.snmp\]\]/" /etc/telegraf/telegraf.conf
sed -i "s/#   version = 2/version = 2/" /etc/telegraf/telegraf.conf
sed -i "s/#   agents = \[ \"127\.0\.0\.1\:161\" \]/agents = \[ \"$SNMP_ROUTER\" \]/" /etc/telegraf/telegraf.conf
sed -i "s/#   community = \"public\"/community = \"$SNMP_COMMUNITY\" /" /etc/telegraf/telegraf.conf
## Copying a file with several OIDs to the telegraf.conf file
cp ~/IOSXR-Telemetry-Collection-Stack/SNMP-Telegraf/SNMP-MIBS-Telegraf ~/analytics/telegraf/
sed -i '2445r SNMP-MIBS-Telegraf' /etc/telegraf/telegraf.conf
## copying MIBs to telegraf
echo -e "\e[1;32m Copying four MIBs for our profile \e[0m";
mkdir -p /etc/telegraf/.snmp/mibs && cp ~/.snmp/mibs/* /etc/telegraf/.snmp/mibs
sleep 5
echo -e "\e[1;32m All is good, ready to start Telegraf \e[0m";
sleep 2
systemctl start telegraf > /dev/null
## Check that Telegraf is running
if pgrep -x "telegraf" > /dev/null
then
		echo -e "\e[1;32m Telegraf is running! \e[0m";
else
		echo -e "\e[1;31m There is something wrong with Telegraf, check manually \e[0m";
		echo -e "\e[1;31m Exiting ... \e[0m";
		exit;
fi
echo -e "\e[1;45m Telegraf is fully installed, we can move on \e[0m";
################################################################################
```

The next component to proceed with is [Kapacitor](https://github.com/vosipchu/XR_TCS/blob/master/IOS-XR-Telemetry-BuildUP-stack.sh#L279-L341). The first steps for the code are identical to both components before. The code will do several changes in the configuration file. Initially, the hostname is changed from "localhost" to the IP address (unique name will be used). As before, all logging is disabled. The code configures the address and credentials of InfluxDB.
The central role of Kapacitor is to generate alerts. Alerting is done using scripts, that's why the code will copy an example [script for CPU alerts](https://github.com/vosipchu/XR_TCS/blob/master/Kapacitor/CPU-ALERT-ROUTERS.tick) and modify it according to your environment.
As was explained in our [previous tutorial](URL TO ALERTING), there might be issues with Kapacitor behind a proxy sending notifications to Slack, that's why a [helping python code](https://github.com/vosipchu/XR_TCS/blob/master/Kapacitor/KAPACITOR-HELPER-CPU.py) was also added. The python code will be updated to include information about your Slack channel. If you don't have a proxy, that's okay, everything will work as well.
The code starts the Kapacitor up, defines and activates the TICK script, checks the status and moves on to the next component.

```
################################################################################
########        This section configures Kapacitor on your server        ########
################################################################################
echo -e "\e[1;46m Configuring Kapacitor, the alerting system for InfluxDB \e[0m";
mkdir -p ~/analytics/kapacitor && cd ~/analytics/kapacitor/
wget https://dl.influxdata.com/kapacitor/releases/kapacitor_1.4.1_amd64.deb
## Check the MD5 hash from the download
MD5_Kapacitor=`md5sum -c <<<"eea9b215f241906570eafe3857e1d4c5 *kapacitor_1.4.1_amd64.deb"`
## A basic notification if things went wrong
MD5_Kapacitor_RESULT=$(echo $MD5_Kapacitor | awk '{ print $2 }' )
if [ "$MD5_Kapacitor_RESULT" == "OK" ];
  then
    echo -e "\e[1;32m MD5 of the file is fine, moving on \e[0m";
  else
    echo -e "\e[1;31m MD5 of the file is wrong, try to start again \e[0m";
    echo -e "\e[1;31m Exiting ... \e[0m";
    exit
fi
## Unpack and install the package after MD5 check
echo -e "\e[1;32m Unpacking the file and making changes in the config file \e[0m";
sudo dpkg -i kapacitor_1.4.1_amd64.deb > /dev/null
## Updating kapacitor.conf to make sure it works as needed
## (removing logging, adding InfluxDB data, etc)
sed -i "s/hostname = \"localhost\"/hostname = \"$SERVER_IP\"/" /etc/kapacitor/kapacitor.conf
sed -i "s/log\-enabled = true/log-enabled \= false/" /etc/kapacitor/kapacitor.conf
sed -i "s/file = \"\/var/#&/" /etc/kapacitor/kapacitor.conf
sed -i "s/\"INFO\"/\"OFF\"/" /etc/kapacitor/kapacitor.conf
sed -i "s/localhost\:8086/$SERVER_IP\:8086/" /etc/kapacitor/kapacitor.conf
sed -i "s/username = \"\"/username = \"admin\"/" /etc/kapacitor/kapacitor.conf
sed -i "s/password = \"\"/password = \"admin\"/" /etc/kapacitor/kapacitor.conf
sed -i '430d' /etc/kapacitor/kapacitor.conf
sed -i '430i enabled = false' /etc/kapacitor/kapacitor.conf
## Adding a TICK script for CPU alert generation
cp ~/IOSXR-Telemetry-Collection-Stack/Kapacitor/* ~/analytics/kapacitor/
sed -i "s/localhost/$SERVER_IP/" ~/analytics/kapacitor/CPU-ALERT-ROUTERS.tick
sed -i "s/localhost/$SERVER_IP/" ~/analytics/kapacitor/KAPACITOR-HELPER-CPU.py
## SED was not able to take "SlackToken" as a variable because of "/" inside the word
## A trick is used to change "/"  to "\/" to be accepted by sed
TOKEN_MODIFIED=$(awk -F'/' -v OFS="\\\/" '$1=$1' <<< $SLACK_TOKEN)
## Modifying Slack Token, channel and username
sed -i "s/TokenID/$TOKEN_MODIFIED/" ~/analytics/kapacitor/KAPACITOR-HELPER-CPU.py
sed -i "s/channelname/$SLACK_CHANNEL/" ~/analytics/kapacitor/KAPACITOR-HELPER-CPU.py
sed -i "s/uname/$SLACK_USERNAME/" ~/analytics/kapacitor/KAPACITOR-HELPER-CPU.py
## Starting Kapacitor
sudo systemctl start kapacitor >/dev/null
## Applying and activating our tick script and helper file
sleep 1
kapacitor define CPU-ALERT-ROUTERS -tick ~/analytics/kapacitor/CPU-ALERT-ROUTERS.tick
kapacitor enable CPU-ALERT-ROUTERS
python ~/analytics/kapacitor/KAPACITOR-HELPER-CPU.py & > /dev/null
sleep 2
## Check that Kapacitor is running
if pgrep -x "kapacitord" > /dev/null
then
		echo -e "\e[1;32m Kapacitor is running \e[0m";
else
		echo -e "\e[1;31m There is probably something wrong, check manually \e[0m";
		echo -e "\e[1;31m Exiting ... \e[0m";
		exit;
fi
echo -e "\e[1;45m Kapacitor is fully installed, we can move on \e[0m";
systemctl disable kapacitor.service
################################################################################
```

The next component to be installed is Prometheus. There is nothing unique with the installation, and the code follows the same steps as above. The script will add a new ["job"](https://prometheus.io/docs/concepts/jobs_instances/) to add monitoring of Pipeline there. A static config with the server IP address will be added.
After checking that Prometheus is running, the code will move to the next step.

```
################################################################################
########        This section configures Prometheus on your server       ########
################################################################################
echo -e "\e[1;46m Configuring Prometheus, the monitoring for Pipeline \e[0m";
mkdir -p ~/analytics/prometheus && cd ~/analytics/prometheus/
wget https://github.com/prometheus/prometheus/releases/download/v1.5.2/prometheus-1.5.2.linux-amd64.tar.gz
## Check the MD5 hash from the download
MD5_Prometheus=`md5sum -c <<<"b5e34d7b3d947dfdef8758aaad6591d5 *prometheus-1.5.2.linux-amd64.tar.gz"`
## A basic notification if things went wrong
MD5_Prometheus_RESULT=$(echo $MD5_Prometheus | awk '{ print $2 }' )
if [ "$MD5_Prometheus_RESULT" == "OK" ];
  then
    echo -e "\e[1;32m MD5 of the file is fine, moving on \e[0m";
  else
    echo -e "\e[1;31m MD5 of the file is wrong, try to start again \e[0m";
    echo -e "\e[1;31m Exiting ... \e[0m";
    exit
fi
## Unpack and install the package after MD5 check
echo -e "\e[1;32m Unpacking the file and making changes in the config file \e[0m";
tar xfz prometheus-1.5.2.linux-amd64.tar.gz > /dev/null
## Add Pipeline monitoring into the configuration file
sed -i "s/localhost/$SERVER_IP/" ~/analytics/prometheus/prometheus-1.5.2.linux-amd64/prometheus.yml
echo "  - job_name: 'pipeline'" >> ~/analytics/prometheus/prometheus-1.5.2.linux-amd64/prometheus.yml
echo "    static_configs:" >> ~/analytics/prometheus/prometheus-1.5.2.linux-amd64/prometheus.yml
echo -e "      - targets: ['$SERVER_IP:8989']" >> ~/analytics/prometheus/prometheus-1.5.2.linux-amd64/prometheus.yml
## Start Prometheus
sleep 2
cd ~/analytics/prometheus/prometheus-1.5.2.linux-amd64/
sudo ./prometheus & > /dev/null
sleep 1
## Check that Prometheus is running
if pgrep -x "prometheus" > /dev/null
then
		echo -e "\e[1;32m Prometheus is running \e[0m";
else
		echo -e "\e[1;31m There is probably something wrong, check manually \e[0m";
		echo -e "\e[1;31m Exiting ... \e[0m";
		exit;
fi
echo -e "\e[1;45m Prometheus is fully installed, we can move on \e[0m";
################################################################################
```

The main and only visualization tool used in the Collection Stack is Grafana. The code follows common initial steps to download the package, check the MD5 hash and install the application.
The code will make several changes in the configuration file, for Grafana to be optimal for our goal:
- Disable reporting (sending your internal information to the Grafana team)
- Admin level username/password. It is admin/admin by default, but it might be changed according to your information in [the file with variables](https://github.com/vosipchu/XR_TCS/blob/master/BuildUP-help.doc#L29-L32)
- Keep your username/password pair active for 30 days before requiring to input them again (in your browser)
- Activate logging, but with reduced volume (it is expected you will interact with Grafana a lot)
Starting with [Grafana 5.x release](http://docs.grafana.org/guides/whats-new-in-v5/), you can add all the dashboards and all the databases before starting Grafana itself. This is very convenient for the automation. You can also create [folders](http://docs.grafana.org/reference/dashboard_folders/) within Grafana Home Dashboards page to sort dashboards according to your needs. The code will implement all these enhancements. [Dashboards, databases and folders structure](https://github.com/vosipchu/XR_TCS/tree/master/Grafana) is already preconfigured, and the code will copy this to your active Grafana directory.
After that, the code will install two popular plugins, the [pie chart](https://grafana.com/plugins/grafana-piechart-panel) and the [diagram panel](https://grafana.com/plugins/jdbranham-diagram-panel). They are useful and will be used later in our next use cases tutorials.
The last step is to start Grafana, check that it is running and move on to the final step.

```
################################################################################
########         This section configures Grafana on your server         ########
################################################################################
echo -e "\e[1;46m Configuring Grafana, the visualisation tool \e[0m";
mkdir -p ~/analytics/grafana && cd ~/analytics/grafana/
wget https://s3-us-west-2.amazonaws.com/grafana-releases/release/grafana_5.0.3_amd64.deb
## Check the MD5 hash from the download
MD5_Grafana=`md5sum -c <<<"f337cc57e24019e7d65892370d86f8bc *grafana_5.0.3_amd64.deb"`
## A basic notification if things went wrong
MD5_Grafana_RESULT=$(echo $MD5_Grafana | awk '{ print $2 }' )
if [ "$MD5_Grafana_RESULT" == "OK" ];
  then
    echo -e "\e[1;32m MD5 of the file is fine, moving on \e[0m";
  else
    echo -e "\e[1;31m MD5 of the file is wrong, try to start again \e[0m";
    echo -e "\e[1;31m Exiting ... \e[0m";
    exit;
fi
## Unpack and install the package after MD5 check
apt-get -y install adduser libfontconfig
echo -e "\e[1;32m Unpacking the file and making changes in the config file \e[0m";
dpkg -i grafana_5.0.3_amd64.deb > /dev/null
## Updating grafana.ini. Disable sending your info to the Grafana Team
sed -i "s/;reporting_enabled = true/reporting_enabled = false/" /etc/grafana/grafana.ini
## Create your admin-level user and password
sed -i "s/;admin_user = admin/admin_user = $GRAFANA_USER/" /etc/grafana/grafana.ini
sed -i "s/;admin_password = admin/admin_password = $GRAFANA_PASSWORD/" /etc/grafana/grafana.ini
## This will make sure your login credentials can be active for 30 days before asking to input again
sed -i "s/;login_remember_days = 7/login_remember_days = 30/" /etc/grafana/grafana.ini
## Grafana will have logging active, but the size of logs will be reduced
sed -i "s/;max_lines = 1000000/max_lines = 10000/" /etc/grafana/grafana.ini
sed -i "s/;max_days = 7/max_days = 2/" /etc/grafana/grafana.ini
echo -e "\e[1;32m Adding databases, dashboards ... \e[0m";
## Adding the databases (InfluxDB, Telegraf, Prometheus) for Grafana
cp ~/IOSXR-Telemetry-Collection-Stack/Grafana/Databases/* /etc/grafana/provisioning/datasources/
## Adding descriptions for the folders structure and locations for Grafana
cp ~/IOSXR-Telemetry-Collection-Stack/Grafana/Dashboards_Description/* /etc/grafana/provisioning/dashboards/
## Copying dashboards
mkdir /var/lib/grafana/dashboards/
cp -r ~/IOSXR-Telemetry-Collection-Stack/Grafana/Dashboards/* /var/lib/grafana/dashboards/
## Adding the correct address for Telegraf-SNMP Server
sed -i "s/\"10.30.110.42\"/\"$SNMP_ROUTER\"/" /var/lib/grafana/dashboards/snmp-telegraf/Influx-SNMP-1522021236416.json
## Installing plugins
echo -e "\e[1;32m Installing popular plugins (for future dashboards) \e[0m";
grafana-cli plugins install grafana-piechart-panel 1.1.6 > /dev/null
grafana-cli plugins install jdbranham-diagram-panel 1.4.0 > /dev/null
## Starting Grafana
sudo systemctl start grafana-server > /dev/null
sleep 2
## Check that Grafana is running
if pgrep -x "grafana-server" > /dev/null
then
		echo -e "\e[1;32m Grafana is running \e[0m";
else
		echo -e "\e[1;31m There is probably something wrong, check manually \e[0m";
		echo -e "\e[1;31m Exiting ... \e[0m";
		exit;
fi
echo -e "\e[1;45m Grafana is fully installed, we can move on \e[0m";
################################################################################
```

The final component of the code to install is Pipeline.
The code will download the [zip file with Pipeline](https://github.com/cisco/bigmuddy-network-telemetry-pipeline), check the MD5 hash of it and unzip, if the hash is correct.
After unzipping, it will rename the created directory and copy there the ["metrics.json"](https://xrdocs.github.io/telemetry/tutorials/2018-03-01-everything-you-need-to-know-about-pipeline/) file to support all the telemetry sensor paths used in the Telemetry Collection Stack.
After that, the code will save the original (downloaded from GitHub) "pipeline.conf" file under a new name ("pipeline-original-github.conf ") and install [the pre-configured one](https://github.com/vosipchu/XR_TCS/blob/master/Pipeline/pipeline.conf) that is used in the Telemetry Collection Stack. The configuration of Pipeline includes:
- Accepting TCP streams to port 5432
- Accepting gRPC connections to port 57500
- Pushing all the counters to InfluxDB ("mdt_db" database)
- Providing internal stats to Prometheus
- Pre-configuration for dumping information to the "dump.txt" (not active by default)
The code will start Pipeline in a [screen](https://www.poftut.com/linux-screen-tutorial-examples/).
After starting Pipeline, the script will check the state and print the last message if everything is working fine.

```
################################################################################
########         This section configures Pipeline on your server        ########
################################################################################
echo -e "\e[1;46m Configuring Pipeline, the collector tool \e[0m";
cd ~/analytics
wget https://github.com/cisco/bigmuddy-network-telemetry-pipeline/archive/master.zip

## Check the MD5 hash from the download
MD5_Pipeline=`md5sum -c <<<"1aac6ae82dbb633bdb7658b1463fd2a5 *master.zip"`
## A basic notification if things went wrong
MD5_Pipeline_RESULT=$(echo $MD5_Pipeline | awk '{ print $2 }' )
if [ "$MD5_Pipeline_RESULT" == "OK" ];
  then
    echo -e "\e[1;32m MD5 of the file is fine, moving on \e[0m";
  else
    echo -e "\e[1;31m MD5 of the file is wrong, try to start again \e[0m";
    echo -e "\e[1;31m Exiting ... \e[0m";
    exit;
fi
## Unzipping and installing Pipeline
echo -e "\e[1;32m Unpacking the file and making changes in the config file \e[0m";
unzip master.zip > /dev/null
## Renaming the directory
mv bigmuddy-network-telemetry-pipeline-master/ pipeline
## Installing the metrics.json file for all the sensor paths used in the Stack
cp ~/IOSXR-Telemetry-Collection-Stack/Pipeline/metrics.json ~/analytics/pipeline/
## Saving original pipeline.conf file
mv ~/analytics/pipeline/pipeline.conf ~/analytics/pipeline/pipeline-original-github.conf
## Copying pre-defined pipeline.conf and updating IP address
cp ~/IOSXR-Telemetry-Collection-Stack/Pipeline/pipeline.conf ~/analytics/pipeline/
sed -i "s/1.2.3.4/$SERVER_IP/" ~/analytics/pipeline/pipeline.conf
cd ~/analytics/pipeline/bin && touch dump.txt
## Stating Pipeline from a screen
echo -e "\e[1;32m Starting Pipeline (in a screen) \e[0m";
screen -dm -S Pipeline bash -c 'cd ~/analytics/pipeline; sudo ./bin/pipeline -config pipeline.conf -pem ~/.ssh/id_rsa; exec /bin/bash'
## Check that Pipeline is running
sleep 2
if pgrep -x "pipeline" > /dev/null
then
		echo -e "\e[1;32m Pipeline is running \e[0m";
else
		echo -e "\e[1;31m There is probably something wrong, check manually \e[0m";
		echo -e "\e[1;31m Exiting ... \e[0m";
		exit;
fi

## This is the end of the script!
echo -e "\e[1;45m IOS XR MDT Collection Stack is up and running. \e[0m";
echo -e "\e[1;45m Go to apply the 'wrapper.sh' script! \e[0m";
echo -e "\e[1;45m Good luck with your telemetry testing! \e[0m";

################################################################################
########                        End of the script                       ########
################################################################################
```

## Wrappers script overview

Running the ["wrappers.sh"](https://github.com/vosipchu/XR_TCS/blob/master/wrappers.sh) script is your last step to have the Telemetry Collection Stack up and running.

The purpose of the code is to modify ["~/.bashrc"](https://www.lifewire.com/bashrc-file-4101947) file of your user in Linux. Basically, we're adding new commands that will be active each time you log in to the system.
The code itself is pretty compact and refers to the ["wrappers.txt"](https://github.com/vosipchu/XR_TCS/blob/master/wrappers.txt) file.

```
#!/bin/bash
################################################################################
#########             Telemetry Collection CLI Aliases                ##########
################################################################################
cd ~/IOSXR-Telemetry-Collection-Stack
sed -i '1r wrappers.txt' ~/.bashrc

echo -e "\e[1;45m Command wrappers were configured! \e[0m";

################################################################################
########                        End of the script                       ########
################################################################################
```

Let's walk through the "wrappers.txt" file to understand all the new lines to be added to the "~/.bashrc" file.

The first command creates an alias for you. As you remember, in the Telemetry Collection Stack Pipeline runs in a screen. Every component from the main code runs with [sudo](https://en.wikipedia.org/wiki/Sudo), or, in other words, with superuser privilege level. That's why for you to get into an opened screen, where Pipeline is running, you have to run it with "sudo" as well. Alias will help you not to type "sudo" every time you want to collect (you can always fall back to a normal mode running "\screen" to ignore the alias).

<div class="highlighter-rouge">
<pre class="highlight">
<code>
alias screen='sudo screen'
if [ -z "$PS1" ]; then
    return
fi
</code>
</pre>
</div>

As was described in our [previous post](LINK TO ALIASES), you can start (and stop) running Pipeline in "troubleshooting mode", where all the Telemetry messages are dumped into the "dump.txt" file.
In order to do that, a special function is added into the "~/.bashrc" file.
When you plan to start using Pipeline in troubleshooting mode, the code will look for an active instance of Pipeline and kill it (together with an active instance of "screen"). Then it will use "SED" to modify the content of the "pipeline.conf" file, by activating dumping to a file. Finally, it starts a Pipeline. It is expected, that you will go and [start checking the "dump.txt" file contents](https://www.howtoforge.com/linux-tail-command/) in real time to see the messages, that's why the code will also remove the previous version of the "dump.txt" file. This way you will see the actual information you're looking for, not the data from previous troubleshooting sessions.
At some moment you won't need to run this mode anymore, and you will stop it. The code will find the active running instance of Pipeline, kill it, update the configuration of the "pipeline.conf" file back and start a new instance of Pipeline in a screen.

```
pipeline() {
    if [[ $@ == "troubleshooting start" ]]; then
        command `PID=$(pgrep pipeline); sudo kill -9 $PID; sudo pkill screen 2>/dev/null`
        command `sudo sed -i '43,48s/^# //' ~/analytics/pipeline/pipeline.conf`
        command `sudo rm ~/analytics/pipeline/bin/dump.txt && sudo touch ~/analytics/pipeline/bin/dump.txt`
        command `screen -dm -S Pipeline bash -c 'cd ~/analytics/pipeline; sudo ./bin/pipeline -config pipeline.conf -pem ~/.ssh/id_rsa -log= -debug; exec /bin/bash'`
        echo -e "\e[1;32m Done! Go to ~/analytics/pipeline/bin/ and execute 'sudo tail -f dump.txt'\e[0m";

    elif [[ $@ == "troubleshooting stop" ]]; then
        command `PID=$(pgrep pipeline); sudo kill -9 $PID; sudo pkill screen 2>/dev/null`
        command `sudo sed -i '43,48s/^/# /' ~/analytics/pipeline/pipeline.conf`
        command `sudo rm ~/analytics/pipeline/bin/dump.txt && sudo touch ~/analytics/pipeline/bin/dump.txt`
        command `screen -dm -S Pipeline bash -c 'cd ~/analytics/pipeline; sudo ./bin/pipeline -config pipeline.conf -pem ~/.ssh/id_rsa; exec /bin/bash'`
        echo -e "\e[1;32m Done! \e[0m";
    fi
}
```

One more function will be added for your convenience. This function gives you a convenient way to check the current state of components of the Telemetry Collection Stack. You can use "show" with any component, e.g., "show pipeline" or "show kapacitor" and you will get a message whether this component is running or not.
If you make a typo, the function will notify you about this and, you should try to type again.

```
show() {
    if [[ $@ == "influxdb" ]]; then
        if pgrep -x "influxd" > /dev/null
                then
                    echo -e "\e[1;32m InfluxDB is running \e[0m";
                else
                    echo -e "\e[1;31m InfluxDB is not running \e[0m";
        fi
    elif [[ $@ == "telegraf" ]]; then
      if pgrep -x "telegraf" > /dev/null
              then
                  echo -e "\e[1;32m Telegraf is running \e[0m";
              else
                  echo -e "\e[1;31m Telegraf is not running \e[0m";
      fi

    elif [[ $@ == "kapacitor" ]]; then
      if pgrep -x "kapacitord" > /dev/null
              then
                  echo -e "\e[1;32m Kapacitor is running \e[0m";
              else
                  echo -e "\e[1;31m Kapacitor is not running \e[0m";
      fi

    elif [[ $@ == "prometheus" ]]; then
        if pgrep -x "prometheus" > /dev/null
                then
                    echo -e "\e[1;32m Prometheus is running \e[0m";
                else
                    echo -e "\e[1;31m Prometheus is not running \e[0m";
        fi
    elif [[ $@ == "grafana" ]]; then
      if pgrep -x "grafana-server" > /dev/null
              then
                  echo -e "\e[1;32m Grafana is running \e[0m";
              else
                  echo -e "\e[1;31m Grafana is not running \e[0m";
      fi

    elif [[ $@ == "pipeline" ]]; then
        if pgrep -x "pipeline" > /dev/null
                then
                    echo -e "\e[1;32m Pipeline is running \e[0m";
                else
                    echo -e "\e[1;31m Pipeline is not running \e[0m";
                fi
    else
	echo -e "\e[1;31m Sorry, you made a typo. Please, try again \e[0m";
    fi
}
```

To manage the components of the stack efficiently and fast, two more functions are added. It allows you to start or stop individual components of the stack in a convenient way. For example, "start telegraf &" or "stop prometheus". As before, in case of a typo, the system notifies you, and you will need to try again.

As with the main script, Pipeline is started using [screen](https://www.poftut.com/linux-screen-tutorial-examples/).
Pay attention that when you start a component, you should add "&" at the end, to make sure you have it running in the background.
All those three functions ("show", "start", "stop") will give you the full level of control and, hopefully, comfort working with the Telemetry Collection Stack.

```
start() {
    if [[ $@ == "influxdb" ]]; then
        command `sudo influxd -config /etc/influxdb/influxdb.conf`
                                echo -e "\e[1;32m Done! \e[0m";
    elif [[ $@ == "telegraf" ]]; then
        command `sudo systemctl start telegraf`
                                echo -e "\e[1;32m Done! \e[0m";
    elif [[ $@ == "kapacitor" ]]; then
        command `sudo systemctl start kapacitor`
                                echo -e "\e[1;32m Done! \e[0m";
    elif [[ $@ == "prometheus" ]]; then
        command `cd ~/analytics/prometheus/prometheus-1.5.2.linux-amd64; sudo ~/analytics/prometheus/prometheus-1.5.2.linux-amd64/prometheus`
                                echo -e "\e[1;32m Done! \e[0m";
    elif [[ $@ == "grafana" ]]; then
        command `sudo systemctl start grafana-server`
                                echo -e "\e[1;32m Done! \e[0m";
    elif [[ $@ == "pipeline" ]]; then
        command `screen -dm -S Pipeline bash -c 'cd ~/analytics/pipeline; sudo ./bin/pipeline -config pipeline.conf -pem ~/.ssh/id_rsa; exec /bin/bash'`
                                echo -e "\e[1;32m Done! \e[0m";
    else
        echo -e "\e[1;31m Sorry, you made a typo. Please, try again \e[0m";
    fi
}

stop() {
    if [[ $@ == "influxdb" ]]; then
        command `PID=$(pgrep influxd); sudo kill -9 $PID;`
                                echo -e "\e[1;32m Done! \e[0m";
    elif [[ $@ == "telegraf" ]]; then
        command `sudo systemctl stop telegraf`
                                echo -e "\e[1;32m Done! \e[0m";
    elif [[ $@ == "kapacitor" ]]; then
        command `sudo systemctl stop kapacitor`
                                echo -e "\e[1;32m Done! \e[0m";
    elif [[ $@ == "prometheus" ]]; then
        command `PID=$(pgrep prometheus); sudo kill -9 $PID;`
                                echo -e "\e[1;32m Done! \e[0m";
    elif [[ $@ == "grafana" ]]; then
        command `sudo systemctl stop grafana-server`
                                echo -e "\e[1;32m Done! \e[0m";
    elif [[ $@ == "pipeline" ]]; then
        command `PID=$(pgrep pipeline); sudo kill -9 $PID; sudo pkill screen 2>/dev/null`
                                echo -e "\e[1;32m Done! \e[0m";
    else
        echo -e "\e[1;31m Sorry, you made a typo. Please, try again \e[0m";
    fi
}
```

For your convenience, all those alias commnads will be printed as a "welcome screen" every time you login. 

## Removing stack overview

To remove the IOS XR Telemetry Collection Stack from your server, you need to run the [removal script](https://github.com/vosipchu/XR_TCS/blob/master/IOS-XR-Telemetry-Destroy-stack.sh).
The script will go through all the installed components, InfluxDB, Telegraf, Kapacitor, Prometheus, Grafana, and Pipeline. For every component, it will find the running process, kill it, uninstall the program itself and remove all the modified directories and files. Messages about the progress will be printed on your screen.

```
#######################################################
######## This removes InfluxDB from your server #######
#######################################################
echo -e "\e[1;46m Removing InfluxDB \e[0m";
PID=`pgrep influxd`
kill -9 $PID
dpkg --purge influxdb
rm -r ~/analytics/influxdb
rm -r /var/lib/influxdb
echo -e "\e[1;45m Done! \e[0m";
#######################################################


#######################################################
####### This removes Telegraf from your server ########
#######################################################
echo -e "\e[1;46m Removing Telegraf \e[0m";
PID=`pgrep telegraf`
kill -9 $PID
dpkg --purge telegraf
rm -r ~/analytics/telegraf
rm -r /etc/telegraf/
echo -e "\e[1;45m Done! \e[0m";
#######################################################


#######################################################
####### This removes Kapacitor from your server #######
#######################################################
echo -e "\e[1;46m Removing Kapacitor \e[0m";
dpkg --purge kapacitor
PID=`pgrep kapacitor`
kill -9 $PID
rm -r ~/analytics/kapacitor
rm -r /var/lib/kapacitor
rm -r /var/log/kapacitor
PID=`ps -ef | grep KAPACITOR-HELPER-CPU | \
grep -v "grep" | awk '{print $2}'`
kill -9 $PID
echo -e "\e[1;45m Done! \e[0m";
#######################################################


#######################################################
###### This removes Prometheus from your server #######
#######################################################
echo -e "\e[1;46m Removing Prometheus \e[0m";
PID=`pgrep prometheus`
kill -9 $PID
rm -r ~/analytics/prometheus
echo -e "\e[1;45m Done! \e[0m";
#######################################################


#######################################################
######## This removes Grafana from your server ########
#######################################################
echo -e "\e[1;46m Removing Grafana \e[0m";
PID=`pgrep grafana`
kill -9 $PID
dpkg --purge grafana
rm -r ~/analytics/grafana
rm -r /etc/grafana
rm -r /var/lib/grafana
echo -e "\e[1;45m Done! \e[0m";
#######################################################


#######################################################
######## This removes Pipeline from your server #######
#######################################################
echo -e "\e[1;46m Removing Pipeline \e[0m";
PID=`pgrep pipeline`
kill -9 $PID
pkill screen 2>/dev/null
rm -r ~/analytics/pipeline
rm ~/analytics/master.zip
rm -r ~/analytics
echo -e "\e[1;45m Every component was removed! \e[0m";

#######################################################
########         End of the script              #######
#######################################################
```

## Removing wrappers overview

If you decided to remove your Collection Stack from the server, you might want to remove all the alias commands and functions created for you as well. Just run the ["wrappers-remove.sh"](https://github.com/vosipchu/XR_TCS/blob/master/wrappers-remove.sh) script.
It will remove all the added lines from ".bashrc" and you will not see anything next time you log in!

```
#!/bin/bash
#######################################################
######## This removes aliases from your server  #######
#######################################################
sed -i 2,192d ~/.bashrc;

echo -e "\e[1;45m All command wrappers from ~/.bashrc were removed! \e[0m";
```


## Kapacitor Tick Script overview

We went through all the scripts within the IP XR Telemetry Collection Stack. The final missing piece to be explained is [the Kapacitor Tick Script](https://github.com/vosipchu/XR_TCS/blob/master/Kapacitor/CPU-ALERT-ROUTERS.tick). This script is a simple one (in our next posts we will share more interesting use cases of Kapacitor). There are small changes in the TICK rules between different version of Kapacitor, so, pay attention to the version you're using and double check after upgrades.

The first line defines the name of the database and the retention policy to be used. Kapacitor can work in ["batch" or "stream" mode](https://www.influxdata.com/blog/batch-processing-vs-stream-processing/). For our scenario, the stream mode is used, as you can see on the [third line of the code](https://github.com/vosipchu/XR_TCS/blob/master/Kapacitor/CPU-ALERT-ROUTERS.tick#L3).
After you selected the mode of operation, you should specify the measurements you will be monitoring. For IOS XR Telemetry, a measurement means a sensor path. In our case, [CPU Utilization sensor path](https://github.com/vosipchu/XR_TCS/blob/master/Kapacitor/CPU-ALERT-ROUTERS.tick#L6) is configured and the purpose of the script is to check the percentage of the CPU load.
The next step is to define an action, that will include thresholds and how to react when the system crosses them. Our action defined is [|alert()](https://docs.influxdata.com/kapacitor/v1.5/nodes/alert_node/). The threshold is "15". Whenever your router crosses 15 (cpu load percentage), the script will generate an [info level](https://docs.influxdata.com/kapacitor/v1.5/nodes/alert_node/#info) update (other options are available, like [warn](https://docs.influxdata.com/kapacitor/v1.5/nodes/alert_node/#warn) or [crit](https://docs.influxdata.com/kapacitor/v1.5/nodes/alert_node/#crit))
Feel free to update this number and use anything that works better for you (like "5", or "30").
The main recipient of alerts is a Slack channel, and you can send to [Slack directly](https://docs.influxdata.com/kapacitor/v1.5/nodes/alert_node/#slack), but in a network behind a proxy server, it might not work correctly. That's why  ["post"](https://docs.influxdata.com/kapacitor/v1.5/nodes/alert_node/#post) action was configured. The message goes to the python script (KAPACITOR-HELPER-CPU.py), that accepts this POST message on a pre-defined port. After receiving the update, the python code does basic conversions and sends the formatted message to the channel provided (the Slack Token, the Channel name, and username are from the initial document with variables).
The [final line](https://github.com/vosipchu/XR_TCS/blob/master/Kapacitor/CPU-ALERT-ROUTERS.tick#L20) makes sure that the Tick script only generates updates when there is an event happening (crossing the threshold up or down). Otherwise, the script will send you messages constantly while your CPU load is above the threshold number.

```
dbrp "mdt_db"."autogen"

stream
    // Select just the cpu measurement from our created mdt_db database.
    |from()
        .measurement('Cisco-IOS-XR-wdsysmon-fd-oper:system-monitoring/cpu-utilization')
        .groupBy('Producer', 'node-name', 'process-cpu__process-name')
    // This defines the action we want to create
    |alert()
	// 15 is the threshold. Crossing 15 means something is wrong
        .info(lambda: "total-cpu-one-minute" > 15)
	// Add different levels of alarms if you want
        //        .warn(lambda: "total-cpu-one-minute" > 20)
        //        .crit(lambda: "total-cpu-one-minute" > 25)
        .log('/tmp/alerts.log')
	// Sending our alarm notification to the python script
        .post('http://localhost:5200/relay')
	// Every alarm will be generated only once after crossing the threshold above
	// Remove this line if you want to have alarms constantly (when above the threshold)
	.stateChangesOnly()
```

## Conclusion

The script used to build the IOS XR Collection Stack had no intention to be complicated. The main primary was to help different people to quickly build a collector to start testing telemetry without a need to read hundreds of pages to understand how to integrate various tools together.
This post contains many details explaining how all the components of the Stack are installed and configured. Hope, this can give you more ideas on what you want to do and how to achieve this.
Stay tuned, there are many other cool things to be delivered and shared with you!