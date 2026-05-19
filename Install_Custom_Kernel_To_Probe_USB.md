# Install The Custom iMac 5K Kernel To The Probe USB

Use this after `makepkg -sri` has installed the custom `linux-imac-5k` package on CachyOS.

The goal is to copy the installed kernel and initramfs to the probe USB ESP, then make the USB `systemd-boot` default to this kernel.

Assumptions:

- the probe USB ESP is mounted at `/mnt/probe-usb`
- the custom package installs a kernel with `imac-5k` in the module directory name
- the probe USB already contains the working OCLP-style probe chain:
  - `\boot.efi`
  - `\System\Library\CoreServices\.diagnostics\Drivers\HardwareDrivers\Product.efi`
  - `\EFI\systemd\systemd-bootx64.efi`

## 1. Mount The Probe USB

Replace `/dev/sdX1` with the USB ESP partition.

```bash
sudo mkdir -p /mnt/probe-usb
sudo mount /dev/sdX1 /mnt/probe-usb
```

Verify the USB has the expected boot files:

```bash
ls -l /mnt/probe-usb/boot.efi
ls -l /mnt/probe-usb/System/Library/CoreServices/.diagnostics/Drivers/HardwareDrivers/Product.efi
ls -l /mnt/probe-usb/EFI/systemd/systemd-bootx64.efi
ls -l /mnt/probe-usb/loader/entries
```

## 2. Find The Installed Custom Kernel Version

```bash
ls /usr/lib/modules | grep imac
```

Set `KVER`:

```bash
KVER="$(ls /usr/lib/modules | grep imac-5k | tail -1)"
echo "$KVER"
```

If this prints nothing, the custom kernel package is not installed or it used a different suffix.

## 3. Generate Initramfs Images

The package hooks may already have generated these, but generating them explicitly is harmless and makes the USB copy step predictable.

```bash
sudo mkinitcpio -k "$KVER" -g "/boot/initramfs-${KVER}.img"
sudo mkinitcpio -k "$KVER" -g "/boot/initramfs-${KVER}-fallback.img" -S autodetect
```

Verify:

```bash
ls -l "/boot/initramfs-${KVER}.img"
ls -l "/boot/initramfs-${KVER}-fallback.img"
```

## 4. Copy Kernel Files To The USB ESP

```bash
sudo mkdir -p /mnt/probe-usb/EFI/Linux/imac5k
sudo cp "/usr/lib/modules/$KVER/vmlinuz" "/mnt/probe-usb/EFI/Linux/imac5k/vmlinuz-${KVER}"
sudo cp "/boot/initramfs-${KVER}.img" "/mnt/probe-usb/EFI/Linux/imac5k/initramfs-${KVER}.img"
sudo cp "/boot/initramfs-${KVER}-fallback.img" "/mnt/probe-usb/EFI/Linux/imac5k/initramfs-${KVER}-fallback.img"
```

Verify:

```bash
ls -lh /mnt/probe-usb/EFI/Linux/imac5k
```

## 5. Find The Root Filesystem UUID

```bash
findmnt -no SOURCE /
blkid "$(findmnt -no SOURCE /)"
```

Copy the `UUID=` value for your root filesystem. You will use it in the loader entry.

## 6. Create The systemd-boot Loader Entry

Create:

```bash
sudo nano /mnt/probe-usb/loader/entries/imac5k-kernel.conf
```

Template:

```ini
title   iMac 5K test kernel
linux   /EFI/Linux/imac5k/vmlinuz-KVER_GOES_HERE
initrd  /EFI/Linux/imac5k/initramfs-KVER_GOES_HERE.img
options root=UUID=YOUR_ROOT_UUID rw quiet splash
```

Replace:

- `KVER_GOES_HERE` with the real value printed by `echo "$KVER"`
- `YOUR_ROOT_UUID` with the real root filesystem UUID

Example shape:

```ini
title   iMac 5K test kernel
linux   /EFI/Linux/imac5k/vmlinuz-7.0.1-imac-5k
initrd  /EFI/Linux/imac5k/initramfs-7.0.1-imac-5k.img
options root=UUID=00000000-1111-2222-3333-444444444444 rw quiet splash
```

## 7. Make It The USB Default

Back up the USB loader config:

```bash
sudo cp /mnt/probe-usb/loader/loader.conf /mnt/probe-usb/loader/loader.conf.bak.$(date +%Y%m%d_%H%M%S)
```

If `loader.conf` already has a `default` line:

```bash
sudo sed -i 's/^default .*/default imac5k-kernel.conf/' /mnt/probe-usb/loader/loader.conf
```

If it does not have a `default` line:

```bash
echo 'default imac5k-kernel.conf' | sudo tee -a /mnt/probe-usb/loader/loader.conf
```

Verify:

```bash
grep '^default' /mnt/probe-usb/loader/loader.conf
```

## 8. Final Sanity Check

```bash
cat /mnt/probe-usb/loader/entries/imac5k-kernel.conf
cat /mnt/probe-usb/loader/loader.conf
ls -lh /mnt/probe-usb/EFI/Linux/imac5k
```

Keep at least one old working loader entry on the USB. If the test kernel fails, select the old entry from the `systemd-boot` menu.

## 9. Boot And Test

Boot the probe USB as before.

Expected chain:

```text
Apple firmware
  -> USB \boot.efi
  -> Product.efi debug shim
  -> USB \EFI\systemd\systemd-bootx64.efi
  -> iMac 5K test kernel
```

After boot:

```bash
dmesg | grep IMAC5K
uname -r
```

`uname -r` should contain `imac-5k`.
