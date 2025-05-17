Windows (or Linux) in Linux with KVM/QEMU
=========================================

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
1. Install root start/stop script [kvm\_qemu/kvm_services](kvm_qemu/kvm_services) to  
  /usr/local/sbin/kvm_services
1. Allow sudo invocation without password  
   YOUR_USERNAME   ALL = NOPASSWD: /usr/local/sbin/kvm_services
1. Call once to stop all system services  
   sudo /usr/local/sbin/kvm_services stop
   sudo /usr/local/sbin/kvm_services disable
1. Put user control script [kvm\_qemu/kvm](kvm_qemu/kvm)
   to a location excutable from the user, e.g. ~/bin/kvm
   - You'll need the [bash-functions](https://github.com/sd2k9/tools/blob/master/bash-functions) too
1. Start everything with: `kvm start`
1. Stop with: `kvm stop`
1. Show more commands with: `kvm --help`
1. Optional create VM starter script, based on [kvm\_qemu/vm\_start](kvm_qemu/vm_start)
   to a location excutable from the user, e.g. ~/bin/kvm
   - You'll need the [bash-functions](https://github.com/sd2k9/tools/blob/master/bash-functions) too


Configuration changes
- /etc/libvirt/lxc.conf
  ```
  # Enable default confinement
  security_default_confined = 1
  security_require_confined = 1
  ```
- /etc/libvirt/qemu.conf
  ```
  # Default security driver
  security_driver = "apparmor"
  # Compress save images
  save_image_format = "bzip2"
  snapshot_image_format = "bzip2"
  # Set process name
  set_process_name = 1
  ```

### Setup VM and Install Windows 10/11
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
        - Current Allocation: 6
        - Copy host CPU configuration
      - Memory: 8192 MiB
      - Add Hardware / Channel
        - For clipboard sharing with spice-guest-tools
        - Name: "com.redhat.spice.0"
        - Device type: "Spice agent (spicevmc)"
      - Use QXL as Video driver, otherwise "Auto resize VM with window" (Enable in virt-manager menu View / Scale Display)
      - Increase video memory to support higher resolutions: XML  video / model type='qxl' / vgamem='16384' --> vgamem='65536'
1. Enable (U)EFI boot
   1. When Importing from Virtualbox required with setting "System/Enable EFI"
   1. Install dependency ovmf
   1. During Setup
      - Overview / Hypervisor Details
      - Chipset: Q35
      - Firmware: UEFI x86_64: /usr/share/OVMF/OVMF_CODE_4M.fd
1. Add TPM (Trusted Platform Module) 2.0 device
  - Required for Windows 11
  - Install packages: swtpm swtpm-tools
    - Note for Debian 12: May require "sudo chown -R tss /var/lib/swtpm-localca" to start
  - systemctl restart libvirtd.service # Needed?
  - Add Hardware / TPM: Emulated, Model CRB, Version 2.0
