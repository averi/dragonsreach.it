---
title: Configuring DNSSEC on your personal domain
author: Andrea Veri
type: post
date: 2013-11-13T13:53:31+00:00
url: /2013/11/13/configuring-dnssec-personal-domain/
categories:
  - DNS
  - Planet Debian
  - Sysadmin
tags:
  - BIND
  - DNSSEC

---
Today I&#8217;ll be working out how to properly configure <a href="http://en.wikipedia.org/wiki/Domain_Name_System_Security_Extensions" target="_blank">DNSSEC</a> on a **BIND9** installation, I&#8217;ll also make sure to give you all the needed instructions to properly verify if a specific domain is being correctly covered by DNSSEC itself. In addition to that a few more details will be provided about adding the relevant **SSHFP**&#8216;s entries on your DNS zone files to be able to automatically verify the authenticity of your domain when connecting to it with SSH avoiding any possible <a href="http://en.wikipedia.org/wiki/Man-in-the-middle_attack" target="_blank">MITM</a> attack.

First of all, let&#8217;s create the Zone Signing Key (**ZSK**) which is the key that will be responsible to sign any other record on the zone file which is not a DNSKEY record itself:

{{< highlight Bash >}}dnssec-keygen -a RSASHA1 -b 1024 -n ZONE gnome.org{{< / highlight >}}

**Note:** the dnssec-keygen binary file should be part of bind97 (RHEL 5) or bind (RHEL6) package according to yum whatprovides:

RHEL 5:

{{< highlight Bash >}}32:bind97-9.7.0-17.P2.el5_9.2.x86_64 : The Berkeley Internet Name Domain (BIND) DNS (Domain Name System) server
Repo : rhel-x86_64-server-5
Matched from:
Filename : /usr/sbin/dnssec-keygen{{< / highlight >}}

RHEL 6:

{{< highlight Bash >}}32:bind-9.8.2-0.17.rc1.el6.3.x86_64 : The Berkeley Internet Name Domain (BIND) DNS (Domain Name System) server
Repo : rhel-x86_64-server-6
Matched from:
Filename : /usr/sbin/dnssec-keygen{{< / highlight >}}

<span style="line-height: 1.5;">Then, create the Key Signing Key (<strong>KSK</strong>), which will be used to sign all the DNSKEY records:</span>

{{< highlight Bash >}}dnssec-keygen -a RSASHA1 -b 2048 -n ZONE -f KSK gnome.org{{< / highlight >}}

Creating the above keys can take several minutes, when done copy the public keys to the zone file this way:

{{< highlight Bash >}}cat Kgnome.org*.key >> gnome.org{{< / highlight >}}

When done you can clean out the useless bits from the zone file and just leave the DNSKEY records (which are not commented out as you will notice)

An additional and cleaner way of accomplishing the above is to use the INCLUDE rule on the zone file itself as follows:

{{< highlight Bash >}}$INCLUDE /srv/dnssec-keys/Kgnome.org+005+12345.key
$INCLUDE /srv/dnssec-keys/Kgnome.org+005+67890.key{{< / highlight >}}

Choosing which method to use is really up to you.

<span style="line-height: 1.5;">Once that is done you can go ahead and sign the zone file. As of myself I&#8217;m making use of the </span><a style="line-height: 1.5;" href="http://infrastructure.fedoraproject.org/cgit/dns.git/tree/do-domains" target="_blank">do-domain</a> <span style="line-height: 1.5;">script taken from the Fedora Infrastructure Team&#8217;s repositories. If you are going to use it yourself, make sure to adjust all the relevant variables to match your setup, especially </span><strong style="line-height: 1.5;">keyspath</strong><span style="line-height: 1.5;">, </span><strong style="line-height: 1.5;">region_zones</strong><span style="line-height: 1.5;">, </span><strong style="line-height: 1.5;">template_zone</strong><span style="line-height: 1.5;">, </span><strong style="line-height: 1.5;">signed_zones</strong> <span style="line-height: 1.5;">and </span><strong style="line-height: 1.5;"><span style="line-height: 1.5;">ARE</span></strong><span style="line-height: 1.5;"><strong>A</strong>. The do-domain script also checks your zone file through </span><strong style="line-height: 1.5;">named-checkzone</strong> <span style="line-height: 1.5;">before signing it.</span>

