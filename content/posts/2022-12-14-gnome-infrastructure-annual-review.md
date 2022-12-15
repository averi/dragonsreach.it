---
title: "2022 GNOME Infrastructure Annual Review"
date: 2022-12-12T17:53:26+01:00
type: post
url: /2022/12/14/gnome-infrastructure-annual-review/
categories:
  - Planet GNOME
tags:
  - planet-gnome
  - sysadmin
  - openshift

---
- [1. Introduction](#1-introduction)
- [2. Achievements](#2-achievements)
  - [2.1. Major achievements](#21-major-achievements)
  - [2.2. Minor achievements](#22-minor-achievements)
  - [2.3. Our brand new and renewed partnerships](#23-our-brand-new-and-renewed-partnerships)
- [3. Highlights](#3-highlights)
  - [3.1. Openshift 4: architecture](#31-openshift-4-architecture)
  - [3.2. Openshift 4: virtualization networking](#32-openshift-4-virtualization-networking)
  - [3.3. Openshift 4: image builds](#33-openshift-4-image-builds)
  - [3.4. Openshift 4: cluster backups](#34-openshift-4-cluster-backups)
  - [3.5. GitLab on Openshift 4: setup](#35-gitlab-on-openshift-4-setup)
  - [3.6. GitLab on Openshift 4: early days](#36-gitlab-on-openshift-4-early-days)
  - [3.7. GitLab on Openshift 4: logging](#37-gitlab-on-openshift-4-logging)
- [Future plans](#future-plans)
- [Expressing my gratitude](#expressing-my-gratitude)

## 1. Introduction

I believe it's kind of vital for the GNOME Infrastructure Team to outline not only the amazing work that was put into place throughout the year, but also the challenges we faced including some of the architectural designs we implemented over the past 12 months. This year has been extremely challenging for multiple reasons, the top one being Openshift 3 (which we deployed in 2018) going EOL in June 2022. We also wanted to make sure we were keeping up with OS currency, specifically finalizing the migration of all our VM-based workloads to RHEL 8 and most importantly to Ansible. The main challenges there being adapting our workflow away from the Source-To-Image (s2i) mechanism into building our own infrastructure images directly through GitLab CI/CD pipelines by ideally also dropping the requirement of hosting an internal containers registry. 

With the community also deciding to move away from Mailman, we also had an hard deadline to finalize the migration to Discourse that was started back in 2018. At the same time the GNOME community was also looking towards a better Matrix to IRC integration while irc.gimp.org (GIMPNet) showing aging sypmptoms and being put into very low maintenance mode due to a lack of IRC operators/admins time.

## 2. Achievements

A list of each of the individual tasks and projects we were able to fulfill during 2022. This particular section will be particularly long but I want to stress the importance of each of these items and the efforts we put in to make sure they were delivered in a timely manner. A subset of these tasks will also receive further explanation in the sections to come. 

### 2.1. Major achievements

1. Architected, deployed, operated an Openshift 4 cluster that replaced our former OCP 3 installation. This was a major undertaking that required a lot of planning, testing and coordination with the community. We also had to make sure we were able to migrate all our existing tenants from OCP 3 to OCP 4. A total of 46 tenants were migrated and/or created during these past 12 months.
2. Developed a brand new workflow moving away from Source-To-Image (s2i) and towards GitLab CI/CD pipelines.
3. Migrated from individual NSD based internal resolvers to FreeIPA's self managed BIND resolvers, this gave us the possibility to use APIs and other IPA's goodies to manage our internal DNS views.
4. For existing virtual machines we wanted to keep around, we leveraged the Openshift Virtualization operator which allows you to benefit from kubevirt's features and effectively run virtual machines from within an OCP cluster. That also includes support for VM templates and automatic bootstraping of VMs out of existing PVCs and/or externally hosted ISO files.
5. We [developed and deployed automation](https://gitlab.gnome.org/Infrastructure/openshift-images/accounts-automation-app) for handling Membership and Accounts related requests. The [documentation](https://wiki.gnome.org/Infrastructure/NewAccounts) has also been updated accordingly.
6. GitLab was migrated from a monolithic virtual machine to Openshift.
7. We introduced DKIM/SPF support for GNOME Foundation members, please see my [announcement](https://discourse.gnome.org/t/announcement-upcoming-changes-to-gnomes-mail-infrastructure/11568) for a list of changes.
8. We rebuilt all the VMs that were not retired due to the migration to OCP to RHEL 8
9. We successfully migrated away from our Puppet infrastructure to Ansible (and Ansible collections). This particular task is a major milestone, Puppet has been around the GNOME Infrastructure for more than 15 years.

### 2.2. Minor achievements

1. Identified root cause and blocked a brute force attempt (from 600+ unique IP addresses) against our LDAP database directory. Some of you surely remind the times where you found your GNOME Account locked and you were unsure around why that was happening. This was the exact reason why. A particular remediation was applied temporarily, that also had the side effect of blocking HTTPs based git clones/pushes. That was resolved by moving GitLab to OpenID (via Keycloak) and using token based authentication.
2. We moved static.gnome.org to S3 and put our CDN in front of it.
3. Re-deployed bastion01, nsd01, nsd02, smtp, master, logs (rsyslog) using Ansible (this also includes building Ansible roles and replicating what we had in Puppet to Ansible)
4. Deployed Minio S3 GW (cache) to avoid incurring in extra AWS S3 costs
5. Deprecated OpenVPN in favor of Wireguard
6. We deprecated GlusterFS entirely and completed our migration to Ceph RBD + CephFS
7. We retired several VM based workloads that were either migrated to Openshift or superseded including reverse proxies, Puppet masters, GitLab, the entirety of OCP 3 virtual machines (with OCP 4 being installed on bare metals directly)
8. Configured blackbox Prometheus exporter and moved services availability checks to it
9. We retired people.gnome.org, barely used by anyone due to the multitude of alternatives we currently provide when it comes to host GNOME related files including GitLab pages, static.gnome.org, GitLab releases, Nextcloud.
10. Started ingesting Prometheus metrics into our existing Prometheus cluster via federation, a wide set of dashboards were also created to keep track of the status of OCP, Ceph, general OS related metrics and databases.
11. We migrated our databases to OCP: Percona MySQL operator, Crunchy PostgreSQL operator
12. Rotated KSK and ZSK DNSSEC keys on gnome.org, gtk.org, gimp.{org,net} domains
13. We migrated from obtaining Let's Encrypt certificates using getssl to OCP CertManager operator. For edge routers, we migrated to certbot and deployed specific hooks to automate the handling of DNS-01 challenges.
14. We migrated GIMP downloads from a plain httpd setup to use mirrorbits to match what the GNOME Project is operating.
15. We deployed AAP (Red Hat Ansible Automation Platform) in order to be able to recreate hourly configuration management runs as we had before with Puppet. These runs are particularly crucial as they make sure the latest content from our Ansible repository is pulled and enforced across all the systems Ansible manages.
16. [irc.gnome.org migration to Libera.Chat](https://discourse.gnome.org/t/gnome-moves-away-from-gimpnet-on-nov-25-15-00-utc/12046), thanks Thibault Martin and Element for the amazing continued efforts supporting GNOME's Matrix to IRC bridge integration!
17. Migrated away from Mailman to Discourse. This particular item has been part of community discussions since 2018, after evaluation by the community itself and the GNOME Project governance the migration to Discourse started and was finalized this year, please read here for a list of [FAQs](https://discourse.gnome.org/t/common-questions-re-mailman-to-discourse/11841).
18. We introduced OpenID authentication (via Keycloak) to help resolve the fragmentation multiple different authentication backends were causing.
19. We introduced [Hedgedoc](https://hedgedoc.gnome.org/), an Etherpad replacement.
20. We enhanced our Splunk cluster with additional dashboards, log based alerts, new sourcetypes
21. We deprecated MeetBot (unused since several years) and CommitsBot, which we replaced with a beta Matrix bot called [Hookshot](https://matrix-org.github.io/matrix-hookshot/latest/), which leverages GitLab webhooks in order to send notifications to Matrix rooms
22. We upgraded FreeIPA to version 4.9.10, and on RHEL 8. We enhanced IPA backups to include hourly file system snapshots (on top of the existing rdiff-backup runs) and daily ipa-backup runs.
23. [We presented at GUADEC 2022](https://www.dragonsreach.it/files/guadec-reports/guadec-2022.html).

### 2.3. Our brand new and renewed partnerships

1. Red Hat kindly sponsored subscriptions for RHEL, Ceph, Openshift, AAP
2. Splunk doubled the sponsorship to a total of 10GB/day traffic
3. AWS confirmed their partnership with a total of 5k USD credit
4. CDN77 provided unlimited bandwidth / traffic on their CDN offering
5. We're extremely close to finalize our partnership with Fastly! They'll be providing us with their Traffic Load Balancing product
6. and thanks to OSUOSL, Packet, DigitalOcean for the continued hosting and sponsorship of a set of GitLab runners, virtual machines and ARM builders!

## 3. Highlights

Without going too much deep into technical details I wanted to provide an overview of how we architected and deployed our Openshift 4 cluster and GitLab as these questions pop up pretty frequently among contributors.

### 3.1. Openshift 4: architecture

The cluster is currently setup with a total of 3 master nodes having similar specs (256G of RAM, 62 Cores, 2x10G NICs, 2x1G NICs) and acting in a hyperconverged setup. That implies we're also hosting a Ceph cluster (in addition to the existing one we setup a while back) we deployed via the Red Hat Openshift Data Foundations operator on the same nodes. OCP (release 4.10), in this scenario, is deployed directly on bare metal with an ingress endpoint per individual node. The current limitation to this particular architecture is there's no proper load balancing (other than DNS Round Robin) in front of these ingresses due to the fact external load balancers are particularly expensive. As I've mentioned earlier we're really close to finalize a partnership with Fastly to help fill the gap here. These nodes receive their configuration using Ignition, we made sure a specific set of MachineConfigs is there to properly configure these systems once they boot due to the way CoreOS works in this regard. At boot time, it fetches an Ignition definition from the Machine Config Operator controller and applies it to the target node.

### 3.2. Openshift 4: virtualization networking

As I previously mentioned we've been leveraging OCP CNV to support our VM based workloads. I wanted to quickly highlight how we handled the configuration of our internal and public networks in order for these VMs to successfully consume these subnets and be able to communicate back and forth with other data center resources and services:

1. A set of bonded interfaces was setup for both the 10G and the 1G NICs
2. A bridge was configured on top of these bonded interfaces, that is required by Openshift Multus to effectively append the VM interfaces to each of these bridges depending what kind of subnet(s) they're required to access
3. We configured OCP Multus (bridge mode) and its dependant NetworkAttachmentDefinition
4. From within an OCP CNV CRD, pass in:

{{< highlight yaml >}}
      networks:
        - multus:
            networkName: internal-network
          name: nic-1
{{< /highlight >}}

And a sample of the internal-network NetworkAttachmentDefinition:

{{< highlight yaml >}}
apiVersion: k8s.cni.cncf.io/v1
kind: NetworkAttachmentDefinition
metadata:
  name: internal-network
  namespace: infrastructure
spec:
  config: >-
    {"name":"internal-network","cniVersion":"0.3.1","plugins":[{"type":"cnv-bridge","bridge":"br0-internal","mtu":9000,"ipam":{}},{"type":"cnv-tuning"}]}
{{< /highlight >}}

### 3.3. Openshift 4: image builds

One of the major changes we implemented with the migration to OCP 4 was the way we built infrastructure related container images. In early days we were leveraging the s2i OCP feature which allowed building images out of a git repository, those builds were directly happening from within OCP worker nodes and pushed to the internal OCP registry. With the new setup what happens instead is:

1. We create a new git repository containing an application and an associated Dockerfile
2. From within that repository, we define a .gitlab-ci.yml file that inherits the build templates from a [common set of templates ](https://gitlab.gnome.org/Infrastructure/openshift-images/ci-templates) we created
3. The image is then built using GitLab CI/CD and pushed to quay.io
4. On the target OCP tenant, we define an ImageStream and point it to the quay.io registry namespace/image combination
5. From there the Deployment/DeploymentConfig resource is updated to re-use the previously created ImageStream, whenever the ImageStream changes, the deployment/deploymentconfig is triggered (via ImageChange triggers)

### 3.4. Openshift 4: cluster backups

When it comes to cluster backups we decided to take the following approach:

1. [Run daily etcd backups](https://gitlab.gnome.org/Infrastructure/openshift-images/openshift-etcd-backup)
2. Backup and dump all the tenants CRDs as json files to an encrypted S3 bucket using Velero

### 3.5. GitLab on Openshift 4: setup

Moving away from hosting GitLab on a monolithic virtual machine was surely one of our top goals for 2022. The reason was particularly simple, anytime we needed to perform a maintenance we were required to cause a service downtime, even during a plain minor platform upgrade. On top of that, we couldn't easily scale the cluster in case of sudden peeks in traffic, but generally when we originally designed our GitLab offering back in 2018 we missed a lot of the goodies OCP provides, the installation has worked well during all these years but the increasing usage of the service, the multitude of new GitLab sub-components made us rethink the way we had to design this particular offering to the community.

These are the main reasons why we migrated GitLab to OCP using the GitLab OCP Operator. Using operator's built-in declarative resources functionalities we could easily replicate our entire cluster config in a single yaml file, the operator at that point picked up each of our definitions and generated individual configmaps, deployments, scaleapps, services, routes and associated CRDs automatically. The only component we decided to not host via OCP directly but use a plain VM on OCP Virt was gitaly. The reason is particularly simple: gitaly requires port 22 to be accessible from outside of the cluster, that is currently not possible with the default haproxy based OCP ingress. We analyzed whether it made sense to deploy the NGINX ingress which also supports reverse proxying non-HTTP ports, we thought that'd have added additional complexity with no particular benefit. MetalL was also a possibility but the product itself is still a WIP, required sending out gARPs on the public network block we 
share with other community tenants for L2, for L3 there was a need to setup BGP peering between each of the individual nodes (speakers using MetalLB terminology) and an adiacent router, overkill for a single VIP.

### 3.6. GitLab on Openshift 4: early days

Right after the migration we started observing some instability with specific pods (webservice, sidekiq) backtracing after a few hours they were running, specifically:

{{< highlight shell >}}
Nov 16 05:08:03 master2.openshift4.gnome.org kernel: cgroup: fork rejected by pids controller in /kubepods.slice/kubepods-burstable.slice/kubepods-burstable-podc69234f3_8596_477c_b7ea_5b51f6d86cce.slice/crio-d36391a108570d6daecf316d6d19ffc6650a3fa3a82ee616944b9e51266c901f.scope
{{< /highlight >}}

also on kubepods.slice:

{{< highlight shell >}}
[core@master2 ~]$ cat /sys/fs/cgroup/pids/kubepods.slice/pids.max 
4194304
{{< /highlight >}}

It was clear the target pods were spawning a major set of new processes that were remaining around for the entire pod lifetime:

{{< highlight shell >}}
$ ps aux | grep gpg | wc -l
773
{{< /highlight >}}

And a sample out of the previous 'ps aux' run:

{{< highlight shell >}}
git        19726  0.0  0.0      0     0 ?        Z    09:58   0:00 [gpg] <defunct>
git        19728  0.0  0.0      0     0 ?        Z    09:58   0:00 [gpg] <defunct>
git        19869  0.0  0.0      0     0 ?        Z    10:06   0:00 [gpg] <defunct>
git        19871  0.0  0.0      0     0 ?        Z    10:06   0:00 [gpg] <defunct>
{{< /highlight >}}

It appears this specific bug was troubleshooted [already](https://gitlab.com/gitlab-com/gl-infra/reliability/-/issues/3989) by the GitLab Infrastructure Team around 4 years ago already. This misbehaviour is related to the intimate nature of GnuPG which requires calling its binaries (gpgconf, gpg, gpgsm, gpg-agent) for every required operation GitLab (webservice or sidekiq) asks it to perform. For some reason these processes never notified their parent process (PID 1 on that particular container) with a SIGCHLD and remained hanging around on the pods until pod's dismissal. We're in touch with the GitLab Open Source program support to understand next steps in order to have a fix implemented upstream.

### 3.7. GitLab on Openshift 4: logging

As part of our intent to migrate as much services as possible to our centralized rsyslog cluster (which then injects those logs into Splunk) we decided to approach GitLab's logging on OCP this way:

1. We mounted a shared PVC on each of the webservice/sidekiq pods, the target directory was the one GitLab was expected to send its logs by default (/var/log/gitlab)
2. From there we deployed a separate rsyslogd deployment that was also mounting the same shared PVC
3. We configured rsyslogd to relay those logs to our centralized rsyslog facility making sure proper facilities, tags, severities were also forwarded as part of the process
4. Relevant configs, Dockerfile and associated deployment files are [publicly available](https://gitlab.gnome.org/Infrastructure/openshift-images/gitlab-rsyslog)

## Future plans

Some of the tasks we have planned for the upcoming months:

1. Move away from ftpadmin and replace it with a web application and/or CLI to securely install a sources tarball without requiring shell access. (Also introduced tarball signatures?)
2. Introduce OpenID on GNOME's Matrix homeserver, merge existing Foundation member accounts
3. Migrate OCP ingress endpoints to Fastly LBs
4. Upgrade Ceph to Ceph 5
5. Look at migrating OCP to OVNKubernetes to start supporting IPv6 endpoints (again) - (minor priority)
6. Load balance IPA's DNS and LDAPs traffic (minor priority)
7. Migrate GitLab runners Ansible roles and playbooks to AAP (minor priority)

## Expressing my gratitude

I wanted to take a minute to thank all the individuals who helped us accomplishing this year amazing results! And a special thank you to Bartłomiej Piotrowski for his precious insights, technical skills and continued friendship.
