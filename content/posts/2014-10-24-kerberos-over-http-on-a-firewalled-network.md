---
title: 'Kerberos over HTTP: getting a TGT on a firewalled network'
author: Andrea Veri
type: post
date: 2014-10-24T13:50:52+00:00
url: /2014/10/24/kerberos-over-http-on-a-firewalled-network/
categories:
  - Planet Fedora
  - Planet GNOME
tags:
  - Kerberos

---
One of the benefits I originally wanted to bring with the <a href="https://www.dragonsreach.it/2014/10/07/the-gnome-infrastructure-is-now-powered-by-freeipa/" target="_blank">FreeIPA move</a> to GNOME contributors was the introduction of an additional authentication system to connect to to the services hosted on the **GNOME** Infrastructure. The authentication system that comes with the FreeIPA bundle that I had in mind was Kerberos. Users willing to use Kerberos as their preferred authentication system would just be required to get a TGT (Ticket-Granting Ticket) from the KDC (Key Distribution Center) through the **kinit** command. Once done authenticating to the services currently supporting Kerberos will be as easy as pointing a configured browser (Google for how to configure your browser to use Krb logins) to account.gnome.org without being prompted for entering the usual username / password combination or pushing to git without using the public-private key mechanism. That theoretically means you won&#8217;t be required to use a SSH key for loggin in to any of the GNOME services at all as entering your password to the **kinit** password prompt will be enough (for at least 24 hours as that&#8217;s the life of the TGT itself on our setup) for doing all you were used to do before the Kerberos support introduction.

<div id="attachment_1210" style="width: 510px" class="wp-caption aligncenter">
  <a href="https://www.dragonsreach.it/wp-content/uploads/2014/10/kerberos-over-http.png"><img class="wp-image-1210" src="https://www.dragonsreach.it/wp-content/uploads/2014/10/kerberos-over-http.png" alt="kerberos-over-http" width="500" height="242" /></a>
  
  <p class="wp-caption-text">
    A successful SSH login using the most recent Kerberos package on Fedora 21
  </p>
</div>

The issue we faced at first was the underlying networking infrastructure firewalling all Kerberos ports blocking the use of **kinit** itself which kept timing out reaching port 88. A few days later I was contacted by RH&#8217;s developer **<span class="gD">Nathaniel McCallum </span>**<span class="gD">who worked out a way to bypass this restriction by creating a KDC proxy that accepts requests from port 443 and proxies them to the internal KDC running on port 88. With the <a href="http://web.mit.edu/kerberos/krb5-1.13/" target="_blank">recent Kerberos release</a> (released on October 15th, 2014 and following the <a href="http://msdn.microsoft.com/en-us/library/hh553774.aspx" target="_blank">MS-KKDCP protocol</a>) a patched <strong>kinit </strong>allows users to retrieve their TGTs directly from the HTTPS proxy completely bypassing the need for port 88 to stay open on the firewall. </span>The **GNOME Infrastructure** now runs the KDC Proxy and we&#8217;re glad to announce Kerberos authentications are working as expected on the hosted services.

If you are facing the same problem and you are curious to know more about the setup, here they come all the details:

On the **KDC**:

  1. No changes are needed on the KDC itself, just make sure to install the **python-kdcproxy** package which is available for RHEL 7, <a href="http://koji.fedoraproject.org/koji/taskinfo?taskID=7937527" target="_blank">HERE</a>.
  2. Tweak your vhost accordingly by following the <a href="https://github.com/npmccallum/kdcproxy" target="_blank">provided documentation</a>.

On the **client**:

  1. Install the krb5-workstation package, make sure it&#8217;s at least version **1.12.2-9** as that&#8217;s the release which had the additional features we are talking about backported. Right now it&#8217;s only available for Fedora 21.
  2. Adjust /etc/krb5.conf accordingly and finally get a TGT through kinit $userid@GNOME.ORG.

{{< highlight ini >}}
[realms]
 GNOME.ORG = {
  kdc = https://account.gnome.org/KdcProxy
  kpasswd_server = https://account.gnome.org/KdcProxy
}
{{< / highlight >}}

That should be all for today!
