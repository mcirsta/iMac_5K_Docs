# OCLP-Style Probe USB Runbook

This creates the bootable USB used for the iMac 5K probe path.

The boot chain is:

```text
Apple firmware -> \boot.efi -> Product.efi debug shim -> \EFI\systemd\systemd-bootx64.efi -> Linux
```

The USB must contain all three pieces on the same FAT32 ESP:

- OCLP/diagnostics-style front door: `\boot.efi`
- Our debug shim: `\System\Library\CoreServices\.diagnostics\Drivers\HardwareDrivers\Product.efi`
- Your normal Linux boot target: `\EFI\systemd\systemd-bootx64.efi`

The staged probe tree on the Windows side is:

```text
C:\proj\reverse\WT6A_INF\uefi_probe_shim\out\edk2-release\stage\esp_probe
```

## 1. Set Paths On Linux

Set this to wherever the `WT6A_INF` workspace is mounted on Linux:

```bash
export WT6A_INF_ROOT=/path/to/WT6A_INF
export STAGE="$WT6A_INF_ROOT/uefi_probe_shim/out/edk2-release/stage/esp_probe"
```

Check that the staged files are visible:

```bash
ls -l "$STAGE/boot.efi"
ls -l "$STAGE/System/Library/CoreServices/.diagnostics/Drivers/HardwareDrivers/Product.efi"
```

## 2. Identify The USB Disk

List disks carefully:

```bash
lsblk -o NAME,SIZE,FSTYPE,LABEL,MODEL,MOUNTPOINTS
```

In the commands below, replace `/dev/sdX` with the USB disk.

Be careful to use the disk path, such as `/dev/sdX`, when partitioning, and the partition path, such as `/dev/sdX1`, when formatting or mounting.

## 3. Create A Fresh FAT32 ESP

This destroys the USB contents.

```bash
sudo parted /dev/sdX --script mklabel gpt
sudo parted /dev/sdX --script mkpart EFI fat32 1MiB 100%
sudo parted /dev/sdX --script set 1 esp on
sudo mkfs.vfat -F 32 -n IMAC5KPROBE /dev/sdX1
```

## 4. Mount The Current ESP And USB ESP

Adjust the current ESP path if yours is different.

```bash
sudo mkdir -p /mnt/current-esp /mnt/probe-usb
sudo mount /dev/nvme0n1p1 /mnt/current-esp
sudo mount /dev/sdX1 /mnt/probe-usb
```

## 5. Copy The Existing systemd-boot Setup

The shim chainloads `\EFI\systemd\systemd-bootx64.efi` from the same USB, so copy your normal ESP content first:

```bash
sudo rsync -aHAX /mnt/current-esp/ /mnt/probe-usb/
```

## 6. Overlay The Probe Front Door

```bash
sudo rsync -aHAX "$STAGE"/ /mnt/probe-usb/
```

## 7. Verify The Critical Files

```bash
ls -l /mnt/probe-usb/boot.efi
ls -l /mnt/probe-usb/System/Library/CoreServices/.diagnostics/Drivers/HardwareDrivers/Product.efi
ls -l /mnt/probe-usb/EFI/systemd/systemd-bootx64.efi
ls -l /mnt/probe-usb/loader/entries
```

These four checks must pass.

If `systemd-boot` does not show your normal Linux entry later, inspect:

```bash
find /mnt/probe-usb/loader -maxdepth 3 -type f -print
find /mnt/probe-usb/EFI -maxdepth 4 -type f -print
```

## 8. Create A USB Boot Entry

Create an EFI boot entry that points to the USB root `\boot.efi`:

```bash
sudo efibootmgr --create --disk /dev/sdX --part 1 --label "iMac5K Probe USB" --loader '\boot.efi'
sudo efibootmgr -v
```

Find the new boot number in the `efibootmgr -v` output.

## 9. Boot It Once

Set the USB entry for the next boot only:

```bash
sudo efibootmgr --bootnext ####
sudo reboot
```

Replace `####` with the new boot number.

If the shim crashes or hangs, power-cycle. Because this uses `BootNext`, the machine should fall back to the normal boot path on the following boot.

## 10. Collect Probe Logs

After Linux boots, mount the USB ESP again if needed:

```bash
sudo mount /dev/sdX1 /mnt/probe-usb
```

The probe logs are:

```text
/mnt/probe-usb/EFI/imac5k/probe/bootprobe.txt
/mnt/probe-usb/EFI/imac5k/probe/bootprobe.bin
```

Useful quick checks:

```bash
ls -l /mnt/probe-usb/EFI/imac5k/probe
sed -n '1,180p' /mnt/probe-usb/EFI/imac5k/probe/bootprobe.txt
```

## Expected Result

The expected working chain is:

```text
Apple firmware
  -> USB \boot.efi
  -> \System\Library\CoreServices\.diagnostics\Drivers\HardwareDrivers\Product.efi
  -> \EFI\systemd\systemd-bootx64.efi
  -> Linux
```

If the machine boots straight to plain `systemd-boot` without creating `bootprobe.txt`, then the probe front door was bypassed and the USB boot entry likely did not point at root `\boot.efi`.

## Notes

Do not mix this blindly with a full OpenCore/OCLP USB layout. For the current probe test, use the staged `esp_probe` tree above because it has the exact diagnostics-style `boot.efi` plus our debug `Product.efi` layout.

This USB is still only a reference/probe path. The final goal remains Apple-free Linux bring-up without this USB.
