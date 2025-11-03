# Proxmox Virtual GPU Setup with Intel Arc Pro B50 (SR-IOV NOT PASSTHROUGH)
## Introduction

This guide outlines the configuration required to utilize the Single Root I/O Virtualization (**SR-IOV**) capabilities of the **Intel B50 graphics card** within a Proxmox environment.

Traditional **GPU Passthrough** is the practice of dedicating an entire physical device to a single VM. For example, my **Intel Arc A770** is currently dedicated via passthrough to a single Ubuntu VM.  In contrast, the **Intel B50 card** utilizes **Single Root I/O Virtualization (SR-IOV)**. This technology allows the single physical GPU to be partitioned into multiple dedicated **Virtual Functions (VFs)**, effectively enabling the card to be **split and shared across several guest Virtual Machines (VMs)** simultaneously.

This guide focuses specifically on how to configure the **Arc Pro B50** to create these multiple virtual GPUs, as it allows me to extend high-performance graphics to several guests without needing a separate card for each one.  This capability, while common in specialized data center hardware, is now accessible to the home user at a reasonable cost, providing performance benefits for:

- **Virtualized Remote Desktop**
- **Guest OS Graphics Acceleration**
- **Media Processing (Encoding/Decoding)**

Due to the relatively new support for SR-IOV on these Intel Arc Pro cards, documented working configurations are currently fragmented across various community threads. This makes the process challenging, as one must sort through numerous evolving troubleshooting steps and comments to assemble a working solution.

