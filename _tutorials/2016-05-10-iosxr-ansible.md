---
permalink: "/tutorials/IOSXR-Ansible"
author: Mike Korshunov
excerpt: Getting started with IOSXR and Ansible playbooks
published: true
title: "IOSXR-Ansible.md"
---
# Introduction
In this tutorial we are going to cover XRv4 configuration with Ansible. We will setup and try simplest configuration with Unix machine connected to XRv64.

## Prerequisites
- Computer with with 4-5GB free RAM;
- Vagrant;
- Ansible;
- [XRv64 image](http://engci-maven-master.cisco.com/artifactory/appdevci-snapshot/);

### Vagrant pre-setup

To add XR box, image should be downloaded, after this issue the command:
	
    $ vagrant box add xrv64 iosxrv-fullk9-x64.box_2016-05-07-19-04-50

Image for Ubuntu will be downloaded from official source:
	
    $ vagrant box add ubuntu/trusty64
    
Let's check for result, we should have box available, box ubuntu/trusty64 and xrv64 should be displayed:
![Box validation](https://cisco.box.com/shared/static/rwa6hxojdjcwu1ln8h0zu5oq6ij30def.png)
![Box validation](https://xrdocs.github.io/xrdocs-images/assets/images/xr_ansible_01_box_list - Copy.png)

Vagrantfile with 2 Vagrant boxes should be like:

```
Vagrant.configure(2) do |config|

  config.vm.provision "shell", inline: "echo Hello User"

  config.vm.define "ubuntu" do |ubuntu|
    ubuntu.vm.box = "ubuntu/trusty64"
    ubuntu.vm.network :private_network, virtualbox__intnet: "link1", ip: "10.1.1.10"
  end

  config.vm.define "xr" do |xr|
    xr.vm.box = "xrv64"
    xr.vm.network :private_network, virtualbox__intnet: "link1", ip: "10.1.1.20"
  end

end

```


Next step to create Vagrantfile and boot up boxes:

mkorshun@MKORSHUN-2JPYH MINGW64 ~/Documents/workCisco/tutorial
$ vagrant up



### Ubuntu box pre-configuration

To access Ubuntu box just issue the command (no password required):
```
vagrant ssh
```


Install Ansible and all required packages for it. 

```
sudo apt-get install python-setuptools python-dev build-essential git libssl-dev libffi-dev
sudo easy_install pip sshpass
wget https://bootstrap.pypa.io/ez_setup.py -O - | sudo python

git clone http://gitlab.cisco.com/aermongk/iosxr-ansible.git
git clone git://github.com/ansible/ansible.git --recursive
cd ansible/ && sudo python setup.py install
```

Create key to access XR in future:
```
vagrant@vagrant-ubuntu-trusty-64:~$ ssh-keygen -t rsa
Generating public/private rsa key pair.
Enter file in which to save the key (/home/vagrant/.ssh/id_rsa):
Enter passphrase (empty for no passphrase):
Enter same passphrase again:
.our identification has been saved in
.pub.public key has been saved in
The key fingerprint is:
4e:a5:24:de:e1:51:7c:85:78:a1:65:b1:08:17:e3:be vagrant@vagrant-ubuntu-trusty-64
The key's randomart image is:
+--[ RSA 2048]----+
|        ..=o==.  |
|         =o*+.   |
|      . + =o.    |
|     . = *       |
|      . S .      |
|       o   .     |
|        . E      |
|                 |
|                 |
+-----------------+
vagrant@vagrant-ubuntu-trusty-64:~$ 
vagrant@vagrant-ubuntu-trusty-64:~$ cat ~/.ssh/id_rsa.pub
ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABAQCfbpi4N3Lcl5i9Y8gd/g4x05IvnIfoPJkhdmGBW2HMFwqWgJQkJF1BM8SuukWeG8+Su4g0Un5tU4/nvbAcqBDR7wFEmB7z7k8VQrXZUxeB4Lc1jEwdDLbxxGOUitLQO+IXlHFJVpkp9Ps6tT82xopaSOQFXKDq0vYXdeEMD/k3NG++0u5pOsJu+kXLIULy1Ix6qvcFDRKbqfT14fU/K7vBYz8gP8Cl+sql9ySm7aOSb0liCBx46/SueBX6uadnMgecVBc1GyvmEX5PwBppUr3Cpby2yOf69iX4NnNcWTTrPY7o7LWuXPNiAgbSAS3R7jVBPyTltB9zCbGq8We5UuVX vagrant@vagrant-ubuntu-trusty-64
vagrant@vagrant-ubuntu-trusty-64:~$ 
```




### XR box pre-configuration

To access XR Linux Shell: 
> vagrant ssh xr

To access XR console it takes one additional step to figure out port (credentials for ssh: vagrant/vagrant):
```
mkorshun@MKORSHUN-2JPYH MINGW64 ~/Documents/workCisco/tutorial
$ vagrant port xr
The forwarded ports for the machine are listed below. Please note that
these values may differ from values configured in the Vagrantfile if the
provider supports automatic port collision detection and resolution.
 22 (guest) = 2223 (host)
 57722 (guest) = 2200 (host)
mkorshun@MKORSHUN-2JPYH MINGW64 ~/Documents/workCisco/tutorial
$ ssh -p 2223 vagrant@localhost
vagrant@localhost's password:
RP/0/RP0/CPU0:ios#
```

Now let's  configure IP address at XR box. Issue the commant at XR CLI.

```
conf t
interface GigabitEthernet0/0/0/0
 ipv4 address 10.1.1.20 255.255.255.0
 no shutdown
!
commit
end
```

Finally, let's check connectivity between boxes:
```
RP/0/RP0/CPU0:ios#ping 10.1.1.20
Mon May  9 08:36:33.071 UTC
Type escape sequence to abort.
Sending 5, 100-byte ICMP Echos to 10.1.1.20, timeout is 2 seconds:
!!!!!
Success rate is 100 percent (5/5), round-trip min/avg/max = 1/5/20 ms
RP/0/RP0/CPU0:ios#
```

Let's copy public part of key from Ubuntu box and allow access without password. For IOS-XR image 6.1.x commands below:

run vi /root/.ssh/authorized_keys


run sed -i.bak -e '/^PermitRootLogin/s/no/yes/' /etc/ssh/sshd_config_operns

run service sshd_operns restart

run chkconfig --add sshd_operns


## Ansible Part

At first we need to set up environment and hosts
