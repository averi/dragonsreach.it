---
title: SSH Tunneling for VNC
author: Andrea Veri
type: post
date: 2012-10-06T13:50:24+00:00
url: /2012/10/06/ssh-tunneling-for-vnc/
aktt_tweeted:
  - 1
aktt_notify_twitter:
  - yes
dcssb_short_url:
  - http://tinyurl.com/8o3sop5
tmac_last_id:
  - 276277221797285888
categories:
  - SSH
  - Sysadmin
tags:
  - SSH
  - Tunneling
  - Virsh
  - VNC

---
Logging in into a Linux machine and executing the hundreds commands available is just one of the most common usages of **OpenSSH**. Another interesting and very useful usage is tunneling some specific (or even all) traffic from your local machine to an external machine you have access to.

Today we&#8217;ll analyze how to access a certain virtual machine&#8217;s **console** by tunneling the relevant **VNC** port locally and accessing it through your favorite VNC client. The scenario:

  1. Machine **A** is our main virtualization machine and hosts several virtual machines. (VMs)
  2. Each **VM** has its own VNC port assigned. (usually the port range goes from **5900** to **5910** or even more if the hosted VMs are more than 10)
  3. We&#8217;ll be using libvirt, thus virsh.

We first need to find out which port got assigned to the VM we want to have console access to:

{{< highlight Bash >}}sudo virsh

virsh # list
Id   Name   Status
----------------------------------------------------
5    foo    running
6    bar    running
7    foobar running

virsh # vncdisplay foobar
:3{{< / highlight >}}

We, then, create a tunnel which redirects all the traffic from the main virtualization machine&#8217;s port to the port we gonna specify in the next command:

{{< highlight Bash >}}ssh -f -N -L 5910:localhost:5903 user@machine-A.com{{< / highlight >}}

A few **details** about the previous command:

  1. **-N **tells SSH to not execute any command after logging in.
  2. **-f** tells SSH to hide into the background just before the command gets executed.
  3. **-L** enables the port forwarding between the local (client) host and the host on the remote side.

And&#8230;why did I choose respectively port **5903** and **5910**

While you can adjust port **5910** with your own choice (that will just move the tunneled traffic from port **5910** to your favorite port), that won&#8217;t work as expected with port 5903 since each VNC port is binded to the number of display virsh assigned to it. (for example, the **bar** VM may be running on display 5, thus its **vncdisplay** port will be **5905**)

When **done**, fire up your favorite VNC client and create a new connection with the following details:

{{< highlight Bash >}}Protocol: VNC - Virtual Network Computing
Server: localhost - 127.0.0.1
Port: 5910{{< / highlight >}}

The connection will load and you&#8217;ll be put in front of your **&#8216;foobar&#8217;** VM console.
