---
layout: post
title: "Basic Security-oriented Arch Setup&#58 Part 1"
date: 2017-07-24
tags: [linux, arch, tutorials, hardening, operating systems]
comments: false
share: true
---

Hello and welcome to the inaugural post! This is just a quick writeup of my basic Arch linux configuration for those interested in perhaps trying out the distro with an eye towards security. This is just part 1 and does not aim to be comprehensive, and I will share my install scripts and dotfiles down the road.

This is an installation with lvm+luks encryption on 1 physical volume and various hardening features installed/enabled that is still usable as a general purpose desktop.

**Disclaimer:** This guide is intended for people reasonably familiar with Linux but not necessarily with Arch, or are familiar only with basic (unecrypted, non-hardened, etc.) Arch installs. I assume general familiarity with concepts such as logical and physical volumes, the Linux filesystem hierarchy, command-line networking, gpg usage, basic cryptography usage, etc.

### Basic setup

Once we boot into the Arch live environment, we can set the time before configuring our disks.

```
# timedatectl set-ntp true
```

With that out of the way, let's create a new GPT partition table. Suppose our target disk is `dev/sdX`. I prefer to use `gdisk`, although `parted` and such will also get the job done. Using `gdisk`, we execute `gdisk /dev/sdX` and then hit `o` to create the partition table.

Next, it's time to create our partitions. If we want lvm+luks, we actually only need two partitions on the physical volume; an EFI boot partition (ESP) and a volume for the lvm container. For the EFI partition, we execute

```
Command (? for help): n
Command (? for help): 1
Command (? for help): [Enter]
Command (? for help): +512M
Command (? for help): ef00
```

This just creates a 512 meg partition numbered 1 and with the type EFI (the hex code at the end). For the lvm container:

```
Command (? for help): n
Command (? for help): 2
Command (? for help): [Enter]
Command (? for help): [Enter]
Command (? for help): 8e00
```

which uses the remainder of the space. After checking the partition scheme with `p`, we can use `w` to write it.

Now that we have partitions, we need to give them actual filesystems. Starting with the ESP:

```
# mkfs.fat -F32 /dev/sdX1
```

which simply formats the ESP as FAT32, as is standard. We don't want to actually format the container obviously, so now we take a detour to set up the encryption using dm-crypt (the `dm_crypt` kmod should be loaded in the Arch install environment). It's very important to understand the next command. Here's what I typically use, as an example:

```
# cryptsetup luksFormat -v -s 512 -h sha512 /dev/sdX2
```

The important flags to note here are `-s`, `h`, and the *lack* of `-c`. These respectively specify the key size, the key derivation hash algorithm (specifically for creating a hash parameter for PBKDF2), and the actual cipher. If one omits a cipher, as I do, dm-crypt defaults to AES in XTS mode, which I personally am comfortable with. Older versions used ESSIV by default, and this essentially comes down to the XTS vs. ESSIV+CBC debate, which I will not rehash. AES-XTS should be sufficient for most users. As one can see, I also like to use a 512-bit key and sha512 as my hash algorithm for key derivation, as I find them a reasonable tradeoff on my systems compared to the default of sha256 and a 256 bit key. `luksFormat` also defaults to using `urandom`, [which you should](https://www.2uo.de/myths-about-urandom/).

However you choose to set up the encryption, we need to actually open the container to work on it, which we do with `luksOpen`:

```
# cryptsetup luksOpen /dev/sdX2 luks
```

Now we can actually set up our logical volumes, and to do so we need to 1) initialize the partition for use by lvm and 2) set up a root volume group, which we can do as follows:

```
# pvcreate /dev/mapper/luks
# vgcreate rootvg /dev/mapper/luks
```

Next comes the logical volumes, i.e. your "actual" partition scheme. I like to have separate home and var partitions, as well as a swap partition despite having a lot of RAM. You'll want to adjust the specifics here to your preferences and use case, e.g. by not having a separate var partition, using different sizes, or not using a swap partition.

```
# lvcreate -n swap -L 32G -C -y rootvg
# lvcreate -n root -L 100G rootvg
# lvcreate -n var -L 50G rootvg
# lvcreate -n home -l 100%FREE rootvg
```

A few notes are in order here. First, most users will likely not need a var partition this big, I just do a *lot* of logging. Second and more importantly, the lower-case l in the last command is not a typo, but is needed to use 100% of the remaining space. Before proceeding, it's worth checking the partition scheme with something like `lvs`.

The next move is to finally create some filesystems, and to activate the swap. Assuming you want to use ext4:

```
# mkfs.ext4 /dev/mapper/rootvg-home
# mkfs.ext4 /dev/mapper/rootvg-root
# mkfs.ext4 /dev/mapper/rootvg-var
# mkswap /dev/mapper/rootvg-swap
# swapon /dev/mapper/rootvg-swap
```

The time has finally come to actually install the minimal system to our now-configured storage. Let's create mount points and mount our logical volumes:

```
# mount /dev/mapper/rootvg-root /mnt
# mkdir /mnt/{var,home,boot}
# mount /dev/mapper/rootvg-home /mnt/home
# mount /dev/mapper/rootvg-var /mnt/var
# mount /dev/sdX1 /mnt/boot
```

