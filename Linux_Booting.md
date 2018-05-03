# Linux_Booting.md

Table of Contents
=================

   * [Linux Booting](#linux-booting)
      * [Hen and Egg](#hen-and-egg)
      * [Boot Process](#boot-process)
         * [Phase I BIOS](#phase-i-bios)
            * [CPU and Mortherboard](#cpu-and-mortherboard)
            * [POST](#post)
            * [Boot Sequence](#boot-sequence)
            * [ACPI](#acpi)
            * [Intel VT](#intel-vt)
         * [Phase II MBR](#phase-ii-mbr)
            * [MBR](#mbr)
            * [Partition table](#partition-table)
         * [Boot Phase III BootLoader](#boot-phase-iii-bootloader)
            * [Boot loader stage 1](#boot-loader-stage-1)
            * [Boot loader stage 1.5](#boot-loader-stage-15)
            * [Boot Loader Stage 2](#boot-loader-stage-2)
         * [Phase IV Operation System initialization](#phase-iv-operation-system-initialization)
            * [Kernel and modules](#kernel-and-modules)
            * [Initrd &amp; Initramfs](#initrd--initramfs)
            * [Rootfs](#rootfs)
            * [Start Kernel](#start-kernel)
            * [INIT process](#init-process)
               * [Runlevel](#runlevel)
               * [rc.sysinit](#rcsysinit)
            * [rc.local](#rclocal)
            * [SysV](#sysv)
            * [upstart](#upstart)
            * [Systemd](#systemd)
               * [Configuration &amp; Concept](#configuration--concept)
               * [Dependency](#dependency)
               * [Service Type](#service-type)
                  * [Units](#units)
            * [Installed units](#installed-units)
            * [Generator](#generator)
            * [Basic command](#basic-command)
            * [Define your own service](#define-your-own-service)
      * [Azure Provisioning](#azure-provisioning)
         * [Waagent](#waagent)
         * [Cloud-init](#cloud-init)
            * [Stages](#stages)
               * [local stage](#local-stage)
               * [network stage](#network-stage)
               * [config stage](#config-stage)
               * [final stage](#final-stage)
            * [Cloud-init configuration](#cloud-init-configuration)
      * [Reference](#reference)



## Hen and Egg	

Boot, and bootstrap. comes from the old saying that "**pull oneself up by one's bootstrap**" 
Bootstraping is kind of self-starting process which is supposed to proceed without any external input. 
The boot process of computer is paradoxical process,  you must run the program before the computer can start, but the computer will not be able to run the program without starting!

## Boot Process 

The boot process can be summerized as below 4 steps 

- BIOS
- MBR
- BootLoader 
- Operation System initialization 

 *Time Flow for boot process* 

![hardware_boot_steps.png](https://github.com/sundaxi/materials/blob/master/pics/linux_compute/hardware_boot_steps.png?raw=true)

*Linux boot view* 					

![boot_process_view.gif](https://github.com/sundaxi/materials/blob/master/pics/linux_compute/boot_process_view.gif?raw=true)
							  

###  Phase I BIOS

#### CPU and Mortherboard 

The first hen and egg question. 
By default, CPU runs instruction set and loads the code from Memory or ROM in some enbbeded system. But when the memory is empty and different hard computer motherboard provides different environment. How CPU knows where to get the correct code to start up the physcial box? Where is the first code?

The answer is the chipset of motherboard sent a RESET signal(pulse signal). When CPU receives the RESET signal, it resets to initial status with inital code. Once the power comes statbile, the RESET signal withdraws, and CPU starts to work with initial code. 

The initial code IN CPU indicates to get further code from a ROM device named BIOS. 

#### POST 

The first BIOS program is POST. Power-on Self-Test. The POST program performs a check of the hardware and it beeps if anything wrong. 
checked below:

- PSU
- CPU chip
- BIOS chip
- timer chip 
- DMA controller 
- Interrupt Controller 

beeps code https://en.wikipedia.org/wiki/Power-on_self-test#Original_IBM_POST_beep_codes

#### Boot Sequence 

After POST, BIOS give the contol to next phase program. 
BIOS needs to know where to find the further codes? BIOS is small, we need a bigger size and persistent storage for operation system, isn't. 
Boot Sequence in BIOS decides where is our OS. And it's configurable. 

*BIOS Boot Sequence* 

![Boot_sequence.png](https://github.com/sundaxi/materials/blob/master/pics/linux_compute/Boot_sequence.png?raw=true)						

**Wait?** BIOS was written in ROM and ROM should be ReadOnly, so where did BIOS stores the configuraiton settings?

Yes, the answer is **CMOS**. It's kind of special RAM, and its battery was provided by motherboard. 

#### ACPI

https://en.wikipedia.org/wiki/Advanced_Configuration_and_Power_Interface
After POST, BIOS will initialize all the hareware resources inclding of I/O port, Interrupt, RAM range and etc... And BIOS will store all the device map to ACPI. This mapping table will be used by Kernel afterwards. 

```bash
[    0.000000] e820: BIOS-provided physical RAM map:
[    0.000000] BIOS-e820: [mem 0x0000000000000000-0x000000000009fbff] usable
[    0.000000] BIOS-e820: [mem 0x000000000009fc00-0x000000000009ffff] reserved
[    0.000000] BIOS-e820: [mem 0x00000000000e0000-0x00000000000fffff] reserved
[    0.000000] BIOS-e820: [mem 0x0000000000100000-0x000000001ffeffff] usable
[    0.000000] BIOS-e820: [mem 0x000000001fff0000-0x000000001fffefff] ACPI data
[    0.000000] BIOS-e820: [mem 0x000000001ffff000-0x000000001fffffff] ACPI NVS
[    0.000000] BIOS-e820: [mem 0x0000000100000000-0x000000014fffffff] usable
[    0.000000] ACPI: RSDP 00000000000f5bf0 00014 (v00 ACPIAM)
[    0.000000] ACPI: RSDT 000000001fff0000 00040 (v01 VRTUAL MICROSFT 06001702 MSFT 00000097)
[    0.000000] ACPI: FACP 000000001fff0200 00081 (v02 VRTUAL MICROSFT 06001702 MSFT 00000097)
[    0.000000] ACPI: DSDT 000000001fff1d24 03CBE (v01 MSFTVM MSFTVM02 00000002 INTL 02002026)
[    0.000000] ACPI: FACS 000000001ffff000 00040
[    0.000000] ACPI: WAET 000000001fff1a80 00028 (v01 VRTUAL MICROSFT 06001702 MSFT 00000097)
[    0.000000] ACPI: SLIC 000000001fff1ac0 00176 (v01 VRTUAL MICROSFT 06001702 MSFT 00000097)
[    0.000000] ACPI: OEM0 000000001fff1cc0 00064 (v01 VRTUAL MICROSFT 06001702 MSFT 00000097)
[    0.000000] ACPI: SRAT 000000001fff0800 00130 (v02 VRTUAL MICROSFT 00000001 MSFT 00000001)
[    0.000000] ACPI: APIC 000000001fff0300 00452 (v01 VRTUAL MICROSFT 06001702 MSFT 00000097)
[    0.000000] ACPI: OEMB 000000001ffff040 00064 (v01 VRTUAL MICROSFT 06001702 MSFT 00000097)
```

#### Intel VT 

If you need to run virtual machine on your physical box, don't forgot to enable the virtualization settings in BIOS as well. 

![Intel_VT.jpg](https://github.com/sundaxi/materials/blob/master/pics/linux_compute/Intel_VT.jpg?raw=true)

### Phase II MBR

OK. It's time to handover the task to next shift engineer. 
BIOS uses INT 13 to load the MBR. 
Master Boot record is short for MBR. The size is 512 bytes which is the first sector of harddrive. 

#### MBR

MBR consists of three part below 

- first 446 bytes, Boot loader Stage 1
- next 64 bytes, partition table 
- the last 2 bytes, Magic Number(should be 55AA)

*MBR View*

![hardware_mster_boot_record_0.png](https://github.com/sundaxi/materials/blob/master/pics/linux_compute/hardware_mster_boot_record_0.png?raw=true)

The Boot loader stage 1 stores on the first 446 bytes of MBR, and this program will take control from BIOS. 

#### Partition table

At most 4 primary partitions or 3 parimary partitions and 1 extented partition. 
Each partition has 16 bytes and the last 4 bytes(total sector numbers) decide the length of partition, the maxisum size is 2^32 which is 2TB. 

The extended partition table is a pointor to logical partition 

### Boot Phase III BootLoader

Grub 

#### Boot loader stage 1

Boot Loader scan the partition table, and figure out a bootable partition with 80h flag
If no bootable flag can be found, it goes the error "No active partition"
Note: Grub doesn't care about bootable flag acctually. 

Boot loader finds out the boot sector and read the first sector of boot partition into memory. Anything wrong prompts the error "Error Loading operation system" 
Notes: it reads the first sector of boot sector not meaning MBR here. 

In the end, boot loader check the MBR magic number 55AA. If not , comes the error "Missing Operation system"

For this stage, error can be conclude below

- No active partition 
- Invalid partition table 
- Error loading operation system 
- Missing operation system 

#### Boot loader stage 1.5

The second hen and egg question here. As you might already know that the grub main code and kernel code are stored under /boot/ mountpoint. But the problem, if you want to mount a mountpoint, you must give it a **filesystem**! And the filesystem module was stored in /boot/initramfs*

The codes in stage 1 are too tiny. We cannot put the filesystem module there.  In order to get access to /boot folder, we need Boot loader stage 1.5 to help provide the filesystem code and mount it temporary. 

Boot loader stage 1.5 was stored on the first 32K of harddrive. From sector 2 to sector 64

```bash
dd if=/dev/sda of=stage_1_5.img skip=1 bs=512 count=63
```

Normally, the partition starts from 2048 sector, which means 1M was reserved by default

```bash
fdisk -l /dev/sda
Disk /dev/sda: 34.4 GB, 34359738368 bytes, 67108864 sectors
Units = sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes
Disk label type: dos
Disk identifier: 0x000abaa3
   Device Boot      Start         End      Blocks   Id  System
/dev/sda1            2048     1026047      512000   83  Linux
/dev/sda2         1026048    67108863    33041408   83  Linux
```

Try this out to check reserved the disk, nothing is there.

```bash
dd if=/dev/sda of=partition_reserved.img skip=512 bs=512 count=1536
hexdump -C partition_after.img
00000000  00 00 00 00 00 00 00 00  00 00 00 00 00 00 00 00  |................|
*
000c0000
```

Question, can /boot folder be mounted as LVM? 

#### Boot Loader Stage 2

Finally, we got the grub configuraiton /boot/grub2/grub.cfg. 
You can select a kernel and even amend it with addtional kernel paremeters
grub environment under /boot/grub2/grubenv
Sample of grub menuentry

```grub
menuentry 'Red Hat Enterprise Linux Server (3.10.0-693.11.6.el7.x86_64) 7.4 (Maipo)' 
{
    linux16 /vmlinuz-3.10.0-693.11.6.el7.x86_64 root=UUID=ef39b601-3484-4f7c-81f0-3cb8b79eeda1 ro console=tty1 console=ttyS0 earlyprintk=ttyS0 rootdelay=300
    initrd16 /initramfs-3.10.0-693.11.6.el7.x86_64.img
}
```

vmlinuz-3.10.0-693.11.6.el7.x86_64: this is the kernel
root=UUID=ef39b601-3484-4f7c-81f0-3cb8b79eeda1: root partition UUID
initramfs-3.10.0-693.11.6.el7.x86_64.img: initramfs image

Looks everything is ready, grub helps us to find out all the necessary information, where is the kernel, where is the root partition and where is initramfs. 

BootLoader loads the kernel and initramfs to memory and give the control to kernel to initilize the system 

Wait, what does initramfs used for?

### Phase IV Operation System initialization  

#### Kernel and modules 

Linux kernel is very small, when we compile the kernel, we only add the necessary code and modules to the kernel itself, all the other modules are loaded dynamically based on real requirement. 
Let's check the root filesystem module xfs for example. 

```bash
# modinfo xfs
filename:       /lib/modules/3.10.0-693.11.6.el7.x86_64/kernel/fs/xfs/xfs.ko.xz
license:        GPL
description:    SGI XFS with ACLs, security attributes, no debug enabled
author:         Silicon Graphics, Inc.
alias:          fs-xfs
rhelversion:    7.4
srcversion:     1BE72505E1F78C8542EA721
depends:        libcrc32c
intree:         Y
vermagic:       3.10.0-693.11.6.el7.x86_64 SMP mod_unload modversions
signer:         Red Hat Enterprise Linux kernel signing key
sig_key:        5F:D8:EB:CF:C4:C5:20:3C:3C:B7:90:52:19:FB:66:9D:5F:4B:E3:FF
sig_hashalgo:   sha256
```

The third hen and egg question,  did you see any interesting here? 
The xfs module was stored on /lib/modules/3.10.0-693.11.6.el7.x86_64/kernel/fs/xfs/xfs.ko.xz which is root mountpoint. 
And / is mounted as xfs filesystem, isn't? How does that happen... 
If you need to mount a mountpoint, you need filesystem module loaded, if you want to load the filesystem module, you need get access to /lib/modules folder first... 
The kernel becomes cubersome if we add all the filesystem modules to the kernel itself 

Check all loaded filesystem 

```bash
cat /proc/filesystems
```

The same question is also for INIT process which is located /sbin/init. 
So we need a middleware to achieve this. That is initramfs/initrd

#### Initrd & Initramfs 

The initramfs is used to solve above issue. 
The device drivers for this generic kernel image are included as loadable kernel modules because statically compiling many drivers into one kernel causes the kernel image to be much larger, perhaps too large to boot on computers with limited memory. This then raises the problem of detecting and loading the modules necessary to mount the root file system at boot time, or for that matter, deducing where or what the root file system is.

The initrd was deprecated after kernel 2.4. 
Initrd is kind of ramdev block device. It's ram-based block device, that is simulated hard disk that uses memory instead of physical disks. And you need to uncompress it into memory and mount it as a filesystem. In this case, the same content exists in memory twice. 

```bash
zcat initrd | dd of=/dev/ram0
mount /dev/ram0 /root
```

Initramfs is kind of tmpfs mounted. 

```bash
mount -t tmpfs nodev /root
zcat initramfs | cpio -i
```

#### Rootfs 

Initramfs is a rootfs instance.  For a rootfs at least contain below folders 

- /etc/
- /bin/ 
- /sbin/
- /lib/
- /dev/

#### Start Kernel 

Start function `arch/x86/boot/header.S` 
*Real Mode to Protected Mode*

![bootstrap_startup_process.png](https://github.com/sundaxi/materials/blob/master/pics/linux_compute/bootstrap_startup_process.png?raw=true)

Kernel will initiate the Interrupt and Memory. 

- Interrupt stores on IDTR table 
- Memory was descripted by GDTR(Global Descriptor Table)

After this, Kernel will be normally loaded with start_kernel() and dmesg starts to work at this moment. 

Then kernel starts the INIT process 

#### INIT process 

The first process of operation system. 

- SystemV Period (RHEL5). /etc/inittab
- Upstart (RHEL6). /etc/init/ 
- Systemd(RHEL7). /usr/lib/systemd/

The INIT process generally did the below 

- Determine the runlevel(systemv), for systemd, it's just a logical old concept which is inherited from SystemV. Initial tty/serial console 
- rc.sysinit to initiate the boot provisioning 
- start the services based on runnlevel 

##### Runlevel 

0 - Halt

1 - Single User mode. No network, no deamon service, only super user could login 

2 - Multi User. No network, no daemon service

3 - Multi User. Normally start the system. No Graphic User Interface

4 - User customized. 

5 - Advanced Runlevel 3 with GUI

6 - Reboot 

##### rc.sysinit 

The initialization process, generally finishes below target/task

- Run network scripts 
- Check SElinux status 
- Print a text banner 
- Set the system clock 
- Initialize hardware 
- Load other user-defined modules 
- Configure kernel paremeters 
- Set the hostname 
- Initialize ACPI bits 
- RAID setup 
- Device mapper & related initialization(LVM)
- Update quotas if necessary 
- Remote the root filesystem read-write 
- Clean up SELinux labels 
- Mount all other filesystems(except for NFS/CIFS and /proc)
- Reset pam_console permission
- Start up swapping 
- Initialize the serial ports 
- Active syslog, write to log files

#### rc.local

For customized script, please define at /etc/rc.local

#### SysV

All the configuration under /etc/inittab, samples below

```bash
# Default runlevel. The runlevels used by RHS are:
#   0 – halt (Do NOT set initdefault to this)
#   1 – Single user mode
#   2 – Multiuser, without NFS (The same as 3, if you do not have networking)
#   3 – Full multiuser mode
#   4 – unused
#   5 – X11
#   6 – reboot (Do NOT set initdefault to this)
id:5:initdefault:
# System initialization.
si::sysinit:/etc/rc.d/rc.sysinit
l0:0:wait:/etc/rc.d/rc 0
l1:1:wait:/etc/rc.d/rc 1
l2:2:wait:/etc/rc.d/rc 2
l3:3:wait:/etc/rc.d/rc 3
l4:4:wait:/etc/rc.d/rc 4
l5:5:wait:/etc/rc.d/rc 5
l6:6:wait:/etc/rc.d/rc 6
# Trap CTRL-ALT-DELETE
ca::ctrlaltdel:/sbin/shutdown -t3 -r now
# When our UPS tells us power has failed, assume we have a few minutes
# of power left.  Schedule a shutdown for 2 minutes from now.
# This does, of course, assume you have powerd installed and your
# UPS connected and working correctly.
pf::powerfail:/sbin/shutdown -f -h +2 “Power Failure; System Shutting Down”
#
# If power was restored before the shutdown kicked in, cancel it.
pr:12345:powerokwait:/sbin/shutdown -c “Power Restored; Shutdown Cancelled”
#
# Run gettys in standard runlevels
1:2345:respawn:/sbin/mingetty tty1
2:2345:respawn:/sbin/mingetty tty2
3:2345:respawn:/sbin/mingetty tty3
4:2345:respawn:/sbin/mingetty tty4
5:2345:respawn:/sbin/mingetty tty5
6:2345:respawn:/sbin/mingetty tty6
#
# Run xdm in runlevel 5
x:5:respawn:/etc/X11/prefdm -nodaemon
```

#### upstart 

/etc/init/ event script. System get the event based on the event.conf
example, trigger a system reboot 

```bash
# /etc/init/control-alt-delete
start on control-alt-delete
initctl emit control-alt-delete
```

#### Systemd

Systemd uses target to manage and control boot process. The boot target is default.target.

Check the default.target. 

```bash
$ ls -lrat /lib/systemd/system/default.target
lrwxrwxrwx 1 root root 16 Feb 20 16:11 /lib/systemd/system/default.target -> graphical.target

$ cat /lib/systemd/system/default.target
[Unit]
Description=Graphical Interface
Documentation=man:systemd.special(7)
Requires=multi-user.target
Wants=display-manager.service
Conflicts=rescue.service rescue.target
After=multi-user.target rescue.service rescue.target display-manager.service
AllowIsolate=yes

$ cat /lib/systemd/system/multi-user.target
[Unit]
Description=Multi-User System
Documentation=man:systemd.special(7)
Requires=basic.target
Conflicts=rescue.service rescue.target
After=basic.target rescue.service rescue.target
AllowIsolate=yes

$ cat /lib/systemd/system/basic.target 
[Unit]
Description=Basic System
Documentation=man:systemd.special(7)
Requires=sysinit.target
After=sysinit.target
Wants=sockets.target timers.target paths.target slices.target
After=sockets.target paths.target slices.target
```

The Basic Flow. systemd-remount-fs

![systemd_boot_process.png](https://github.com/sundaxi/materials/blob/master/pics/linux_compute/systemd_boot_process.png?raw=true)

##### Configuration & Concept 

All the units are stored on /usr/lib/systemd/system/ and /etc/systemd/system/ directory.  And /etc/systemd/system has a higher priority. 

##### Dependency 

Typically, unit A requires B. `Required=B` or `After=B` could achieve this. If the dependency is alternative, `Wants=B` or `After=B`.  That means, if A failed, wants will try to start B but requires never do so. 
Note: `Wants=` and `Requires=` don't equal to `After=`, if no after defined, A and B will be started parrellelly. 

check dependency 

```bash
systemctl list-dependencies sysinit.target
```

##### Service Type 

**Type=simple:**  default. The service will be started up directly and service process won't be focked. 
**Type=forking:**  systemd treats this service as fork process. Normally needs to define the `PIDFILE=` and used for systemd to tail parent process
**Type=oneshot:**  run one time and exit. Needs to configure`RemainAfterExit=yes` to let systemd treats it as activated. 
**Type=notify:**  when the service is ready, send a signal to systemd. `libsystem-daemon.so` 
**Type=dbus:**  when the service is booted, BusName will be shown on dbus and systemd knows the service is ready
**Type=idle:** idle service will be started once all the other services are ready

###### Units 

service: define system service 
mount: define system mount point 
sockets: define system socket for IPC 
device: define system device 
swap: define system swap device 
path: define system file or directory 
target: Used for manage the boot process and simulate runlevel
timer: systemd managed timer

- The default unit type is service
- For mount point /home, it equals to home.mount 
- For device such as /dev/sda2, it equals to dev-sda2.device 

Note: @ stands for instance. ex, name@string.service means the instance of name@service

#### Installed units 

Symbolic link to `/etc/systemd/system/multi-user.target.wants`

#### Generator 

Generators are small binaries that live in /usr/lib/systemd/user-generators/ and other directories. Systemd will execute thoese binaries very early at bootup and at configuration reload time - before unit files are loaded. Example, systemd-fstab-generator 

#### Basic command

Check the boot process of systemd 

```bahs
systemd-analyze critical-chain --fuzz 1h --no-pager
```

Check activate status 

```bash
systemctl
systemctl list-units
```

Check failed units 

```bash
systemctl --failed 
```

Check all installed service 

```bash
systemctl list-unit-files 
```

Reload all the units, if you did some change on the units configuration file. 

```bash
systemctl daemon-reload
```

Check runlevel 

```bash
systemctl get-default
```

Switch the runlevel 

```bash
systemctl isolate graphical.target
```

Set teh default runlevel

```bash
systemctl set-default multi-user.target
```

#### Define your own service 

We could define a service named test.service 

```
[Unit]
Description=Azure Linux customized script
Wants=network-online.target sshd.service sshd-keygen.service
After=network-online.target
ConditionFileIsExecutable=/etc/rc.d/test
[Service]
Type=oneshot
ExecStart=/etc/rc.d/test start
[Install]
WantedBy=multi-user.target
```

## Azure Provisioning 

### Waagent

- Set resourceDisk 
- Set useraccount/password/publickey
- Set Hostname 

### Cloud-init

Cloud-init -local.service is before sysinit.target

cloud-init gets the metadata from datasource. And it helps to finish following provisioning

- Set default locale
- Set hostname 
- Add authorized keys 
- set user/password
- install packages

#### Stages

Four stages to start up the Linux system. The modules decide all the behavior 
For example, in Azure, when you resize the os disk on ubuntu, you don't need to use resize the partition table and filesystem manually, due to modules "resizefs && growpart"

| Stages  | Service Name             | Execuate Command                                             |
| ------- | ------------------------ | ------------------------------------------------------------ |
| local   | cloud-init-local.service | /usr/bin/cloud-init init --local;<br />/bin/touch /run/cloud-init/network-config-ready |
| network | cloud-init.service       | /usr/bin/cloud-init init                                     |
| config  | cloud-config.service     | /usr/bin/cloud-init modules --mode=config                    |
| final   | cloud-final.service      | /usr/bin/cloud-init modules --mode=final                     |

##### local stage 

Cloud-init get the configuration information from config drive then write into /etc/network/interfaces/ By default, use DHCP to startup network interface

##### network stage

disk_setup and mounts modules are used to format disk and configure the mountpoint based on /etc/fstab

##### config stage

Run configuration module 

##### final stage 

Start some automation tool like chef or saltstack 

The configuration of cloud-init acctually consists of modules, the modules configuration determine the workflow. 

*Note: Not every module failure results in a fatal cloud-init overall configuration failure. For example, using the `runcmd` module, if the script fails, cloud-init will still report provisioning succeeded because the runcmd module executed.*

#### Cloud-init configuration 

http://cloudinit.readthedocs.io/en/latest/topics/examples.html

The custom-data should be started with "#cloud-config"


## Reference 

https://manybutfinite.com/post/kernel-boot-process/
https://www.ibm.com/developerworks/library/l-linuxboot/
https://manybutfinite.com/post/motherboard-chipsets-memory-map/
https://en.wikipedia.org/wiki/Initial_ramdisk
https://www.ibm.com/developerworks/library/l-initrd/
https://linoxide.com/linux-how-to/systemd-boot-process/



