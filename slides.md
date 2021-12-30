---
marp: true
---

<!-- _class: invert -->

## Cloud Infrastructure (IaaS) Provisioning

* Provisioning servers or container hosts requires software installation and
  configuration.

* There are 2 approaches to provision such instances

  1. by installing and configuring softwares when the instance launches or

  2. by creating and uploading a golden image with a corresponding tool.

* The first can be performed when an instance comes up and it might be flexible
  for some cases to get the latest packages and the second would be more safe
  for preparing fixed and consistent images when using them for load balancers
  or container hosts because of consistency and idempotence.

* Cloud-init (cloud config) can be used for the first approach.

---

## Cloud-Init

* Cloud-init is the industry standard multi-distribution method for
  cross-platform cloud instance initialization. It is supported across all major
  public cloud providers, provisioning systems for private cloud infrastructure,
  and bare-metal installations.

* Cloud instances are initialized from a disk image and instance data:

  * Cloud metadata
  * User data (optional)
  * Vendor data (optional)

---

## Cloud-Init (II)

* Cloud-init will identify the cloud it is running on during boot, read any
  provided metadata from the cloud and initialize the system accordingly. This
  may involve setting up the network and storage devices to configuring SSH
  access key and many other aspects of a system. Later on the cloud-init will
  also parse and process any optional user or vendor data that was passed to the
  instance.

---

## Cloud-Init (III)

* Cloud-init provides support across a wide list of public clouds:

  * Amazon Web Services
  * Microsoft Azure
  * Google Cloud Platform
  * and many more

* Additionally, cloud-init is supported on these private clouds:

  * Bare metal installs
  * OpenStack
  * Metal-as-a-Service (MAAS)
  * VMware
  * and others

---

## Cloud-Init Boot Stages

* Cloud-init is integrated into the boot in controlled way.

* There are five stages:

  * Generator
  * Local
  * Network
  * Config
  * Final

---

## Cloud-Init Boot Stages (II)

* When booting under systemd, a **generator** will run that determines if
  cloud-init.target should be included in the boot goals.

* The purpose of the **local** stage is to locate “local” data sources and to
  apply networking configuration to the system (including “Fallback”).

* The **network** stage requires all configured networking to be online, as it
  will fully process any user-data that is found.

* The **config** stage runs config modules only. Modules that do not really have
  an effect on other stages of boot are run here, including runcmd.

* The **final** stage runs as late in boot as possible. Any scripts that a user
  is accustomed to running after logging into a system run correctly here.

---

## Cloud-Init First Boot Determination

* Cloud-init has to determine whether or not the current boot is the first boot
  of a new instance or not, so that it applies the appropriate configuration. On
  an instance’s first boot, it should run all “per-instance” configuration,
  whereas on a subsequent boot it should run only “per-boot” configuration. This
  section describes how cloud-init performs this determination, as well as why
  it is necessary.

* When it runs, cloud-init stores a cache of its internal state for use across
  stages and boots.

---

## Cloud-Init Implementation in AWS

* In AWS we can put our configurations in **user data** of an instance. Your are
  allowed to use both shell script and cloud-init directives in user-data field.

