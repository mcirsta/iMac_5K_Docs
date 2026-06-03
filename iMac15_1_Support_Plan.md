# iMac15,1 Apple 5K Support Plan

Status: new evidence note and implementation plan.

Input evidence:

- `15.1/dmesg-iMac-15-1.log`
- `15.1/edid-decode-iMac15-1-eDP-1.log`
- `15.1/edid-decode-iMac15-1-DP-1.log`

This document describes what needs to change in the current `linux-imac-5k`
kernel branch to support the iMac15,1 panel family, and what must be logged
when testing so failures can be diagnosed without another blind boot cycle.

## 1. What the 15,1 capture proves

The iMac15,1 is still the same broad hardware class as the iMac19,1 work:
an Apple 27" 5K internal panel exposed as two physical DisplayPort-family
tile streams.

The important difference is the base EDID product IDs:

| Endpoint | Observed identity | Role | Key modes / metadata |
|---|---|---|---|
| `eDP-1` | `APP/AE01` | root / left tile before tile mode | 4K-compatible EDID, no DisplayID TILE block, `3840x2160`, `3200x1800`, `2560x1440` |
| `DP-1` | `APP/AE02` | slave / right tile when reachable | `2560x2880`, DisplayID 1.3 TILE block, grid `2x1`, location `1,0`, tile size `2560x2880` |
| DisplayID tiled product | `APP/AE03` | tile-group identity field | This is not the base EDID product ID used by `apply_edid_quirks()` |

The dmesg capture is from stock Fedora `7.0.9-205.fc44.x86_64` with amdgpu SI
enabled. It initializes a Pitcairn GPU on DCE 6.0 and only brings up the root
`eDP-1` sink as `APP/AE01`; all DP connectors stay disconnected in that boot.
So the DP-1 EDID file proves the right tile's real identity/topology when the
route is reachable, while the dmesg shows that stock Linux does not preserve
or wake that route by itself.

This is exactly the type of "second model" evidence requested in
`Mainline_Plan_iMac5K.md`, but it contradicts the narrow assumption that all
pre-2020 5K iMacs fit only `APP/AE25` and `APP/AE26`.

## 2. Current branch gap

The current `linux-imac-5k` `mainline-prep` branch gates the Apple 5K
panel-patch behavior only on:

```c
drm_edid_encode_panel_id('A', 'P', 'P', 0xAE25)
drm_edid_encode_panel_id('A', 'P', 'P', 0xAE26)
```

That means the iMac15,1 `APP/AE01` root will not set
`tiled_root_force_edid_reread`, and the `APP/AE02` slave will not set
`tiled_slave_root_wake`, `tiled_slave_keep_connected`, source-DPCD, training,
stream-latch, or genlock flags.

Result: with the current branch as-is, the 15,1 support path is expected to be
a no-op unless some older DMI/object-id code is reintroduced, which we do not
want.

## 3. Kernel changes needed

### 3.1 Add the 15,1 panel IDs to the EDID gate

Extend the Apple 5K panel-family match in
`drivers/gpu/drm/amd/display/amdgpu_dm/amdgpu_dm_helpers.c`:

```c
case drm_edid_encode_panel_id('A', 'P', 'P', 0xAE01):
case drm_edid_encode_panel_id('A', 'P', 'P', 0xAE02):
case drm_edid_encode_panel_id('A', 'P', 'P', 0xAE25):
case drm_edid_encode_panel_id('A', 'P', 'P', 0xAE26):
```

Keep role selection exactly as it is:

- `SIGNAL_TYPE_EDP` means root / left tile.
- `SIGNAL_TYPE_DISPLAY_PORT` means slave / right tile.

Do not use `AE01` vs `AE02` as the role check. If the 15,1 behaves like the
19,1, the root may re-read from `AE01` into `AE02` after the panel is latched
into tile mode.

Do not add `AE03` to the base-EDID quirk gate unless a future capture shows it
as a base EDID product. In the current evidence, `AE03` is the DisplayID tiled
topology product, not the EDID product used by `apply_edid_quirks()`.

