---
title: A few other additions to my Mutt and Desktop setup!
author: Andrea Veri
type: post
date: 2011-09-11T18:25:32+00:00
url: /2011/09/11/a-few-other-additions-to-my-mutt-and-desktop-setup/
aktt_notify_twitter:
  - no
tmac_last_id:
  - 276277235311312896
categories:
  - Irssi
  - Mutt
tags:
  - Goobook
  - Login Screen
  - Mutt

---
A few days ago I <a href="https://www.dragonsreach.it/2011/09/04/new-desktop-mutt-and-irssi-setup/" target="_blank">blogged</a> about my main computer&#8217;s configuration files and desktop&#8217;s appearance and today I managed to add a few little tweaks to those, they are:

  * Google&#8217;s <a href="https://www.google.com/contacts" target="_blank">contacts list</a> integrated into Mutt
  * a cleaner and nicer Login screen

Curious to know how you can easily integrate your Google&#8217;s contacts into Mutt? Well, you should be able to achieve that within a few minutes after reading this small **HowTo**:

**1.** Download and install **goobook** as explained <a href="http://pypi.python.org/pypi/goobook/1.3a1#source-installation" target="_blank">here</a>.

**2.** Setup a **_.goobookrc_** file into your Home directory. It should look like this:

<div>
  <pre>
machine google.com
login example@gmail.com
password yourpassword
</pre>
</div>

**3.** Add the relevant configuration bits into your **_/etc/Muttrc_** file:

<pre>set query_command="goobook query '%s'"
bind editor  complete-query
macro index,pager a ";goobook add" "Add sender's address to your Google contacts"</pre>

**4.** Your configuration should be good to go now, so here&#8217;s a few examples on goobook&#8217;s **usage** within Mutt:

  * Use **TAB** if you want to auto-complete a mail address when specifying the **To:** field.
  * Use the **A** key if you want to add sender&#8217;s address to your Google contacts.
  * Use the **Q** key for querying your contacts list.

We can now move on on customizing your Login Screen running GDM3. Let&#8217;s begin with a **screenshoot**:

![login_screen](/wp-content/uploads/2011/09/login_screen.png)

I definitely love it, it&#8217;s **clear** and **clean** and most of all it has everything I need, no extra toolbars or menus. If you agree with me, open up the **_/etc/gdm3/greeter.gconf-defaults_** file and do the needed changes. This is how your greeter.gconf-defaults file should look like:

<pre>/desktop/gnome/background/picture_filename      /path/to/your/dusty-bg/file # dusty's background can be downloaded <a href="http://gnome-look.org/content/show.php/Dusty?content=94332" target="_blank">here</a>.
/desktop/gnome/interface/gtk_theme              Darklooks # this is my main theme, feel free to adapt that to your needs.
/apps/gdm/simple-greeter/logo_icon_name         debian-swirl # this is the default on Debian's systems.
/desktop/gnome/sound/event_sounds               false # I don't like hearing any sound when when I am prompted to insert my user's details on the Login Screen.
/apps/gdm/simple-greeter/disable_user_list      true # users list will be disabled, you won't be able to select your username from a list but you'll have to insert that yourself.
/apps/metacity/general/compositing_manager      false # default, no need to change this.
/apps/gnome-power-manager/ui/icon_policy        never # default, no need to change this.</pre>

We are close to the end but we are missing an important **detail**: how can you safely remove bottom&#8217;s toolbar and menus for a clearer and cleaner Login Screen? Open up the **/var/lib/gdm3/.gconf.mandatory/%gconf-tree.xml** file, search for the **\<dir name="general"\>** section and apply the following **change**:

<pre>- &lt;entry name="compositing_manager" mtime="1315580582" type="bool" value="false"/>;
+ &lt;entry name="compositing_manager" mtime="1315580582" type="bool" value="true"/></pre>

But what if you prefer keeping the toolbar as it is, but you definitely don&#8217;t like seeing the _**Accessibility**_ icon appearing on your Login Screen? On the same file as above, search for the **\<dir name="general"\>** section and **modify** the following string as it follows:

<pre>- &lt;entry name="enable" mtime="1315580582" type="bool" value="true"/>
+ &lt;entry name="enable" mtime="1315580582" type="bool" value="false"/></pre>

See you on my next blog post and don&#8217;t forget to have a look at my GitHub&#8217;s <a href="https://github.com/averi/config-files" target="_blank">repository</a>! Oh&#8230;and follow <a href="http://twitter.com/andrea_veri" target="_blank">me</a> on Twitter!