![/me while editing the do-domains script with the preview of gnome-code-assistance!"](/wp-content/uploads/2013/11/gedit-code-assistance.png)

If instead you don&#8217;t want to use the script above, you can sign the zone file manually in the following way:

{{< highlight Bash >}}dnssec-signzone -K /path/to/your/dnssec/keys -e +3024000 -N INCREMENT gnome.org{{< / highlight >}}

By default, the above command will append &#8216;.signed&#8217; to the file name, you can modify that behaviour by appending the &#8216;-f&#8217; flag to the dnssec-signzone call. The &#8216;-N INCREMENT&#8217; will increment the serial number automatically making use of the RFC 1982 arithmetics while the &#8216;-e&#8217; flag will extend the zone signature end date from the default 30 days to 35. (this way we can safely run a monthly cron job that will sign the zone file automatically)

You can make use of the following script to achieve the above:

{{< highlight Bash >}}#!/bin/sh
SIGNZONE="/usr/sbin/dnssec-signzone"
DNSSEC_KEYS="/srv/dnssec-keys"
NAMEDCHROOT="/var/named/chroot"
ZONEFILES="gnome.org"

cd $NAMEDCHROOT

for ZONE in $ZONEFILES; do
$SIGNZONE -K $DNSSEC_KEYS -e +3024000 -f $ZONE.signed -N INCREMENT $ZONE
done

/sbin/service named reload{{< / highlight >}}

Once the zone file has been signed just make sure to include it on named.conf and restart named:

{{< highlight Bash >}}zone "gnome.org" {
file "gnome.org.signed";
};{{< / highlight >}}

When you&#8217;re done with that we should be moving ahead adding a **DS** record for our domain at our domain registrar. My example is taken from the known **gandi.net** registrar.

![Gandi DNSSEC Interface](/wp-content/uploads/2013/11/gandi.png)

Select **KSK (257)** and **(RSA/SHA-1)** on the dropdown list and paste your public key on the box. You will find the public key you need on one of the Kgnome.org*.key files, you should look for the DNSKEY 257 entry as &#8216;_dig DNSKEY gnome.org_&#8216; shows:

{{< highlight Bash >}};; ANSWER SECTION:
gnome.org. 888 IN DNSKEY 257 3 5 AwEAAbRD7AymDFuKc2iXta7HXZMleMkUMwjOZTsn4f75ZUp0of8TJdlU DtFtqifEBnFcGJU5r+ZVvkBKQ0qDTTjayL54Nz56XGGoIBj6XxbG8Es+ VbZCg0RsetDk5EsxLst0egrvOXga27jbsJ+7Me3D5Xp1bkBnQMrXEXQ9 C43QfO2KUWJVljo1Bii3fTfnHSLRUsbRn8Puz+orK71qxs3G9mgGR6rm n91brkpfmHKr3S9Rbxq8iDRWDPiCaWkI7qfASdFk4TLV0gSVlA3OxyW9 TCkPZStZ5r/WRW2jhUY/kjHERQd4qX5dHAuYrjJSV99P6FfCFXoJ3ty5 s3fl1RZaTo8={{< / highlight >}}

Once that is done you should have a fully covered DNSSEC domain, you can verify that this way:

{{< highlight Bash >}}dig . DNSKEY | grep -Ev '^($|;)' > root.keys

dig +sigchase +trusted-key=./root.keys gnome.org. A | cat -n{{< / highlight >}}

The result:

{{< highlight Bash >}}105 ;; WE HAVE MATERIAL, WE NOW DO VALIDATION
106 ;; VERIFYING DS RRset for org. with DNSKEY:59085: success
107 ;; OK We found DNSKEY (or more) to validate the RRset
108 ;; Ok, find a Trusted Key in the DNSKEY RRset: 19036
109 ;; VERIFYING DNSKEY RRset for . with DNSKEY:19036: success
110
111 ;; Ok this DNSKEY is a Trusted Key, DNSSEC validation is ok: SUCCESS{{< / highlight >}}

**Bonus content: Adding SSHFP entries for your domain and verifying them**

You can retrieve the SSHFP entries for a specific host with the following command:

{{< highlight Bash >}}ssh-keygen -r $(hostname --fqdn) -f /etc/ssh/ssh_host_rsa_key.pub{{< / highlight >}}

When retrieved just add the **SSHFP** entry on the zone file for your domain and verify it:

{{< highlight Bash >}}ssh -oVerifyHostKeyDNS=yes -v subdomain.gnome.org{{< / highlight >}}

Or directly add the above parameter into your /etc/ssh/ssh_config file this way:

{{< highlight Bash >}}VerifyHostKeyDNS=yes{{< / highlight >}}

And run &#8216;ssh -v subdomain.gnome.org&#8217;, the result you should receive:

{{< highlight Bash >}}debug1: Server host key: RSA 00:39:fd:1a:a4:2c:6b:28:b8:2e:95:31:c2:90:72:03
debug1: found 1 secure fingerprints in DNS
debug1: matching host key fingerprint found in DNS
debug1: ssh_rsa_verify: signature correct{{< / highlight >}}

That&#8217;s it! Enjoy!
