---
title: A few useful Puppet snippets
author: Andrea Veri
type: post
date: 2012-01-31T18:43:26+00:00
url: /2012/01/31/a-few-useful-puppet-snippets/
aktt_notify_twitter:
  - no
tmac_last_id:
  - 276277235311312896
categories:
  - Puppet
  - Sysadmin
tags:
  - Puppet
  - snippets

---
As per Wikipedia: 

> Puppet is a tool for managing the configuration of Unix-like systems, declaratively. The developer provides puppet templates for describing parts of the system, and, when these templates are deployed, the runtime puts the managed systems into the declared state.
> 
> Puppet consists of a custom declarative language to describe system configuration, distributed using the client-server paradigm (using XML-RPC protocol), and a library to realize the configuration. The resource abstraction layer enables administrators to describe the configuration in high-level terms, such as users, services and packages.

I&#8217;ve been playing with the aforementioned tool lately both on my home network and within the Fedora&#8217;s Infrastructure team and I thought some of the work I did might be useful for anyone out there being stuck with a Puppet&#8217;s manifest or an ERB template. 

**Snippet #1**: Make sure the user **&#8216;foo&#8217;** is always created with its own home directory, password, shell, and full name.

{{< highlight Puppet >}}class users {
    users::add { "foo":
        username        => 'foo',
        comment         => 'Foo's Full Name',
        shell           => '/bin/bash',
        password_hash   => 'pwd_hash_as_you_can_see_in_/etc/shadow'
    }

define users::add($username, $comment, $shell, $password_hash) {
    user { $username:
        ensure => 'present',
        home   => "/home/${username}",
        comment => $comment,
        shell  => $shell,
        managehome => 'true',
        password => $password_hash,
    }
  }
}{{< / highlight >}}

**Snippet #2:** Make sure the user **&#8216;foo&#8217;** gets added into **/etc/sudoers**.

{{< highlight Puppet >}}class sudoers {

file { "/etc/sudoers":
      owner   => "root",
      group   => "root",
      mode    => "440",
     }
}

augeas { "addfootosudoers":
  context => "/files/etc/sudoers",
  changes => [
    "set spec[user = 'foo']/user foo",
    "set spec[user = 'foo']/host_group/host ALL",
    "set spec[user = 'foo']/host_group/command ALL",
    "set spec[user = 'foo']/host_group/command/runas_user ALL",
  ],
}{{< / highlight >}}

**Snippet #3:** Make sure that **openssh-server** is: installed, running on Port 222 and accepting **RSA** authentications only.

{{< highlight Puppet >}}class openssh-server {

  package { "openssh-server": 
      ensure => "installed",
  }

    service { "ssh":
        ensure    => running,
        hasstatus => true,
        require   => Package["openssh-server"],
    }

augeas { "sshd_config":
  context => "/files/etc/ssh/sshd_config",
    changes => [
    "set PermitRootLogin no",
    "set RSAAuthentication yes",
    "set PubkeyAuthentication yes",
    "set AuthorizedKeysFile	%h/.ssh/authorized_keys",
    "set PasswordAuthentication no",
    "set Port 222",
  ],
 }
}{{< / highlight >}}

**Snippet #4:** Don&#8217;t apply a specific **IPTABLES** rule if an host is tagged as &#8216;staging&#8217; in the relevant node file.

On **templates/iptables.erb**:

{{< highlight Puppet >}}# Allow unlimited traffic on eth0
-A INPUT -i eth0 -j ACCEPT
-A OUTPUT -o eth0 -j ACCEPT

# Allow unlimited traffic from trusted IP addresses
-A INPUT -s 192.168.1.1/24 -j ACCEPT

&lt;% if environment == "production" %>

-A INPUT -m state --state NEW -m tcp -p tcp --dport 22 -j ACCEPT
-A INPUT -m state --state NEW -m tcp -p tcp --dport 25 -j ACCEPT
-A INPUT -m state --state NEW -m tcp -p tcp --dport 80 -j ACCEPT

&lt;% unless defined?(staging).nil? %>
-A INPUT -s X.X.X.X -j REJECT --reject-with icmp-host-prohibited
&lt;% end -%>

&lt;% end -%>{{< / highlight >}}

On the **manifest** file:

{{< highlight Puppet >}}class iptables {
    package { iptables:
        ensure => installed;
    }

    service { "iptables":
        ensure    => running,
        hasstatus => true,
        require   => Package["iptables"],
    }

    file { "/etc/sysconfig/iptables":
        owner   => "root",
        group   => "root",
        mode    => 644,
        content => template("iptables/iptables.erb"),
        notify  => Service["iptables"],
    }
}{{< / highlight >}}

That&#8217;s all for now!
