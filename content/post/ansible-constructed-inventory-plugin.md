---
title: "Ansible Constructed Inventory Plugin"
date: 2020-01-16T09:40:51+01:00
draft: false
tags: [ansible, tower]
---

In this blog post we will look at how we can easily enrich an existing dynamic inventory using the constructed inventory plugin. But, before we do that, let's take a step back.

Ansible is simple automation tool that allows users to achieve common use cases such as configuration management, provisioning and complex multi-tier orchestration. Part of what makes Ansible so simple, is the fact that it is agentless. One doesn't need to deploy agents or any heavyweight control software to get up and running with Ansible automation. So, if Ansible is agentless, how does it know what nodes it should manage? We use an [inventory](https://docs.ansible.com/ansible/latest/user_guide/intro_inventory.html#inventory-basics-formats-hosts-and-groups) in Ansible to provide it with this list of managed nodes. 

In it's simplest form, an inventory is just an INI or YAML file which contains a list of our managed nodes. This is ideal when getting started with Ansible, but as you start to really use Ansible in anger and at scale you will probably face a couple of questions.


1. How do I classify my infrastructure so that I can be more selective in what devices I automate against?
2. How do I effectively and efficiently maintain a list of all of my managed nodes?



The answer to both of these questions, can quite often be to use a [dynamic inventory](https://docs.ansible.com/ansible/latest/user_guide/intro_dynamic_inventory.html). A dynamic inventory is a script or plugin which will go to a source of truth and discover the nodes I need to manage. It will also automatically classify the nodes by putting them into groups which can be used to more selectively target devices when automating with Ansible. There are a number of sources that Ansible can talk to out of the box such as VMware vCenter, AWS, GCP, Azure or Red Hat Satellite (Foreman upstream). There are also community inventory scripts such this the [ServiceNow CMDB](https://github.com/ServiceNowITOM/ansible-sn-inventory) script.  And finally, you can [develop your own](https://docs.ansible.com/ansible/latest/dev_guide/developing_inventory.html) dynamic inventory. The benefit to me as a user of Ansible is quite clear. I now no longer need to maintain a list of managed nodes. As I add or remove managed nodes, the dynamic inventory will continue to provide me with an up to date list.

But the dynamic inventory scripts and plugins also provide us with another benefit. They provide rich information about our managed nodes and the environment they are running in. This information is made available to us in the form of variables which we can use in our automation. For example, if we use the AWS (EC2) inventory plugin, then we will retrieve variables for each discovered device such as private IP addresses, regions, security groups and so on. 

### Dynamic Inventory Example

Let's look at an example of an inventory plugin to see what type of information it returns. I'm going to look at the Red Hat Satellite/Foreman plugin. [Red Hat Satellite](https://www.redhat.com/en/technologies/management/satellite) is a tool that is often used for managing RHEL content and patching of RHEL servers. The plugin is actually called foreman because foreman is one of the upstream components in Red Hat Satellite. 

Here are the contents of my inventory plugin file. We have a plaintext password here but we'll see how we can avoid that later with Ansible Tower.

```bash
plugin: foreman
url: https://sat6.demolab.local
user: sat-tower
password: Redhat123
validate_certs: False
```

Below is a list of the groups that are returned by my Satellite server. For example, we can see a group called foreman_hostgroup_prod_web which contains two hosts - prod-web1.demolab.local and prod-web2.demolab.local. This is because I have a hostgroup in Satellite which these two servers are members of.

```bash
$ ansible-inventory -i foreman.yaml --graph

@all:
  |--@foreman_dev_web:
  |  |--dev-web1.demolab.local
  |--@foreman_prod_haproxy:
  |  |--lb.demolab.local
  |--@foreman_prod_web:
  |  |--prod-web1.demolab.local
  |  |--prod-web2.demolab.local
```

As well as the groupings, we also get a number of facts from Satellite. Again, here are some snippets from one of the hosts:

```bash
$ ansible-inventory -i foreman.yaml --host prod-web1.demolab.local

"foreman_subscription_facet_attributes": {
    "autoheal": true, 
    "id": 4883, 
    "last_checkin": "2020-01-14 02:08:23 UTC", 
    "purpose_addons": [], 
    "purpose_role": null, 
    "purpose_usage": null, 
    "registered_at": "2019-12-10 10:21:03 UTC", 
    "registered_through": "sat6.demolab.local", 
    "release_version": null, 
    "service_level": "", 
    "user": null, 
    "uuid": "3a9c2c19-57e4-4f98-b675-99d50d2d5b1e"
    
   
    "lifecycle_environment_name": "Prod", 
    "upgradable_module_stream_count": 0, 
    "upgradable_package_count": 0, 
...
  
    "foreman_location_name": "London", 
...
   
    "foreman_organization_name": "Red Hat", 
  
...
    "foreman_ip": "10.50.0.31", 
```

This is just a fraction of the information we get from the inventory source, but we can see lot's of useful information here about the subscription status, available errata and other foreman objects. These facts are actually host_vars which we can use in our automation.

### How can we "construct" additional data?

So far we have been talking about pretty standard stuff with regards inventories. We've seen the type of facts and groups we get back from the foreman inventory plugin. But what if we want to dynamically create more groups based on the facts that we are getting back? Of course, if you have the necessary skills then you can modify the inventory plugin to return the information that you need. But, you may not need to do that. There is a really useful inventory plugin which serves this purpose. The [constructed](https://docs.ansible.com/ansible/latest/plugins/inventory/constructed.html) inventory plugin takes information from another inventory source and uses that to construct new groups and variables. I really like this because it allows us to keep Ansible simple and avoid writing code if we don't want to.

Let's use our foreman example above to construct some new groups. As an example use case, let's say I want to group hosts based on the Satellite node they are registred through. This would potentially allow me to identify which datacenter or network zone they are in which could be useful if I need to use a jump host to allow Ansible to manage them. Looking at the output above, we can see that this information is available as a variable:

**foreman_subscription_facet_attributes.registered_through**

I'm also going to group based on lifecycle environment in Satellite - so Prod or Dev in my case.

For that I need the following variable:

**foreman_content_facet_attributes.lifecycle_environment_name**

### Example 1 - Generating new groups

We need a directory for our foreman inventory plugin to live in so that we can source multiple inventory plugins or scripts.

```bash
inventories/
├── foreman_constructed.yml
└── foreman.yml
```

**NOTE** the constructed inventory plugin relies on data from another inventory  source so we need the foreman.yml plugin to be invoked before the constructed inventory. When sourcing a directory as an Ansible inventory they are executed alphabetically. More information can be found [here](https://docs.ansible.com/ansible/latest/user_guide/intro_inventory.html#using-multiple-inventory-sources).

Here are the contents of my **foreman_constructed.yml** file.

```bash
plugin: constructed
strict: False
keyed_groups:
  -  prefix: ""
     separator: ""
     key: foreman_subscription_facet_attributes.registered_through
  -  prefix: ""
     separator: ""
     key: foreman_content_facet_attributes.lifecycle_environment_name
```

Now we can source the directory which will incorporate both inventory sources:

```bash
ansible-inventory -i inventories/ --graph
@all:
  |--@Dev:
  |  |--dev-web1.demolab.local
  |--@Prod:
  |  |--lb.demolab.local
  |  |--prod-web1.demolab.local
  |  |--prod-web2.demolab.local
  |--@foreman_dev_web:
  |  |--dev-web1.demolab.local
  |--@foreman_prod_haproxy:
  |  |--lb.demolab.local
  |--@foreman_prod_web:
  |  |--prod-web1.demolab.local
  |  |--prod-web2.demolab.local
  |--@sat6_demolab_local:
  |  |--dev-web1.demolab.local
  |  |--lb.demolab.local
  |  |--prod-web1.demolab.local
```

Note the new groups we now have available to us. I can now target these new groups or assign variables to them using group_vars.

### Example 2 - Generating new variables

As well as generating new groups, the constructed inventory plugin can also be used to "compose" new variables. For this example, I am going to use the IP address that Satellite provided me with **foreman_ipa** variable to set the **ansible_host** variable.

The complete **foreman_constructed.yml** file now looks as follows:

```bash
plugin: constructed
strict: False
compose:
  ansible_host: foreman_ip
keyed_groups:
  -  prefix: ""
     separator: ""
     key: foreman_subscription_facet_attributes.registered_through
  -  prefix: ""
     separator: ""
     key: foreman_content_facet_attributes.lifecycle_environment_name
```

This results in the ansible_host variable being set:

```bash
$ ansible-inventory -i inventories/ --host prod-web1.demolab.local

{
    "ansible_host": "10.50.0.31", 
```

### Using the constructed inventory in Tower

I've been using Ansible Engine and the command line so far. But what happens if I am using Ansible Tower for my Ansible automation. The good news is that using a constructed inventory in Ansible Tower is straight forward. We will source the inventory plugins from a source control repository. This ensures I can use source control branching techniques to maintain control over any changes before they are pushed to production.

My source control repository has the same structure as before with the exception that I no longer need the foreman.ini file. This is because I will pass my credentials from Tower. The repository is [here](https://github.com/pharriso/ansible_constructed_inventory). 

```bash
inventories/
├── foreman_constructed.yml
└── foreman.yml
```

Also, note how the foreman.yml file no longer contains a username, password or Satellite server URL. 

```bash
plugin: foreman
validate_certs: False
```

We will use Ansible Tower's credential management capabilities to pass these details to the inventory plugin. But, first we need to create a project in Tower which will pull in our inventory files from source control.

![](/images/constructed-tower-project.png)

As previously mentioned, we are going to use [Ansible Tower's credential management](https://docs.ansible.com/ansible-tower/latest/html/userguide/credentials.html) to store all of the credentials that we need to talk to our Satellite server. We need to define a custom credential type to talk to Satellite for the purpose of our inventory plugin. In the Tower UI, Navigate to **Credential Types** and add a new credential with the following details:

**INPUT CONFIGURATION**
```bash
fields:
  - id: FOREMAN_USER
    type: string
    label: Username
  - id: FOREMAN_PASSWORD
    type: string
    label: Password
    secret: true
  - id: FOREMAN_SERVER
    type: string
    label: Satellite Server
required:
  - FOREMAN_USER
  - FOREMAN_PASSWORD
  - FOREMAN_SERVER
```

**INJECTOR CONFIGURATION**
```bash
env:
  FOREMAN_PASSWORD: '{{ FOREMAN_PASSWORD }}'
  FOREMAN_SERVER: '{{ FOREMAN_SERVER }}'
  FOREMAN_USER: '{{ FOREMAN_USER }}'
```

You can look at the docs page for the various inventory plugins to understand what variables they expect. An example with a snippet of output can be seen below for the foreman plugin:

```bash
$ ansible-doc -t inventory foreman

= user
        foreman authentication user
 
        set_via:
          env:
          - name: FOREMAN_USER
```

Now that we have a credential type in Ansible Tower, we can create a credential using this new type. 

![](/images/constructed-tower-credential.png)

Next we need to create an inventory:

![](/images/constructed-tower-inventory.png)

Then we can add an inventory source to the inventory. Ensure to add the correct credential that we created earlier.

![](/images/constructed-tower-inventory-source.png)

Once the inventory source has finished syncing, we should see that the relevant hosts have been imported with the constructed groups and composed variables.

![](/images/constructed-tower-inventory-groups.png)

### Summary

The constructed inventory plugin can be really useful for manipulating the information returned from existing dynamic inventory plugins and scripts. This example used a plugin but the constructed inventory plugin works in the same way with inventory scripts. It is worth noting that some inventory plugins provide in-built capabilities to construct variables and generate groups. Check the [inventory plugins](https://docs.ansible.com/ansible/latest/plugins/inventory.html#plugin-list) before deciding if you need to also use the constructed inventory plugin.


