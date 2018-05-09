---
title: Adding reCAPTCHA support to Mailman
author: Andrea Veri
type: post
date: 2014-05-03T09:37:23+00:00
url: /2014/05/03/adding-recaptcha-support-to-mailman/
categories:
  - Linux
  - Mailman
  - Planet Debian
  - Sysadmin
tags:
  - captchas
  - mailman
  - Planet Debian
  - rhel

---
The **GNOME** and many other infrastructures have been recently attacked by an huge amount of **subscription-based spam** against their **Mailman** istances. What the attackers were doing was simply launching a GET call against a specific REST API URL passing all the parameters it needed for a subscription request (and confirmation) to be sent out. Understanding it becomes very easy when you look at the following example taken from our apache.log:

{{< highlight verilog >}}May 3 04:14:38 restaurant apache: 81.17.17.90, 127.0.0.1 - - [03/May/2014:04:14:38 +0000] "GET /mailman/subscribe/banshee-list?email=example@me.com&fullname=&pw=123456789&pw-conf=123456789&language=en&digest=0&email-button=Subscribe HTTP/1.1" 403 313 "http://spam/index2.html" "Mozilla/5.0 (Windows NT 6.1; WOW64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/34.0.1847.131 Safari/537.36"
{{< / highlight >}}

As you can the see attackers were sending all the relevant details needed for the subscription to go forward (and specifically the full name, the email, the digest option and the password for the target list). At first we tried to either stop the spam by banning the subnets where the requests were coming from, then when it was obvious that more subnets were being used and manual intervention was needed we tried banning their **User-Agents**. Again no luck, the spammers were smart enough to change it every now and then making it to match an existing browser User-Agent. (with a good percentage to have a lot of false-positives)

Now you might be wondering why such an attack caused a lot of issues and pain, well, the attackers made use of addresses found around the web for their malicius subscription requests. That means we received a lot of emails from people that have never heard about the GNOME mailing lists but received around **10k subscription requests** that were seemingly being sent by themselves.

It was obvious we needed to look at a backup solution and luckily someone on our support channel suggested the freedesktop.org sysadmins recently added CAPTCHAs support to Mailman.  I&#8217;m now sharing the patch and providing a few more details on how to properly set it up on either DEB or RPM based distributions. Credits for the patch should be given to Debian Developer <span style="font-weight: bold; color: #222222;">Tollef Fog Heen</span><span style="color: #222222;">, who has been so kind to share it with us.</span>

Before patching your installation make sure to install the **python-recaptcha** package (tested on Debian with Mailman 2.1.15) on DEB based distributions and **python-recaptcha-client** on RPM based distributions. (I personally tested it against Mailman release 2.1.15, RHEL 6)

### The Patch

{{< highlight Python >}}diff --git a/Mailman/Cgi/listinfo.py b/Mailman/Cgi/listinfo.py
index 4a54517..d6417ca 100644
--- a/Mailman/Cgi/listinfo.py
+++ b/Mailman/Cgi/listinfo.py
@@ -30,6 +31,8 @@ from Mailman import Errors
 from Mailman import i18n
 from Mailman.htmlformat import *
 from Mailman.Logging.Syslog import syslog
+from recaptcha.client import captcha
 
 # Set up i18n
 _ = i18n._
@@ -200,6 +203,9 @@ def list_listinfo(mlist, lang):
     replacements[''] = mlist.FormatFormStart('listinfo')
     replacements[''] = mlist.FormatBox('fullname', size=30)
 
+    # Captcha
+    replacements['mm-recaptcha-javascript'] = captcha.displayhtml(mm_cfg.RECAPTCHA_PUBLIC_KEY, use_ssl=False)
+
     # Do the expansion.
     doc.AddItem(mlist.ParseTags('listinfo.html', replacements, lang))
     print doc.Format()
diff --git a/Mailman/Cgi/subscribe.py b/Mailman/Cgi/subscribe.py
index 7b0b0e4..c1c7b8c 100644
--- a/Mailman/Cgi/subscribe.py
+++ b/Mailman/Cgi/subscribe.py
@@ -21,6 +21,8 @@ import sys
 import os
 import cgi
 import signal
