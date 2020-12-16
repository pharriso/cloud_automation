---
title: "Testing Ansible roles with Molecule and VMware"
date: 2020-12-08T20:29:31Z
draft: false
tags: [ansible, tower, molecule, vmware]
---

Molecule is a tool to aid with the testing and development of [Ansible roles](https://docs.ansible.com/ansible/latest/user_guide/playbooks_reuse_roles.html). It allows users to spin up test infrastructure to provide a quick and easy means with which to test Ansible roles. Typically the test instances are containers as they are lightweight and typically very fast to create and destroy. There are a number of posts out there that describe the process of using [molecule with containers](https://www.ansible.com/blog/developing-and-testing-ansible-roles-with-molecule-and-podman-part-1) but I didn't find many that describe the process for testing molecule with Virtual Machines.

There may be valid reasons for wanting to test Ansible roles on a VM rather than in a container. In this example, I want to test an Ansible role that is used to register VM's to Red Hat Satellite. Containers typically adopt the subscriptions and repositories of the host system so it felt like a good use-case for deploying a VM to test my role. I'm going to use VMware vSphere as the target platform for my test VM's as this is the most common on-prem virtualisation platform that I come across with customers.

Molecule provides a delegated driver which gives users the flexibility of Ansible to write their own playbooks to provision test infrastructure - "Under this driver, it is the developers responsibility to implement the create and destroy playbooks"

All of the configuration from this blog post are in my [github repo](https://github.com/pharriso/ansible_molecule_vmware).

### Overview of the steps

Let's take a look at the high-level steps we are going to go through.

1) Install molecule
2) Initialise our first role with molecule
3) Configure molecule
4) Write playbooks to create and destroy test VM's.
5) Secure any secrets with Ansible Vault
6) Write our role
7) Test!


### Installing molecule

Molecule requires Ansible - in my case I installed ansible from an RPM. Molecule itself is installed with pip. I could install molecule in a python virtualenv but I won't cover that in this example. To install molecule run the following:

```bash
$ sudo yum install ansible -y
$ sudo pip3 install molecule
```

I want to do lint tests against my role as well to test for best practices so I will install ansible-lint as well:

```bash
$ sudo pip3 install ansible-lint
```

### Creating the directory structure for molecule

molecule can help us get started by creating the directory structure it expects to see as well as some of the initial configuration files. To create the scaffolding for a role called **register_vm** run the following:

**NOTE the use of the delegated driver here**

```bash
$ molecule init role register_vm --driver-name delegated
```

Let's take a look at what this created for us. We should see the standard role structure with an additional **molecule** directory. This includes the molecule.yml configuration file as well as the playbooks for creating, deleting and testing our role. More on these later.

```
$ tree
.
├── defaults
│   └── main.yml
├── files
├── handlers
│   └── main.yml
├── meta
│   └── main.yml
├── molecule
│   └── default
│       ├── converge.yml
│       ├── create.yml
│       ├── destroy.yml
│       ├── INSTALL.rst
│       ├── molecule.yml
│       ├── vault.yml
│       └── verify.yml
├── README.md
├── tasks
│   └── main.yml
├── templates
├── tests
│   ├── inventory
│   └── test.yml
└── vars
    └── main.yml
```

### Configuring Molecule

The main molecule configuration file is **molecule/default/molecule.yml** - this file describes things like the virtual instances that are provisioned, the test sequence that should be run and whether I want to do any linting. The file starts off looking like this:

```
---
dependency:
  name: galaxy
driver:
  name: delegated
platforms:
  - name: instance
provisioner:
  name: ansible
verifier:
  name: ansible
```

##### Configuring the driver

With the delegated driver it is my responsibility to create the playbooks to create and destroy test instances. Within the driver section I am just adding some platform specific variables that I will use later when I start to create instances. These are variables that apply to the vSphere platform itself rather than instance specific variables.

```
driver:
  name: delegated
  vcenter_datacenter: demolab
  vcenter_cluster: rhnode2
  vcenter_folder: /demolab/vm
```

