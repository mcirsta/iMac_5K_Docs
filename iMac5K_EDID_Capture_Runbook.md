# iMac 5K Tile-Aware EDID Capture Runbook

Date: 2026-05-18

## Purpose

Capture the tile-aware EDID that the iMac19,1 internal 5K panel reports when it
is in paired-tile state, for use in:

1. **Local validation**: testing whether `drm.edid_firmware=` override gets us
   to working 5K on a plain boot with no other workarounds.
2. **Upstream patch evidence**: extracting the panel vendor/product IDs and
   tile-group data needed for a `drm-misc` panel-quirk patch.

## Background

On Apple-blessed boot paths (macOS, OCLP/diagnostics), Apple's preboot firmware
puts the iMac 5K panel into paired-tile state. In that state, both halves of
the panel report a 256-byte EDID with a DisplayID extension describing the
2x1 tile topology at 2560x2880 per tile. The kernel's standard
`drm_edid_connector_update()` consumes this naturally and everything
downstream (modes, tile_group, encoder native_mode, DC validation) works.

On plain (non-Apple-blessed) boots, the primary tile reports a 4K-fallback
EDID with no tile information, and the secondary tile starts with dead AUX.
Even after we wake the secondary via the `0x4F1=1` probe, the primary's EDID
never updates — its byte content is identical pre- and post-wake (verified
across cold_boot_2 through cold_boot_9).

The cleanest upstreamable solution provides the panel's own tile-aware EDID
back to the kernel via `drm.edid_firmware=` (or via a panel quirk in
mainline). Capturing that EDID from a working OCLP boot is the prerequisite.

## Prerequisites

- The iMac running CachyOS (or any Linux distro) booted via
  **OCLP/diagnostics** path with 5K already working. This is the same boot
  path used in obs_22.
- Root or sudo access.
- Network or USB to transfer the captured files off the iMac afterwards.

Do NOT run this from a plain-boot kernel. The captures will not contain
tile data because the panel will be reporting its 4K-fallback EDID.

## Step 1 — Confirm working 5K state

Verify visually that the desktop spans the full 5K, and via shell:

```bash
xrandr | grep -E "(eDP-1|DP-1)" | head -2
```

Both connectors should show `connected` and a per-tile mode around
`2560x2880`. If the resolution is different or one connector shows
`disconnected`, abort — you are not in the paired-tile state we need.

## Step 2 — Identify the AMD GPU card number

```bash
ls /sys/class/drm/ | grep card
```

Look for the `cardN` directory that owns `cardN-eDP-1` and `cardN-DP-1` — the
internal panel connectors. Typically `card1` on this iMac; substitute your
number throughout the rest of this runbook.

Verify it is the AMD GPU:

```bash
cat /sys/class/drm/card1/device/uevent | head -5
```

The vendor field should read `PCI_ID=1002:...` where `0x1002` is the AMD
vendor code.

## Step 3 — Verify the connectors are in tile mode right now

The most reliable check is to read connector status and try to inspect tile
properties. Note that `/sys/class/drm/cardN-eDP-1/tile` may not exist on
every kernel version — it depends on whether the connector exposes the
tile attribute through sysfs. Its absence is not a problem; the EDID
itself is the source of truth and we decode it in step 5.

```bash
echo "=== eDP-1 status ==="
cat /sys/class/drm/card1-eDP-1/status
echo "=== DP-1 status ==="
cat /sys/class/drm/card1-DP-1/status

# Tile sysfs (may not exist on all kernels — that's fine, skip)
ls -la /sys/class/drm/card1-eDP-1/tile /sys/class/drm/card1-DP-1/tile 2>&1
cat /sys/class/drm/card1-eDP-1/tile 2>/dev/null
cat /sys/class/drm/card1-DP-1/tile 2>/dev/null
```

Both `status` files must print `connected`.

If `tile` sysfs files exist and contain data, they should look like
`<vendor>:<group>:<flags>:<num_h>:<num_v>:<loc_h>:<loc_v>:<h_size>:<v_size>`.
The primary (`eDP-1`) should end with `0:0:2560:2880` (left-tile) and the
secondary (`DP-1`) should end with `1:0:2560:2880` (right-tile). Vendor
and group bytes should match between the two — that proves a matched
tile pair.

If `tile` sysfs files do not exist, use one of these alternatives:

```bash
# Via xrandr — look for a "TILE:" property under each connector
xrandr --verbose | sed -n '/^eDP-1\|^DP-1/,/^[A-Z]/p' | head -80

# Via modetest
sudo modetest -M amdgpu -c | grep -A 50 -E "(eDP-1|DP-1)" | grep -B 2 -A 5 -i "tile"

# Most reliable: just look at the desktop size
xrandr | head -5
```

