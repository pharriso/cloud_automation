---
title: "Ansible Tower Dynamic Inventories - Manage Enabled Hosts"
date: 2020-04-20T17:41:05+01:00
draft: false
---

# Ansible Tower Dynamic Inventories - Manage Enabled Hosts

If you have used [dynamic inventories in Ansible Tower](https://docs.ansible.com/ansible-tower/latest/html/userguide/inventories.html#smart-inventories) before, you may have noticed that it adds some logic to determine whether a host should be enabled for job execution or not. For example, when syncing with our Red Hat Virtualisation environment, it has set some of our hosts to disabled.

![tower_disabled_hosts](/cloud_automation/images/tower_inventory_disabled.png)

