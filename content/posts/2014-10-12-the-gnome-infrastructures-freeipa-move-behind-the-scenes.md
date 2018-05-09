---
title: The GNOME Infrastructure’s FreeIPA move behind the scenes
author: Andrea Veri
type: post
date: 2014-10-12T18:02:02+00:00
url: /2014/10/12/the-gnome-infrastructures-freeipa-move-behind-the-scenes/
categories:
  - Planet Fedora
  - Planet GNOME
  - SSH
  - Sysadmin
tags:
  - foundation
  - freeipa
  - OpenLDAP
  - SSH
  - sysadmin

---
A few days ago I wrote about the <a href="https://www.dragonsreach.it/2014/10/07/the-gnome-infrastructure-is-now-powered-by-freeipa/" target="_blank">GNOME Infrastructure moving to FreeIPA</a>, the post was mainly an announcement to the relevant involved parties with many informative details for contributors to properly migrate their account details off from the old authentication system to the new one. Today&#8217;s post is a follow-up to that announcement but it&#8217;s going to take into account the reasons about our choice to migrate to **FreeIPA**, what we found interesting and compelling about the software and why we think more projects (them being either smaller or bigger) should migrate to it. Additionally I&#8217;ll provide some details about how I performed the migration from our previous **OpenLDAP** setup with a step-by-step guide that will hopefully help more people to migrate the infrastructure they manage themselves.

## The GNOME case

It&#8217;s very clear to everyone an infrastructure should reflect the needs of its user base, in the case of **GNOME** a multitude between developers, translators, documenters and between them a very good number of Foundation members, contributors that have proven their non-trivial contributions and have received the status of members of the GNOME Foundation with all the relevant benefits connected to it.

The situation we had before was very tricky, LDAP accounts were managed through our LDAP istance while Foundation members were being stored on a MySQL database with many of the tables being related to the yearly Board of Director&#8217;s elections and one specifically meant to store all the information from each of the members. One of the available fields on that table was defined as &#8216;userid&#8217; and was supposed to store the LDAP &#8216;uid&#8217; field the Membership Committee member processing a certain application had to update when accepting the application. This procedure had two issues:

  1. Membership Committee members had no access to LDAP information
  2. No checks were being run on the web UI to verify the &#8216;userid&#8217; field was populated correctly taking in multiple inconsistencies between LDAP and the MySQL database

In addition to the above Mango (the software that helped the GNOME administrative teams to manage the user data for multiple years had no maintainer, no commits on its core since 2008 and several limitations)

## What were we looking for as a replacement for the current setup?

It was very obvious to me we would have had to look around for possible replacements to Mango. What we were aiming for was a software with the following characteristics:

  1. It had to come with a **pre-built web UI** providing a wide view on several LDAP fields
  2. The web UI had to be **extensible in some form** as we had some custom LDAP schemas we wanted users to see and modify
  3. The sofware had to be **actively developed** and responsive to eventual security reports (given the high security impact a breach on LDAP could take in)

FreeIPA clearly matched all our expectations on all the above points.

## The Migration process &#8211; RFC2307 vs RFC2307bis

Our previous OpenLDAP setup was following [RFC 2307][1], which means that above all the other available LDAP attributes (listed on the RFC under point **2.2**) group&#8217;s membership was being represented through the &#8216;memberUid&#8217; attribute. An example:

{{< highlight Bash >}}
cn=foundation,cn=groups,cn=compat,dc=gnome,dc=org
objectClass: posixGroup
objectClass: top
gidNumber: 524
memberUid: foo
memberUid: bar
memberUid: foobar
{{< / highlight >}}

As you can each of the members of the group &#8216;foundation&#8217; are represented using the &#8216;memberUid&#8217; attribute followed by the &#8216;uid&#8217; of the user itself. FreeIPA does not make directly use of RFC2307 for its trees, but RFC2307bis instead. (RFC2307bis was not published as a RFC by the **IETF** as the author didn&#8217;t decide to pursue it nor the companies (HP, Sun) that then adopted it)

RFC2307bis uses a different attribute to represent group&#8217;s membership, it being &#8216;member&#8217;. Another example:

{{< highlight Bash >}}
cn=foundation,cn=groups,cn=accounts,dc=gnome,dc=org
objectClass: posixGroup
objectClass: top
gidNumber: 524
member: uid=foo,cn=users,cn=accounts,dc=gnome,dc=org
member: uid=bar,cn=users,cn=accounts,dc=gnome,dc=org
member: uid=foobar,cn=users,cn=accounts,dc=gnome,dc=org
{{< / highlight >}}

