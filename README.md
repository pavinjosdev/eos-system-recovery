# Comprehensive Linux system recovery - from broken OS or hardware failure to a fully functional system.

This guide is written for [EndeavourOS](https://endeavouros.com/) but can be easily adapted for any Arch based system, and less easily for any other Linux based system. The concepts are the same irrespecitve of distribution.

Written from the perspective of a beginner to [copy-on-write](https://en.wikipedia.org/wiki/Copy-on-write) filesystems, this was originally intended as a personal reference but hopefully someone may find this updated version useful. Feel free to open an issue or PR for corrections.

---

# Goals

- Fast incremental snapshots of whole system that can be restored easily in the event of a broken OS.
- Offsite backup of snapshots that can be restored easily in the event of hardware failure.

> This setup should not be used in production, it was originally intended for a personal laptop without RAID and that means without the self healing feature offered by the Btrfs filesystem.

---

# Tools / technologies

- [Btrfs](https://btrfs.wiki.kernel.org/index.php/Main_Page) ("Butter" FS): CoW filesystem with super fast snapshots
- [snapper](http://snapper.io/): automatic creation & management of btrfs snapshots
- [snap-pac](https://github.com/wesbarnett/snap-pac): automatic creation of pre/post btrfs snapshots on package changes when using pacman (Arch package manager)
- [grub-btrfs](https://github.com/Antynea/grub-btrfs): include btrfs snapshots in grub (bootloader) menu options
- [borg](https://borgbackup.readthedocs.io): backup program that supports deduplication, compression, and client side encryption - used for offsite backup and restore
- [borgmatic](https://torsion.org/borgmatic/): automatic creation & management of borg backups

---

# Information sources

Much of this was implemented by referring to the ArchWiki (Arch Linux documentation), this would not have been possible without their excellent documentation and its contributors. Thanks also goes to EndeavourOS forum members who were kind enough to clear up some doubts and the OS/package maintainers.

Sources not listed previously:

- Btrfs ArchWiki: https://wiki.archlinux.org/title/Btrfs
- Snapper ArchWiki: https://wiki.archlinux.org/title/snapper

---

# Disk topology

> For ease of setup, this guide assumes you're `root`. Be careful and read twice before executing any commands!

Disk and partitions:

```
[root@eos-laptop ~]# parted -l
Model: ATA KINGSTON SV300S3 (scsi)
Disk /dev/sda: 120GB
Sector size (logical/physical): 512B/512B
Partition Table: gpt
Disk Flags:  

Number  Start   End    Size    File system     Name  Flags
1      2097kB  539MB  537MB   fat32                 boot, esp
2      539MB   111GB  110GB   btrfs           root  legacy_boot
3      111GB   120GB  9449MB  linux-swap(v1)        swap
```

```
[root@eos-laptop ~]# lsblk /dev/sda
NAME   MAJ:MIN RM   SIZE RO TYPE MOUNTPOINTS
sda      8:0    0 111.8G  0 disk
├─sda1   8:1    0   512M  0 part /boot/efi
├─sda2   8:2    0 102.5G  0 part /home
│                                /var/log
│                                /var/cache
│                                /
└─sda3   8:3    0   8.8G  0 part [SWAP]
```

It's not required to have the exact aforementioned layout, but knowing the disk/partition topology would make following this guide easier.

We're naturally interested in the btrfs partition: `/dev/sda2`

Get the properties including UUID (universally unique identifier) of `/dev/sda2`:

```
[root@eos-laptop ~]# blkid /dev/sda2
/dev/sda2: UUID="63745996-340f-4bda-98b1-63af36e6cdae" UUID_SUB="c52df0d3-929a-4f2e-b980-5035036125e0" BLOCK_SIZE="4096" TYPE="btrfs" PARTLABEL="root" PARTUUID="a1969063-479c-3042-9a63-68560d5f5e23"
```

---

# Btrfs

Btrfs ("Butter" FS) is a modern CoW (copy-on-write) filesystem with a lot of interesting features like checksumming for both data and metadata that can be used for finding and correcting errors.

Since we're not using RAID (with mirroring), our setup can only find errors and not fix it automatically. We can always restore from the offsite backups if the original ever gets corrupted beyond repair.

For the purpose of this guide it's important to understand Btrfs _subvolume_ and _CoW_ concepts.

![Btrfs logo](https://i.imgur.com/muX701g.png)

## Subvolumes

- A Btrfs subvolume is an independently mountable POSIX filetree and not a block device (and cannot be treated as one). Most other POSIX filesystems have a single mountable root, Btrfs has an independent mountable root for the volume (top level subvolume) and for each subvolume; a Btrfs volume can contain more than a single filetree, it can contain a forest of filetrees. A Btrfs subvolume can be thought of as a POSIX file namespace.
- The top-level subvolume (with Btrfs id 5) (which one can think of as the root of the volume) can be mounted, and the full filesystem structure will be seen at the mount point; alternatively any other subvolume can be mounted (with the mount options subvol or subvolid, for example subvol=@home) and only anything below that subvolume will be visible at the mount point. For example when we map the subvolume `@home` to `/home` directory, only the contents of subvolume `@home` will be visible at the mount point `/home`.

Here's the filesystem layout we'll be implementing:

```
<subvol>        <mountpoint>
@               /
@home           /home
@log            /var/log
@cache          /var/cache
@snapshots      /.snapshots
```

Listing current subvolumes:

```
[root@eos-laptop ~]# btrfs subvolume list /
ID 276 gen 1629 top level 5 path @
ID 277 gen 1629 top level 5 path @home
ID 278 gen 1629 top level 5 path @log
ID 279 gen 910 top level 5 path @cache
```

Manually mount the btrfs partition to see what's inside:

```
# Mount partition /dev/sda2 at /mnt, here we're explicitly mentioning the type as btrfs. It would probably work without it as well!
[root@eos-laptop ~]# mount -t btrfs /dev/sda2 /mnt

[root@eos-laptop ~]# cd /mnt

[root@eos-laptop mnt]# ls
@  @cache  @home  @log

[root@eos-laptop mnt]# ls @
bin  boot  dev  etc  home  lib  lib64  mnt  opt  proc  root  run  sbin  srv  sys  tmp  usr  var

[root@eos-laptop mnt]# ls @cache
ldconfig  man  pacman  private

[root@eos-laptop mnt]# ls @home
pavin

# Unmount
[root@eos-laptop ~]# umount /mnt
```

EndeavourOS creates these subvolumes automatically when choosing to install on a btrfs partition, we'll create the `@snapshots` subvolume manually later.

[Source](https://btrfs.wiki.kernel.org/index.php/SysadminGuide#Subvolumes)

Whew! That was a handful to digest, but don't worry if you don't immediately get it, you'll understand it better once we implement it.

## Copy on Write (CoW)

- The CoW operation is used on all writes to the filesystem.
- This makes it much easier to implement lazy copies, where the copy is initially just a reference to the original, but as the copy (or the original) is changed, the two versions diverge from each other in the expected way.

This is why snapshots can be created so fast and easily in btrfs. A snapshot subvolume is simply a reference to another subvolume.

[Source](https://btrfs.wiki.kernel.org/index.php/SysadminGuide#Copy_on_Write_.28CoW.29)

---

# Installing packages

Update repos and upgrade packages:

```
pacman -Syu
```

Install packages:

```
pacman -S snapper snap-pac grub-btrfs borg borgmatic
```

---

# Configure snapper

Create a snapper config for root subvolume mounted at `/`

```
snapper -c root create-config /
```

Modify the snapper config, the defaults are probably fine.

File: `/etc/snapper/configs/root`

You may create other snapper configs as needed, for example for `/home` but it will not be covered here as we're backing up `/home` offsite

Enable systemd timer to have snapper create a snapshot on each boot:

```
systemctl enable snapper-boot.timer
```

Backup non-btrfs partition `/boot` (EFI system partition) on pacman kernel updates

File: `/etc/pacman.d/hooks/50-bootbackup.hook`

Contents:

```
[Trigger]
Operation = Upgrade
Operation = Install
Operation = Remove
Type = Path
Target = usr/lib/modules/*/vmlinuz

[Action]
Depends = rsync
Description = Backing up /boot...
When = PostTransaction
Exec = /usr/bin/rsync -a --delete /boot /.bootbackup
```

---

# Configure btrfs snapshots

Snapper automatically creates a `.snapshots` directory inside the directory it's snapshotting.
For the above config for root it will be at `/.snapshots`.
But we don't need it as we'll create an empty directory of same name/path for our `@snapshots` subvolume to be mounted at.

Delete the `.snapshots` directory snapper created at `/`

```
rm -rI /.snapshots
```

Create the directory anew

```
mkdir /.snapshots
```

Mount btrfs partition and create new subvolume

```
mount -t btrfs /dev/sda2 /mnt
btrfs subvolume create /mnt/@snapshots
umount /mnt
```

Automatically mount the `@snapshots` subvolume at `/.snapshots`

File: `/etc/fstab`

Contents:

```
# /etc/fstab: static file system information.
#
# Use 'blkid' to print the universally unique identifier for a device; this may
# be used with UUID= as a more robust way to name devices that works even if
# disks are added and removed. See fstab(5).
#
# <file system>             <mount point>  <type>  <options>  <dump>  <pass>
UUID=E199-B52A                            /boot/efi      vfat    umask=0077 0 2
UUID=63745996-340f-4bda-98b1-63af36e6cdae /              btrfs   subvol=/@,defaults,noatime,space_cache,autodefrag,compress=lzo 0 1
UUID=63745996-340f-4bda-98b1-63af36e6cdae /home          btrfs   subvol=/@home,defaults,noatime,space_cache,autodefrag,compress=lzo 0 2
UUID=63745996-340f-4bda-98b1-63af36e6cdae /var/cache     btrfs   subvol=/@cache,defaults,noatime,space_cache,autodefrag,compress=lzo 0 2
UUID=63745996-340f-4bda-98b1-63af36e6cdae /var/log       btrfs   subvol=/@log,defaults,noatime,space_cache,autodefrag,compress=lzo 0 2
UUID=831cc5b9-7c99-459c-8b25-0db765c5ce3b swap           swap    defaults,noatime 0 0
tmpfs                                     /tmp           tmpfs   defaults,noatime,mode=1777 0 0

# Snapshots
UUID=63745996-340f-4bda-98b1-63af36e6cdae  /.snapshots  btrfs  subvol=/@snapshots,defaults,noatime,space_cache,compress=lzo 0 0
```

Mount all partitions listed in `/etc/fstab`:

```
mount -a
```

---

# Create a manual snapshot

Check if everything works by creating a manual snapshot:

```
snapper -c root create -c number -d 'Test snap'
```

Flags (options):

- The first `-c` flag is for specifying the config name, which is `root`. To list all snapper configs: `snapper list-configs`
- The second `-c` flag is for specifying the cleanup algorithm to use, which is `number`. Snapshots older than the number limit specified in `/etc/snapper/configs/root` would be pruned
- The `-d` flag specifies a description for the snapshot

List the snapshots:

```
snapper -c root list
```

List all subvolumes:

```
[root@eos-laptop mnt]# btrfs subvolume list /
ID 276 gen 1987 top level 5 path @
ID 277 gen 1987 top level 5 path @home
ID 278 gen 1987 top level 5 path @log
ID 279 gen 1987 top level 5 path @cache
ID 290 gen 1987 top level 5 path @snapshots
ID 291 gen 349 top level 290 path @snapshots/1/snapshot
```

> We can see the snapshot we just created manually.

Get property of subvolume:

```
[root@eos-laptop mnt]# btrfs property get /.snapshots/1/snapshot
ro=true

[root@eos-laptop mnt]# btrfs property get /
ro=false
```

> By default, snapper creates read-only snapshots. This is preferred as it would prevent us from accidentally modifying any snapshots, one of which may be our last chance at restoring to a functional system state.
> To restore from a snapshot, we create a read-write snapshot from the read-only snapshot (shown later).

---

# Update grub menu with snapshot entries

Generate grub configuration manually once:

```
grub-mkconfig -o /boot/grub/grub.cfg
```

To automatically update grub menu entries on detecting changes in `/.snapshots` directory, the `grub-btrfs` package provides a systemd script:

```
systemctl enable grub-btrfs.path
```

---

# Restoring from a snapshot

Power on or reboot the system.

From the snapshot list, boot into the last functional snapshot. Make a mental note of the snapshot number.

> The system should boot without issues as `/home`, `/var/log` and `/var/cache` are all read-write, only `/` will be read-only.
> If the system does not boot from the snapshot, boot from a live USB.

Make sure we have booted into a read-only snapshot:

```
[root@eos-laptop mnt]# btrfs property get /
ro=true
```

> The aforementioned step is crucial, do not proceed if it's _not_ a read-only snapshot.

Optional: If you don't know the correct snapshot number to restore, read the `info.xml` file associated with each snapshot created by snapper:

```
nano /.snapshots/*/info.xml
```

As an example, we will proceed with restoration of root `/` (subvolume `@`) from snapshot number `33`

Mount the btrfs partition to `/mnt`:

```
mount -t btrfs /dev/sda2 /mnt
```

Rename the `@` subvolume:

```
mv /mnt/@ /mnt/@.broken
```

Create a new `@` subvolume from our chosen snapshot:

```
btrfs subvolume snapshot /.snapshots/33/snapshot /mnt/@
```

> By default, btrfs creates read-write snapshot. To create a read-only snapshot, specify the `-r` flag.

Unmount and reboot into the restored system:

```
umount /mnt
reboot
```

Once rebooted, delete the `@.broken` subvolume:

```
mount -t btrfs /dev/sda2 /mnt
btrfs subvolume delete /mnt/@.broken
umount /mnt
```

Optionally, scrub the filesystem (verifies all data & metadata checksums):

```
btrfs scrub start /
btrfs scrub status /
```

---

# Check and repair btrfs filesystem

Boot from a live USB

List all block devices:

```
lsblk
parted -l
```

Check the unmounted btrfs filesystem:

```
btrfs check /dev/sda2
```

Repairing an unmounted btrfs filesystem (DANGEROUS OPTION, see man page for `btrfs-check`):

```
btrfs check --repair /dev/sda2
```

---

# Setup offsite backups

Install borg and create user on *remote server* with SSH access (in this case it's a Debian server):

```
apt install borgbackup
adduser borg --disabled-password
```

From local machine:

```
mkdir -p /root/.ssh
cd /root/.ssh
ssh-keygen -t rsa
```

Copy the public key to *remote server* location: `/home/borg/.ssh/authorized_keys`.

From local machine:

```
ssh borg@remote-server.tld
mkdir eos-laptop
```

> In the previous command, `eos-laptop` refers to local machine's hostname and `remote-server.tld` refers to remote server's FQDN

Initialize borg repository from local machine:

```
borg init -e repokey borg@remote-server.tld:/home/borg/eos-laptop
```

> Make sure to securely backup the borg repo password as well as the private key created earlier.

Generate borgmatic config from local machine:

```
mkdir -p /etc/borgmatic
generate-borgmatic-config -d /etc/borgmatic/config.yaml
chmod 600 /etc/borgmatic/config.yaml
chown root:root /etc/borgmatic/config.yaml
```

Edit the borgmatic config file `/etc/borgmatic/config.yaml`:

```
location:
    source_directories:
        - /mnt/@.latest
        - /mnt/@home.latest
        - /mnt/@log.latest
        - /mnt/@cache.latest

    repositories:
        - borg@remote-server.tld:/home/borg/eos-laptop

storage:
    encryption_passphrase: "<your-borg-repo-password>"

retention:
    keep_daily: 7
    keep_weekly: 4

hooks:
    before_backup:
        - mount /dev/sda2 /mnt
        - btrfs subvolume snapshot -r /mnt/@ /mnt/@.latest
        - btrfs subvolume snapshot -r /mnt/@home /mnt/@home.latest
        - btrfs subvolume snapshot -r /mnt/@log /mnt/@log.latest
        - btrfs subvolume snapshot -r /mnt/@cache /mnt/@cache.latest

    after_backup:
        - btrfs subvolume delete /mnt/@.latest
        - btrfs subvolume delete /mnt/@home.latest
        - btrfs subvolume delete /mnt/@log.latest
        - btrfs subvolume delete /mnt/@cache.latest
        - umount /mnt

    on_error:
        - btrfs subvolume delete /mnt/@.latest
        - btrfs subvolume delete /mnt/@home.latest
        - btrfs subvolume delete /mnt/@log.latest
        - btrfs subvolume delete /mnt/@cache.latest
        - umount /mnt
```

Validate the borgmatic config file `/etc/borgmatic/config.yaml`:

```
validate-borgmatic-config
```

Perform a backup (this may take a while as it's a first time sync):

```
borgmatic --verbosity 1
```

Setup cron to automatically perform backups.

File: `/etc/cron.d/borgmatic`

Contents:

```
0 3 * * * root /usr/bin/borgmatic --verbosity 1 --syslog-verbosity 1
```

Secure cron file:

```
chmod 600 /etc/cron.d/borgmatic
chown root:root /etc/cron.d/borgmatic
```

List all backups:

```
borgmatic list
```

Get information about backups:

```
borgmatic info
```

Break lock if previous backup was interrupted and no borg processes are running:

```
borgmatic borg break-lock
```

---

# Restore from offsite backups

This section details restoring to a fresh installation of the OS.

Boot into freshly installed system and install borg and borgmatic as stated in the previous section.

List all backups:

```
borgmatic list
```

Mount the latest backup to check files:

```
borgmatic mount --archive latest --mount-point /mnt
```

Unmount:

```
borgmatic umount --mount-point /mnt
```

Extract files to `/root`:

```
borgmatic extract --archive latest --destination /root --progress
```

Rename backup:

```
cd /root
mv mnt backup
```

Reboot into live environment

Mount btrfs partition:

```
mount -t btrfs /dev/sda2 /mnt
```

Rename existing subvolumes:

```
cd /mnt
mv @ @.fresh
mv @home @home.fresh
mv @log @log.fresh
mv @cache @cache.fresh
```

Move and rename backup files:

```
mv /mnt/@.fresh/root/backup/* /mnt
mv @.latest @
mv @home.latest @home
mv @log.latest @log
mv @cache.latest @cache
```

Copy `/boot` dir from `@.fresh` to `@`:

```
cp -a /mnt/@.fresh/boot /mnt/@
```

Edit `/mnt/@/etc/fstab` with UUID of new EFI, btrfs and swap partitions:

```
lsblk
blkid /dev/sda1
blkid /dev/sda2
blkid /dev/sda3
```

Reboot back into OS.

Edit `/etc/default/grub` with UUID of new swap partition (for hibernation resume) and run:

```
grub-mkconfig -o /boot/grub/grub.cfg
```

Reboot server.

---

Here's a cute picture of _Butters_ from _South Park._

![Butters from South Park looks pleased at hearing about btrfs](https://i.imgur.com/UvG9wCq.png)

<!-- THE END-->
