# Prerequisites & Info

<details>
	<summary>Tested FreeBSD versions</summary>

- FreeBSD 14.0-RELEASE-p4
- FreeBSD 14.0-RELEASE-p5
- FreeBSD 15.0-CURRENT amd64  (By @eternalblissed)

Will work on 15, should technically work on 13.
Don't know anything about FreeBSD forks or NetBSD.
</details>

<details>
	<summary>Needed packages</summary>

 - squashfs-tools
 - pacman		(optional for managing packages without chroot)
 - pulseaudio   (optional for sound)
 - alsa-lib			(optional for sound)
 - alsa-utils		(optional for sound)
</details>


# Tutorial

> THIS TUTORIAL IS UNFINISHED!

Make sure you have installed the prerequisite packages before following.

This tutorial assumes you have basic Linux, FreeBSD and just common UNIX skills.

Everything here is ran from the root account.

## /compat set-up

If you're on ZFS, create a dataset for the `/compat` directory and add it to `/etc/fstab` to have more control over your disk. What's the point of using ZFS if you clump everything in a single root dataset?

A simple way to do that would be 

```
zfs create -o mountpoint=/compat zpool/compat
echo "zpool/compat /compat zfs rw 0 0" >> /etc/fstab
```
Assuming your zpool name is "zpool"

---
`git clone` this repository into any directory, from now on this tutorial will assume you did it in `/tmp/artix-tutor`, then download the latest stable Artix dinit ISO from https://artixlinux.org/download.php

---
Unpack the ISO into `/compat/artix` using [unsqua.sh](scripts/unsqua.sh) from this repository. Example:
```
./unsqua.sh /root/artix.iso /compat/artix
```
The output will look like this:
```
# ./unsqua.sh /root/artix.iso /compat/artix
/usr/local/bin/unsquashfs
unsquashfs found

Parallel unsquashfs: Using 12 processors
82749 inodes (72471 blocks) to write

[=====================================================================|] 155220/155220 100%

created 71465 files
created 6658 directories
created 9220 symlinks
created 0 devices
created 0 fifos
created 0 sockets
created 2064 hardlinks
```
## Or if you prefer to do it manualy
Assuming your trying to run  `./unsqua.sh artix.iso /compat/artix LiveOS/rootfs.img` in `/tmp/artix-compat/tutor`

Associate the ISO with a Device:
Use mdconfig to create a memory disk from the ISO file:

```bash
mdconfig_device=$(mdconfig artix.iso)
```

This command will output the name of the memory disk created, such as md0.

Create a Temporary Mount Point:
You might want to create a temporary directory for mounting the ISO:

```bash
temp_mount=/tmp/$(md5 -q -s "$RANDOM")  # Creates a somewhat unique directory name
mkdir -p "$temp_mount"
```

Mount the ISO:
Mount the ISO to the temporary directory:

```bash
mount -t cd9660 /dev/$mdconfig_device "$temp_mount"
```

Extract the SquashFS:
Now, use unsquashfs to extract the SquashFS filesystem to your desired location:

```bash
unsquashfs -d /compat/artix "$temp_mount/LiveOS/rootfs.img"
```

This extracts the contents of LiveOS/rootfs.img from the ISO to /compat/artix.

Cleanup:
After extraction, unmount the ISO and remove the memory disk:

```bash
umount "$temp_mount"
mdconfig -d -u $mdconfig_device
```

And remove the temporary mount directory:

```bash
rmdir "$temp_mount"
```

Check if your FreeBSD installation has `/opt`, if it doesn't then create it, then copy [scripts/artix](scripts/artix) from this repository into `/usr/local/etc/rc.d` and run `service artix enable && service artix start` to mount all the filesystems in linuxulator.

---
Run `printf "nameserver 1.1.1.1\nnameserver 1.0.0.1\n">>/compat/artix/etc/resolv.conf` to populate resolv.conf in compat

---
You can now run simple Linux apps, but we're still not done yet!

## Fixing pacman & updating

If you haven't done that yet, install pacman (`pkg ins pacman`), then replace your pacman configs with the ones from this repo:

Copy [pacman/chroot.conf](pacman/chroot.conf) to `/compat/artix/etc/pacman.conf` and [pacman/pacman.conf](pacman/pacman.conf) to `/usr/local/etc/pacman.conf`.

---
Go to [archlinux.org/mirrorlist](https://archlinux.org/mirrorlist) and generate a mirrorlist for `/compat/artix/etc/pacman.d/mirrorlist-arch` (place the text in there), then comment out the mirrors you don't need from `/compat/artix/etc/pacman.d/mirrorlist` since you don't want to use mirrors located on the opposite side of this planet.

---
Link the keyring directory with `ln -sv /compat/artix/usr/share/pacman/keyrings /usr/local/share/pacman/keyrings`, then run `pacman-key --init && pacman-key --populate` to initialize the keying

---
DO NOT `pacman -Syu` YET!

Sync the repos and install the Arch Linux keyring for the additional repositories (forcefully updating the artix keyring too):
```
pacman -Sy archlinux-keyring artix-keyring
pacman-key --init
pacman-key --populate
```
This will save you from a lot of GPG headaches

---
Update time!

Remove the packages we're not going to use, then update all the packages we have left:
```
pacman -Runc linux linux-firmware linux-firmware-whence alsa-firmware intel-ucode amd-ucode grub
pacman -Syu
```
- "Why would you remove linux??"

The linux and linux-firmware packages are only needed to boot the system on real hardware, you don't need them to use Linuxulator.

## Installing Discord

Install Discord with its additional dependencies with `pacman -S discord libappindicator-gtk3 xdg-utils alsa-lib pulseaudio`

---
Drop the [scripts/opt](scripts/opt) directory into `/opt/scripts`, copy `/opt/scripts/example-wrapper` to `/opt/discord/wrapper` and `/opt/scripts/example-exec` to `/usr/local/bin/discord`

---
Run `discord` as a normal user.

# Installing an AUR helper
create a new user with the same name and UID as your FreeBSD user using `useradd $name -u $uid -m -s /bin/bash -G wheel,input,audio,video,games` (replace $uid and $name with the appropriate strings)

Set a password for root and the freshly created user with `passwd` and add the user to `/etc/sudoers`.

`su` into the user and install [yay](https://github.com/Jguer/yay) or [paru](https://github.com/Morganamilo/paru) using the instructions provided in their repository.
