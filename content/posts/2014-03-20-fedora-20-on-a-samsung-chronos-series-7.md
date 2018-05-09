---
title: Fedora 20 on a Samsung Chronos Series 7
author: Andrea Veri
type: post
date: 2014-03-20T15:36:38+00:00
url: /2014/03/20/fedora-20-on-a-samsung-chronos-series-7/
categories:
  - Linux
  - Planet Fedora
tags:
  - fedora 20
  - NP700Z3C
  - samsung chronos series 7

---
It&#8217;s been a while now since the very first time I posed my hands on this shiny new **Samsung Chronos Series 7** laptop and oh dear&#8230; how much pain did my metallic-grey fellow take me in order to figure out how properly have every single piece of the hardware working as expected?

What I did right after unboxing it was dropping Windows 8 with a copy of Fedora 20 (yeah, stupid me, I could have booted Windows 8 at least once to check for UEFI / firmware updates) and setting everything up as usual. Right after booting the machine I disabled Windows 8&#8217;s Secure Boot, configured the laptop to boot from the USB Key I plugged in and restarted it to finally perform the real OS installation.

The laptop booted back (with UEFI mode marked as on) and the installation started. The Chronos Series 7 came with an iSSD of 16G in size, not much but definitely enough for keeping the root partition, swap and home directory. (I don&#8217;t need an huge home dir given all the various data is stored and mounted through NFS directly from the NAS)

The installation went just fine and no issues arised at all until I booted into the system. The laptop since the beginning started to reach high CPU and graphics card temperatures (sticking around **75-78 C** for the CPU and **60 C** for the NVIDIA card), the fans were constantly on and I could feel the heat of the machine while typing. But that&#8217;s not all, the backlit keyboard had its lights always set as on (even in the case of a locked screen) and the battery life was sticking around one hour and an half.

I&#8217;m now going to list all my findings and solutions for the above issues after spending some time debugging and trying hard.

## UEFI vs CSM (Legacy BIOS Compatibility mode)

Installing Fedora 20 with UEFI will result in your laptop not loading the **samsung-laptop** kernel module at all. That is the result of a known bug (with a good chance to brick your laptop) on Samsung laptops and UEFI boots (mainly related to the incompatibility between the samsung-laptop kernel module and the Samsung&#8217;s UEFI firmware and its corruption when the module is loaded) . More details <a href="https://bugs.launchpad.net/ubuntu-cdimage/+bug/1040557" target="_blank">here</a> and <a href="http://www.bit-tech.net/news/hardware/2013/01/31/linux-samsung-deaths/1" target="_blank">here</a>.

That said, before starting the installation, press **Fn+F2** right after powering up your laptop, you will be then prompted with the laptop&#8217;s Setup configuration. Switch to the **Boot** tab and disable Secure Boot making sure **CSM** is selected on the dropdown window. (before moving on make also sure to disable the **Fast BIOS Mode** on the Advanced tab before proceeding)

When done, boot up a copy of Fedora 20 with your preferred media (I did use the Live CD myself) and make your way through the installation. Make sure to read on before touching the disk partitioning schemas.

## iSSD not recognized with CSM mode marked as on

During one of my trials I did try to install the OS directly on the iSSD (in our examples, _/dev/sdb_) itself. The result was the system being completely un-bootable probably cause the EFI firmware being unable to recognize the iSSD on CSM mode. (the only disk that was getting recognized was the 750G HDD (in our examples, _/dev/sda_) the laptop has as an additional storage)

While the EFI firmware is not able to recognize the iSSD at all properly (when in CSM mode) it can flawlessly detect the HDD. That means one thing: we can keep the root, home and swap partitions on the iSSD and move the boot, bios boot partitions on the HDD itself and boot up the machine from there.

## The installation

We left our installation tutorial right before starting the installation itself through Anaconda. Let&#8217;s resume from there by making sure the following partitions are created on the HDD (from now on _/dev/sda_):

  1. a **bios-boot** partition (details on how to set it up at <a href="http://wiki.gentoo.org/wiki/GRUB2#BIOS.2FMBR_or_BIOS.2FGPT" target="_blank">here</a>).
  2. a **boot** partition (ext4, 500M in size)

Once done, perform the installation on the iSSD (from now on _/dev/sdb_), my setup on the 16G iSSD:

LVM Volume Group with three logical volumes:

  1. **5.12 G** /home + Luks (5.12 G for an home directory seems not enough but that&#8217;s definitely more than enough when you have a local HDD and a NAS with more than 2T of storage)
  2. **7.8 G** / (I prefer keeping root a big bigger than home to prevent the need to cleanup the yum cache and other packages cruft every now and then)
  3. **1.5 G** swap space

Another working setup might be:

  1. A **15G** / partition on the iSSD, no need for LVM here
  2. An LVM Volume Group that will store the /home and swap space so you can expand it to be more than just **5G / 1.5G**. The LVM VolGroup should ideally go on its own partition on _/dev/sda_

When the system has been installed, mount the _/dev/sda_ boot partition you previously created and install grub:

{{< highlight Bash >}}mount /dev/sda3 /mnt/boot-sda

grub2-install /dev/sda{{< / highlight >}}

I did assume _/dev/sda3_ was your /boot partition, make sure to check that is right for your case as well. (just run _fdisk -l /dev/sda_ to find out)

When done, mount the _/dev/sdb1_ partition and copy all the files to the previously created mount point /mnt/boot-sda. From there figure out the UUID (_ls -l /dev/disk/by-uuid_) for the /dev/sda3 partition and modify the relevant entries on the _/boot/grub2/grub.cfg_ file removing the UUID for the _/dev/sdb1_ (in my case that was the partition containing /boot) partition with the one of _/dev/sda3_. Save the file and reboot the machine.

You should then be able to unmount the /boot partition from _/dev/sdb1_ and modify the relevant _/etc/fstab_ entry with _/dev/sda3_&#8216;s UUID. That way new kernel&#8217;s installations will be handled correctly without the need to manually edit _/boot/grub2/grub.cfg_ and moving around the initramfs / vmlinuz images. At this point it should be safe to remove the _/dev/sdb1_ partition completely.

## Things to do after installing Fedora 20

### Installing Bumblebee

The **NP700Z3C** has a **NVIDIA GeForce GT 630M** that benefits from the **NVIDIA Optimus technology** which allows the user to gather the maximum performance possible when launching specific high-demand applications (like during gameplay or while watching an HD movie) and fallback to the integrated GPU when the user is performing normal operations like browsing the web, writing emails or text editing.

Luckily the Bumblebee project comes in help about this providing Optimus support for a variety of Linux distributions. More details on how to set it up <a href="https://fedoraproject.org/wiki/Bumblebee" target="_blank">here</a>. (you will need **bumblebee**, **bumblebee-nvidia** and **bbswitch**)

### Installing TLP

From the **tlp**&#8216;s website:

> TLP brings you the benefits of advanced power management for Linux without the need to understand every technical detail. TLP comes with a default configuration already optimized for battery life, so you may just install and forget it. Nevertheless TLP is highly customizable to fulfil your specific requirements.

I can tell you power management for your laptop has never been easier with tlp. Make sure to visit tlp&#8217;s <a href="http://linrunner.de/en/tlp/docs/tlp-linux-advanced-power-management.html" target="_blank">homepage</a> for more details on how to set it up.

### Samsung Tools

From the **samsung-tool**&#8216;s homepage:

> Samsung Tools is the successor of Samsung Scripts provided by the &#8216;Linux On My Samsung&#8217; project.
> 
> It enables control in a friendly way of the devices available on Samsung laptops (bluetooth, wireless, webcam, backlight, CPU fan, special keys) and the control of various aspects related to power management, like the CPU undervolting (when a PHC-enabled kernel is available).

Given there&#8217;s no RPM available for samsung-tools, downloading the tarball and running make as root should suffice for installing it on your laptop.

### Kernel flags

What seem to have helped me a lot with the high CPU temperatures (and thus with the noisy fans going on and on) are the following Kernel flags you should pass to Grub through the _/etc/sysconfig/grub_ file on the _GRUB\_CMDLINE\_LINUX_ line:

{{< highlight Bash >}}pcie_aspm=force i915.i915_enable_rc6=1 i915.i915_enable_fbc=1 i915.lvds_downclock=1 i915.semaphores=1 i915.modeset=1 acpi_osi=Linux rdblacklist=nouveau
{{< / highlight >}}

### Additional notes

The above procedure has been tested with the laptop having the following <a href="http://www.samsung.com/uk/consumer/pc-peripherals/notebook-computers/high-performance/NP700Z3C-S02UK-spec" target="_blank">specs</a>.

The average temperature for the CPU is sticking around **47-51 C**, while the discrete GPU at **49-51C**. I could also get around 3.5 &#8211; 4 hours of battery life!

That should be all! Please leave me a comment in case of questions, troubles with the above setup!