### 3.2 Generalize comments and commit messages

Update comments that currently say "AE25/AE26" as if they are the whole panel
family. The better framing is:

- older 5K family: `APP/AE01` root-compat identity, `APP/AE02` tiled identity
- newer 5K family: `APP/AE25` root-compat identity, `APP/AE26` tiled identity
- role still comes from `connector_signal`, not from product ID

BootCamp RE already saw an odd/even Apple panel pattern:

- `AE01` -> Apple/tiled capability tag `0x3A`
- `AE02` -> Apple/tiled capability tag `0x3B`
- `AE25` -> Apple/tiled capability tag `0x3A`
- `AE26` -> Apple/tiled capability tag `0x3B`

So adding `AE01` and `AE02` is consistent with the RE model; it is not a random
new special case.

### 3.3 Verify the peer-selection heuristic

The current Phase 0 code wires the root to the first DP sibling after the root
EDID matches the Apple 5K family. The 15,1 stock dmesg reports:

- link 0: embedded DisplayPort, root panel
- link 1: DisplayPort, likely `DP-1`
- link 2: DisplayPort
- link 3: DisplayPort

This makes the first-DP heuristic plausible for 15,1, but it must be logged.
If the chosen DP sibling fails to respond to the pre-detect AUX poll, add a
temporary debug fallback that tries each DP sibling and logs which one returns
a valid DPCD revision or EDID. Once the correct 15,1 link ordering is known,
collapse back to the narrowest acceptable implementation.

Longer term, the robust upstream shape should still resolve the peer from real
tile metadata once both endpoints have EDID. The first-DP rule is only the
bootstrap path for the slave pre-detect phase, where the slave has no sink yet.

### 3.4 Keep the 19,1 mechanism otherwise unchanged

The existing iMac19,1 sequence remains the target:

1. Root `eDP` EDID matches Apple 5K panel family.
2. Wire `tiled_peer` between root and the likely DP slave.
3. Pulse root DPCD `0x4F1 = 1` before slave AUX operations.
4. Treat the slave as hardwired-connected when AUX proves it is real.
5. Read slave EDID and confirm `APP/AE02` with DisplayID TILE `loc 1,0`.
6. Publish source DPCD `0x310`.
7. Train the slave link at its reported full cap.
8. Force a root EDID re-read after the slave is tile-aware.
9. Expect root to change from `APP/AE01`/no-TILE to a tile-aware EDID, probably
   `APP/AE02` with DisplayID TILE `loc 0,0`.
10. Prefer `2560x2880` on both endpoints.
11. Genlock/sync the two tile streams.

If step 9 does not happen, do not synthesize TILE metadata immediately. First
log enough raw evidence to distinguish:

- root was not actually woken before the re-read
- root was re-read but still returned `AE01`
- root returned `AE02` but DRM did not apply TILE metadata
- root got TILE, but userspace/KMS did not group it correctly

## 4. Expected successful 15,1 boot trace

A successful test boot of the modified branch should show the following
high-level sequence:

```text
APPLE5K: panel match base=APP/AE01 signal=EDP link=0
APPLE5K: tiled_peer wired root link[0] <-> dp link[1]
APPLE5K: root wake 0x4F1 stage=post-root-edid status=OK
APPLE5K: slave pre-detect AUX link[1] dpcd_rev != 0
APPLE5K: panel match base=APP/AE02 signal=DP link=1
APPLE5K: slave EDID has TILE grid=2x1 loc=1,0 tile=2560x2880 tile_product=APP/AE03
APPLE5K: source DPCD 0x310 write OK value=04 1d 03
APPLE5K: link-config writes OK rate=HBR2 lanes=4
APPLE5K: root pre-reread base=APP/AE01 has_tile=0
APPLE5K: root post-reread base=APP/AE02 has_tile=1 loc=0,0 grid=2x1 tile=2560x2880
APPLE5K: preferred tile mode set on eDP-1 and DP-1
APPLE5K: timing sync group_size=2 master=eDP
DRM/fbdev/userspace sees 5120x2880 from two 2560x2880 tile streams
```

