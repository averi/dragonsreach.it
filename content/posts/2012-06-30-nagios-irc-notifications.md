---
title: Nagios IRC Notifications
author: Andrea Veri
type: post
date: 2012-06-30T12:59:28+00:00
url: /2012/06/30/nagios-irc-notifications/
aktt_notify_twitter:
  - no
tmac_last_id:
  - 276277235105812480
categories:
  - Nagios
  - Sysadmin
tags:
  - IRC
  - Nagios
  - Notifications

---
Lately (as I earlier pointed out on my [blog][1]) I&#8217;ve been working on improving GNOME&#8217;s infrastructure monitoring services. After configuring XMPP it was time to find out a good way for sending out relevant notifications to our IRC channel hosted on GIMPNET. I achieved that with a nice combo: **supybot** + **supybot-notify**, all that mixed up with a few grains of Nagios command definitions.

But here we go with a little step-by-step guide:

**Requirements**

1. Install supybot and configure a new installation:

{{< highlight Bash >}}apt-get install supybot or yum install supybot
mkdir /home/$user/nagbot && cd /home/$user/nagbot
supybot-wizard (follow the directions to get the bot initially configured){{< / highlight >}}

2. Install and load the supybot-notify plugin by doing:

{{< highlight Bash >}}git clone git://git.fedorahosted.org/supybot-notify.git && cd supybot-notify
mkdir -p /home/$user/nagbot/plugins/notify && cp -r * /home/$user/nagbot/plugins/notify{{< / highlight >}}

Finally, **[load][2]** the plugin. (this will require you to [authenticate][3] to the bot)

**Nagios configuration**

Add the relevant command definitions to the **commands.cfg** file:

{{< highlight Bash >}}# 'notify-by-ircbot' command definition
define command{
    command_name    notify-by-ircbot
    command_line    /usr/bin/printf "%b" "#channel $NOTIFICATIONTYPE$ - $HOSTALIAS$/$SERVICEDESC$ is $SERVICESTATE$: $SERVICEOUTPUT$ ($$(hostname -s))" | nc -w 1 localhost 5050
    }

# 'host-notify-by-ircbot' command definition
define command{
	command_name	host-notify-by-ircbot
	command_line	/usr/bin/printf "%b" "#channel $NOTIFICATIONTYPE$ - $HOSTALIAS$ is $HOSTSTATE$: $HOSTOUTPUT$ ($$(hostname -s))" | nc -w 1 localhost 5050
	}{{< / highlight >}}

\* adjust the **Netcat&#8217;s** host and port to your needs, in my case Supybot and Nagios were running on the same host. In the case of a Supybot running on a different host than Nagios, tweak Iptables to allow the desired port:

{{< highlight Bash >}}-A INPUT -m state --state NEW,ESTABLISHED -m tcp -p tcp --dport 5050 -j ACCEPT{{< / highlight >}}

Add a new entry on the **contacts.cfg** file:

{{< highlight Bash >}}define contact{
 	contact_name    nagbot
   	use             generic-contact        
        alias		Nagios IRC Bot
        email           example@example.com
        service_notification_commands   notify-by-ircbot
 	host_notification_commands      host-notify-by-ircbot
}{{< / highlight >}}

Reload Nagios:

{{< highlight Bash >}}sudo /etc/init.d/nagios3 reload{{< / highlight >}}

And finally, enjoy the result:

{{< highlight Bash >}}PROBLEM - $hostalias/load average is CRITICAL: CRITICAL - load average: 30.45, 16.24, 7.16 (nagioshost)
 RECOVERY - $hostalias/load average is OK: OK - load average: 0.06, 0.60, 3.65 (nagioshost){{< / highlight >}}

 [1]: http://blogs.gnome.org/woody/2012/02/18/nagios-xmpp-notifications-for-gtalk/
 [2]: http://supybook.fealdia.org/devel/#_plugins
 [3]: http://supybook.fealdia.org/devel/#_identifying_to_the_bot