##### Defining the test instances

The platforms section defines the test instances that I want to create. This section is just defining the list of VM's that I want to test against. These aren't generic molecule variables - they are specific to the playbook that I will write later on to create and delete test VM's. While the driver section had platform wide variables, these variables are specific to the instance. I could have multiple instances below with different configurations.

```
platforms:
  - name: molecule.demolab.local
    vm_rhel_version: rhel8
    vm_memory: 2048
    vm_cpu_cores: 2
    vm_network: infra
    vm_domain: demolab.local
    vm_user: root
    vm_port: 22    
    ssh_key_file: /root/.ssh/demolab-root
```

##### Configure the way molecule calls Ansible

The provisioner section specifies how Ansible should run when I create, destroy and converge (actually deploy my Ansible role to the VM). This includes options that I might want to pass to Ansible, and variables that my role might require. In my case, I want to pass encrypted VMware credentials so I need to pass in a file to decrypt my vault file. I also need to pass some variables into my role. I can do this by defining **group_vars** and **host_vars** as I would normally with Ansible.

**NOTE: These are the variables that our actual role needs - not what molecule needs to create and destroy our test instances.**

```
provisioner:
  name: ansible
  config_options:
    defaults:
      vault_password_file: ~/vault_password.txt
  inventory:
    group_vars:
      all:
        tower_user_name: pharriso
```

##### Add our linting (Optional)

ansible-lint is a useful tool for verifying the content of our Ansible playbooks and roles and validating against best practices. 

* Have I named every task? 
* Do I have white spaces? 
* Am I using the shell module? 

I can add a linting section to the **molecule.yml** file.

**NOTE: I am excluding the molecule directory itself. I only want to lint my Ansible role and not the playbooks that I am using to create and destroy instances. Also note that ansible-lint can ignore certain rules. Here I am excluding rule 305 which is the use of the shell module**


```
lint: |
  /usr/local/bin/ansible-lint -x 305 --exclude molecule/default/
```
##### The finished molecule.yml file

Here is the finished file:

```
---
dependency:
  name: galaxy
driver:
  name: delegated
  vcenter_datacenter: demolab
  vcenter_cluster: rhnode2
  vcenter_folder: /demolab/vm
platforms:
  - name: molecule.demolab.local
    vm_rhel_version: rhel8
    vm_memory: 2048
    vm_cpu_cores: 2
    vm_network: infra
    vm_domain: demolab.local
    vm_user: root
    vm_port: 22
    ssh_key_file: ~/.ssh/demolab-root
provisioner:
  name: ansible
  config_options:
    defaults:
      vault_password_file: ~/vault_password.txt
  inventory:
    group_vars:
      all:
        tower_user_name: pharriso
verifier:
  name: ansible
lint: |
  /usr/local/bin/ansible-lint -x 305 --exclude molecule/default/
```

### Encrypting secrets with Ansible Vault (Optional)

In my **molecule.yml** file I have defined a password file to decrypt an ansible vault file. This vault file will contain the credentials for my VMware environment as well as any other secrets I might need to protect. Let's create those files now.

First let's create the Ansible vault file that will be used to encrypt our secrets. When prompted I need to enter a password to decrypt the file. I will use **Redhat123** to encrypt my file here:

```bash
$ ansible-vault create molecule/default/vault.yml
New Vault password: 
Confirm New Vault password:
```

Here are the contents of the vault file - just a standard ansible-vault file with a bunch of variables:

```
vcenter_username: administrator@vsphere.local
vcenter_password: Redhat123
vcenter_hostname: vcenter.demolab.local
vcenter_validate_certs: no
sat_reg_user: admin
sat_reg_passwd: Redhat123
sat_server: sat6.demolab.local
```

Finally I can create a password file that molecule will read to decrypt the Ansible vault file. This is the file that I defined in my molecule.yml file as **vault_password_file** and the file should contain the plaintext password that I used earlier to encrypt my vault file - in my case **Redhat123**:

```bash
$ echo Redhat123 > ~/vault_password.txt
```

