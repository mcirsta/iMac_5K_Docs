# iMac 5K Cold-Boot Enablement Plan

Created: 2026-05-15

This document tracks the next phase of the iMac19,1 5K work: booting Linux directly, without OCLP, diagnostics disguises, or an extra preboot shim, while still reaching the full internal 5K tiled panel state.

## Goal

Enable the internal iMac 5K panel from a plain Linux boot path:

```text
Apple firmware -> normal EFI boot entry -> systemd-boot/GRUB -> Linux
```

The finished solution should not require:

- OCLP
- Apple diagnostics `boot.efi`
- `Product.efi` shim
- manually selected alternate boot path
- fake 5120x2880 sink replacing the real tile routes

The preferred final implementation is an upstreamable or at least tightly gated Linux/amdgpu quirk that cold-initializes or revives the real secondary tile route.

## Current Model

The internal panel is a two-tile display.

- Primary tile: AMD object `0x3114`, Linux `eDP-1`, DDC4, UNIPHY_C.
- Secondary tile: AMD object `0x3113`, Linux `DP-1` when exposed, DDC3, UNIPHY_D.
- Full 5K state requires two real `2560x2880` tile streams, correct TILE metadata, and a stable secondary AUX/link backend.

The OCLP/probe path works because Apple firmware leaves or creates a richer preboot display state before Linux starts. Plain boot currently reaches a 4K-compatible fallback state where the secondary `0x3113` route may be enumerable but is not live: no HPD/AUX/local sink/EDID/TILE.

The Linux cold-boot task is therefore not just a userspace TILE metadata problem. Linux must make the real secondary route usable.

## RE Checkpoint: CoreEG2 Cold-Boot Clues

IDA MCP currently exposes `CoreEG2.efi.i64`.

Findings from the 2026-05-15 CoreEG2 pass:

- `CoreEg2ComplexDisplayInit` is gated by `dword_E670 & 1`. If that bit is clear, CoreEG2 skips the complex path and falls back to `CoreEg2GenericInitializeGraphicsDevice`.
- `dword_E670` is read from a provider/protocol during module entry, not from EDID directly.
- The provider key GUID for `dword_E670` is `1823A1A6-0EAF-4F3A-93D3-BF61F5EEFA0D`.
- The provider protocol for this read is `AC5E4829-A8FD-440B-AF33-9FFE013B12D8`, published by `ApplePlatformInfoDatabaseDxe.efi`.
- The selector data is HOB/platform-database backed. In the raw platform database, selector `1823A1A6-...` points to a 48-byte table:
  - `j139` -> returned dword `5`
  - `j138` -> returned dword `1`
- The same platform database maps `j138` to `iMac19,1` and `j139` to `iMac19,2`.
- The complex path finds a root/internal connector, creates or reuses a display object, sets signal `11`, then performs a second-connector setup phase.
- During second-connector setup, CoreEG2 sends opcode/address `1265` (`0x4F1`) with payload `1` through the root connector backend, stalls `10 ms`, then searches for a sibling connector whose connector id is root id plus one.
- The sibling connector must expose an Apple internal-panel record matching byte `0xAE`, big-endian word `0x0610`, and `(flags & 3) == 2`.
- If second-connector validation fails, CoreEG2 sends opcode/address `1265` with payload `0` through the same root connector backend before returning to generic fallback.
- After the paired path succeeds, CoreEG2 link-trains both connectors and publishes `ComplexDisplaySetup` through the platform-info backend.

Implication:

The primary/root-side `0x4F1` probe is now justified as a firmware-shaped experiment, even though the Windows runtime path still argues that the final production solution must operate on the real secondary `0x3113` route once that route exists.

The current evidence does not support "OCLP makes `dword_E670` bit0 set while plain boot leaves it clear" as the main difference. For the target `iMac19,1` token `j138`, the platform policy value should be `1`, so bit0 should already be set. The OCLP/diagnostics difference is more likely that the sibling connector/panel state becomes discoverable after the root-side `0x4F1` sequence, while plain boot fails that second-connector validation and falls back.

## RE Checkpoint: AMD GOP Connector Backend

The second IDA MCP instance is attached to the Polaris/Apple GOP module:

```text
RE_Files/FirmwarePE32_All/F0140FC0-EC2A-451E-ADC5-D671AC453A8C.efi.i64
```

Findings from the 2026-05-15 GOP pass:

