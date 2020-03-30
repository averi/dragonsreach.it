---
title: GNOME Infrastructure updates
author: Andrea Veri
type: post
date: 2020-03-30T12:20:23+00:00
url: /2020/03/30/2020-03-30-gnome-infrastructure-updates
categories:
  - Planet GNOME
tags:
  - planet-gnome
  - sysadmin

---
As you may have noticed from outage and maintenance notes we sent out last week the GNOME Infrastructure has been undergoing a major redesign due to the need of moving to a different datacenter. It's probably a good time to update the Foundation membership, contributions and generally anyone consuming the multitude of services we maintain of what we've been up to during these past months.

## New Data Center

One of the core projects for 2020 was moving off services from the previous DC we were in (located in PHX2, Arizona) over to the Red Hat community cage located in RAL3. This specific task was made possible right after we received a new set of machines that allowed us to refresh some of the ancient hardware we had (with the average box dating back to 2013). The new layout is composed of a total of 5 (five) bare metals and 2 (two) core technologies: Openshift (v. 3.11) and Ceph (v. 4).

The major improvements that are worth being mentioned:

 1. VMs can be easily scheduled across the hypervisors stack without having to copy disks over across hypervisors themselves. VM disks and data is now hosted within Ceph.
 2. IPv6 is available (not yet enabled/configured at the OS, Openshift router level)
 3. Overall better external internet uplink bandwidth
 4. Most of the VMs that we had running were turned into pods and are now successfully running from within Openshift

## RHEL 8 and Ansible

One of the things we had to take into account was running Ceph on top of RHEL 8 to benefit from its containarized setup. This originally presented itself as a challenge due to the fact RHEL 8 ships with a much newer Puppet release than the one RHEL 7 provides. At the same time we didn't want to invest much time in upgrading our Puppet code base due to the amount of VMs we were able to migrate to Openshift and to the general willingess of slowly moving to use Ansible (client-side, no more need of maintaining server side pieces). On this specific regard we:

 1. Landed support for RHEL 8 provisioning
 2. Started experimenting with Image Based deployments (much more faster than Cobbler provisioning)
 3. Cooked a set of [base Ansible roles](https://gitlab.gnome.org/Infrastructure/ansible/-/tree/master/roles) to support our RHEL 8 installs including IDM, chrony, Satellite, Dell OMSA , NRPE etc.

## Openshift

As [originally announced](https://www.dragonsreach.it/2018/10/18/2018-10-18-gnome-infrastructure-moving-to-openshift), the migration to the Openshift Container Platform (OSCP) has progressed and we now count a total of 34 tenants (including the entirety of GIMP websites). This allowed us to:

 1. Retire running VMs and prevented the need to upgrade their OS whenever they're close to EOL. Also, in general, less maintenance burden
 2. Allow the community to easily provision services on top of the platform with total autonomy by choosing from a wide variety of frameworks, programming languages and database types (currently Galera and PSQL, both managed outside of OSCP itself)
 3. Easily scale the platform by adding more nodes/masters/routers whenever that is made necessary by additional load
 4. Data replicated and made redundant across a GlusterFS cluster (next on the list will be introducing Ceph support for pods persistent storage
 5. Easily set up services such as Rocket.Chat and Discourse without having to mess much around with Node.JS or Ruby dependencies

## Special thanks

I'd like to thank Bartłomiej Piotrowski for all the efforts in helping me out with the migration during the past couple of weeks and Milan Zink from the Red Hat Storage Team who helped out reviewing the Ceph infrastructure design and providing useful information about possible provisioning techniques.