+from recaptcha.client import captcha
 
 from Mailman import mm_cfg
 from Mailman import Utils
@@ -132,6 +130,17 @@ def process_form(mlist, doc, cgidata, lang):
     remote = os.environ.get('REMOTE_HOST',
                             os.environ.get('REMOTE_ADDR',
                                            'unidentified origin'))
+
+    # recaptcha
+    captcha_response = captcha.submit(
+        cgidata.getvalue('recaptcha_challenge_field', ""),
+        cgidata.getvalue('recaptcha_response_field', ""),
+        mm_cfg.RECAPTCHA_PRIVATE_KEY,
+        remote,
+        )
+    if not captcha_response.is_valid:
+        results.append(_('Invalid captcha'))
+
     # Was an attempt made to subscribe the list to itself?
     if email == mlist.GetListEmail():
         syslog('mischief', 'Attempt to self subscribe %s: %s', email, remote)
{{< / highlight >}}

### Additional setup

Then on the _/var/lib/mailman/templates/en/listinfo.html_ template (right below _<mm-digest-question-end>_)  add:

{{< highlight html >}}<tr>
  <td>
    Please fill out the following captcha
  </td>
  <td>
    <mm-recaptcha-javascript>
  </TD>
</tr>
{{< / highlight >}}

Make also sure to generate a public and private key at <a href="https://www.google.com/recaptcha" target="_blank">https://www.google.com/recaptcha</a> and add the following paramaters on your _mm_cfg.py_ file:

  * RECAPTCHA\_PRIVATE\_KEY
  * RECAPTCHA\_PUBLIC\_KEY

Loading **reCAPTCHAs** images from a trusted **HTTPS** source can be done by changing the following line:

{{< highlight html >}}replacements['<mm-recaptcha-javascript>'] = captcha.displayhtml(mm_cfg.RECAPTCHA_PUBLIC_KEY, use_ssl=False)
{{< / highlight >}}

to

{{< highlight html >}}replacements['<mm-recaptcha-javascript>'] = captcha.displayhtml(mm_cfg.RECAPTCHA_PUBLIC_KEY, use_ssl=True)
{{< / highlight >}}

### EPEL 6 related details

A few additional details should be provided in case you are setting this up against a **RHEL 6** host: (or any other machine using the EPEL 6 package _python-recaptcha-client-1.0.5-3.1.el6_)

Importing the recaptcha.client module will fail for some strange reason, importing it correctly can be done this way:

{{< highlight Bash >}}ln -s /usr/lib/python2.6/site-packages/recaptcha/client /usr/lib/mailman/pythonlib/recaptcha
{{< / highlight >}}

and then fix the imports also making sure _sys.path.append(&#8220;/usr/share/pyshared&#8221;)_ is not there:

{{< highlight Python >}}from recaptcha import captcha
{{< / highlight >}}

That&#8217;s not all, the package still won&#8217;t work as expected given the API\_SSL\_SERVER, API\_SERVER and VERIFY\_SERVER variables on captcha.py are outdated (filed as <a href="https://bugzilla.redhat.com/show_bug.cgi?id=1093855" target="_blank">bug #1093855</a>), substitute them with the following ones:

{{< highlight ini >}}
API_SSL_SERVER="https://www.google.com/recaptcha/api"
API_SERVER="http://www.google.com/recaptcha/api"
VERIFY_SERVER="www.google.com"
{{< / highlight >}}

And then on line 76:

{{< highlight ini >}}
url = "https://%s/recaptcha/api/verify" % VERIFY_SERVER,
{{< / highlight >}}

### reCAPTCHA v2

Google&#8217;s reCAPTCHA v1 will be deactivated starting from the 31th of March 2018, read more on how to migrate your Mailman install to version 2 [here][1].

That should be all! Enjoy!

 [1]: https://www.dragonsreach.it/2018/02/26/adding-recaptcha-v2-support-mailman/
