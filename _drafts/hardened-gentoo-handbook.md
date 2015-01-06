---
layout: post
title: "Hardened Gentoo Handbook"
tags: [gentoo,linux,security]
---

Gentoo is a GNU/Linux distribution that compiles all packages from source.
The compilation process is highly configurable with USE flags and compiler options.
I like it because the packages are not that different from upstream and it's very easy to look at the patches or even create your own patches.
It is also exceptionally well integrated with Grsecurity which in my opinion is the state of the art exploit mitigation and mandatory access control system.
Having control over the compilation process means you can take full advantage of toolchain memory protections and ASLR.
One disadvantage of this approach is the increased package installation time and resource requirements.
It can get pretty bad with larger packages needing gigabytes of disk space and compiling for almost a day.

Gentoo has a hardened profile optimized for security.
It compiles everything with stack smashing protection, creates ET_DYN executables and compiles without JIT support whenever possible.
Portage, the package manager, also handles PaX flags for you by default.
(Don't worry if you don't understand these terms yet, everything will become clear.)
Gentoo has an excellent installation guide.
The one thing that's missing from it is how to install a hardened system from the start.
Following the handbook then switching later is somewhat inconvenient but it can be done.

This post is intended to be used alongside the Gentoo Handbook for those of us who want the most secure installation.
I'm going to assume AMD64 architecture here and link to the relevant pages of the AMD64 handbook.
You can probably follow the guide using on a different architecture with minimal differences.

# Preparing the disks

[Relevant handbook chapter][handbook-disks].

We can boot the system from an encrypted volume.
This is the recommended approach if you are installng to a laptop or desktop computer.
Servers are different, they need to be able to boot without any human interaction.
Asking for a password on boot in not desirable.

Create 2 partitions with cfdisk as described in the handbook.
The first will be /boot, the second will be the encrypted LUKS volume.
On /boot you really only need enough space for a few kernels and initrd files, 64 MB should be more than enough.
Make sure /boot has the bootable flag set.
Allocate the rest for the LUKS partition.

If you want to use RAID, create the same partitions on all the disks.
Add the /boot partitions to a RAID 1 array *with v0.9 metadata*, and add the other partitions to a RAID array of your choice.
Use `/dev/md1`, `/dev/md2`, etc instead `/dev/sda1`, `/dev/sda2` for the remainder of this howto.

Example RAID setup:

<pre><code>mdadm --create /dev/md1 --level=1 --metadata=0.9 --raid-devices=4 /dev/sda1 /dev/sdb1 /dev/sdc1 /dev/sdd1
mdadm --create /dev/md2 --level=1 --raid-devices=2 /dev/sda2 /dev/sdb2
mdadm --create /dev/md3 --level=1 --raid-devices=2 /dev/sdc2 /dev/sdd2
mdadm --create /dev/md4 --level=0 --raid-devices=2 /dev/md2 /dev/md3
</code></pre>

Create the encrypted volume.

LUKS uses a password or key file to encrypt up to 8 key slots.
These key slots contain the actual data encryption key.
This allows for multiple users with different passwords and password change without full reencryption.
The keys are distributed among many physical disk blocks ensuring that you can easily nuke slots even if the hard disk firmware relocates blocks.
This is an essential property with SSDs and useful with magnetic disks as well.

Use PBKDF2-SHA512 to derive a key from the password and AES-XTS to encrypt the data.
[XTS][xts] is a very interesting cipher mode.
Unlike GCM and OCB, it does not provide integrity but unlike CTR and CBC, it can't be manipulated in a way that has predicable effects on the plaintext.
It also protects against watermarking attacks, when an attacker can place a file on the disk and confirm its presence by only looking at the ciphertext.
This is the best we can achieve for full disk encryption with any reasonable performance.

<pre><code>cryptsetup luksFormat /dev/sda2 --hash sha512 --cipher aes-xts-plain64 --key-size 256
cryptsetup luksOpen /dev/sda2 luks1</code></pre>

It is recommended that you use the encrypted volume as an LVM physical volume and create logical volumes for the different partitions the original handbook wants you to create.
The swap is fine as an LV too.
Just to mention one thing, it will be possible to create snapshots.
However, it is possible to create the filesystem directly on `/dev/mapper/luks1`.

<pre><code>pvcreate /dev/mapper/luks1
vgcreate vg1 /dev/mapper/luks1
lvcreate --name root --size 500G vg1
...
lvcreate --name swap --size 4G vg1
</code></pre>

Continue with filesystem creation as normal.

# Installing stage3

# Installing base system

# Configuring the kernel

# Configuring the system

# Installing tools

# Configuring the bootloader

# Finalizing

[handbook-disks]: https://wiki.gentoo.org/wiki/Handbook:AMD64/Installation/Disks
[xts]: https://en.wikipedia.org/wiki/Disk_encryption_theory#XEX-based_tweaked-codebook_mode_with_ciphertext_stealing_.28XTS.29
