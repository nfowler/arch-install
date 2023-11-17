This is a installation guide or "runbook" detailing my personal Arch Linux setup. The unique points of this install are as follows:

1. LUKS
2. Btrfs with bootable snapshots
3. rEFInd for above bootable snapshots

In addition, this guide takes great care to not just tell you *what* to do, but the *why*. It doesn't help you learn if you're just entering commands or setting options with no idea what they're actually doing.

To begin, this guide assumes you've followed steps 1.1 through 1.8 on the [official Arch Linux installation guide.](https://wiki.archlinux.org/title/installation_guide#) We'll pick up on step 1.9 "Partition the disks".

### Partitions

On the most basic level, we only need two partitions: a boot partition and the partition that will contain a LUKS encrypted container. This encrypted container contains our actual filesystem, sometimes called the "root partition".

There are multiple ways to go about creating these partitions, and you should use the method you're most comfortable with. For demonstration purposes, and because it's what the official guide uses, we'll use `fdisk`.

First, verify the disk you'll be partitioning.

```
# fdisk -l
```

In our case, we're experimenting with a VM, so our device is simply called `/dev/vda`. Your device might be something like `/dev/sda` or, if you have an NVMe `/dev/nvme0n1`. Once you've verified your device, create the partitions.

In `fdisk` this means executing the command with the device as an option.

```
# fdisk /dev/vda
```

First, create a new empty GPT partition table. Then, create two new partitions. The first should be 512MB, and the second consists of the remaining space. 

```
Command: g
Command: n
Partition number: 
First sector: 
Last sector: +512M
```

This will create a new partition of size 512MB, of type `Linux filesystem`. Note that empty commands mean we're going with the default option.

Since this is our ESP, we want it to be of type `EFI System`. In `fdisk` we can use the `uefi` alias, or `1` for our type.

```
Command: t
Partition type or alias: uefi
```

The partition type should change from `Linux filesystem` to `EFI System`.

Now, create another partition. Since the default type is what we want, we don't need to bother changing it. Since we're creating this as our second and final partition, `fdisk` will also take care of the size.

```
Command: n
Partition number: 
First sector: 
Last sector:
```

Simple as that.

We can now write our partitions to the disk and exit. 

```
Command: w
```

### LUKS

Now we can get to creating our LUKS scheme.

First, run the following command to see which encryption method you should use. This will ensure that you're wasting the least amount of time encrypting and decrypting your drive.

```
# cryptsetup benchmark
```

In my benchmark, the fastest algorithm is `aes-xts` with a key of `256b`.

With this information, we can craft the following command to create a LUKS container with our specific needs.

```
# cryptsetup luksFormat --type luks2 --key-size 512 -cipher aes-xts-plain64 --hash sha256 /dev/vda2
```

Note that we're using a keysize of `512` despite just noting that the fastest is `256`. This is because the `aes-xts` cipher splits our key in half.

Since a lot of these values are in fact the defaults, we can just use:

```
# cryptsetup luksFormat /dev/vda2
```

If you're installing on bare-metal with an SSD as your device, you might want to include the following options, which aren't defaults but will increase performance:

```
# cryptsetup luksFormat --perf-no_read_workqueue --perf-no_write_workqueue /dev/vda2
```

Once you execute this command, you'll receive a warning that all data on the disk will be deleted, and you'll be asked to enter a password. Make sure you pick a good one, since it's likely to be the weakest part of this secure system. Also make sure you don't forget it, since you definitely won't be able to recover it or your data if you do.

Now that we've created our encrypted container, we can open it.

```
# cryptsetup open /dev/vda2 cryptlvm
```

Here, the `cryptlvm` option acts as the name of the container. It can thus be anything you would like, but make sure to remember it and replace instances of `cryptlvm` in future commands with what you chose.

### LVM

This step is optional and experimental. It is unclear if it's even worth pursing this additional layer of complexity given the features and benefits of Btrfs.
#### Physical Volume

