# Android device tree for basic x86_64 PC

This device tree is meant to remain minimal, for kickstarting purposes.

If you are looking forward to something that should have as many as possible working features on most of devices,
please visit the [mainline/generic](https://github.com/LineageOS/android_device_mainline_generic) device tree.

```
#
# SPDX-FileCopyrightText: The LineageOS Project
# SPDX-License-Identifier: Apache-2.0
#
```

Using this as a starting point for your own device? See
`device/mainline/common/docs/README.md`.

# Steps to boot

(The example environment used in below is a x86_64 PC running Debian with GRUB boot manager, and the hard drive is connected to SATA bus)

1. Build this target, and then enter the output directory, using the following commands:

```
source build/envsetup.sh
breakfast basic_x86_64_pc
m kernel ramdisk recoveryimage systemimage vendorimage
cout
```

2. Put the kernel and ramdisk images to anywhere that the boot manager can access.

```
mkdir /android
cp kernel ramdisk.img ramdisk-recovery.img /android/
```

3. Create empty partitions

Ensure to have the following partitions:

| Purpose  | Minimum size | Example path |
|----------|--------------|--------------|
|  system  |    2.5 GB    |  /dev/sda5   |
|  vendor  |    256 MB    |  /dev/sda6   |
| userdata |     2 GB     |  /dev/sda7   |
| metadata |    16 MB     |  /dev/sda8   |

4. Write system and vendor image

```
dd if=system.img of=/dev/sda5
dd if=vendor.img of=/dev/sda6
```

5. Add GRUB boot entries

Get the kernel parameters using the following command:

```
get_build_var BOARD_KERNEL_CMDLINE
```

Add the following to `/boot/grub/custom.cfg`:

```
menuentry "Android" {
    set gfxpayload=keep # For VESA framebuffer when booted from BIOS
    linux /android/kernel <kernel parameters>
    initrd /android/ramdisk.img
}

menuentry "Android Recovery" {
    set gfxpayload=keep # For VESA framebuffer when booted from BIOS
    linux /android/kernel <kernel parameters>
    initrd /android/ramdisk-recovery.img
}
```

Update the parameter `androidboot.partition_map`:

- Make it reflect the actual partition setup. For this example: `sda5,system;sda6,vendor;sda7,userdata;sda8,metadata`
- Escape ';' characters, by prepending '\\' before these. For this example: `sda5,system\;sda6,vendor\;sda7,userdata\;sda8,metadata`

# Details about creating bootable GRUB disk

## A FAT32 partition containing GRUB and Android boot files

The partition should be `EFI System Partition` on GPT disk, or a partition with type `W95 FAT32` otherwise.

Directory structure:

| Path | Source | Description |
|------|--------|-------------|
| /android/kernel | out/target/product/basic_x86_64_pc/kernel | |
| /android/ramdisk.img | out/target/product/basic_x86_64_pc/ramdisk.img | |
| /android/ramdisk-recovery.img | out/target/product/basic_x86_64_pc/ramdisk-recovery.img | |
| /boot/grub/grub.cfg | Manually written | Content is provided below |
| /boot/grub/i386-pc/ | prebuilts/bootmgr/grub/linux-x86/i386-pc/lib/grub/i386-pc/ | For BIOS boot |
| /boot/grub/x86_64-efi/ | prebuilts/bootmgr/grub/linux-x86/x86_64-efi/lib/grub/x86_64-efi/ | For UEFI boot |
| /EFI/BOOT/BOOTX64.EFI | Generated using command | For UEFI boot |

Content of `/boot/grub/grub.cfg`:

```
insmod normal
insmod linux
insmod test

if [ "$grub_platform" = "efi" ]; then
    insmod efi_gop
elif [ "$grub_platform" = "pc" ]; then
    insmod vbe
fi

<Same as the content from `/boot/grub/custom.cfg` in above of the page>
```

Content of `grub-standalone.cfg`:

```
search --file --no-floppy --set=root /android/kernel
set prefix=($root)'/boot/grub'
configfile $prefix/grub.cfg
```

Command to create `/EFI/BOOT/BOOTX64.EFI`:

```
PATH=prebuilts/bootmgr/tools/linux-x86/bin/:$PATH prebuilts/bootmgr/grub/linux-x86/x86_64-efi/bin/grub-mkstandalone \
    -d prebuilts/bootmgr/grub/linux-x86/x86_64-efi/lib/grub/x86_64-efi \
    --fonts="" \
    --format=x86_64-efi \
    --locales="" \
    --modules="configfile disk fat part_gpt search" \
    --output=BOOTX64.EFI \
    "boot/grub/grub.cfg=grub-standalone.cfg"
```

## BIOS boot support

Import GRUB `boot.img`:

```
dd if=prebuilts/bootmgr/grub/linux-x86/i386-pc/lib/grub/i386-pc/boot.img of=boot.img bs=446 count=1
```

Create GRUB `core.img`:

```
PATH=prebuilts/bootmgr/tools/linux-x86/bin/:$PATH prebuilts/bootmgr/grub/linux-x86/i386-pc/bin/grub-mkimage \
    --compression=auto \
    --config=grub-standalone.cfg \
    --directory=prebuilts/bootmgr/grub/linux-x86/i386-pc/lib/grub/i386-pc \
    --format=i386-pc \
    --output=core.img \
    --prefix=/boot/grub \
    fat msdospart part_msdos part_gpt biosdisk search configfile
```

Confirm the start sector of GRUB `core.img` on disk.
If the disk uses GPT partition scheme, create a 1MB partition with type `BIOS boot partition`, and use its start sector.
Otherwise, use sector 1.

Patch and write GRUB `boot.img` and `core.img`:

```
gcc device/virt/virt-common/utilities/grub_i386-pc_img_patch/grub_i386-pc_img_patch.c -o grub_i386-pc_img_patch
./grub_i386-pc_img_patch <start sector of core.img> boot.img core.img
dd if=boot.img of=/dev/sdX bs=446 count=1
dd if=core.img of=/dev/sdX bs=512 count=2048 seek=<start sector of core.img>
```
