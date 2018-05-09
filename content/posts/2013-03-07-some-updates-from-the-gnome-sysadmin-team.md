---
title: Some updates from the GNOME Sysadmin Team
author: Andrea Veri
type: post
date: 2013-03-07T14:49:26+00:00
url: /2013/03/07/some-updates-from-the-gnome-sysadmin-team/
categories:
  - Planet GNOME
  - Sysadmin
tags:
  - infrastructure
  - Planet GNOME
  - sysadmin

---
It&#8217;s been more than a month now since I started looking into the many outstanding items we had waiting on our To Do list here at the **GNOME Infrastructure**. A lot has been done and a lot has yet to come during the next months, but I would like to share with you some of the things I managed to look at during these weeks.

As you may understand many Sysadmin&#8217;s tasks are not perceived at all by users especially the ones related to the so-called &#8220;Puppet-ization&#8221; which refers to the process of creating / modifying / improving our internal Puppet repository. A lot of work has been done on that side and several new modules have been added, specifically Cobbler, Amavisd, SpamAssassin, ClamAV, Bind, Nagios / Check_MK (enabling Apache eventhandlers for automatic restart of faulty httpd processes), Apache.

Another top priority item was migrating some of our services off to the old physical machines to virtual machines I did setup earlier. The machines that are now recycled are the following: (Two more are missing on the list, specifically window (which still hosts art.gnome.org, people.gnome.org and projects.gnome.org due to be migrated to another host in the next weeks) and label. (which still hosts our Jabberd, an <a href="https://mail.gnome.org/archives/foundation-list/2013-March/msg00000.html" target="_blank">interesting discussion</a> on its future is currently ongoing on Foundation-list)

  1. **menubar**, our old Postfix host served the GNOME Foundation since 2004 and processed millions of e-mails from and to @gnome.org addresses.
  2. **container**, our old main NFS node was serving the GNOME Foundation since 2003, it hosted our mail archives, our FTP archives and all the /home/users/* directories.
  3. **button** hosted many services (MySQL databases, LDAP, Mango) and served the Foundation since 2004, a faulty hardware took it down on January 2012.

And if you ever wanted to see how menubar, container and button look like, I have two photos for you with the machines being pulled out the **GNOME** rack:

![old-machines](/wp-content/uploads/2013/03/old-machines.jpg)
<br>
![old-machines-2](/wp-content/uploads/2013/03/old-machines-2.jpg)

Some of the things you may have perceived directly on your skin should be the following:

  1. Our live.gnome.org istance has been upgraded to the latest MoinMoin stable release, 1.9.6.
  2. The Services bot has been added to the **GIMPNET** network and currently manages all GNOME channels, it currently acts as a Nickserv, Chanserv. More information about how you can register your nickname and gain the needed ACLs at the following <a href="https://live.gnome.org/Sysadmin/IRC" target="_blank">wiki page</a>.
  3. Several **GNOME** services and domains are now covered by SSL as you may have noticed on planet.gnome.org, news.gnome.org, blogs.gnome.org, l10n.gnome.org, git.gnome.org, help.gnome.org, developer.gnome.org.
  4. Re-design of our Mailman archives as you can see at [https://mail.gnome.org/archives/foundation-list][3]. A big &#8220;thank you&#8221; goes to Olav Vitters for taking the time to rebuild our &#8220;archive&#8221; script from Perl to Python. About the Mailman topic, someone proposed me the use of <a href="https://fedorahosted.org/hyperkitty/" target="_blank">HyperKitty</a>, that&#8217;s something we will evaluate in the next coming months but I find it a very interesting alternative to the current mail archiving.

What should you expect next?

  1. **Bugzilla** will be moved to another virtual machine and will be upgraded to the latest release.
  2. An **Owncloud** istance will be setup for all the GNOME Foundation members and GNOME Teams that will need access to it.
  3. A discussion will be started for setting up a **Gitorious** istance on the GNOME Infrastructure.
  4. A long-term item will be rewriting <a href="https://live.gnome.org/Mango" target="_blank">Mango</a> in Django and adding several other features to it than the ones it has now. (ideally voting for Board elections, logins for managing your LDAP information such as your @gnome.org&#8217;s alias forward, shutdown of old an unused accounts after a certain period of time, automatic @gnome.org&#8217;s alias creation after the &#8220;Foundation Membership&#8221; flag is selected on LDAP, etc.)

Thanks a lot for all the mails I&#8217;ve received during these weeks containing reports and suggestions about how we should improve our Infrastructure! Please stay tuned, a lot more news are yet to come!

 [1]: http://www.dragonsreach.it/wp-content/uploads/2013/03/old-machines.jpg
 [2]: http://www.dragonsreach.it/wp-content/uploads/2013/03/old-machines-2.jpg
 [3]: https://mail.gnome.org/archives/foundation-list/
