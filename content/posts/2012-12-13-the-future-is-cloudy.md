---
title: The future is Cloudy
author: Andrea Veri
type: post
date: 2012-12-13T14:22:25+00:00
url: /2012/12/13/the-future-is-cloudy/
categories:
  - Cloud Computing
tags:
  - Cloud
  - Cloud Computing

---
Have you ever heard someone talking extensively about **Cloud Computing** or generally **Clouds**? and have you ever noticed the fact many people (even the ones who present themselves as experts) don&#8217;t really understand what a Cloud is at all? That happened to me multiple times and one of the most common misunderstandings is many see the Cloud as something being on the **internet**. Many companies add a little **logo** representing a cloud on their frontpage and without a single change on their infrastructure (but surely with a **price increment**) they start calling their products as being on the Cloud. Given the lack of knowledge about this specific topic people tend to buy the product presented as being on the Cloud without understanding what they really bought.

<img class="aligncenter size-full wp-image-706" title="cloud-computing" src="http://www.dragonsreach.it/wp-content/uploads/2012/12/cloud-computing.png" alt="" width="400" height="362" />

But what Cloud Computing really means? it took several years and more than fifteen drafts to the **National Institute of Standards and Technology**&#8216;s (**NIST**) to find a definition. The final accepted proposal:

> <p style="text-align: left;">
>   <em><strong>Cloud computing is a model for enabling ubiquitous, convenient, on-demand network access to a shared pool of configurable computing resources (e.g., networks, servers, storage, applications and services) that can be rapidly provisioned and released with minimal management effort or service provider interaction.</strong></em>
> </p>

The above definition requires a few more clarifications specifically when it comes to understand where should we focus on while checking for a Cloud Computing solution. A few key points:

  1. **On-demand self-service**: every consumer will be able to unilaterally provision multiple computing capabilities like server time, storage, bandwidth, dedicated RAM or CPU without requiring any sort of human interaction from their respective Cloud providers.
  2. **Rapid elasticity and scalability**: all the computing capabilities outlined above can be elastically provisioned and released depending on how much demand my company will have in a specific period of time. Suppose the X company is launching a new product today and it expects a very large number of customers. The X company will add more resources to their  Cloud for the very first days (where they suppose the load to be very high) and then it&#8217;ll scale the resources back as they were before. Elasticity and scalability permit the X company to improve and enhance their infrastructure when they need it with an huge saving in monetary terms.
  3. **Broad network access**: capabilities are available over the network and accessed through standard mechanisms that promote use by heterogeneous thin or thick client platforms (e.g., mobile phones, tablets, laptops, and workstations).
  4. **Measured service**: Cloud systems allow maximum transparency between the provider and the consumer, the usage of all the resources is monitored, controlled, and reported. The consumer knows how much will spend, when and in how long.
  5. **Resource pooling**: each provider&#8217;s computing resources are pooled to serve multiple consumers at the same time. The consumer has no control or knownledge over the exact location of the provided resources but may be able to specify location at a higher level of abstraction (e.g., country, state, or datacenter).
  6. **Resources price**: when buying a Cloud service make sure the cost for two units of RAM, storage, CPU, bandwidth, server time is exactly the double of the price of one unit of the same capability. An example, if a provider offers you one hour of bandwitdh for 1 Euro, the price of  two hours will have to be 2 Euros.

<div>
  Another common error I usually hear is people feeling Cloud Computing just as a place to put their files online as a backup or for sharing them with co-workers and friends. That is just one of the available Cloud &#8220;<strong>features</strong>&#8220;, specifically the &#8220;<strong>Cloud Storage</strong>&#8220;, where typical examples are companies like <strong>Dropbox</strong>, <strong>Spideroak</strong>, <strong>Google Drive</strong>,<strong> iCloud</strong> and so on. But let&#8217;s make a little note about the other three &#8220;features&#8221;:
</div>

