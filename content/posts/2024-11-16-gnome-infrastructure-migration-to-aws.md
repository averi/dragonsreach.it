---
title: "GNOME Infrastructure migration to AWS"
date: 2024-11-16T14:31:23+01:00
type: post
url: /2024/11/16/gnome-infrastructure-migration-to-aws
categories:
  - Planet GNOME
tags:
  - planet-gnome
  - cloud-computing
  - openshift
  - aws

---

## 1. Some historical background

The GNOME Infrastructure has been hosted as part of one of Red Hat's datacenters for over 15 years now. The "community cage", which is how we usually define the hosting platform that backs up multiple Open Source projects including [OSCI](https://www.osci.io), is made of a set of racks living within the RAL3 (located in Raleigh) datacenter. Red Hat has not only been contributing to GNOME by maintaining the Red Hat's Desktop Team operational, sponsoring events (such as GUADEC) but has also been supporting the project with hosting, internet connectivity, machines, RHEL (and many other RH products subscriptions). When the infrastructure was originally stood up it was primarily composed of a set of bare metal machines, workloads were not yet virtualized at the time and many services were running directly on top of the physical nodes. The advent of virtual machines and later containers reshaped how we managed and operated every component. What however remained the same over time was the networking layout of these services: a single L2 and a shared (with other tenants) public internet L3 domains (with both IPv4 and IPv6).

## Recent challenges

When GNOME's Openshift 4 environment was built back in 2020 we had to make specific calls:

1. We'd have ran an Openshift Hyperconverged setup (with storage (Ceph), control plane, workloads running on top of the same subset of nodes)
2. The total amount of nodes we received budget for was 3, this meant running with masters.schedulable=true
3. We'd have kept using our former Ceph cluster (as it had slower disks, a good combination for certain workloads we run), this is however not supported by ODF (Openshift Data Foundation) and would have required some glue to make it completely functional
4. Migrating GNOME's private L2 network to L3 would have required an effort from Red Hat's IT Network Team who generally contributes outside of their working hours, no changes were planned in this regard
5. No changes were planned on the networking equipment side to make links redundant, that means a code upgrade on switches would have required a full services downtime

Over time and with GNOME's users and contributors base growing (46k users registered in GitLab, 7.44B requests and 50T of traffic per month on services we host on Openshift and kindly served by Fastly's load balancers) we started noticing some of our original architecture decisions weren't positively contributing to platform's availability, specifically:

1. Every time an Openshift upgrade was applied, it resulted in a cluster downtime due to the unsupported double ODF cluster layout (one internal and one external to the cluster). The behavior was stuck block devices preventing the machines to reboot with associated high IO (and general SELinux labeling mismatches), with the same nodes also hosting OCP's control plane it was resulting in API and other OCP components becoming unavailable
2. With no L3 network, we had to create a next-hop on our own to effectively give internet access through NAT to machines without a public internet IP address, this was resulting in connectivity outages whenever the target VM would go down for a quick maintenance

## Migration to AWS

With budgets season for FY25 approaching we struggled finding the necessary funds in order to finally optimize and fill the gaps of our previous architecture. With this in mind we reached out to [AWS Open Source Program](https://aws.amazon.com/opensource/) and received a substantial amount for us to be able to fully transition GNOME's Infrastructure to the public cloud. 

What we achieved so far:

1. Deployed and configured VPC related resources, this step will help us resolve the need to have a next-hop device we have to maintain
2. Deployed an Openshift 4.17 cluster (which uses a combination of network and classic load balancers, x86 control plane and arm64 workers)
3. Deployed IDM nodes that are using a Wireguard tunnel between AWS and RAL3 to remain in sync
4. Migrated several applications including SSO, Discourse, Hedgedoc

What's upcoming:

1. Migrating away from Splunk and use a combination of rsyslog/promtail/loki
2. Keep migrating further applications, the idea is to fully decommission the former cluster and GNOME's presence within Red Hat's community cage during Q1FY25
3. Introduce a [replacement](https://gitlab.gnome.org/Infrastructure/openshift-images/gnome-release-service) for master.gnome.org and GNOME tarballs installation
4. Migrate applications to GNOME's SSO
5. Retire services such as GNOME's wiki (MoinMoin, a static copy will instead be made available), NSD (authoritative DNS servers were outsourced and replaced with ClouDNS and GitHub's pipelines for DNS RRs updates), Nagios, Prometheus Blackbox (replaced by ClouDNS endpoints monitoring service), Ceph (replaced by EBS, EFS, S3)
6. Migrate smtp.gnome.org to OSCI in order to maintain current public IP's reputation

And benefits of running GNOME's services in AWS:

1. Scalability, we can easily scale up our worker nodes pool
2. We run our services on top of AWS SDN and can easily create networks, routing tables, benefit from faster connectivity options, redundant networking infrastructure
3. Use EBS/EFS, don't have to maintain a self-managed Ceph cluster, easily scale volumes IOPS
4. Use a local to-the-VPC load balancer, less latency for traffic to flow between the frontend and our VPC
5. Have access to AWS services such as AWS Shield for advanced DDOS protection (with one bringing down GNOME's GitLab just a week ago)

I'd like to thank AWS (Tom "spot" Callaway, Mila Zhou) for their sponsorship and the massive opportunity they are giving to the GNOME's Infrastructure to improve and provide resilient, stable and highly available workloads to GNOME's users and contributors base. And a big thank you to Red Hat for the continued sponsorship over more than 15 years on making the GNOME's Infrastructure run smoothly and efficiently, it's crucial for me to emphatise how critical Red Hat's long term support has been.