With that in order, we'll select our mirrors. Luckily mirror selection in Arch is very easy because of the `rankmirrors` script. First back up your default mirrorlist:

```
# cp /etc/pacman.d/mirrorlist /etc/pacman.d/mirrorlist.backup
```

Then we use the `rankmirrors` script on the backup and direct the output back to the actual mirrorlist file. The `-n` option specifies how many of the top-ranked mirrors to keep as sources:

```
# rankmirrors -n 6 /etc/pacman.d/mirrorlist.backup > /etc/pacman.d/mirrorlist
```

At this point we're finally ready to synchronize with the repos and install the base system. So either execute a `# pacman -Syy` or `Syu` depending on if you want to upgrade the packages in the install environment or not. I usually do, but this can be a problem on small install media. I also like to run 

```
# pacman-key --init
# pacman-key --populate archlinux
```

to make sure the keyring is initialized and to repopulate the master signing keys (which should be verified). This usually is not necessary, but I've been burned by it before. After that we can finally install the base system into the environment we created:

```
# pacstrap /mnt base
```

And after this is complete, generate an `fstab`:

```
# genfstab -p /mnt > /mnt/etc/fstab
```

We're finally ready to chroot into our actual environment.

```
# arch-chroot /mnt
```

I like to set up my hostname first because otherwise I forget:

```
# echo <hostname> > /etc/hostname
```

And then immediately change the root password, because duh:

```
# passwd
```

And then the timezone, for similar reasons (exact location redacted):

```
# ln -s /usr/share/zoneinfo/Foo/Bar /etc/localtime
```

Continuing with the boring, mundane stuff, we need to set up our locale:

```
# vi /etc/locale.gen
```

Uncomment your locale(s) (e.g. `en_US.UTF-8`) and then run

```
# locale-gen
# echo "LANG=en_US.UTF-8" > /etc/locale.conf
```

Using `en_US.UTF-8` as the example. At this point, before further configuration, I like to install some basic software both so I don't be silly and forget, and as it aids in further configuration. The specifics are obviously up to you, but I like to do something like:

```
# pacman -S base-devel vim bash-completion sudo gnupg git
```

Now we can do some more interesting things, like enabling the IPv6 privacy extention. Create and edit the file `/etc/sysctl.d/ipv6_tempaddr.conf` with the following contents:

```
~ net.ipv6.conf.all.use_tempaddr = 2
~ net.ipv6.conf.default.use_tempaddr = 2
```

Sadly I lied, we're not quite done with the boring stuff. We need to make sure time is working and configure sudo and our non-root user(s). For sudo, simply use `visudo` and simply uncomment the option you prefer (I use the `%wheel%` group), and for time we just enable systemd's time service:

```
# systemctl enable systemd-timesyncd
```

Then add a non-root user, in this case with sudo privileges:

```
# useradd -m -G wheel philo
# passwd philo
```

Now we'll do more interesting things, in particular configuring `mkinitcpio` to work correctly with our encryption scheme. edit `/etc/mkinitcpio.conf` and modify the `HOOKS` line such that `keyboard keymap encrypt lvm2` all come *before* `filesystems`, which is necessary to actually unlock the volume. Then we can generate the `initrd`:

```
# mkinitcpio -p linux
```

Then install the bootloader:

```
# bootctl install
```

Which at the time of writing defaults to using systemd-boot rather than grub, so the rest of this guide will assume that. Everything should work with e.g. GRUB, but you will need to reference the documentation to find the equivalents. Regardless, we need to actually create a bootloader entry, and when we're using lvm+luks encryption we need to declare that, point to the correct device, and indicate where root resides. The best way to do this is via the UUID of the physical volume on which, in this example, is a single physical storage device, which can be obtained and written to a new bootloader configuration file as follows:

```
# blkid | grep sdX2 | cut -f2 -d\" > /boot/loader/entries.arch.conf
```

At this point the UUID will just be sitting at the top of the file by itself, so next we edit the file so that in the case of systemd-boot, we have something like:

```
~ title Arch Linux
~ linux /vmlinuz-linux
~ initrd /initramfs-linux.img
~ options cryptdevice=UUID=[The UUID from earlier]:luks root=/dev/mapper/rootvg-root rw
```

Before I reboot into my new (barebones) install, I like to at least get sound and U2F support set up for FIDO keys:

```
# pacman -S alsa-utils libu2f-host
```

Finally, let's leave our environment and reboot into it:

```
# exit
# umount -R /mnt
# reboot
```

Once we've rebooted into our new system, depending on if you chose to install any additional packages beyond what I said here, you'll likely have to manually configure the network, e.g. using `ip link` and `dhcpcd`.

In part 2, I'll discuss kernel hardening with the new (and awesome, seriously check them out) `linux-hardened` kernel and graphics/a DE to work with it, setting up a firewall and a MAC solution (SELinux in my case), jails, and more. And in part 3, I'll talk about setting up a StrongSwan-based IPSEC VPN on Arch, as well as additional mitigations like DNSSEC. Stay tuned!