As you can see the **DN** representing the group &#8216;foundation&#8217; differs between the two examples. That is why FreeIPA comes with a **Compatibility plugin** (cn=compat) which automatically creates RFC2307-compliant trees and entries whenever an append / modify / delete operation happens on any of the hosted RFC2307bis-compliant trees. What&#8217;s the point of doing this when we could just stick with RFC2307bis trees and go with it? As the plugin name points out the Compatibility plugin is there to prevent breakages between the directory server and any of the clients or softwares out there still retrieving information and data by using the &#8216;memberUid&#8217; attribute as specified on RFC2307.

FreeIPA migration tools (ipa migrate-ds) do come with a &#8216;&#8211;schema&#8217; flag you can use to specify what attribute the istance you are migrating from was following (values are RFC2307 and RFC2307bis as you may have guessed already), in the case of GNOME the complete command we ran (after installing all the relevant tools through &#8216;ipa-server-install&#8217; and copying the custom schemas under /etc/dirsrv/slapd-$ISTANCE-NAME/schema) was:

{{< highlight Bash >}}ipa migrate-ds --bind-dn=cn=Manager,dc=gnome,dc=org --user-container=ou=people,dc=gnome,dc=org --group-container=ou=groups,dc=gnome,dc=org --group-objectclass=posixGroup ldap://internal-IP:389 --schema=RFC2307
{{< / highlight >}}

Please **note** that before running the command you should make sure custom schemas you had on the istance you are migrating from are available to the directory server you are migrating your tree to.

More information on the **migration process** from an existing OpenLDAP istance can be found <a href="http://docs.fedoraproject.org/en-US/Fedora/15/html/FreeIPA_Guide/Migrating_from_a_Directory_Server_to_IPA-Performing_a_Server_based_Migration.html" target="_blank">HERE</a>.

## The Migration process &#8211; Extending the directory server with custom schemas

One of the other challenges we had to face has been extending the available LDAP schemas to include Foundation membership attributes. This operation requires the following changes:

  1. Build the LDIF (that will include two new custom fields: FirstAdded and LastRenewedOn)
  2. Adding the LDIF in place on **/etc/dirsrv/slapd-$ISTANCE-NAME/schema**
  3. Extend the web UI to include the new attributes

I won&#8217;t be explaining how to build a LDIF on this post but I&#8217;m however pasting the schema I made to help you getting an idea:

{{< highlight Bash >}}attributeTypes: ( 1.3.6.1.4.1.3319.8.2 NAME 'LastRenewedOn' SINGLE-VALUE EQUALITY caseIgnoreMatch SUBSTR caseIgnoreSubstringsMatch DESC 'Last renewed on date' SYNTAX 1.3.6.1.4.1.1466.115.121.1.15 )
attributeTypes: ( 1.3.6.1.4.1.3319.8.3 NAME 'FirstAdded' SINGLE-VALUE EQUALITY caseIgnoreMatch SUBSTR caseIgnoreSubstringsMatch DESC 'First added date' SYNTAX 1.3.6.1.4.1.1466.115.121.1.15 )
objectClasses: ( 1.3.6.1.4.1.3319.8.1 NAME 'FoundationFields' AUXILIARY MAY ( LastRenewedOn $ FirstAdded ) )
{{< / highlight >}}

After copying the schema in place and restarting the directory server, extend the web UI:

on **/usr/lib/python2.7/site-packages/ipalib/plugins/foundation.py**:

{{< highlight Python >}}from ipalib.plugins import user
from ipalib.parameters import Str
from ipalib import _
from time import strftime
import re

def validate_date(ugettext, value):
if not re.match("^[0-9]{4}-(0[1-9]|1[0-2])-(0[1-9]|[1-2][0-9]|3[0-1])$", value):
return _("The entered date is wrong, please make sure it matches the YYYY-MM-DD syntax")

user.user.takes_params = user.user.takes_params + (
Str('firstadded?', validate_date,
cli_name='firstadded',
label=_('First Added date'),
),
Str('lastrenewedon?', validate_date,
cli_name='lastrenewedon',
label=_('Last Renewed on date'),
),
)
{{< / highlight >}}

on **/usr/share/ipa/ui/js/plugins/foundation/foundation.js**:

