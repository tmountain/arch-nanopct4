# NanoPC-T4 Arch Linux Install Guide

### Introduction

This guide provides a relatively easy method for bootstrapping Arch Linux
on the NanoPC-T4's integrated eMMC storage.

The NanoPC-T4 is a fun piece of kit sporting an integrated Rockchip RK3399 CPU.
Recent updates to u-boot to officially support the T4 in its mainline branch
(`nanopc-t4-rk3399_defconfig`) remove a lot of friction regarding a previous
requirement to build custom `idbloader.bin`, `trust.bin`, `uboot.img` image files
from offshoot repos.


### Install Armbian onto you SD card.

Download an image from here (Focal recommended).

https://www.armbian.com/nanopc-t4/

### Flash the image to your SD card.

Using dd utility on Linux:

    dd if=Armbian_xx.xx.x_Nanopct4_focal_current_x.x.xx_desktop.img of=/dev/diskN bs=1M

Here's a video more comprehensive flash instructions:

https://docs.armbian.com/User-Guide_Getting-Started/

### Boot into Armbian using SD:

Press BOOT key to prevent the board from eMMC booting, making the board enter MASKROM mode.

Connect to your network inside of Armbian (network widget in top-right corner).

### Prepare the eMMC flash for Arch install (eMMC is /dev/mmcblk2).

Zero the beginning of the eMMC to wipe any existing boot images.

    # dd if=/dev/zero of=/dev/sdX bs=1M count=32

### Partition the eMMC device.

```
# fdisk /dev/mmcblk2
Command (m for help): g
Created a new GPT disklabel (GUID: 2E750097-829F-614C-AD9E-271DA3413E3E).
Command (m for help): n
Partition number (1-128, default 1):
First sector (2048-30535646, default 2048): 32768
Last sector, +/-sectors or +/-size{K,M,G,T,P} (32768-30535646, default 30535646):
Created a new partition 1 of type 'Linux filesystem' and of size 14.6 GiB.
Command (m for help): w
```

### Make the filesystem.

    # mkfs.ext4 /dev/mmcblk2p1

### Mount the filesystem.

```
# mkdir root
# mount /dev/mmcblk2p1 root
```

### Download Arch and extract the root filesystem (as root).

```
# wget http://os.archlinuxarm.org/os/ArchLinuxARM-aarch64-latest.tar.gz
# bsdtar -xpf ArchLinuxARM-aarch64-latest.tar.gz -C root
```

### Create the following boot.txt file.

```
# After modifying, run ./mkscr

part uuid ${devtype} ${devnum}:${bootpart} uuid
setenv bootargs console=ttyS2,1500000 root=PARTUUID=${uuid} rw rootwait earlycon=uart8250,mmio32,0xff1a0000
setenv fdtfile rockchip/rk3399-nanopc-t4.dtb

if load ${devtype} ${devnum}:${bootpart} ${kernel_addr_r} /boot/Image; then
  if load ${devtype} ${devnum}:${bootpart} ${fdt_addr_r} /boot/dtbs/${fdtfile}; then
    fdt addr ${fdt_addr_r}
    fdt resize
    if load ${devtype} ${devnum}:${bootpart} ${ramdisk_addr_r} /boot/initramfs-linux.img; then
      booti ${kernel_addr_r} ${ramdisk_addr_r}:${filesize} ${fdt_addr_r};
    else
      booti ${kernel_addr_r} - ${fdt_addr_r};
    fi;
  fi;
fi
```

### Optional: host root filesystem on NVMe storage.

If you want to move the root filesystem to NVMe storage at a later time, you can modify `boot.txt`
and add the appropriate UUID for your root partition. The full process for migrating to NVMe is
beyond the scope of this guide.

Get the UUID for the NVMe partition.

```
# blkid | grep nvme
/dev/nvme0n1p1: UUID="a56421bd-4e0d-48cc-b7a9-ee4fbc48aef8" TYPE="ext4" PARTUUID="8a5c1dfe-01"
```

Add the NVMe partition UUID to `boot.txt`.

    rootdev=UUID=a56421bd-4e0d-48cc-b7a9-ee4fbc48aef8

### Convert the boot.txt into a boot.scr.

    # mkimage -A arm -O linux -T script -C none -n "U-Boot boot script" -d boot.txt boot.scr

### Copy the boot.scr and boot.txt to /boot in the Arch root filesystem.

    # cp boot.scr root/boot
    # cp boot.txt root/boot

### Mount host devices into root filesystem for chroot.

```
# mount --bind /proc root/proc
# mount --bind /sys root/sys
# mount --bind /dev root/dev
# mount --bind /dev/pts root/dev/pts
```

### Chroot into the Arch filesystem to install packages required for wifi setup.

```
# chroot root
# echo "nameserver 1.1.1.1" > /etc/resolv.conf
# pacman-key --init
# pacman-key --populate archlinuxarm
# pacman -Syu
# pacman -S dialog wpa_supplicant
# exit
```

### Unmount the root Arch root filesystem.

    # umount root

### The u-boot files for the NanoPC-T4 are included with the official u-boot.

```
# dpkg -L linux-u-boot-nanopct4-current
...
/usr/lib/linux-u-boot-current-nanopct4_20.11.6_arm64/idbloader.bin
/usr/lib/linux-u-boot-current-nanopct4_20.11.6_arm64/trust.bin
/usr/lib/linux-u-boot-current-nanopct4_20.11.6_arm64/uboot.img
...
```

### Flash the boot loader to the eMMC.
```
# cd /usr/lib/linux-u-boot-current-nanopct4_20.11.6_arm64
# dd if=idbloader.bin of=/dev/mmcblk2 seek=64 conv=notrunc
# dd if=uboot.img of=/dev/mmcblk2 seek=16384 conv=notrunc
# dd if=trust.bin of=/dev/mmcblk2 seek=24576 conv=notrunc
```

Power off, remove the SD card, and power up the machine.
