---
title: The GNOME Infrastructure is now powered by FreeIPA!
author: Andrea Veri
type: post
date: 2014-10-07T09:21:33+00:00
url: /2014/10/07/the-gnome-infrastructure-is-now-powered-by-freeipa/
categories:
  - Planet Debian
  - Planet Fedora
  - Planet GNOME
  - Planet Ubuntu
tags:
  - freeipa
  - gnome.org
  - OpenLDAP
  - rhel
  - SSSD
  - sysadmin

---
As preannounced <a href="https://mail.gnome.org/archives/infrastructure-announce/2014-October/msg00000.html" target="_blank">here</a> the GNOME Infrastructure switched to a new Account Management System which is reachable at <a href="https://account.gnome.org" target="_blank">https://account.gnome.org</a>. All the details will follow.

## Introduction

It&#8217;s been a while since someone actually touched the underlying authentication infrastructure that powers the GNOME machines. The very first setup was originally configured by **Jonathan Blandford** (jrb) who configured an **OpenLDAP** istance with several customized schemas. (pServer fields in the old CVS days, pubAuthorizedKeys and GNOME modules related fields in recent times)

While OpenLDAP-server was living on the GNOME machine called clipboard (aka ldap.gnome.org) the clients were configured to synchronize users, groups, passwords through the nslcd daemon. After several years **Jeff Schroeder** joined the Sysadmin Team and during one cold evening (date is Tue, February 1st 2011) spent some time configuring SSSD to replace the nslcd daemon which was missing one of the most important SSSD features: caching. What surely convinced Jeff to adopt SSSD (a very new but promising sofware at that time as the first release happened right before 2010&#8217;s Christmas) and as the commit log also states (&#8220;New sssd module for ldap information caching&#8221;) was SSSD&#8217;s caching feature.

It was enough for a certain user to log in once and the &#8216;/var/lib/sss/db&#8217; directory was populated with its login information preventing the LDAP daemon in charge of picking up login details (from the LDAP server) to query the LDAP server itself every single time a request was made against it. This feature has definitely helped in many occasions especially when the LDAP server was down for a particular reason and sysadmins needed to access a specific machine or service: without SSSD this wasn&#8217;t ever going to work and sysadmins were probably going to be locked out from the machines they were used to manage. (except if you still had &#8216;/etc/passwd&#8217;, &#8216;/etc/group&#8217; and &#8216;/etc/shadow&#8217; entries as fallback)

Things were working just fine except for a few downsides that appeared later on:

  1. the web interface (view) on our LDAP user database was managed by <a href="https://wiki.gnome.org/Attic/Mango" target="_blank"><strong>Mango</strong></a>, an outdated tool which many wanted to rewrite in Django that slowly became a huge dinosaur nobody ever wanted to look into again
  2. the Foundation membership information were managed through a MySQL database, so two databases, two sets of users unrelated to each other
  3. users were not able to modify their own account information on their own but even a single e-mail change required them to mail the GNOME Accounts Team which was then going to authenticate their request and finally update the account.

Today&#8217;s infrastructure changes are here to finally say the issues outlined at (1, 2, 3) are now fixed.

## What has changed?

The GNOME Infrastructure is now powered by Red Hat&#8217;s **FreeIPA** which bundles several FOSS softwares into one big &#8220;bundle&#8221; all surrounded by an easy and intuitive web UI that will help users update their account information on their own without the need of the Accounts Team or any other administrative entity. Users will also find two custom fields on their &#8220;Overview&#8221; page, these being &#8220;Foundation Member since&#8221; and &#8220;Last Renewed on date&#8221;. As you may have understood already we finally managed to migrate the Foundation membership database into LDAP itself to store the information we want once and for all. As a side note it might be possible that some users that were Foundation members in the past won&#8217;t find any detail stored on the Foundation fields outlined above. That is actually expected as we were able to migrate all the current and old Foundation members that had an LDAP account registered at the time of the migration. If that&#8217;s your case and you still would like the information to be stored on the new setup please get in contact with the Membership Committee at stating so.

## Where can I get my first login credentials?

Let&#8217;s make a little distinction between users that previously had access to Mango (usually maintainers) and users that didn&#8217;t. If you were used to access Mango before you should be able to login on the new Account Management System by entering your GNOME username and the password you were used to use for loggin in into Mango. (after loggin in the very first time you will be prompted to update your password, please choose a strong password as this account will be unique across all the GNOME Infrastructure)

If you never had access to Mango, you lost your password or the first time you read the word Mango on this post you thought &#8220;why is he talking about a fruit now?&#8221; you should be able to reset it by using the following command:

{{< highlight Bash >}}
ssh -l yourgnomeuserid account.gnome.org
{{< / highlight >}}

The command will start an SSH connection between you and account.gnome.org, once authenticated (with the SSH key you previously had registered on our Infrastructure) you will trigger a command that will directly send your brand new password on the e-mail registered for your account. From my tests seems GMail sees the e-mail as a phishing attempt probably because the body contains the word &#8220;password&#8221; twice. That said if the e-mail won&#8217;t appear on your INBOX, please **double-check** your Spam folder.

## Now that Mango is gone how can I request a new account?

With Mango we used to have a form that automatically e-mailed the maintainer of the selected GNOME module which was then going to approve / reject the request. From there and in the case of a positive vote from the maintainer the Accounts Team was going to create the account itself.

With the <a href="https://mail.gnome.org/archives/gnome-i18n/2014-February/msg00000.html" target="_blank">recent introduction of a commit robot directly on l10n.gnome.org</a> the number of account requests reduced its numbers. In addition to that users will now be able to perform pretty much all the needed maintenance on their accounts themselves. That said and while we will probably work on building a form in the future we feel that requesting accounts can definitely be achieved directly by mailing the Accounts Team itself which will mail the maintainer of the respective module and create the account. As just said the number of account creations has become very low and the queue is currently clear. The documentation has been updated to reflect these changes at:

<a href="https://wiki.gnome.org/AccountsTeam" target="_blank">https://wiki.gnome.org/AccountsTeam</a>
  
<a href="https://wiki.gnome.org/AccountsTeam/NewAccounts" target="_blank">https://wiki.gnome.org/AccountsTeam/NewAccounts</a>

## I was used to have access to a specific service but I don&#8217;t anymore, what should I do?

The migration of all the user data and ACLs has been massive and I&#8217;ve been spending a lot of time reviewing the existing HBAC rules trying to spot possible errors or misconfigurations. If you happen to not being able to access a certain service as you were used to in the past, please get in contact with the Sysadmin Team. All the possible ways to contact us are available at <a href="https://wiki.gnome.org/Sysadmin/Contact" target="_blank">https://wiki.gnome.org/Sysadmin/Contact</a>.

## What is missing still?

Now that the Foundation membership information has been moved to LDAP I&#8217;ll be looking at porting some of the existing membership scripts to it. What I managed to port already are welcome e-mails for new or existing members. (renewals)

Next step will be generating a membership page from LDAP (to populate <a href="http://www.gnome.org/foundation/membership" target="_blank">http://www.gnome.org/foundation/membership</a>) and all the your-membership-is-going-to-lapse e-mails that were being sent till today.

## Other news &#8211; /home/users mount on master.gnome.org

You will notice that loggin in into **master.gnome.org** will result in your home directory being empty, don&#8217;t worry, you did not lose any of your files but master.gnome.org is now currently hosting your home directories itself. As you may have been aware of adding files to the public_html directory on master resulted in them appearing on your **people.gnome.org/~userid** space. That was unfortunately expected as both master and webapps2 (the machine serving people.gnome.org&#8217;s webspaces) were mounting the same GlusterFS share.

We wanted to prevent that behaviour to happen as we wanted to know who has access to what resource and where. From today master&#8217;s home directories will be there just as a temporary spot for your tarballs, just scp and use ftpadmin against them, that should be all you need from master. If you are interested in receiving or keeping using your people.gnome.org&#8217;s webspace please mail <a href="mailto:accounts@gnome.org" target="_blank"><accounts AT gnome DOT org></a> stating so.

## Other news &#8211; a shiny and new error 500 page has been deployed

Thanks to **Magdalen Berns** (magpie) a new error 500 web page has been deployed on all the Apache istances we host. The page contains an iframe of **status.gnome.org** and will appear every single time the web server behind the service you are trying to reach will be unreachable for maintenance or other purposes. While I hope you won&#8217;t see the page that often you can still enjoy it at <a href="https://static.gnome.org/error-500/500.html" target="_blank">https://static.gnome.org/error-500/500.html</a>. Make sure to whitelist status.gnome.org on your browser as it currently loads it without https. (as the service is currently hosted on OpenShift which provides us with a *.rhcloud.com wildcard certificate, which differs from the CN the browser would expect it to be)

## Updates

**UPDATE** on status.gnome.org&#8217;s SSL certificate: the certificate has been provisioned and it should result in the 500&#8217;s page to be displayed correctly with no warnings from your browser.

**UPDATE** from Adam Young on Kerberos ports being closed on many DC&#8217;s firewalls:

> The next version of upstream MIT Kerberos will have support for fetching a ticket via ports 443 and marshalling the request over HTTPS. We’ll need to run a proxy on the server side, but we should be able to make it work:
> 
> Read up here
  
> <a href="http://adam.younglogic.com/2014/06/kerberos-firewalls/" rel="nofollow">http://adam.younglogic.com/2014/06/kerberos-firewalls</a>