<div>
  <ol>
    <li>
      <strong>Infrastructure as a Service</strong> (<strong>IaaS</strong>): the capability provided to the consumer is to provision processing, storage, networks, and other fundamental computing resources where the consumer is able to deploy and run arbitrary software, which can include operating systems and applications. In this specific case the consumer has still no control or management over the underlying Cloud infrastructure but has control over operating systems, storage, and deployed applications. A customer will be able to add and destroy virtual machines (VMs), install an operating system on them based on custom kickstart files and eventually manage selected networking components like firewalls, hosted domains, accounts.
    </li>
    <li>
      <strong>Platform as a Service</strong> (<strong>PaaS</strong>). the capability provided to the consumer is to deploy onto the cloud infrastructure consumer-created or acquired applications created using programming languages, libraries, services, and tools  (like Mysql + PHP + PhpMyAdmin or Ruby on Rails) supported by the provider. In this specific case the consumer has still no control or management over the Cloud infrastructure itself (servers, OSs, storage, bandiwitdh etc.) but has control over the deployed applications and configuration settings for the application-hosting environment.
    </li>
    <li>
      <strong>Software as a Service</strong> (<strong>SaaS</strong>): the capability provided to the consumer is to use the provider’s applications running on a Cloud infrastructure. The applications are accessible through various client devices, such as a browser, a mobile phone or a program interface. The consumer doesn&#8217;t not manage nor control the Cloud infrastructure (servers, OSs, storage, bandwidth, etc.) that allows the applications to run. Even the provided applications aren&#8217;t customizable by the consumer, which should rely on limited configuration settings.
    </li>
  </ol>
</div>

[<img class="size-full wp-image-704 aligncenter" title="cloud-service-models" src="http://www.dragonsreach.it/wp-content/uploads/2012/12/cloud-service-models.jpg" alt="" width="550" height="322" />][1]

The Cloud Computing technology is reasonably the future but can we trust Cloud providers? Are we sure that no one will ever have access to our files except us? and what about governments interested in acquiring a specific customer data hosted on the Cloud?

I always suggest to read deeply both the **Privacy Policy** and **Terms of Use** of a certain service before signing in especially when it comes to choose a Cloud storage provider. Many providers have the same aspect, they seem to provide the same resources, the same amount of storage for the same price but legally they may present different problems, and that is the case of **Spideroak** vs **Dropbox**. Quoting from the Dropbox&#8217;s **Privacy Policy**:

> <div>
>   <strong><em>Compliance with Laws and Law Enforcement Requests; Protection of DropBox’s Rights. We may disclose to parties outside Dropbox files stored in your Dropbox and information about you that we collect when we have a good faith belief that disclosure is reasonably necessary to (a) comply with a law, regulation or compulsory legal request; (b) protect the safety of any person from death or serious bodily injury; (c) prevent fraud or abuse of DropBox or its users; or (d) to protect Dropbox’s property rights. If we provide your Dropbox files to a law enforcement agency as set forth above, we will remove Dropbox’s encryption from the files before providing them to law enforcement. However, Dropbox will not be able to decrypt any files that you encrypted prior to storing them on Dropbox.</em></strong>
> </div>

It&#8217;s evident that Dropbox employees can access your data or be forced by legal process to turn over your data **unencrypted**. On the other side, Spideroak on its latest update to its [Privacy Policy][2] states that data stored on their  Cloud  is **encrypted** and **inaccessible** without user&#8217;s key, which is stored locally on user&#8217;s computers.

And what about the latest research paper, titled &#8220;**<a href="http://papers.ssrn.com/sol3/papers.cfm?abstract_id=2181534" target="_blank">Cloud Computing in Higher Education and Research Institutions and the USA Patriot Act</a>**&#8221; written by the legal experts of the **University of Amsterdam&#8217;s Institute for Information Law** stating the anti-terror **Patriot Act** could be theoretically used by U.S. law enforcement to bypass strict European privacy laws to acquire citizen data within the European Union without their consensus?

The only requirement for the data acquisition is the provider being an U.S company or an European company conducting systematic business in the U.S. For example an Italian company storing their documents (protected by the European privacy laws and under the general Italian jurisdiction) on a provider based in Europe but conducting systematic business in the United States, could be forced by U.S. law enforcement to transfer data to the U.S. territory for inspection by law enforcement agencies.

Does someone really **care** about the **privacy** of companies, consumers and users at all? or better does **privacy** exists at all for the millions of the people that connect to the internet every day?

 [1]: http://www.dragonsreach.it/wp-content/uploads/2012/12/cloud-service-models.jpg
 [2]: https://spideroak.com/blog/20120502022627-spideroak-privacy-policy-update
