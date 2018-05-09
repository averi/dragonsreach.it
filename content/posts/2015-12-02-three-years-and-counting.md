---
title: Three years and counting
author: Andrea Veri
type: post
date: 2015-12-02T16:32:41+00:00
url: /2015/12/02/three-years-and-counting/
categories:
  - Planet Debian
  - Planet Fedora
  - Planet GNOME
  - Planet Ubuntu
  - Sysadmin

---
It&#8217;s been a while since my last &#8220;what&#8217;s been happening behind the scenes&#8221; e-mail so I&#8217;m here to report on what has been happening within the GNOME Infrastructure, its future plans and my personal sensations about a challenge that started around three (3) years ago when <a href="https://plus.google.com/115250422803614415116/posts" target="_blank">Sriram Ramkrishna</a> and <a href="http://www.digitalprognosis.com/" target="_blank">Jeff Schroeder</a> proposed my name as a possible candidate for coordinating the team that runs the systems behind the GNOME Project. All this followed by the <a href="https://www.gnome.org/news/2013/03/behind-the-scene-andrea-veri-is-new-gnome-part-time-sysadmin/" target="_blank">official hiring</a> achieved by <a href="https://en.wikipedia.org/wiki/Karen_Sandler" target="_blank">Karen Sandler</a> back in February 2013.

The **GNOME Infrastructure** has finally reached stability both in terms of reliability and uptime, we didn&#8217;t have any service disruption this and the past year and services have been running smoothly as they were expected to in a project like the one we are managing.

As many of you know service disruptions and a total lack of maintenance were very common before I joined back in 2013, I&#8217;m so glad the situation has dramatically changed and developers, users, passionates are now able to reach our websites, code repositories, build machines without experiencing slowness, downtimes or
  
unreachability. Additionally all these groups of people now have a reference point they can contact in case they need help when coping with the infrastructure they daily use. The ticketing system allows users to get in touch with the members of the <span class="il">Sysadmin</span> Team and receive support right away within a very short period of time (Also thanks to <a href="https://www.pagerduty.com" target="_blank">Pagerduty</a>, service the Foundation is kindly sponsoring)

Before moving ahead to the future plans I&#8217;d like to provide you a summary of what has been done during these roughly three years so you can get an idea of why I define the changes that happened to the infrastructure a complete revamp:

  1. Recycled several ancient machines migrating services off of them while consolidating them by placing all their configuration on our central configuration management platform ran by Puppet. This includes a grand total of 7 machines that were replaced by new hardware and extended warranties the Foundation kindly sponsored.
  2. We strenghten our websites security by introducing SSL certificates everywhere and recently replacing them with SHA2 certificates.
  3. We introduced several services such as Owncloud, the Commits Bot, the Pastebin, the Etherpad, Jabber, the GNOME Github mirror.
  4. We restructured the way we backup our machines also thanks to the Fedora Project sponsoring the disk space on their backup facility. The way we were used to handle backups drastically changed from early years where a magnetic tape facility was in charge of all the burden of archiving our data to today where a NetApp is used together with <a href="http://www.nongnu.org/rdiff-backup/" target="_blank">rdiff-backup</a>.
  5. We upgraded Bugzilla to the latest release, a huge thanks goes to Krzesimir Nowak who kindly helped us building the migration tools.
  6. We introduced the <a href="https://wiki.gnome.org/Sysadmin/Apprentices" target="_blank">GNOME Apprentice program</a> open-sourcing our internal Puppet repository and cleansing it (shallow clones FTW!) from any sensitive information which now lives on a different repository with restricted access.
  7. We retired Mango and our OpenLDAP instance in favor of <a href="https://account.gnome.org" target="_blank">FreeIPA</a> which allows users to modify their account information on their own without waiting for the Accounts Team to process the change.
  8. We <a href="https://wiki.gnome.org/Sysadmin/SOP" target="_blank">documented</a> how our internal tools are customized to play together making it easy for future <span class="il">Sysadmin</span> Team members to learn how the infrastructure works and supersede existing members in case they aren&#8217;t able to keep up their position anymore.
  9. We started providing hosting to the GIMP and GTK projects which now completely rely on the GNOME Infrastructure. (DNS, email, websites and other services infrastructure hosting)
 10. We started providing hosting not only to the GIMP and GTK projects but to localized communities as well such as GNOME Hispano and GNOME Greece
 11. We configured proper monitoring for all the hosted services thanks to Nagios and Check-MK
 12. We migrated the IRC network to a newer ircd with proper IRC services (Nickserv, Chanserv) in place.
 13. We made sure each machine had a configured management (mgmt) and KVM interface for direct remote access to the bare metal machine itself, its hardware status and all the operations related to it. (hard reset, reboot, shutdown etc.)
 14. We <a href="https://mail.gnome.org/archives/infrastructure-announce/2013-May/msg00000.html" target="_blank">upgraded MoinMoin</a> to the latest release and made a substantial cleanup of old accounts, pages marked as spam and trashed pages.
 15. We deployed DNSSEC for several domains we manage including gnome.org, guadec.es, gnomehispano.es, guadec.org, gtk.org and gimp.org
 16. We <a href="https://mail.gnome.org/archives/infrastructure-announce/2014-March/msg00000.html" target="_blank">introduced an account de-activation policy</a> which comes into play when a contributor not committing to any of the hosted repositories at git.gnome.org since two years is caught by the script. The account in question is marked as inactive and the gnomecvs (from the old cvs days) and ftpadmin groups are removed.
 17. We planned mass reboots of all the machines roughly every month for properly applying security and kernel updates.
 18. We introduced <a href="http://mirrorbrain.org/" target="_blank">Mirrorbrain</a> (MB), the mirroring service serving GNOME and related modules tarballs and software all over the world. Before introducing MB GNOME had several mirrors located in all the main continents and at the same time a very low amount of users making good use of them. Many organizations and companies behind these mirrors decided to not host GNOME sources anymore as the statistics of usage were very poor and preferred providing the same service to projects that really had a demand for these resources. MB solved all this allowing a proper redirect to the closest mirror (through mod_geoip) and making sure the sources checksum matched across all the mirrors and against the original tarball uploaded by a GNOME maintainer and hosted at master.gnome.org.

