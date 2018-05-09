---
title: 'Automatic Gmail’s Trash & Spam folders cleanup'
author: Andrea Veri
type: post
date: 2011-07-03T21:29:35+00:00
url: /2011/07/03/automatic-gmails-trash-spam-folders-cleanup/
aktt_notify_twitter:
  - no
tmac_last_id:
  - 276277242479382530
categories:
  - Gmail
tags:
  - Cleanup
  - Gmail
  - Spam
  - Trash

---
Since some time I&#8217;ve been thinking about a possible way to delete my **Gmail&#8217;s Trash & Spam folders** content automatically without having to bother doing it manually every single time I wanted to check my mail and clean it up. (I **love** keeping everything in place and having my Trash&Spam folders empty as they should be makes me pretty **happy**)

A few years ago when Mutt was my main mail client I had the need to filter my mail through IMAP and while googling around for that I found out a **great** piece of software: <a href="https://github.com/lefcha/imapfilter" target="_blank">imapfilter</a>. Today while analyzing the above quoted issue I suddenly told myself: &#8220;Hey, but why don&#8217;t you use your dear and old friend imapfilter to fulfil your needs?&#8221;

After a few minutes I came up with a small _lua_ script that was doing exactly what I wanted: my **Trash&Spam folders** are no longer crowded and I finally don&#8217;t have to delete mails twice! But here they come a few details about my script:

{{< highlight INI >}}
options.timeout = 120
options.subscribe = true

account = IMAP {
 server = 'imap.gmail.com',
 username = 'example@gmail.com',
 password = 'password',
 ssl = 'ssl3'
 }

trash = account['[Gmail]/Trash']:is_undeleted()
account['[Gmail]/Trash']:delete_messages(trash)

spam = account['[Gmail]/Spam']:is_unanswered()
account['[Gmail]/Spam']:delete_messages(spam)
{{< / highlight >}}

The script does two things:

  1. It checks whether a mail is **not** marked as &#8220;deleted&#8221; (moving an e-mail into the Trash does **not** mark it as &#8220;**to be deleted**&#8221; automatically) already and removes it.
  2. It checks whether a mail on the Spam folder has been **answered** (I never had to answer a single e-mail contained into my Spam folder)  and if **not** removes it.

Using the above script is really easy (you should run imapfilter on **interactive mode** first to generate Gmail&#8217;s **certificates,** do that before having cron to run the script for you or otherwise it&#8217;ll just hang), just make sure to have **imapfilter** installed on your system and then run it through cron every half an hour or less depending on your needs:

{{< highlight Bash >}}
crontab -e

*/30 * * * * imapfilter -c /home/user/imapfilter.lua >> /home/user/imapfilter.log
{{< / highlight >}}

Please also remember to setup appropriate permissions on the config file since it contains your Gmail&#8217;s **password** and most of all make sure that your Spam folder is **visible** through IMAP (this option can be found on the **label** menu available under your <a href="https://mail.google.com/mail/?hl=it&#038;&#038;shva=1#settings" target="_blank">Gmail&#8217;s settings</a>) otherwise imapfilter will just report an error.

Enjoy!

&nbsp;
