# kvm-vfio-notes

Notes on getting KVM VFIO working on my hardware:
- CPU: Ryzen 7 5700G
- MB: Biostar B550T-SILVER
- Host GFX: Ryzen 7 5700G iGPU
- Passthrough GFX: PCIe Nvidia RTX 4000 SFF Ada Generation 20GB
- BIOS version: B55AK830 (AMD AM4 AGESA 1.2.0.Cc) (https://www.biostar.com.tw/app/en/mb/introduction.php?S_ID=1010&data-type=DOWNLOAD)

As of 2024-12-05, KVM VFIO passthrough of a current-gen Nvidia card while retaining the Ryzen 5700G's IGP as the host's framebuffer console is fully working.

I now provide a set of screenshots documenting every BIOS option, whether I changed it or not, as a means of providing a "known-good snapshot" of a working BIOS configuration. You will find them in the /screenshots directory of this repo.


[ December 2024 NOTES: ]
1. I have found that the linux-amd-znver3 kernel reliaby works for passthrough, initializing devices in a sane order and respecting boot-time blacklist kernel args https://aur.archlinux.org/packages/linux-amd-znver3
2. I have since managed to get the Nvidia RTX A2000 12GB working in passthrough
3. The RTX 4000 SFF Ada Generation 20GB works as a drop-in replacement for the A2000.
4. Stability is much improved - I don't know what to blame, but crashes are infrequent enough (i.e. one crash every 6-8 months) that I haven't spent time on it.
5. I have recently transitioned away from Arch to openSUSE Tumbleweed, which works out-of-the-box as a passthrough host and does not constantly break. [ December 2024 note: openSUSE is love, openSUSE is life. ]
6. I am separately maintaning a variant of the linux-amd-znver3 and linux-amd-znver2 configs which have been tweaked to work on openSUSE and are available at https://github.com/theodric/linux-amd-zen2-zen3/



2022 NOTES - FULLY WORKING ALL-AMD PASSTHROUGH 

In this configuration, I have two fully independent "computer stations" that can be used simultaneously:
1. the host Linux system, on which I have a KDE desktop
2. the guest VM, which can be Windows, Linux, or (theoretically) macOS (but that's illegal, so don't break the law, OK?)

Arch Linux
CPU: Ryzen 7 5700G
MB: Biostar B550T-SILVER
Host GFX: Ryzen 7 5700G iGPU
Passthrough GFX: PCIe MSI Radeon 560 ITX (fully working in Windows and Linux guests)

In UEFI setup:

In Advanced -> PCI Subsystem Settings:
- Enable Above 4G decoding
- Disable Resizable BAR
- Enable SR-IOV support
- Enable BME DMA Mitigation

In Advanced -> CSM Support
- Option ROM Execution:
- Network - UEFI
- Storage - UEFI
- Video - Legacy - forces PCIe GPU to legacy mode so it doesn't get grabbed by UEFI, which would stop it from being free for passthrough to the VM
- Other PCI Device - UEFI

In Chipset:
- In North Bridge Configuration
- IOMMU -> Enabled
- Primary video device -> IGD Video

append kernel args: iommu=pt and vfio-pci.ids=<PCI IDs of all cards to be passed through>

Find those IDs and check their PCIe group assignments with this script:

`[root@cube ~]# cat vfio.sh
#!/bin/bash
shopt -s nullglob
for g in $(find /sys/kernel/iommu_groups/* -maxdepth 0 -type d | sort -V); do
    echo "IOMMU Group ${g##*/}:"
    for d in $g/devices/*; do
        echo -e "\t$(lspci -nns ${d##*/})"
    done;
done;`

I am passing through 1 or 2 ports of the onboard USB 3.2 controller that is in its own PCIe device group, because the other controllers are in groups with other devices and don't break out cleanly even with the additional kernel args, resulting in instability.

The WLAN socket on this board functions perfectly as a standard PCIe 4x slot with an adapter cable, but again, it's part of a group of several other devices. However, adding a USB3 card here helps make up for the ports lost to passthrough.
-> TODO: see if my PCIe switch works on this

Works with all kernels tested so far:

[root@cube ~]# cat /proc/cmdline
BOOT_IMAGE=/boot/vmlinuz-linux-zen root=UUID=cbde20e5-0804-4179-ae95-7c1752d86f04 rw loglevel=3 net.ifnames=0 biosdevname=0 noibrs noibpb nopti nospectre_v2 nospectre_v1 l1tf=off nospec_store_bypass_disable no_stf_barrier mds=off tsx=on tsx_async_abort=off mitigations=off iommu=pt vfio-pci.ids=1002:67ff,1002:aae0,1022:1639

After that, it's just a matter of assigning the PCIe devices and/or host USB devices to the VM in either the virt-manager GUI, or else directly in the config file if you're a masochist.
To get rid of the Spice display that is auto-populated, you may have to resort to editing XML, because the GUI will often refuse to let you delete it, citing some dependency.
After that, you can just toggle the virtual (non-GPU) display on and off as required by changing the Video -> Model setting in the VM properties between "None" and the other options.

######### OLD 2021 NOTES - PROXMOX ON MAC PRO 5,1 ################
The device to be passed through can't be held by any driver. 
OSX-KVM instructions say to blacklist the graphics card driver and use only integrated graphics, but that doesn't work for me because my Socket 1366 Xeon does not have onboard graphics, and both of the video cards in the system use the amdgpu kernel module.

The solution is to use the sysfs interface to unbind the device I will pass through to macOS from the driver, like so:
root@kvm-mule:/sys/bus/pci/drivers/amdgpu# lspci -vv | grep -i amd
01:00.0 VGA compatible controller: Advanced Micro Devices, Inc. [AMD/ATI] Baffin [Radeon RX 550 640SP / RX 560/560X] (rev cf) (prog-if 00 [VGA controller])
	Kernel driver in use: amdgpu
	Kernel modules: amdgpu
01:00.1 Audio device: Advanced Micro Devices, Inc. [AMD/ATI] Baffin HDMI/DP Audio [Radeon RX 550 640SP / RX 560/560X]
02:00.0 VGA compatible controller: Advanced Micro Devices, Inc. [AMD/ATI] Tahiti XT [Radeon HD 7970/8970 OEM / R9 280X] (prog-if 00 [VGA controller])
	Kernel modules: radeon, amdgpu
02:00.1 Audio device: Advanced Micro Devices, Inc. [AMD/ATI] Tahiti HDMI Audio [Radeon HD 7870 XT / 7950/7970]

root@kvm-mule:/sys/bus/pci/drivers/amdgpu# echo -n 0000:01:00.0 > unbind #detach - works live
root@kvm-mule:/sys/bus/pci/drivers/amdgpu# echo -n 0000:01:00.0 > bind #reattach - works live