### Create, destroy and verify playbooks

##### Creating instances

Now that I have configured molecule, I can write the playbooks that will create and destroy my test instance as well as validate the role is achieving the desired end result. The create playbook is located at **molecule/default/create.yml** - when I created the molecule scaffolding at the start, it created these playbooks for me. I need to add two things to this file:

1) the specific tasks for my environment to allow me to create a VM. In the **create.yml** playbook there is a **TODO** section where these tasks will need to go.
2) as an output of the VM creation, I need to pass variables to molecule so that it can then connect to my test VM - things like IP address, username etc. Some of these things I might not know until my VM launches and I query the IP address. Others might be more predictable such as the port to connect on (22/tcp SSH for Linux).

I don't want to show the whole file here because we can look at that in my [git repo](https://github.com/pharriso/ansible_molecule_vmware). Instead I want to call out specific sections that I needed to update from the default example. I have updated the default playbook to include my Ansible vault file in the **vars_file** section. I've also added a task to create a VMware VM from a template. This is where the variables from my **molecule.yml** file come in to play. Things like the VMware template, memory and cpu count all come from that file. Remember, my secret vCenter credentials are all being passed in from my Ansible vault file.

```
---
- name: Create
  hosts: localhost
  connection: local
  gather_facts: false
  vars_files:
  - vault.yml
  no_log: "{{ molecule_no_log }}"
  tasks:

    - name: create vm from template
      vmware_guest:
        hostname: "{{ vcenter_hostname }}"
        username: "{{ vcenter_username }}"
        password: "{{ vcenter_password }}"
        validate_certs: "{{ vcenter_validate_certs }}"
        datacenter: "{{ molecule_yml.driver.vcenter_datacenter }}"
        name: "{{ item.name }}"
        cluster: "{{ molecule_yml.driver.vcenter_cluster }}"
        folder: "{{ molecule_yml.driver.vcenter_folder }}"
        template: "{{ item.vm_rhel_version }}_template"
        state: poweredon
        hardware:
          memory_mb: "{{ item.vm_memory }}"
          num_cpus: "{{ item.vm_cpu_cores }}"
          scsi: paravirtual
        networks:
        - name: "{{ item.vm_network }}"
          type: dhcp
        customization:
          hostname: "{{ item.name.split('.')[0] }}"
          domain: "{{ item.vm_domain }}"
        wait_for_ip_address: yes
        wait_for_customization: yes
      register: server
      delegate_to: localhost
      loop: "{{ molecule_yml.platforms }}"
```
Below this section we need to populate the variables that will be used to build the instance configuration file. As mentioned before, the instance configuration file is used by molecule to actually connect into my test instance so that it can run my role against it.

Here I am taking variables from my **molecule.yml** file and also variables that came back from the VM creation task (ipv4) to build the instance configuration file. These variables will vary depending on target platform and the way in which you provision VM's.

```
    - when: server.changed | default(false) | bool
      block:
        - name: Populate instance config dict
          set_fact:
            instance_conf_dict: {
              'instance': "{{ item.item.name }}",
              'address': "{{ item.instance.ipv4 }}",
              'user': "{{ item.item.vm_user }}",
              'port': "{{ item.item.vm_port }}",
              'identity_file': "{{ item.item.ssh_key_file }}", }
          with_items: "{{ server.results }}"
          register: instance_config_dict
```

##### Destroying instances

I need to follow the same process for destroying test instances. The only difference is that I don't need to worry about instance configuration data on delete. The default playbook is called **molecule/default/destroy.yml** 

Let's just focus on the changes again - I need to include tasks for deleting a VM. Again, notice how I am relying on variables from **molecule.yml** and my Ansible vault file as I delete my VM. I'm also adding a second task to clear down the VM from Red Hat Satellite.