To set up LVM, we first need to create a physical volume. We do this by simply using the LUKS encrypted container we just created and opened.

```
# pvcreate /dev/mapper/cryptlvm
```

Remember to replace `cryptlvm` with whatever you named your LUKS container in the previous step.
#### Volume Group

Now, we'll create a volume group.

```
# vgcreate secure /dev/mapper/cryptlvm
```

In this command, `secure` is the name of the volume group. As with previous steps, you can change this name, but make you you remember what you set it as.
#### Logical Volumes

We have a volume group, so let's make our logical volumes. You can think of these as sort of "virtual" partitions.

We'll create two volumes, one swap volume, to act as a virtual swap partition, and one root volume, to contain all of our actual data and the system itself.

```
# lvcreate -L 4G secure -n swap
```

This creates the swap volume. Since it's inside the `secure` volume group, and named `swap`, we can refer to this as `secure-swap`, and it is located at `/dev/mapper/secure-swap`. We'll use this reference later.

One thing to note is that it's 4 gigabytes in size in size. How big or small your swap partition or file should be is a matter of some debate and controversy. If you'd like to hibernate your system, then you want it to be roughly as big as your RAM. Personally, I never hibernate, so don't need that much swap.

To make an educated decision, it's important to possess at least a baseline level of knowledge regarding what swap is exactly.

When you run out of RAM, the Linux kernel has two choices: it can kill certain applications it thinks you won't miss, thus freeing up RAM, or it can take lesser used chunks of data in the RAM and offload them to your swap.

Thus, having swap means your kernel won't kill applications on its own, which some people value.

If your swap is big enough, you can also set your computer into "hibernate". This is a type of deep sleep, so deep that your RAM loses all data, due to its nature as volatile memory. So, the system will copy everything on the RAM into your swap, and restore it into RAM when you wake it up from hibernate. That's why hibernate requires roughly as much swap as you have RAM, sometimes even more.

Nowadays, it's not infeasible to have so much RAM it would take a significant chunk of your storage space dedicated to swap to allow hibernate.

Moving on, let's create the our root volume.

```
# lvcreate -l 100%FREE secure -n root
```

Now, we've created our root volume, referred to as `secure-root`.

We now have all of our partitions, both real and virtual, so let's move onto filesystems and mounting.
### ESP

First, we'll format our initial partition, the one we assigned as an `EFI System`. We're booting off of this partition, so it needs to be a particular format. Note that, since it exists outside of the encrypted container we just made, we could have technically done this earlier, but for readability purposes we're doing it here.

On that note, the reason this boot partition isn't encrypted is due to the fundamental difficulty of booting off of an encrypted partition. It is possible, but requires a different boot loader than the one we'll be installing today, and is out of scope of this guide regardless.

With that out of the way, let's format our boot partition.

```
# mkfs.fat -F 32 -n ESP /dev/vda1
```

Some guides will have a command like:

```
# mkfs.vfat -F32 -n ESP /dev/vda1
```

These commands are identical to the computer, and do the exact same thing.

In both commands, the option following `-n` is the name of the partition. In this case we're naming it `ESP` for `EFI system partition`. Other common names are `EFI` and `BOOT`. Any name works, and is purely for human readability purposes.

### Btrfs

Now, let's do the exciting bit! Formatting our root partition as Btrfs, and creating subvolumes.

Before we jump into commands however, I'll explain some of the benefits of Btrfs, and briefly list some other file systems you might want to consider.
#### Why Btrfs?

The primary advantage of Btrfs is, as far as this guide is considered, its snapshot feature. Btrfs snapshots allow you to go back to a previous version of your file system, and thus your computer. This is very useful if you've made a mistake and can't fix it, or if the fix is particularly troublesome and time consuming.

Instead of having to troubleshoot and fix problems, you can just boot into a snapshot and pretend you never made a mistake in the first place!

In addition, snapshots can, and should, be made to only snapshot particular portions of your file system. For example, you can set it so that only snapshots of your root filesystem are captured, while ignoring your home (containing all of your personal files and configurations) and the system logs.

