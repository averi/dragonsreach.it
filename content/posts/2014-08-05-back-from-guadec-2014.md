---
title: Back from GUADEC 2014
author: Andrea Veri
type: post
date: 2014-08-05T17:27:23+00:00
url: /2014/08/05/back-from-guadec-2014/
categories:
  - Planet GNOME
tags:
  - GUADEC

---
Coming back from **GUADEC** has never been easy, so much fun, so much great people to speak with and amazing talks to watch but this year has definitely been harder as I totally felt in love with the city that was hosting the event. Honestly speaking I&#8217;ve been amazed by how Strasbourg looks like: alsace houses and buildings are just delightful, the cathedral is stunning and people have been so welcoming during my whole stay. (cooks at the Canteen even prepared a few italian dishes and welcomed us in italian every time we were heading there&#8230;how cool is that?)

But let&#8217;s get back to our business now as I would probably never stop talking about Strasbourg and how great it was staying there! I did not have a personal talk this year but I presented the yearly Sysadmin Team report during the Foundation&#8217;s AGM. If you weren&#8217;t there all the slides are available <a href="https://people.gnome.org/~av/sysadmin-team-report/guadec2014.html" target="_blank">here</a>.

Apart from presenting what we did and what the changes we introduced on the GNOME Infrastructure were I participated to Patrick Uiterwijk&#8217;s talk about FedOAuth and all the upcoming changes that are planned on the infrastructure during the next months. If you were not able to attend Patrick&#8217;s talk this little resume should be for you:

**Current problems**:

  * The GNOME Infrastructure currently has a lot of different user databases which implies different users and passwords across the services we host
  * The Foundation&#8217;s database is currently MySQL-based while we do have LDAP in place for all our other needs already
  * Some of the tools we do use for managing our LDAP istance are not being maintained properly

**Possible solutions**:

  * Introduce FedOAuth, a SSO solution written and developed by Patrick Uiterwijk
  * Unify the various databases and make sure our LDAP istance is used for authentication everywhere
  * Remove Mango and configure FreeIPA

**Benefits after the move**:

  * Users will be able to manage their accounts on their own, no more need to poke the accounts team for updating passwords, emails, SSH keys. The accounts team will still be around to adjust ACLs
  * No more need for dozen of accounts, one for every single service we provide
  * More freedom when managing sudo accesses and accounts on the various machines we manage, this will help new people contributing to the Sysadmin Team (Making our puppet repository public and introducing a GNOME Infrastructure Apprentice group for newcomers is something we will be seriously evaluating after the FreeIPA move)

**Where we are now:**

  * Our SSO infrastructure is live at <a href="https://id.gnome.org" target="_blank">https://id.gnome.org</a>
  * Your OpenID URL is **https://$GNOME_USERID.id.gnome.org**
  * Right now you can login with your GNOME account at the following services: l10n.gnome.org, opw.gnome.org. We are slowly migrating all the existing services to the new SSO infrastructure, please be patient and bear with us!

More information, slides and screenshots from Patrick&#8217;s talk are available <a href="http://patrick.uiterwijk.org/2014/07/28/gnome-authentication/#1" target="_blank">here</a>. Stay tuned and many thanks to the **GNOME Foundation** for sponsoring my travel and accomodation expenses!


{{< gallery hover-effect="none" caption-position="center" >}}
  {{< figure caption="The GUADEC 2014 group photo!" src="/wp-content/uploads/2014/08/14581494979_18c62c3083_z.jpg" >}}
  {{< figure caption="A view of the Petite France district" src="/wp-content/uploads/2014/08/IMG_20140729_193107.jpg" >}}
  {{< figure caption="Another great view of the Petite France district" src="/wp-content/uploads/2014/08/IMG_20140729_195206.jpg" >}}
  {{< figure caption="The Notre Dame cathedral" src="/wp-content/uploads/2014/08/IMG_20140725_103921.jpg" >}}
  {{< figure caption="Another view of the Notre Dame cathedral" src="/wp-content/uploads/2014/08/IMG_20140725_103927.jpg" >}}
  {{< figure caption="Place Gutenberg" src="/wp-content/uploads/2014/08/IMG_20140725_103655.jpg" >}}
{{< /gallery >}}

&nbsp;