I can keep the list going for dozens of other accomplished tasks but I&#8217;m sure many of you are now more interested in what the future plans actually are in terms of where the **GNOME Infrastructure** should be in the next couple of years.

One of the main topics we&#8217;ve been discussing will be migrating our Git infrastructure away from cgit (which is mainly serving as a code browsing tool) to a more complete platform that is surely going to include a code review tool of some sort. (Gerrit, Gitlab, Phabricator)

Another topic would be migrating our mailing lists to Mailman 3 / Hyperkitty. This also means we definitely need a staging infrastructure in place for testing these kind of transitions ideally bound to a separate Puppet / Ansible repository or branch. Having a different repository for testing purposes will also mean helping apprentices to test their changes directly on a live system and not on their personal computer which might be running a different OS / set of tools than the ones we run on the GNOME Infrastructure.

What I also aim would be seeing **GNOME** Accounts being the only authentication resource in use within the whole GNOME Infrastructure. That means one should be able to login to a specific service with the same username / password in use on the other hosted services. That&#8217;s been on my todo list for a while already and it&#8217;s probably time to push it forward together with Patrick Uiterwijk, responsible of <a href="https://fedorahosted.org/ipsilon/" target="_blank">Ipsilon</a>&#8216;s development at Red Hat and GNOME Sysadmin.

While these are the top priority items we are soon receiving new hardware (plus extended warranty renewals for two out of the three machines that had their warranty renewed a while back) and migrating some of the VMs off from the current set of machines to the new boxes is definitely another task I&#8217;d be willing to look at in the next couple of months (one machine (ns-master.gnome.org) is being decommissioned giving me a chance to migrate away from BIND into NSD).

The **GNOME Infrastructure** is evolving and it&#8217;s crucial to have someone maintaining it. On this side I&#8217;m bringing to your attention the fact the assigned <span class="il">Sysadmin</span> funds are running out as reported on the Board minutes from the <a href="https://mail.gnome.org/archives/foundation-announce/2015-November/msg00003.html" target="_blank">27th of October</a>. On this side <a href="http://jeff.ecchi.ca/" target="_blank">Jeff Fortin</a> started looking for possible sponsors and came up with the idea of making a brochure with a set of accomplished tasks that couldn&#8217;t have been possible without the <a href="https://mail.gnome.org/archives/foundation-announce/2010-June/msg00000.html" target="_blank"><span class="il">Sysadmin </span>fundraising campaign</a> launched by <a href="https://en.wikipedia.org/wiki/Stormy_Peters" target="_blank">Stormy Peters</a> back in <a href="http://stormyscorner.com/2010/03/one-step-closer-to-a-sys-admin.html" target="_blank">June 2010</a> being a success. The Board is well aware of the importance of having someone looking at the infrastructure that runs the GNOME Project and is making sure the brochure will be properly reviewed and published.

And now some stats taken from the Puppet Git Repository:

{{< highlight Bash >}}
$ cd /git/GNOME/puppet && git shortlog -ns

3520 Andrea Veri
506 Olav Vitters
338 Owen W. Taylor
239 Patrick Uiterwijk
112 Jeff Schroeder
71 Christer Edwards
4 Daniel Mustieles
4 Matanya Moses
3 Tobias Mueller
2 John Carr
2 Ray Wang
1 Daniel Mustieles García
1 Peter Baumgarten
{{< / highlight >}}

and from the <a href="https://www.bestpractical.com/rt/" target="_blank">Request Tracker</a> database (52388 being my assigned ID):

{{< highlight MySQL >}}
mysql> select count(*) from Tickets where LastUpdatedBy = '52388';
+----------+
| count(*) |
+----------+
| 3613 |
+----------+
1 row in set (0.01 sec)

mysql> select count(*) from Tickets where LastUpdatedBy = '52388' and Status = 'Resolved';
+----------+
| count(*) |
+----------+
| 1596 |
+----------+
1 row in set (0.03 sec){{< / highlight >}}

It&#8217;s been a long run which made me proud, for the things I learnt, for the tasks I&#8217;ve been able to accomplish, for the great support the GNOME community gave me all the time and most of all for the same fact of being part of the team responsible of the systems hosting the GNOME Project. **Thank you** GNOME community for your continued and never ending backing, we daily work to improve how the services we host are delivered to you and the support we receive back is fundamental for our passion and enthusiasm to remain high!

&nbsp;