This document compiles the specific, verified steps for the Proxmox host setup, including firmware updates, driver settings, and VM configuration, into one cohesive, matter-of-fact guide.
## My Configuration
* **Asus Z790 Prime** Motherboard with Intel Raptor Lake **i7 13700K** CPU and **64GB RAM** 
* **Intel Arc A770 16GB** with direct passthrough to single Ubuntu Guest for various Docker and LLM loads.  (This may change in the future, but working fine in this configuration for now. In the future it might even be possible to use this in conjunction with a virtual B50 on the same machine to provide some additional memory for LLM use)
* **Intel Arc Pro B50 16GB** to be divided amount several Windows and Linux VMs (The subject of this post)
## Host Preparation and Inspection
There are several prep steps required to get this working.
### General
(Required) - Understand that all of this is at your own risk and you should ensure you have good backups
(Recommended) Ensure you are on the newest firmware for your motherboard
(Required) Enable Resizable BAR and SR-IOV in your BIOS
(Required) Ensure your Proxmox instance booting from UEFI and the Compatibility Support Module, CSM, needs to be disabled.
(Optional) Consider updating your grub `GRUB_CMDLINE_LINUX_DEFAULT=` to include `pcie_aspm=off` if you are having stability issues.  This turns off the pcie power management low power states which can cause issues for some.
### Arc Pro B50 Firmware Update
The B50 card **must** have its firmware updated before SR-IOV can function correctly. This step **cannot** safely be performed when the card is passed through to a VM (VFIO stubbed) on Proxmox.  (Some have had success, but others have bricked their cards, so it probably isn't worth the risk)

- **Temporary Setup:** Temporarily install the **Intel B50 card** into a standard PC running **Windows**.   
- **Driver & Firmware Install:** Download and install the latest **Intel Arc Pro Graphics Windows driver**. The installer contains the utility needed to flash the card's firmware.  You may need to update a few times, and I had to install an older update first before it would allow me to install the most recent.    
- **Verify Update:** Allow the process to complete, ensuring the firmware is successfully updated.    
- **Return to Proxmox:** Move the card back to your Proxmox server for the remaining steps.
### Proxmox 9.0.11+ with 6.17+ Kernel
This process is designed for a Proxmox 9.0.11 host running the **6.17+ Linux kernel**.  Kernel 6.17 has support for Intel xe, but at this time, there is a manual step required to get to that version.
[Opt-in Linux 6.17 Kernel for Proxmox VE 9 available on test & no-subscription | Proxmox Support Forum](https://forum.proxmox.com/threads/opt-in-linux-6-17-kernel-for-proxmox-ve-9-available-on-test-no-subscription.173920/)

### Confirm the Arc Pro B50 is properly recognized with the correct drivers loaded

Scroll through the results of the lspci tool to find the physical GPU controller.  Note the key items to review underlined in red.
`lspci -vvv`
<img width="1275" height="1888" alt="Pasted image 20251103123316" src="https://github.com/user-attachments/assets/6c662a7a-396e-47ea-9491-e12fcf6ccbb1" />




---
## Splitting the card into Virtual GPU's
At this point, Proxmox sees the Arc Pro GPU and the XE driver should be loaded for the GPU, but it isn't available to the guest VM's.  Here is the process to split them out.

##### 1. Find the PCI ID associated with your GPU.  It should be the first ID shown in the response, which in my case is `0a:00.0`.
``` bash
lspci -nnk | grep "Battlemage"
0a:00.0 VGA compatible controller [0300]: Intel Corporation Battlemage G21 [Intel Graphics] [8086:e212]
```

##### 2. Use that ID to find the full PCI device path of the sriov_numfs.
```bash
find /sys/devices -path "*0a:00.0*sriov_numvfs"
/sys/devices/pci0000:00/0000:00:1b.4/0000:08:00.0/0000:09:01.0/0000:0a:00.0/sriov_numvfs
```

##### 3. To set the number of virtual GPU's, set the count of virtual GPU's in sriov_numvfs.  In this case I'm allocating 6 GPU's

``` bash
echo 6 > /sys/devices/pci0000:00/0000:00:1b.4/0000:08:00.0/0000:09:01.0/0000:0a:00.0/sriov_numvfs
```

Note - To change the number of virtual GPU's to 8, I set the value back to 0 first, otherwise I see errors.
``` bash
echo 0 > /sys/devices/pci0000:00/0000:00:1b.4/0000:08:00.0/0000:09:01.0/0000:0a:00.0/sriov_numvfs
echo 8 > /sys/devices/pci0000:00/0000:00:1b.4/0000:08:00.0/0000:09:01.0/0000:0a:00.0/sriov_numvfs
```

##### 4. Using the same command as in step 1, confirm the virtual GPU's are now showing.
```bash
lspci -nnk | grep "Battlemage"
0a:00.0 VGA compatible controller [0300]: Intel Corporation Battlemage G21 [Intel Graphics] [8086:e212]
0a:00.1 VGA compatible controller [0300]: Intel Corporation Battlemage G21 [Intel Graphics] [8086:e212]
0a:00.2 VGA compatible controller [0300]: Intel Corporation Battlemage G21 [Intel Graphics] [8086:e212]
0a:00.3 VGA compatible controller [0300]: Intel Corporation Battlemage G21 [Intel Graphics] [8086:e212]
0a:00.4 VGA compatible controller [0300]: Intel Corporation Battlemage G21 [Intel Graphics] [8086:e212]
0a:00.5 VGA compatible controller [0300]: Intel Corporation Battlemage G21 [Intel Graphics] [8086:e212]
0a:00.6 VGA compatible controller [0300]: Intel Corporation Battlemage G21 [Intel Graphics] [8086:e212]
0a:00.7 VGA compatible controller [0300]: Intel Corporation Battlemage G21 [Intel Graphics] [8086:e212]
0a:01.0 VGA compatible controller [0300]: Intel Corporation Battlemage G21 [Intel Graphics] [8086:e212]
```

You can also use pvesh to confirm Proxmox sees them
```bash
pvesh get /nodes/pve1/hardware/pci
```

##### 5.  Make the count of virtual GPU's persistent through reboots. 
Using the echo commands above work fine for testing, but they will not stay through a reboot.  Some have reported getting udev rules to work, but after several attempts, I was unsuccessful.  Instead, I chose to use sysfsutils as mentioned in some of the posts.

Install sysfsutils: 
```
sudo apt update
sudo apt install sysfsutils
```
Edit `/sys/sysfs.conf`
```shell
#### Arc Pro B50 ####
# Note the different format and heirarchy naming for this conf file compared to changes using echo in shell

# (Optional) Disable automatic driver probing, but only if you intend to manually load the vfio-pci drivers for each virtual gpu. Only added this because some posts mentioned it, but I'm not using it.
#devices/pci0000:00/0000:00:1b.4/0000:08:00.0/0000:09:01.0/0000:0a:00.0/sriov_drivers_autoprobe = 0

# Enable # of VFs.  Note setting to 0 first ensures that count changes can be applied without restart
devices/pci0000:00/0000:00:1b.4/0000:08:00.0/0000:09:01.0/0000:0a:00.0/sriov_numvfs = 0
devices/pci0000:00/0000:00:1b.4/0000:08:00.0/0000:09:01.0/0000:0a:00.0/sriov_numvfs = 6

```

## Connecting and Activating the Virtual GPU on a Guest 
##### 1. Create the Proxmox PCI mapping
In the Proxmox gui, under Resource Mappings, create add a PCI Device mapping for each virtual GPU.  In my case, I just added a number at the end to identify them.
<img width="953" height="635" alt="Pasted image 20251103102753" src="https://github.com/user-attachments/assets/dbefc086-8818-4666-a27e-37b349e2ce2e" />

Note - There may be a way to do a single mapping that shows all available as mediated devices, but I was not able to make that work.

Confirm the virtual GPU's in Proxmox are running the vfio-pci driver.  Use the lspci command, then scroll through to find the correct "child" device.  In this example, it is the .1 device, and you can see the `Kernel driver in use: vfio-pci` which means that it is ready to be passed off to the VM.  To avoid confusion, please note the kernel modules will show xe, but the important thing is that it has loaded the vfio-pci driver which what we require.

`lspci -vvv`
<img width="835" height="779" alt="Pasted image 20251103114919" src="https://github.com/user-attachments/assets/c27b59cf-d828-4a8d-82e1-ce6105e45bb4" />



##### 7. Add the the PCI Device to the virtual machine
Do not select `Primary GPU` at this time.  Once that is selected, the built in NoVNC Proxmox console stop working, so you need to have your GPU setup, and be using RDP or some other remote desktop tool.  Note - In my situation, using a Windows 11 Pro VM, it automatically uses the GPU without the need to select it as the primary.
<img width="872" height="776" alt="Pasted image 20251103104659" src="https://github.com/user-attachments/assets/970ba4df-25e0-4f7a-a920-0477b52e70c6" />


##### 8. Using the Virtual GPU on a Windows VM
At this point in time, the virtual GPU's are running on the Proxmox host with the vfio-pci drivers.  In order for the guest vm to use this device, it requires virtIO drivers on the guest VM as well.
* Install [Windows VirtIO Drivers - Proxmox VE](https://pve.proxmox.com/wiki/Windows_VirtIO_Drivers)

Once the VirtIO drivers are installed on the guest VM, restart the guest VM, then download the newest Intel Arc Pro B50 drivers on the guest.  In my experience, it took 2-3 times of installing the recommended version for it to finally say I was fully updated, but that seems to be a software issue on the Intel side since I experienced the same then when doing the original update on a Windows PC with the card installed.
## Results
<img width="516" height="446" alt="Pasted image 20251103133144" src="https://github.com/user-attachments/assets/09e67563-9c64-4042-995e-c17460d63873" />

<img width="1216" height="720" alt="Pasted image 20251103130440" src="https://github.com/user-attachments/assets/cd3c7b18-3cdf-45cd-9500-352829a0b0e9" />

<img width="2106" height="967" alt="Pasted image 20251103132835" src="https://github.com/user-attachments/assets/2594541c-2e08-4fb6-a833-3af35590ee72" />








