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
2. MARS only runs on a 64-bit compatible OS. This guide was built and tested using Ubuntu 14.04 x64, so it is recommended that you also use the same version the first time you are doing this and then you can try it out on a newer version after understanding the basics.
3. You will need a host to recompile the kernel, this tutorial assumes your host already has all the utilities required to perform this task. Moreover, the kernel will be built using [the old fashioned way](https://help.ubuntu.com/community/Kernel/Compile#Alternate_Build_Method:_The_Old-Fashioned_Debian_Way) method.

## Kernel compilation

1. Determine which kernel version you are going to use and install it

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

2. Download your kernel's source code for recompilation

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

3. Patch your kernel with MARS' pre-patch files

Clone MARS' repo into your ```/home/myuser/kernel/linux-lts[kernel version]/block``` folder (make sure you perform the clone from the right folder, you don't want to clone your files on the wrong folder)

```
git clone https://github.com/schoebel/mars
```

And now you need to patch your kernel source with the right files. This action needs to be performed from the ```/home/myuser/kernel/linux-lts[kernel version]``` folder

```
patch -p1 < /home/myuser/kernel/linux-[kernel version]/block/mars/pre-patches/vanilla-[kernel version]/[patch name].patch
```

Just make sure you apply all the patches listed there.

4. Build your new kernel packages

First, you are going to enable MARS prior to performing the compilation, just execute the following command:

```
make oldconfig
```

Just make sure that when you get this prompt "storage system MARS (EXPERIMENTAL) (MARS) [N/m/y/?] (NEW)" you type **m**. You can simply press ENTER for the rest of the questions to keep the default values.

Now you are ready to actually compile your new kernel and generate the .deb files you need to patch your existing kernel:

```
make deb-pkg
```

At the end of this process and if everything goes well, you will see some new .deb files under ```/home/myuser/kernel```.