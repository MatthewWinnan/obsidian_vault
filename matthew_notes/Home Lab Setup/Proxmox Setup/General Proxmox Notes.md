1. Image and versioning is given here [Proxmox Image](https://www.proxmox.com/en/downloads/proxmox-virtual-environment/iso/proxmox-ve-8-1-iso-installer) 
2. Used pi-imager to burn the image onto a bootable stick.
3. Went with the terminal installer to keep processing down, shows ip after initial boot.
4. Inspiration came from here [Ultimate Router](https://www.youtube.com/watch?v=8QTdW0Q8U3E&list=PLqb1H99dvYSce9o74cU1bebA5TDGjovmb&index=10)
5. Upload #VyOS install image to Datacenter -> Host_Name -> local -> ISO Images
6. In order to make emergency boots possible I need to passthrough my EMMC to proxmox. A general guide is here [passthrough storage](https://dannyda.com/2020/08/26/how-to-passthrough-hdd-ssd-physical-disks-to-vm-on-proxmox-vepve/)
7. In general I configured it by running ls -l /dev/disk/by-id/, in my case it displayed:
```
lrwxrwxrwx 1 root root 10 Apr  7 17:27 dm-name-pve-root -> ../../dm-1
lrwxrwxrwx 1 root root 10 Apr  7 17:27 dm-name-pve-swap -> ../../dm-0
lrwxrwxrwx 1 root root 10 Apr  7 17:27 dm-uuid-LVM-Puzh0VGlzzOOitxvuTscy1y09DbRlYXvDYyqgeRWskHkhs29tPYkXqYcnYfmcQco -> ../../dm-0
lrwxrwxrwx 1 root root 10 Apr  7 17:27 dm-uuid-LVM-Puzh0VGlzzOOitxvuTscy1y09DbRlYXvtP1mF2TQ9VBnrQquRdIYqvtYZyd32efX -> ../../dm-1
lrwxrwxrwx 1 root root 10 Apr  7 17:27 lvm-pv-uuid-dFALrQ-ueJq-cnef-EewX-WyNP-Pdke-Sc3GTP -> ../../sda3
lrwxrwxrwx 1 root root 13 Apr  7 17:27 mmc-Y29128_0x26902dcf -> ../../mmcblk0
lrwxrwxrwx 1 root root 15 Apr  7 17:27 mmc-Y29128_0x26902dcf-part1 -> ../../mmcblk0p1
lrwxrwxrwx 1 root root 15 Apr  7 17:27 mmc-Y29128_0x26902dcf-part2 -> ../../mmcblk0p2
lrwxrwxrwx 1 root root 15 Apr  7 17:27 mmc-Y29128_0x26902dcf-part3 -> ../../mmcblk0p3
lrwxrwxrwx 1 root root  9 Apr  7 17:27 usb-Realtek_RTL9210B-CG_012345678906-0:0 -> ../../sda
lrwxrwxrwx 1 root root 10 Apr  7 17:27 usb-Realtek_RTL9210B-CG_012345678906-0:0-part1 -> ../../sda1
lrwxrwxrwx 1 root root 10 Apr  7 17:27 usb-Realtek_RTL9210B-CG_012345678906-0:0-part2 -> ../../sda2
lrwxrwxrwx 1 root root 10 Apr  7 17:27 usb-Realtek_RTL9210B-CG_012345678906-0:0-part3 -> ../../sda3
```

My main mount on the emmc is thus lrwxrwxrwx 1 root root 13 Apr  7 17:27 mmc-Y29128_0x26902dcf -> ../../mmcblk0 so this ID I want to passthrough.

8. A nice enough pcie nic passthrough guide [Utilizing PCI Passthrough (VT-d) on Proxmox VE](https://kb.protectli.com/kb/pci-passthrough-vt-d-proxmox-ve/). Remember you need IOMMU to be enabled.
