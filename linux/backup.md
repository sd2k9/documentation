Local Encrypted Backup on External Drive
=========================================


### TOC

1. TOC
{:toc}


### Introduction

To store local backups of your data an external drive is usually sufficient.  
To protect the sensitive backup data from others, this backup should be better
encrypted.

So let's do it with the help of btrfs, even when you don't use btrfs (why not?)
as your working file system :-)

For large files where it is acceptable to only backup the latest version
another approach is used.  
This is usually the case for virtual machine disk image files.


### Setup

##### btrfs-backup
1. Clone the [btrfs-backup](https://github.com/3coma3/btrfs-backup) script repository
   - Last verified version within this guide: v1.4.3
1. Put the backup script to system path
   ```
   sudo cp -v backup /usr/local/bin/btrfs-backup
   sudo chown root:root /usr/local/bin/btrfs-backup
   sudo chmod u+rwx,og=rx /usr/local/bin/btrfs-backup
   ```
1. Put configuration file [backup-full.conf](backup/backup-full.conf) to `/etc/btrfs-backup/`
   - Update _HOMEUSER
   - Tweak filters expression to include/exclude other parts of the file system

##### External Disk
1. Take a external drive and partition it appropriately, e.g. with `cfdisk` or any
   GUI application of your choice
   - Placeholder of the partition device file: DISKPART
1. Create encrypted file sytem and open it  
   `sudo cryptsetup luksFormat --type luks2 DISKPART   # Enter a safe password here`
1. Open encrypted file sytem  
   `sudo cryptsetup open DISKPART backup_offline_crypt`
1. Create BTRFS file system  
   `sudo mkfs -t btrfs -L backupdisk /dev/mapper/backup_offline_crypt`
1. Mount the file system  
   `sudo mount -t btrfs -o async,noatime,nodev,nosuid,compress=zlib,noexec /dev/mapper/backup_offline_crypt /media/backup_offline`
1. Create directories  
   `sudo mkdir /media/backup_offline/virtualbox-backup`
1. Unmount the file system again  
   `sudo umount /media/backup_offline`
1.  Close encryption again  
    `sudo cryptsetup close backup_offline_crypt`

##### Backup Runner Script
1. Put the backup runner script [do-backup](backup/do-backup) to system path
   ```
   sudo cp -v do-backup /usr/local/bin/
   sudo chown root:root /usr/local/bin/do-backup
   sudo chmod u+rwx,og=rx /usr/local/bin/do-backup
   ```
1. Tweak the script to your needs
   - See comments "UPDATE: "

##### Disable automatic mounting of encrypted devices
1. Add udev-rule: `/etc/udev/rules.d/udisks2-ignore-crypto.rules`
   ```
   # Do not automatically ask key for encrypted devices
   # They are usually used by scripts which supply their own key
   ENV{ID_FS_USAGE}=="crypto", ENV{UDISKS_AUTO}="0"
   ```
1. If you want to use the default behaviour instead, you have to update the
   [backup runner script](backup/do-backup) accordingly


### Running
1. Call `sudo do-backup` with the wanted options


### Final Remarks
- To enhance security you can also use gnupg to store an (long) intermediate passphrase
  which is used to encrypt the external drive
  - For unlocking gnupg you just use your "local" (shorter) password
  - Let me know when you're interested about this option, I can document the setup and usage
