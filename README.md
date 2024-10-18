*Attention! This guide is only relevant for Nouveau driver. Please read the text carefully before you start system changes.*

[Based on this AskUbuntu comment](https://askubuntu.com/a/1410487)

See list of IOMMU-supporting hardware (list may not be complete).

Important: To improve guest machine productivity, install the guest operating system on an additional physical disk (passthrough solution). Also You will need 8GB RAM for operating system (OS) and 8GB RAM for guest machine, the 16GB RAM may significantly increase your HDD's or SSD's performance and lifetime.
REQUIRED STEPS

### STEP 1. Enable Hardware-assisted virtualization in the BIOS

    For Intel based system enable Intel Virtualization Tech (VT-x) and Intel VT-D Tech

    For AMD based system enable SVM Mode.

### STEP 2. Set the BIOS to use the Integrated Graphics as the primary boot device

*Attention! If you boot the system with designed for passthrough PCI-Express Graphics Device, error code 43* will appear.*

### STEP 3. Check for IOMMU Support on your CPU

For **AMD** processor:

```$ cat /proc/cpuinfo | grep --color svm```

For **Intel** processor:

```$ cat /proc/cpuinfo | grep --color vmx```

You should see the highlighted text svm or vmx.

### STEP 4. Enable IOMMU

```$ sudo nano /etc/default/grub```

Add the following options to GRUB_CMDLINE_LINUX_DEFAULT=""

For **AMD** processor:

```amd_iommu=on kvm.ignore_msrs=1```

For **Intel** processor:

```intel_iommu=on kvm.ignore_msrs=1```

### STEP 5. Update GRUB

```$ sudo grub-mkconfig -o /boot/grub/grub.cfg```

### STEP 6. Reboot your system

### STEP 7. Check that IOMMU is enabled

```$ sudo dmesg | grep -i -e DMAR -e IOMMU```

You should see message like below:

```
[    0.123456] pci 0000:09:00.0: Adding to iommu group 1
[    0.234567] pci 0000:0a:00.0: Adding to iommu group 2
[    0.345678] DMAR: Intel(R) Virtualization Technology for Directed I/O
```

### STEP 8. Find your device

```$ lspci -nnk```

You should see text like below:

```
01:00.0 VGA compatible controller [0300]: NVIDIA Corporation TU117 [GeForce GTX 1650] [10aa:10bb] (rev a1) (prog-if 00 [VGA controller])
    Subsystem: Micro-Star International Co., Ltd. [MSI] TU117 [GeForce GTX 1650] [0101:a1a1]
    Flags: bus master, fast devsel, latency 0, IRQ 151, IOMMU group 1
    Memory at de000000 (32-bit, non-prefetchable) [size=16M]
    Memory at c0000000 (64-bit, prefetchable) [size=256M]
    Memory at d0000000 (64-bit, prefetchable) [size=32M]
    I/O ports at e000 [size=128]
    Expansion ROM at 000c0000 [disabled] [size=128K]
    Capabilities: <access denied>
    Kernel driver in use: nouveau
    Kernel modules: nvidiafb, nouveau

01:00.1 Audio device [0403]: NVIDIA Corporation Device [01cc:01ee] (rev a1)
    Subsystem: Micro-Star International Co., Ltd. [MSI] Device [0202:a2a2]
    Flags: bus master, fast devsel, latency 0, IRQ 17, IOMMU group 1
    Memory at df080000 (32-bit, non-prefetchable) [size=16K]
    Capabilities: <access denied>
    Kernel driver in use: snd_hda_intel
    Kernel modules: snd_hda_intel
```

If You see text "Kernel driver in use: nvidia" like below:

```
01:00.0 VGA compatible controller [0300]: NVIDIA Corporation TU117 [GeForce GTX 1650] [10aa:10bb] (rev a1)
    Subsystem: Micro-Star International Co., Ltd. [MSI] TU117 [GeForce GTX 1650] [0101:a1a1]
    Kernel driver in use: nvidia
    Kernel modules: nvidiafb, nouveau, nvidia_drm, nvidia
01:00.1 Audio device [0403]: NVIDIA Corporation Device [01cc:01ee] (rev a1)
    Subsystem: Micro-Star International Co., Ltd. [MSI] Device [0202:a2a2]
    Kernel driver in use: snd_hda_intel
    Kernel modules: snd_hda_intel
```

your system uses **NVIDIA Proprietary Drivers**. You need to remove all old video drivers. Here is a recipe which removes all old video drivers, and reinstalls nouveau:

```
$ sudo nvidia-settings --uninstall
$ sudo apt-get remove --purge nvidia*
$ sudo apt-get remove --purge xserver-xorg-video-nouveau
$ sudo apt-get remove --purge xserver-xorg-video-nv
$ sudo apt-get install nvidia-common
$ sudo apt-get install xserver-xorg-video-nouveau
$ sudo apt-get install xserver-xorg-video-all
$ sudo apt-get install --reinstall libgl1-mesa-glx libgl1-mesa-dri
$ sudo apt-get install --reinstall xserver-xorg-core
$ sudo dpkg-reconfigure xserver-xorg
```

or run the following command:

```$ software-properties-gtk --open-tab=4```

select the option “X.Org X server -- Nouveau display driver from xserver-xorg-video-nouveau (open source)”

Run autoremove:

```$ sudo apt autoremove```

Then reboot your system and repeat the STEP 8.

### STEP 9. Create a new file called vfio.conf

```$ sudo nano /etc/modprobe.d/vfio.conf```

Add the following lines with your device IDs from the STEP 8:

```
blacklist nouveau
blacklist snd_hda_intel
options vfio-pci ids=10aa:10bb,01cc:01ee
```

### STEP 10. Update the existing initramfs

```$ sudo update-initramfs -u```

### STEP 11. Reboot your system

### STEP 12. Make sure everything is OK

```$ lspci -nnk```

You should see text like below, the lines “Kernel driver in use: nouveau” and “Kernel driver in use: snd_hda_intel” should not be present in the text:

```
01:00.0 VGA compatible controller [0300]: NVIDIA Corporation TU117 [GeForce GTX 1650] [10aa:10bb] (rev a1)
    Subsystem: Micro-Star International Co., Ltd. [MSI] TU117 [GeForce GTX 1650] [0101:a1a1]
    Kernel modules: nvidiafb, nouveau
01:00.1 Audio device [0403]: NVIDIA Corporation Device [01cc:01ee] (rev a1)
    Subsystem: Micro-Star International Co., Ltd. [MSI] Device [0202:a2a2]
    Kernel modules: snd_hda_intel
```

### STEP 13. Add kayboard and mouse events to VM

[Based on this Arch Wiki page](https://wiki.archlinux.org/title/PCI_passthrough_via_OVMF)

If you do not have a spare mouse or keyboard to dedicate to your guest, and you do not want to suffer from the video overhead of Spice, you can setup evdev to share them between your Linux host and your virtual machine.

First, find your keyboard and mouse devices in /dev/input/by-id/. Only devices with event in their name are valid. You may find multiple devices associated to your mouse or keyboard, so try cat /dev/input/by-id/device_id and either hit some keys on the keyboard or wiggle your mouse to see if input comes through, if so you have got the right device. Now add those devices to your configuration:

```
<devices>
  ...
  <input type='evdev'>
    <source dev='/dev/input/by-id/MOUSE_NAME'/>
  </input>
  <input type='evdev'>
    <source dev='/dev/input/by-id/KEYBOARD_NAME' grab='all' repeat='on' grabToggle='ctrl-ctrl'/>
  </input>
  ...
</devices>
```