- `GopGraphicsConnectorProtocolTransfer` is the direct connector protocol transfer entry.
- `GopGraphicsConnectorBackendDispatchOpcode` is the lower backend entry CoreEG2 reaches for full-opcode operations.
- The backend dispatch mode `a4 == 1` selects the extended opcode window and preserves the full caller opcode `a2` as a value up to `0xFFFFF`.
- Therefore CoreEG2's decimal `1265` send is not truncated to low byte `0xF1`; GOP sends it as full opcode/address `0x4F1`.
- `GopWriteExtendedOpcodeWindow` writes payload chunks through the command window at `a1 + 35392..35396`.
- `GopReadExtendedOpcodeWindow` reads payload chunks through the same command window.
- `GopSubmitConnectorMailboxTransaction` submits the prepared command and polls the status/count word.
- `GopPublishDpcdTrainingProperties` programs normal DP DPCD training addresses through the same extended opcode transport: `0x100`, `0x101`, `0x102`, `0x103`, `0x107`, `0x108`, and reads training status from `0x202`.
- GOP has an optional `0x10A` write path gated by internal flags, matching the Windows/Linux suspicion that `0x10A` is part of the Apple panel policy.
- `GopRefreshConnectorStateSlotFromExtendedWindows` skips broad extended ranges including `0x410..0x4FF`; this makes CoreEG2's `0x4F1` write look deliberate rather than a generic connector-state refresh side effect.

IDA annotations saved in the GOP IDB:

- `sub_1ADF0` -> `GopExecuteConnectorMailboxTransaction`
- `sub_1BA90` -> `GopSubmitConnectorMailboxTransaction`
- `sub_1BFA0` -> `GopWriteExtendedByteOpcode`
- `sub_1BF30` -> `GopReadExtendedByteOpcode`
- `sub_CF60` -> `GopReadDpTrainingStatusWindows`
- `sub_E5F0` -> `GopReadExtended2210Bit3`

Implication:

The firmware and Windows/Linux models now agree better than before. `0x4F1` is still best treated by Linux as a DPCD/AUX address, but the Apple firmware gets there through GOP's full-opcode connector backend. The one-boot H1 primary/root probe is stronger, while the production H2 path still needs the real secondary `0x3113` AUX/link route.

## RE Checkpoint: CoreEG2/GOP Shared Graphics Handshake

New 2026-05-15 finding from the two-IDB pass:

- `CoreEG2.efi` publishes Apple shared graphics protocol `63FAECF2-E7EE-4CB9-8A0C-11CE5E89E33C` at module entry.
- The AMD/Apple GOP locates that same protocol in `GopLocateSharedGraphicsProtocol`.
- GOP calls the shared protocol to fetch platform info and, in saved-display-config paths, connector record sets from CoreEG2 display objects.
- CoreEG2 shared slot `+0x20` is now named `CoreEg2SharedFetchConnectorRecordSetForGop`.
- That method matches the GOP's device/connector request against CoreEG2's device list, finds or creates a display object for the requested connector signal, and returns a copied connector-record set from `display+0x90` / `display+0x98`.
- GOP's saved-config probe can pre-seed its connector-record caches from CoreEG2 state:
  - `GopProbeConnectorRecordCachesFromDisplayTable`
  - `GopFetchSharedConnectorRecords`
  - `GopGetConnectorRecordCacheByIndex`
  - `GopGetConnectorRecordValidFlagByIndex`
- GOP connector id `1` maps to backend mask `0x100` and connector-record cache index `4`.
- GOP connector id `2` maps to backend mask `0x200` and connector-record cache index `3`.
- These are GOP connector ids, not AMD DAL object ids. CoreEG2 still chooses the sibling by `connector_id == root_id + 1`.

Interpretation:

This explains another way the "OCLP/probe path" can differ from a plain boot: if an Apple-looking path causes CoreEG2 to create and cache richer display objects, the GOP can later seed connector records from that shared CoreEG2 state. That does not overturn the main CoreEG2 complex-display decision, though. `CoreEg2ComplexDisplayInit` still makes its paired 5K/4K choice by live-fetching connector records after the root-side `0x4F1 = 1` pulse and validating the sibling Apple internal-panel record.

So the current suspected plain-boot failure point is narrower:

```text
dword_E670 bit0 is probably already set
-> root connector record must qualify as Apple internal panel
-> CoreEG2 writes root/backend 0x4F1 = 1
-> sibling connector id root+1 must exist and carry signal 11
-> sibling record opcode 0xA0 must return +8/+9 == 0x0610, +0x0B == 0xAE, flags&3 == 2
-> root record must still/re-again validate the same way
-> only then does CoreEG2 commit the paired display object
```

