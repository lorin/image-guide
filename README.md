# OpenStack Virtual Machine Image guide

## Introduction

This guide describes how to obtain, create, and modify virtual machine images that are compatible with OpenStack.

If you are running KVM, the most common image formats are raw and QCOW2. However, you may need to convert from a different format (e.g., if you are importing an image from another hypervisor such as VirtualBox).

## Obtaining images

The simplest way to obtain a virtual machine that works with OpenStack is is to download it from elsewhere.

### Test image (CirrOS)

[CirrOS] is a minimal Linux distribution that you can use to test that you can boot images on your OpenStack cloud.

[CirrOS download page]


### Ubuntu

Canonical maintains a set of [Ubuntu cloud images], organized by release version and release date. For example, the [Precise "current" page] (scroll down to the bottom) has 

[Ubuntu cloud images]: http://cloud-images.ubuntu.com/
[Precise "current" page]: http://cloud-images.ubuntu.com/precise/current/

### Other distributions 

As of this writing, there are no officially released free virtual machine images from distributions such as CentOS, Debian, Fedora.

### Unofficial images


[CirrOS]: https://launchpad.net/cirros
[CirrOS download page]: https://launchpad.net/cirros/+download


# Converting image formats

You'll often need to convert images from one format to another.

## Raw to QCOW2

This will convert a raw image file named centos63.dsk to a qcow2 image file 

	qemu-img convert -O qcow2 centos64.dsk centos64.qcow2

To convert from qcow2 to raw, you would do:

	qemu-img convert -O raw centos64.qcow2 centos64.img

Once converted, you can now add it to the OpenStack Image Service. For example:

	glance image-create --name centos64 --disk-format qcow2 --container-format bare --is-public True < centos64.qcow2
	
## VDI (VirtualBox) to Raw

VirtualBox uses the VDI format for virtual machine images. If you've created an image using VirtualBox, you can convert it to raw format using the VBoxManage command-line tool:

    VBoxManage clonehd ~/VirtualBox\ VMs/fedora18.vdi fedora18.img --format raw

# Altering an image

## guestfish

[guestfish] is a tool that allows you to modify the files inside of a virtual machine image. The Ubuntu package name is `guestfish`. 

Note that guestfish doesn't mount the image directly into the local filesystem. Instead, it provides you with a shell interface that provides similar functionality, allowing you to view, edit, and delete files.

Assume we have a CentOS raw image called `centos63_desktop.img` and we want to delete the `/etc/udev/rules.d/70-persistent-net.rules` file and `/lib/udev/rules.d/75-persistent-net-generator.rules` file and remove `HWADDR` from the `/etc/sysconfig/network-scripts/ifcfg-eth0` file. 

We would mount the image in read-write mode by doing, as root:

	guestfish --rw -a centos63_desktop.img
	
This starts a guestfish session. We must first use the `run` command at the guestfish prompt:

	><fs> run
	
We can view the filesystems in the image using the `list-filesystems` command:

	><fs> list-filesystems
	/dev/vda1: ext4
	/dev/vg_centosbase/lv_root: ext4
	/dev/vg_centosbase/lv_swap: swap


We need to mount the logical volume that contains the root partition:

    ><fs> mount /dev/vg_centosbase/lv_root /
    
We want to delete some files:

	><fs> rm /etc/udev/rules.d/70-persistent-net.rules
	><fs> rm /lib/udev/rules.d/75-persistent-net-generator.rules  

We want to edit the ifcfg-eth0 file to remove the HWADDR line. The `edit` command will copy the file locally, invoke an editor, and then copy the file back. 

	><fs> edit /etc/sysconfig/network-scripts/ifcfg-eth0
	

We want to load the 8021q kernel at boot time. So we do:

	><fs> touch /etc/sysconfig/modules/8021q.modules
	><fs> edit /etc/sysconfig/modules/8021q.modules
	
We add the following to the file:

	modprobe 8021q

Then we set to executable:

	><fs> chmod 0755 /etc/sysconfig/modules/8021q.modules



Exit now that we are done:

	><fs> exit


[guestfish]: http://libguestfs.org/guestfish.1.html


## Mounting an image to the lcoal file system

loop devices, kpartx and network block devices (nbd).


# Manually creating an image
Eventually, you'll probably need to create an image from scratch. To do this, you'll need access to:

 * An installation CD or DVD ISO image file for the distribution you want to build from
 * A virtualization tool (KVM, VirtualBox, …)
 
## Manual creation with KVM
 
 


# Automating image creation
There are several automated tools for doing image creation. 

## Oz

Oz is a tool that automates the process of manually creating a virtual machine image file.  It is a Python application that interacts with libvirt and KVM. It uses a predefined set of kickstart (RedHat-based systems) and preseed files (Debian-based systems) for supported operating systems.

The main downside of Oz is that it isn't yet packaged for Ubuntu, which means you'd need to do a source install to get it to work.

Here's how you would create a CentOS 6.4 image with Oz. 


Create a file called centos64.tdl, with the follwowing contents. The only thing you need to change is the `<rootpw>` entry.
	
	<template>
	  <name>centos64</name>
	  <os>
	    <name>CentOS-6</name>
	    <version>4</version>
	    <arch>x86_64</arch>
	    <install type='iso'>
	      <iso>http://mirror.rackspace.com/CentOS/6/isos/x86_64/CentOS-6.4-x86_64-bin-DVD1.iso</iso>
	    </install>
	    <rootpw>CHANGE THIS TO YOUR ROOT PASSWORD</rootpw>
	  </os>
	  <description>CentOS 6.4 x86_64</description>
	  <repositories>
	    <repository name='epel-6'>
	      <url>http://download.fedoraproject.org/pub/epel/6/$basearch</url>
	      <signed>no</signed>
	    </repository>
	  </repositories>
	  <packages>
	    <package name='epel-release'/>
	    <package name='cloud-init'/>
	  </packages>
	  <commands>
	    <command name='update'>
	yum -y update
	yum clean all
	sed -i '/^HWADDR/d' /etc/sysconfig/network-scripts/ifcfg-eth0
	echo -n > /etc/udev/rules.d/70-persistent-net.rules
	echo -n > /lib/udev/rules.d/75-persistent-net-generator.rules
	    </command>
	  </commands>
	</template>

This Oz template specifies whwere to download the Centos 6.4 install ISO. It adds EPEL as a repository and install `epel-release` and `cloud-init` packages.

Once the image is created, Oz will customize it by doing an update and removing any reference to the eth0 device that libvirt creates while Oz was doing the customizing.

To run this, do:

    oz-install -d3 -u centos64.tdl 
    
The `-d3` flag tells Oz to show status information as it runs, and the "-u" tells Oz to do the customization (install extra packages, run the commands) once it does the initial install. 

If you leave out the `-u` flag, or you want to edit the file to do additional customizations, you can use the os-customize command, using the libvirt XML file that `oz-install` creates. For example:

    oz-customize -d3 centos64.tdl centos64Apr_03_2013-12\:39\:42


This will use libvirt boot the image inside of KVM, ssh into it, and perform the customizations.


## vmbuilder

vmbuilder ships with Ubuntu. Unfortunately, it currently only supports creating Ubuntu images. 
