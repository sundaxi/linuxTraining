# Linux_Booting.md

=================

   * [Linux Booting](#linux-booting)
      * [Hen and Egg](#hen-and-egg)
      * [Boot Process](#boot-process)
            * [Boot Sequence](#boot-sequence)
         * [Phase II MBR](#phase-ii-mbr)
            * [MBR](#mbr)
            * [Partition table](#partition-table)
         * [Boot Phase III BootLoader](#boot-phase-iii-bootloader)
            * [Boot loader stage 1](#boot-loader-stage-1)
            * [Boot loader stage 1.5](#boot-loader-stage-15)
            * [Boot Loader Stage 2](#boot-loader-stage-2)
            * [Kernel and modules](#kernel-and-modules)
            * [Initrd &amp; Initramfs](#initrd--initramfs)
            * [Rootfs](#rootfs)
            * [Start Kernel](#start-kernel)
               * [Runlevel](#runlevel)
               * [rc.sysinit](#rcsysinit)
            * [rc.local](#rclocal)
            * [Systemd](#systemd)
               * [Dependency](#dependency)
                  * [Units](#units)
               * [Basic command](#basic-command)
      * [Azure Provisioning](#azure-provisioning)
         * [Waagent](#waagent)
            * [Stages](#stages)
               * [local stage](#local-stage)
               * [network stage](#network-stage)
               * [config stage](#config-stage)
               * [final stage](#final-stage)
      * [Reference](#reference)





## Hen and Egg	

Boot, and bootstrap. comes from the old saying that "**pull oneself up by one's bootstrap**" 
Bootstraping is kind of self-starting process which is supposed to proceed without any external input. 

## Boot Process 


- MBR
- BootLoader 


![hardware_boot_steps.png](https://github.com/sundaxi/materials/blob/master/pics/linux_compute/hardware_boot_steps.png?raw=true)

*Linux boot view* 					

![boot_process_view.gif](https://github.com/sundaxi/materials/blob/master/pics/linux_compute/boot_process_view.gif?raw=true)
							  







checked below:

- PSU
- timer chip 
- DMA controller 


#### Boot Sequence 



![Boot_sequence.png](https://github.com/sundaxi/materials/blob/master/pics/linux_compute/Boot_sequence.png?raw=true)						





```bash
```




### Phase II MBR


#### MBR

MBR consists of three part below 

- first 446 bytes, Boot loader Stage 1
- next 64 bytes, partition table 
- the last 2 bytes, Magic Number(should be 55AA)

*MBR View*

![hardware_mster_boot_record_0.png](https://github.com/sundaxi/materials/blob/master/pics/linux_compute/hardware_mster_boot_record_0.png?raw=true)


#### Partition table

At most 4 primary partitions or 3 parimary partitions and 1 extented partition. 


### Boot Phase III BootLoader

Grub 

#### Boot loader stage 1

Boot Loader scan the partition table, and figure out a bootable partition with 80h flag
If no bootable flag can be found, it goes the error "No active partition"
Note: Grub doesn't care about bootable flag acctually. 

Boot loader finds out the boot sector and read the first sector of boot partition into memory. Anything wrong prompts the error "Error Loading operation system" 
Notes: it reads the first sector of boot sector not meaning MBR here. 


For this stage, error can be conclude below

- No active partition 
- Invalid partition table 
- Error loading operation system 
- Missing operation system 

#### Boot loader stage 1.5



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
Disk label type: dos
Disk identifier: 0x000abaa3
   Device Boot      Start         End      Blocks   Id  System
/dev/sda1            2048     1026047      512000   83  Linux
/dev/sda2         1026048    67108863    33041408   83  Linux
```


```bash
dd if=/dev/sda of=partition_reserved.img skip=512 bs=512 count=1536
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


#### Kernel and modules 

Linux kernel is very small, when we compile the kernel, we only add the necessary code and modules to the kernel itself, all the other modules are loaded dynamically based on real requirement. 
Let's check the root filesystem module xfs for example. 

```bash
# modinfo xfs
filename:       /lib/modules/3.10.0-693.11.6.el7.x86_64/kernel/fs/xfs/xfs.ko.xz
license:        GPL
author:         Silicon Graphics, Inc.
alias:          fs-xfs
rhelversion:    7.4
depends:        libcrc32c
intree:         Y
vermagic:       3.10.0-693.11.6.el7.x86_64 SMP mod_unload modversions
signer:         Red Hat Enterprise Linux kernel signing key
sig_hashalgo:   sha256
```

And / is mounted as xfs filesystem, isn't? How does that happen... 
If you need to mount a mountpoint, you need filesystem module loaded, if you want to load the filesystem module, you need get access to /lib/modules folder first... 


```bash
cat /proc/filesystems
```


#### Initrd & Initramfs 


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


After this, Kernel will be normally loaded with start_kernel() and dmesg starts to work at this moment. 




- SystemV Period (RHEL5). /etc/inittab
- Upstart (RHEL6). /etc/init/ 
- Systemd(RHEL7). /etc/


- Determine the runlevel(systemv), for systemd, it's just a logical old concept which is inherited from SystemV
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


- Run network scripts 
- Print a text banner 
- Set the system clock 
- Initialize hardware 
- Load other user-defined modules 
- Set the hostname 
- RAID setup 
- Device mapper & related initialization(LVM)
- Update quotas if necessary 
- Remote the root filesystem read-write 
- Reset pam_console permission
- Start up swapping 
- Initialize the serial ports 
- Active syslog, write to log files

#### rc.local

For customized script, please define at /etc/rc.local

#### Systemd



```bash
$ ls -lrat /lib/systemd/system/default.target
lrwxrwxrwx 1 root root 16 Feb 20 16:11 /lib/systemd/system/default.target -> graphical.target

$ cat /lib/systemd/system/default.target
[Unit]
Description=Graphical Interface
Documentation=man:systemd.special(7)
Requires=multi-user.target
Wants=display-manager.service
After=multi-user.target rescue.service rescue.target display-manager.service
AllowIsolate=yes
```


![systemd_boot_process.png](https://github.com/sundaxi/materials/blob/master/pics/linux_compute/systemd_boot_process.png?raw=true)


All the units are stored on /usr/lib/systemd/system/ and /etc/systemd/system/ directory.  And /etc/systemd/system has a higher priority. 

##### Dependency 

Note: `Wants=` and `Requires=` don't equal to `After=`, if no after defined, A and B will be started parrellelly. 

check dependency 

```bash
systemctl list-dependencies sysinit.target
```



###### Units 

service: define system service 
mount: define system mount point 
device: define system device 
swap: define system swap device 
path: define system file or directory 
target: Used for manage the boot process and simulate runlevel
timer: systemd managed timer

- For mount point /home, it equals to home.mount 
- For device such as /dev/sda2, it equals to dev-sda2.device 

Note: @ stands for instance. ex, name@string.service means the instance of name@service

##### Basic command 


```bahs
systemd-analyze critical-chain --fuzz 1h --no-pager
```


```bash
systemctl
systemctl list-units
```


```bash
systemctl --failed 
```


```bash
systemctl list-unit-files 
```

Reload all the units, if you did some change on the units configuration file. 

```bash
systemctl daemon-reload
```


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

## Azure Provisioning 

### Waagent






cloud-init gets the metadata from datasource. And it helps to finish following provisioning

- Set default locale
- Set hostname 
- Add authorized keys 
- set user/password
- install packages

#### Stages

For example, in Azure, when you resize the os disk on ubuntu, you don't need to use resize the partition table and filesystem manually, due to modules "resizefs && growpart"

| ------- | ------------------------ | ------------------------------------------------------------ |
| local   | cloud-init-local.service | /usr/bin/cloud-init init --local;<br />/bin/touch /run/cloud-init/network-config-ready |
| network | cloud-init.service       | /usr/bin/cloud-init init                                     |
| config  | cloud-config.service     | /usr/bin/cloud-init modules --mode=config                    |
| final   | cloud-final.service      | /usr/bin/cloud-init modules --mode=final                     |

##### local stage 


##### network stage

disk_setup and mounts modules are used to format disk and configure the mountpoint based on /etc/fstab

##### config stage

Run configuration module 

##### final stage 

Start some automation tool like chef or saltstack 


*Note: Not every module failure results in a fatal cloud-init overall configuration failure. For example, using the `runcmd` module, if the script fails, cloud-init will still report provisioning succeeded because the runcmd module executed.*


http://cloudinit.readthedocs.io/en/latest/topics/examples.html



## Reference 

https://manybutfinite.com/post/kernel-boot-process/
https://www.ibm.com/developerworks/library/l-linuxboot/
https://manybutfinite.com/post/motherboard-chipsets-memory-map/
https://en.wikipedia.org/wiki/Initial_ramdisk
https://www.ibm.com/developerworks/library/l-initrd/
https://linoxide.com/linux-how-to/systemd-boot-process/
