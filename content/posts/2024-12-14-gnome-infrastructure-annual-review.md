---
title: "2024 GNOME Infrastructure Annual Review"
date: 2024-12-13T16:29:20-05:00
type: post
url: /2024/12/14/gnome-infrastructure-annual-review/
categories:
  - Planet GNOME
tags:
  - planet-gnome
  - sysadmin
  - openshift

---
## Table of Contents
- [Table of Contents](#table-of-contents)
- [1. Introduction](#1-introduction)
- [2. Achievements](#2-achievements)
  - [2.1. Major achievements](#21-major-achievements)
  - [2.2. Minor achievements](#22-minor-achievements)
  - [2.3 Minor annoyances/bugs that were also fixed in 2024](#23-minor-annoyancesbugs-that-were-also-fixed-in-2024)
  - [2.3. Our brand new and renewed partnerships](#23-our-brand-new-and-renewed-partnerships)
  - [Expressing my gratitude](#expressing-my-gratitude)

## 1. Introduction

Time is passing by very quickly and another year will go as we approach the end of 2024. This year has been fundamental in shaping the present and the future of GNOME's Infrastructure with its major highlight being a completely revamped platform and a migration of all GNOME services over to AWS. In this post I'll try to highlight what the major achievements have been throughout the past 12 months.

## 2. Achievements

In the below is a list of individual tasks and projects we were able to fulfill in 2024. This section will be particularly long but I want to stress the importance of each of these items and the efforts we put in to make sure they were delivered in a timely manner.

### 2.1. Major achievements

1. All the applications (except for ego, which we expect to handle as soon as next week or in January) were migrated to our new AWS platform (see [GNOME Infrastructure migration to AWS](https://www.dragonsreach.it/2024/11/16/gnome-infrastructure-migration-to-aws/))
2. During each of the apps migrations we made sure to:
  2a. Migrate to sso.gnome.org and make 2FA mandatory
  2b. Make sure database connections are handled via connection poolers
  2c. Double check the container images in use were up-to-date and GitLab CI/CD pipeline schedules were turned on for weekly rebuilds (security updates)
  2d. For GitLab, we made sure repositories were migrated to an EBS volume to increase IO throughput and bandwidth
3. Migrated away our backup mechanism away from rdiff-backup into AWS Backup service (which handles both our AWS EFS and EBS snapshots)
4. Retired our NSD install and migrated our authoritative name servers to CloudNS (it comes with multiple redundant authoritative servers, DDOS protection and automated DNSSEC keys rotation and management)
5. We moved away from Ceph and the need to maintain our own storage solution and started leveraging AWS EFS and EBS
6. We deprecated Splunk and built a solution around promtail and Loki in order to handle our logging requirements
7. We deprecated Prometheus blackbox and started leveraging CloudNS monitoring service which we interact with using an API and a set of CI/CD jobs we host in GitHub
8. We archived GNOME's wiki and turned it into a static HTML copy
9. We replaced ftpadmin with the GNOME Release Services, thanks speknik! More information around what steps should GNOME Maintainers now follow when doing a module release are available [here](https://handbook.gnome.org/maintainers/making-a-release.html). The service uses JWT tokens to verify and authorize specific CI/CD jobs and only allows new releases when the process is initiated by a project CI living within the GNOME GitLab namespace and a protected tag. With master.gnome.org and ftpadmin being in production for literally ages, we wanted to find a better mechanism to release GNOME software and avoid a single maintainer SSH key leak to allow a possible attacker to tamper tarballs and potentially compromise milions of computers running GNOME around the globe. With this change we don't leverage SSH anymore and most importantly we don't allow maintainers to generate GNOME modules tarballs on their personal computers rather we force them to use CI/CD in order to achieve the same result. We'll be coming up shortly with a dedicated and isolated runner that will only build jobs tagged as releasing GNOME software.
10. We retired our mirroring infrastructure based on Mirrorbits and replaced it with our CDN partner, CDN77
11. We decoupled GIMP mirroring service from GNOME's one, GIMP now hosts its tarballs (and associated rsync daemon) on top of a different master node, thanks OSUOSL for sponsoring the VM that makes this possible!

### 2.2. Minor achievements

1. Retired multiple VMs: splunk, nsd0{1,2}, master, ceph-metrics, gitaly
2. We started managing our DNS using an API and CI/CD jobs hosted in GitHub (this to avoid relying on GNOME's GitLab which in case of unavailability would prevent us to update DNS entries)
3. We migrated smtp.gnome.org to OSCI in order to not lose IP reputations and various whitelists we received throughout the years by multiple organizations
4. We deprecated our former internal DNS authoritatives based on FreeIPA. We are now leveraging internal VPC resolvers and Route53 Private zones
5. We deprecated all our OSUOSL GitLab runners due to particularly slow IO and high steal time and replaced them with a new Heztner EX44 instance, kindly sponsored by GIMP. OSUOSL is working on coming up with local storage on their Openstack platform. We are looking forward to test that and introduce new runners as soon as the solution will be made available
6. Retired idm0{1,2} and redirected them to a new FreeIPA load balanced service at https://idm.gnome.org
7. We retired services which weren't relevant for the community anymore: surveys.gnome.org, roundcube (aka webmail.gnome.org)
8. We migrated nmcheck.gnome.org to Fastly and are using Synthetic responses to handle HTTP responses to clients
9. We upgraded to Ansible Automation Platform (AAP) 2.5
10. As part of the migration to our new AWS based platform, we upgrade Openshift to release 4.17
11. We received a 2k grant from Microsoft which we are using for an Azure ARM64 GitLab runner
12. All of our GitLab runners fleet are now hourly kept in sync using AAP (Ansible roles were built to make this happen)
13. We upgraded Cachet to 3.x series and fixed dynamic status.gnome.org updates (via a customized version of cachet-monitor)
14. OS Currency: we upgraded all our systems to RHEL 9
15. We converted all our Openshift images that were using a web server to Nginx for consistency/simplicity
16. Replaced NRPE with Prometheus metrics based logging, checks such as IDM replication and status are now handled via the Node Exporter textfile plugin
17. Migrated download.qemu.org (yes, we also host some components of QEMU's Infrastructure) to use nginx-s3-gateway, downloads are then served via CDN77
  
### 2.3 Minor annoyances/bugs that were also fixed in 2024

1. Invalid OCSP responses from CDN77, https://gitlab.gnome.org/Infrastructure/Infrastructure/-/issues/1511
2. With the migration to [USE_TINI](https://gitlab.com/gitlab-org/build/CNG/-/issues/1853) for GitLab, no gpg zombie processes are being generated anymore

### 2.3. Our brand new and renewed partnerships

1. From November 2024 and ongoing, AWS will provide sponsorship and funding to the GNOME Project to sustain the majority of its infrastructure needs
2. Red Hat kindly sponsored subscriptions for RHEL, Openshift, AAP as well as hosting, bandwidth for the GNOME Infrastructure throughout 2024
3. CDN77 provided unlimited bandwidth / traffic on their CDN offering
4. Fastly renewed their unlimited bandwidth / traffic plan on their Delivery/Compute offerings
5. and thanks to OSUOSL, Packet, DigitalOcean, Microsoft for the continued hosting and sponsorship of a set of GitLab runners, virtual machines and ARM builders!

### Expressing my gratitude

As I'm used to do at the end of each calendar year, I want to express my gratitude to Bartłomiej Piotrowski for our continued cooperation and also to Stefan Peknik for his continued efforts in developing the GNOME Release Service. We started this journey together many months ago when Stefan was trying to find a topic to base his CS bachelor thesis on. With this in mind I went straight into the argument of replacing ftpadmin with a better technology also in light of what happened with the xz case. Stefan put all his enthusiasm and professionality into making this happen and with the service going into production on the 11th of December 2024 history was made.

That being said, we're closing this year being extremely close to retire our presence from RAL3 which we expect to happen in January 2025. The GNOME Infrastructure will also send in a proposal to talk at GUADEC 2025, in Italy, to present and discuss all these changes with the community.
