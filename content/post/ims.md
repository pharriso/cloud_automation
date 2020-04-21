---
title: "Infrastructure Migration Solution"
date: 2019-01-09T19:47:46+01:00
draft: false
tags: [rhv,ims]
---

While VMware is undoubtedly a great virtualisation offering (I spent a number of users working with VMware tech) some customers are looking beyond traditional virtualisation. The growing adoption of containerisation is a great example of this and using RHV as a stepping stone to Container Native Virtualisation (CNV) is a good example use case.

Other customers simply want to save costs and are looking at alternative virtualisation offerings. Whatever the reason for moving from VMware, we need a method for migrating workloads to RHV.

This is what our Infrastructure Migration Solution (IMS) provides. A simple method for migrating workloads from VMware to RHV. We use a tool called CloudForms to drive the migration process (ManageIQ is the upstream project for CloudForms). 

CloudForms is a manager of managers and can interact with various endpoints which are known as providers. A provider could be VMware vSphere, Red Hat Virtualisation, AWS, OpenStack etc. CloudForms is able to ingest information from these providers such as configuration details and performance metrics as well as orchestrate these providers. 

Using the new CloudForms "migration" tooling we can create mappings between VMware infrastructure and Red Hat Virtualisation. These include clusters, storage and networking. We can then detect the VM's currently running in VMware and using virt-v2v under the covers we can automate the migration of those VM's across to RHV.

This short video shows a demo of the process.

[![IMS](/images/ims.png)](http://www.youtube.com/watch?v=NdjGuJaDSOU)


