---
title: "Install Ansible Tower from Satellite 6"
date: 2020-07-26T19:22:35+01:00
draft: true
tags: [ansible, tower, satellite6]
---

In this post, we will look at how you can use Red Hat Satellite as a content source for installing Ansible Tower. You may be wondering why we even need to write this process up? Well the simple answer is that the Ansible Tower installation doesn’t follow the “usual” pattern. What is the “usual” pattern?

Many of the Red Hat products are made available via Red Hat’s Content Delivery Network (CDN) in the form of RPM repositories. Using Red Hat Satellite, you can import Red Hat Products and manage their associated RPM repositories. This allows you to store Red Hat software closer to the clients that need it or can’t reach the CDN directly, as well as other benefits like establishing versioned patch baselines. Take a look at the [Satellite datasheet](https://www.redhat.com/en/resources/satellite-datasheet) for more details.

In the case of Ansible Tower, the RPMs are not hosted in the CDN. Instead, the Ansible Tower installer is made available as a tar file. This installer assumes that you have internet access to retrieve the Tower RPMs from an Ansible specific repository - https://releases.ansible.com/ansible-tower/. For customers in disconnected environments, there is also a bundled tar file which includes both the installer and the Ansible Tower RPMs.

For many customers, the bundled tar file is sufficient and works well. However, some customers want or need to follow the same process for importing and managing RPMs for Tower as they do for other Red Hat software.

### Setting it up

The process for adding Ansible Tower content into Satellite is straightforward. I won’t go through all of the prerequisite steps to configure Red Hat Satellite and synchronise Red Hat content. There is a thorough [Content Management document](https://access.redhat.com/documentation/en-us/red_hat_satellite/6.6/html/content_management_guide/index) which goes over this as well as the core concepts in this blog post in more detail. Instead, we will focus on the steps needed to get Ansible Tower RPM content into Satellite and how we configure the Tower installer to consume this content.

In this example, I will be installing Ansible Tower on a Red Hat Enterprise Linux 8 server. As well as the Ansible Tower software, the following Red Hat repositories must also be made available (Always consult the [Ansible Tower installation documentation](https://docs.ansible.com/ansible-tower/latest/html/installandreference/index.html) for the latest requirements).

1. rhel-8-for-x86_64-baseos-rpms
2. rhel-8-for-x86_64-appstream-rpms
3. ansible-2-for-rhel-8-x86_64-rpms

### Adding the Red Hat GPG Key to Satellite

This step is optional but is considered a best practice. Ansible Tower RPMs are signed by the [Red Hat GPG key](https://access.redhat.com/security/team/key). To ensure clients are receiving signed content we can add the necessary GPG key into Satellite so that it is associated with our custom product.

In the Satellite web Interface, go to the **Content** menu and select **Content Credentials**. Click the **Create Content Credential** button. Now enter a name for the credential and ensure the Type is set to GPG Key. 

To get the GPG key, copy the contents of the file **/etc/pki/rpm-gpg/RPM-GPG-KEY-redhat-release** from any existing RHEL server, into the Content Credential Contents field. The result should look something like this.

![](/images/tower_from_sat_gpg.png)

### Create custom product for Ansible Tower

Next we can create our custom product for Ansible Tower. Go to the **Content** menu, select **Products** and click the **Create Product** button. Enter a name for the product and select the GPG key which you just created and then press Save.

![](/images/tower_from_sat_product.png)

You will now be taken to the **Repositories** tab of your new product. Click the **New Repository** button and complete the following information. Below is my example for Ansible Tower 3.6 on RHEL8:


```
Name: ansible-tower
Type: yum
Upstream URL: https://releases.ansible.com/ansible-tower/rpm/epel-8-x86_64/
GPG Key: Red Hat GPG Key
```

Repeat these steps for the Ansible Tower dependencies repository.

```
Name: ansible-tower-dependencies
Type: yum
Upstream URL: https://releases.ansible.com/ansible-tower/rpm/dependencies/3.6/epel-8-x86_64/
GPG Key: Red Hat GPG Key
```

Now let’s sync the Ansible Tower content, select both repositories via the checkboxes and press the **Sync Now** button. If you had already navigated away from this page, then you can also go to the **Content** menu and select **Sync Status**. From here you can select the repositories you would like to sync. 

### Create Content View

Satellite uses [Content Views](https://access.redhat.com/documentation/en-us/red_hat_satellite/6.6/html/content_management_guide/managing_content_views) to make content available to clients. A Content View allows us to take a point-in-time snapshot of one or more software repositories and promote that content throughout the various lifecycle environments you may have. For example, we could have Dev, Test & Production environments. Using Content Views allows me to promote new content through my environments so that I know it has been fully tested before I make it available to production servers. To create a Content View, go to the **Content menu** and select **Content Views**. Click **Create New View** and give your content view a name and description. I’m not creating a Composite Content View in this example but you may want to. You can learn more about Composite Content views in the [documentation](https://access.redhat.com/documentation/en-us/red_hat_satellite/6.6/html/content_management_guide/managing_content_views#Managing_Content_Views-Defining_Composite_Content_Views).

![](/images/tower_from_sat_cv.png)

After clicking **save**, we can select the repositories we want to include in this Content View. For RHEL8 I have included the following, which includes our custom Ansible Tower repositories.

![](/images/tower_from_sat_repos.png)

Click the **Publish New Version** button to publish the content view. Optionally, you may want to Promote the content view to the relevant [lifecycle environment](https://access.redhat.com/documentation/en-us/red_hat_satellite/6.6/html/content_management_guide/creating_an_application_life_cycle) for your host.

### Create an Activation Key

Again, this is optional. We can create an activation key to automate the process of registering new clients to Satellite, and ensuring they receive access to the correct subscriptions and software repositories. Go to the Content menu, select Activation Keys and click Create Activation Key. Provide a name, description and the lifecycle environment to associate to the activation key.

![](/images/tower_from_sat_ak.png)

After clicking **Save** we can further edit the activation key. We need to associate subscriptions and content to the key. From the **Subscriptions** tab, select the relevant subscriptions to cover RHEL and our custom Ansible Tower product:

![](/images/tower_from_sat_subs.png)

Finally, in the **Repository Set** tab, we can select the repositories which should be enabled for the client. I’m going to set the Ansible Tower repositories to be disabled. This is because the Ansible Tower installer will enable those repositories for us and I don’t want to install software from the Tower repositories unless I am running the Tower installer. More on this later. To set the repositories to disabled, select them via the checkbox and press the **Select Action** button and **Override to Disabled**. I’m also going to leave the “Red Hat Ansible Engine 2 for RHEL 8 x86_64 (RPMs)” repository disabled. Again, I don’t want to upgrade Ansible Engine each time I run yum update, but instead I want to be more selective over when I upgrade Ansible Engine. The repository sets should look as follows:

![](/images/tower_from_sat_enabled_repos.png)

### Register Tower server to Satellite

I’ve got a freshly installed RHEL8 server and I’m going to register it using my activation key. If you go to the **Content menu**, select **Activation Keys** and then select your Tower activation key, you will see the command you need to run to register a client with that key. In My example:

![](/images/tower_from_sat_reg_command.png)

Let’s check our key and ensure that only the RHEL8 baseos and appstream repositories are enabled:

```bash
# subscription-manager register --org="pharriso" --activationkey="Ansible Tower AK"
# subscription-manager repos --list-enabled | grep "Repo ID"
Repo ID:   rhel-8-for-x86_64-baseos-rpms
Repo ID:   rhel-8-for-x86_64-appstream-rpms
```

As the installation of Tower relies on Ansible Engine, we need to ensure Ansible is either installed or available to install. I’m going to install it manually from the RHEL8 Ansible Engine repository. Remember that I didn’t want the Ansible Engine repository to be enabled by default. I don’t want to update Ansible Engine every time I patch my server, but instead I want to be in control of when I update Ansible Engine. To install ansible, run the following command.

```bash
# yum -y install ansible --enablerepo=ansible-2-for-rhel-8-x86_64-rpms
```

Now we can download the Ansible setup tar file from [here](https://releases.ansible.com/ansible-tower/setup/) and unpack it. Obviously, in a disconnected environment you won’t be able to get straight out to download the file, but for the purposes of this demonstration, I will download the latest release directly to my Tower server (NOTE: this is the lightweight installer that does NOT include RPMs).

```bash
# curl -O https://releases.ansible.com/ansible-tower/setup/ansible-tower-setup-latest.tar.gz
# tar xvzf ansible-tower-setup-latest.tar.gz
```

As per the Ansible Tower installation documentation, you need to configure the inventory file ready for installation. The configuration of this file is out of scope for this blog post but the documentation can be found [here](https://docs.ansible.com/ansible-tower/latest/html/quickinstall/install_script.html#setup-inventory-file).

There are two configuration options that I will discuss though. These variables define which repositories the installer should use to get the Tower RPMs. First, let’s find the repository ID’s for our custom Ansible Tower content in Satellite. We can use subscription-manager on our Tower server to do this.

```bash
# subscription-manager repos

Repo ID:   pharriso_Ansible_Tower_ansible-tower
Repo Name: ansible-tower
Repo URL:  https://sat6.example.com/pulp/repos/pharriso/Library/RHEL8_Ansible_Tower/custom/Ansible_Tower/ansible-tower
Enabled:   0

Repo ID:   pharriso_Ansible_Tower_ansible-tower-dependencies
Repo Name: ansible-tower-dependencies
Repo URL:  https://sat6.example.com/pulp/repos/pharriso/Library/RHEL8_Ansible_Tower/custom/Ansible_Tower/ansible-tower-dependencies
Enabled:   0

<output truncated>
```

Now that we have repository ID’s, we can set our variables in the Tower installer **inventory** file. Under the **[all:vars]** section, set the following.

```bash
ansible_tower_repo=pharriso_Ansible_Tower_ansible-tower
ansible_tower_dependency_repo=pharriso_Ansible_Tower_ansible-tower-dependencies
```

### Run installer

Let’s run the **setup.sh script** to install Ansible Tower. If everything was configured correctly, you should see a message saying “The setup process completed successfully”.

```bash
# cd ansible-tower-setup-3.6.3-1
# ./setup.sh

<output truncated> 

The setup process completed successfully.
```

We can also verify that the Ansible Tower RPMs were installed from Satellite. For example:

```bash
# yum info ansible-tower
Updating Subscription Management repositories.
Installed Packages
Name         : ansible-tower
Version      : 3.6.3
Release      : 1.el8at
Arch         : x86_64
Size         : 0.0   
Source       : ansible-tower-3.6.3-1.el8at.src.rpm
Repo         : @System
From repo    : pharriso_Ansible_Tower_ansible-tower
Summary      : REST api, server, plugins, UI, & CLI client for ansible
URL          : http://ansible.github.com
License      : Commercial, see https://www.redhat.com/en/about/red-hat-end-user-license-agreements and https://www.redhat.com/en/about/licenses
Description  : REST api, server, plugins, Web UI, & CLI client for ansible.
```

### Summary

Using Red Hat Satellite’s content management capability, we have synced the Ansible Tower content, versioned it using Content Views and automated the consumption of this content with activation keys. This process allows customers in disconnected or highly regulated environments to utilise their existing mechanisms to manage the content required to install Ansible Tower. 
