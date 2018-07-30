---
title: Back from GUADEC 2018
author: Andrea Veri
type: post
date: 2018-07-30T10:27:23+00:00
url: /2018/07/30/back-from-guadec-2018/
categories:
  - Planet GNOME
tags:
  - GUADEC
  - planet-gnome

---
Been a while since GUADEC 2018 has ended but subsequent travels and tasks reduced the time to write up a quick summary of what happened during this year's GNOME conference. The topics I'd like to emphasize mainly are:

 * We're hiring another Infrastructure Team member
 * We've successfully finalized the cgit to GitLab migration
 * Future plans including the migration to Openshift

## GNOME Foundation hirings

With the recent donation of 1M the Foundation has started recruiting on a variety of different new professional roles including a new Infrastructure team member. On this side I want to make sure that although the [job description](https://www.gnome.org/foundation/careers/devops-sysadmin/) mentions the word **sysadmin** the figure we're looking for is a systems engineer with a proven experience on Cloud computing platforms and tools such as AWS, Openshift and generally configuration management softwares such as Puppet and Ansible. Additionally this person should prove to have a clear understanding of the network and operating system (mainly RHEL) layers.

We've already identified a set of candidates and will be proceeding with interviews in the coming weeks. This doesn't mean we've hired anyone already, please keep sending CVs if interested and feeling the position would match your skills and expectations.

## cgit to GitLab

As announced on several occasions the GNOME Infrastructure has [successfully finalized the cgit to GitLab migration](https://mail.gnome.org/archives/desktop-devel-list/2018-May/msg00051.html). From a read-only view against .git directories to a fully compliant CI/CD infrastructure. The next step on this side will be deprecating Bugzilla which has already started with bugmasters turning products read-only in case they were migrated to the new platform or by identifying whether any of the not-yet-migrated products can be archived. The idea here is waiting to see zero activity on BZ in terms of new comments to existing bugs and no new bugs being submitted at all (we have redirects in place to make sure whenever enter_bug.cgi is triggered the request gets sent to the /issues/new endpoint for that specific GitLab project) and then turn the entire BZ instance into an HTML archive for posterity and to reduce the maintenance burden of keeping an instance up-to-date with upstream in terms of CVEs.

Thanks to all the involved parties including Carlos, Javier and GitLab itself given the prompt and detailed responses they always provided to our queries. Also thanks for sponsoring our AWS activities related to GitLab!

## Future plans 

With the service == VM equation we've been following for several years it's probably time for us to move to a more scalable infrastructure. The next generation platform we've picked up is going to be Openshift, its benefits:

 1. It's not important where a service runs behind scenes: it only has to run (VM vs pods and containers that are part of a pod that get scheduled randomly on the available Openshift nodes)
 2. Easily scalable in case additional resources are needed for a small period of time
 3. Built-in monitoring starting from Openshift 3.9 (the release we'll be running) based on Prometheus (+ Grafana for dashboarding)
 4. GNOME services as containers
 5. Individual application developers to schedule their own builds and see their code deployed with one click directly in production

The base set of VMs and bare metals has been already configured. Red Hat has been so great to provide the GNOME Project with a set of Red Hat Container Platform (and GlusterFS for heketi-based brick provisioning) subscriptions. We'll start moving over to the infrastructure in the coming weeks. It's going to take some good time but in the end we'll be able to free up a lot of resources and retire several VMs and related deprecated configurations.

## Misc

Slides from the Foundation AGM Infrastructure team report are available [here](https://www.dragonsreach.it/files/guadec-reports/guadec2018.html).

Many thanks to the GNOME Foundation for the sponsorship of my travel!

![GUADEC 2018 Badge](/img/2018-GUADEC-badge.png)