The exact wording can differ, but the facts above need to be visible.

## 5. Logging requirements for 15,1 bring-up

The goal is not permanent log spam. The goal is one or two diagnostic boots
that can answer exactly where the chain breaks.

Use a compact prefix such as `APPLE5K:` and log these fields.

### 5.1 Machine and GPU identity

Log once:

- DMI product name and board ID
- kernel version and command line
- GPU PCI ID and subsystem ID
- DC generation / DCE version
- VBIOS string
- number of DC links

This separates "15,1 on Pitcairn/DCE6" failures from "19,1 on Polaris/DCE11"
failures.

### 5.2 Link inventory

For every DC link before detection:

- `link_index`
- DRM connector name if already known
- `connector_signal`
- raw object/link ID if available
- DDC hardware instance / I2C line
- HPD GPIO and current HPD state
- encoder transmitter if available
- initial `aux_mode`

For 15,1 this tells us whether the internal slave is really link 1 / `DP-1`,
or whether the first-DP heuristic picked an external connector.

### 5.3 EDID and panel-family matching

Whenever `apply_edid_quirks()` runs on an Apple-looking panel, log:

- base EDID manufacturer/product, for example `APP/AE01`
- `connector_signal`
- link index
- display name
- extension block tags, at least `0x02` CTA vs `0x70` DisplayID
- whether the Apple 5K panel-family gate matched
- which panel-patch flags were set

Important: log base EDID product separately from DisplayID tiled product. On
15,1, base `APP/AE02` and tile product `APP/AE03` are different fields.

### 5.4 Tile metadata

After every EDID read that reaches DRM connector update, log:

- connector name
- `has_tile`
- `tile_is_single_monitor`
- tile group pointer or group ID
- grid, for example `2x1`
- tile location, for example `0,0` or `1,0`
- tile size, for example `2560x2880`
- DisplayID tile manufacturer/product/serial if easy to decode

This is the core evidence KWin needs later. KWin can only form one logical
output if both physical outputs carry coherent tile metadata.

### 5.5 Peer wiring

Log:

- root link chosen
- every candidate DP sibling
- final slave link chosen
- whether the chosen link already had a sink
- if multiple DP siblings exist, why the others were skipped

If the first choice fails AUX, a temporary debug boot should log attempted DP
fallback candidates and the first one that returns a valid DPCD revision.

### 5.6 Root wake and DPCD writes

For every root `0x4F1 = 1` write, log:

- stage: `post-root-edid`, `slave-predetect`, `source-dpcd`, `training`,
  `training-retry`, `stream-enable`
- root link index
- slave link index when applicable
- write status
- optional verify-read status/value for debug boots
- elapsed delay before the next slave AUX operation

For slave source-DPCD, log:

- `DP_SOURCE_TABLE_REVISION` / DPCD `0x310` write status
- value written, expected `04 1d 03` on DCE6/DCE11-era hardware

### 5.7 Slave AUX and link training

Log:

- pre-detect DPCD revision poll status
- HPD state before/after the poll
- elapsed time to first successful AUX response
- reported link cap: link rate, lane count, bandwidth
- verified link cap before and after verification
- link-config write statuses for downspread, lane count, link bandwidth
- number of re-wake retries
- final training result

Failure fingerprints:

- no DPCD revision: wrong peer or root wake did not revive the slave
- EDID reads but verified cap stays failsafe: training/write wake timing problem
- reported cap unknown: AUX-read problem, not only write training

### 5.8 Root EDID re-read

This is the most important 15,1 unknown. Log pre/post:

- base EDID product, for example `AE01 -> AE02`
- raw EDID bytes 8..23 for debug boots
- extension tags
- first DTD mode
- `has_tile`
- tile location/grid/size
- number of `2560x2880` modes
- result: `CHANGED`, `UNCHANGED`, or `READ_FAILED`

Expected 15,1 outcome is probably:

```text
APP/AE01 no TILE -> APP/AE02 TILE loc 0,0
```

