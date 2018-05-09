---
title: Manage passwords with ‘pass’
author: Andrea Veri
type: post
date: 2013-12-24T15:51:20+00:00
url: /2013/12/24/manage-passwords-with-pass/
categories:
  - Linux
  - Planet Debian
  - Sysadmin
tags:
  - encrypt
  - gpg
  - pass
  - passwords

---
Fighting with passwords have always been one of my favorite battles in the past and unfortunately the former always won. I never liked using the root user that much for administering a machine and made a massive use of sudo, I won&#8217;t list all the benefits of using sudo, but the <a href="https://help.ubuntu.com/community/RootSudo#Benefits_of_using_sudo" target="_blank">following wiki page</a> has a pretty nice overview of them.

Said that, when using sudo it&#8217;s definitely ideal to combine a **strong password** that is also easy to remember and type again when prompted. Sadly strong passwords that are also easy to remember can be considered an oxymoron. How hard would it be to recall a 30+ chars long password? Honestly that would be close to impossible for an human being but what if a little software available on the major GNU/Linux distributions could handle that for us? That&#8217;s where _pass_ comes handy, but what is pass? from the pass manpage itself:

> pass is a very simple password store that keeps passwords inside gpg2(1) encrypted files inside a simple directory tree residing at ~/.password-store. The pass utility provides a series of commands for manipulating the password store, allowing the user to add, remove, edit, synchronize, generate, and manipulate passwords.

I&#8217;m sure that a lot of you guys have been looking for a tool like this one for ages: _pass_ allows you to generate very strong passwords with <a href="http://linux.die.net/man/1/pwgen" target="_blank">pwgen</a>, GPG encrypt them with your GPG Key, store them safely on your disk and make them available whenever you need them with a single command. But let&#8217;s move to the practice, give the following steps a try and enjoy how powerful your pass setup will be.

### First setup

Install the software:

{{< highlight Bash >}}yum/apt-get install pass{{< / highlight >}}

 Generate a GPG Key if you don&#8217;t have one already, a detailed guide can be found <a href="http://fedoraproject.org/wiki/Creating_GPG_Keys#Creating_GPG_Keys_Using_the_Command_Line" target="_blank">here</a>. Initialize your passwords storage. (**GPGKEYID** can be retrieved by running **gpg &#8211;list-keys** and then looking for a line similar to this one: **pub 4096R/B3A6223D 2012-06-25**)

{{< highlight Bash >}}pass init GPGKEYID{{< / highlight >}}

Generate your first password and call it &#8216;sudo_password&#8217; given you are going to make use of it as your brand new sudo password. (we want it at least 30+ chars long)

{{< highlight Bash >}}pass generate sudo_password 30{{< / highlight >}}

(Optional) Create as much passwords as you need and make sure to save them with unique names, that way you will be able to identify what a password is used for easily.

{{< highlight Bash >}}pass generate gmail_password 30{{< / highlight >}}

### Additional maintenance commands on your password database

Look at the existing passwords on your database.

{{< highlight Bash >}}pass ls{{< / highlight >}}

Result:

{{< highlight Bash >}}Password Store
├── gmail_password
├── sudo_password
└── root_password{{< / highlight >}}

Manually edit a password.

{{< highlight Bash >}}pass edit password_name{{< / highlight >}}

Remove a password from your database.

{{< highlight Bash >}}pass rm password_name{{< / highlight >}}

Copy a password on your clipboard and paste it.

{{< highlight Bash >}}pass -c password_name{{< / highlight >}}

Are you wondering if _pass_ supports a VCS? Yeah, it does, it currently allows you to manage your passwords database with Git, so that each applied change to the database will be tracked through a VCS so that you won&#8217;t forget when and how you updated a specific password.
