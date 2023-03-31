Windows in Linux with KVM/QEMU
==============================

### TOC

1. TOC
{:toc}


### Introduction

There are a lot of guides out there explaining how to run
Windows 10 in a Linux VM. Unfortunately all the guides I found did
lack a few pieces of the puzzle, which I had to find somewhere else.

That's why I (hopefully) collected everything here, for the next to use.

Base System/tested with
- Debian 10 (Buster) Stable with Backports enabled
- Updated for Debian 11 (Bullseye)

The guides I used as starting point were
- [https://getlabsdone.com/install-windows-10-on-ubuntu-kvm/](https://getlabsdone.com/install-windows-10-on-ubuntu-kvm/)
- [https://dennisnotes.com/note/20180614-ubuntu-18.04-qemu-setup/](https://dennisnotes.com/note/20180614-ubuntu-18.04-qemu-setup/)


### Installation

Packages to install
- virt-manager
  - From buster-backports
  - Only this recommendations
    - libvirt-daemon-system
      - All recommend
    - gir1.2-spiceclientgtk-3.0
- qemu-system-x86 # Satisfies recommendation of libvirt-daemon
  - From buster-backports
  - All recommended, but not ovmf
  - This should install these packages:  
    qemu-system-common qemu-system-data qemu-system-gui qemu-system-x86 qemu-utils


### System Setup

Add user to libvirt group
- adduser YOUR_USERNAME libvirt
  - The group name is configured in /etc/libvirt/libvirtd.conf:unix_sock_group
- Relogin

When installed dnsmasq and not used, disable it
- Not needed for dnsmasq-base, which is installed as dependency
- systemctl stop dnsmasq
- systemctl disable dnsmasq
- /etc/default/dnsmasq
  ```
  # Disabled, just needed for qemu/kvm
  # ENABLED=1
  ENABLED=0
  ```
- /etc/dnsmasq.conf
  ```
  # Never forward plain names (without a dot or domain part)
  # Enabled
  domain-needed
  # Never forward addresses in the non-routed address spaces.
  # Enabled
  bogus-priv
  ```

If you don't want to run LibVirt all the time use the helper scripts to
start virtualisation support on demand
1. Install root start/stop script [kvm_qemu/kvm_services](kvm_qemu/kvm_services) to  
  /usr/local/sbin/kvm_services
1. Allow sudo invocation without password  
   YOUR_USERNAME   ALL = NOPASSWD: /usr/local/sbin/kvm_services
1. Call once to stop all system services  
   sudo /usr/local/sbin/kvm_services stop
   sudo /usr/local/sbin/kvm_services disable
1. Put user control script [kvm_qemu/kvm](kvm_qemu/kvm)
   to a location excutable from the user, e.g. ~/bin/kvm
   - You'll need the [bash-functions](https://github.com/sd2k9/tools/blob/master/bash-functions) too
1. When using firehol as your firewall, allow routing to enjoy internet connection by
   adding the content of the file [kvm_qemu/firehol-addon.conf](kvm_qemu/firehol-addon.conf)
   to your /etc/firehol/firehol.conf
   - Make sure that `FIREHOL_RULESET_MODE="accurate"`
     is set /etc/firehol/firehol-defaults.conf
     - Otherwise DHCP server traffic for KVM/Windows machine
       may be blocked and no IP address can be assigned
1. Start everything with: `kvm start`
1. Stop with: `kvm stop`
1. Show more commands with: `kvm --help`


Configuration changes
- /etc/libvirt/lxc.conf
  ```
  # Enable default confinement
  security_default_confined = 1
  security_require_confined = 1
  ```
- /etc/libvirt/qemu.conf
  ```
  # Compress save images
  save_image_format = "bzip2"
  snapshot_image_format = "bzip2"
  # Set process name
  set_process_name = 1
  ```

### Setup VM and Install Windows 10
1. Download Windows 10 image
  - Source: [https://www.microsoft.com/de-de/software-download/windows10ISO](https://www.microsoft.com/de-de/software-download/windows10ISO)
  - For non-network install updates can be downloaded manually from
    [https://www.catalog.update.microsoft.com](https://www.catalog.update.microsoft.com)
1. Download Guest drivers
   - These files and the Windows installer ISO must be in a location accessible by user libvirt-qemu
   - [https://fedorapeople.org/groups/virt/virtio-win/direct-downloads/stable-virtio/virtio-win.iso](https://fedorapeople.org/groups/virt/virtio-win/direct-downloads/stable-virtio/virtio-win.iso)
1. Create VM with virt-manager [--debug]
   1. When QEMU/KVM not visible, add with File/Add Connection
   1. Name: Windows_10
   1. Storage: Create custom storage
      - Green "plus" bottom-left: Create storage pool
        - Name: kvm_storage
        - Type: dir
        - Path: /opt/kvm
      - Create new volume in this pool: green "plus" near "Volumes"
        - qcow2 Format
      - Attention: Storage file must be accessible by user libvirt-qemu
   1. Select "Customize before install"
      - SATA Disk 1/Avanced Options/Disk bus
        - SATA --> VirtIO
        - Optional: Cache mode to writeback for better performance (and less safety)
      - NIC/Device model: e1000e --> virtio
      - Mount VirtIO driver ISO
        - Add Hardware: Storage
          - Manage: /Path/to/virtio-win.iso
          - Device Type: CDROM
      - Boot options
        - Enable Boot Menu
        - Order: CDROM 1 - Disk 1 - CDROM 2
      - CPUs
        - Current Allocation: 4
        - Copy host CPU configuration
        - Manually set CPU topology: 1 socket, 2 cores and 2 threads
      - Memory: 8192 MiB
      - Add Hardware / Channel
        - For clipboard sharing with spice-guest-tools
        - Name: "com.redhat.spice.0"
        - Device type: "Spice agent (spicevmc)"
1. Start network before machine start in virt-manager
   - Win10/Edit/Connection Details/Virtual Networks
   - Start/Stop
1. Start Installation Windows 10
   - Load file driver from CDROM 2: viostor/w10/amd64
   - Load network driver from CDROM 2: drive/NetKVM/w10/amd64
   - Installation Notes
     - First account created during installation has always local admin permissions
     - Create user account without admin rights (Settings/Accounts)
     - Uninstall apps: Settings/Apps/Apps&Features  + Optional features
   - After Windows 10 installation is done, install the VirtIO tools and drivers
     - Start/Device Manager/Other devices/*
     - Search CDROM 2 for drivers
   - Install SPICE Guest tools from
     - [https://www.spice-space.org/download/windows/spice-guest-tools/spice-guest-tools-latest.exe](https://www.spice-space.org/download/windows/spice-guest-tools/spice-guest-tools-latest.exe)
1. Afterwards: Disable boot menu and boot from CDROM
   - Virtual Machine/View/Details (or 2. button)
   - Boot options
     - Disable Boot Menu
     - Order:  Disk 1
   - Remove CDROM 1 + 2
1. virt-manager Setup
   - Win10/Edit/Connection Details
     - Storage
       - Remove Autostart/On Boot for all
       - Delete default - can also delete /var/lib/libvirt/images
       - Delete tmp
       - Delete installation media
     - Virtual Networks
       - Remove Autostart/On Boot for all
   - `virsh --connect=qemu:///system edit Windows_10`
     - Source: [https://blog.geierb.de/mehr-geschwindigkeit-bei-windows-10-in-libvirtkvm/](https://blog.geierb.de/mehr-geschwindigkeit-bei-windows-10-in-libvirtkvm/)
     - Increase video memory:  video / model type='qxl' / vgamem='16384' --> vgamem='32768' or '65536'
     - Enable HPET:  clock offset='localtime' / timer name='hpet' present='no' --> present='yes'
1. Networking NAT
   - Force IP address to 192.168.122.96 for Windows 10 machine, because thats
     what the firewall rules are written for
     ```
     virsh --connect=qemu:///system  net-edit default
     # Add in dhcp section, update MAC address and name accordingly
     <host mac='52:54:00:5f:ff:8e' name='win10' ip='192.168.122.96' />
     ```
   - Check for Leases and addresses
     ```
     virsh --connect=qemu:///system  net-dhcp-leases default
     virsh --connect=qemu:///system  domifaddr Windows_10 --full
     ```
   - Make sure forwarding is enabled
     - cat /proc/sys/net/ipv4/ip_forward # 1
   - Host IP: 192.168.122.1 (interface virbr0)
1. SSH Server in Windows
   - For simple file sharing
     - Use e.g. Samba for more comfortable sharing
   - Settings/Apps/Apps&Features/Optional Features/Add a feature/OpenSSH Server
   - Start Server
     - Power Shell / Right-Click / Run As Administrator
     - Start-Service sshd
       ```
       # Status
       Get-Service sshd
       # OPTIONAL but recommended:
       Set-Service -Name sshd -StartupType 'Automatic'
       # Confirm the Firewall rule is configured. It should be created automatically by setup.
       Get-NetFirewallRule -Name *ssh*
       # There should be a firewall rule named "OpenSSH-Server-In-TCP", which should be enabled
       # If the firewall does not exist, create one
       New-NetFirewallRule -Name sshd -DisplayName 'OpenSSH Server (sshd)' -Enabled True -Direction Inbound -Protocol TCP -Action Allow -LocalPort 22
       ```
   - Access: ssh username@192.168.122.96
1. Optional: Access host's CD/DVD drive or ISO images from Windows_10 guest
   - Power down VM
   - In virt-manager Add Hardware: Storage
     - Device type: CDROM


### Configuration Files

- /etc/libvirt/
  - Connection aliases: libvirt-admin.conf, libvirt.conf
  - Machine setting: /etc/libvirt/qemu
- virt-manager: dconf (accessible with dconf-editor) under /org/virt-manager/virt-manager/
- /var/lib/libvirt
  - Snapshots: /var/lib/libvirt/qemu/snapshot/


### virt-manager Usage

- Mount CDROM/DVD or ISO images
  - In running VM: View/Details
  - In the CDROM setup select the ISO or device file
    - The directory is automatically added to the storage pool
    - Delete afterwards when this is not wanted  
      virt-manager Main Screen / Edit / Connection Details / Storage
  - ISO files can be also directly loaded from a
    (newly created or existing) storage pool
    - Can be used as shortcut
- Redirect USB device to client: Virtual Machine / Redirect USB Device
- No background operation: --no-fork
- Show running VM  
  `virt-manager --connect=qemu:///system --show-domain-console VMNAME`



### Virsh Command Line Management Interface

Selected commands

- Edit XML configuration file  
    virsh --connect=qemu:///system edit Windows_10
- List/Start of domains (i.e. virtual machines)  
    virsh --connect=qemu:///system list --all  
    virsh --connect=qemu:///system start Windows_10  
    See [virt-manager Usage](#virt-manager-usage) for showing the VM also
- Network Status  
    virsh --connect=qemu:///system net-list
- Start/Stop/Edit default network connection  
    virsh --connect=qemu:///system net-start default  
    virsh --connect=qemu:///system net-destroy default  
    virsh --connect=qemu:///system  net-edit default
- List network interfaces for domain  
    virsh --connect=qemu:///system domiflist Windows_10
- Network cable plug status  
    virsh --connect=qemu:///system domif-getlink Windows_10 vnet0
- Connect/Disconnect network cable  
    virsh --connect=qemu:///system domif-setlink Windows_10 vnet0 up  
    virsh --connect=qemu:///system domif-setlink Windows_10 vnet0 down
- Check for Leases and addresses  
    virsh --connect=qemu:///system  net-dhcp-leases default  
    virsh --connect=qemu:///system  domifaddr Windows_10 --full


### Further Commands and Documentation

- Libvirt Schemas: /usr/share/libvirt/schemas
  - Validate with virt-xml-validate
  - Automatically done when editing XML files with virsh
- Domain XML file: [https://libvirt.org/formatdomain.html](https://libvirt.org/formatdomain.html)
  - All other XML files: [https://libvirt.org/format.html](https://libvirt.org/format.html)


### Windows XP Notes

- Installation not tried, here are some notes for reference
-  If the OS supports multiple CPUs, then you should install it with 2 CPUs
   (or more) available. WinXP will install with either 1 or 2 CPU support in
   the HAL. If you install with 1 CPU/Core, then you are basically stuck with
  1 CPU until a reinstall. That isn’t really true, but the steps to enable a
  second CPU is non-trivial. Reinstalling the OS is much easier.
- https://wiki.linuxmuster.net/archiv/anwenderwiki:virtualisierung:kvm:kvm_winxp
- https://pve.proxmox.com/wiki/Windows_XP_Guest_Notes
- https://wiki.gentoo.org/wiki/QEMU/Windows_guest


### Direct QVM Usage

When you're using QVM directly (should not be needed for this guide)
here some notes.

Invocation
- Running Windows 95/98 System  
  ```
  qemu-system-x86_64 \
    -hda /home/virtualbox/kvm/Win_98.qcow2 \
    -audiodev pa,id=snd \
     -machine pcspk-audiodev=snd \
    -device ac97,audiodev=snd \
    -m size=512M \
    -vga cirrus \
    -parallel none \
    -rtc base=localtime \
    -boot c
  ```
- Start [Boot] with CDROM mounted, add  
  `-cdrom win98se.iso [-boot d]`
- Install with boot disk  
  `-boot a -fda disk01.img -cdrom Win95_OSR25.iso`
- Better mouse performance without mouse capture  
  `-usb -device usb-tablet`

Reduce image file size by unsparsing and compressing contents
- Attention: Snapshots are lost, so remove beforehand
  - Or afterwards from directory /var/lib/libvirt/qemu/snapshot/
- Compression is read-only, that is file modifications or new files
  are uncompressed
- `qemu-img convert -O qcow2 -c -p -m 8 image.qcow2_backup image.qcow2`
  - Compress: `-c`
  - Progress bar: `-p`
  - 8 parallel instances: `-m 8`
- Automation with `virt-sparsify` from paket libguestfs-tools
- More information
  - https://pve.proxmox.com/wiki/Shrink_Qcow2_Disk_Files
  - https://kofler.info/wie-ich-ein-qcow2-image-auf-ein-drittel-geschrumpft-habe/

Snapshots
- Information about disk file  
  `qemu-img info --backing-chain diskimage.qcow2`
- Manage snapshots  
  `qemu-img snapshot CMD diskimage.qcow2`
   - List:   `-l`
   - Create: `-c NAME`
   - Restore (Apply): `-a NAME`
   - Delete: `-d NAME`

Change CDROM in running system
- Src: [https://www.linux-kvm.org/page/Change_cdrom](https://www.linux-kvm.org/page/Change_cdrom)
- Use Monitor interface (Control-Alt-2 or menu)
- `info block`
- `eject ide1-cd0`
- `change ide1-cd0 cdromimage.iso`

Networking Guides
- https://wiki.qemu.org/Documentation/Networking
- https://en.wikibooks.org/wiki/QEMU/Networking

More monitor commands
- In monitor: help
- info network: Network setup
- Outdated: [https://en.wikibooks.org/wiki/QEMU/Monitor](https://en.wikibooks.org/wiki/QEMU/Monitor)

### Errors

- When a VM does not start with an error like  
  `apparmore profile libvirt-XXX is missing`  
  and there really is no such file in the directory `/etc/apparmor.d/libvirt/`
  then possibly the invocation of virt-aa-helper failed
  - This can happen when the VM XML file is broken in a way that virt-xml-validate
    doesn't detect the error
  - Try reproducing this behaviour by bisecting the XML file and feed it to
    virt-aa-helper manually
  - Call: `/usr/lib/libvirt/virt-aa-helper --create --uuid libvirt-UUID_FROM_XML_FILE < /etc/libvirt/qemu/Windows_10-COPY.xml`