If `xrandr` shows a logical screen size around `5120x2880`, both connectors
showing `connected` at `2560x2880`-ish modes, you are in the paired-tile
state and the capture (step 4) is meaningful regardless of whether the
sysfs `tile` file exists.

If `xrandr` shows a single screen at `3840x2160` or one connector
`disconnected`, you are not in OCLP paired state and the capture cannot be
used. Reboot via OCLP and try again.

## Step 4 — Capture both EDIDs

```bash
mkdir -p ~/imac5k_edid_capture
cd ~/imac5k_edid_capture

sudo dd if=/sys/class/drm/card1-eDP-1/edid of=imac5k_primary_tile.bin   bs=4096 status=none
sudo dd if=/sys/class/drm/card1-DP-1/edid  of=imac5k_secondary_tile.bin bs=4096 status=none

ls -la *.bin
```

`dd` with `bs=4096` and no `count=` reads everything the kernel returns and
stops at EOF — bigger than any real-world EDID, so no truncation risk. We
silence `dd`'s usual stats with `status=none`; drop it if you want to see
the byte counts printed.

(Equivalent with `cat` works too — `sudo cat /sys/class/drm/cardN-X/edid >
file.bin` produces the same byte-for-byte file because shell redirection
bypasses terminal interpretation. `dd` is just more idiomatic for binary
captures.)

Both files should be **256 bytes** (a 128-byte base EDID block + a 128-byte
extension carrying the DisplayID tile data). If either file is 0 bytes or
smaller than 128 bytes, that connector was not reporting EDID at the moment
of the read — re-verify it shows `connected` and retry.

## Step 5 — Decode and verify

Install `edid-decode` if it is not present:

```bash
sudo pacman -S edid-decode
```

Decode both EDIDs and save the human-readable versions alongside the
binaries:

```bash
edid-decode imac5k_primary_tile.bin   | tee imac5k_primary_tile.txt
edid-decode imac5k_secondary_tile.bin | tee imac5k_secondary_tile.txt
```

Look for the following in each decoded output:

- **Manufacturer:** `APP` (Apple).
- **Product Code:** the 16-bit model number. Note this — it identifies the
  panel for the upstream panel-quirk table.
- **Year / Serial:** for completeness.
- **DisplayID block** containing a "Tiled Display Topology Data Block" with:
  - Vendor ID (8 bytes) — should match between primary and secondary.
  - "Number of horizontal tiles: 2".
  - "Number of vertical tiles: 1".
  - "Tile location (h, v)" = `(0, 0)` for the primary capture, `(1, 0)`
    for the secondary capture.
  - "Tile size: 2560 x 2880 pixels".
- **A detailed timing descriptor** for `2560x2880 @ ~60 Hz`. Note the
  pixel clock — should be close to `483.25 MHz`.

If all of the above is present in both decoded EDIDs, the capture is good
and the panel really is reporting full tile-aware EDID under OCLP.

If the decoded output lacks the DisplayID tile block on the primary, the
panel was not in paired state when we read its EDID — the capture cannot
be used. Reboot via OCLP and retry.

## Step 6 — Capture supporting state

These are useful for cross-referencing during patch authoring and for
commit-message justification.

```bash
cd ~/imac5k_edid_capture

# Connector tile blobs as strings
for c in /sys/class/drm/card1-*/tile; do
    echo "=== $c ==="
    cat "$c"
done > tile_properties.txt

# Mode lists per connector
for c in /sys/class/drm/card1-*/modes; do
    echo "=== $c ==="
    cat "$c"
done > mode_lists.txt

# Full DRM state from modetest
sudo modetest -M amdgpu > modetest_full.txt 2>&1

# Connector inventory
ls -la /sys/class/drm/ > drm_inventory.txt

# Kernel / system info
uname -a            > uname.txt
cat /etc/os-release > os_release.txt
sudo dmidecode -s system-product-name > dmi_system.txt 2>&1
sudo dmidecode -s system-version      > dmi_version.txt 2>&1
sudo dmidecode -s baseboard-product   > dmi_baseboard.txt 2>&1
```

## Step 7 — Bundle for off-machine review

```bash
cd ~
tar czf imac5k_edid_capture.tar.gz imac5k_edid_capture/
ls -la imac5k_edid_capture.tar.gz
```

Transfer to your normal RE workstation (e.g. via `scp`, USB stick, or any
shared mount) into:

```
C:\proj\reverse\WT6A_INF\edid_capture\
```

or wherever you keep RE artifacts.

