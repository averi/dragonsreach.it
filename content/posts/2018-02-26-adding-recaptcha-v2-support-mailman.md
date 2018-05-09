---
title: Adding reCAPTCHA v2 support to Mailman
author: Andrea Veri
type: post
date: 2018-02-26T15:13:09+00:00
url: /2018/02/26/adding-recaptcha-v2-support-mailman/
categories:
  - Mailman
  - Planet Debian
  - Planet Fedora
  - Planet GNOME
  - Planet Ubuntu
  - Sysadmin

---
As a follow-up to the reCAPTCHA v1 [post][1] published back in 2014 here it comes an updated version for migrating your Mailman instance off from version 1 (being decommissioned on the 31th of March 2018) to version 2. The [original python-recaptcha library][2] was forked into <https://github.com/redhat-infosec/python-recaptcha> and made compatible with reCAPTCHA version 2.

The relevant changes against the original library can be resumed as follows:

  1. Added &#8216;version=2&#8217; against displayhtml, load_scripts functions
  2. Introduce the v2submit (along with submit to keep backwards compatibility) function to support reCAPTCHA v2
  3. The updated library is backwards compatible with version 1 to avoid unexpected code breakages for instances still running version 1

The required changes are located on the following files:

**/usr/lib/mailman/Mailman/Cgi/listinfo.py**

{{< highlight Python >}}
--- listinfo.py	2018-02-26 14:56:48.000000000 +0000
+++ /usr/lib/mailman/Mailman/Cgi/listinfo.py	2018-02-26 14:08:34.000000000 +0000
@@ -31,6 +31,7 @@
 from Mailman import i18n
 from Mailman.htmlformat import *
 from Mailman.Logging.Syslog import syslog
+from recaptcha.client import captcha
 
 # Set up i18n
 _ = i18n._
@@ -244,6 +245,10 @@
     replacements['<mm-lang-form-start>'] = mlist.FormatFormStart('listinfo')
     replacements['<mm-fullname-box>'] = mlist.FormatBox('fullname', size=30)
 
+    # Captcha
+    replacements['<mm-recaptcha-javascript>'] = captcha.displayhtml(mm_cfg.RECAPTCHA_PUBLIC_KEY, use_ssl=True, version=2)
+    replacements['<mm-recaptcha-script>'] = captcha.load_script(version=2)
+
     # Do the expansion.
     doc.AddItem(mlist.ParseTags('listinfo.html', replacements, lang))
     print doc.Format()
{{<  / highlight >}}
 
**/usr/lib/mailman/Mailman/Cgi/subscribe.py**

{{< highlight Python >}}
--- subscribe.py	2018-02-26 14:56:38.000000000 +0000
+++ /usr/lib/mailman/Mailman/Cgi/subscribe.py	2018-02-26 14:08:18.000000000 +0000
@@ -32,6 +32,7 @@
 from Mailman.UserDesc import UserDesc
 from Mailman.htmlformat import *
 from Mailman.Logging.Syslog import syslog
+from recaptcha.client import captcha
 
 SLASH = '/'
 ERRORSEP = '\n\n<p>'
@@ -165,6 +166,17 @@
             results.append(
     _('There was no hidden token in your submission or it was corrupted.'))
             results.append(_('You must GET the form before submitting it.'))
+
+    # recaptcha
+    captcha_response = captcha.v2submit(
+        cgidata.getvalue('g-recaptcha-response', ""),
+        mm_cfg.RECAPTCHA_PRIVATE_KEY,
+        remote,
+    )
+
+    if not captcha_response.is_valid:
+        results.append(_('Invalid captcha: %s' % captcha_response.error_code))
+
     # Was an attempt made to subscribe the list to itself?
     if email == mlist.GetListEmail():
         syslog('mischief', 'Attempt to self subscribe %s: %s', email, remote)
{{<  / highlight >}}
  
<p>
<strong>/usr/lib/mailman/templates/en/listinfo.html</strong>
</p>

  
{{< highlight html >}}
--- listinfo.html	2018-02-26 15:02:34.000000000 +0000
+++ /usr/lib/mailman/templates/en/listinfo.html	2018-02-26 14:18:52.000000000 +0000
@@ -3,7 +3,7 @@
 <HTML>
   <HEAD>
     <TITLE><MM-List-Name> Info Page</TITLE>
-  
+    <MM-Recaptcha-Script> 
   </HEAD>
   <BODY BGCOLOR="#ffffff">
 
@@ -116,6 +116,11 @@
       </tr>
       <mm-digest-question-end>
       <tr>
+      <tr>
+        <td>Please fill out the following captcha</td>
+        <td><mm-recaptcha-javascript></TD>
+      </tr>
+      <tr>
 	<td colspan="3">
 	  <center><MM-Subscribe-Button></center>
     </td>
{{< / highlight >}}
    
    
<p>
The updated RPMs are being rolled out to Fedora, EPEL 6 and EPEL 7. In the meantime you can find them <a href="https://fedorapeople.org/~averi/RPMs/python-recaptcha-client">here</a>.
</p>

[1]: https://www.dragonsreach.it/2014/05/03/adding-recaptcha-support-to-mailman/
[2]: https://pypi.python.org/pypi/recaptcha-client
