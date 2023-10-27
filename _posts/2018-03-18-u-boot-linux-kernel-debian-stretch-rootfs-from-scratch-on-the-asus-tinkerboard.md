---
title: "u-boot, Linux Kernel & Debian Stretch rootfs from scratch on the Asus Tinkerboard"
date: "2018-03-18"
categories: 
  - "development"
  - "embedded"
tags: 
  - "asus"
  - "cross-compile"
  - "debootstrap"
  - "device-tree"
  - "embedded"
  - "linux"
  - "rootfs"
  - "tinkerboard"
  - "u-boot"
---

This post details how to build your own components for running Debian Stretch on the Asus Tinkerboard from scratch!

I am a [Ubuntu Desktop LTS 14.04.3](https://www.ubuntu.com/download/desktop) hosted using [VMWare Workstation 12.5 Player](https://www.vmware.com/go/tryplayerpro-win-64) (Non-commerical edition) on a Windows 10 Machine.  Ubuntu and VMWare Player are free to use for non-commerical use to it should be easy to replicate these steps.

I am also using a USB to UART converter so connect to the Tinker Board over make a serial connection from my PC. The exact model i am using can be brought for £6 [here](https://www.amazon.co.uk/gp/product/B00AFRXKFU/ref=oh_aui_detailpage_o00_s00?ie=UTF8&psc=1). The converter only needs to use GND, 5v, Tx and Rx to connect with the UART on the Tinkler Board. When using the 5v, you dont actually need to power the cable via the micro USB, the power comes through the 5V. However i like to not have the 5v plugged in and use a MicroUSB cable because the Tinkerboard is known to have some power issues.

![](/images/tinker-layout.png)
You can connect the 5v on the serial converter to pins 2 or 4, the GND on the converter to pins 6,9,14,25,30,34 or 39. The Transmix (Tx) of the serial converter needs to go to the UART receive pin at 10 and the Receive (Rx) of the converter needs to go to the transmit pin of the tinker board at 8.
![](/images/2.png)
![](/images/3.png)

Then, on your PC look at the COM ports in device manager and once the serial converter drivers have been installed, you should see the converter available on a COM port. ![](/images/COM_port_serial_converter.png)

Using Putty, you can open a serial connection to the board on that port. Im using the common baud rate of 115200. ![](/images/putty.png)

The Tinkerboard uses an ARM hard float chip so all programs will have to be compiled for that architecture. Download the ARM hard float compiler on the Ubuntu VM via

```sudo apt-get install gcc-arm-linux-gnueabihf```

Download the Device Tree Compiler so that we can compile the customised device tree file

```sudo apt-get install device-tree-compiler ```

## Preparing the SD card

To use the following method for building a Tinkerboard image, youll need an SD card with two partitions. The first partition must be FAT32 formatted and can be small in size. It will host the bootloader image, the kernel image (compressed) and the device tree image. The second partition will be the rest of the size of the SD card but should be at least a couple of GB.

If you have gparted installed, its very easy to use. However if not, you can do the following to format the SD card:

Plug in the SD card, in a command prompt, type in lsblk to see the device name. sudo lsblk

Then use fdisk on the device:

```
$ sudo fdisk /dev/sdb
...

#d //delete any existing partitions (formatting SD card)
#o //deletes any existing MBR on the card

#n //creates new partition for boot
#p //select ‘primary’ partition
#1 //give partition number
#[enter] //start at 2048
#+20M //partition size of 20Mb

#n //creates new partition for rootfs
#p //select ‘primary’ partition
#2 //give partition number
#[enter] //start at end of partition 1
#+8G //partition size of 8Gb

#w //writes the above to the SD card
```

Once completed, open fdisk again and press p to check the partitions. Use lsblk to see if the partitions are there as well, if not safely eject the SD card, reinsert and use lsblk again. Change partition filesystem types :

```
mkfs.vfat /dev/sda1
mkfs.ext4 /dev/sda2
```

## u-boot

Download the Tinkerboard RK3288 U-Boot

```git clone -b linux4.4-rk3288 https://github.com/TinkerBoard/debian_u-boot.git```

Set a couple of session variables, this saves having to declare them in the make statements

```
export ARCH=arm
export CROSS_COMPILE=arm-linux-gnueabi-
```

Create the configuration for the RK3288 Chip, then compile u-boot

```
make evb-rk3288_defconfig
make
```

Once compilation is complete, you will have a number of u-boot files. The one we need is called "u-boot.img"

## Linux Kernel

Download the Tinkerboard Kernel

```git clone https://github.com/TinkerBoard/debian_kernel.git ```

If using a different terminal session, remember to set the ARCH and CROSS_COMPILE flags
```
export ARCH=arm
export CROSS_COMPILE=arm-linux-gnueabi-
```

Make the configuration for the RK3288 chip, make the Kernel Image, make an modules and finally make the device tree

```
make ARCH=arm miniarm-rk3288_defconfig
make zImage ARCH=arm CROSS_COMPILE=arm-linux-gnueabi-
make modules ARCH=arm CROSS_COMPILE=arm-linux-gnueabi-
make ARCH=arm rk3288-miniarm.dtb CROSS_COMPILE=arm-linux-gnueabi-
```

You can ignore the warnings for the device tree compilation for now. The files we need are the zImage which is the compressed kernel image and the compiled device tree file.

The zImage can be found at /arch/arm/boot/zImage The compiled Device Tree can be found at /arch/arm/boot/dts/rk3288-miniarm.dtb

Note - as we have only compiled on device tree it can be copied using

`sudo cp /arch/arm/boot/dts/*.dtb`

## Root File System

For the root file system, we will use QEMU Debotstrap to emulate an ARM HF architecture the same as the tinker board and we will then download a basic root file system. Once downloaded, we will use Schroot to chroot into the directory and download any packages and create users accounts. Once the root file system is setup, we will copy it to an SD card in the ext4 partition.

Download the prerequisites needed for a debootstrap of the root fs

```sudo apt-get install binfmt-support qemu qemu-user-static debootstrap ```

Make a directory where the rootfs will go

```
mkdir ~/Desktop/arm_stretch
```

Debootstrap!

```sudo qemu-debootstrap --arch armhf stretch ~/Desktop/arm_stretch http://deb.debian.org/debian/ ```

Make sure schroot is downloaded, then create a configuration for this particular schroot session. Ive called mine stretch.conf Then copy the following configuration, where directory is the path to the rootfs and change the usernames

```
sudo apt-get install schroot
sudo nano /etc/schroot/chroot.d/stretch.conf
 
[stretch]
description=stretch for the tinker
directory=/home/ashley/Desktop/arm_stretch/
root-users=ashley
type=directory
users=ashley
```

View all schroot configurations (should be at least 2, the default one and the one just created)

`sudo schroot -l`

Use the schroot we just created

`sudo schroot -c stretch -u root`

Next, update & upgrade, install any packages and create user passwords then exit

```
apt-get update
apt-get upgrade
apt-get install sudo
passwd
useradd ashley
passwd ashley
usermod -aG sudo ashley
exit
```

## SD Card Transfer

Mount the FAT 32 partition and copy across the u-boot image, the kernel zimage and the kernel device tree file.

```
sudo mount /dev/sda1 /mnt
sudo cp zImage rk3288-miniarm.dtb u-boot.img /mnt
sudo umount
```

Mount the EXT4 partition and copy across the root file system

```
sudo mount /dev/sda2 /mnt
sudo cp -r ~/Desktop/arm_stretch/* /mnt
sudo umount /mnt
```

Then safely eject the SD card and plug into the Tinker board and power on

## u-boot Configuration

When u-boot starts up, interrupt the process.

Enter the following commands to configure u-boot to boot correctly. Firstly, check which mmc block your SD card is recognised on, for me it was block 1 and not zero. The first number indicates the device, the second is the partition.

```
ls mmc 0:1
ls mmc 0:2
ls mmc 1:1
ls mmc 1:2
```

Create the following variables. For the MMC root, if your block is recognised as 0, then change the mmc block to 1, if it is recognised as 1 then change the mmcblock as 0.

```
setenv console ttyS1
setenv baudrate 115200
setenv mmcroot /dev/mmcblk0p2 rootwait rw
setenv mmcargs "setenv bootargs console=${console},${baudrate} root=${mmcroot} rw rootfstype=ext4 rootwait init=/sbin/init"
setenv fdt_file rk3288-miniarm.dtb
setenv fdt_high 0x0fffffff
setenv boot_normal "fatload mmc 1:1 0x01f00000 ${fdt_file} && fatload mmc 1:1 0x02000000 zimage && run mmcargs && bootz 0x02000000 - 0x01f00000"
setenv bootcmd "run boot_normal"
saveenv
run bootcmd
```

What this does is it loads the device tree and kernel image into RAM and tells uboot to boot. It also sets the runtime arguments to that the kernel knows what serial port to use and where it can expect to find the rootfs. As we are using a compressed linux kernel without the u-boot header (zImage) the command is bootz. Bootz expects three arguments . As we are not using a ramdisk, the middle argument is replaced with an "-".

We are changing the bootcmd command to a custom variable called boot_normal which when called will boot our os. Then we are then saving the changes to we dont have to enter these every time. Finally we call the bootcmd command which will leave u-boot and test our new configuration.

Once setup, you should be able to log in using the credentials configured during the schroot stage. I recommend then changing the hostname as that is something you cannot do when using schroot.

`sudo nano /etc/hostname`

![](/images/logged-in.png)

To help with any debugging, here are my uboot variables, board info, kernel dmesg and rootfs messages.

### Uboot Variables

```
U-Boot SPL 2017.07-g7d17b02823 (Nov 15 2017 - 14:29:05)
Returning to boot ROM...
 
U-Boot 2017.07-g7d17b02823 (Nov 15 2017 - 14:29:05 +0800), Build: jenkins-rockchip-rk3288-tinker-board-debian-107
 
Model: Tinker-RK3288
DRAM:  2 GiB
MMC:   dwmmc@ff0c0000: 1, dwmmc@ff0f0000: 0
In:    serial
Out:   serial
Err:   serial
Model: Tinker-RK3288
Net:   eth0: ethernet@ff290000
Hit any key to stop autoboot:  0
=>
=> printenv
arch=arm
autoload=no
baudrate=115200
board=tinker_rk3288
board_name=tinker_rk3288
boot_a_script=load ${devtype} ${devnum}:${distro_bootpart} ${scriptaddr} ${prefix}${script}; source ${scriptaddr}
boot_efi_binary=load ${devtype} ${devnum}:${distro_bootpart} ${kernel_addr_r} efi/boot/bootarm.efi; if fdt addr ${fdt_addr_r}; then bootefi ${kernel_addr_r} ${fdt_addr_r};else bootefi ${kernel_addr_r} ${fdtcontroladdr};fi
boot_extlinux=sysboot ${devtype} ${devnum}:${distro_bootpart} any ${scriptaddr} ${prefix}extlinux/extlinux.conf
boot_net_usb_start=usb start
boot_normal=fatload mmc 1:1 0x01f00000 rk3288-miniarm.dtb && fatload mmc 1:1 0x02000000 zimage && run mmcargs && bootz 0x02000000 - 0x01f00000
boot_prefixes=/ /boot/overlays
boot_script_dhcp=boot.scr.uimg
boot_scripts=boot.scr.uimg boot.scr
boot_targets=mmc0 mmc1 usb0 pxe dhcp
bootcmd=run boot_normal
bootcmd_dhcp=run boot_net_usb_start; if dhcp ${scriptaddr} ${boot_script_dhcp}; then source ${scriptaddr}; fi;setenv efi_fdtfile ${fdtfile}; if test -z "${fdtfile}" -a -n "${soc}"; then setenv efi_fdtfile ${soc}-${board}${boardver}.dtb; fi; setenv efi_old_vci ${bootp_vci};setenv efi_old_arch ${bootp_arch};setenv bootp_vci PXEClient:Arch:00010:UNDI:003000;setenv bootp_arch 0xa;if dhcp ${kernel_addr_r}; then tftpboot ${fdt_addr_r} dtb/${efi_fdtfile};if fdt addr ${fdt_addr_r}; then bootefi ${kernel_addr_r} ${fdt_addr_r}; else bootefi ${kernel_addr_r} ${fdtcontroladdr};fi;fi;setenv bootp_vci ${efi_old_vci};setenv bootp_arch ${efi_old_arch};setenv efi_fdtfile;setenv efi_old_arch;setenv efi_old_vci;
bootcmd_mmc0=setenv devnum 0; run mmc_boot
bootcmd_mmc1=setenv devnum 1; run mmc_boot
bootcmd_pxe=run boot_net_usb_start; env set bootfile;dhcp; if pxe get; then pxe boot; fi
bootcmd_usb0=setenv devnum 0; run usb_boot
bootdelay=1
bootfstype=fat
console=ttyS1
cpu=armv7
devnum=0
devplist=1
devtype=mmc
distro_bootcmd=for target in ${boot_targets}; do run bootcmd_${target}; done
efi_dtb_prefixes=/ /dtb/ /dtb/current/
ethact=ethernet@ff290000
ethaddr=2c:4d:54:23:f0:b7
fdt_addr_r=0x01f00000
fdt_file=rk3288-miniarm.dtb
fdt_high=0x0fffffff
fdt_overlay_addr_r=0x01e00000
fdtcontroladdr=7df59f10
hw_conf_addr_r=0x00700000
initrd_high=0x0fffffff
kernel_addr_r=0x02000000
load_efi_dtb=load ${devtype} ${devnum}:${distro_bootpart} ${fdt_addr_r} ${prefix}${efi_fdtfile}
mmc_boot=if mmc dev ${devnum}; then setenv devtype mmc; run scan_dev_for_boot_part; fi
mmcargs=setenv bootargs console=ttyS1,115200 root=/dev/mmcblk0p2 rootwait rw rw rootfstype=ext4 rootwait init=/sbin/init
mmcroot=/dev/mmcblk0p2 rootwait rw
partitions=uuid_disk=${uuid_gpt_disk};name=loader1,start=32K,size=4000K,uuid=${uuid_gpt_loader1};name=reserved1,size=64K,uuid=${uuid_gpt_reserved1};name=reserved2,size=4M,uuid=${uuid_gpt_reserved2};name=loader2,size=4MB,uuid=${uuid_gpt_loader2};name=atf,size=4M,uuid=${uuid_gpt_atf};name=boot,size=112M,bootable,uuid=${uuid_gpt_boot};name=rootfs,size=-,uuid=69DAD710-2CE4-4E3C-B16C-21A1D49ABED3;
pxefile_addr_r=0x00100000
ramdisk_addr_r=0x04000000
scan_dev_for_boot=echo Scanning ${devtype} ${devnum}:${distro_bootpart}...; for prefix in ${boot_prefixes}; do run scan_dev_for_extlinux; run scan_dev_for_scripts; done;run scan_dev_for_efi;
scan_dev_for_boot_part=part list ${devtype} ${devnum} -bootable devplist; env exists devplist || setenv devplist 1; for distro_bootpart in ${devplist}; do if fstype ${devtype} ${devnum}:${distro_bootpart} bootfstype; then run scan_dev_for_boot; fi; done
scan_dev_for_efi=setenv efi_fdtfile ${fdtfile}; if test -z "${fdtfile}" -a -n "${soc}"; then setenv efi_fdtfile ${soc}-${board}${boardver}.dtb; fi; for prefix in ${efi_dtb_prefixes}; do if test -e ${devtype} ${devnum}:${distro_bootpart} ${prefix}${efi_fdtfile}; then run load_efi_dtb; fi;done;if test -e ${devtype} ${devnum}:${distro_bootpart} efi/boot/bootarm.efi; then echo Found EFI removable media binary efi/boot/bootarm.efi; run boot_efi_binary; echo EFI LOAD FAILED: continuing...; fi; setenv efi_fdtfile
scan_dev_for_extlinux=if test -e ${devtype} ${devnum}:${distro_bootpart} ${prefix}extlinux/extlinux.conf; then echo Found ${prefix}extlinux/extlinux.conf; run boot_extlinux; echo SCRIPT FAILED: continuing...; fi
scan_dev_for_scripts=for script in ${boot_scripts}; do if test -e ${devtype} ${devnum}:${distro_bootpart} ${prefix}${script}; then echo Found U-Boot script ${prefix}${script}; run boot_a_script; echo SCRIPT FAILED: continuing...; fi; done
scriptaddr=0x00000000
soc=rockchip
usb_boot=usb start; if usb dev ${devnum}; then setenv devtype usb; run scan_dev_for_boot_part; fi
vendor=rockchip
 
Environment size: 4686/8188 bytes
=>
```

### Uboot Board Info

```
=> bdinfo
arch_number = 0x00000000
boot_params = 0x00000000
DRAM bank   = 0x00000000
-> start    = 0x00000000
-> size     = 0x80000000
baudrate    = 115200 bps
TLB addr    = 0x7FFF0000
relocaddr   = 0x7FF66000
reloc off   = 0x7FF66000
irq_sp      = 0x7DF59F00
sp start    = 0x7DF59EF0
Early malloc usage: 9e8 / 2000
fdt_blob = 7df59f10
```

### Kernel DMESG & Rootfs Messages

```
=> run bootcmd
reading rk3288-miniarm.dtb
68823 bytes read in 8 ms (8.2 MiB/s)
reading zimage
7697224 bytes read in 358 ms (20.5 MiB/s)
## Flattened Device Tree blob at 01f00000
   Booting using the fdt blob at 0x1f00000
   Loading Device Tree to 0ffec000, end 0ffffcd6 ... OK
 
Starting kernel ...
 
[    0.000000] Booting Linux on physical CPU 0x500
[    0.000000] Initializing cgroup subsys cpuset
[    0.000000] Initializing cgroup subsys cpu
[    0.000000] Initializing cgroup subsys cpuacct
[    0.000000] Linux version 4.4.103+ (ashley@ubuntu) (gcc version 5.4.0 20160609 (Ubuntu/Linaro 5.4.0-6ubuntu1~16.04.9) ) #1 SMP Sat Mar 17 13:08:35 GMT 2018
[    0.000000] CPU: ARMv7 Processor [410fc0d1] revision 1 (ARMv7), cr=10c5387d
[    0.000000] CPU: PIPT / VIPT nonaliasing data cache, VIPT aliasing instruction cache
[    0.000000] Machine model:
[    0.000000] cma: Reserved 128 MiB at 0x78000000
[    0.000000] Memory policy: Data cache writealloc
[    0.000000] PERCPU: Embedded 14 pages/cpu @eef7f000 s24728 r8192 d24424 u57344
[    0.000000] Built 1 zonelists in Zone order, mobility grouping on.  Total pages: 522752
[    0.000000] Kernel command line: console=ttyS1,115200 root=/dev/mmcblk0p2 rootwait rw rw rootfstype=ext4 rootwait init=/sbin/init
[    0.000000] PID hash table entries: 4096 (order: 2, 16384 bytes)
[    0.000000] Dentry cache hash table entries: 131072 (order: 7, 524288 bytes)
[    0.000000] Inode-cache hash table entries: 65536 (order: 6, 262144 bytes)
[    0.000000] Memory: 1929364K/2097152K available (11264K kernel code, 961K rwdata, 3172K rodata, 1024K init, 572K bss, 36716K reserved, 131072K cma-reserved, 1179648K highmem)
[    0.000000] Virtual kernel memory layout:
[    0.000000]     vector  : 0xffff0000 - 0xffff1000   (   4 kB)
[    0.000000]     fixmap  : 0xffc00000 - 0xfff00000   (3072 kB)
[    0.000000]     vmalloc : 0xf0800000 - 0xff800000   ( 240 MB)
[    0.000000]     lowmem  : 0xc0000000 - 0xf0000000   ( 768 MB)
[    0.000000]     pkmap   : 0xbfe00000 - 0xc0000000   (   2 MB)
[    0.000000]     modules : 0xbf000000 - 0xbfe00000   (  14 MB)
[    0.000000]       .text : 0xc0008000 - 0xc0c00000   (12256 kB)
[    0.000000]       .init : 0xc1100000 - 0xc1200000   (1024 kB)
[    0.000000]       .data : 0xc1200000 - 0xc12f0440   ( 962 kB)
[    0.000000]        .bss : 0xc12f2000 - 0xc13813a4   ( 573 kB)
[    0.000000] SLUB: HWalign=64, Order=0-3, MinObjects=0, CPUs=4, Nodes=1
[    0.000000] Hierarchical RCU implementation.
[    0.000000]  Build-time adjustment of leaf fanout to 32.
[    0.000000] NR_IRQS:16 nr_irqs:16 16
[    0.000000] L2C: failed to init: -19
[    0.000000] rockchip_clk_register_branches: unknown clock type 9
[    0.000000] Architected cp15 timer(s) running at 24.00MHz (phys).
[    0.000000] clocksource: arch_sys_counter: mask: 0xffffffffffffff max_cycles: 0x588fe9dc0, max_idle_ns: 440795202592 ns
[    0.000008] sched_clock: 56 bits at 24MHz, resolution 41ns, wraps every 4398046511097ns
[    0.000029] Switching to timer-based delay loop, resolution 41ns
[    0.003488] Console: colour dummy device 80x30
[    0.003531] Calibrating delay loop (skipped), value calculated using timer frequency.. 48.00 BogoMIPS (lpj=240000)
[    0.003555] pid_max: default: 32768 minimum: 301
[    0.003693] Security Framework initialized
[    0.003711] Yama: becoming mindful.
[    0.003796] Mount-cache hash table entries: 2048 (order: 1, 8192 bytes)
[    0.003817] Mountpoint-cache hash table entries: 2048 (order: 1, 8192 bytes)
[    0.004962] Initializing cgroup subsys devices
[    0.004992] Initializing cgroup subsys freezer
[    0.005064] CPU: Testing write buffer coherency: ok
[    0.005114] ftrace: allocating 42290 entries in 125 pages
[    0.126102] CPU0: update cpu_capacity 430
[    0.126123] CPU0: thread -1, cpu 0, socket 5, mpidr 80000500
[    0.126516] Setting up static identity map for 0x100000 - 0x100058
[    0.130217] CPU1: update cpu_capacity 430
[    0.130226] CPU1: thread -1, cpu 1, socket 5, mpidr 80000501
[    0.132239] CPU2: update cpu_capacity 430
[    0.132248] CPU2: thread -1, cpu 2, socket 5, mpidr 80000502
[    0.134252] CPU3: update cpu_capacity 430
[    0.134261] CPU3: thread -1, cpu 3, socket 5, mpidr 80000503
[    0.134390] Brought up 4 CPUs
[    0.134432] SMP: Total of 4 processors activated (192.00 BogoMIPS).
[    0.134441] CPU: All CPU(s) started in SVC mode.
[    0.136310] devtmpfs: initialized
[    0.162563] VFP support v0.3: implementor 41 architecture 3 part 30 variant d rev 0
[    0.163069] clocksource: jiffies: mask: 0xffffffff max_cycles: 0xffffffff, max_idle_ns: 19112604462750000 ns
[    0.163108] futex hash table entries: 1024 (order: 4, 65536 bytes)
[    0.169281] xor: measuring software checksum speed
[    0.265955]    arm4regs  :  1038.000 MB/sec
[    0.366050]    8regs     :   801.200 MB/sec
[    0.466151]    32regs    :   799.600 MB/sec
[    0.466164] xor: using function: arm4regs (1038.000 MB/sec)
[    0.466207] pinctrl core: initialized pinctrl subsystem
[    0.467644] NET: Registered protocol family 16
[    0.469687] DMA: preallocated 256 KiB pool for atomic coherent allocations
[    0.496263] cpuidle: using governor ladder
[    0.526299] cpuidle: using governor menu
[    0.572759] hw-breakpoint: found 5 (+1 reserved) breakpoint and 4 watchpoint registers.
[    0.572774] hw-breakpoint: maximum watchpoint size is 4 bytes.
[    0.786953] raid6: int32x1  gen()    87 MB/s
[    0.956968] raid6: int32x1  xor()   110 MB/s
[    1.127648] raid6: int32x2  gen()   118 MB/s
[    1.297533] raid6: int32x2  xor()   125 MB/s
[    1.468139] raid6: int32x4  gen()   102 MB/s
[    1.638045] raid6: int32x4  xor()   100 MB/s
[    1.808489] raid6: int32x8  gen()   128 MB/s
[    1.978598] raid6: int32x8  xor()   108 MB/s
[    1.978611] raid6: using algorithm int32x8 gen() 128 MB/s
[    1.978621] raid6: .... xor() 108 MB/s, rmw enabled
[    1.978631] raid6: using intx1 recovery algorithm
[    1.981803] iommu: Adding device ff910000.cif_isp to group 0
[    1.981958] iommu: Adding device ff930000.vop to group 1
[    1.982091] iommu: Adding device ff940000.vop to group 2
[    1.982227] iommu: Adding device ff9a0000.vpu-service to group 3
[    1.982363] iommu: Adding device ff9c0000.hevc-service to group 4
[    1.985405] SCSI subsystem initialized
[    1.985773] usbcore: registered new interface driver usbfs
[    1.985868] usbcore: registered new interface driver hub
[    1.985968] usbcore: registered new device driver usb
[    1.986174] Linux cec interface: v0.10
[    1.986279] Linux video capture interface: v2.00
[    1.986377] pps_core: LinuxPPS API ver. 1 registered
[    1.986389] pps_core: Software ver. 5.3.6 - Copyright 2005-2007 Rodolfo Giometti <giometti@linux.it>
[    1.986428] PTP clock support registered
[    1.987479] rockchip ion: success to create - cma-heap
[    1.987509] rockchip ion: success to create - system-heap
[    1.989909] Advanced Linux Sound Architecture Driver Initialized.
[    1.990890] Bluetooth: Core ver 2.21
[    1.990949] NET: Registered protocol family 31
[    1.990961] Bluetooth: HCI device and connection manager initialized
[    1.990982] Bluetooth: HCI socket layer initialized
[    1.990999] Bluetooth: L2CAP socket layer initialized
[    1.991044] Bluetooth: SCO socket layer initialized
[    1.992937] clocksource: Switched to clocksource arch_sys_counter
[    2.071348] FS-Cache: Loaded
[    2.090326] NET: Registered protocol family 2
[    2.091211] TCP established hash table entries: 8192 (order: 3, 32768 bytes)
[    2.091326] TCP bind hash table entries: 8192 (order: 5, 163840 bytes)
[    2.091588] TCP: Hash tables configured (established 8192 bind 8192)
[    2.091672] UDP hash table entries: 512 (order: 2, 24576 bytes)
[    2.091735] UDP-Lite hash table entries: 512 (order: 2, 24576 bytes)
[    2.092047] NET: Registered protocol family 1
[    2.092546] RPC: Registered named UNIX socket transport module.
[    2.092561] RPC: Registered udp transport module.
[    2.092572] RPC: Registered tcp transport module.
[    2.092581] RPC: Registered tcp NFSv4.1 backchannel transport module.
[    2.093883] hw perfevents: enabled with armv7_cortex_a12 PMU driver, 7 counters available
[    2.097134] Initialise system trusted keyring
[    2.112430] squashfs: version 4.0 (2009/01/31) Phillip Lougher
[    2.115434] NFS: Registering the id_resolver key type
[    2.115508] Key type id_resolver registered
[    2.115520] Key type id_legacy registered
[    2.115560] nfs4filelayout_init: NFSv4 File Layout Driver Registering...
[    2.115600] Installing knfsd (copyright (C) 1996 okir@monad.swb.de).
[    2.116905] FS-Cache: Netfs 'cifs' registered for caching
[    2.117434] Key type cifs.spnego registered
[    2.117483] Key type cifs.idmap registered
[    2.117515] ntfs: driver 2.1.32 [Flags: R/W DEBUG].
[    2.118021] fuse init (API version 7.23)
[    2.119009] SGI XFS with ACLs, security attributes, realtime, no debug enabled
[    2.128124] NET: Registered protocol family 38
[    2.128168] async_tx: api initialized (async)
[    2.128200] Key type asymmetric registered
[    2.128219] Asymmetric key parser 'x509' registered
[    2.128338] bounce: pool size: 64 pages
[    2.128620] Block layer SCSI generic (bsg) driver version 0.4 loaded (major 247)
[    2.128646] io scheduler noop registered
[    2.128666] io scheduler deadline registered
[    2.128720] io scheduler cfq registered (default)
[    2.129314] rockchip-usb-phy ff770000.syscon:usbphy: vbus_drv is not assigned!
[    2.133539] rk-vcodec ff9a0000.vpu-service: probe device
[    2.133628] rk-vcodec ff9a0000.vpu-service: vpu mmu dec eeb03210
[    2.133951] rk-vcodec ff9a0000.vpu-service: allocator is drm
[    2.134048] rk-vcodec ff9a0000.vpu-service: checking hw id 4831
[    2.134862] rk-vcodec ff9a0000.vpu-service: init success
[    2.135678] rk-vcodec ff9c0000.hevc-service: probe device
[    2.135772] rk-vcodec ff9c0000.hevc-service: vpu mmu dec eeb03610
[    2.136074] rk-vcodec ff9c0000.hevc-service: allocator is drm
[    2.136166] rk-vcodec ff9c0000.hevc-service: checking hw id 6867
[    2.136749] rk-vcodec ff9c0000.hevc-service: init success
[    2.140278] dma-pl330 ff250000.dma-controller: Loaded driver for PL330 DMAC-241330
[    2.140302] dma-pl330 ff250000.dma-controller:       DBUFF-128x8bytes Num_Chans-8 Num_Peri-20 Num_Events-16
[    2.141575] dma-pl330 ffb20000.dma-controller: Loaded driver for PL330 DMAC-241330
[    2.141598] dma-pl330 ffb20000.dma-controller:       DBUFF-64x8bytes Num_Chans-5 Num_Peri-6 Num_Events-10
[    2.142554] Serial: 8250/16550 driver, 5 ports, IRQ sharing disabled
[    2.145730] ff180000.serial: ttyS0 at MMIO 0xff180000 (irq = 39, base_baud = 1500000) is a 16550A
[    2.147035] console [ttyS1] disabled
[    2.147112] ff190000.serial: ttyS1 at MMIO 0xff190000 (irq = 40, base_baud = 1500000) is a 16550A
[    3.180731] console [ttyS1] enabled
[    3.185797] ff690000.serial: ttyS2 at MMIO 0xff690000 (irq = 41, base_baud = 1500000) is a 16550A
[    3.196804] ff1b0000.serial: ttyS3 at MMIO 0xff1b0000 (irq = 42, base_baud = 1500000) is a 16550A
[    3.207966] ff1c0000.serial: ttyS4 at MMIO 0xff1c0000 (irq = 43, base_baud = 1500000) is a 16550A
[    3.220469] gpiomem-rk3288 ff750000.rk3288-gpiomem: Initialised: Registers at 0xff750000
[    3.229584] [drm:drm_core_init] Initialized drm 1.1.0 20060810
[    3.240975] usbcore: registered new interface driver udl
[    3.247659] mali ffa30000.gpu: Failed to get regulator
[    3.253378] mali ffa30000.gpu: Power control initialization failed
[    3.261976] brd: module loaded
[    3.275315] loop: module loaded
[    3.279894] zram: Added device: zram0
[    3.284018] lkdtm: No crash points registered, enable through debugfs
[    3.292344] rockchip-pinctrl pinctrl: pin gpio5-12 already requested by ff1c0000.serial; cannot claim for ff110000.spi
[    3.304212] rockchip-pinctrl pinctrl: pin-164 (ff110000.spi) status -22
[    3.311536] rockchip-pinctrl pinctrl: could not request pin 164 (gpio5-12) from group spi0-clk  on device rockchip-pinctrl
[    3.323771] rockchip-spi ff110000.spi: Error applying setting, reverse things back
[    3.337648] tun: Universal TUN/TAP device driver, 1.6
[    3.343274] tun: (C) 1999-2004 Max Krasnyansky <maxk@qualcomm.com>
[    3.350409] CAN device driver interface
[    3.356373] rk_gmac-dwmac ff290000.ethernet: phy regulator is not available yet, deferred probing
[    3.367422] PPP generic driver version 2.4.2
[    3.372588] usbcore: registered new interface driver rndis_wlan
[    3.379501] usbcore: registered new interface driver rt2800usb
[    3.386011] Rockchip WiFi SYS interface (V1.00) ...
[    3.391572] pegasus: v0.9.3 (2013/04/25), Pegasus/Pegasus II USB Ethernet driver
[    3.399873] usbcore: registered new interface driver pegasus
[    3.406255] usbcore: registered new interface driver rtl8150
[    3.412599] usbcore: registered new interface driver r8152
[    3.418799] usbcore: registered new interface driver asix
[    3.424899] usbcore: registered new interface driver ax88179_178a
[    3.431723] usbcore: registered new interface driver cdc_ether
[    3.438297] usbcore: registered new interface driver dm9601
[    3.444599] usbcore: registered new interface driver smsc75xx
[    3.451067] usbcore: registered new interface driver smsc95xx
[    3.457547] usbcore: registered new interface driver net1080
[    3.463926] usbcore: registered new interface driver rndis_host
[    3.470570] usbcore: registered new interface driver MOSCHIP usb-ethernet driver
[    3.478901] usbcore: registered new interface driver ipheth
[    3.485207] usbcore: registered new interface driver cdc_ncm
[    3.491550] usbcore: registered new interface driver cdc_mbim
[    3.498357] ff540000.usb supply vusb_d not found, using dummy regulator
[    3.505789] ff540000.usb supply vusb_a not found, using dummy regulator
[    3.693274] dwc2 ff540000.usb: DWC OTG Controller
[    3.698522] dwc2 ff540000.usb: new USB bus registered, assigned bus number 1
[    3.706397] dwc2 ff540000.usb: irq 48, io mem 0x00000000
[    3.712626] usb usb1: New USB device found, idVendor=1d6b, idProduct=0002
[    3.720175] usb usb1: New USB device strings: Mfr=3, Product=2, SerialNumber=1
[    3.728188] usb usb1: Product: DWC OTG Controller
[    3.733412] usb usb1: Manufacturer: Linux 4.4.103+ dwc2_hsotg
[    3.739769] usb usb1: SerialNumber: ff540000.usb
[    3.745846] hub 1-0:1.0: USB hub found
[    3.750052] hub 1-0:1.0: 1 port detected
[    3.755392] ff580000.usb supply vusb_d not found, using dummy regulator
[    3.762793] ff580000.usb supply vusb_a not found, using dummy regulator
[    4.103012] dwc2 ff580000.usb: EPs: 10, dedicated fifos, 972 entries in SPRAM
[    4.111570] dwc2 ff580000.usb: DWC OTG Controller
[    4.116853] dwc2 ff580000.usb: new USB bus registered, assigned bus number 2
[    4.124714] dwc2 ff580000.usb: irq 49, io mem 0x00000000
[    4.130843] usb usb2: New USB device found, idVendor=1d6b, idProduct=0002
[    4.138407] usb usb2: New USB device strings: Mfr=3, Product=2, SerialNumber=1
[    4.146427] usb usb2: Product: DWC OTG Controller
[    4.151632] usb usb2: Manufacturer: Linux 4.4.103+ dwc2_hsotg
[    4.158011] usb usb2: SerialNumber: ff580000.usb
[    4.163203] usb 1-1: new high-speed USB device number 2 using dwc2
[    4.164101] hub 2-0:1.0: USB hub found
[    4.164159] hub 2-0:1.0: 1 port detected
[    4.166267] ehci_hcd: USB 2.0 'Enhanced' Host Controller (EHCI) Driver
[    4.166282] ehci-platform: EHCI generic platform driver
[    4.166663] ehci-platform ff500000.usb: EHCI Host Controller
[    4.166985] ehci-platform ff500000.usb: new USB bus registered, assigned bus number 3
[    4.167212] ehci-platform ff500000.usb: irq 47, io mem 0xff500000
[    4.183004] ehci-platform ff500000.usb: USB 2.0 started, EHCI 1.00
[    4.183294] usb usb3: New USB device found, idVendor=1d6b, idProduct=0002
[    4.183305] usb usb3: New USB device strings: Mfr=3, Product=2, SerialNumber=1
[    4.183314] usb usb3: Product: EHCI Host Controller
[    4.183322] usb usb3: Manufacturer: Linux 4.4.103+ ehci_hcd
[    4.183330] usb usb3: SerialNumber: ff500000.usb
[    4.184280] hub 3-0:1.0: USB hub found
[    4.184338] hub 3-0:1.0: 1 port detected
[    4.185424] usbcore: registered new interface driver cdc_acm
[    4.185428] cdc_acm: USB Abstract Control Model driver for USB modems and ISDN adapters
[    4.185526] usbcore: registered new interface driver cdc_wdm
[    4.185730] usbcore: registered new interface driver usb-storage
[    4.185888] usbcore: registered new interface driver usbserial
[    4.185967] usbcore: registered new interface driver usbserial_generic
[    4.186030] usbserial: USB Serial support registered for generic
[    4.186124] usbcore: registered new interface driver ch341
[    4.186174] usbserial: USB Serial support registered for ch341-uart
[    4.186266] usbcore: registered new interface driver cp210x
[    4.186316] usbserial: USB Serial support registered for cp210x
[    4.186447] usbcore: registered new interface driver ftdi_sio
[    4.186498] usbserial: USB Serial support registered for FTDI USB Serial Device
[    4.186801] usbcore: registered new interface driver keyspan
[    4.186853] usbserial: USB Serial support registered for Keyspan - (without firmware)
[    4.186902] usbserial: USB Serial support registered for Keyspan 1 port adapter
[    4.186950] usbserial: USB Serial support registered for Keyspan 2 port adapter
[    4.187017] usbserial: USB Serial support registered for Keyspan 4 port adapter
[    4.187117] usbcore: registered new interface driver option
[    4.187168] usbserial: USB Serial support registered for GSM modem (1-port)
[    4.187571] usbcore: registered new interface driver oti6858
[    4.187622] usbserial: USB Serial support registered for oti6858
[    4.187713] usbcore: registered new interface driver pl2303
[    4.187763] usbserial: USB Serial support registered for pl2303
[    4.187866] usbcore: registered new interface driver qcserial
[    4.187917] usbserial: USB Serial support registered for Qualcomm USB modem
[    4.188040] usbcore: registered new interface driver sierra
[    4.188091] usbserial: USB Serial support registered for Sierra USB modem
[    4.189471] usbcore: registered new interface driver iforce
[    4.189597] usbcore: registered new interface driver xpad
[    4.189942] usbcore: registered new interface driver usbtouchscreen
[    4.190612] i2c /dev entries driver
[    4.192616] rk808 0-001b: Pmic Chip id: 0x0
[    4.196349] DCDC_REG1: supplied by vcc_sys
[    4.197378] DCDC_REG2: supplied by vcc_sys
[    4.198256] DCDC_REG3: supplied by vcc_sys
[    4.198717] DCDC_REG4: supplied by vcc_sys
[    4.199320] vcc_sd: supplied by vcc_io
[    4.199399] vcc_flash: supplied by vcc_io
[    4.199575] LDO_REG1: supplied by vcc_sys
[    4.200953] LDO_REG2: supplied by vcc_sys
[    4.202145] LDO_REG3: supplied by vcc_sys
[    4.203337] LDO_REG4: supplied by vcc_io
[    4.204523] LDO_REG5: supplied by vcc_io
[    4.205700] LDO_REG6: supplied by vcc_io
[    4.206929] LDO_REG7: supplied by vcc_sys
[    4.208128] LDO_REG8: supplied by vcc_sys
[    4.209331] SWITCH_REG1: supplied by vcc_io
[    4.209756] SWITCH_REG2: supplied by vcc_io
[    4.213847] rk808-rtc rk808-rtc: rtc core: registered rk808-rtc as rtc0
[    4.214370] rk3x-i2c ff650000.i2c: Initialized RK3xxx I2C bus at f091a000
[    4.215297] rk3x-i2c ff140000.i2c: Initialized RK3xxx I2C bus at f091c000
[    4.216358] tinker-mcu: tinker_mcu_probe: address = 0x45
[    4.216364] tinker-mcu: send_cmds: 80
[    4.216565] tinker-mcu: send_cmds: send command failed, ret = -6
[    4.216570] tinker-mcu: tinker_mcu_probe: init_cmd_check failed, -6
[    4.216875] tinker-ft5406: tinker_ft5406_probe: address = 0x38
[    4.502984] usb 3-1: new high-speed USB device number 2 using ehci-platform
[    4.812968] tinker-ft5406: tinker_ft5406_probe: wait connected timeout
[    4.820245] rk3x-i2c ff150000.i2c: Initialized RK3xxx I2C bus at f091e000
[    4.824556] usb 1-1: New USB device found, idVendor=05e3, idProduct=0610
[    4.824568] usb 1-1: New USB device strings: Mfr=0, Product=1, SerialNumber=0
[    4.824577] usb 1-1: Product: USB2.0 Hub
[    4.825678] hub 1-1:1.0: USB hub found
[    4.826041] hub 1-1:1.0: 4 ports detected
[    4.856100] usb 3-1: config 1 has an invalid interface number: 255 but max is 6
[    4.864236] usb 3-1: config 1 has no interface number 6
[    4.871137] rk3x-i2c ff160000.i2c: Initialized RK3xxx I2C bus at f0920000
[    4.879541] usb 3-1: New USB device found, idVendor=0bda, idProduct=481a
[    4.887007] usb 3-1: New USB device strings: Mfr=3, Product=1, SerialNumber=2
[    4.894936] usb 3-1: Product: USB Audio
[    4.896586] at24 2-0050: 1024 byte 24c08 EEPROM, writable, 1 bytes/write
[    4.896618] rk3x-i2c ff660000.i2c: Initialized RK3xxx I2C bus at f0922000
[    4.897713] imx219 2-0010: probing...
[    4.897732] imx219 2-0010: probing successful
[    4.898127] ov7750 2-0036: probing...
[    4.898144] ov7750 2-0036: probing successful
[    4.899223] lirc_dev: IR Remote Control driver registered, major 242
[    4.899239] IR NEC protocol handler initialized
[    4.899251] IR RC5(x/sz) protocol handler initialized
[    4.899262] IR RC6 protocol handler initialized
[    4.899273] IR JVC protocol handler initialized
[    4.899284] IR Sony protocol handler initialized
[    4.899295] IR SANYO protocol handler initialized
[    4.899306] IR Sharp protocol handler initialized
[    4.899318] IR MCE Keyboard/mouse protocol handler initialized
[    4.899329] IR LIRC bridge handler initialized
[    4.899340] IR XMP protocol handler initialized
[    4.899953] usbcore: registered new interface driver uvcvideo
[    4.899957] USB Video Class driver (1.1.1)
[    4.900240] Driver for 1-wire Dallas network protocol.
[    4.904083] clk: couldn't get clock 0 for /tsadc@ff280000
[    4.905311] md: linear personality registered for level -1
[    4.905326] md: raid0 personality registered for level 0
[    4.905339] md: raid1 personality registered for level 1
[    4.905351] md: raid10 personality registered for level 10
[    4.923465] md: raid6 personality registered for level 6
[    4.923470] md: raid5 personality registered for level 5
[    4.923475] md: raid4 personality registered for level 4
[    4.923495] md: multipath personality registered for level -4
[    4.923508] md: faulty personality registered for level -5
[    4.924621] device-mapper: ioctl: 4.34.0-ioctl (2015-10-28) initialised: dm-devel@redhat.com
[    4.924924] Bluetooth: Virtual HCI driver ver 1.5
[    4.925175] Bluetooth: HCI UART driver ver 2.3
[    4.925182] Bluetooth: HCI UART protocol H4 registered
[    4.925188] Bluetooth: HCI UART protocol Three-wire (H5) registered
[    4.925203] rtk_btcoex: rtk_btcoex_init: version: 1.2
[    4.925206] rtk_btcoex: create workqueue
[    4.925683] rtk_btcoex: alloc buffers 1408, 2240 for ev and l2
[    4.925839] usbcore: registered new interface driver bfusb
[    4.925958] usbcore: registered new interface driver btusb
[    4.925975] Bluetooth: Generic Bluetooth SDIO driver ver 0.1
[    4.926982] cpu cpu0: chip_version: 0 0x1
[    4.950040] cpu cpu0: Failed to get cpu_reg
[    4.950194] cpufreq-dt cpufreq-dt: failed register driver: -17
[    4.950199] cpufreq-dt: probe of cpufreq-dt failed with error -17
[    4.950242] sdhci: Secure Digital Host Controller Interface driver
[    4.950243] sdhci: Copyright(c) Pierre Ossman
[    4.950247] Synopsys Designware Multimedia Card Interface Driver
[    4.950688] dwmmc_rockchip ff0c0000.dwmmc: IDMAC supports 32-bit address mode.
[    4.950728] dwmmc_rockchip ff0c0000.dwmmc: Using internal DMA controller.
[    4.950735] dwmmc_rockchip ff0c0000.dwmmc: Version ID is 270a
[    4.950759] dwmmc_rockchip ff0c0000.dwmmc: DW MMC controller at irq 29,32 bit host data width,256 deep fifo
[    4.950772] dwmmc_rockchip ff0c0000.dwmmc: 'clock-freq-min-max' property was deprecated.
[    4.950969] rockchip-iodomain ff770000.syscon:io-domains: Setting to 3300000 done
[    4.950973] rockchip-iodomain io-domains: Setting to 3300000 done
[    4.964792] rockchip-iodomain ff770000.syscon:io-domains: Setting to 3300000 done
[    4.964796] rockchip-iodomain io-domains: Setting to 3300000 done
[    4.992943] mmc_host mmc0: Bus speed (slot 0) = 400000Hz (slot req 400000Hz, actual 400000HZ div = 0)
[    5.013957] dwmmc_rockchip ff0c0000.dwmmc: 1 slots initialized
[    5.014198] dwmmc_rockchip ff0d0000.dwmmc: IDMAC supports 32-bit address mode.
[    5.014228] dwmmc_rockchip ff0d0000.dwmmc: Using internal DMA controller.
[    5.014234] dwmmc_rockchip ff0d0000.dwmmc: Version ID is 270a
[    5.014255] dwmmc_rockchip ff0d0000.dwmmc: DW MMC controller at irq 30,32 bit host data width,256 deep fifo
[    5.014264] dwmmc_rockchip ff0d0000.dwmmc: 'clock-freq-min-max' property was deprecated.
[    5.014283] dwmmc_rockchip ff0d0000.dwmmc: No vmmc regulator found
[    5.014286] dwmmc_rockchip ff0d0000.dwmmc: No vqmmc regulator found
[    5.024658] dwmmc_rockchip ff0f0000.dwmmc: set maskrom gpio to enable emmc
[    5.024672] dwmmc_rockchip ff0f0000.dwmmc: IDMAC supports 32-bit address mode.
[    5.026711] dwmmc_rockchip ff0f0000.dwmmc: Using internal DMA controller.
[    5.026717] dwmmc_rockchip ff0f0000.dwmmc: Version ID is 270a
[    5.026741] dwmmc_rockchip ff0f0000.dwmmc: DW MMC controller at irq 31,32 bit host data width,256 deep fifo
[    5.026749] dwmmc_rockchip ff0f0000.dwmmc: 'clock-freq-min-max' property was deprecated.
[    5.026766] dwmmc_rockchip ff0f0000.dwmmc: No vmmc regulator found
[    5.026768] dwmmc_rockchip ff0f0000.dwmmc: No vqmmc regulator found
[    5.052951] mmc_host mmc1: Bus speed (slot 0) = 400000Hz (slot req 400000Hz, actual 400000HZ div = 0)
[    5.072965] dwmmc_rockchip ff0f0000.dwmmc: 1 slots initialized
[    5.073105] sdhci-pltfm: SDHCI platform and OF driver helper
[    5.073489] ledtrig-cpu: registered to indicate activity on CPUs
[    5.073535] hidraw: raw HID events driver (C) Jiri Kosina
[    5.075035] usbcore: registered new interface driver usbhid
[    5.075037] usbhid: USB HID core driver
[    5.075079] dtbocfg_module_init
[    5.075097] dtbocfg_module_init: OK
[    5.075599] ashmem: initialized
[    5.076790] find panel: asus,tc358762
[    5.076803] ff960000.dsi.0 supply power not found, using dummy regulator
[    5.077074] rockchip-vop ff940000.vop: invalid resource
[    5.077076] rockchip-vop ff940000.vop: failed to get vop cabc lut registers
[    5.077420] rockchip-drm display-subsystem: bound ff940000.vop (ops 0xc0c78214)
[    5.077445] rockchip-vop ff930000.vop: invalid resource
[    5.077448] rockchip-vop ff930000.vop: failed to get vop cabc lut registers
[    5.077779] rockchip-drm display-subsystem: bound ff930000.vop (ops 0xc0c78214)
[    5.077809] panel doesn't be connected
[    5.077815] rockchip-drm display-subsystem: bound ff960000.dsi (ops 0xc0cce838)
[    5.077844] dwhdmi-rockchip ff980000.hdmi: hdmi_i2s_audio_disable=false
[    5.078076] i2c i2c-6: of_i2c: modalias failure on /hdmi@ff980000/ports
[    5.078088] dwhdmi-rockchip ff980000.hdmi: registered DesignWare HDMI I2C bus driver
[    5.078126] dwhdmi-rockchip ff980000.hdmi: Detected HDMI TX controller v2.00a with HDCP (DWC MHL PHY)
[    5.078677] Registered IR keymap rc-cec
[    5.078857] input: RC for dw_hdmi as /devices/platform/ff980000.hdmi/rc/rc0/input0
[    5.078962] rc0: RC for dw_hdmi as /devices/platform/ff980000.hdmi/rc/rc0
[    5.079220] rockchip-drm display-subsystem: bound ff980000.hdmi (ops 0xc0c72e30)
[    5.079228] [drm:drm_vblank_init] Supports vblank timestamp caching Rev 2 (21.10.2013).
[    5.079230] [drm:drm_vblank_init] No driver support for vblank timestamp query.
[    5.079305] rockchip-drm display-subsystem: failed to parse display resources
[    5.079323] rockchip-drm display-subsystem: No connectors reported connected with modes
[    5.079330] [drm:drm_fb_helper_single_fb_probe] Cannot find any crtc or sizes - going 1024x768
[    5.164730] rockchip-iodomain ff770000.syscon:io-domains: Setting to 3300000 done
[    5.164735] rockchip-iodomain io-domains: Setting to 3300000 done
[    5.175154] rockchip-iodomain ff770000.syscon:io-domains: Setting to 1800000 done
[    5.175158] rockchip-iodomain io-domains: Setting to 1800000 done
[    5.398193] mmc_host mmc0: Bus speed (slot 0) = 148500000Hz (slot req 150000000Hz, actual 148500000HZ div = 0)
[    5.618046] dwmmc_rockchip ff0f0000.dwmmc: Busy; trying anyway
[    5.618051] mmc_host mmc1: Timeout sending command (cmd 0x202000 arg 0x0 status 0x0)
[    5.665853] usb 3-1: Manufacturer: Generic
[    5.665857] usb 3-1: SerialNumber: 201405280001
[    5.671511] Console: switching to colour frame buffer device 128x48
[    5.687119] input: Generic USB Audio as /devices/platform/ff500000.usb/usb3/3-1/3-1:1.255/0003:0BDA:481A.0001/input/input1
[    5.705642] rockchip-drm display-subsystem: fb0:  frame buffer device
[    5.743212] hid-generic 0003:0BDA:481A.0001: input,hiddev0,hidraw0: USB HID v1.11 Device [Generic USB Audio] on usb-ff500000.usb-1/input255
[    5.743484] [board_info] create Board_info_proc_file sucessed!
[    5.743537] project_id_2:0x1, project_id_1:0x1, project_id_0:0x1
[    5.743569] ram_id_2:0x0, ram_id_1:0x1, ram_id_0:0x0
[    5.743599] pcb_id_2:0x0, pcb_id_1:0x1, pcb_id_0:0x0
[    5.775855] dwmmc_rockchip ff0c0000.dwmmc: Successfully tuned phase to 150
[    5.775866] mmc0: new ultra high speed SDR104 SDHC card at address 0007
[    5.776138] mmcblk0: mmc0:0007 SD16G 14.4 GiB
[    5.789097]  mmcblk0: p1 p2
[    7.933141] usbcore: registered new interface driver snd-usb-audio
[    7.949424] u32 classifier
[    7.952425] Netfilter messages via NETLINK v0.30.
[    7.957671] nfnl_acct: registering with nfnetlink.
[    7.963050] nf_conntrack version 0.5.0 (16384 buckets, 65536 max)
[    7.970340] ctnetlink v0.93: registering with nfnetlink.
[    7.976977] xt_time: kernel timezone is -0000
[    7.981805] ip_set: protocol 6
[    7.985278] IPVS: Registered protocols (TCP, UDP, SCTP, AH, ESP)
[    7.991957] IPVS: Connection hash table configured (size=4096, memory=32Kbytes)
[    8.000187] IPVS: Creating netns size=1496 id=0
[    8.005242] IPVS: ipvs loaded.
[    8.008622] IPVS: [rr] scheduler registered.
[    8.013399] IPVS: [wrr] scheduler registered.
[    8.018217] IPVS: [lc] scheduler registered.
[    8.022962] IPVS: [wlc] scheduler registered.
[    8.027791] IPVS: [lblc] scheduler registered.
[    8.032711] IPVS: [lblcr] scheduler registered.
[    8.037741] IPVS: [dh] scheduler registered.
[    8.042463] IPVS: [sh] scheduler registered.
[    8.047199] IPVS: [sed] scheduler registered.
[    8.052015] IPVS: [nq] scheduler registered.
[    8.056753] IPVS: ftp: loaded support on port[0] = 21
[    8.062339] IPVS: [sip] pe registered.
[    8.067117] ip_tables: (C) 2000-2006 Netfilter Core Team
[    8.073098] ipt_CLUSTERIP: ClusterIP Version 0.8 loaded successfully
[    8.080136] arp_tables: (C) 2002 David S. Miller
[    8.085272] Initializing XFRM netlink socket
[    8.090415] NET: Registered protocol family 10
[    8.095772] ip6_tables: (C) 2000-2006 Netfilter Core Team
[    8.101928] sit: IPv6 over IPv4 tunneling driver
[    8.107371] NET: Registered protocol family 17
[    8.112288] NET: Registered protocol family 15
[    8.117228] bridge: automatic filtering via arp/ip/ip6tables has been deprecated. Update your scripts to load br_netfilter if you need this.
[    8.131182] Bridge firewalling registered
[    8.135626] Ebtables v2.0 registered
[    8.139659] can: controller area network core (rev 20120528 abi 9)
[    8.146521] NET: Registered protocol family 29
[    8.151431] can: raw protocol (rev 20120528)
[    8.156156] can: broadcast manager protocol (rev 20120528 t)
[    8.162409] can: netlink gateway (rev 20130117) max_hops=1
[    8.168606] Bluetooth: RFCOMM TTY layer initialized
[    8.174007] Bluetooth: RFCOMM socket layer initialized
[    8.179699] Bluetooth: RFCOMM ver 1.11
[    8.183869] Bluetooth: HIDP (Human Interface Emulation) ver 1.2
[    8.190409] Bluetooth: HIDP socket layer initialized
[    8.195929] 8021q: 802.1Q VLAN Support v1.8
[    8.200559] lib80211: common routines for IEEE802.11 drivers
[    8.206845] [WLAN_RFKILL]: Enter rfkill_wlan_init
[    8.212182] [WLAN_RFKILL]: Enter rfkill_wlan_probe
[    8.217498] [WLAN_RFKILL]: wlan_platdata_parse_dt: wifi_chip_type = ap6212
[    8.225098] [WLAN_RFKILL]: wlan_platdata_parse_dt: enable wifi power control.
[    8.232983] [WLAN_RFKILL]: wlan_platdata_parse_dt: wifi power controled by gpio.
[    8.241188] [WLAN_RFKILL]: wlan_platdata_parse_dt: get property: WIFI,host_wake_irq = 150, flags = 0.
[    8.251379] [WLAN_RFKILL]: rfkill_wlan_probe: init gpio
[    8.257153] [WLAN_RFKILL]: Exit rfkill_wlan_probe
[    8.262388] [BT_RFKILL]: Enter rfkill_rk_init
[    8.267429] [BT_RFKILL]: bluetooth_platdata_parse_dt: get property: uart_rts_gpios = 139.
[    8.276494] [BT_RFKILL]: bluetooth_platdata_parse_dt: get property: BT,reset_gpio = 149.
[    8.285449] [BT_RFKILL]: bluetooth_platdata_parse_dt: get property: BT,wake_gpio = 146.
[    8.294307] [BT_RFKILL]: bluetooth_platdata_parse_dt: get property: BT,wake_host_irq = 151.
[    8.303556] [BT_RFKILL]: bluetooth_platdata_parse_dt: clk_get failed!!!.
[    8.310981] [BT_RFKILL]: Request irq for bt wakeup host
[    8.316783] [BT_RFKILL]: BT_WAKE_HOST IRQ fired
[    8.316796] [BT_RFKILL]: ** disable irq
[    8.326028] [BT_RFKILL]: bt_default device registered.
[    8.331754] Key type dns_resolver registered
[    8.336739] cif_isp10_v4l2_drv_probe: probing...
[    8.341907] cif_isp10_pltfrm_dev_init(1224) ERR: could not get default pinstate
[    8.349994] cif_isp10_pltfrm_dev_init WARN: could not get pins_sleep pinstate
[    8.357880] cif_isp10_pltfrm_dev_init WARN: could not get pins_inactive pinstate
[    8.391918] ov7750.ov_camera_module_write_config(182) ERR: no active sensor configuration
[    8.400765] ov7750.ov_camera_module_write_config(233) ERR: failed with error -14
[    8.409277] ov7750.pltfrm_camera_module_read_reg(996) ERR: i2c read from offset 0x0000300a failed with error -6
[    8.420573] ov7750.pltfrm_camera_module_read_reg(996) ERR: i2c read from offset 0x0000300b failed with error -6
[    8.431732] ov7750.ov7750_check_camera_id(571) ERR: register read failed, camera module powered off?
[    8.441829] ov7750.ov7750_check_camera_id(589) ERR: failed with error (-6)
[    8.449447] ov7750.ov_camera_module_attach(256) ERR: failed with error -6
[    8.457004] cif_isp10_img_src_v4l2_i2c_subdev_to_img_src(59) ERR: failed with error -6
[    8.465756] cif_isp10_img_src_to_img_src(70) ERR: to_img_src failed!
[    8.472771] cif_isp10_img_src_to_img_src(78) ERR: failed with error -14
[    8.505802] imx219.pltfrm_camera_module_read_reg(996) ERR: i2c read from offset 0x00000000 failed with error -6
[    8.517094] imx219.pltfrm_camera_module_read_reg(996) ERR: i2c read from offset 0x00000001 failed with error -6
[    8.528249] imx219.imx219_check_camera_id(759) ERR: register read failed, camera module powered off?
[    8.538343] imx219.imx219_check_camera_id(776) ERR: failed with error (-6)
[    8.545962] imx219.imx_camera_module_attach(236) ERR: failed with error -6
[    8.553605] cif_isp10_img_src_v4l2_i2c_subdev_to_img_src(59) ERR: failed with error -6
[    8.562349] cif_isp10_img_src_to_img_src(70) ERR: to_img_src failed!
[    8.569375] cif_isp10_img_src_to_img_src(78) ERR: failed with error -14
[    8.576678] cif_isp10_img_srcs_init(1099) ERR: failed with error -14
[    8.583703] cif_isp10_create(5772) ERR: cif_isp10_img_srcs_init failed
[    8.590910] cif_isp10_create(5808) ERR: failed with error -14
[    8.597564] ThumbEE CPU extension supported.
[    8.602284] Registering SWP/SWPB emulation handler
[    8.607934] registered taskstats version 1
[    8.612462] Loading compiled-in X.509 certificates
[    8.618325] W : [File] : drivers/gpu/arm/midgard_for_linux/platform/rk/mali_kbase_config_rk.c; [Line] : 107; [Func] : kbase_platform_rk_init(); power-off-delay-ms not available.
[    8.636005] mali ffa30000.gpu: GPU identified as 0x0750 r0p0 status 1
[    8.643469] I : [File] : drivers/gpu/arm/midgard_for_linux/backend/gpu/mali_kbase_devfreq.c; [Line] : 371; [Func] : kbase_devfreq_init(); success initing power_model_simple.
[    8.660806] mali ffa30000.gpu: Probed as mali0
[    8.666277] devfreq ffa30000.gpu: Couldn't update frequency transition information.
[    8.666300] rk_gmac-dwmac ff290000.ethernet: clock input or output? (input).
[    8.666303] rk_gmac-dwmac ff290000.ethernet: TX delay(0x30).
[    8.666307] rk_gmac-dwmac ff290000.ethernet: RX delay(0x10).
[    8.666317] rk_gmac-dwmac ff290000.ethernet: integrated PHY? (no).
[    8.666378] rk_gmac-dwmac ff290000.ethernet: clock input from PHY
[    8.671386] rk_gmac-dwmac ff290000.ethernet: init for RGMII
[    8.671436] stmmac - user ID: 0x10, Synopsys ID: 0x35
[    8.671437]  Ring mode enabled
[    8.671440]  DMA HW capability register supported
[    8.671441]  Normal descriptors
[    8.671442]  RX Checksum Offload Engine supported (type 2)
[    8.671443]  TX Checksum insertion supported
[    8.671443]  Wake-Up On Lan supported
[    8.671466]  Enable RX Mitigation via HW Watchdog Timer
[    9.698729] libphy: stmmac: probed
[    9.702562] eth%d: PHY ID 001cc915 at 0 IRQ POLL (stmmac-0:00) active
[    9.709843] eth%d: PHY ID 001cc915 at 1 IRQ POLL (stmmac-0:01)
[    9.722865] rockchip-thermal ff280000.tsadc: Missing rockchip,grf property
[    9.735376] dwmmc_rockchip ff0d0000.dwmmc: IDMAC supports 32-bit address mode.
[    9.743966] dwmmc_rockchip ff0d0000.dwmmc: Using internal DMA controller.
[    9.751475] dwmmc_rockchip ff0d0000.dwmmc: Version ID is 270a
[    9.757862] dwmmc_rockchip ff0d0000.dwmmc: DW MMC controller at irq 30,32 bit host data width,256 deep fifo
[    9.768648] dwmmc_rockchip ff0d0000.dwmmc: 'clock-freq-min-max' property was deprecated.
[    9.777621] dwmmc_rockchip ff0d0000.dwmmc: No vmmc regulator found
[    9.784456] dwmmc_rockchip ff0d0000.dwmmc: No vqmmc regulator found
[    9.791614] dwmmc_rockchip ff0d0000.dwmmc: allocated mmc-pwrseq
[    9.812972] mmc_host mmc2: Bus speed (slot 0) = 400000Hz (slot req 400000Hz, actual 400000HZ div = 0)
[    9.842983] dwmmc_rockchip ff0d0000.dwmmc: 1 slots initialized
[    9.849881] asoc-simple-card sound-simple-card: audio jack plug out
[    9.857973] asoc-simple-card sound-simple-card: call_usermodehelper fail, ret=-2, status=0
[    9.869133] asoc-simple-card sound-simple-card: ASoC: DAPM unknown pin Headphones
[    9.877973] asoc-simple-card sound-simple-card: i2s-hifi <-> ff890000.i2s mapping ok
[    9.886976] input: rockchip,miniarm-codec Headphones as /devices/platform/sound-simple-card/sound/card1/input2
[    9.898665] input: gpio-keys as /devices/platform/gpio-keys/input/input3
[    9.906603] mmc_host mmc2: Bus speed (slot 0) = 49500000Hz (slot req 50000000Hz, actual 49500000HZ div = 0)
[    9.917374] rk808-rtc rk808-rtc: setting system clock to 2013-01-18 00:45:53 UTC (1358469953)
[    9.918115] mmc2: new high speed SDIO card at address 0001
[    9.933919] device-tree: Duplicate name in testcase-data, renamed to "duplicate-name#1"
[    9.944556] ### dt-test ### start of unittest - you will see error messages
[    9.952817] /testcase-data/phandle-tests/consumer-a: could not get #phandle-cells-missing for /testcase-data/phandle-tests/provider1
[    9.966002] /testcase-data/phandle-tests/consumer-a: could not get #phandle-cells-missing for /testcase-data/phandle-tests/provider1
[    9.979193] /testcase-data/phandle-tests/consumer-a: could not find phandle
[    9.986915] /testcase-data/phandle-tests/consumer-a: could not find phandle
[    9.994622] /testcase-data/phandle-tests/consumer-a: arguments longer than property
[   10.003094] /testcase-data/phandle-tests/consumer-a: arguments longer than property
[   10.012240] irq: no irq domain found for /testcase-data/interrupts/intc0 !
[   10.022940] asoc-simple-card sound-simple-card: audio jack plug out
[   10.024552] overlay_is_topmost: #5 clashes #6 @/testcase-data/overlay-node/test-bus/test-unittest8
[   10.024554] overlay_removal_is_ok: overlay #5 is not topmost
[   10.024555] of_overlay_destroy: removal check failed for overlay #5
[   10.045460] ### dt-test ### end of unittest - 148 passed, 0 failed
[   10.052237] vcc_sd: disabling
[   10.052241] vcc_flash: disabling
[   10.052244] vdd_logic: disabling
[   10.052600] ALSA device list:
[   10.052604]   #0: Generic USB Audio OnBoard at usb-ff500000.usb-1, high speed
[   10.052605]   #1: rockchip,miniarm-codec
[   10.085823] asoc-simple-card sound-simple-card: call_usermodehelper fail, ret=-2, status=0
[A[   10.095178] ttyS1 - failed to request DMA
[   10.099662] md: Waiting for all devices to be available before autodetect
[   10.107180] md: If you don't use raid, use raid=noautodetect
[   10.113968] md: Autodetecting RAID arrays.
[   10.118491] md: Scanned 0 and added 0 devices.
[   10.123410] md: autorun ...
[   10.126489] md: ... autorun DONE.
[   11.352234] EXT4-fs (mmcblk0p2): mounted filesystem with ordered data mode. Opts: (null)
[   11.361212] VFS: Mounted root (ext4 filesystem) on device 179:2.
[   11.370368] devtmpfs: mounted
[   11.374301] Freeing unused kernel memory: 1024K
[   11.392935] vendor storage:20160801 ret = -1
[   11.614087] systemd[1]: System time before build time, advancing clock.
[   11.630759] systemd[1]: Failed to insert module 'autofs4': No such file or directory
[   11.658128] random: systemd: uninitialized urandom read (16 bytes read, 80 bits of entropy available)
[   11.673272] systemd[1]: systemd 232 running in system mode. (+PAM +AUDIT +SELINUX +IMA +APPARMOR +SMACK +SYSVINIT +UTMP +LIBCRYPTSETUP +GCRYPT +GNUTLS +ACL +XZ +LZ4 +SECCOMP +BLKID +ELFUTILS +KMOD +IDN)
[   11.693748] systemd[1]: Detected architecture arm.
 
Welcome to Debian GNU/Linux 9 (stretch)!
 
[   11.714599] systemd[1]: Set hostname to <debian>.
[   11.735112] random: systemd: uninitialized urandom read (16 bytes read, 81 bits of entropy available)
[   11.759248] random: systemd-sysv-ge: uninitialized urandom read (16 bytes read, 81 bits of entropy available)
[   11.766682] random: systemd-cryptse: uninitialized urandom read (16 bytes read, 81 bits of entropy available)
[   11.792962] random: systemd-gpt-aut: uninitialized urandom read (16 bytes read, 82 bits of entropy available)
[   11.804229] random: systemd-gpt-aut: uninitialized urandom read (16 bytes read, 82 bits of entropy available)
[   11.859409] random: systemd: uninitialized urandom read (16 bytes read, 84 bits of entropy available)
[   11.869741] random: systemd: uninitialized urandom read (16 bytes read, 84 bits of entropy available)
[   11.880072] random: systemd: uninitialized urandom read (16 bytes read, 84 bits of entropy available)
[   11.890527] random: systemd: uninitialized urandom read (16 bytes read, 84 bits of entropy available)
[   12.040275] systemd[1]: Listening on Journal Socket (/dev/log).
[  OK  ] Listening on Journal Socket (/dev/log).
[   12.063132] systemd[1]: Reached target Swap.
[  OK  ] Reached target Swap.
[   12.083245] systemd[1]: Listening on udev Control Socket.
[  OK  ] Listening on udev Control Socket.
[   12.103193] systemd[1]: Listening on /dev/initctl Compatibility Named Pipe.
[  OK  ] Listening on /dev/initctl Compatibility Named Pipe.
[   12.123208] systemd[1]: Started Forward Password Requests to Wall Directory Watch.
[  OK  ] Started Forward Password Requests to Wall Directory Watch.
[   12.153266] systemd[1]: Listening on Syslog Socket.
[  OK  ] Listening on Syslog Socket.
[   12.173161] systemd[1]: Reached target Remote File Systems.
[  OK  ] Reached target Remote File Systems.
[   12.193254] systemd[1]: Listening on udev Kernel Socket.
[  OK  ] Listening on udev Kernel Socket.
[   12.213753] systemd[1]: Created slice System Slice.
[  OK  ] Created slice System Slice.
[   12.235567] systemd[1]: Mounting Debug File System...
         Mounting Debug File System...
[   12.253899] systemd[1]: Created slice system-serial\x2dgetty.slice.
[  OK  ] Created slice system-serial\x2dgetty.slice.
[   12.285116] systemd[1]: Mounting POSIX Message Queue File System...
         Mounting POSIX Message Queue File System...
[   12.313432] systemd[1]: Started Dispatch Password Requests to Console Directory Watch.
[  OK  ] Started Dispatch Password Requests to Console Directory Watch.
[   12.343143] systemd[1]: Reached target Encrypted Volumes.
[  OK  ] Reached target Encrypted Volumes.
[   12.363109] systemd[1]: Reached target Paths.
[  OK  ] Reached target Paths.
[   12.383651] systemd[1]: Listening on Journal Socket.
[  OK  ] Listening on Journal Socket.
[   12.407988] systemd[1]: Starting Load Kernel Modules...
         Starting Load Kernel Modules...
[   12.435810] systemd[1]: Created slice system-getty.slice.
[  OK  ] Created slice system-getty.slice.
[   12.470817] systemd[1]: Reached target Sockets.
[  OK  ] Reached target Sockets.
[   12.495509] systemd[1]: Starting Journal Service...
         Starting Journal Service...
[   12.515478] systemd[1]: Starting Remount Root and Kernel File Systems...
         Starting Remount Root and Kernel File Systems...
[   12.545565] systemd[1]: Starting Create Static Device Nodes in /dev...
         Starting Create Static Device Nodes in /dev...
[   12.573659] systemd[1]: Reached target Slices.
[  OK  ] Reached target Slices.
[   12.598512] systemd[1]: Mounted POSIX Message Queue File System.
[  OK  ] Mounted POSIX Message Queue File System.
[   12.623370] systemd[1]: Mounted Debug File System.
[  OK  ] Mounted Debug File System.
[   12.643919] systemd[1]: Started Load Kernel Modules.
[  OK  ] Started Load Kernel Modules.
[   12.665014] systemd[1]: Started Remount Root and Kernel File Systems.
[  OK  ] Started Remount Root and Kernel File Systems.
[   12.693562] systemd[1]: Started Create Static Device Nodes in /dev.
[  OK  ] Started Create Static Device Nodes in /dev.
[   12.713301] systemd[1]: Started Journal Service.
[  OK  ] Started Journal Service.
         Starting udev Kernel Device Manager...
         Starting Load/Save Random Seed...
[  OK  ] Reached target Local File Systems (Pre).
[  OK  ] Reached target Local File Systems.
         Starting udev Coldplug all Devices...
         Starting Flush Journal to Persistent Storage...
         Starting Apply Kernel Variables...
         Mounting Configuration File System...
         Mounting FUSE Control File System...
[  OK  ] Mounted Configuration File System.
[  OK  ] Mounted FUSE Control File System.
[  OK  ] Started udev Kernel Device Manager.
[  OK  ] Started Load/Save Random Seed.
[  OK  ] Started Apply Kernel Variables.
[   13.022793] systemd-journald[157]: Received request to flush runtime journal from PID 1
         Starting Raise network interfaces...
[  OK  ] Started Flush Journal to Persistent Storage.
         Starting Create Volatile Files and Directories...
[  OK  ] Started Create Volatile Files and Directories.
[  OK  ] Started udev Coldplug all Devices.
[  OK  ] Found device /dev/ttyS1.
         Starting Update UTMP about System Boot/Shutdown...
         Starting Network Time Synchronization...
[  OK  ] Started Update UTMP about System Boot/Shutdown.
[  OK  ] Reached target Sound Card.
[  OK  ] Listening on Load/Save RF Kill Switch Status /dev/rfkill Watch.
[  OK  ] Started Raise network interfaces.
[  OK  ] Reached target Network.
[  OK  ] Started Network Time Synchronization.
[  OK  ] Reached target System Initialization.
[  OK  ] Reached target Basic System.
         Starting Permit User Sessions...
[  OK  ] Started Regular background program processing daemon.
[  OK  ] Started Daily Cleanup of Temporary Directories.
         Starting getty on tty2-tty6 if dbus and logind are not available...
         Starting System Logging Service...
[  OK  ] Reached target System Time Synchronized.
[  OK  ] Started Daily apt download activities.
[  OK  ] Started Daily apt upgrade and clean activities.
[  OK  ] Reached target Timers.
[  OK  ] Started System Logging Service.
[  OK  ] Started Permit User Sessions.
[  OK  ] Started Serial Getty on ttyS1.
[  OK  ] Started Getty on tty1.
[  OK  ] Started Getty on tty2.
[  OK  ] Started Getty on tty3.
[  OK  ] Started Getty on tty4.
[  OK  ] Started Getty on tty5.
[  OK  ] Started Getty on tty6.
[  OK  ] Started getty on tty2-tty6 if dbus and logind are not available.
[  OK  ] Reached target Login Prompts.
[  OK  ] Reached target Multi-User System.
[  OK  ] Reached target Graphical Interface.
         Starting Update UTMP about System Runlevel Changes...
[  OK  ] Started Update UTMP about System Runlevel Changes.
 
Debian GNU/Linux 9 debian ttyS1
 
debian login: [   16.397794] random: nonblocking pool is initialized
root
Password:
Last login: Thu Nov  3 17:17:11 UTC 2016 on ttyS1
Linux debian 4.4.103+ #1 SMP Sat Mar 17 13:08:35 GMT 2018 armv7l
 
The programs included with the Debian GNU/Linux system are free software;
the exact distribution terms for each program are described in the
individual files in /usr/share/doc/*/copyright.
 
Debian GNU/Linux comes with ABSOLUTELY NO WARRANTY, to the extent
permitted by applicable law.
(stretch)root@debian:~#
```