## Step 8 — Validate the override path locally

Once captured, test that the firmware-override path works end-to-end on a
plain boot before investing in upstream patches:

```bash
# Copy the primary EDID into the kernel firmware tree
sudo mkdir -p /lib/firmware/edid
sudo cp ~/imac5k_edid_capture/imac5k_primary_tile.bin \
        /lib/firmware/edid/imac5k_primary.bin

# Add to kernel cmdline. For systemd-boot:
#   edit /boot/loader/entries/<your-entry>.conf
#   on the "options" line, add:
#     drm.edid_firmware=eDP-1:edid/imac5k_primary.bin
#
# For GRUB:
#   edit /etc/default/grub
#   GRUB_CMDLINE_LINUX_DEFAULT="... drm.edid_firmware=eDP-1:edid/imac5k_primary.bin"
#   sudo grub-mkconfig -o /boot/grub/grub.cfg
#
# Make sure /lib/firmware/edid/imac5k_primary.bin is in the initramfs if your
# distro requires firmware to be loaded from initramfs (CachyOS usually does
# not for /lib/firmware, but verify after mkinitcpio if you regenerate).
```

Reboot **without** OCLP (plain systemd-boot or GRUB direct to the kernel).
Capture dmesg and verify:

- `drm: Got external EDID base block ...` (kernel confirms the override was loaded for eDP-1).
- The IMAC5K probe still wakes the secondary (`primary_4f1_probe write 0x4f1 ... status=1`).
- After the probe + post-probe redetect, KWin sees both connectors as a tile
  pair **at startup** rather than as eDP-only-then-add-DP — this is the
  pivotal behavioral change.
- Atomic_check passes; SDDM appears at full 5K.

If this works with no other code path needed (no mode injection, no 4K
removal, no encoder native_mode override), the upstream design is validated
empirically.

## Step 9 — What we do with the captures

Once the local override is proven:

1. **Extract panel IDs.** Manufacturer + product code from the decoded EDIDs
   become entries in a `drm-misc` quirk table.
2. **Extract tile data.** Vendor 8 bytes + 2560x2880 timing descriptors
   become the data the quirk injects when those panel IDs match.
3. **Draft Patch 1** for `drm-misc`: panel-ID-gated EDID augmentation that
   adds the tile DisplayID block during `drm_edid_connector_update()`.
4. **Slim down Patch 2** for `amd-gfx`: DMI-gated iMac19,1 wake (probe +
   `0x4F1=1` write + redetect). No mode injection, no
   tile-property-from-secondary copy, no encoder cache overrides — Patch 1
   makes all of that unnecessary.

## Troubleshooting

| Symptom | Likely cause | Fix |
| --- | --- | --- |
| `card*-eDP-1` directory does not exist | Wrong card number, or amdgpu not bound | Verify with `lspci -k | grep -A3 VGA`; use the card whose driver is `amdgpu` |
| `tile` sysfs file is missing or empty | Kernel may not expose tile via sysfs (depends on version) — use `xrandr --verbose` or `modetest -c` instead, or just rely on step 5 (EDID decode) which is the source of truth |
| Tile data missing from `xrandr` AND missing from decoded EDID | Not in OCLP boot, or panel did not pair | Verify you booted via OCLP/diagnostics; check `dmesg` for `discovered secondary tiled head on DP-1` |
| Primary `edid` is 128 bytes (no extension) | Panel reporting only the base EDID, no DisplayID block | Confirm you are in paired-tile mode; the base-only EDID is the 4K-fallback one and cannot be used |
| `tile` files show different vendor / group bytes | Two displays, not a tile pair | Confirm both connectors are internal panel halves; check `cat /sys/class/drm/card*-eDP-1/edid_override` is empty (no stale override) |
| `edid-decode` not found | Package not installed | `sudo pacman -S edid-decode` (CachyOS) or equivalent |
| Capture works but the override boot still fails | Firmware not in initramfs, or path wrong, or cmdline not applied | Check `dmesg | grep -i 'edid_firmware\|external EDID'`; if absent, the override never loaded |

## Notes

- Do not commit captured EDIDs to a public Git repo without checking what
  they contain. The base block includes the panel's serial number; some
  privacy-conscious projects strip those.
- The captured primary EDID is bound to the specific iMac19,1 panel
  generation. Other iMac models (iMac17,1, iMac18,3) will need their own
  captures — they share the panel family but may differ in product code and
  tile vendor ID. If those captures become available, the upstream quirk
  patch can list all variants.
- `drm.edid_firmware=` is upstream as of mainline; we are using an existing
  mechanism, not patching the kernel for the override itself.
