---
title: "Microsoft Azure Nested Virtualization: Creating on-premise Windows Domain environments"
date: "2025-08-04"
categories: 
  - "cloud"
  - "security"
  - "azure"
tags: 
  - "azure"
  - "compute"
  - "virtualization"
  - "simulation"
  - "labs"
  - "security"
---

# Microsoft Azure Nested Virtualization: Creating on-premise Windows Domain environments
## Summary
Modern Microsoft Azure hypervisors support nested virtualization; meaning you can run virtual machines within virtual machines. With a bit of magic, you can create an effective "Virtual ESXi" server and run a few virtual machines from ISO files, which is much closer to an on-premise environment.

However, instead of ISOs, if I have a few existing Virtual Hard Disks (VHDs) that are configured to be domain-joined windows VMs, I can attach them as additional disks to a Virtual Machine running Hyper-V. Hyper-V is then configured with a few VMs, using an attached disk as their image source.

![](../images/nested-vms/nested-vms-architecture.png)

The end result is that I can have an effective "Virtual ESXi" server, running an on-premise Windows domain environment; all virtualized and deployed in a few minutes. Check out this video demonstration [here](https://youtu.be/D6Jf8D_AHtg).

Supporting IaC, including packer image definitions, can be found [here](https://github.com/akingscote/msft-azure-nested-virtualization-windows-domain).

# Introduction
Nested virtualization [isn't a new capability for Microsoft (~2017)](https://azure.microsoft.com/en-us/blog/nested-virtualization-in-azure/), and as far as I'm aware, [AWS](https://repost.aws/questions/QUXqOvN3_dR-Gsv5CJC6rBpQ/what-aws-instances-can-support-nested-virtualisation) dosen't really support it, but [Google Cloud does](https://cloud.google.com/compute/docs/instances/nested-virtualization/overview).

I've got a fair bit of experience building on-premise Windows domains (mainly Public Key Infrastructure), and trying to create immutable images of Windows domains is super frustrating. You've often got to wait for your Windows server Domain Controller (DC) to be up and running, and then try to coordinate numerous other machines to configure themselves and join the domain, which may require numerous restarts. If you want to configure the machines with ansible, you'll need SSH installed on the machines, which isn't (wasn't) default. It just gets really messy, complex, and slow.

However, with this approach, once my domain is configured, I can effectively "snapshot" the entire domain and replay it using this nested-virtualization technique, rather than configuring anything at run time. This speeds up deployments and lends itself really nicely for lab environments.

> FWIW, Beginning with Windows 10 build 1809 and Windows Server 2019, OpenSSH is available as a [feature on demand](https://learn.microsoft.com/en-us/windows-server/administration/openssh/openssh-overview). For Windows Server 2025, it's [installed but not enabled](https://learn.microsoft.com/en-us/windows-server/administration/openssh/openssh_install_firstuse?tabs=powershell&pivots=windows-server-2025).
{: .prompt-info }

**Why not just run directly in Azure?**
Of course, you can just deploy a Windows server machine directly into Azure as a virtual machine, BUT this technique allows you to simulate much closer to an on-premise environment.

I guess the main advantage is that you have an entirely sandboxed network (depending on your configuration). Your IP addressing scheme can be exactly as it is on-premise. You don't need to worry about the Azure networking fabric getting in the way, such as the first few IP addresses being reserved in each subnet, or ARP issues, or NSG rules, or flow logs....

Also, although the Windows kernels are [supposedly pretty much identical between on-premise and the cloud](https://learn.microsoft.com/en-us/windows-server/get-started/azure-edition#key-differences), the Virtual machines are subtly different. For example, many of the available OS images are running the [Azure Windows VM Agent](https://learn.microsoft.com/en-us/azure/virtual-machines/extensions/agent-windows), meaning you can run VM extensions and get provisioning updates.

> A note on **Nested Virtualization Networking**</br>With Azure nested-virtualization, you can configure reverse NAT on the host, which provides direct access to ports from VMs within Hyper-V.<br>For example, if you have a Windows Server running within Hyper-V, which is running IIS web server, you can expose that port externally. For my scenario I haven't done that, as I want a completely isolated network within the Hyper-V VM. If you want to learn more about that, check out [this blog](https://techcommunity.microsoft.com/blog/itopstalkblog/how-to-setup-nested-virtualization-for-azure-vmvhd/1115338).
{: .prompt-info }


# Technical Details
Let's assume that you have some Virtual Hard Disks (VHDs) and you want to deploy them in the above scenario.

In order to use nested virtualization in Microsoft Azure, you'll first need to use a SKU that supports it.

Essentially, you need to find a SKU that supports [hyper-threading](https://techcommunity.microsoft.com/blog/azure-ai-services-blog/nested-virtualization-on-azure--a-step-by-step-guide/4368074), which isn't particularly obvious. I haven't found the capability exported by any APIs, and have had to go through the [documentation looking for hyperthreading in the descriptions](https://learn.microsoft.com/en-us/azure/virtual-machines/sizes/general-purpose/dsv3-series?tabs=sizebasic).

Fortunately, it seems that the newest generations of SKUs (E-series) support hyper-threading 👍
![](../images/nested-vms/hyperthreading.png)

Once you've selected a virtual machine SKU that supports nested virtualization, nested virtualization will not [automatically be running on VM launch](https://learn.microsoft.com/en-us/troubleshoot/azure/virtual-machines/windows/troubleshoot-vm-by-use-nested-virtualization).

It's a pretty easy fix, just run the following from an administrative powershell prompt:
```
bcdedit /set hypervisorlaunchtype auto
```
and then restart the PC.

> There are some more sophisticated [official startup scripts](https://github.com/MicrosoftDocs/Virtualization-Documentation/blob/main/hyperv-tools/Nested/Enable-NestedVm.ps1), but they haven't been updated in 10 years 👀

You'll then need to install the Hyper-V feature, which you can do via the GUI, or via powershell with:
```
Install-WindowsFeature -Name DHCP,Hyper-V -IncludeManagementTools
```

On the VM, it's also worth setting Hyper-V enhanced session mode, which allows you copy-paste into VMs, and resize VM windows.
```
Set-VMHost -EnableEnhancedSessionMode $True
```

You now have a machine capable of nested virtualization. I've put all this code, including a sysprep step, into a packer image which you can find [here](https://github.com/akingscote/msft-azure-nested-virtualization-windows-domain/blob/main/hyperVWindowsServer2025/hyperVWindowsServer2025.pkr.hcl).

> **A note on 6th generation images**<br>[6th generation Microsoft Azure images are backed by NVMe](https://learn.microsoft.com/en-us/azure/virtual-machines/nvme-overview#scsi-to-nvme), as opposed to SCSI, which makes them very fast. Unfortunately, [I found a bug with Packer](https://github.com/hashicorp/packer-plugin-azure/issues/521), which wouldn't let me deploy an image built for 6th generation images (I'm not the only one who's [had issues](https://discuss.hashicorp.com/t/creating-azure-arm-image-for-latest-v6-generation-of-x86-vms/73778).
{: .prompt-warning }

> **A note on VM disks**<br>If creating a new virtual machine in Hyper-V, it will create a `.vhdx` disk storage format. This is essentially a dynamic disk size, as opposed to `.vhd`, which is a static size. Obviously `.vhdx` is better, as it *may* use less space. However, in order to create Microsoft Azure data disks from Storage Account Objects, the virtual disk object must be in `.vhd` format, and not `.vhdx`. Fortunately, you can easily convert the disk in Powershell, it just takes a while and uses up a lot of space:`Convert-VHD -Path "C:\ProgramData\Microsoft\Windows\Virtual Hard Disks\WindowsServerMember.vhdx" -DestinationPath "D:\WindowsServerMember.vhd" -VHDType Fixed`
{: .prompt-info }

Once a VHD disk has been attached to the VM, in order to use it in Hyper-V, the disk needs to be in an "offline" state from the OS's perspective. You can easily do this in powershell via the following:
```
Set-Disk -Number 2 -IsOffline $true
```
The first disk (starting at 0) is normally the OS, and then the second disk a temporary RAM disk.
![](../images/nested-vms/disks-offline.png)

Once the disks are offline, in Hyper-V you can create your VM configuration, selecting a source disk as the VM source.
![](../images/nested-vms/vm-config-specify-disk.png)

In order to boot the VM, you'll likely need to disable Secure Boot:
![](../images/nested-vms/disable-secure-boot.png)

Otherwise, you may find the following error:
![](../images/nested-vms/scsidvd-error.png)

> 6th generation NVMe images likely have some considerable performance benefits, but because of the Packer bug, I was unable to use it in this proof of concept. However, looking at the Hyper-V configuration and the secure boot error, I'm not sure if I could use NVMe, as Hyper-V is expecting an SCSI-controller. The "older" generations of VMs are all using SCSI instead of NVMe.
{: .prompt-info }

Of course, all this can be automated in Powershell:
```
Set-Disk -Number 2 -IsOffline $true
New-VM -Name "WindowsServerDC" -MemoryStartupBytes 4GB -Generation 2 -NoVHD -SwitchName "Internal"
Add-VMHardDiskDrive -VMName "WindowsServerDC" -DiskNumber 2 -ControllerType SCSI -ControllerNumber 0 -ControllerLocation 1
Set-VMFirmware -VMName "WindowsServerDC" -EnableSecureBoot Off -FirstBootDevice $(Get-VMHardDiskDrive -VMName "WindowsServerDC")
Set-VM -Name "WindowsServerDC" -AutomaticStartAction Start -AutomaticStartDelay 30
Start-VM -Name "WindowsServerDC"
```

## VM Networking
There is only one piece of smoke and mirrors with this proof of concept, which is the networking.

As we are just starting a `.vhd` file, all the VM's configuration is already set. If the VM is already joined to a domain, then when starting the VM, it'll still be joined. However, just because it's already joined the domain, dosent mean that the domain is reachable on this fresh deployment. 

With Hyper-V, the network interfaces within the VM are effectively ephemeral, and are controlled by Hyper-V. This means that when a VM is booted from disk, it dosent have any IP address configuration. That means that any domain is then unreachable, unless there is some startup scripts configured within the OS to reset the IP address on boot. Windows on-premise domains usually use statically assigned IP addresses, which means we will need to reconfigure these IP addresses on first boot.

Additionally, depending on your requirements, you'll likely need to create a Hyper-V networking switch, and associate your VMs with that "network".
```
New-VMSwitch -Name "Internal" -SwitchType Internal
```
For my PoC, I've saved the following script (or variants of it) on the Windows VMs:
```
Start-Transcript -Path "C:\Logs\Set-StaticIP-$(Get-Date -Format 'yyyyMMdd-HHmmss').log"

try {
$IPAddress = "192.168.1.14"
$SubnetMask = "255.255.255.0"
$DNS1 = "192.168.1.1"

# Get the first active Ethernet interface that's not virtual or disconnected
$adapter = Get-NetAdapter | Where-Object {
    $_.Status -eq 'Up' -and 
    $_.InterfaceType -notlike '*Loopback*' -and 
    $_.Virtual -eq $false
} | Select-Object -First 1

if ($null -eq $adapter) {
    Write-Host "No active network adapters found. Please check your network connections."
    exit 1
}

Write-Host "Configuring network adapter '$($adapter.Name)' with static IP $IPAddress"

# Remove any existing IP addresses (DHCP or static)
Remove-NetIPAddress -InterfaceIndex $adapter.InterfaceIndex -Confirm:$false -ErrorAction SilentlyContinue

# Remove any existing routes
Remove-NetRoute -InterfaceIndex $adapter.InterfaceIndex -Confirm:$false -ErrorAction SilentlyContinue

# Set static IP address
New-NetIPAddress -InterfaceIndex $adapter.InterfaceIndex `
                 -IPAddress $IPAddress `
                 -PrefixLength 24 `
                 | Out-Null

# Set DNS servers
Set-DnsClientServerAddress -InterfaceIndex $adapter.InterfaceIndex `
                           -ServerAddresses $DNS1

# Restart the adapter to ensure changes take effect
Restart-NetAdapter -Name $adapter.Name -Confirm:$false

# Verify the configuration
$newConfig = Get-NetIPConfiguration -InterfaceIndex $adapter.InterfaceIndex

Write-Host "`nConfiguration successful!"
Write-Host "Interface: $($adapter.Name)"
Write-Host "IP Address: $($newConfig.IPv4Address.IPAddress)"
Write-Host "Subnet Mask: $SubnetMask"
Write-Host "DNS Server: $($newConfig.DNSServer.ServerAddresses)"

}
catch {
    Write-Host "ERROR: $_" -ForegroundColor Red
}
finally {
    Stop-Transcript
}
```
I'm then just using Windows Task scheduler to run the script on boot.

For the Domain Controller, it's important to set the DNS to `127.0.0.1`, and run the tasks as the system account and not a local user.

![](../images/nested-vms/run-as-system.png)

I'm just running the script directly with Powershell on boot: `C:\Windows\System32\WindowsPowerShell\v1.0\powershell.exe -file c:\temp\SetupNic.ps1"`
![](../images/nested-vms/setNIC.png)

# Result
The proof of concept successfully demonstrates:

- **On-demand domain environments**: Deploy a fully functional Windows domain (DC + member servers) E2E in under 10 minutes by attaching VHDs to a Hyper-V host VM in Azure.
- **Network fidelity**: Isolated, on-premise-like IP addressing (e.g., `192.168.1.0/24`) without Azure subnet reservations or NSG interference.
- **Automation-friendly**: Domain connectivity validated via `nltest /sc_reset:akingscote.com`, with static IPs configured via startup scripts.

You can tie all this together and automate the lot of it, as demonstrated in [this video](https://youtu.be/D6Jf8D_AHtg).

The complete packer definition for the Hyper-V image can be found [here](https://github.com/akingscote/msft-azure-nested-virtualization-windows-domain/blob/main/hyperVWindowsServer2025/hyperVWindowsServer2025.pkr.hcl).
Complete terraform example can be found [here](https://github.com/akingscote/msft-azure-nested-virtualization-windows-domain/tree/main/range).

Nested Virtualization is used by Microsoft themselves in the HCI stack. I recommend checking out [this video](https://www.youtube.com/watch?v=GwaP9b1N3mU), which served as inspiration for this idea.

# Creating Virtual Hard Disks (VHDs)
If you don't have a bunch of VHDs ready, you might want to deploy the above packer image (Hyper-V host) and use that to create the VHDs.

So essentially create [this image with packer](https://github.com/akingscote/msft-azure-nested-virtualization-windows-domain/blob/main/hyperVWindowsServer2025/hyperVWindowsServer2025.pkr.hcl), then run [this](https://github.com/akingscote/msft-azure-nested-virtualization-windows-domain/tree/main/base) terraform, which will create what I call a "base" VM. It's just the Hyper-V enabled VM, with a decent size hard disk which will be enough to build out a few VMs.

You'll need to create a VM switch within Hyper-V, so that the VMs can talk to each other. You'll also likely need to give each IP a static IP address and then install the various Windows server roles.

![](../images/nested-vms/virtual-switch-manager.png)
![](../images/nested-vms/vm-switch-internal)

Once your VMs are ready, you'll need to shut them down, and as explained earlier, you'll unfortunately need to convert the disks from `vhdx` files to fixed-size `vhd` files. This takes a little time and requires a fair amount of disk space. You can do this with the following powershell script:
```
Convert-VHD -Path "C:\ProgramData\Microsoft\Windows\Virtual Hard Disks\WindowsServerMember.vhdx" -DestinationPath "D:\WindowsServerMember.vhd" -VHDType Fixed
```

Once that operation is completed, you'll need to upload to your storage account. As you are copying such large files (~64GB), it's best to use `Az-Copy`.
```
$vhdPath = "C:\ProgramData\Microsoft\Windows\Virtual Hard Disks\UbuntuServer\UbuntuServer.vhdx"
$destinationUrl = "https://vhdstoragea3905e7a.blob.core.windows.net/vhds?sv=2020-02-10&ss=b&srt=co&sp=rwac&se=2025-07-28T09:50:34Z&st=2025-07-27T09:50:34Z&spr=https&sig=DbBKo%2BAC2OjCS6AmlBvbOozKn9JXbJrJW8Me1B7ISOs%3D"

# Check if AzCopy is available
if (-not (Get-Command azcopy -ErrorAction SilentlyContinue)) {
    Write-Output "Downloading AzCopy..."
    $azcopyUrl = "https://aka.ms/downloadazcopy-v10-windows"
    $downloadPath = "$env:TEMP\azcopy.zip"
    Invoke-WebRequest -Uri $azcopyUrl -OutFile $downloadPath
    Expand-Archive -Path $downloadPath -DestinationPath "$env:TEMP\azcopy" -Force
    $azcopyExe = (Get-ChildItem "$env:TEMP\azcopy" -Recurse -Filter "azcopy.exe").FullName
    Copy-Item $azcopyExe -Destination "$env:windir\system32\azcopy.exe" -Force
}

# Upload using AzCopy
Write-Output "Starting upload with AzCopy..."
azcopy copy $vhdPath $destinationUrl --blob-type PageBlob --log-level INFO

if ($LASTEXITCODE -eq 0) {
    Write-Output "VHD uploaded successfully"
} else {
    Write-Error "VHD upload failed with error code $LASTEXITCODE"
}
```
In the above example, I've added in a SAS token for the `vhds` container within the `vhdstoragea3905e7a` storage account. The storage account has long since been deleted.

Once that copy operation has completed, you are now in the starting position for the beginning of this blog.

# Summary
For security labs, training, or pre-production testing, this method reduces setup time to minutes while preserving on-premise behaviors. As Microsoft expands nested virtualization support (e.g., for Azure HCI), expect tighter integration and fewer workarounds.

I'd like to try this proof of concept again when the 6th generation Packer image bug is fixed, allowing me to use NVMe images (assuming it's possible with Hyper-V). I'd also like to up the specification for the host VM, as well as the individual VMs within the host. I'd also like to explore more mature mechanisms for configuring the VM networking, without having to rely on OS-level scripts.

I'd also love it if Microsoft allowed data disks to be created from `vhdx`, rather than forcing me to convert to fixed `vhd` files.
In a couple of years, hopefully the space is more mature.