* We can pass user data to an instance as **plain text**, as a file (this is
  useful for launching instances using the command line tools, or as
  base64-encoded text (for API calls).

* If you’re familier with shell-scripting, you can use **shell script** starting
  with #! characters and put the path to the interpreter you want to read the
  script (commonly /bin/bash) in user data. This is simpler way to give
  instructions for an instance.

* If you give characters **#cloud-config**, an instance recognizes this is
  cloud-init instructions and you can use cloud-init directives in user data
  field. You can check logs in */var/log/cloud-init-output.log*.

---

<!-- _class: invert -->

## Cloud-Config Example

```
#cloud-config
package_update: true

packages:
 - containerd

runcmd:
- sudo mkdir -p /opt/cni/bin
- curl -sSL https://github.com/containernetworking/plugins/releases/download/v1.0.1/cni-plugins-linux-amd64-v1.0.1.tgz | sudo tar -xz -C /opt/cni/bin
- systemctl daemon-reload && systemctl start containerd
- systemctl enable containerd
- curl -sLSf https://github.com/containerd/nerdctl/releases/download/v0.14.0/nerdctl-0.14.0-linux-amd64.tar.gz | tar Cxzvvf /usr/local/bin -
```

* This one
  * updates packages,
  * installs CNI (Container Network Interface) Plugins,
  * (containerd is already installed) 
  * starts and enables containerd
  * and installs nerdctl (instead of *docker run* just use *nerdctl*)
  
---

<!-- _class: invert -->

## Running an AWS Instance with Cloud-Config

```
aws ec2 run-instances \
    <<< ... lines intentionally omitted ... >>>
    --user-data file://cloud-config.txt \
    <<< ... lines intentionally omitted ... >>>
```

---

## Cloud-Init Implementation in AWS

* Cloud-init in AWS consists of 4 services in Amazon Linux 2. These 4 services
  start cloud-init software and take user data given from AWS to install
  softwares and configuring softwares when an EC2 instance is launched.

* User data scripts and cloud-init directives run only during the boot cycle
  when you first launch an instance. You can update your configuration to ensure
  that your user data scripts and cloud-init directives run every time you
  restart your instance. Here’s the how-to link of the official documentation.

```
$ systemctl list-unit-files --type service| grep ^cloud
cloud-config.service                          enabled
cloud-final.service                           enabled
cloud-init-local.service                      enabled
cloud-init.service                            enabled
```

---

## Cloud-Init Directives

* There are a lot of directives in cloud-init and you can find the examples in the official documentation. 

* AWS cloud-init might not support all directives of cloud-init because it is customized and altered for AWS.

### Repository Update and Upgrade Packages

* repo_update is used to resynchronize the package index

* repo_upgrade is used to install the newest versions of all packages

```
#cloud-config
repo_update: true
repo_upgrade: all
```

---

### Install Packages

```
#cloud-config

# [<package>, <version>] wherein the specifc package version will be installed.
packages:
 - httpd
 - mariadb-server
 - [libpython2.7, 2.7.3-0ubuntu3.1]
```

---

### Run Commands on Boot

* There are 2 directives to run commands on instance’s boot stage.

  * **bootcmd** directive will run commands on every boot under cloud-init.service

  * **runcmd** directive runs commands only during the first boot in cloud-config.service

```
#cloud-config
bootcmd:
  - echo 192.168.1.130 us.archive.ubuntu.com >> /etc/hosts

runcmd:
 - systemctl start httpd
 - sudo systemctl enable httpd
```

---

### Writing Files

```
#cloud-config
write_files:
 - content: |
      #!/bin/sh
      version=$(rpm -q --qf '%{version}' system-release)
      cat << EOF
     _                   
    | |                  
  __| |_____ ____   ___  
 / _  | ___ |    \ / _ \ 
( (_| | ____| | | | |_| |
 \____|_____)_|_|_|\___/ 
      https://aws.amazon.com/amazon-linux-ami/$version-release-notes/
      EOF
   owner: root:root
   path: /etc/update-motd.d/30-banner
   permissions: '0766'
```

---

### Configure SSH Keys

* There are 2 directives to configure ssh keys in an instance.

  * ssh_authorized_keys directive adds keys in *.ssh/authorized_keys* file.

  * ssh_keys directive can set pre-generated ssh private key.

```
#cloud-config

ssh_authorized_keys:
  - ssh-rsa <ssh-rsa key> anonymous@hacked.com

ssh_keys:
  rsa_private: |
    -----BEGIN RSA PRIVATE KEY-----
    <<< ... lines intentionally omitted ... >>>
    -----END RSA PRIVATE KEY-----

  rsa_public: ssh-rsa <public key> anonymous@localhost
```

---

## Demo Time

<!-- _class: invert -->

* In the following demo

  * we use cloud-config to initialize an AL2 instance with containerd,

  * login via ssh, and

  * run a container via *nerdctl*, its the whalesay one with "cowsay boo".