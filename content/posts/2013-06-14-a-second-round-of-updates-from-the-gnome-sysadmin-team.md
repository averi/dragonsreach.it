---
title: A second round of updates from the GNOME Sysadmin Team
author: Andrea Veri
type: post
date: 2013-06-14T11:37:06+00:00
url: /2013/06/14/a-second-round-of-updates-from-the-gnome-sysadmin-team/
categories:
  - Planet GNOME
  - Sysadmin
tags:
  - infrastructure
  - Planet GNOME
  - sysadmin

---
![nagios.gnome.org](/wp-content/uploads/2013/06/nagios.gnome_.org_.png)

I haven&#8217;t been blogging so much in the past months as I actually promised myself I would have but given the fact a lot has been done on the **GNOME Infrastructure** lately it&#8217;s time for me to announce all the updates we did since my <a href="http://www.dragonsreach.it/2013/03/07/some-updates-from-the-gnome-sysadmin-team/" target="_blank">latest blog post</a>. So here we come with all the items we&#8217;ve been looking at recently:


  * Our main **LDAP** istance was moved from a very ancient machine (which unfortunately died with a broken disk a few weeks ago) to a newer box that currently contains several other admin tools like Mango and Daily Reports. (a little script written by **Owen Taylor** for creating and storing several reports mainly related to backups and SSL certificates expiration dates) In addition to migrating our LDAP master to a newer machine, we did configure and setup replication to an LDAP slave to share a bit the load and most of all to link all the external (machines outside the RH&#8217;s internal network) machines to it.

  * A lot of efforts have been spent in the so-called &#8220;**Puppet-ization**&#8221; (Puppet allows you to reproduce a complete environment with just a few commands, it&#8217;s very very handy in the case of host&#8217;s migrations) and several new modules are now stored into our internal Puppet repository. Specifically all the **Iptables** rules are currently managed in a centralized way, each node has its own rules and policies, finally there&#8217;s no need to ssh into each machine to retrieve the information we need for a specific firewall. In addition to the Iptables class, also Cobbler, Owncloud, Jabberd, Denyhosts and several other modules have been properly configured and currently reside in Puppet.

  * Another item that had top priority on the list was setting up another &#8220;webapps&#8221; virtual machine to migrate several services from one of the existing ancient machines to it. I can finally tell that the GNOME Infrastructure has get rid of all the old machines, all the services have been migrated to newer machines and most of all all the services are currently being served through SSL. (git, planet, www.gnome.org, l10n, guadec.org, bugzilla, blogs, developer, help, people, news etc.) In regard of SSL and **Bugzilla**, we&#8217;ve configured our Bugzilla istance to serve attachments through a secondary domain which will look like: **_https://bug-id.bugzilla-attachments.gnome.org_**, this to prevent cross-site scripting attacks in a better way than what we did before.

  * We&#8217;ve also spent some time working on our Nagios istance hosted at **https://nagios.gnome.org**. We&#8217;ve improved it dramatically by adding several new checks and covering all the services we currently take care of but that&#8217;s not all. **Event Handlers** have been setup to help us addressing problems right after they occur on our web servers. The Nagios event handlers are currently configured to read the status of a specific Nagios service and in the case the status is set to CRITICAL, they restart httpd once, which is usually enough in the case of random Apache&#8217;s timeouts. But that&#8217;s again not all. A public view for **Nagios** is now ready, every single GNOME contributor and developer should be able to check the current status of all the services we maintain just by loggin in with the username &#8220;**anonymous**&#8221; and the password &#8220;**anonymous**&#8221; at nagios.gnome.org.

  * Our wiki was upgraded to the latest **MoinMoin** release, at the moment of writing, version **1.9.7**. This release introduces stronger password hashes, please make sure to update your password as soon as you can to strenghten the security of your account. It was also clear that live.gnome.org was behaving a bit sluggish lately, we spent some time cleaning up spammers, old and deleted pages and things started flushing way better. More details about the cleanup can be found at <https://mail.gnome.org/archives/foundation-list/2013-May/msg00098.html>. (we cleaned up around 23000 spammers!)

  * The most exciting things I usually love to announce are new services. While I always prefer keeping the number of maintained services as low as possible it was time for the GNOME Infrastructure to broaden its horizons satisfying the requests coming from the community and the developers involved into the project. I won&#8217;t spend any more word about this since I&#8217;m sure you are all waiting for me to list the new services, so here they are:

   *  A completely new **Jabber** service hosted at jabber.gnome.org and accessible by all the GNOME Foundation members requesting access to it. More details about it can be found at <a href="https://live.gnome.org/Sysadmin/Jabber" target="_blank">https://live.gnome.org/Sysadmin/Jabber</a>.
   * GNOME is extensively using IRC as its main communication tool, thus we&#8217;ve improved our Services IRC Bot to use a plugin called **MeetBot**. Having a meeting and storing the logs in a public web server is currently possible with a really minor effort of learning a few commands to administer the plugin correctly. If you are going to have a meeting and you want to make use of MeetBot, make sure that Services is there, and give a look at <http://meetbot.gnome.org/Manual.html>.
   * Do you want to be always up-to-date with the status of the GNOME Infrastructure? and are you actually wondering what&#8217;s the best way to do so? if yes, you should probably have a look at <a href="http://status.gnome.org" target="_blank">http://status.gnome.org</a>. This service makes use of <a href="http://git.fedorahosted.org/git/fedora-status" target="_blank">Fedora-Status</a> by **Patrick Uiterwijk** and allows the GNOME Sysadmin Team to let everyone know whether there is a problem with any of the services listed in the page. This service, together with the public view for Nagios and the brand new <a href="https://mail.gnome.org/mailman/listinfo/infrastructure-announce" target="_blank">infrastructure-announce@gnome.org </a>mailing list will definitely help everyone finding out what&#8217;s going on, where and how long it&#8217;ll take for the issue to be fixed.
   * It took me a lot of pain having the **KGB IRC Collaboration bot** packaged into the EPEL repositories but I finally managed to set it up on the GNOME Infrastructure. KGB has become very handy since the time Cia.vc closed its hosting and it&#8217;s available for anyone requesting access to it at **irc.gnome.org**. If you are looking for Git commit notifications of a specific module directly on your IRC channel, this is what you want :-)
   * An Owncloud instance is also available, more reading about what are the requirements for requesting access to it at the following <a href="https://live.gnome.org/MembershipCommittee/MembershipBenefits#Account_on_cloud.gnome.org" target="_blank">link</a>.
   * An Etherpad istance is also available at <a href="http://etherpad.gnome.org" target="_blank">https://etherpad.gnome.org</a> to all the GNOME Teams that need it! Please drop me an e-mail at <av at gnome dot org> if you are interested. (Pad&#8217;s creation is currently disabled for preventing spam)
   * Build.gnome.org has been revived and it&#8217;s currently hosting an <a href="https://live.gnome.org/OSTree" target="_blank">OSTree</a> istance. GNOME Daily images are costantly being generated, interested in testing one the those images? give this <a href="http://worldofgnome.org/how-to-try-gnome-os-yes-gnome-os/" target="_blank">article</a> a look.

That&#8217;s all for now! See you all at **GUADEC** and thanks everyone for all the hints, suggestions and mails you&#8217;ve been sending me in the past months! And a special thanks to **Ekaterina Gerasimova** for taking the time to brainstorm with me suggesting new features and improvements over the GNOME Infrastructure!

![status.gnome.org](/wp-content/uploads/2013/06/status.gnome_.org_.png)
