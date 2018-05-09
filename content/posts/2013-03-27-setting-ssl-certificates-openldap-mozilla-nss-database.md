---
title: Setting up your SSL certificates on OpenLDAP by using a Mozilla NSS database
author: Andrea Veri
type: post
date: 2013-03-27T12:04:02+00:00
url: /2013/03/27/setting-ssl-certificates-openldap-mozilla-nss-database/
categories:
  - Linux
  - Sysadmin
tags:
  - Mozilla NSS
  - OpenLDAP
  - SSL
  - SSSD
  - TLS

---
I&#8217;ve recently spent some time setting up **TLS/SSL** encryption (**SSSD** won&#8217;t send a password in clear text when an user will try to authenticate against your LDAP server) on an **OpenLDAP** istance and as you may know the only way for doing that on a **RHEL / CentOS** environment is dealing with a **Mozilla NSS** database (which is, in fact, a **SQLite** database). I&#8217;ve been reading all the man pages of the relevant tools available to manipulate Mozilla NSS databases and I thought I would have shared the whole procedure and commands I used to achieve my goal. Even if you aren&#8217;t running an RPM based system you can opt to use a Mozilla NSS database to store your certificates as your preferred setup.

### On the LDAP (SLAPD) server

**Re-create *.db files**

{{< highlight Bash >}}mkdir /etc/openldap/certs
modutil -create -dbdir /etc/openldap/certs{{< / highlight >}}

**Setup a CA Certificate**

{{< highlight Bash >}}certutil -d /etc/openldap/certs -A -n "My CA Certificate" -t TCu,Cu,Tuw -a -i /etc/openldap/cacerts/ca.pem{{< / highlight >}}

where **ca.pem** should be your CA&#8217;s certificate file.

**Remove the password from the Database**

{{< highlight Bash >}}modutil -dbdir /etc/openldap/certs -changepw 'NSS Certificate DB'{{< / highlight >}}

**Creates the .p12 file and imports it on the Database**

{{< highlight Bash >}}openssl pkcs12 -inkey domain.org.key -in domain.org.crt -export -out domain.org.p12 -nodes -name 'LDAP-Certificate'

pk12util -i domain.org.p12 -d /etc/openldap/certs{{< / highlight >}}

where **domain.org.key **and **domain.org.crt **are the names of the certificates you previously created at your CA&#8217;s website.

**List all the certificates on the database and make sure all the informations are correct**

{{< highlight Bash >}}certutil -d /etc/openldap/certs -L{{< / highlight >}}

**Configure /etc/openldap/slapd.conf and make sure the TLSCACertificatePath points to your Mozilla NSS database**

{{< highlight Bash >}}TLSCACertificateFile /etc/openldap/cacerts/ca.pem
TLSCACertificatePath /etc/openldap/certs/
TLSCertificateFile LDAP-Certificate{{< / highlight >}}

### Additional commands

**Modify the trust flags if necessary**

{{< highlight Bash >}}certutil -d /etc/openldap/certs -M -n "My CA Certificate" -t "TCu,Cu,Tuw"{{< / highlight >}}

**Delete a certificate from the database**

{{< highlight Bash >}}certutil -d /etc/openldap/certs -D -n "My LDAP Certificate"{{< / highlight >}}

### On the clients (nslcd uses ldap.conf while sssd uses /etc/sssd/sssd.conf)

**On /etc/openldap/ldap.conf**

{{< highlight Bash >}}BASE dc=domain,dc=org
URI ldaps://ldap.domain.org

TLS_REQCERT demand
TLS_CACERT /etc/openldap/cacerts/ca.pem{{< / highlight >}}

**On /etc/sssd/sssd.conf**

{{< highlight Bash >}}ldap_tls_cacert = /etc/openldap/cacerts/ca.pem
ldap_tls_reqcert = demand
ldap_uri = ldaps://ldap.domain.org{{< / highlight >}}

### How to test the whole setup {#How_to_test_the_whole_setup}

{{< highlight Bash >}}ldapsearch -x -b 'dc=domain,dc=org' -D "cn=Manager,dc=domain,dc=org" '(objectclass=*)' -H ldaps://ldap.domain.org -W -v{{< / highlight >}}

**Troubleshooting**

If anything goes wrong you can run SLAPD with the following args for its debug mode:

{{< highlight Bash >}}/usr/sbin/slapd -d 256 -f /etc/openldap/slapd.conf -h "ldaps:/// ldap:///"{{< / highlight >}}

**Possible errors: **

If you happen to see an error similar to this one: &#8220;**TLS error -8049:Unrecognized Object Identifier.**&#8220;, try running ldapsearch with its debug mode this way:

{{< highlight Bash >}}ldapsearch -d 1 -x -ZZ -H ldap://ldap.domain.org{{< / highlight >}}

Make also sure that the **FQDN** you are trying to connect to is listed on the trusted FQDN&#8217;s list of your **domain.org.crt**.

**Update**: as SSSD&#8217;s developer **Stephen Gallagher** correctly pointed out on the comments using ldap\_tls\_reqcert = allow isn&#8217;t a best practice since it may take in <a href="http://en.wikipedia.org/wiki/Man-in-the-middle_attack" target="_blank">Man in the Midle Attacks</a>, adjusting the how to to match his suggestions.