1. Setup [Networking](#networking)
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
1. Install KVM Virtio driver and guest tools
   - https://pve.proxmox.com/wiki/Windows_VirtIO_Drivers
   - From ISO: https://fedorapeople.org/groups/virt/virtio-win/direct-downloads/stable-virtio/virtio-win.iso
     - Or https://fedorapeople.org/groups/virt/virtio-win/direct-downloads/latest-virtio/virtio-win.iso
     - virtio-win-gt-x64, virtio-win-guest-tools
   - Or use direct installer
     - https://fedorapeople.org/groups/virt/virtio-win/direct-downloads/stable-virtio/
     - Or https://fedorapeople.org/groups/virt/virtio-win/direct-downloads/latest-virtio/
     - virtio-win-gt-x64.msi, virtio-win-guest-tools.exe
   - Not needed anymore: [SPICE Guest tools](https://www.spice-space.org/download/windows/spice-guest-tools/spice-guest-tools-latest.exe)
1. virt-manager Setup
   - Win10/Edit/Connection Details
     - Storage
       - Remove Autostart/On Boot for all
       - Delete default - can also delete /var/lib/libvirt/images
       - Delete tmp
       - Delete installation media
     - Virtual Networks
       - Remove Autostart/On Boot for all
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
1. Implement  [KVM Tuning](#kvm-tuning)


### Linux Guest
1. See also [Setup VM and Install Windows 10/11](#setup-vm-and-install-windows-10-11) and
   [KVM Tuning](#kvm-tuning)
1. Settings / Video / Model=Virtio, 3D acceleration when supported (Linux yes, Windows no)
1. Kernel
  - Use linux-image-virtual-*
  - linux-image-kvm not booting


### KVM Tuning

##### General Tuning
1. Add `<ioapic driver='kvm'/>` to your `<features>` section
1. Change Disk Controller from SATA to VirtIO
   1. Add New Virtual Hardware / Controller / SCSI / VirtIO SCSI
   1. Add New Virtual Hardware / Storage / VirtIO
   1. Disk controller for VirtIO
      after the line `<controller type='scsi' ...>`
      add "driver" line:
      `<driver iothread='1'/>`
   1. SATA Disk 1/Avanced Options/Disk bus: SATA --> VirtIO
1. Disk performance
   - Cache mode to "writeback" for better performance (and less safety)
   - Discard mode to "unmap" for automatically shrinking the disk image when space is freed
1. Dedicate I/O Threads
   - After the first line `<domain ...>`, add "iothreads" line:
     `<iothreads>1</iothreads>`
1. Enabling multi-queue virtio-scsi and virtio-net
   - Where N is the total number of vCPUs
   - `<controller type="scsi" index="0" model="virtio-scsi">`
     - Add to or create "driver": `<driver queues="N"/>`
     - `</controller>`
   - `<interface type="network">`  
     `<model type="virtio"/>`
     - Add to or create driver": `<driver queues="N"/>`
     - `</interface>`
1. [Increase the Transmit Queue Length for virt-io Interfaces](https://publicdoc.rbbn.com/display/SBXDOC121/KVM+Performance+Tuning)
   - After VM startup: `sudo /usr/local/sbin/kvm_services nettxlen`
1. CPUs
   - Host CPU Passthrough
     - Can set in virt-manager as "copy host CPU configuration"
     - XML: `<cpu mode="host-passthrough"/>`
  - Do not "Manually set CPU topology"
1. Add timers to reduce CPU usage
   - `<clock ...>`
   - Add or enable  
     `<timer name='hpet' present='yes'/>`  
     `<timer name='hypervclock' present='yes'/>`
   - `</clock>`
1. Hugepages
   - Enable automatic use of transparent hugepages until next rebooot
      `echo 'always' > /sys/kernel/mm/transparent_hugepage/enabled`  
      `echo 'always' > /sys/kernel/mm/transparent_hugepage/defrag`
  - Persistent
    - Install package sysfsutils
    - File: /etc/sysfs.conf  
      ```
      ###################################################################
      #
      # Enable transparent hugepages and memory defragmentation
      kernel/mm/transparent_hugepage/enabled = always
      kernel/mm/transparent_hugepage/defrag  = always
      ```
  - Has downsides, e.g. defragmenting RAM pages can stall the system
  - Check the amount of memory used by THP (or HugePages):  
    `grep -i huge /proc/meminfo`
  - THP monitoring: `egrep 'trans|thp' /proc/vmstat`
  - Check THP usage per process  
    `awk  '/AnonHugePages/ { if($2>4){print FILENAME " " $0; system("ps -fp " gensub(/.*\/([0-9]+).*/, "\\1", "g", FILENAME))}}' /proc/*/smaps`
  - When using static hugepages instead (pre-allocated), add the following XML
    - `<memoryBacking> <hugepages/> </memoryBacking>`
    - Not needed for transparent hugepages


##### Windows-Only Tuning
1. Change Disk Controller from SATA to VirtIO after installation
   - Add VirtIO controller and new temporary VirtIO storage device
     as described in "Change Disk Controller from SATA to VirtIO"
     - The new disk must not be formatted, it just triggers installation
       of the required device drivers
     - Check in device manager or Start / Right-Click / Disk Management
       that the VirtIO storage device is recognized
   - Change the boot disk from SCSI to VirtIO disk
     - May need to remove and re-add it
   - Remove temporary VirtIO storage drive
1. Enabling Hyper-V enlightenments  
    Add/Modify the following `<hyperv>` sub-section to the `<features>` section of the XML:  
    ```
    <features>
    [...]
      <hyperv>
        <relaxed state='on'/>
        <vapic state='on'/>
        <spinlocks state='on' retries='8191'/>
        <vendor_id state='on' value='KVM Hv'/>
        <vpindex state='on'/>
        <runtime state='on' />
        <synic state='on'/>
        <stimer state='on'>
          <direct state='on'/>
        </stimer>
        <frequencies state='on'/>
        <reset state='on'/>
        <tlbflush state='on'/>
        <reenlightenment state='on'/>
        <ipi state='on'/>
        <evmcs state='on'/>               <!-- Only for Intel! Remove for AMD processors -->
      </hyperv>
    [...]
    </features>
    ```
1. Configure VirtIO Network driver parameters
   - In Windows VM Device Manager
   - `Logging.Enable = Disable`
   - After every update/reinstall of network driver


### Configuration Files

- /etc/libvirt/
  - Connection aliases: libvirt-admin.conf, libvirt.conf
  - Machine setting: /etc/libvirt/qemu
- virt-manager: dconf (accessible with dconf-editor) under /org/virt-manager/virt-manager/
- /var/lib/libvirt


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
- Don't capture window manger hotkeys (like keepassx, move windows etc) in virt-manager
  - One time: Press Ctrl+Alt to release the keyboard/mouse from the guest
    - Configure key in virt-manger / Edit / Preferences / Console / Grab keys
  - Permanently not possible: [Issue](https://gitlab.com/virt-viewer/virt-viewer/-/issues/72)
- Disable Clipboard Sharing
  - Is enabled by default in both directions
  - Edit the guest definition XML: virsh edit $GUEST
    domain/devices/graphics
  - Or virt-manger / View / Details / Display Spice / XML
  -  domain/devices/graphics, or through virt-manager’s View > Details > Display Spice > XML
  - Add or update an element `<clipboard copypaste="no"/>` under the `<graphics>` node



### Virsh Command Line Management Interface

Selected domain (i.e. virtual machine) commands

- Edit XML configuration file  
    virsh --connect=qemu:///system edit Windows_10
- Show XML configuration file  
    virsh --connect=qemu:///system dumpxml Windows_10
- List of domains  
    virsh --connect=qemu:///system list --all  
- Start domains  
    virsh --connect=qemu:///system start Windows_10  
    See [virt-manager Usage](#virt-manager-usage) for showing the VM also
- Change device configurations of a running domain  
  virsh --connect=qemu:///system update-device Windows_10 FILE.xml --live
- Send keycode sequences to domain  
  virsh --connect=qemu:///system send-key Windows_10 keycodes


Selected network commands

- Network Status  
    virsh --connect=qemu:///system net-list [--all]
- Start/Stop/Edit default (or other by name) network connection  
    virsh --connect=qemu:///system net-start default  
    virsh --connect=qemu:///system net-destroy default  
    virsh --connect=qemu:///system net-edit default
- List network interfaces for domain  
    virsh --connect=qemu:///system domiflist Windows_10
- Network cable plug status  
    virsh --connect=qemu:///system domif-getlink Windows_10 vnet0
- Connect/Disconnect network cable  
    virsh --connect=qemu:///system domif-setlink Windows_10 vnet0 up  
    virsh --connect=qemu:///system domif-setlink Windows_10 vnet0 down  
    These commands are failing in Debian 12 Bookworm, see below for Woraround
- Check for Leases and addresses  
    virsh --connect=qemu:///system net-dhcp-leases default  
    virsh --connect=qemu:///system domifaddr Windows_10 --full

Selected storage commands

- Configuration directory: /etc/libvirt/storage
- List all pools  
   virsh  --connect=qemu:///system pool-list --all
- Remove a pool  
   virsh  --connect=qemu:///system pool-destroy POOLNAME  
   virsh  --connect=qemu:///system pool-undefine POOLNAME

Selected device commands
- Attach or detach a host device to {configuration file|running} VM  
  ```
  virsh --connect=qemu:///system {detach-device|attach-device} Windows_10 {--config|--live} /dev/stdin <<END
  XML ADDITION-OR-DELETION
  END
  ```

### Networking

##### Bridge Networking
1. Appear as own host on network
1. There are a lot of guides out there using older and more complicated and manual approaches
   - In 2025 you can let lbivirt do the work with the following simple steps.
   - Complete GUI usage should also be possible, confirm generated XML with the snippets below.
1. Use a macvtap device, which directly attaches to the NIC, configuration within libvirt
   - Create multiple macvtap devices for multiple VMs
1. Create definition, update interface for your network link name
   ```
   virsh --connect=qemu:///system net-define --validate network.xml
     <network>
        <name>macvtap0</name>
        <forward mode="bridge">
           <interface dev="enx089204252244"/>
        </forward>
     </network>
   ```
1. In virt-manager add to VM / View (Hardware) Details / Directly edit XML for NIC Hardware
   ```
   <interface type="network">
     <source network="macvtap0"/>
     <model type="virtio"/>
   </interface>
   ```
1. Add Libvirt/KVM interface to the firehol firewall script
   ```
   # Add Libvirt/KVM  to the World interface
   interface "... virbr+" World
      # The default policy is DROP. You can be more polite with REJECT.
      policy reject
      # The following means that this machine can REQUEST anything
      client all accept
      # ...
   ```
1. Execute before VM bring-up  
   virsh --connect=qemu:///system net-start macvtap0
1. Optional cleanup after VM termination  
   virsh --connect=qemu:///system net-destroy macvtap0
1. Reference documentation
   - Use libvirt approach described above instead
   - Manual create bridge
     - ip link add dev br0 type bridge
     - ip link set dev enx089204252244 master br0
     - ip link set br0 up
     - Remove an interface from a bridge: ip link set eth1 nomaster
     - Delete a bridge: ip link delete bridge_name type bridge
   - Manual create macvtap/macvlan device
     - ip link add link enx089204252244 name macvtap0 type macvtap
     - ip link set macvtap0 up

##### NAT
1. Routes all traffic over the host, which acts as DHCP server and router
  - No direct access to guest from the outside world
1. When using firehol, allow routing to enjoy internet connection by
   adding the content of the file [kvm_qemu/firehol-addon-nat.conf](kvm_qemu/firehol-addon-nat.conf)
   to your /etc/firehol/firehol.conf
   - It also overrides `FIREHOL_RULESET_MODE` to `accurate"`,
     otherwise DHCP server traffic for KVM/Windows machine
     is blocked and no IP address can be assigned.
1. Force IP address to 192.168.122.96 for Windows 10 machine, because thats
   what the firewall rules are written for
   ```
   virsh --connect=qemu:///system  net-edit default
   # Add in dhcp section, update MAC address and name accordingly
   <host mac='52:54:00:5f:ff:8e' name='win10' ip='192.168.122.96' />
   ```
1. Check for Leases and addresses
   ```
   virsh --connect=qemu:///system  net-dhcp-leases default
   virsh --connect=qemu:///system  domifaddr Windows_10 --full
   ```
1. Make sure forwarding is enabled
   - cat /proc/sys/net/ipv4/ip_forward # 1
1. Host IP: 192.168.122.1 (interface virbr0)

##### Host-Only Network
1. Over this network only the host and guests can communicate.
1. Create definition, update interface for your network link name
   ```
   virsh --connect=qemu:///system net-define --validate network.xml
   <network>
    <name>hostonly</name>
    <bridge name="virbr1"/>
    <port isolated='yes'/>    <!-- Isolate guests from each other -->
    <ip address="192.168.56.1" netmask="255.255.255.0"> <!-- IP of the Host-facing bridge -->
      <dhcp>  <!-- Enable DHCP on Guest-facing network, otherwise assign static IPs by yourself -->
        <range start="192.168.56.2" end="192.168.56.254"/>
      </dhcp>
    </ip>
   </network>
   ```
1. In virt-manager add new NIC to VM / View (Hardware) Details / Directly edit XML for NIC Hardware
   ```
   <interface type="network">
     <source network="hostonly"/>
     <model type="virtio"/>
   </interface>
   ```
1. Show XML again, now a MAC should be assigned. Use in next step
1. Add static IP based on MAC address assigned
   ```
   virsh --connect=qemu:///system net-edit hostonly
   Inside <dhcp> tag
      <host mac='52:54:00:5f:ff:8e' name='tutn-5tfkpn3' ip='192.168.56.3' /
  ```
1. Add Libvirt/KVM interface to the firehol firewall script
   ```
   # Add Libvirt/KVM  to the World interface
   interface "... virbr+" World
      # The default policy is DROP. You can be more polite with REJECT.
      policy reject
      # The following means that this machine can REQUEST anything
      client all accept
      # ...
   ```
1. Enable DHCP Support in firehol firewall script
   ```
   # Add before virbr+ World interface
   # Libvirt/KVM interface which needs special attention due to DHCP
   if [[ "$FIREHOL_USER_FEATURES" =~ ^kvm ]]; then
     interface "virbr+" VirtualDhcp
       # Continue processing after Dhcp Rules
       policy return
       # Host machine is DHCP server
       # Standard DHCP rule is not working, dunno why: server dhcp accept
       server custom virdhcp udp/67 68 accept
   fi
   ```
1. Execute before VM bring-up  
   virsh --connect=qemu:///system net-start hostonly
1. Optional cleanup after VM termination  
   virsh --connect=qemu:///system net-destroy hostonly


### Host Device Forwarding

Dynamic USB Device Attachment
- Select command in running VM window menu to attach USB device from host

Manual add USB device
1.  In Virt-Manager VM configuration: Add Hardware / USB Host Device
1.  You can't forward an USB hub and it's devices.
    - If you forward the hub then still the devices plugged into it are catched by the host
1. XML for adding USB device by vendor and product ID  
   ```
   <hostdev mode='subsystem' type='usb' managed='yes'>
    <source startupPolicy='optional'>
      <vendor id='0x1234'/>
      <product id='0xbeef'/>
    </source>
    <boot order='2'/>
   </hostdev>
   ```
1. XML for adding USB device by bus and device ID  
   ```
   <hostdev mode='subsystem' type='usb'>
    <source startupPolicy='optional'>
      <address bus='1' device='1'/>
    </source>
   </hostdev>
   ```

PCI Forwarding
- Can use for example with USB host when USB device change it's type during operation.
  - This usually causes trouble/don't work when using virt-manager's USB forwarding feature.
  - Src: https://wiki.archlinux.org/title/PCI_passthrough_via_OVMF#Passing_through_other_devices
  - List USB Bus, PCI Device and IOMMU Groups  
    `for usb_ctrl in /sys/bus/pci/devices/*/usb*; do pci_path=${usb_ctrl%/*}; iommu_group=$(readlink $pci_path/iommu_group); echo "Bus $(cat $usb_ctrl/busnum) --> ${pci_path##*/} (IOMMU group ${iommu_group##*/})"; lsusb -s ${usb_ctrl#*/usb}:; echo; done`
- Need to forward all PCI members of IOMMU group; Check with:  
  ```
  shopt -s nullglob
  for g in $(find /sys/kernel/iommu_groups/* -maxdepth 0 -type d | sort -V); do
    echo "IOMMU Group ${g##*/}:"
    for d in $g/devices/*; do
        echo -e "\t$(lspci -nns ${d##*/})"
    done;
  done;
  ```
- In Virt-Manager VM configuration: Add Hardware / PCI/PCI Host device


### Files and File System Access

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


Resize qcow2 disk image
1. Make a backup
1. Delete all snapshots, see below or in virt-manager
1. Optional "Reduce image file size by unsparsing and compressing contents" (see above)
1. Add more disk space `qemu-img resize IMAGE.qcow2 +20G`
1. Resize disk with guest tools
   - Windows: "Disk Management" utility, Right click on C:, "Extend Volume"

Convert Virtualbox VDI (or any other) disk images to QCOW2
- `vboxmanage list vms`
- `vboxmanage export VMNAME -o VMNAME.ova --ovf10`
- `tar -xvf VMNAME.ova`
- `qemu-img convert -O qcow2 -p -m 8 VMNAME-disk001.vmdk VMNAME.qcow2`
  - Optional compression: `-c`

Snapshots
- Snapshots can be created with virt-manager
  - Stored below /var/lib/libvirt/qemu/snapshot/
  - Ubuntu 22: Not possible when (U)EFI boot is enabled
    - Error creating snapshot: Operation not supported: internal snapshots of a VM with pflash based firmware are not supported
- Following commands are for direct disk image snapshots
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

QCow2 access
- guestmount -a path_to_image.qcow2 -i --ro /mount_point
- NBD: https://www.baeldung.com/linux/mount-qcow2-image

Virtiofs: Sharing files across host and guest
- https://libvirt.org/kbase/virtiofs.html


### Further Commands and Documentation

- Libvirt Schemas: /usr/share/libvirt/schemas
  - Validate with virt-xml-validate
  - Automatically done when editing XML files with virsh
- Domain XML file: [https://libvirt.org/formatdomain.html](https://libvirt.org/formatdomain.html)
  - All other XML files: [https://libvirt.org/format.html](https://libvirt.org/format.html)
- Logging
  - journalctl -u libvirtd.service
  - /var/log/libvirt/qemu/DOMAIN.log
  - Configure logging level in /etc/libvirt configuration files


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


### Direct QEMU Usage

When you're using QEMU directly (should not be needed for this guide)
here some notes.

List available machines
- qemu-system-x86_64 -machine help
- XML entry: `<os><type arch="x86_64" machine="pc-q35-6.2">`
- Latest machine: q35

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
- Windows VM: No network connection
  - Error in Network Card Driver: Device Manager (virtio, e1000e) - Code 56
  - Known error with German Windows version  
    https://github.com/virtio-win/kvm-guest-drivers-windows/issues/750
  - Workaround: Use machine pc-q35-5.1 instead of 6.2 or newer
  - Fix: Reinstall NIC drivers
    - Open device manager - right click on the NIC - select "uninstall device", remove driver files
    - Mark the top most entry and select "scan for hardware".
    - The NIC will be re-discovered
    - Search drivers on virtio-win.iso (Mounted with Right-Click / Bereitstellen) and re-installed
