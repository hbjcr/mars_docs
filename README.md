# [MARS (Multiversion Asynchronous Replicated Storage)](http://schoebel.github.io/mars/) Installation Tutorial

## Overview

This is an effort to document [MARS'](https://github.com/schoebel/mars) installation process using Ubuntu.

## Introduction

You can find information about the installation process in at least 3 places:

1. [MARS Manual](https://github.com/schoebel/mars/blob/master/docu/mars-manual.pdf)
2. [MARS Installation Documentation](https://github.com/schoebel/mars/blob/master/INSTALL)
3. [MARS pre-patching folder](https://github.com/schoebel/mars/tree/master/pre-patches)

This document merges all this information plus additional research that was required to make MARS work using Ubuntu 14.04. Future documentation will also include details on how to perform the installation process on Ubuntu 16.04.

## Pre-requisites

1. Read the manual. Seriously.
2. MARS only runs on a 64-bit compatible OS. I highly recommend doing this tutorial using Ubuntu 14.04 the first time because that will increase your chances of success. After getting MARS up and running you can then try a newer OS version or even a different OS when you understand the basics.
3. You will need a host to recompile the kernel, this tutorial assumes your host already has all the utilities required to perform this task. Moreover, the kernel will be built using [the old fashioned way](https://help.ubuntu.com/community/Kernel/Compile#Alternate_Build_Method:_The_Old-Fashioned_Debian_Way) method.
4. You will need an extra partition to host the data that will be replicated by MARS. This means at the very minimum, you will need the default partition to host your OS and an additional partition to host your MARS data.

## Kernel compilation

**1. Determine which kernel version you are going to use and install it**

MARS requires patching the kernel sources prior to compilation, these patches are [available for several versions](https://github.com/schoebel/mars/tree/master/pre-patches) of the kernel but not all of them. Ubuntu 14.04 in particular comes by default with a kernel version (3.13) that is not listed among this list which is why it is highly recommended to patch the OS with a supported version prior to performing any other action.

To install a new kernel version in Ubuntu, you need to execute the following command:

```
sudo apt-get install linux-generic-lts-[kernel first name]
```

An updated list of Ubuntu Kernel names is available [here](https://en.wikipedia.org/wiki/Ubuntu_version_history#Table_of_versions). Just look up the kernel version you want to use and replace [kernel first name] with the actual name you want. For instance, say you want to install kernel "xenial xerus" (4.4), then the command you will have to execute would be:

```
sudo apt-get install linux-generic-lts-xenial
```

Then make sure you **reboot your server** to ensure your new kernel is now being used.

**2. Download your kernel's source code for recompilation**

You are going to need a working folder, this guide will assume your working directory is going to be __/home/myuser__ from now on.

Create a new folder under /home/myuser named "kernel" and from that folder run the following command:

```
apt-get source linux-image-$(uname -r)
```

This command will download the source code for the kernel that's currently running on your host (thus the importance of performing a reboot prior to executing this step).

If everything goes well, you will end up with a structure similar to this:

```
/home/myuser/kernel/linux-lts[kernel version]
```

**3. Patch your kernel with MARS' pre-patch files**

Clone MARS' repo into your ```/home/myuser/kernel/linux-lts[kernel version]/block``` folder (make sure you perform the clone from the right folder, you don't want to clone your files on the wrong folder)

```
git clone https://github.com/schoebel/mars
```

And now you need to patch your kernel source with the right files. This action needs to be performed from the ```/home/myuser/kernel/linux-lts[kernel version]``` folder

```
patch -p1 < /home/myuser/kernel/linux-[kernel version]/block/mars/pre-patches/vanilla-[kernel version]/[patch name].patch
```

Just make sure you apply all the patches listed there.

**4. Build your new kernel packages**

First, you are going to enable MARS prior to performing the compilation, just execute the following command:

```
make oldconfig
```

Just make sure that when you get this prompt "storage system MARS (EXPERIMENTAL) (MARS) [N/m/y/?] (NEW)" you type **m**. You can simply press ENTER on each prompt to keep the default values.

Now you are ready to actually compile your new kernel and generate the .deb files you need to patch your existing kernel:

```
make deb-pkg
```

At the end of this process and if everything goes well, you will see some new .deb files under ```/home/myuser/kernel```.

**5. Copy your new kernel into all your MARS hosts**

Create a new folder into all of your MARS hosts

```
mkdir -p /home/myuser/kernel
```

Copy the following files into your newly created folders on all of your MARS hosts

```
/home/myuser/kernel/linux-headers-[kernel version]_amd64.deb
/home/myuser/kernel/linux-image-[kernel version]_amd64
/home/myuser/kernel/linux-[kernel version]/block/mars/userspace/marsadm
```

## Prepare MARS server nodes

**1. Log into all of your MARS host servers and install your new kernel on each of them**

```
dpkg -i /home/myuser/kernel/*.deb
```

Make sure you **reboot all of your MARS hosts after completing the kernel update**.

**2. Install the marsadm utility**

```
cp /home/myuser/kernel/marsadm /usr/local/bin && chmod +x /usr/local/bin/marsadm
```

**3. Enable SSH root login**

You will have to select one of your MARS hosts as your "Node A" (the first node on your cluster). Your other MARS hosts will require remote SSH access using their local root account.

The first step is to enable SSH root login on your Node A, you need to manually open the SSHD configuration file ```/etc/ssh/sshd_config``` within your Node A and change these lines

```
FROM:
PermitRootLogin prohibit-password
PermitRootLogin no
TO:
PermitRootLogin yes
PermitRootLogin yes
```

Then reload the configuration file to apply your changes

```
service ssh reload
```

**4. SSH Login Without Password**

Make sure you execute these commands using your **root** account. You will have to repeat these instructions in all of your MARS hosts except from your Node A. Execute the following command and press ENTER on each prompt

```
ssh-keygen
```
```
Generating public/private rsa key pair.
Enter file in which to save the key (/home/user/.ssh/id_rsa):
Enter passphrase (empty for no passphrase): 
Enter same passphrase again: 
Your identification has been saved in /home/user/.ssh/id_rsa.
Your public key has been saved in /home/user/.ssh/id_rsa.pub.
The key fingerprint is:
b2:ad:a0:80:85:ad:6c:16:bd:1c:e7:63:4f:a0:00:15 user@host
The key's randomart image is:
+--[ RSA 2048]----+
|  E.             |
| .               |
|.                |
|.o.              |
|.ooo o. S        |
|oo+ * .+         |
|++ +.+...        |
|o. ...+.         |
|  .   ..         |
+-----------------+
```

Then copy your newly created key into your Node A

```
ssh-copy-id root@[Node A]
```

Finally log into your Node A from your current MARS host

```
ssh root@[Node A]
```
## Create your data block device

This section assumes you have a /dev/sdb disk available to create the new block device. More information on how to create block devices can be found [here](http://software.clapper.org/cheat-sheets/linux.html). If you want to learn more on how to create partition then you will find more information [here](https://download.parallels.com/desktop/v5/docs/en/Parallels_Desktop_Users_Guide/23117.htm).

**1. Create a new MBR partition**

Start fdisk using the following command:

```
fdisk /dev/sdb

```

In fdisk, to create a new partition, type the following command:

```
n
```

* When prompted to specify the Partition type, type p to create a primary partition or e to create an extended one. There may be up to four primary partitions. If you want to create more than four partitions, make the last partition extended, and it will be a container for other logical partitions.
* When prompted for the Number, in most cases, type 3 because a typical Linux virtual machine has two partitions by default.
* When prompted for the Start cylinder, type a starting cylinder number or press Return to use the first cylinder available.
* When prompted for the Last cylinder, press Return to allocate all the available space or specify the size of a new partition in cylinders if you do not want to use all the available space.

Write your changes:

```
w
```

**2. Create a new physical volume**

```
pvcreate /dev/sdb1
```

**3. Create a new volume group**

```
vgcreate vol_grp_myspace /dev/sdb1
```

**4. Create the logical volume**

```
lvcreate --name myspace --extents 100%FREE vol_grp_myspace
```

**5. Format the new volume**

```
mkfs.ext4 /dev/vol_grp_myspace/myspace
```

## Setup a new MARS cluster

**1. Create a new folder for the transaction logs**

MARS uses the folder ```/mars``` to store its transaction logs, it is a good idea to mount a different partition (dedicated disk) into this folder, but in this tutorial you will simply create a new folder in every MARS host

```
mkdir /mars
```

**2. Create a new MARS cluster**

Execute the following command in the Node A only

```
marsadm create-cluster
```

**3. Join the rest of the cluster nodes to your newly created MARS cluster**

```
marsadm join-cluster [Node A]
```

**4. Load the MARS kernel module**

Execute the following command on all of your MARS cluster nodes

```
modprobe mars
```

**5. Check if the cluster is healthy**

If the cluster is healthy, you will see communication going back and forth using port 7777

```
netstat --tcp | grep 7777
```

## Create a new resource for your MARS cluster

**1. Create a new primary resource**

In order to create a new cluster, execute the following command **ONE TIME ONLY** on your Node A

```
marsadm create-resource mydata /dev/vol_grp_myspace/myspace
```

As a result, a directory /mars/resource-mydata/ will be created on Node A, containing some symlinks. Node A will automatically start in the primary role for this resource. Therefore, a new pseudo-device /dev/mars/mydata will also appear after a few seconds.
Note that the initial contents of /dev/mars/mydata will be exactly the same as in your pre-existing disk /dev/vol_grp_myspace/myspace.

**2. Create a new folder to mount the volume**

```
mkdir /mydata
```

**3. Mount the new volume**

```
mount /dev/mars/mydata /mydata
```

At this point you can access the contents of your disk using the mountpoint /mydata.

## Join the other nodes to your MARS cluster

**1. Join a primary resource**

```
marsadm join-resource mydata /dev/vol_grp_myspace/myspace
```

**2. Check all of your nodes to ensure everything is OK**

```
marsadm view mydata
```
