# iMac 5K Probe USB Runbook

This runbook is for creating a **test USB ESP** on the Linux-running iMac from scratch, copying the probe shim onto it, and booting it once without disturbing the current working boot path.

The goal is to boot this chain from USB:

- `\boot.efi` = Apple diagnostics front door
- `\System\Library\CoreServices\.diagnostics\Drivers\HardwareDrivers\Product.efi` = our read-only probe shim
- shim chainloads `\EFI\systemd\systemd-bootx64.efi` on the **same USB ESP**

## What You Need

### Required source files

Preferred source: copy the already staged tree from this Windows workspace location:

- [esp_probe](/C:/proj/reverse/WT6A_INF/uefi_probe_shim/out/edk2-release/stage/esp_probe)

That staged tree already contains the two critical files in the correct layout:

- [boot.efi](/C:/proj/reverse/WT6A_INF/uefi_probe_shim/out/edk2-release/stage/esp_probe/boot.efi)
- [Product.efi](/C:/proj/reverse/WT6A_INF/uefi_probe_shim/out/edk2-release/stage/esp_probe/System/Library/CoreServices/.diagnostics/Drivers/HardwareDrivers/Product.efi)

If you cannot use the staged tree directly, the raw source files are:

- Apple diagnostics front door:
  - [diags_pr777.efi](/C:/proj/reverse/WT6A_INF/RE_Files/diags_pr777.efi)
- Shim binary:
  - [Product.efi](/C:/proj/reverse/WT6A_INF/uefi_probe_shim/out/edk2-release/Product.efi)

### Important runtime requirement

The probe shim opens and chainloads this path on the **same USB volume**:

- `\EFI\systemd\systemd-bootx64.efi`

So the USB must also contain a working copy of your current ESP content, especially:

- `\EFI\systemd\systemd-bootx64.efi`
- `\loader\loader.conf`
- `\loader\entries\*.conf`
- any EFI/UKI/kernel/initrd files referenced by those entries

## Safety Model

This test should be done from a **separate USB**, not by modifying the internal ESP.

Use a one-shot boot:

- create a dedicated USB boot entry
- use `efibootmgr --bootnext`

If the shim crashes or hangs, power-cycle the machine and the next boot should return to the normal path.

## Step 1: Identify Devices

Run:

```bash
lsblk -o NAME,SIZE,FSTYPE,LABEL,PARTLABEL,MOUNTPOINTS,MODEL
sudo efibootmgr -v
```

You need to identify:

- the current working internal ESP, for example `/dev/nvme0n1p1`
- the USB disk, for example `/dev/sda`

In the commands below, replace:

- `ESP_DEV` with the current internal ESP partition
- `USB_DISK` with the whole USB disk
- `USB_PART` with the USB EFI partition, usually `${USB_DISK}1`
- `WIN_DEV` with the Windows data partition if you will mount the Windows workspace directly from Linux

## Step 2: Create a Fresh FAT32 ESP on the 32GB USB

Warning: this wipes the USB disk.

Example assumes:

- USB disk = `/dev/sda`
- USB EFI partition = `/dev/sda1`

Commands:

```bash
sudo umount /dev/sda* 2>/dev/null || true
sudo parted /dev/sda --script mklabel gpt
sudo parted /dev/sda --script mkpart EFI fat32 1MiB 100%
sudo parted /dev/sda --script set 1 esp on
sudo mkfs.vfat -F 32 -n IMAC5KPROBE /dev/sda1
```

## Step 3: Mount the Internal ESP and the USB ESP

Example mount points:

```bash
sudo mkdir -p /mnt/current-esp /mnt/probe-usb /mnt/windows
sudo mount ESP_DEV /mnt/current-esp
sudo mount USB_PART /mnt/probe-usb
```

If you will pull the staged shim tree directly from the Windows workspace, also mount the Windows partition:

```bash
sudo mount -o ro WIN_DEV /mnt/windows
```

## Step 4: Locate the Probe Source Tree

Preferred path after mounting the Windows partition:

```bash
find /mnt/windows -path '*/proj/reverse/WT6A_INF/uefi_probe_shim/out/edk2-release/stage/esp_probe' 2>/dev/null
```

The source directory you want is the Linux-visible form of this Windows path:

- `C:\proj\reverse\WT6A_INF\uefi_probe_shim\out\edk2-release\stage\esp_probe`

For the rest of the runbook, set:

```bash
export PROBE_STAGE="/mnt/windows/proj/reverse/WT6A_INF/uefi_probe_shim/out/edk2-release/stage/esp_probe"
```

If your Windows partition mounts differently, adjust that path.

Verify:

```bash
ls -l "$PROBE_STAGE/boot.efi"
ls -l "$PROBE_STAGE/System/Library/CoreServices/.diagnostics/Drivers/HardwareDrivers/Product.efi"
```

## Step 5: Clone the Current Working ESP to the USB

This is the easiest way to ensure the USB contains the exact `systemd-boot` files and loader entries your Linux setup already uses.

```bash
sudo rsync -aHAX /mnt/current-esp/ /mnt/probe-usb/
sync
```

