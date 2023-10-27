---
title: "Hard Drive Recovery - uas error"
date: "2018-06-08"
categories: 
  - "miscellaneous"
tags: 
  - "disk"
  - "external"
  - "linux"
  - "recovery"
  - "ubuntu"
---

I was asked to see if i could recover the contents of a 2TB external hard drive as it was just coming up as required to be formatted in Windows. The external hard drive was actually an internal hard drive with a fancy case and external power supply and sold as external. I took it out of the case as the connections were ruined and used a [USB adapter](https://www.amazon.co.uk/gp/product/B01GIZSC9G) to try and connect the drive to my machine.

The hard drive spin up, but on my Windows 10 machine I couldnt view the contents, it just kept prompting me to format the drive which isnt helpful.

I thought I would connect the hard drive to my Ubuntu virtual machine as linux & Ubuntu have tonnes of tools to debug what was going on. The first thing i tried was to just check fdisk and mount the drive to see the contents, however fdisk didnt show an external drive.

I used lsusb to check whether or not the drive was recognised by my virtual machine:

```
lsusb -v
Bus 004 Device 004: ID 2109:0711 VIA Labs, Inc. 
Couldn't open device, some information will be missing
Device Descriptor:
  bLength                18
  bDescriptorType         1
  bcdUSB               3.10
  bDeviceClass            0 (Defined at Interface level)
  bDeviceSubClass         0 
  bDeviceProtocol         0 
  bMaxPacketSize0         9
  idVendor           0x2109 VIA Labs, Inc.
  idProduct          0x0711 
  bcdDevice            3.36
  iManufacturer           1 
  iProduct                2 
  iSerial                 3 
  bNumConfigurations      1
  Configuration Descriptor:
    bLength                 9
    bDescriptorType         2
    wTotalLength          121
    bNumInterfaces          1
    bConfigurationValue     1
    iConfiguration          0 
    bmAttributes         0xc0
      Self Powered
    MaxPower                2mA
    Interface Descriptor:
      bLength                 9
      bDescriptorType         4
      bInterfaceNumber        0
      bAlternateSetting       0
      bNumEndpoints           2
      bInterfaceClass         8 Mass Storage
      bInterfaceSubClass      6 SCSI
      bInterfaceProtocol     80 Bulk-Only
      iInterface              0 
      Endpoint Descriptor:
        bLength                 7
        bDescriptorType         5
        bEndpointAddress     0x81  EP 1 IN
        bmAttributes            2
          Transfer Type            Bulk
          Synch Type               None
          Usage Type               Data
        wMaxPacketSize     0x0400  1x 1024 bytes
        bInterval               0
        bMaxBurst              15
      Endpoint Descriptor:
        bLength                 7
        bDescriptorType         5
        bEndpointAddress     0x02  EP 2 OUT
        bmAttributes            2
          Transfer Type            Bulk
          Synch Type               None
          Usage Type               Data
        wMaxPacketSize     0x0400  1x 1024 bytes
        bInterval               0
        bMaxBurst              15
    Interface Descriptor:
      bLength                 9
      bDescriptorType         4
      bInterfaceNumber        0
      bAlternateSetting       1
      bNumEndpoints           4
      bInterfaceClass         8 Mass Storage
      bInterfaceSubClass      6 SCSI
      bInterfaceProtocol     98 
      iInterface              0 
      Endpoint Descriptor:
        bLength                 7
        bDescriptorType         5
        bEndpointAddress     0x04  EP 4 OUT
        bmAttributes            2
          Transfer Type            Bulk
          Synch Type               None
          Usage Type               Data
        wMaxPacketSize     0x0400  1x 1024 bytes
        bInterval               0
        bMaxBurst               0
        Command pipe (0x01)
      Endpoint Descriptor:
        bLength                 7
        bDescriptorType         5
        bEndpointAddress     0x85  EP 5 IN
        bmAttributes            2
          Transfer Type            Bulk
          Synch Type               None
          Usage Type               Data
        wMaxPacketSize     0x0400  1x 1024 bytes
        bInterval               0
        bMaxBurst              15
        MaxStreams             32
        Data-in pipe (0x03)
      Endpoint Descriptor:
        bLength                 7
        bDescriptorType         5
        bEndpointAddress     0x06  EP 6 OUT
        bmAttributes            2
          Transfer Type            Bulk
          Synch Type               None
          Usage Type               Data
        wMaxPacketSize     0x0400  1x 1024 bytes
        bInterval               0
        bMaxBurst              15
        MaxStreams             32
        Data-out pipe (0x04)
      Endpoint Descriptor:
        bLength                 7
        bDescriptorType         5
        bEndpointAddress     0x87  EP 7 IN
        bmAttributes            2
          Transfer Type            Bulk
          Synch Type               None
          Usage Type               Data
        wMaxPacketSize     0x0400  1x 1024 bytes
        bInterval               0
        bMaxBurst               0
        MaxStreams             32
        Status pipe (0x02)
```
This was the device i was confident was the external drive as the others were just standard stuff.

I had a look at the usb-devices to see if there was any hints:

```
usb-devices
 
....
T:  Bus=04 Lev=01 Prnt=01 Port=00 Cnt=01 Dev#=  4 Spd=5000 MxCh= 0
D:  Ver= 3.10 Cls=00(>ifc ) Sub=00 Prot=00 MxPS= 9 #Cfgs=  1
P:  Vendor=2109 ProdID=0711 Rev=03.36
S:  Manufacturer=Ugreen
S:  Product=20231
S:  SerialNumber=00000012DE2A
C:  #Ifs= 1 Cfg#= 1 Atr=c0 MxPwr=8mA
I:  If#= 0 Alt= 1 #EPs= 4 Cls=08(stor.) Sub=06 Prot=62 Driver=uas
```

I decided to check DMESG to see if i could see more what was going on with the kernel. Unfortunately I didnt save the actual DMESG log, but it looked similar to this (ignore the timestamps):

```
[11701.253956] sd 3:0:0:0: uas_eh_device_reset_handler
[11704.252182] scsi host3: uas_eh_task_mgmt: LOGICAL UNIT RESET timed out
[11704.252196] scsi host3: uas_eh_bus_reset_handler start
[11704.252443] usb 4-1: stat urb: killed, stream 0
[11704.358882] usb 4-1: reset high-speed USB device number 3 using ehci-pci
[11704.483206] scsi host3: uas_eh_bus_reset_handler success
 
[ 3215.684900] usb 4-1.2: URB BAD STATUS -71
[ 3215.752727] usb 4-1.2: reset high speed USB device using ehci_hcd and address 5
[ 3215.845484] scsi 7:0:0:0: Device offlined - not ready after error recovery
```

This pointed me to see that <uas> was resetting the device. So i disabled uas for this specific device. Take the Vendor ID and Product ID from the lsusb output and create a file at the kernel module called disable-uas.conf. The filename dosent really matter. Add in the values for the vendor and product id in the quirks argument. This will disable uas for this specific device. You then need to update the ramfs and reboot the virtual machine.

```
sudo nano /etc/modprobe.d/disable-uas.conf
...
options usb-storage quirks=0x2109:0x0711:u
options usbcore autosuspend=-1
```

Update ramfs

`update-initramfs -u`

After reboot, i could finally see the device under fdisk - hooray!

I noticed that the filesystem type wasnt FAT as i hoped, but was instead HPFS/NTFS/exFAT so i still couldnt mount. There were a couple of things i tried but bottled committing because i didnt want to overwrite the contents.

I found a great tool called testdisk which i used to scan, recovery and fix the drive and assign the partition as primary without any risk to the data. testdisk asks for the partition type but does a good job at guessing it.

![](/images/testdisk.png)

After i did that, i was able to mount the drive and view the contents! Hooray!
