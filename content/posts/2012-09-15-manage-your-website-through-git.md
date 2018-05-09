---
title: Manage your website through Git
author: Andrea Veri
type: post
date: 2012-09-15T12:52:08+00:00
url: /2012/09/15/manage-your-website-through-git/
aktt_notify_twitter:
  - no
tmac_last_id:
  - 276277228277473281
categories:
  - Git
  - Sysadmin
tags:
  - Git
  - Manage
  - Website

---
Ever wondered how you can update your **website** (in our case a static website with a bunch of **HTML** and **PHP** files) by committing to a **Git repository** hosted on a **different server**? if the answer to the previous question is **yes**, then you are in the right place.

The **scenario**:

 * Website hosted on **server A**.

 * Git repository hosted on **server B**.

and a few details about why would you opt for maintaining your website through **Git**:

  1. You need multiple people to access the static content of your website and you also want to maintain all the history of changes together with all the Git&#8217;s magic.
  2. You think using an FTP server is not secure enough.
  3. You think giving out SSH access or more permissions on the server to multiple users it&#8217;s not what you want. (also using **<a href="http://linux.die.net/man/1/scp" target="_blank">scp</a>** will overwrite the files directly with all its consequences)

the **setup**, on **server A**:

Clone the Git repository over the directory that your Web server will serve.

{{< highlight Bash >}}cd /srv/www.domain.com/http && sudo git clone http://git.example.com/git/domain.git{{< / highlight >}}

Grab the needed package:

{{< highlight Bash >}}apt-get install fishpolld or yum install fishpolld{{< / highlight >}}

Set up a &#8220;<a href="http://git.fishsoup.net/cgit/fishpoll/tree/README" target="_blank">topic</a>&#8221; (called **website_update**) that will call a &#8216;**git pull**&#8216; each time the repository hosted on **server B** receives an update. (the file has to be placed into the **/etc/fishpoll.d** directory)

{{< highlight Bash >}}#!/bin/bash

PATH="/usr/local/sbin:/usr/local/bin:/sbin:/bin:/usr/sbin:/usr/bin"
WEBSITEDIR="/srv/www.domain.com/http"

if [ -d "${WEBSITEDIR}" ]; then
cd "${WEBSITEDIR}"
else
echo "Unable to access theme directory. Failing."
exit 1
fi

git pull{{< / highlight >}}

Add a little configuration file that will enable the **website_update**&#8216;s topic at daemon&#8217;s startup. (you should name it **website_update.conf**)

{{< highlight Bash >}}[fishpoll]
on_start = True{{< / highlight >}}

Open the relevant port on **Iptables** so that **server A** and **server B** can communicate as expected, the daemon runs over** port 27527**.

{{< highlight Bash >}}-A INPUT -m state --state NEW -m tcp -p tcp -s <strong>server-B-IP</strong> --dport 27527 -j ACCEPT{{< / highlight >}}

Start the daemon. (by default, logs are sent to **syslog** but you can run the daemon in **Debug** mode by using the **-D flag**)

{{< highlight Bash >}}sudo /etc/init.d/fishpolld start or sudo systemctl start fishpolld.service or sudo /usr/sbin/fishpolld -D{{< / highlight >}}

and on **server B:**

Grab the needed package:

{{< highlight Bash >}}apt-get install fishpoke or yum install fishpoke{{< / highlight >}}

Configure the relevant Git hook (**post-update**, located into the **hooks** directory) and make it **executable**.

{{< highlight Bash >}}#!/bin/sh

echo "Triggering update of configuration on server A"
fishpoke server-A-IP-or-DNS website_update{{< / highlight >}}

Finally test the whole setup by committing to the repository hosted on **server B** and verify your changes being sent live on your website!