```
    - name: remove vm from vmware
      vmware_guest:
        hostname: "{{ vcenter_hostname }}"
        username: "{{ vcenter_username }}"
        password: "{{ vcenter_password }}"
        validate_certs: "{{ vcenter_validate_certs }}"
        name: "{{ item.name }}"
        state: absent
        force: yes
      delegate_to: localhost
      loop: "{{ molecule_yml.platforms }}"

    - name: delete host in satellite
      redhat.satellite.host:
        username: "{{ sat_reg_user }}"
        password: "{{ sat_reg_passwd }}"
        server_url: "https://{{ sat_server }}"
        validate_certs: no
        name: "{{ item.name }}"
        state: absent
      delegate_to: localhost
      register: satremoved
      retries: 1
      delay: 3
      until: satremoved is success
      loop: "{{ molecule_yml.platforms }}"
```

##### Verifying the result of my role

The final playbook that I need to create is a one that will verify if my role is actually achieving the desired results. The **molecule/default/verify.yml** playbook can be adapted to perform this validation. In my case I am just going to confirm that the output of subscription-manager returns an exit code of 0.

```
---
# This is an example playbook to execute Ansible tests.

- name: Verify
  hosts: all
  gather_facts: false
  tasks:
  - name: run subscription-manager status
    shell: subscription-manager status
    register: subscription_status

  - name: check if subscription manager status returned exit code 0
    assert:
      that: subscription_status.rc == 0
```

### Now to do some real work - developing your role

To create and test my role I am going to run through the following molecule stages:

1) **create** - launch my test instance. 
2) **lint** - lint my role to check for best practices.
3) **converge** - run my role against the test instance.
4) **idempotence** - check if my role is idempotent.
5) **verify** - check the end result of my role.
6) **test** - run a full test suite by destroying the test instance, re-create it and run my tests again.


##### **create** - Launch a test instance

So far I have just been preparing my molecule environment. I haven't actually started doing the thing that started this whole process - writing an Ansible role! I can use molecule to start a test instance so that I can begin developing my role. From the root of my role directory I can use **molecule create** to launch a test instance:

```bash 
$ molecule create
```

If the playbook succeeded then I should have a test instance running in VMware. I can confirm this with **molecule list** and also try to connect into the instance with **molecule login**

```bash
$ molecule list
DEBUG    Validating schema /home/pharriso/register_vm/molecule/default/molecule.yml.
INFO     Running default > list
                         ╷             ╷                  ╷               ╷         ╷            
  Instance Name          │ Driver Name │ Provisioner Name │ Scenario Name │ Created │ Converged  
╶────────────────────────┼─────────────┼──────────────────┼───────────────┼─────────┼───────────╴
  molecule.demolab.local │ delegated   │ ansible          │ default       │ true    │ false      
                         ╵             ╵                  ╵               ╵         ╵     
```

```bash
$ molecule login
DEBUG    Validating schema /home/pharriso/register_vm/molecule/default/molecule.yml.
INFO     Running default > login
Last login: Mon Dec 14 12:26:23 2020 from 10.50.2.254
[root@molecule ~]# 
```

##### Explore some of the molecule configuration data

If you remember our **create.yml** playbook, I had to define variables so that molecule could build instance configuration data - the means with which to actually connect into to my test instance to run my Ansible role. I can look at that instance configuration:

```bash
$ cat .cache/molecule/register_vm/default/instance_config.yml 
# Molecule managed

- {address: 10.50.2.73, identity_file: ~/.ssh/demolab-root, instance: molecule.demolab.local,
  port: '22', user: root}
```

I can also look at the inventory file that molecule has generated for me based on this data:

```bash
$ cat ~/.cache/molecule/register_vm/default/inventory/ansible_inventory.yml 
# Molecule managed

---
all:
  hosts:
    molecule.demolab.local: &id001
      ansible_host: 10.50.2.73
      ansible_port: '22'
      ansible_private_key_file: ~/.ssh/demolab-root
      ansible_ssh_common_args: -o UserKnownHostsFile=/dev/null -o ControlMaster=auto
        -o ControlPersist=60s -o ForwardX11=no -o LogLevel=ERROR -o IdentitiesOnly=yes
        -o StrictHostKeyChecking=no
      ansible_user: root
```

