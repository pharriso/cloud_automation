---
title: "Ansible Tower Dynamic Inventories - Manage Enabled Hosts"
date: 2020-02-10T17:41:05+01:00
draft: false
tags: [ansible, tower]
---

If you have used [dynamic inventories in Ansible Tower](https://docs.ansible.com/ansible-tower/latest/html/userguide/inventories.html#smart-inventories) before, you may have noticed that it adds some logic to determine whether a host should be enabled for job execution or not. For example, when syncing with our Red Hat Virtualisation environment, it has set some of our hosts to disabled.

![tower_disabled_hosts](/images/tower_inventory_disabled.png)

Why do we get this behaviour? In the case of Red Hat Virtualisation you can probably tell from the screen shot - Tower will disable any VM's which are powered off and we can see the VM's which are disabled show a state of "status_down". For VMware, it will disable any VM's where VMware tools isn't reporting a guest state. A lot of the time this is really useful and can reduce job failures - why try to connect to a VM which is powered off? But, there may be times when you want to override this behaviour. For example, you might want to deploy VMware Tools to some guests, but they are being excluded from job execution. Another example we had recently was that we wanted to delete a bunch of VM's in our Red Hat Virtualisation platform which were powered off, but they were being excluded - because they were powered off. Rather than powering on the VM's so that we could delete them, we just amended the behaviour in Tower so that these VM's weren't marked as disabled.

To see how Tower determines if a node is enabled or disabled, you can take a look in **/var/lib/awx/venv/awx/lib/python3.6/site-packages/awx/settings/defaults.py**

Looking at RHV (Upstream name is oVirt), we can see:

```bash
# ---------------------
# ----- oVirt4 -----
# ---------------------
RHV_ENABLED_VAR = 'status'
RHV_ENABLED_VALUE = 'up'
```

And for VMware:

```bash
# Inventory variable name/values for determining whether a host is
# active in vSphere.
VMWARE_ENABLED_VAR = 'guest.gueststate'
VMWARE_ENABLED_VALUE = 'running'
```

## Changing the default behaviour

Now that we have seen the default settings and know the variable names, we can override the default behaviour. The easiest way to do this is to create a custom configuration file in /etc/tower/conf.d and override the variable value. For example, to enable all VM's regardless of power status you could create the following file and set the RHV_ENABLED_VALUE to be empty:

```bash
$ cat /etc/tower/conf.d/custom.py 
RHV_ENABLED_VALUE = ''
```

After creating this configuration file, restart the tower services:

```
$ sudo ansible-tower-service
```

Now when we sync the inventory we can now see that all of the hosts are enabled regardless of power state:

![tower_enabled_hosts](/images/tower_inventory_enabled.png)

## Summary

Ansible Tower tries to be helpful in the way it disables hosts that have been imported from dynamic inventories to try to avoid unnecessary job failures. If you need to configure this behaviour then you can follow the steps outlined above. These steps should not be confused with dynamic inventory filters, which will completely exclude a host from being imported into Ansible Tower. 
