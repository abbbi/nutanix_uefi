# Nutanix AHV install on UEFI only bios 

How to make nutanix community installation image bootable from uefi

About
-------------

Trying to install Nutanix AHV Hypervisor on recent NUC models fails
because the nutanix installation image cannot be booted from UEFI
BIOS without legacy mode.

This short howto shows a way to create a bootable medium from the downloadable
Nutani AHV Installation image, using the EFI components from Centos7.

Prerequisites
-------------

 * Nutanix installation image (ce-2019.02.11-stable.img)
 * Centos 7 Installation image (CentOS-7-x86_64-DVD-1810.iso)
 * Linux system to perform steps and test via qemu

Steps
-------------

All steps are executed with UID 0 (root)
 
Append 40 MB to the existing Nutanix installation image using fallocate (note
that the offset represents the size of the image used in this example, might
differ in future relases):

```
 fallocate -o 7444889600 -l 40m ce-2019.02.11-stable.img
```

Create a new partion using (g)parted or cfdisk in the free 40 MB region of the
disk image:

```
 cfdisk ./ce-2019.02.11-stable.img
 -> Perform steps to create the partition
```

Map the image into loopback device using kpartx:

```
 kpartx -av ./ce-2019.02.11-stable.img
 add map loop0p1 (254:0): 0 14538752 linear 7:0 2048
 add map loop0p2 (254:2): 0 81920 linear 7:0 14540800
```

Format the created 40 mb loopback partition device matching your output with
vfat.

```
 mkfs.vfat /dev/mapper/loop0p2
```

Mount the loopback partition and mount the iso image:

```
 mkdir /nutanix_vfat
 mkdir /nutanix_root
 mkdir /centos
 mount /dev/mapper/loop0p1 /nutanix_root
 mount /dev/mapper/loop0p2 /nutanix_vfat
 mount -o loop CentOS-7-x86_64-DVD-1810.iso /centos
```

Copy the needed EFI components from the Centos installation media
to the vfat partition:

```
 cp -r /centos/EFI /nutanix_vfat/
```

Edit the grub.cfg to have following contents (kernel verisons and
initrd versions might differ. I did this to work around video
garbage issues.


/nutanix_vfat/EFI/BOOT/grub.cfg:

```
set default="1"

function load_video {
  insmod efi_gop
  insmod efi_uga
  insmod video_bochs
  insmod video_cirrus
  insmod all_video
}


menuentry 'Nutanix Community Edition AHV (4.4.77-1.el7.nutanix.20190211.279.x86_64) 7 (Core)' --class nutanix --class gnu-linux --class gnu --class os --unrestricted $menuentry_id_option 'gnulinux-4.4.77-1.el7.nutanix.20190211.279.x86_64-advanced-4d8b0f1e-e014-4058-a050-c6d2ed188094' {
    load_video
    set gfxpayload=keep
    insmod gzio
    insmod part_msdos
    insmod ext2
    if [ x$feature_platform_search_hint = xy ]; then
      search --no-floppy --fs-uuid --set=root  4d8b0f1e-e014-4058-a050-c6d2ed188094
    else
      search --no-floppy --fs-uuid --set=root 4d8b0f1e-e014-4058-a050-c6d2ed188094
    fi
    linuxefi /boot/vmlinuz-4.4.77-1.el7.nutanix.20190211.279.x86_64 root=UUID=4d8b0f1e-e014-4058-a050-c6d2ed188094 ro crashkernel=128M rhgb quiet hugepages=0 intel_iommu=on,igfx_off iommu=pt elevator=noop vga=791 vfio_iommu_type1.allow_unsafe_interrupts=1
    initrdefi /boot/initramfs-4.4.77-1.el7.nutanix.20190211.279.x86_64.img
}
```

Umount everything, remove the loopback mappings and enjoy booting
nutanix AHV installer from UEFI BIOS without legacy mode.

To test with qemu use something like this:

```
  qemu-system-x86_64 --enable-kvm --bios /usr/share/qemu/OVMF.fd ce-2019.02.11-stable.img -hda  -m 15000 -hdb installme.img -hdc installme2.img 
```