## Step 6: Inspect What Your Loader Entries Reference

This is just a sanity check.

```bash
sudo grep -R "^\(linux\|initrd\|efi\)" /mnt/probe-usb/loader/entries/*.conf
```

If those entries refer to files under:

- `\EFI\Linux\...`
- `\EFI\systemd\...`
- or other ESP-resident paths

then step 5 already copied them.

If they point elsewhere, copy those referenced files onto the USB too.

## Step 7: Prevent the USB from Bypassing Our Front Door

On the **USB only**, rename any fallback boot files that could be selected before root `\boot.efi`.

```bash
[ -e /mnt/probe-usb/EFI/BOOT/BOOTX64.EFI ] && sudo mv /mnt/probe-usb/EFI/BOOT/BOOTX64.EFI /mnt/probe-usb/EFI/BOOT/BOOTX64.EFI.bak
[ -e /mnt/probe-usb/EFI/BOOT/APPLE/X64/BOOT.efi ] && sudo mv /mnt/probe-usb/EFI/BOOT/APPLE/X64/BOOT.efi /mnt/probe-usb/EFI/BOOT/APPLE/X64/BOOT.efi.bak
[ -e /mnt/probe-usb/System/Library/CoreServices/boot.efi ] && sudo mv /mnt/probe-usb/System/Library/CoreServices/boot.efi /mnt/probe-usb/System/Library/CoreServices/boot.efi.bak
sync
```

## Step 8: Overlay the Probe Front Door Onto the USB

Copy the staged tree onto the USB:

```bash
sudo rsync -aHAX "$PROBE_STAGE"/ /mnt/probe-usb/
sync
```

That should place:

- `/mnt/probe-usb/boot.efi`
- `/mnt/probe-usb/System/Library/CoreServices/.diagnostics/Drivers/HardwareDrivers/Product.efi`

## Step 9: Verify the Critical USB Files

Run:

```bash
ls -l /mnt/probe-usb/boot.efi
ls -l /mnt/probe-usb/System/Library/CoreServices/.diagnostics/Drivers/HardwareDrivers/Product.efi
ls -l /mnt/probe-usb/EFI/systemd/systemd-bootx64.efi
ls -l /mnt/probe-usb/loader/loader.conf
ls -l /mnt/probe-usb/loader/entries
```

If `systemd-bootx64.efi` is missing, stop and fix that before booting.

## Step 10: Unmount Cleanly

```bash
sync
sudo umount /mnt/probe-usb
sudo umount /mnt/current-esp
sudo umount /mnt/windows 2>/dev/null || true
```

## Step 11: Create a USB Boot Entry Pointing to `\boot.efi`

Example:

```bash
sudo efibootmgr --create --disk USB_DISK --part 1 --label "iMac5K Probe USB" --loader '\boot.efi'
sudo efibootmgr -v
```

Example with a real disk:

```bash
sudo efibootmgr --create --disk /dev/sda --part 1 --label "iMac5K Probe USB" --loader '\boot.efi'
```

Note the new boot number, for example `Boot0017`.

## Step 12: Use a One-Shot Boot

This is the safest way to test.

Example:

```bash
sudo efibootmgr --bootnext 0017
sudo reboot
```

If the probe boot fails, the next boot should return to your normal default path after power cycling.

## Step 13: After Linux Boots Through the Probe USB

Mount the USB again:

```bash
sudo mkdir -p /mnt/probe-usb
sudo mount USB_PART /mnt/probe-usb
```

Check for the probe output:

```bash
ls -l /mnt/probe-usb/EFI/imac5k/probe
```

Copy the logs somewhere convenient:

```bash
cp /mnt/probe-usb/EFI/imac5k/probe/bootprobe.txt ~/
cp /mnt/probe-usb/EFI/imac5k/probe/bootprobe.bin ~/
```

## Files I Want Back After the Test

- `bootprobe.txt`
- `bootprobe.bin`
- whether the USB boot reached `systemd-boot` cleanly
- whether Linux still comes up at 4K or any display behavior changes

## Exact Windows Source Locations

Preferred staged source tree:

- [esp_probe](/C:/proj/reverse/WT6A_INF/uefi_probe_shim/out/edk2-release/stage/esp_probe)

Raw binaries if you need to reconstruct manually:

- [diags_pr777.efi](/C:/proj/reverse/WT6A_INF/RE_Files/diags_pr777.efi)
- [Product.efi](/C:/proj/reverse/WT6A_INF/uefi_probe_shim/out/edk2-release/Product.efi)

## Manual Reconstruction If The Staged Tree Is Unavailable

If you cannot access the staged tree, create these USB paths manually:

- `/mnt/probe-usb/boot.efi`
- `/mnt/probe-usb/System/Library/CoreServices/.diagnostics/Drivers/HardwareDrivers/Product.efi`

Then copy:

- `diags_pr777.efi` -> `boot.efi`
- `Product.efi` -> `System/Library/CoreServices/.diagnostics/Drivers/HardwareDrivers/Product.efi`

But the staged tree is strongly preferred, because it already has the right layout.
