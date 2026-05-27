# Plain EFI Probe USB Runbook

Created: 2026-05-20

Purpose: reuse the existing diagnostics probe USB, but boot the v1.3 probe as a normal generic EFI application instead of through Apple diagnostics `boot.efi` / OCLP-style front door.

This test should produce:

```text
Apple firmware -> generic removable EFI boot -> v1.3 probe -> systemd-boot -> Linux
```

It should not use:

```text
\boot.efi
\System\Library\CoreServices\.diagnostics\Drivers\HardwareDrivers\Product.efi
```

## Before You Start

On the USB EFI partition, confirm that Linux's bootloader exists at:

```text
\EFI\systemd\systemd-bootx64.efi
```

The probe expects that exact path. If it is missing, the probe should still write `bootprobe.txt`, but chainloading Linux will fail after logging.

The local v1.3 probe binary is:

```text
C:\proj\reverse\WT6A_INF\uefi_probe_shim\out\edk2-release\Product.efi
```

## Convert The USB To Plain EFI Probe Boot

1. Rename the Apple diagnostics front door:

```text
\boot.efi
```

to:

```text
\boot.efi.diags_backup
```

2. Create this folder if it does not already exist:

```text
\EFI\BOOT
```

3. Copy the v1.3 probe binary:

```text
C:\proj\reverse\WT6A_INF\uefi_probe_shim\out\edk2-release\Product.efi
```

to the USB as:

```text
\EFI\BOOT\BOOTX64.EFI
```

4. Optional clarity step: rename the old diagnostics Product binary:

```text
\System\Library\CoreServices\.diagnostics\Drivers\HardwareDrivers\Product.efi
```

to:

```text
\System\Library\CoreServices\.diagnostics\Drivers\HardwareDrivers\Product.efi.diags_backup
```

This optional step should not be required once `\boot.efi` is renamed, but it makes the USB's active path unambiguous.

## Boot The Plain Probe

1. Power on the iMac while holding Option.
2. Pick the USB entry shown as `EFI Boot`.
3. Let the probe run and chainload systemd-boot.
4. After the run, collect:

```text
\EFI\imac5k\probe\bootprobe.txt
```

The first line should be:

```text
imac5k boot probe v1.3
```

The new comparison target is:

```text
== GOP BootCamp Support ==
```

The key question is whether plain EFI boot still has:

```text
replay current_endpoint(+0x8A2): 0 count(+0x8A4): 2 live(+0x8A8): 1
replay endpoint masks(+0x8B0): 0x00000100 0x00000200 ...
output slot cache bytes(+0x8E0..0x8E7): ... 00 01 ...
```

## Revert The USB To Diagnostics Probe Boot

1. Remove or rename the generic removable EFI probe:

```text
\EFI\BOOT\BOOTX64.EFI
```

For example:

```text
\EFI\BOOT\BOOTX64.EFI.plain_probe_backup
```

2. Restore the Apple diagnostics front door:

```text
\boot.efi.diags_backup
```

to:

```text
\boot.efi
```

3. If you did the optional clarity rename, restore diagnostics Product:

```text
\System\Library\CoreServices\.diagnostics\Drivers\HardwareDrivers\Product.efi.diags_backup
```

to:

```text
\System\Library\CoreServices\.diagnostics\Drivers\HardwareDrivers\Product.efi
```

4. Boot with Option and pick the USB `EFI Boot` entry again. The diagnostics path should again be:

```text
\boot.efi -> \System\Library\CoreServices\.diagnostics\Drivers\HardwareDrivers\Product.efi
```

