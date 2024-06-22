# 1) Description

kvm is a set of VM bhyve management tools for FreeBSD (writted and tested on 14.0-RELEASE). **It requires ZFS**.  
Each created VM can be associated with a rc script in order that this VM could be launched by `service VMname [one]start` and could be run at system startup (VMname_enable=YES in rc.conf).  
kvm does not spread files in the system. All stays in the same place (depending the pool you choose), except the  rc scripts which land in /usr/local/etc/rc.d/.  


# 2) Installation

Copy the files kvminst and kvm.txz at the same place (this place has no importance). Then, chmod +x kvminst and run it as root user. kvminst will ask you on which pool you want to install kvm and then run the following commands (\*kpool\* = choosen pool):  

`zfs create *kpool*/kvm`  
`zfs create *kpool*/kvm/vm`  
`tar -xf kvm.txz -C /*kpool*/kvm`  

Please, even if it is not mandatory, install edk2-bhyve and grub2-bhyve packages/ports.


# 3) Deinstallation

Be sure that all VMs are stopped. Check in /usr/local/etc/rc.d/ if there are some scripts that belong to kvm and delete them. Look into /etc/rc.conf if there are some related vars and remove them (e.g. VMname_enable=YES).  
Run the folowing command:  
`# zfs destroy -rf *kpool*/kvm`


# 4) Basic usage

Most of the commands need the root privilege to be run.  

cd into /\*kpool\*/kvm.

Create a VM:  
`# ./vmcreate myVM`  
Answer to the questions. Warning! There are few to no check of what you're typing. So do it with caution. In particular, don't forget the unit suffixes when you enter the memory and the main disk sizes. That said, you can always re-read, edit /\*kpool\*/kvm/vm/myVM/set.conf and modify/set some vars before to actually start the VM. It's advised anyway.  

You can choose to define a basic network at VM creation. It uses a tap that it's passed to the VM as a virtio-net device (so the guest OS needs to have a driver for this. It is present in Linux and FreeBSD but not in Microsoft Windows). This tap is connected to a bridge (already existing or not) and this last needs to be connected to the interface that brings internet to the host.  

Don't forget to choose a means to communicate with the VM. It can be VNC or serial console (COM). VNC can be used only if the loader is of uefi type.  

Once done, you probably need to use an iso image in order to install an OS. Use:  
`# ./insert myVM /path/to/isofile`  

To start the VM:  
`# ./start myVM &`  
Note: the VM is run inside an infinite loop of the start script in order to make it possible to reboot the VM. If you do just `./start myVM`, the console is exclusively held by the start script.  

