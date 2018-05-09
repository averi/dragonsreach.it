---
title: Backup your Gmail in a few easy steps!
author: Andrea Veri
type: post
date: 2011-07-25T22:50:57+00:00
url: /2011/07/26/backup-your-gmail-in-a-few-easy-steps/
aktt_notify_twitter:
  - no
tmac_last_id:
  - 276277242479382530
categories:
  - Gmail
tags:
  - Backup
  - Getmail
  - Gmail

---
I&#8217;ve actually spent a few hours searching around for a good backup solution for my mailbox until I decided to stick with <a href="http://pyropus.ca/software/getmail/" target="_blank">getmail</a>.  What you&#8217;ll be able to achieve after reading this HowTo and deploying the following setup is:

  1. A full backup of your e-mail DATA in the Mbox format. (yes, Gmail&#8217;s labels / folders as well)
  2. Prevent getmail to mark all mails as **read** after delivering them. (this was a pretty bad issue since getmail was marking all my mails as read even if I did not access my e-mail at all)
  3. Keep your backups **up-to-date** with the **latest** content from your mailbox. (by default getmail grabs all the DATA from your mailbox and fills up the Mbox / Maildir content keeping deleted mails. So let&#8217;s say I deleted a mail two days ago, well it&#8217;ll still appear on today&#8217;s backups. This behaviour is definitely unwanted)

I&#8217;ll now move to explain a few details about my new configuration but before moving to tweak getmail&#8217;s main config file, please do the following change on **_retrieverbases.py**\* :

{{< highlight Python >}}return self._getmsgpartbyid(msgid, '(RFC822)'){{< / highlight >}}

**to**

{{< highlight Python >}}return self._getmsgpartbyid(msgid, '(BODY.PEEK[])'){{< / highlight >}}

When done grab the following _**getmailrc**_ and adapt it to your needs\*\*:

{{< highlight INI >}}
[retriever]
type = SimpleIMAPSSLRetriever ## or SimplePOP3SSLRetriever.
server = imap.gmail.com ## or pop.gmail.com for POP3.
username = example@gmail.com
password = password

## so-called Gmail's labels should be listed one by one here for getmail to retrieve mail from them successfully.

mailboxes = ("INBOX", "[Gmail]/Sent mail",
"ubuntu", "gnome/example", "linux/example")

[destination]
type = Mboxrd
path = ~/.getmail/backup.mbox

[options]
delivered_to = false ## No delivered_to header added automatically.
received = false ## No received header added automatically.
verbose = 2 ## getmail will print messages about each of its actions.
{{< / highlight >}}

When done we should go ahead setting up getmail&#8217;s directories and config file:

{{< highlight Bash >}}
mkdir $HOME/.getmail
cp $HOME/getmailrc $HOME/.getmail/
{{< / highlight >}}

Adapt **$HOME/getmailrc** to whatever dir you put that file into. But&#8230;pretty much all the remaining work will be done by a small shell script I wrote:

{{< highlight Bash >}}
#!/bin/sh

WORKDIR=$HOME/.getmail
date=`date "+%d-%m-%Y_%H:%M"`

if [ ! -f  $WORKDIR/backup.mbox ]
then
touch $WORKDIR/backup.mbox
fi

getmail > $WORKDIR/getmail.log
OUT=$?
if [ $OUT -eq 0 ]
then
mkdir -p $WORKDIR/backups/ && { mv $WORKDIR/backup.mbox $WORKDIR/backups/backup_$date.mbox ;}
else [ $OUT -eq 1 ]
exit 1
fi

## Cleanup older than 3 days backups
find $WORKDIR/backups/* -mtime +3 -exec rm {} ;
cd $WORKDIR && { rm -rf oldmail-* ;}
{{< / highlight >}}

This script will:

  1. Run getmail using the getmailrc config file you previously worked on.
  2. If the above command will be successful, it&#8217;ll create a_ **backups**_ dir into **$HOME/.getmail** and move the latest Mbox file there appending a date and time to its name. (by doing this we are sure next getmail run will happen on an empty **backup.mbox** file, thus it will just contain the **latest** content from your mailbox)
  3. It&#8217;ll re-create a **backup.mbox** file on **$HOME/.getmail** to avoid the next getmail run to fail.
  4. In the end, it&#8217;ll clean up older than 3 days backups to avoid a too crowded **backups** folder. (it removes the oldmail file as well since it is useless in our case)

In the end set up a cronjob that will run the above script and generate the backups for you every one hour:

{{< highlight Bash >}}
0 * * * * $HOME/.getmail/getmail_run.sh > /dev/null
{{< / highlight >}}

Feel free to let me know if you&#8217;ve encountered any issue while following the above HowTo. Enjoy!

\* /usr/share/getmail4/getmailcore/**_retrieverbases.py** on line **901**.

\*\* More documentation about the _**getmailrc**_ file and syntax can be found on getmail&#8217;s <a href="http://pyropus.ca/software/getmail/configuration.html#conf-retriever" target="_blank">documentation</a> page.
