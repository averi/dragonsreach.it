---
title: The GNOME Infrastructure is moving to Openshift 
author: Andrea Veri
type: post
date: 2018-10-18T10:27:23+00:00
url: /2018/10/18/2018-10-18-gnome-infrastructure-moving-to-openshift/
categories:
  - Planet GNOME
tags:
  - planet-gnome
  - sysadmin
  - openshift

---
During GUADEC 2018 we [announced](https://www.dragonsreach.it/2018/07/30/back-from-guadec-2018) one of the core plans of this and the coming year: it being moving as many GNOME web applications as possible to the GNOME Openshift instance we architected, deployed and configured back in July. Moving to Openshift will allow us to:

 1. Save up on resources as we're deprecating and decommissioning VMs only running a single service
 2. Allow app maintainers to use the most recent Python, Ruby, preferred framework or programming language release without being tied to the release RHEL ships with
 3. Additional layer of security: containers
 4. Allow app owners to modify and publish content without requiring external help
 5. Increased apps redundancy, scalability, availability
 6. Direct integration with any VCS that ships with webhooks support as we can trigger the Openshift provided endpoint whenever a commit has occurred to generate a new build / deployment

## Architecture

The cluster consists of 3 master nodes (controllers, api, etcd), 4 compute nodes and 2 infrastructure nodes (internal docker registry, cluster console, haproxy-based routers, SSL edge termination). For the persistent storage we're currently making good use of the Red Hat Gluster Storage (RHGS) product that Red Hat is kindly sponsoring together with the Openshift subscriptions. For any app that might require a database we have an external (as not managed as part of Openshift) fully redundant, synchronous, multi-master MariaDB cluster based on Galera (2 data nodes, 1 arbiter).

The release we're currently running is the [recently released](https://docs.openshift.com/container-platform/3.11/release_notes/ocp_3_11_release_notes.html) 3.11, which comes with the so-called "Cluster Console", a web UI that allows you to manage a wide set of the underlying objects that previously were only available to the oc cli client and with a set of Monitoring and Metrics toolings (Prometheus, Grafana) that can be accessed as part of the Cluster Console (Grafana dashboards that show how the cluster is behaving) or externally via their own route.

## SSL Termination

The SSL termination is currently happening on the edge routers via a wildcard certificate for the gnome.org and guadec.org zones. The process of renewing these certificates is automated via Puppet as we're using Let's Encrypt behind the scenes (domain verification for the wildcard certs happen at the DNS level, we built [specific hooks](https://gitlab.gnome.org/Infrastructure/sysadmin-bin/tree/master/letsencrypt) in order to make that happen via the [getssl](https://github.com/srvrco/getssl) tool). The backend connections are following two different paths:

 1. edge termination with no re-encryption in case of pods containing static files (no logins, no personal information ever entered by users): in this case the traffic is encrypted between the client and the edge routers, plain text between the routers and the pods (as they're running on the same local broadcast domain)
 2. re-encrypt for any service that requires authentication or personal information to be entered for authorization: in this case the traffic is encrypted from end to end

## App migrations

App migrations have started already, we've successfully migrated and deprecated a set of GUADEC-related web applications, specifically:

 1. $year.guadec.org where $year spaces from 2013 to 2019
 2. wordpress.guadec.org has been deprecated

We're currently working on migrating the GNOME Paste website making sure we also replace the current unmaintained software to a [supported one](https://github.com/LINKIWI/modern-paste). Next on the list will be the Wordpress-based websites such as www.gnome.org and blogs.gnome.org (Wordpress Network). I'd like to thank the GNOME Websites Team and specifically **Tom Tryfonidis** for taking the time to migrate existing assets to the new platform as part of the GNOME websites refresh program.
