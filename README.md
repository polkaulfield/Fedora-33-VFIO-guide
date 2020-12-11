# Fedora 33 VFIO Guide

Here I'll post the methods I used to run a Windows 10 and OSX Catalina installs in KVM using GPU passthrough and USB controller passthrough.

# Getting the requirements
You need:
 - Fedora 33 Workstation on your system
 - Two GPUs in different IOMMU groups (I will explain that later)
 - A KVM Switch (I'm using this one [PW-SH0201B](https://www.amazon.es/gp/product/B07M6YVYST/ref=ppx_yo_dt_b_asin_title_o00_s00?ie=UTF8&psc=1), it even has keyboard bindings to switch gpus and mouse/keyboard and includes all the cables)

Optional for audio
 - Male to Male Jack cable
 - Usb soundcard


# Installing the software
Just install this group, it contains everything you need. And add yourself to the **libvirt** group.
```sh
sudo dnf install @virtualization
sudo usermod -a -G libvirt YOUR_USERNAME
```

# Enabling IOMMU
This is crucial, you need to enable the IOMMU option in your motherboard's BIOS.
Then save this as **ls-iommu.sh**
```sh
#!/bin/bash
for d in /sys/kernel/iommu_groups/*/devices/*; do
  n=${d#*/iommu_groups/*}; n=${n%%/*}
  printf 'IOMMU Group %s ' "$n"
  lspci -nns "${d##*/}"
done
```
Run it and check if your guest GPU (and usb controller) are in their own groups. In my case:

Guest GPU:
```
IOMMU Group 18:
	26:00.0 VGA compatible controller [0300]: Advanced Micro Devices, Inc. [AMD/ATI] Ellesmere [Radeon RX 470/480/570/570X/580/580X/590] [1002:67df] (rev e7)
	26:00.1 Audio device [0403]: Advanced Micro Devices, Inc. [AMD/ATI] Ellesmere HDMI Audio [Radeon RX 470/480 / 570/580/590] [1002:aaf0]
```
Guest USB controller:
```
IOMMU Group 22:
	28:00.3 USB controller [0c03]: Advanced Micro Devices, Inc. [AMD] Matisse USB 3.0 Host Controller [1022:149c]
```

You need to take note of their IDs. (In my GPU are **1002:67df** and **1002:aaf0**. The USB one is **1022:149c**)
For now, I'll call them **GPU_ID1** **GPU_ID2** and **USB_ID**

# Setting up VFIO and PCI-STUB
These are 2 drivers that bind to PCI devices so they are free from the host. VFIO is superior normally, but I had problems binding it to my USB interface.

First add the drivers to your initramfs with Dracut.
```sh
$ sudo vim /etc/dracut.conf.d/vfio.conf
```
Add this to that file:
```
add_drivers+=" vfio vfio_iommu_type1 vfio_pci vfio_virqfd "
```
And remake the initramfs.
```sh
$ sudo dracut -f
```

Then we need to edit your kernel parameters in GRUB.
```sh
$ sudo vim /etc/default/grub
```

Remember the GPU_ID1, GPU_ID2 and USB_ID from before? Now it's time to use them.

For AMD use:
```
... quiet amd_iommu=on amd_iommu=pt rd.driver.pre=vfio-pci vfio-pci.ids=GPU_ID1,GPU_ID2 pci-stub.ids=USB_ID
```

For Intel use:
```
... quiet intel_iommu=on intel_iommu=pt rd.driver.pre=vfio-pci vfio-pci.ids=GPU_ID1,GPU_ID2 pci-stub.ids=USB_ID
```

Regenerate your GRUB config and reboot.
```sh
$ sudo grub2-mkconfig -o /etc/grub2-efi.cfg
```

After the reboot, don't panic, your guest GPU should be disabled and your specified USB controller should too.
Check if they are bound to the VFIO and PCI-STUB drivers before proceeding.
```sh
lspci -nnv
```
Look for this in the output:
```
Kernel driver in use: vfio-pci
```
The USB controller sometimes attaches to vfio, others to pci-stub for some reason. But it works so... No idea.

If you got here, we can go to the next step. Attach your KVM USB cable and your USB audio card to your guests USB controller. 
# Creating the VM
The easiest way is to create one normally with virt-manager. Be careful at the last step to choose UEFI (OVMF) instead of BIOS. We need OVMF for this to work.

When you have it created, open up the VM config and attach your passed through devices. (Add Hardware > PCI Host Device). Check the IDs if you are not sure which ones are.

If you are gonna use the KVM switch and passed through GPU right away, remove the **Video** and **Graphics** virtual devices from your VM config. You can do it after installation though.

I recommend using Windows 10 Enterprise LTSC 2019 for the guest. It's light and doesn't take a lot of resources.
Filename and hash (use Google)
```
en_windows_10_enterprise_ltsc_2019_x64_dvd_74865958.iso
d6c7eca8741948eb91638717b3d927c3f122a803545a2e05fe412abcadddb8fe
```
Activate with this:
(https://github.com/abbodi1406/KMS_VL_ALL_AIO)

When you have everything installed and working we can move to the next step.

# Audio with low latency (loopback interface with pulseaudio)
The idea is to connect the output jack of your USB soundcard into the line-in jack (NOT THE MICROPHONE) of your motherboard.

We now we need to find out the ID of the input jack so we can create the loopback device. Use pacmd.
```sh
$ pacmd list-sources
```
And look for the correct ID, in my case:
```
name: <alsa_input.pci-0000_28_00.4.analog-stereo>
```

Now we can create the loopback device. The easiest way is to copy  the **/etc/pulse/default.pa** file in your **~/.config/pulse//** directory, and add the extra parameters that we need.

```sh
$ cp /etc/pulse/default.pa ~/.config/pulse
$ vim ~/.config/pulse/default.pa
```

Add this at the end of the file.
```
## Line in loopback
load-module module-loopback latency_msec=1 source='alsa_input.pci-0000_28_00.4.analog-stereo' source_dont_move=1
```
Where in source you have to add the ID you got before.
Reset pulse or login and out and you should have the interface enabled. Configure volume levels and everything with **pavucontrol**. GNOME is limited when it comes to managing pulse devices.

```sh
$ sudo dnf install pavucontrol
```

# Setting up Samba for sharing files.
For samba we need to install the samba server, configure it and create an isolated network to connect from the guest to the host. We also need to add a firewall rule.
Start with:
```sh
$ sudo dnf install samba
$ sudo smbpasswd -a YOUR_USERNAME
$ sudo vim /etc/samba/smb.conf
```
There, paste this config editing it to your needs. (Check the interface name so it matches with your isolated network one)

```ini
[global]
	workgroup = SAMBA
	security = user
	passdb backend = tdbsam
	bind interfaces only = yes
	force user = YOUR_USERNAME

[VMshare]
	path = /home/absurd/VMshare # Here the path you want.
	browseable = yes
	read only = no
	force create mode = 0660
	force directory mode = 2770
	valid users = @YOUR_USERNAME
```
If you want to have the shared folder inside your home directory, apply this SELinux rule.

```sh
$ sudo setsebool -P samba_enable_home_dirs on
```

Add a firewall exception to the libvirt firewall zone.
```sh
$ sudo firewall-cmd --zone=libvirt --add-service=samba --permanent
$ sudo firewall-cmd --reload
```
Start and enable samba.
```sh
$ sudo systemctl start smb.service
$ sudo systemctl enable smb.service
```
Try to access your shared folder from your guest using this address: **\\192.168.110.1**

# OSX Host with GPU passthrough
**TODO**: Explain how to set up everything with help of this project: (https://github.com/foxlet/macOS-Simple-KVM/)

# Improving performance
This guide has a lot of info: (https://mathiashueber.com/performance-tweaks-gaming-on-virtual-machines/)