The next useful observable is not another platform-provider bit. It is whether the `0x4F1` pulse makes the secondary/sibling record become live enough to pass the Apple internal-panel test.

## RE Checkpoint: GOP Device Protocol And Commit Point

Additional 2026-05-15 finding:

- The GOP publishes another private protocol, `APPLE_GOP_DEVICE_PROTOCOL_GUID = 53CFEB0D-9B9E-4D0A-90B6-024A79906A58`.
- CoreEG2 has a dedicated driver binding for that GOP device protocol:
  - `CoreEg2BindingSupportedGopDeviceProtocol`
  - `CoreEg2BindingStartGopDeviceProtocol`
  - `CoreEg2BindingStopGopDeviceProtocol`
- GOP start order is:

```text
GopInstallDeviceMailboxProtocol
-> install APPLE_GOP_DEVICE_PROTOCOL_GUID
-> GopInstallConnectorMailboxProtocols
-> install APPLE_AMD_CONNECTOR_PROTOCOL_GUID / CF611019-...
-> GopInstallFramebufferProtocols
-> install APPLE_GOP_FRAMEBUFFER_PROTOCOL_GUID / F977B8AE-...
```

- CoreEG2 binds the GOP device protocol first, then counts the connector/framebuffer protocol instances.
- `CoreEg2ComplexDisplayInit` fires only when both expected counts are satisfied. This is still DXE driver-binding connect time, before BDS launches `systemd-boot`, GRUB, or any bootloader.
- If the paired path succeeds, CoreEG2 allocates a 384-byte complex-display replay record and calls GOP device-protocol slot `+0x40`.
- GOP slot `+0x40` is `GopReplaySavedDisplayConfiguration`.
- If the paired path fails or needs cleanup, CoreEG2 calls the same GOP replay method with `NULL`.

Important correction:

The shared-graphics cache seeding path is real, but it is not the main CoreEG2 5K commit path. The main commit path is:

```text
CoreEg2ComplexDisplayInit
-> validate post-0x4F1 root+sibling connector records
-> build 384-byte complex-display replay record
-> call GopReplaySavedDisplayConfiguration(record)
```

So OCLP/diagnostics probably do not replace CoreEG2. They likely enter a boot path where CoreEG2's normal DXE complex-display path reaches the GOP replay call instead of failing back to generic.

## RE Checkpoint: Complex-Display Replay Record Layout

The CoreEG2 success path builds a zero-initialized 384-byte record for `GopReplaySavedDisplayConfiguration`.

Top-level layout:

```text
record +0x000: replay header, 24 bytes
record +0x018: global/logical timing block, 80 bytes used by GOP
record +0x070: tile0 timing block, 80 bytes used by GOP (88 bytes copied by CoreEG2)
record +0x0C8: tile1 timing block, 80 bytes used by GOP (88 bytes copied by CoreEG2)
record +0x120: endpoint descriptor 0, 48 bytes
record +0x150: endpoint descriptor 1, 48 bytes
```

Header:

```text
+0x00 u32 version        = 0x10000
+0x04 u32 endpoint_count = 2
+0x08 ptr endpoints      = record+0x120
+0x10 ptr global_timing  = record+0x018
```

Global/logical block, selected fields:

```text
+0x00 u32 version = 0x10001
+0x10 u64 normal pixel clock-ish value = 938250000
+0x18 u32 logical width  = 5120
+0x1C u32 logical height = 2880
```

If `dword_E670 & 4` is set, CoreEG2 writes the lower 4096x2304-style logical timing instead of the normal 5120x2880 timing.

Endpoint descriptors:

```text
endpoint +0x00 u32 version       = 0x10000
endpoint +0x04 u32 connector_id  = GOP connector id
endpoint +0x1C u32 viewport_x
endpoint +0x20 u32 viewport_y
endpoint +0x28 ptr tile_timing
```

For the normal two-column 5K path:

```text
endpoint0.connector_id = root connector id
endpoint0.viewport     = 0,0
endpoint0.tile_timing  = record+0x070

endpoint1.connector_id = root connector id + 1
endpoint1.viewport     = tile_width,0
endpoint1.tile_timing  = record+0x0C8
```

GOP consumes this as expected:

