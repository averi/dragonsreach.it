---
title: Building Debian packages with Deb-o-Matic
author: Andrea Veri
type: post
date: 2012-09-17T13:07:14+00:00
url: /2012/09/17/building-debian-packages-with-deb-o-matic/
aktt_notify_twitter:
  - no
tmac_last_id:
  - 276277228277473281
categories:
  - Planet Debian
  - Planet Ubuntu
tags:
  - Deb-o-Matic
  - Planet Debian

---
Today I&#8217;ll be telling you about an interesting way to build your Debian packages using **Deb-o-Matic**, a tool developed and maintained by <a href="http://dktrkranz.wordpress.com" target="_blank">Luca Falavigna</a>. Some more details about this tool from the package&#8217;s description:

> Deb-o-Matic is an easy to use build machine for Debian source packages based on pbuilder, written in Python.
> 
> It provides a simple tool to automate build of source packages with limited user interaction and a simple configuration. It has some useful features such as automatic update of pbuilder, automatic scan and selection of source packages to build and modules support.

The **setup.**

Download the package.

{{< highlight Bash >}}apt-get install debomatic{{< / highlight >}}

Modify the **main** configuration file as follows:

{{< highlight ini >}}[default]
builder: pbuilder
packagedir: /home/john/debomatic # Take note of the following path since we'll need it for later use.
configdir: /etc/debomatic/distributions
pbuilderhooks: /usr/share/debomatic/pbuilderhooks
maxbuilds: 3 # The number of builds you can perform at the same time.
inotify: 1
sleep: 60 # The sleep time between one build and another.
logfile: /var/log/debomatic.log

[gpg]
gpg: 0 # Change to 1 if you want Deb-O-Matic to check the GPG signature of the uploaded packages.
keyring: /etc/debomatic/debomatic.gpg # Add the GPG Keys you want Deb-O-Matic to accept in this keyring.

[modules]
modules: 1 # A list of all the available modules will follow right after.
modulespath: /usr/share/debomatic/modules

[runtime]
alwaysupdate: unstable experimental precise
distblacklist:
modulesblacklist: Lintian Mailer
mapper: {'sid': 'unstable',
'wheezy': 'testing',
'squeeze': 'stable'}

[lintian]
lintopts: -i -I -E --pedantic # Run Lintian in Pedantic mode.

[mailer] # You need an SMTP server running on your machine for the mailer to work. You can have a look at the 'Ssmtp' daemon which is a one-minute-setup MTA, check an example over <a href="https://github.com/averi/config-files/blob/master/backups/offlineimap%20%2B%20ssmtp%20%2B%20imapfilter/ssmtp.conf" target="_blank">here</a>.
fromaddr: debomatic@localhost
smtphost: localhost
smtpport: 25
authrequired: 0
smtpuser: user
smtppass: pass
success: /etc/debomatic/mailer/build_success.mail-template # Update the build success or failure mails as you wish by modifying the relevant files.
failure: /etc/debomatic/mailer/build_failure.mail-template

[internals]
configversion = 010a{{< / highlight >}}

The available modules are:

  1.  &#8220;**Contents**&#8220;, which acts as a &#8216;dpkg -c&#8217; over the built packages.
  2. &#8220;**DateStamp**&#8220;, which displays build start and finish times into a file in the build directory.
  3. &#8220;**Lintian**&#8220;, which stores Lintian output on top of the built package in the pool directory.
  4. &#8220;**Mailer**&#8220;, which sends a reply to the uploader once the build has finished.
  5. &#8220;**PrevBuildCleaner**&#8220;, which deletes all files generated by the previous build.
  6. &#8220;**Repository**&#8220;, which generates a local repository of built packages.

Configure &#8216;<a href="http://manpages.ubuntu.com/manpages/precise/man5/dput.cf.5.html" target="_blank">dput</a>&#8216; to upload package&#8217;s sources to your local repository, edit the **/etc/dput.cf** file and add this entry:

{{< highlight ini >}}[debomatic]
method = local
incoming = /home/john/debomatic

or the following if you are going to upload the files to a different machine through SSH:

[debomatic]
login = john
fqdn = debomatic.example.net
method = scp
incoming = /debomatic
{{< / highlight >}}

Add a new **Virtual Host** on Apache and access the repository / built packages directly through your browser:

{{< highlight Apache >}}<VirtualHost *:80>

ServerAdmin john@example.net
ServerName debomatic.example.net
DocumentRoot /home/john/debomatic

<Directory /home/john/debomatic>
Options Indexes FollowSymLinks MultiViews
AllowOverride None
Order allow,deny
allow from all
<Directory>

</VirtualHost>{{< / highlight >}}

Start the daemon:

{{< highlight Bash >}}sudo /etc/init.d/debomatic start{{< / highlight >}}

(**Optional**) Add your repository to **APT**&#8216;s sources.list:

{{< highlight Bash >}}deb http://debomatic.example.net/ unstable main contrib non-free{{< / highlight >}}

(**Optional**) Start Deb-O-Matic at system&#8217;s startup by modifying the **/etc/init.d/debomatic** file at **line 21**:

{{< highlight Bash >}}- [ -x "$DAEMON" ] || exit 0
- [ "$DEBOMATIC_AUTOSTART" = 0 ] && exit 0

+ [ -x "$DAEMON" ] || exit 0
+ [ "$DEBOMATIC_AUTOSTART" = 1 ] && exit 0{{< / highlight >}}

and finally add it to the desired **runlevels**:

{{< highlight Bash >}}update-rc.d debomatic defaults{{< / highlight >}}

Enjoy!