##### Write our role

This part doesn't really require any explaining. Write your Ansible role as you would normally. Again, [this git repo](https://github.com/pharriso/ansible_molecule_vmware) is available to view.


##### **lint** - check for best practices

Now I am ready to test my role. Firstly, I can run lint against it to check for best practices. Any errors will be reported to the terminal. e.g.

```bash
$ molecule lint

DEBUG    Using selector: EpollSelector
WARNING  Listing 1 violation(s) that are fatal
You can skip specific rules or tags by adding them to your configuration file:

┌──────────────────────────────────────────────────────────────────────────────┐
│ # .ansible-lint                                                              │
│ warn_list:  # or 'skip_list' to silence them completely                      │
│   - '201'  # Trailing whitespace                                             │
└──────────────────────────────────────────────────────────────────────────────┘
201 Trailing whitespace
tasks/main.yml:4
  include_vars:


```

##### **converge** - run our role

Next I can run converge - this actually runs my role within my test instance. You'll see the familiar ansible-playbook output here where you can validate that your role executes without errors.

```bash
$ molecule converge

PLAY [Converge] ******************************************************************************************************************************************************************************

TASK [Gathering Facts] ***********************************************************************************************************************************************************************
ok: [molecule.demolab.local]

TASK [Include register_vm] *******************************************************************************************************************************************************************

TASK [register_vm : load vars for os type] ***************************************************************************************************************************************************
ok: [molecule.demolab.local]

TASK [register_vm : install satellite katello certificate] ***********************************************************************************************************************************
changed: [molecule.demolab.local]

TASK [register_vm : register to satellite] ***************************************************************************************************************************************************
changed: [molecule.demolab.local]

TASK [register_vm : enable required repos] ***************************************************************************************************************************************************
changed: [molecule.demolab.local]

TASK [register_vm : install katello host tools] **********************************************************************************************************************************************
changed: [molecule.demolab.local]

PLAY RECAP ***********************************************************************************************************************************************************************************
molecule.demolab.local     : ok=6    changed=4    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0
```

##### **idempotence** - is our role idempotent?

The idempotence molecule test will validate whether my role can be re-applied without making changes. Let's skip the majority of the output below and just look at the summary:

```bash
$ molecule idempotence

PLAY RECAP ***********************************************************************************************************************************************************************************
molecule.demolab.local     : ok=6    changed=0    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0

INFO     Idempotence completed successfully.
```

##### **verify** - did my role do what it should?

The verify stage will run my **verify.yml** playbook to ensure that my role achieved he desired result by running the tests that I defined.

```bash
$ molecule verify

PLAY [Verify] ********************************************************************************************************************************************************************************

TASK [run subscription-manager status] *******************************************************************************************************************************************************
changed: [molecule.demolab.local]

TASK [check if subscription manager status returned exit code 0] *****************************************************************************************************************************
ok: [molecule.demolab.local] => {
    "changed": false,
    "msg": "All assertions passed"
}

PLAY RECAP ***********************************************************************************************************************************************************************************
molecule.demolab.local     : ok=2    changed=1    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0
```

### Running a full test

So far I have been running individual tests as I develop my role. Once I am happy with my role I can initiate a full molecule test. As mentioned above, this will destroy any existing test instances, re-create them, run lint checks, run my Ansible role, check they are idempotent and then destroy the test VM's.

```bash
$ molecule test
```

### Next Steps

Molecule provides a really powerful and flexible way of developing and testing Ansible roles. This helps me to ensure that my Ansible roles are suitably tested before I start to commit them to git. I will still want to run CI tests as well as running molecule - my CI test could end up just running molecule again but it means that when my colleagues peer reviews my work and look at my merge request they can look at the CI test results.

Molecule is a community project and isn't supported by Red Hat at the time of writing.

* Learn more by reading the [official documentation for molecule](https://molecule.readthedocs.io/en/latest/).
* You can take a look at all of the configuration files used in this post in my [github repo](https://github.com/pharriso/ansible_molecule_vmware).