- `record+0x04` becomes the replay display count.
- `record+0x10 + 0x18/0x1C` becomes logical framebuffer width/height.
- Each endpoint's `+0x04` connector id is mapped back to a GOP connector object.
- GOP derives the backend mask from that connector object, not directly from the replay record.
- Each endpoint's `+0x1C/+0x20` viewport offset is passed into `GopAdjustViewportForTiledDisplayLayout`.
- Each endpoint's `+0x28` timing pointer is copied into GOP per-display timing/config state.
- The replay consumer does not directly run connector mailbox/DPCD/link-training transactions. It seeds GOP's replayed multi-display state, publishes properties, and programs display-window/viewport registers.

Training happens immediately after replay succeeds:

```text
CoreEg2SetDpTrainingRequestFromTiming(display)
CDLinkTrainDPConnector1 -> root backend +0x28 / GopGraphicsConnectorBackendApplyOutputMask(display+0x160)
CDLinkTrainDPConnector2 -> sibling backend +0x28 / GopGraphicsConnectorBackendApplyOutputMask(display+0x160)
```

The GOP non-NULL backend `+0x28` path parses `display+0x160`, prepares grouped `0xF00` connector-state training, then retries real DP training up to 10 times. A failure on connector 2 logs `LinkTrainFailureInfo` and calls root backend `+0x28` with `NULL`, which is the disable/cleanup path.

Implication for Linux:

This is almost exactly the model Linux should end up presenting once the real secondary route is live: two real endpoints, one logical 5120x2880 tiled display, primary tile at `0,0`, secondary tile at `tile_width,0`. The missing plain-boot step is still below this level: making the sibling endpoint AUX-backed/live enough to validate, train, and describe.

## Non-Goals

Do not pursue these as the main solution unless new evidence changes the model:

- Blindly writing `0x4F1` on arbitrary AUX paths.
- Treating TILE metadata alone as sufficient.
- Creating a fake/emulated secondary sink as the primary path.
- Replacing the two physical tile streams with one synthetic 5120x2880 sink.
- Relying on Apple EFI runtime services after ExitBootServices.
- Shipping a primary-AUX latch write without strong hardware evidence and strict gating.

## Hypotheses

### H1: Primary AUX Can Reach A Shared Panel Latch

In 4K fallback, writing sink-private DPCD `0x4F1 = 1` via the primary `eDP-1` AUX might reach a shared panel/TCON latch and cause the secondary `0x3113` route to become detectable.

Confidence: medium.

Why test it:

- It is small.
- It gives a quick answer in one boot.
- A positive result would simplify cold boot dramatically.
- CoreEG2 has a firmware-side analogue: during `CoreEg2ComplexDisplayInit`, it sends opcode/address `1265` (`0x4F1`) with a one-byte payload value of `1` through the root/primary connector backend, stalls `10 ms`, and only then searches/validates the sibling internal connector.

Why it is suspicious:

- Windows runtime evidence says `0x4F1 = 1` is later written on the real object-specific route, especially secondary `0x3113`, not blindly through primary eDP.
- CoreEG2's primary/root connector backend is not automatically identical to Linux's normal primary eDP AUX path.
- Existing notes classify blind primary-AUX `0x4F1` as a retired/superseded production idea for the Windows-shaped runtime path.

Interpretation:

- If positive: primary AUX can affect shared panel state; design a safer, narrow cold-boot bring-up around this fact.
- If negative: this only disproves the shortcut. It does not disprove the real secondary-route cold-init path.

### H2: Linux Must Construct Or Revive The Real `0x3113` Route

Plain boot must build enough of the Windows-equivalent route to perform real AUX/DPCD operations on `0x3113`.

Required ingredients:

- object id `0x3113`
- DDC3 / selector `0x4871`
- UNIPHY_D transmitter
- usable AUX backend
- DP signal path
- EDID/DisplayID or recoverable tile metadata
- link training and stream enable

Likely Windows-shaped order:

1. Preserve or create the real object-specific AUX route for `0x3113`.
2. Apply the secondary-route ASSR policy: DPCD `0x10A` bit `0`.
3. Publish source-specific DPCD, including known DCE value `0x310 = 04 1D 03`.
4. Run normal link training / stream enable.
5. Write sink-private DPCD `0x4F1 = 1`.
6. Preserve the live secondary route through transient HPD/detect instability.

### H3: Bootloader-Only Solution Requires Apple-Looking Front Door