You can stop the VM with (it's like switch off the power):  
`# ./stop myVM`  

After installation you can remove the iso image from the VM with:  
`# ./eject myVM`


# 5) Principles

Unlike most famous tools, this one is not really automated. It's based on a simple sh script `start` that sources two sh files for a given VM: a setting file (set.conf) and a run file (run). All the vars are present in the initial set.conf file in the shape of explanatory comments. The run file contains two functions, preRun() and postRun(), which are sh scripts that are respectively launched before to start the VM and after the VM is terminated (including reboot).  

The most used loaders are provided in their binary forms: grub-bhyve and BHYVE_UEFI.fd (corresponding to VMrom="grub" and VMrom="uefi", respectively). It's not really to avoid dependencies but mainly to protect the VMs from potentialy desastrous upgrades of these loaders. If a vulnerability is detected, one can change the defaults binary files in /\*kpool\*/kvm/. That's why I forcely advise to install edk2-bhyve and grub2-bhyve. Note that nothing prohibits to give the full path of a loader like /usr/local/share/uefi-firmware/BHYVE_UEFI.fd or any others in the VMrom var.  

Each VM is located in its own dataset, you can reach each of them on /\*kpool\*/kvm/vm/"VMname"/. This dir contains configuration and disk files (it's advised to leave the disk here). You can even add any file of importance concerning the VM (e.g. documentation).  

The main disk is a simple file you set at VM creation time. I choose a file and not a zvol because it's simpler to make and manage snapshots. I didn't see a real speed difference between zvol and file. I found that just the driver matters and nvme is the best (so, it's the default). You are free to override this default driver and even to use a zvol, providing you give its path in the var "Disk". You can also manually add disks in the VM with the parameter "Devices".  

Network settings examples:  

Example1 (this one matches more or less the automated form you can set at VM creation):  
Let's say you want a basic network. The VM needs to access the internet. You already set a bridge at startup (bridge0) and this bridge has your network card as member.  

- In the free vars of set.conf, you can set: `LAN="tapMYVM"` (the name must begin by "tap").  
- In the Devices parameter: `Devices="4,virtio-net,$LAN"`.  
- In the preRun() script add:  
  `CreateIF tap $LAN`   
	`ifconfig bridge0 addm $LAN`  
- In the postRun() script add: `DestroyIF $LAN`   

In fact, all is possible with pre/postRun scripts. You can create any interface including a bridge, change the default route and so on. The related functions are in the /\*kpool\*/kmv/functions file: SetResolv, SetDefaultRoute, CreateIF, DestroyIF. You may want to see comments for these functions. Anyway, they are just some thin wrappers that use the base commands like `ifconfig`.  

Example2:  
You have a machine with a realtek network card. You set up a pfsense VM and you want that when you launch it, this VM serves as firewall, protecting this machine from your local network. Your local ip is given by DHCP, your current gateway is 192.168.1.1. The lan interface ip of pfsense is 192.168.2.1. pfsense uses DHCP for the passtrhu re0 ("WAN").  

`# pciconf -l re0`  
`re0@pci0:3:0:0: class=0x020000 rev=0x06 hdr=0x00 vendor=0x10ec device=0x8168 subvendor=0x1462 subdevice=0x7798`  

**File: set.conf** 

VMoptions="ASHP"  
VMrom="uefi"  
Disk="disk"  
Cores="2"  
Mem="4G"  
LAN="tapLAN"  
BRI="bridgeLOCAL"  
IpLAN="192.168.2.2/24"  
GW="192.168.2.1"  
Devices="4,virtio-net,$LAN 5,passthru,3/0/0"  

**File: run**  
 
preRun()  
{  
	&emsp;# Network infrastructure  
	&emsp;devctl set driver -f pci0:3:0:0 ppt  
	&emsp;sleep 1  
	&emsp;CreateIF tap $LAN  
	&emsp;CreateIF bridge $BRI  
	&emsp;ifconfig $LAN inet $IpLAN  
	&emsp;ifconfig $BRI addm $LAN  
	&emsp;ifconfig $BRI up  
	&emsp;SetDefaultRoute $GW   
	&emsp;SetResolv $GW  
}  

postRun()  
{  
	&emsp;DestroyIF $LAN  
	&emsp;sleep 1  
	&emsp;devctl set driver -f pci0:3:0:0 re  
	&emsp;sleep 1  
	&emsp;ifconfig $BRI addm re0  
	&emsp;SetResolv 192.168.1.1   
}  



# 6) kvm man

- **dorc**  
&emsp;`dorc VMname`  
This utility creates a rc script in /usr/local/etc/rc.d/ for a given VM. Then, you can start, stop and get the status of this VM with the command `service`. The aim is to start this VM at boot time (VMname_enable=YES in /etc/rc.conf). Choose carefully the contents of REQUIRE:, BEFORE: and KEYWORD: fields. See man rc.d and man rcorder.

- **eject**  
&emsp;`eject VMname`  
Remove an inserted iso file from the VM. Be aware that the change takes effect when the VM is started.  

- **insert**  
&emsp;`insert VMname /path/to/isofile`  
Insert an iso image into the VM. Be aware that the change takes effect when the VM is started.  

- **snap**  
Manage VM snapshots.  
&emsp;`snap [create|destroy|rollback] VMname SnapshotName`  
&emsp;`snap list VMname`  
All commands can be abbreviated like c* = create, d* = destroy, r* = rollback and l* = list. It's just the first letter that matters. You can rollback only the most recent snapshot. If you need to rollback before that, you have to destroy the most recent snapshot(s) before.  
A snapshot contains all the content of the zfs dataset (\*kpool\*/kvm/vm/VMname). This is why it's advised to keep the virtual disk (a simple file by default) in this place.  

- **start**  
&emsp;`start VMname [&]`  
It's the main script of the kvm tools. It checks some essential parameters, the existence of the disk and iso files if any. It determines if the loader is grub and takes appropriate mesures. In case of the use of VNC, it looks for a free TCP port starting from 5900 and writes it at the console. Just before to launch the VM, it prints the resulting bhyve command line. It's convenient when debug time comes.  
The bhyve command line is included in an infinite loop that exits when an error is encountered or poweroff received.  

- **status** (no need to be root)  
&emsp;`status VMname`  
Get the pid of the bhyve VM. If the VMname VM isn't running it returns 0. If the VMname VM doesn't exist, it replies -1.  

- **stop**  
&emsp;`stop VMname`  
Force the VM to poweroff.  

- **vmcreate**  
&emsp;`vmcreate VMname`  
Self-explanatory, it's interactive. Some questions will be asked. Some checks will be made.  

- **vmdestroy**  
&emsp;`vmdestroy VMname`    
Destroy an existing VM. All its data and its snapshots will be lost. No return from there.

- **vminfo** (no need to be root)  
&emsp;`vminfo VMname`  
Give detailled informations about a specified VM. You will see if it's running and since how many time. What's its loader, path of its main disk, network tap name if any, passthru devices if any and so on.  

- **vmlist** (no need to be root)  
Give a list of running VMs, and their associated pid. Give also a list of not running VMs.  

- **vmrename**  
&emsp;`vmrename oldname newname`  
Also self-explanatory. This process is prone to errors and side-effects, so it's advised to check the result by hand.  


