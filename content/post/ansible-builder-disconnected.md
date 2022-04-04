---
title: "ansible-builder in a disconnected environment"
date: 2022-04-01T21:07:28+01:00
draft: false
tags: [ansible, ansible-builder, execution environments]
---

[Ansible Automation Platform 2](https://www.ansible.com/blog/introducing-ansible-automation-platform-2) introduced execution environments as a replacement for python virtual environments. [Execution environments](https://docs.ansible.com/automation-controller/latest/html/userguide/execution_environments.html) are container images which contain everything needed to execute an Ansible playbook:

* A version of ansible-core
* A version of Python
* Python modules/dependencies
* Ansible collections (optional)

The move to containers is really about solving the problem of how we package and distribute everything needed to run a playbook so that it runs consistently wherver we run it - a laptop, a RHEL server or [automation controller](https://docs.ansible.com/automation-controller/latest/html/userguide/overview.html).

Red Hat provide a number of pre-built execution environments to make it easier for users to get started. Often the pre-built images work great and they include a number of sensible python dependencies for typical automation use-cases. However, there will be times when we will need to customise execution environments. This is typically when using additional Ansible modules which have additional python dependencies.

[ansible-builder](https://www.ansible.com/blog/introduction-to-ansible-builder) was created to aid with the customisation and creation of execution environments. The idea of ansible-builder is to provide a layer of abstraction between the Ansible user and the build of an execution environment container - removing the need to get our hands too dirty with docker/podman commands.

In true Ansible fashion, the definition of an execution environment is described in YAML. The ansible-builder command line tool then takes that simple definition and creates an execution environment. As with most things, in a connected environment the process works pretty seamlessly. If working in a disconnected environment it does require some additional configuration.

Let's walk through an example!

### Execution environmnent definition

**NOTE** This post doesn't cover the synchronisation of container images into a disconnected network. In our case we have already synced the necessary images and uploaded them into [private automation hub](https://www.ansible.com/blog/whats-new-in-ansible-automation-platform-2-private-automation-hub).

Here is a really simple execution environment definition which requires the dnspython module. The contents of execution-environment.yml:

```bash
# cat execution-environment.yml 
---
version: 1
build_arg_defaults:
  EE_BASE_IMAGE: 'automation-hub.demolab.local/ansible-automation-platform-21/ee-supported-rhel8:latest'
  EE_BUILDER_IMAGE: 'automation-hub.demolab.local/ansible-automation-platform-21/ansible-builder-rhel8:latest'

dependencies:
  python: requirements.txt
```

And the contents of the requirements.txt file:

```bash
# cat requirements.txt 
dnspython==1.15.0
```

### Problem #1 - external yum repositories

Before looking at yum dependencies, a quick note on how yum repositores are handled in the context of execution environments. Firstly, Red Hat provide execution environments which are based on a [RHEL 8 universal base image](https://www.redhat.com/en/blog/introducing-red-hat-universal-base-image). The ubi8 (universal base image RHEL8) container is configured to use a special yum repository which allows rpms to be installed without needing to attach subscriptions. 

If you don't have internet access and are building containers on a RHEL system which has been registered to Red Hat's content delivery network or Red Hat Satellite using subscription-manager, then the ubi8 container will inherit the subscription status of the host and get access to the same repositories as the underlying host.

If you are building containers on a RHEL system which isn't using subscription-manager (e.g. using a local repository or something like Artifactory) then the ubi8 container doesn't inherit the repositories from the underlying host. This is important to understand when considering container builds in a disconnected environment.


OK, let's try building an execution environment with ansible-builder in a disconnected environment.


```bash
$ ansible-builder build -t disconnected_ee:1.0
Running command:
  podman build -f context/Containerfile -t disconnected_ee:1.0 context
...showing last 20 lines of output...
+++ REDHAT_SUPPORT_PRODUCT_VERSION=8.5
++ echo rhel
+ RELEASE=rhel
+ PKGMGR=
+ PKGMGR_OPTS=
+ '[' -z ']'
+ PKGMGR=/usr/bin/dnf
+ '[' -f /usr/bin/microdnf ']'
+ PKGMGR=/usr/bin/microdnf
+ '[' -z ']'
+ PKGMGR_OPTS='--nodocs --setopt install_weak_deps=0'
+ mkdir -p /output/bindep
+ mkdir -p /output/wheels
+ mkdir -p /tmp/src
+ cd /tmp/src
+ /usr/bin/microdnf update -y
Downloading metadata...
error: cannot update repo 'ubi-8-baseos': Cannot download repomd.xml: Cannot download repodata/repomd.xml: All mirrors were tried; Last error: Curl error (7): Couldn't connect to server for https://cdn-ubi.redhat.com/content/public/ubi/dist/ubi8/8/x86_64/baseos/os/repodata/repomd.xml []
[3/3] STEP 1/4: FROM automation-hub.demolab.local/ansible-automation-platform-21/ee-supported-rhel8:latest
Error: error building at STEP "RUN assemble": error while running runtime: exit status 1
```

Note how the build fails because the base image is trying to connect to an external yum repository hosted on redhat.com "Couldn't connect to server for **https://cdn-ubi.redhat.com**".

To get around this issue, we are going to modify the Containerfile. Re-running ansible-builder with the **create** argument will generate the Containerfile but not actually build the resulting container image:

```bash
$ ansible-builder create
Complete! The build context can be found at: /root/disconnected_ee/context
```

Now we can edit the Containerfile and remove the external yum repo that the base image is trying to use:

```bash
$ cat context/Containerfile 
ARG EE_BASE_IMAGE=automation-hub.demolab.local/ansible-automation-platform-21/ee-supported-rhel8:latest
ARG EE_BUILDER_IMAGE=automation-hub.demolab.local/ansible-automation-platform-21/ansible-builder-rhel8:latest

FROM $EE_BASE_IMAGE as galaxy
ARG ANSIBLE_GALAXY_CLI_COLLECTION_OPTS=
USER root

ADD _build /build
WORKDIR /build


FROM $EE_BUILDER_IMAGE as builder
ADD _build/requirements.txt requirements.txt
RUN ansible-builder introspect --sanitize --user-pip=requirements.txt --write-bindep=/tmp/src/bindep.txt --write-pip=/tmp/src/requirements.txt
# Remove ubi repo
RUN rm -f /etc/yum.repos.d/ubi.repo
RUN assemble

FROM $EE_BASE_IMAGE
USER root
COPY --from=builder /output/ /output/
# Remove ubi repo
RUN rm -f /etc/yum.repos.d/ubi.repo
RUN /output/install-from-bindep && rm -rf /output/wheels
```

**Note** the steps to remove /etc/yum.repos.d/ubi.repo. If we needed to access an internal repository on a system which isn't registered with subscription-manager then we'd need to copy our own yum repo file in at this point.

We can now try to build the execution environment using our modified Containerfile. We will need to use a podman command to do this now that we have modified the Containerfile.

```bash
$ podman build -f context/Containerfile -t disconnected_ee:1.0

<output truncated>

+ /usr/bin/microdnf update -y
Downloading metadata...
Downloading metadata...
Package                                          Repository                        Size
Installing:                                                                            
 openssl-1:1.1.1k-6.el8_5.x86_64                 rhel-8-for-x86_64-baseos-rpms 725.9 kB
 openssl-pkcs11-0.4.10-2.el8.x86_64              rhel-8-for-x86_64-baseos-rpms  67.5 kB
Upgrading:                                                                             
 openssl-libs-1:1.1.1k-6.el8_5.x86_64            rhel-8-for-x86_64-baseos-rpms   1.5 MB
  replacing openssl-libs-1:1.1.1k-5.el8_5.x86_64                                       
 tzdata-2022a-1.el8.noarch                       rhel-8-for-x86_64-baseos-rpms 485.3 kB
   replacing tzdata-2021e-1.el8.noarch                                                 


<output truncated>

WARNING: Retrying (Retry(total=4, connect=None, read=None, redirect=None, status=None)) after connection broken by 'NewConnectionError('<pip._vendor.urllib3.connection.HTTPSConnection object at 0x7f11d8278460>: Failed to establish a new connection: [Errno 101] Network is unreachable')': /simple/dnspython/
```

The build gets further now as we are able to complete the yum transaction. Now we hit the next problem - accessing python dependencies.

### Problem #2 - python dependencies

As we have just discovered with rpms, if we are going to try to install additional python dependencies we will also need an internal repository. By default the execution enviroment build is going to try to reach pypi.org to satisfy our dnspython dependency. To get around this issue we are going to copy a pip configuration file into the execution environment which will instruct it to use a local mirror. In this example we are using [Sonatype Nexus](https://www.sonatype.com/products/repository-oss) and a pypi proxy repository.

Create a pip.conf which points to the local mirror in the **context** directory adjacent to the **Containerfile**.


```bash
$ cat context/pip.conf 
[global]
index-url = https://nexus-nexus.apps.celeron.demolab.local/repository/pypi-proxy/simple/
```

Now edit the **context/Containerfile** to copy the **pip.conf** into place:


```bash
$ cat context/Containerfile
[root@disconnected context]# cat Containerfile 
ARG EE_BASE_IMAGE=automation-hub.demolab.local/ansible-automation-platform-21/ee-supported-rhel8:latest
ARG EE_BUILDER_IMAGE=automation-hub.demolab.local/ansible-automation-platform-21/ansible-builder-rhel8:latest

FROM $EE_BASE_IMAGE as galaxy
ARG ANSIBLE_GALAXY_CLI_COLLECTION_OPTS=
USER root

ADD _build /build
WORKDIR /build


FROM $EE_BUILDER_IMAGE as builder
ADD _build/requirements.txt requirements.txt
RUN ansible-builder introspect --sanitize --user-pip=requirements.txt --write-bindep=/tmp/src/bindep.txt --write-pip=/tmp/src/requirements.txt
# Remove ubi repo
RUN rm -f /etc/yum.repos.d/ubi.repo
# Add pip.conf for internal pypi proxy
ADD pip.conf /etc/pip.conf
RUN assemble

FROM $EE_BASE_IMAGE
USER root
COPY --from=builder /output/ /output/
# Remove ubi repo
RUN rm -f /etc/yum.repos.d/ubi.repo
# Add pip.conf for internal pypi proxy
ADD pip.conf /etc/pip.conf
RUN /output/install-from-bindep && rm -rf /output/wheels
```

**Note** the new entries to copy pip.conf from the context directory to /etc/pip.conf in the base image and builder image.

### Problem #3 - (Optional) Internal certificates

Accessing an internal pypi mirror might require some certificates to be deployed to the execution environments. We are marking this section as optional because there are different scenarios and different ways of getting around problems. For example, an internal repository might be using self-signed certificates or you could bypass certificate checking in pip.conf. In our example, our nexus repository is using an internal certificate authority and we want to configure our execution environment to use this certificate authority. Here is an example of the kind of error we might see from ansible-builder if we can't verify the certificate on our Nexus repository.


```bash
ERROR: Could not find a version that satisfies the requirement dnspython==1.15.0 (from -r /tmp/src/requirements.txt (line 1)) (from versions: none)
ERROR: No matching distribution found for dnspython==1.15.0 (from -r /tmp/src/requirements.txt (line 1))
Could not fetch URL https://nexus-nexus.apps.celeron.demolab.local/repository/pypi-proxy/simple/pip/: There was a problem confirming the ssl certificate: HTTPSConnectionPool(host='nexus-nexus.apps.celeron.demolab.local', port=443): Max retries exceeded with url: /repository/pypi-proxy/simple/pip/ (Caused by SSLError(SSLCertVerificationError(1, '[SSL: CERTIFICATE_VERIFY_FAILED] certificate verify failed: unable to get local issuer certificate (_ssl.c:1125)'))) - skipping
```

To fix this, let's copy the relevant CA certificate into the container images as part of the build process. Copy the relevant CA certificate file into the **context** directory adjacent to the **Containerfile**.

Finally, edit the Containerfile to copy the CA certificate file into place. Here is our final Containerfile:


```bash
$ cat Containerfile 
ARG EE_BASE_IMAGE=automation-hub.demolab.local/ansible-automation-platform-21/ee-supported-rhel8:latest
ARG EE_BUILDER_IMAGE=automation-hub.demolab.local/ansible-automation-platform-21/ansible-builder-rhel8:latest

FROM $EE_BASE_IMAGE as galaxy
ARG ANSIBLE_GALAXY_CLI_COLLECTION_OPTS=
USER root

ADD _build /build
WORKDIR /build


FROM $EE_BUILDER_IMAGE as builder
ADD _build/requirements.txt requirements.txt
RUN ansible-builder introspect --sanitize --user-pip=requirements.txt --write-bindep=/tmp/src/bindep.txt --write-pip=/tmp/src/requirements.txt
# Remove ubi repo
RUN rm -f /etc/yum.repos.d/ubi.repo
# Add pip.conf for internal pypi proxy
ADD pip.conf /etc/pip.conf
# Add CA certificate and update trust
ADD demolab-ca.crt /etc/pki/ca-trust/source/anchors/demolab-ca.crt
RUN update-ca-trust
RUN assemble

FROM $EE_BASE_IMAGE
USER root
COPY --from=builder /output/ /output/
# Remove ubi repo
RUN rm -f /etc/yum.repos.d/ubi.repo
# Add pip.conf for internal pypi proxy
ADD pip.conf /etc/pip.conf
# Add CA certificate and update trust
ADD demolab-ca.crt /etc/pki/ca-trust/source/anchors/demolab-ca.crt
RUN update-ca-trust
RUN /output/install-from-bindep && rm -rf /output/wheels
```

**NOTE** the new entries to copy the CA certificate into place and update trust.


### Finally - let's build!

We are ready to run our build again!


```bash
$ podman build -f context/Containerfile -t disconnected_ee:1.0
```

If the build worked we should see a message like this:


```bash
--> 2316db485a1
Successfully tagged localhost/disconnected_ee:1.0
2316db485a1c4e7be4a687c682d0fc90335372d7e5564774f1ff6451840ac35f
```

We can quickly check that the dnspython python requirement was added to the execution environment:


```bash
$ podman run --rm -it localhost/disconnected_ee:1.0 pip3 freeze | grep dnspython
dnspython==1.15.0
```

### Next Steps

As with most things in the world of IT, working in a disconnected environment provides us with additional challenges. Hopefully this post gave some ideas on how to use ansible-builder in a disconnected environment to build custom execution environments.

* Learn more about ansible-builder - https://ansible-builder.readthedocs.io/en/stable/
* Learn more about Red Hat Ansible Automation Platform - https://www.redhat.com/en/technologies/management/ansible