If this is not the outcome, keep the raw EDID log. It decides whether the
panel, DC sink cache, or DRM connector cache is the failing layer.

### 5.9 Mode validation and preferred mode

Log once per connector after modes are built:

- list of probed modes
- which mode is preferred
- whether `2560x2880` exists
- whether non-tile modes were demoted, not removed
- mode validation result for `2560x2880`
- any "No DP link bandwidth" rejection

For the slave tile, also log if non-tile modes are rejected with `MODE_PANEL`.

### 5.10 Timing sync and final KMS layout

Log:

- number of streams in the DC commit
- stream signal, connector, tile location, and timing
- sync group size
- sync type
- master stream, expected eDP/root
- CRTC positions, expected `(0,0)` and `(2560,0)`
- fbdev or first compositor framebuffer size, expected `5120x2880`

If both tiles light but behave as two desktops, the kernel wake path may be
working and the remaining issue is the userspace tiled-output contract.

## 6. Recommended capture bundle for each 15,1 test

Boot with:

```text
drm.debug=0x1e log_buf_len=32M
```

Capture:

```bash
uname -a
cat /proc/cmdline
sudo dmesg -T > dmesg-iMac15-1-apple5k.log
```

Capture all connector EDIDs that exist:

```bash
for edid in /sys/class/drm/card*-*/edid; do
    [ -s "$edid" ] || continue
    name="$(basename "$(dirname "$edid")")"
    edid-decode -C "$edid" > "edid-decode-iMac15-1-${name}.log"
done
```

Capture simple connector state:

```bash
for dir in /sys/class/drm/card*-*; do
    [ -d "$dir" ] || continue
    echo "## ${dir}"
    for f in status enabled modes; do
        [ -e "$dir/$f" ] && { echo "# $f"; cat "$dir/$f"; }
    done
done > drm-connectors-iMac15-1.txt
```

If available, also capture:

```bash
drm_info > drm-info-iMac15-1.txt
modetest -M amdgpu > modetest-iMac15-1.txt
```

## 7. Failure triage table

| Symptom | Likely layer | Next thing to inspect |
|---|---|---|
| No `APPLE5K` panel match on `eDP-1` | EDID gate | Is base product `APP/AE01` missing from the switch? |
| Root matches but no `tiled_peer` | peer wiring | Did link inventory show DP candidates? Did first-DP choose the wrong one? |
| Slave DPCD poll never succeeds | wake / peer / AUX | Was `0x4F1` written on root? Try/log other DP siblings in debug build. |
| Slave EDID reads but has no TILE | wrong DP link or incomplete wake | Compare base product and extension tags. |
| Slave TILE exists, root stays `AE01` no-TILE after re-read | root re-read sequence | Check root wake timing and raw EDID bytes before/after. |
| Root becomes tile-aware but `2560x2880` is pruned | link cap / mode validation | Compare reported vs verified cap and bandwidth logs. |
| Both tile modes exist but only 4K is preferred | mode preference policy | Check tile-mode preference ran on both root and slave. |
| Both streams commit but display blanks/tears | timing sync / genlock | Check sync group size, sync type, and eDP master. |
| Both tiles work as two separate desktops | userspace grouping | Check DRM tile properties, then KWin `TiledOutputGroup` work. |

## 8. Acceptance bar for 15,1 support

Minimum success:

- iMac15,1 root `eDP-1` matches `APP/AE01`.
- Slave `DP-1` is woken and kept connected.
- Slave EDID is real `APP/AE02` with DisplayID TILE `loc 1,0`.
- Root re-read produces a real tile-aware EDID, expected `APP/AE02` with
  DisplayID TILE `loc 0,0`.
- Both endpoints expose and prefer `2560x2880`.
- DC commits both streams together and syncs them with eDP/root as master.
- DRM/fbdev or the compositor can use a `5120x2880` layout from the two tile
  streams.

Only after this is proven should `Mainline_Plan_iMac5K.md` be updated from
"AE25/AE26 panel family" to "Apple 5K panel-family table" and list the 15,1
`AE01/AE02` evidence as a tested second model.