This has the effect of booting into a previous version of your file system, while having a completely up-to-date version of your home, and being able to look into the logs to see what exactly went wrong.

Some additional features of Btrfs is inbuilt compression and the ability to merge devices into a single indistinguishable file system, one of the main advantages of LVM. There are many other features, and you're recommended to look into them all before making a decision. A good starting point, as with most things, is the Arch Linux wiki page for [Btrfs](https://wiki.archlinux.org/title/Btrfs).
#### Alternative file systems

Some popular alternative file systems are `ext4`, `zfs`, and `xfs`. 

[Ext4](https://wiki.archlinux.org/title/Ext4) is the most widely used, and thus documented Linux file system there is, and is the default for a vast number of distributions. If the device you're installing to is mission critical, you might want to go with this. Having said that, you probably shouldn't install Arch on mission critical systems anyways, given its nature as a rolling release distribution.

[ZFS](https://wiki.archlinux.org/title/ZFS) is another advanced file system like Btrfs, with pooled storage, snapshots, and a massive maximum storage limit. It's used by supercomputers for its incredible storage capabilities. However, it's also under a licence that renders it incompatible with the Linux kernel, and thus requires a few extra steps.

[XFS](https://wiki.archlinux.org/title/XFS) is a "high-performance journaling system". Additional information can be found at the provided wiki page.

All of these are great file systems, but we'll be using Btrfs for this guide.
#### Formatting Btrfs

With that out of the way, let's format our encrypted partition to Btrfs.

```
# mkfs.btrfs -L ROOT /dev/mapper/secure-root
```

There are two variables here that might be different for you. The first is the label of the file system, which we have set here as `ROOT`. Feel free to set this to whatever label you would like. As with the boot partition name, it is purely for human readability and reference purposes. The second is the `crypt` label. As you might remember, we designated this as the name of our encrypted container, and it might be different for you depending on what you called it when you opened it.

Next, let's mount this new file system to `/mnt`. Right now you can imagine that the whole file system is floating around, not connected to anything, and not accessible to anything. We'll need to do some more configuration inside the file system, so we need to be able to access it at a set "mount point".

```
# mount /dev/mapper/secure-root /mnt
```

#### Subvolumes

Now, we can start creating Btrfs subvolumes. These subvolumes are an important part of the Btrfs file system, but what subvolumes you define vary depending on what you're concerned about and your needs.

Remember that we can make snapshots of a subvolume, and we can boot into those snapshots. Well, a snapshot of a subvolume will ignore other subvolumes. So, the primary benefit of subvolumes is defining certain parts of the file system that we don't want to be rolled back when or if we boot into a snapshot of /.

Below is a list of the subvolumes I've defined, and my reasoning for them. Aside from a few basic ones like @ (the root subvolume) and @home (the home subvolume) the rest are really up to you.

| SUBVOL     | PATH                      | DESCRIPTION                                                         |
| ---------- | ------------------------- | ------------------------------------------------------------------- |
| @          | /mnt                      | The root subvolume, absolutely necessary.                           |
| @home      | /mnt/home                 | Contains your personal files and configurations                     |
| @log       | /mnt/var/log              | Contains the system journal, so you can more easily diagnose issues |
| @pkg       | /mnt/var/cache/pacman/pkg | Contains the pacman cache, to keep the size of snapshots down       |
| @snapshots | /mnt/.snapshots           | Contains your actual snapshots                                      |

There are many other directories that you could defined as being subvolumes, but I feel that this is a good happy medium between too little and too much. Feel free to modify this scheme however you'd like.

Once you know your desired scheme, we can actually create the subvolumes.

```
# btrfs subvolume create /mnt/@
# btrfs subvolume create /mnt/@home
# btrfs subvolume create /mnt/@log
# btrfs subvolume create /mnt/@pkg
# btrfs subvolume create /mnt/@snapshots
```

Once you've created all of them, you can unmount.

```
# umount -R /mnt
```

The `-R` option here recursively unmounts everything, which is necessary since we're also mounted at `/mnt/boot` for the boot partition.

Then, we can mount all of our subvolumes instead. Since our file system at the moment has barely been created, we're going to have to create a few of these directories ahead of time.

```
# mount -o noatime,compress=zstd,commit=120,subvol=@ /dev/mapper/secure-root /mnt
```

This is our root and most important subvolume.

To explain the options a little: by default Linux file systems will write to the disk every time a file or directory is accessed. This can be fairly expensive, and frankly isn't useful today. The `noatime` option disables this function, saving you performance. You might also see mount options with `nodiratime`. If you have `noatime` you don't need `nodiratime`, since `noatime` implies it. 

The `compress=zstd` option defines the type of compression we'd like, with `zstd` being the most aggressive and effective. Other compression options are described on the Btrfs wiki page.

The `commit=120` option sets how frequently data is synced to permanent storage, with the value being the time in seconds. Higher values means more performance, but a greater chance of significant data lose in the event of a crash. By default this is set to `30`, but unless you're working with sensitive data that can change relatively quickly, you're probably fine with higher.

Some options you might see being passed in other guides are: `ssd`, `space_cache=v2`. 

`ssd` is a Btrfs-specific option that provides optimizations if you're installing on an SSD. However, Btrfs will auto-detect if you're installing on an SSD, and pass the option for you. There's no harm passing it yourself however.

`space_cache=v2` is another Btrfs-specific option, and is now the default. You don't need to pass it explicitly.

With all that explained, let's mount the rest of our subvolumes. Before we can do that though, we need to create a few directories. Otherwise, we'll be mounting to places that don't exist, and it'll complain.

```
# mkdir -p /mnt/{home,var/log,var/cache/pacman/pkg,.snapshots}
```

Now we can finally mount our subvolumes.

```
# mount -o noatime,compress=zstd,commit=120,subvol=@home /dev/mapper/secure-root /mnt/home
# mount -o noatime,compress=zstd,commit=120,subvol=@log /dev/mapper/secure-root /mnt/var/log
# mount -o noatime,compress=zstd,commit=120,subvol=@pkg /dev/mapper/secure-root /mnt/var/cache/pacman/pkg
# mount -o noatime,compress=zstd,commit=120,subvol=@snapshots /dev/mapper/secure-root /mnt/.snapshots
```

You can execute the following command to check that all of your subvolumes are mounted to the correct places:

```
# findmnt -nt btrfs
```

One last thing, we need to mount our bootable partition, otherwise none of this will work. 

```
# mount --mkdir /dev/vda1 /mnt/boot
```

Remember that, in this guide, `/dev/vda1` is our boot partition. Replace it with your own when running this command.

### Swap

We've formatted and mounted our boot partition, and now formatted and mounted our `secure-root` volume. Let's not forget our `secure-swap` volume.

We already covered what swap is and why you might want it, so we'll move straight into setting it up.

```
# mkswap /dev/mapper/secure-swap
```

This formats and sets the volume as being for swap.

```
# swapon -s /dev/mapper/secure-swap
```

And this actually enables it as swap.
### Recap

With that done, let's quickly recap what we've done so far.

We formatted our disk into two partitions, one of size 512MB for our boot partition, and one containing the rest of our space for our root partition.

We encrypted the root partition after checking all the available ciphers for optimal performance.

We setup our LVM scheme inside the encrypted partition, creating volumes for our root and swap.

We created all the subvolumes we needed and wanted, and mounted those too.

We formatted and mounted our boot partition.

We formatted and enabled our swap volume.

With all that done, it's time to get into the actually exciting bit: installing our system.

Currently, we're not actually in our system, we're in a live ISO environment, interacting with our system from the outside. That's why we've had to use the /mnt path everywhere.

### Pacstrap

Our live environment right now has a few packages that we won't have access to in the real system, including some really essential ones. We can install those packages into the system from our current environment using the `pacstrap` command.

The exact list of packages you might want to install depends on your wants and your system.

NOTE: Before you do this, double check that your boot partition (in this guide `/dev/vda1`) is mounted at `/mnt/boot`.

```
# pacstarp /mnt linux linux-firmware base
```

This is the minimum that you would need to create a working system, but we're going to want a bit more than this.

```
# pacstrap /mnt linux linux-firmware base base-devel btrfs-progs refind lvm2 networkmanager nano
```

This gives us all the necessary bits, along with a way to connect to the internet in the form of `networkmanager`, and a text editor for editing configuration files - which we're going to be doing a lot of - in the form of `nano`. If you're comfortable with vim, you can replace `nano` with `vim` or `neovim`.

Further, it includes a few things we'll need to configure out system and getting it working properly. The `base-devel` package includes a lot of useful stuff, but most importantly it'll allow us to install things from the AUR. More on that later.

The `btrfs-progs` package includes some necessary utilities for managing our Btrfs file system.

The `refind` package is what we need to make our system actually bootable, using the rEFInd boot manager.

The `lvm2` package is necessary for making our LVM volumes work.

In addition, you'll want to add either `intel-ucode` or `amd-ucode` depending on if you have an Intel or AMD CPU, respectively. These packages update the [microcode](https://wiki.archlinux.org/title/Microcode) of your CPU, which can fix bugs and issues that might otherwise cause serious issues.

Once all of your packages from the `pacstrap` command are successfully downloaded and installed, there one final thing to do before we can get inside the system itself.
### Fstab

We've mounted quite a few things so far, but we had to do all of that manually. We don't want to have to remember and execute a bunch of `mount` commands before we can actually use our computer, and attempting to boot into a system without a mounted boot partition wouldn't work anyways.

So, we'll generate an `fstab` file that defines what needs to be mounted and where, every single time the computer boots.

```
# genfstab -U /mnt >> /mnt/etc/fstab
```

This creates an `fstab` file from all the things we've mounted and currently have mounted so far, and moves it into the correct location on the actual system.

### Configure the system

To get into the system, we'll use the `arch-chroot` command.

```
# arch-chroot /mnt
```

That's it.

Now, we're inside the system, still as `root` however. From here, we'll do some important configuration.

#### Time Zone

First, we'll set the time zone. To do this, find the zone you belong to in:

```
# ls /usr/share/zoneinfo/
```

If you're location belong to one of the listed regions, then `ls` into that region directory to see more specific options. For example:

```
# ls /usr/share/zoneinfo/America/
```

Then, you can pick a city that will track the time zone you live in. Then, execute this command as such:

```
# ln -sf /usr/share/zoneinfo/America/Chicago /etc/localtime
```

If you can't find a city for you, but you know your time zone in respect to GMT (i.e. GMT+1, -1, ect.), then you can find all available time zones in `/usr/share/zoneinfo/Etc/`.

Also run:

```
# hwclock --systohc
```

To set the hardware clock.

#### Localization

Use `nano` or another text editor to edit `/etc/locale.gen`. Find your locale and uncomment them (remove the preceding "#"). For example, you'll probably want to uncomment `en_US.UTF-8 UTF-8` if you live in America.

Once you have uncommented your desired locales, generate them by running:

```
# locale-gen
```

Create a new file: `/etc/locale.conf` by simply executing your text editor as if it already existed. Set your LANG variable in the following format:

```
LANG=en_US.UTF-8
```

#### Network configuration

As before with the `locale.conf` file, create a `hostname` file at `/etc/hostname`. Add your host name, which is the name of your computer.

You can also use echo to create the file with your desired name:

```
# echo archlinux > /etc/hostname 
```

You can check the contents of the file with `cat`:

```
# cat /etc/hostname
```

#### Initramfs

Since we're using LUKS and LVM, we need to do some configuration of our mkinitcpio. The mkinitcpio is a script that generates an initramfs, or initial ramdisk environment. The important thing to know is that this environment is what exists very early in the boot process, and is thus responsible for things like decrypting an encrypted system. 

Use your text editor to edit `/etc/mkinitcpio.conf`. Search or scroll until you find a line in the form of `HOOKS=()`. We'll modify the contents of this array slightly.

If you've followed this guide even semi-closely, here is a working list of HOOKS:

```
HOOKS=(base udev keyboard autodetect modconf block encrypt lvm2 btrfs filesystems resume fsck)
```

What exactly each of these hooks does is outside the scope of this guide. You're recommended to read the [mkinitcpio](https://wiki.archlinux.org/title/Mkinitcpio) Arch Linux wiki page.

Once you've edited your `mkinitcpio.conf`, rebuild your initramfs image by running:

```
# mkinitcpio -P
```

Note that you'll probably get a bunch of warnings about missing firmware. You can ignore those.

#### Root password

Create a password for the root account of your computer:

```
# passwd
```

Try and make it strong and unique, since anyone gaining access to this account has almost complete and unfettered access to your device and all data on it.

#### Boot loader

Now onto big stuff once again: installing a boat loader. There's a wide variety of options to choose from, from the simple and reliable `systemd-boot` to the versatile and powerful `GRUB`. We won't be using those. Instead, we'll be using `rEFInd`. 

As to why, well the simple answer is that either `rEFInd` or `GRUB` would work perfectly for what we'd like to be able to do: boot into Btrfs snapshots. Which one you choose is a personal choice, and really doesn't matter that much. `GRUB` is much more common, and thus you'll find more documentation and support online for it. However, I personally prefer `rEFInd` and so that's what we'll be using.

Assuming you followed the `pacstrap` command from before, you should already have the necessary packages installed.

You can thus install the boat loader by running:

```
# refind-install
```

This utility should automatically detect your kernel(s) and install everything for you.

Once it's installed however, we have to do a fair amount of configuration.

First, you'll need to run this command. It'll give you multiple outputs. Remember or write down the "UUID" value.

```
# blkid /dev/vda2
```

Where `vda2` is your root partition.

Navigate to `/boot/EFI/refind/` and edit the `refind.conf` file. Scroll or search until you see the multi-line entry beginning with `menuentry "Arch Linux"`.

You'll need to edit the `options` portion with the following values, with your own values entered when necessary:

```
options  "cryptdevice=UUID=YOUR-UUID:secure root=/dev/mapper/secure-root rw rootflags=subvol=@ add_efi_memmap initrd=\amd-ucode.img resume=/dev/mapper/secure-swap"
```

Where the UUID value you got from the previous `blkid` command is in place of the `device-UUID` portion. Also remember to replace `amd-ucode.img` with `intel-ucode.img` if you have an Intel CPU.

Once final thing, replace any instances you see of `boot/` with `/`.

### Final configurations

With that, we should have a fully working, bootable system! There's a few things we want to do first though.

#### Create a user

Right now, the only user, and thus the only way to execute commands, is the root user. It's generally a bad idea to always be root, so we'll create a user we can use most of the time.

```
# useradd -m -G wheel,users -s /bin/bash USERNAME
```

Replace `USERNAME` with whatever you want your user to be called.

Set the user's password with:

```
# passwd USERNAME
```

#### Sudo

We often want to execute commands with the power of root, even though we don't want to be root all the time. Instead of constantly switching to the root user to execute these commands, we can use `sudo` to elevate any given command to root permissions.

First, install the `sudo` package.

```
# pacman -S sudo
```

Then, enter the `visudo` configuration file like so:

```
# EDITOR=nano visudo
```

This is a special requirement for this specific file, and you won't be able to edit it like we've done previously.

Once you're in the file, find and un-comment this line:

```
%wheel ALL=(ALL) ALL
```

This grants any user with the `wheel` group permission to use `sudo`. You might remember that when we created the user previously, we mentioned `wheel`. This is why.

