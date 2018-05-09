---
title: IPv6 tunneling with Hurricane Electrics (HE)
author: Andrea Veri
type: post
date: 2012-12-21T13:08:20+00:00
url: /2012/12/21/ipv6-tunneling-hurricane-electrics/
categories:
  - Networking
  - Planet Debian
  - Planet Fedora
  - Planet Ubuntu
tags:
  - Hurricane Electrics
  - IPv6

---
I&#8217;ve been looking around for a possible way to connect to the IPv6 internet for some time now and given the fact my provider didn&#8217;t allow me to run IPv6 natively I had to find an alternative solution. **Hurricane Electrics** (HE) provides (for free) five configurable **IPv4-to-IPv6** tunnels together with a free **DNS service** and an interesting **certification program**.

![Hurricane Electrics IPv6 Certification](/wp-content/uploads/2012/12/certificate_badge.png)

Willing to test the latest revision of the Internet Protocol on your **Debian**, **Ubuntu**, **Fedora** machines? Here&#8217;s **how**:

**1.** Register yourself at Hurricane Electrics by visiting <a href="http://tunnelbroker.net/" target="_blank">tunnelbroker.net</a>.

**2.** <a href="http://tunnelbroker.net/new_tunnel.php" target="_blank">Create a new tunnel</a> and make sure to use your **public IP** address as your **IPv4 Endpoint**.

**3.** Write down the relevant details of your tunnel, specifically:

  1. Server IPv6 Address: 2001:470:**1f0a**:a6f::1 /64
  2. Server IPv4 Address: 216.66.84.46 (this actually depends on which server did you choose on the previous step)
  3. Client IPv6 Address: 2001:470:**1f0a**:a6f::2/64

![Tunnel Broker](/wp-content/uploads/2012/12/tunnel_broker.png)

**4.** Create a little script that will update your **IPv4 tunnel endpoint** every time your internet IP **changes**. (this step is not needed if you have an internet connection with a **static IP**):

{{< highlight Bash >}}#!/bin/bash
USERNAME=yourHEUsername
PASSWORD=yourHEPassword
TUNNELID=yourHETunnelID
GET "https://$USERNAME:$PASSWORD@ipv4.tunnelbroker.net/ipv4_end.php?tid=$TUNNELID"{{< / highlight >}}

**5.** Create the networking **configuration** files on your computer:

**Debian / Ubuntu**, on the **/etc/network/interfaces** file:

{{< highlight Bash >}}auto he-ipv6
iface he-ipv6 inet6 v4tunnel
address 2001:470:<b>1f0a</b>:a6f::2
netmask 64
endpoint 216.66.80.30
local 192.168.X.X (Your PC's LAN IP address)
ttl 255
gateway 2001:470:<b>1f0a</b>:a6f::1
pre-up /home/user/bin/update_tunnel.sh{{< / highlight >}}

**Fedora**, on the ** /etc/sysconfig/network-scripts/ifcfg-he-ipv6** file:

{{< highlight Bash >}}DEVICE=he-ipv6
TYPE=sit
BOOTPROTO=none
ONBOOT=yes
IPV6INIT=yes
IPV6TUNNELIPV4="216.66.80.30"
IPV6TUNNELIPV4LOCAL="192.168.X.X" (Your PC's LAN IP address)
IPV6ADDR="2001:470:<b>1f0a</b>:a6f::2/64"{{< / highlight >}}

and on the **/etc/sysconfig/network** file, add:

{{< highlight Bash >}}NETWORKING_IPV6=yes
IPV6_DEFAULTGW="2001:470:<b>1f0a</b>:a6f::1"
IPV6_DEFAULTDEV="he-ipv6"{{< / highlight >}}

You can then set up a little **/sbin/ifup-pre-local** script to update the IPv4 tunnel endpoint when your dynamic IP changes or simply add the script on the **/etc/cron.daily** directory and have it executed when you turn up your computer.

![A sample image taken from ipv6-test.com.](/wp-content/uploads/2012/12/ipv6_test.png)

**6.** Change the DNS servers on **/etc/resolv.conf**:

**OpenDNS:**

{{< highlight Bash >}}nameserver 2620:0:ccc::2
nameserver 2620:0:ccd::2{{< / highlight >}}

**Google DNS**:

{{< highlight Bash >}}nameserver 2001:4860:4860::8888
nameserver 2001:4860:4860::8844{{< / highlight >}}

**7.** Restart your network and enjoy IPv6!

**8.** If you want to know more about IPv6 take some time for the <a href="http://ipv6.he.net/certification" target="_blank">HE Certification program</a>, you will learn a lot and eventually win a sponsored **t-shirt**, I just finished mine :-)

**EDIT**: Be aware of the fact that as soon as the tunnel is up, your computer will be exposed to to the internet without any kind of firewall (the tunnel sets up a direct connection to the internet, even bypassing your router&#8217;s firewall), you can secure your machine by using **ip6tables**. Thanks Michael Zanetti for pointing this out!