{{< highlight JavaScript >}}define([
'freeipa/phases',
'freeipa/user'],
function(phases, user_mod) {

// helper function
function get_item(array, attr, value) {
for (var i=0,l=array.length; i&lt;l; i++) {
if (array[i][attr] === value) return array[i];
}

return null;
}

var foundation_plugin = {};

foundation_plugin.add_foundation_fields = function() {

var facet = get_item(user_mod.entity_spec.facets, '$type', 'details');
var section = get_item(facet.sections, 'name', 'identity');
section.fields.push({
name: 'firstadded',
label: 'Foundation Member since'
});

section.fields.push({
name: 'lastrenewedon',
label: 'Last Renewed on date'
});

section.fields.push({
name: 'description',
label: 'Previous account changes'
});

return true;

};

phases.on('customization', foundation_plugin.add_foundation_fields);
return foundation_plugin;
});
{{< / highlight >}}

Once done, restart the web server. The next step would be migrating all the **FirstAdded** and **LastRenewedOn** attributes off from MySQL into LDAP now that our custom schema has been injected.

The relevant MySQL fields were following the YYYY-MM-DD syntax to store the dates and a little Python script to read from MySQL and populate the LDAP attributes was then made. If interested or you are in a similar situation you can find it <a href="https://git.gnome.org/browse/sysadmin-bin/tree/membership/migrate-foundation-field-to-freeipa.py" target="_blank">HERE</a>.

## The Migration process &#8211; Own SSL certificates for HTTPD

As you may be aware of FreeIPA comes with its own certificate tools (powered by **Certmonger**), that means a **CA** is created (during the ipa-server-install run) and certificates for the various services you provide are then created and signed with it. This is definitely great and removes the burden to maintain an underlying self-hosted PKI infrastructure. At the same time this seems to be a problem for publicly-facing web services as browsers will start complaining they don&#8217;t trust the CA that signed the certificate the website you are trying to reach is using.

The problem is not really a problem as you can specify what certificate HTTPD should be using for displaying FreeIPA&#8217;s web UI. The procedure is simple and involves the NSS database at /etc/httpd/alias:

{{< highlight Bash >}}certutil -d /etc/httpd/alias/ -A -n "StartSSL CA" -t CT,C, -a -i sub.class2.server.ca.pem
certutil -d /etc/pki/nssdb -A -n "StartSSL CA" -t CT,C, -a -i sub.class2.server.ca.pem
openssl pkcs12 -inkey freeipa.example.org.key -in freeipa.example.org.crt -export -out freeipa.example.org.p12 -nodes -name 'HTTPD-Server-Certificate'
pk12util -i freeipa.example.org.p12 -d /etc/httpd/alias/
{{< / highlight >}}

Once done, update /etc/httpd/conf.d/nss.conf with the correct NSSNickname value. (which should match the one you entered after &#8216;-name&#8217; on the third of the above commands)

## The Migration process &#8211; Equivalent of authorized_keys&#8217; &#8220;command&#8221;

At GNOME we do run several services that require users to login to specific machines and run a command. At the same time and for security purposes we don&#8217;t want all the users to reach a shell. Originally we were making use of SSH&#8217;s authorized_keys file to specify the &#8220;command&#8221; these users should have been restricted to. FreeIPA handles Public Key authentications differently (through the **sss\_ssh\_authorizedkeys** binary) which means we had to find an alternative way to restrict groups to only a specific command. SSH&#8217;s **ForceCommand** came in help, an example given a group called &#8216;foo&#8217;:

{{< highlight Bash >}}
Match Group foo,!bar
X11Forwarding no
PermitTunnel no
ForceCommand /home/admin/bin/reset-my-password.py
{{< / highlight >}}

The above **Match Group** will be applied to all the users of the &#8216;foo&#8217; group except the ones that are also part of the &#8216;bar&#8217; group. If you are interested in the reset-my-password.py script which resets a certain user password and sends a temporary one to the registered email address (by checking the mail LDAP attr) for the user, click <a href="https://git.gnome.org/browse/sysadmin-bin/tree/reset-my-password.py" target="_blank">HERE</a>.

## The Migration process &#8211; Kerberos host Keytabs

Here, at **GNOME**, we still have one or two RHEL 5 hosts hanging around and SSSD reported a failure when trying to authenticate with the given Keytab (generated with RHEL 7 default values) to the KDC running (as you may have guessed) RHEL 7. The issue is simple as RHEL 5 does not support many of the encryption types which the Keytab was being encrypted with. Apparently the only currently supported Keytab encryption type on a RHEL 5 machine is **rc4-hmac**. Creating a Keytab on the KDC accordingly can be done this way:

{{< highlight Bash >}}ipa-getkeytab -s server.example.org -p host/client.example.org -e rc4-hmac -k /root/keytabs/client.example.org.keytab
{{< / highlight >}}

That should be all for today, I&#8217;ll make sure to update this post with further details or answers to possible comments.

 [1]: https://www.ietf.org/rfc/rfc2307.txt