A bootloader can keep 5K only if firmware sees an Apple diagnostics/recovery-style path first. Direct stock `systemd-boot` probably cannot influence the CoreEG2 decision once firmware has already classified the boot as unsupported/generic.

This remains useful as a compatibility fallback, but it is not the preferred average-user solution.

## Immediate Experiment: `primary_4f1_probe`

This is a debug-only one-boot experiment for H1.

The RE reason for this probe is now stronger than the original Windows-only model:

```text
CoreEg2ComplexDisplayInit
  -> find/create primary/root internal display object
  -> send opcode/address 1265 with payload 1 through root connector backend
  -> Stall(10000)
  -> search connector whose id is root id + 1
  -> fetch/validate Apple internal connector record
  -> on failure, send opcode/address 1265 with payload 0 through the same root backend
```

Linux should still treat this as a probe, not a production implementation, because the UEFI connector backend transport may not be exactly the same as a raw Linux AUX write on `eDP-1`.

### Safety Rules

The probe must be:

- iMac19,1-only.
- behind an explicit debug/module parameter.
- disabled by default.
- limited to the internal Apple 5K panel fingerprint.
- limited to obvious 4K fallback state.
- logging-only unless every gate passes.
- easy to remove after one or two runs.

Do not ship this as a normal path unless runtime evidence proves it is both necessary and safe.

### Entry Conditions

Run the primary-AUX write only when all are true:

- iMac 5K quirk is enabled.
- primary route matches `0x3114` / eDP / DDC4 / UNIPHY_C.
- fallback state is detected:
  - physical size is `600x340`, or
  - primary EDID lacks tiled 5K metadata, or
  - no live secondary sink exists on `0x3113`.
- secondary route, if present, matches the expected identity:
  - object id `0x3113`
  - DDC3 / hw instance `2`
  - UNIPHY_D
  - signal `32` / DP
- no already-live two-tile state exists.

### Probe Sequence

```text
1. Log primary and secondary route identity.
2. Log primary fallback indicators.
3. Log secondary HPD/AUX/local_sink/EDID/TILE state.
4. Try reading primary AUX DPCD 0x4F1.
5. Write primary AUX DPCD 0x4F1 = 1.
6. If the write fails, wait 10 ms and retry once.
7. Wait 10 ms.
8. Log secondary HPD/AUX/local_sink state again.
9. Re-attempt secondary detect using a safe lower DC detect path or delayed work.
10. Wait or schedule a 100 ms follow-up snapshot.
11. Log whether secondary now has AUX, EDID, TILE, and 2560x2880 modes.
```

Avoid recursively invoking full DRM connector detect while detect locks are held. If needed, schedule delayed work to perform the secondary re-detect.

### Expected Log Tags

Use stable strings so one boot can answer the question quickly:

```text
IMAC5K: primary_4f1_probe gate ...
IMAC5K: primary_4f1_probe before primary ...
IMAC5K: primary_4f1_probe before secondary ...
IMAC5K: primary_4f1_probe read 0x4f1 ...
IMAC5K: primary_4f1_probe write 0x4f1 status=...
IMAC5K: primary_4f1_probe retry 0x4f1 status=...
IMAC5K: primary_4f1_probe after10 secondary ...
IMAC5K: primary_4f1_probe redetect secondary ...
IMAC5K: primary_4f1_probe after100 secondary ...
IMAC5K: primary_4f1_probe result=...
```

### Result Table

| Result | Meaning | Next Step |
| --- | --- | --- |
| Primary `0x4F1` write fails | Primary AUX cannot reach that register in fallback state, or timing is wrong | Stop H1 unless another primary-AUX timing lead appears |
| Write succeeds, secondary unchanged | Shared latch shortcut is probably false | Focus on H2 real `0x3113` route construction |
| Write succeeds, secondary AUX wakes but no EDID | Add route/power/link instrumentation around `0x3113` | Test `0x10A`, source-DPCD, training sequence on secondary |
| Write succeeds, secondary EDID/TILE appears | H1 likely true | Design a safer production-grade primary-latch path |
| Secondary reaches `2560x2880` tile state | Major cold-boot breakthrough | Move from probe to minimal gated implementation |

## Main Cold-Boot Track: Real `0x3113` Bring-Up

The long-term Linux path should remain Windows-shaped and route-specific.

### Stage 0: Plain-Boot Instrumentation

Track whether plain boot creates the expected route:

- all `dc_link` object ids
- DDC hw instance
- AUX selector
- HPD source and state
- transmitter
- signal type
- encoder object
- local sink presence
- first DPCD `0x000` read result
- EDID read result
- TILE metadata presence

Success criteria:

- We can say whether plain boot has a real `0x3113` route with the expected DDC3/UNIPHY_D identity.

### Stage 1: H1 One-Boot Probe

Implement `primary_4f1_probe` as above.

Success criteria:

- One dmesg run conclusively says whether primary AUX can wake or expose secondary tile state.

### Stage 2: Secondary Route Construction Or Revival

If H1 fails, focus on `0x3113`.

Work items:

- Find where plain boot accepts or loses class-3 object `0x3113`.
- Check whether AUX factory creation exists for DDC3.
- Check whether base-pin/property bits disable AUX creation.
- Compare Linux route construction against Windows:
  - `0x3113`
  - selector `0x4871`
  - AUX pin index `2`
  - UNIPHY_D
- Ensure Linux can issue DPCD reads/writes on that route before normal connector detect gives up.

Success criteria:

- Plain boot can perform at least one successful DPCD read on the real secondary route.

### Stage 3: Windows-Like Secondary Activation

Once secondary AUX works:

- set `0x10A` bit `0` for exact `0x3113` route
- publish source-DPCD `0x310 = 04 1D 03`
- log `0x111`, `0x316`, `0x4F1`, `0x205`
- train the secondary link
- write `0x4F1 = 1` on secondary route
- retry `0x4F1` once after `10 ms` if needed

Success criteria:

- secondary route has EDID/DisplayID and exposes a real `2560x2880` tile mode.

### Stage 4: Preserve Through KMS Takeover

Avoid losing the real secondary sink during early userspace commits.

Work items:

- preserve only a proven real `0x3113` sink/link
- require prior AUX evidence such as source-DPCD or `0x4F1` success
- suppress only false disconnect teardown, not real disconnects
- reuse cached link settings where Windows would avoid fresh training
- keep HPD-low behavior narrowly gated and heavily logged

Success criteria:

- plymouth/GNOME/KDE takeover does not kill secondary AUX/link state.

### Stage 5: User-Visible 5K Presentation

Expose the correct tiled-monitor shape:

- eDP tile: `2560x2880`, tile location `0,0`
- DP tile: `2560x2880`, tile location `1,0`
- group size `2x1`
- logical desktop `5120x2880`
- no half-screen or independent-monitor failure

Success criteria:

- GNOME and/or KDE presents one stable 5K desktop after direct boot.

## Data To Capture Per Boot

Minimum:

```bash
dmesg > dmesg.txt
drm_info > imac5k_drm_info.txt
modetest -c > imac5k_modetest.txt
cat /sys/class/drm/card*-*/status > imac5k_status.txt
```

Preferred:

- full `capture_live_linux_5k_state.sh`
- debugfs connector state
- DPCD snapshots for `0x10A`, `0x111`, `0x310`, `0x316`, `0x4F1`, `0x205`
- `xrandr --verbose` or compositor equivalent once GUI starts

## Completion Criteria

The cold-boot project is complete when:

- machine boots through direct normal Linux boot path
- no OCLP/probe/diagnostics shim is involved
- kernel detects both internal tile routes
- both routes are real AUX/link-backed objects
- both tiles expose `2560x2880`
- TILE metadata forms one `2x1` group
- GUI reaches stable `5120x2880`
- reboot cycle reproduces the result
- fallback behavior remains safe if the quirk does not match

## Tracking Checklist

- [ ] Add plain-boot route identity instrumentation for all links.
- [ ] Add debug/module parameter for `primary_4f1_probe`.
- [ ] Implement H1 primary `0x4F1` one-boot probe.
- [ ] Capture direct-boot dmesg after H1 probe.
- [ ] Decide H1 result from logs.
- [ ] If H1 positive, design production-grade primary-latch path.
- [ ] If H1 negative, retire primary-AUX shortcut again.
- [ ] Trace where plain boot creates or loses `0x3113`.
- [ ] Make secondary `0x3113` AUX readable in plain boot.
- [ ] Apply exact-route `0x10A` ASSR policy.
- [ ] Publish secondary source-DPCD.
- [ ] Train secondary route.
- [ ] Write secondary `0x4F1 = 1`.
- [ ] Preserve real secondary sink/link through KMS takeover.
- [ ] Verify GNOME/KDE 5K tiled presentation.
- [ ] Reduce debug code to the smallest maintainable quirk.
