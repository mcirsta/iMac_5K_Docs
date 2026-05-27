# iMac19,1 AMD 5K Reverse Engineering Notebook

Current curated snapshot: see `RE_5K_iMac19_1_CURRENT.md` first.

This file is now the raw lab notebook/provenance trail. It contains older hypotheses that were later corrected, including fake/emulated sink paths and early interpretations of `1265`. Use the current snapshot before making kernel or RE decisions.

## 2026-05-15 GOP MCP Checkpoint

Second IDA MCP endpoint `http://127.0.0.1:13338/mcp` is attached to:

```text
RE_Files/FirmwarePE32_All/F0140FC0-EC2A-451E-ADC5-D671AC453A8C.efi.i64
```

Stable GOP findings:

- `GopGraphicsConnectorProtocolTransfer` is the direct connector protocol transfer entry.
- `GopGraphicsConnectorBackendDispatchOpcode` is the lower full-opcode backend used by CoreEG2.
- Backend mode `a4 == 1` selects the extended opcode window and preserves caller opcode `a2` up to `0xFFFFF`.
- Therefore CoreEG2 decimal `1265` is sent as full `0x4F1`, not as low byte `0xF1`.
- `GopWriteExtendedOpcodeWindow` and `GopReadExtendedOpcodeWindow` chunk transfer data through the command window at `a1 + 35392..35396`.
- `GopSubmitConnectorMailboxTransaction` submits/polls the prepared command.
- Normal DP DPCD training is also done through this extended transport (`0x100`, `0x101`, `0x102`, `0x103`, `0x107`, `0x108`, status `0x202`), so Linux should continue treating `0x4F1` as a DPCD/AUX-facing panel operation.
- GOP has an optional `0x10A` write path gated by internal flags, reinforcing the current secondary-route ASSR policy work.
- CoreEG2 publishes Apple shared graphics protocol `63FAECF2-E7EE-4CB9-8A0C-11CE5E89E33C`; GOP locates that protocol through `GopLocateSharedGraphicsProtocol`.
- The shared protocol is a CoreEG2/GOP backchannel. GOP uses it to fetch platform info and, in saved-display-config paths, connector record sets from CoreEG2 display objects.
- CoreEG2 shared slot `+0x20` is now named `CoreEg2SharedFetchConnectorRecordSetForGop`; it returns a copied connector record from `display+0x90` / `display+0x98`.
- GOP connector id `1` maps to backend mask `0x100` / record-cache index `4`; connector id `2` maps to backend mask `0x200` / record-cache index `3`.
- `GopProbeConnectorRecordCachesFromDisplayTable` can pre-seed GOP record caches from CoreEG2 state if an Apple-looking path has already caused CoreEG2 to create richer display objects.
- GOP publishes `APPLE_GOP_DEVICE_PROTOCOL_GUID = 53CFEB0D-9B9E-4D0A-90B6-024A79906A58`; CoreEG2 has a dedicated driver binding for it.
- GOP start order: install GOP device protocol, install `CF611019-...` connector protocols, then install `F977B8AE-...` framebuffer/root-display protocols.
- CoreEG2 runs `CoreEg2ComplexDisplayInit` only after the expected connector and framebuffer/root-display protocol counts are both reached.
- On success, CoreEG2 calls GOP device-protocol slot `+0x40`, now identified as `GopReplaySavedDisplayConfiguration`, with the 384-byte complex-display record. On cleanup/failure it calls the same slot with `NULL`.
- The 384-byte complex-display record layout is now decoded:
  - `record+0x000`: header, version `0x10000`, endpoint count `2`, endpoint pointer `record+0x120`, global-timing pointer `record+0x018`
  - `record+0x018`: global/logical timing, version `0x10001`, normal logical `5120x2880`, pixel-clock-ish value `938250000`
  - `record+0x070` / `record+0x0C8`: per-tile timing blocks
  - `record+0x120` / `record+0x150`: 48-byte endpoint descriptors, with connector id at `+0x04`, viewport at `+0x1C/+0x20`, and tile-timing pointer at `+0x28`
- Normal success shape is endpoint0 on the root connector at `0,0`, endpoint1 on `root connector id + 1` at `tile_width,0`. GOP consumes this as two real endpoints under one logical `5120x2880` tiled display.
- `dword_E670 & 4` selects a lower 4096x2304-style logical timing in the same record; iMac19,1 still decodes as `dword_E670 = 1`, so that lower-mode bit is not expected on the target unless another runtime source overrides the platform DB.
- GOP replay consumer recheck: `GopReplaySavedDisplayConfiguration` seeds GOP multi-display/tile state, publishes properties, and calls `GopAdjustViewportForTiledDisplayLayout`. That helper performs display-window/viewport register programming through the newly named `GopMmioRead32Indexed` / `GopMmioWrite32Indexed`; it does not directly perform connector mailbox or DPCD/link-training work.
- New training-order clarification: immediately after replay succeeds, CoreEG2 runs `CDLinkTrainDPConnector1` and `CDLinkTrainDPConnector2`. Both calls enter graphicsconnector backend slot `+0x28` (`GopGraphicsConnectorBackendApplyOutputMask`) with the same `display+0x160` DP training request. GOP parses that request in `GopPrepareDpTrainingStateForSelectedMask`, then the non-NULL path reaches `GopTrainConnectorMaskWithDpTrainingLoop` and `GopTrainDisplayPortAndPublishDpcd`.

Related CoreEG2 flag source:

- CoreEG2 reads platform display flags into `dword_E670` via provider key GUID `1823A1A6-0EAF-4F3A-93D3-BF61F5EEFA0D`.
- `dword_E670 & 1` gates `CoreEg2ComplexDisplayInit`, which is the paired-display path that sends `0x4F1 = 1` before sibling connector discovery.
- 2026-05-15 recheck: the CoreEG2 provider-table entry for this read is `AC5E4829-A8FD-440B-AF33-9FFE013B12D8`, published by `ApplePlatformInfoDatabaseDxe.efi`. The misleading callback name was corrected in IDA to `CoreEg2RegisterPlatformInfoDatabaseProvider`.
- The raw platform database entry for selector `1823A1A6-...` still decodes as `j139 -> 5`, `j138 -> 1`; the same block maps `j139` to `iMac19,2` and `j138` to `iMac19,1`.
- 2026-05-15 raw-scan recheck: the selector GUID byte pattern is not present in the PE32 module set; it appears in platform-info raw freeform file `5E7BE016-33CF-2D42-8758-C69FA5CDBB2F`, at raw-section offset `0x78`, pointing to value block `0x7A0` / length `0x30`. That block contains the static rows `j139 = 5` and `j138 = 1`.
- So for this target, the strongest evidence is `iMac19,1 -> j138 -> dword_E670 = 1`; the OCLP/plain difference is probably not "bit0 clear vs set".
- The stronger current failure target is the live post-`0x4F1` connector-record validation: the sibling record must report `0x0610`, `0xAE`, and `flags&3 == 2`, then the root record must validate again before CoreEG2 commits the paired display object.

IDA annotations saved in the GOP IDB:

- `sub_1ADF0` -> `GopExecuteConnectorMailboxTransaction`
- `sub_1BA90` -> `GopSubmitConnectorMailboxTransaction`
- `sub_1BFA0` -> `GopWriteExtendedByteOpcode`
- `sub_1BF30` -> `GopReadExtendedByteOpcode`
- `sub_EE10` -> `GopMmioWrite32Indexed`
- `sub_EF20` -> `GopMmioRead32Indexed`
- `sub_F720` -> `GopStallMicroseconds`
- `sub_DD50` -> `GopTrainConnectorMaskWithDpTrainingLoop`
- `sub_E0B0` -> `GopDisableConnectorMaskPath`
- `sub_12DB0` -> `GopSubmitDpTrainingControlPacket4C`
- `sub_CF60` -> `GopReadDpTrainingStatusWindows`
- `sub_E5F0` -> `GopReadExtended2210Bit3`
- `sub_19EC0` -> `GopGetConnectorRecordCacheByIndex`
- `sub_19FA0` -> `GopGetConnectorRecordValidFlagByIndex`
- `sub_1A160` -> `GopGetConnectorMaskPresenceFlag`
- `sub_9C10` -> `GopDeviceProtocolUnsupported`
- `sub_2DF0` -> `GopInstallFramebufferProtocols`
- `sub_2E80` -> `GopInstallFramebufferProtocol`
- `sub_30A0` -> `GopUninstallFramebufferProtocol`
- `UNKNOWN_PROTOCOL_GUID_1` -> `APPLE_GOP_FRAMEBUFFER_PROTOCOL_GUID`
- `UNKNOWN_PROTOCOL_GUID_2` -> `APPLE_AMD_CONNECTOR_PROTOCOL_GUID`
- `UNKNOWN_PROTOCOL_GUID_3` -> `APPLE_GOP_DEVICE_PROTOCOL_GUID`

IDA annotations saved in the CoreEG2 IDB:

- `sub_63C9` -> `CoreEg2SharedRegisterGpuUnsupported`
- `sub_6564` -> `CoreEg2SharedFetchConnectorRecordSetForGop`
- `sub_6679` -> `CoreEg2SharedGetDisplayConfigIsMode129`
- `sub_205C` -> `CoreEg2CopyConnectorRecordSetToDisplayObject`
- `sub_5F41` -> `CoreEg2BindingSupportedGopDeviceProtocol`
- `sub_5F83` -> `CoreEg2BindingStartGopDeviceProtocol`
- `sub_6203` -> `CoreEg2BindingStopGopDeviceProtocol`
- `sub_BFC6` -> `CoreEg2AllocateZeroPool`
- `sub_1025` -> `CoreEg2ZeroMem`
- `sub_4A30` -> `CoreEg2SetDpTrainingRequestFromTiming`
- `sub_47C9` -> `CoreEg2CheckDpTrainingStatus202`
- `sub_48A7` -> `CoreEg2ApplyConnectorMaskAndRetryTraining`
- `sub_4A9A` -> `CoreEg2LinkTrainWithForceRetryLtFallback`
- `sub_4BE5` -> `CoreEg2ApplyDisplayWorkaroundRetry`
- `unk_E0B8` -> `CoreEg2SharedGraphicsProtocolInterface`
- `unk_E340` -> `APPLE_SHARED_GRAPHICS_PROTOCOL_GUID`
- `unk_E350` -> `APPLE_AMD_CONNECTOR_PROTOCOL_GUID`
- `unk_E3C0` -> `APPLE_GOP_FRAMEBUFFER_PROTOCOL_GUID`
- `unk_E410` -> `APPLE_GOP_DEVICE_PROTOCOL_GUID`

## Purpose

This notebook preserves the current reverse-engineering state for enabling 5K output on a 2019 iMac (`iMac19,1`) under Linux with AMDGPU/DC. The working problem statement is:

- Windows Boot Camp reaches the panel's 5K mode through AMD's Windows display stack.
- Linux currently exposes the internal panel as a single 4K eDP display.
- The goal is to identify the missing firmware/topology enablement and then mirror, consume, or work around it in AMDGPU/DC.

This note is meant to be sufficient for another engineer or agent to continue the work without chat history.

## Workspace Roots and Evidence

- Windows package root:
  - `B416406/amdkmdag.sys`
  - `u0418524.inf`
- Linux AMD DRM tree:
  - `AMD_Linux/`
- Runtime captures from the target system:
  - `iMac_info/`

## Target System Identity

Confirmed from the Linux runtime captures:

- Machine:
  - Apple `iMac19,1`
  - Source: `iMac_info/dmesg.txt:36`
- GPU:
  - Polaris10 `1002:67DF / 106B:019D`
  - Source: `iMac_info/dmesg.txt:897`
- Display core:
  - AMD Display Core initialized on `DCE 11.2`
  - Source: `iMac_info/dmesg.txt` around the Display Core initialization lines
- Internal panel EDID:
  - Manufacturer `APP`
  - Product `0xAE25` (`44581`)
  - Source: `iMac_info/imac_edid_decoded.txt`

## Current Linux Runtime State

The current Linux state is consistent with "plain 4K eDP panel, no exposed tiled topology":

- Internal connector is a single connected `eDP` output.
  - Source: `iMac_info/drm_info.txt` connector block
- Preferred mode is `3840x2160@60`.
  - Source: `iMac_info/drm_info.txt`
- The immutable `TILE` property is empty (`blob = 0`).
  - Source: `iMac_info/drm_info.txt`
- `/sys/class/drm/card1-eDP-1/modes` contains `3840x2160`, `3200x1800`, `2560x1440`, etc., but no `5120x2880`.
  - Source: `iMac_info/sys_class.txt`
- The EDID advertises:
  - `3840x2160`
  - `2560x1440`
  - `3200x1800`
  - no DisplayID tile block
  - two Apple CTA vendor-specific blocks with OUI `00-10-FA`
  - Source: `iMac_info/imac_edid_decoded.txt`

This is the key gap to explain: Linux sees a normal 4K-class eDP panel, while Windows Boot Camp reaches a path that can enable a secondary tile and expose 5K.

## AUX / DPCD Probe Result (2026-03-20)

Probe file:

- `iMac_info/imac_aux_probe_20260320_202309.txt`

This probe was intended to answer a very specific question:

- does Linux already expose more than one meaningful AUX/DPCD path for the internal panel?

### Confirmed Results

- Four AUX devices exist:
  - `drm_dp_aux0`
  - `drm_dp_aux1`
  - `drm_dp_aux2`
  - `drm_dp_aux3`
- They map one-to-one to the visible connectors:
  - `drm_dp_aux0 -> card1-eDP-1`
  - `drm_dp_aux1 -> card1-DP-1`
  - `drm_dp_aux2 -> card1-DP-2`
  - `drm_dp_aux3 -> card1-DP-3`
- Only `drm_dp_aux0` returned valid DPCD data.
- `drm_dp_aux1`, `drm_dp_aux2`, and `drm_dp_aux3` correspond to the disconnected external DP connectors and returned no useful DPCD payload in the probe output.

### DPCD Seen on `drm_dp_aux0` / `eDP-1`

From the probe dump:

- DPCD `0x000-0x00f`:
  - `12 14 c4 01 01 00 01 80 02 01 04 01 0f 00 01 00`
- `DP_SUPPORTED_LINK_RATES` (`0x010-0x01f`):
  - all zero
- `DP_MSTM_CAP` (`0x021`):
  - `00`
- `LINK_BW_SET` (`0x100`):
  - `14`
- `LINK_RATE_SET` (`0x115`):
  - `00`

Working interpretation:

- DPCD revision appears to be `0x12` (DP 1.2-class behavior)
- current link bandwidth is `0x14` (HBR2-class)
- no MST capability is exposed
- no eDP 1.4 supported-link-rates table is exposed
- no `LINK_RATE_SET` mode is active

### What This Means

This is a strong result.

- On the current cold Linux boot, there is **not** an already-exposed hidden second AUX path for the internal panel.
- Linux currently sees exactly one meaningful internal AUX/DPCD path, and it looks like a normal single-path eDP sink.
- There is no exposed MST capability on that internal AUX path.
- There is no exposed eDP 1.4 link-rate-set table on that internal AUX path.

This makes the current theory stronger:

- the missing Windows prerequisite state is probably **not** already visible to Linux as a second readable AUX endpoint in this boot configuration
- the missing state is more likely:
  - firmware / preboot dependent
  - hidden behind Apple-specific setup that Linux never receives
  - or only surfaced after a special bring-up sequence that Linux never enters

## OCLP 5K Path and `diags.efi`

This section covers the OCLP side of the investigation, specifically PR `#777` and the Apple diagnostics binary it uses for 5K iMac handling.

### Confirmed OCLP PR `#777` Behavior

The exact PR that introduced the 5K workaround is:

- commit `ca734ff0551234c2fbb20354372f1801580a4bcd`
- subject: `Add inital 5k Patchset`

Confirmed from the patch:

- OCLP identifies 5K-capable iMacs / iMac Pro models as needing special boot handling.
- Its own comments state that if the firmware sees an "unsupported" boot entry anywhere in the boot chain, it forces the internal panel into a 4K-compatible single-stream mode.
- The workaround is to make Apple firmware treat the boot as an Apple diagnostics path instead of a normal third-party boot path.

The PR does this by changing the boot-chain layout:

- sets `LauncherPath` to `\\boot.efi`
- installs a custom `diags.efi`
- moves the real loader into:
  - `System/Library/CoreServices/.diagnostics/Drivers/HardwareDrivers/Product.efi`

This is a critical finding:

- the OCLP 5K fix is not just a normal OpenCore config tweak
- it is a diagnostics boot-path disguise

### `diags_pr777.efi` Binary Used For RE

The exact PR-pinned file analyzed in IDA is:

- `RE_Files/diags_pr777.efi`

The IDB is:

- `RE_Files/diags_pr777.efi.i64`

Basic triage from IDA / MCP:

- PE32+ / UEFI-style binary
- 64-bit
- image size `0x1e000`
- entrypoint `_ModuleEntryPoint`
- approximately 197 functions
- Hex-Rays decompilation works

### Confirmed `diags.efi` Findings

#### High-level identity

This binary identifies itself as an Apple diagnostics / test-agent component, not an OpenCore-specific helper.

Evidence recovered from strings and control flow:

- `Efi Test Agent Version %a`
- `Product ID is %d`
- `Diagnostics`
- `Test Agent is booting to the Recovery OS`
- `CASAttemptLocalBoot`

The module version string is:

- `1.0.30r5`

#### UEFI protocols confirmed in code

Using IDA Python to decode embedded GUIDs:

- `09576E91-6D3F-11D2-8E39-00A0C969723B`
  - `EFI_DEVICE_PATH_PROTOCOL_GUID`
- `5B1B31A1-9562-11D2-8E3F-00A0C969723B`
  - `EFI_LOADED_IMAGE_PROTOCOL_GUID`

This matters because it confirms the binary is walking normal UEFI loaded-image and device-path state as part of the Apple diagnostics boot flow.

#### `InitializeProductDriver` (`sub_3872`)

This function was renamed in the IDB to:

- `InitializeProductDriver`

Confirmed behavior:

- obtains the loaded-image protocol for the current image handle
- obtains the device-path protocol from the loaded image's device handle
- builds a file device path for `\\`
- calls the local file loader
- calls `LoadImage` / `StartImage`
- logs:
  - `InitializeProductDriver Status %r`

Current interpretation:

- this is a strong candidate for the diagnostics "product driver" handoff used by the OCLP `Product.efi` trick
- or at minimum it is consuming a product-driver image/context already prepared by Apple diagnostics boot infrastructure

Important note:

- no literal `Product.efi` string was recovered from the binary during this first pass
- this supports the idea that `Product.efi` is part of the surrounding Apple diagnostics framework / boot policy path, not a normal hardcoded filename inside this binary

#### `EFILoadLocalFile` (`sub_30FB`)

This function was renamed in the IDB to:

- `EFILoadLocalFile`

Confirmed behavior:

- opens a file from either a provided volume or the current image's filesystem
- gets file size / metadata
- reads the file into memory
- returns the loaded buffer and size

This is the generic local-file loader used by both:

- the product-driver init path
- the recovery / BaseSystem boot path

#### `HandleGuiTask` (`sub_189A`)

This function was renamed in the IDB to:

- `HandleGuiTask`

Confirmed behavior:

- dispatches GUI / test-agent tasks
- task `0xD2` logs:
  - `Test Agent is booting to the Recovery OS`
- task `0xD2` then calls the recovery boot path below

#### `CASAttemptLocalBoot` (`sub_55EB`)

This function was renamed in the IDB to:

- `CASAttemptLocalBoot`

Confirmed behavior:

- finalizes GUI state
- finds the recovery filesystem
- loads:
  - `\\BaseSystem.chunklist`
  - `\\BaseSystem.dmg`
- validates the asset
- creates / fills a ramdisk-like backing store
- mounts the image
- calls `LoadImage` / `StartImage` on the mounted Recovery OS image

This is clearly an Apple diagnostics / recovery boot path, not a 5K mode-setting routine.

### What `diags.efi` Has *Not* Shown So Far

During this first pass, no direct 5K-specific logic was found inside `diags_pr777.efi` itself:

- no obvious tiled-display strings
- no obvious EDID parsing strings
- no obvious display-mode-setting strings
- no obvious GPU/display-controller configuration strings

This is an important negative result.

### Working Interpretation

The current best interpretation is:

- OCLP does **not** add its own 5K-enabling display logic in `diags.efi`
- instead, it forces Apple firmware / Apple diagnostics boot policy to classify the boot as a supported Apple diagnostics path
- that likely preserves the panel's dual-stream / 5K-capable preboot state that would otherwise be collapsed to 4K

This aligns well with:

- the OCLP PR comments
- the lack of direct display-mode logic in `diags.efi`
- the Linux AUX probe, which currently shows only a plain single internal eDP path
- the Windows driver findings, where runtime enablement only happens after a special panel/topology state is already present

### Immediate OCLP-side Next Steps

The highest-value next OCLP RE targets are now:

- continue tracing the exact "product driver" handoff in `diags_pr777.efi`
- analyze Apple `boot.efi`
- analyze the Apple diagnostics boot policy / firmware path that decides to preserve or collapse the 5K panel state

The practical goal is no longer "understand all of OCLP", but:

- identify the smallest boot-time condition needed to keep 5K available
- so it can eventually be replaced by:
  - a tiny EFI shim before `systemd-boot`
  - or a Linux-side runtime enablement path if that proves sufficient

## RE Workflow Used So Far

### IDA + MCP Setup

The Windows RE was done in IDA Pro 9.3 with Hex-Rays and `ida-pro-mcp`, using the MCP tools for:

- decompilation
- xref chasing
- call graph inspection
- config string / descriptor table recovery
- function-level tracing from config read to hardware action

### Windows RE Approach

The Windows path was approached in this order:

1. Search for `5K`, `tile`, `MST`, `BootCamp`, and DAL config strings in `amdkmdag.sys`.
2. Identify config seeds in `u0418524.inf`.
3. Resolve string-backed config descriptors and their callsites.
4. Trace Boot Camp policy flow through DAL startup and settings export.
5. Trace the special display bring-up path that eventually enables the secondary tile.
6. Cross-map each confirmed Windows behavior onto the closest Linux AMDGPU/DC hook.

### Linux Crosswalk Approach

The Linux source pass focused on:

- `display/amdgpu_dm/`
- `display/dc/`
- current runtime captures under `iMac_info/`

The goal was to find:

- pipe-split equivalents
- MST / tile detection gates
- panel or DMI quirk insertion points
- eDP supported-link-rate logic
- any Apple-specific EDID or vendor-block handling

## Confirmed Windows Findings

This section contains confirmed findings from the Windows package and IDA analysis. Inference is called out separately later.

### INF Seeds

The INF provides Boot Camp and Apple-related seeds, but does not itself contain the full 5K logic.

- `PP_Apple_Bootcamp_Enable=1`
  - Source: `u0418524.inf` near line 1745
- `KMD_BootCampPlatform=1`
  - Source: `u0418524.inf` near line 1806
- `DalNumOfPathPerDpMstConnector`
  - present in the INF settings list
  - Source: `u0418524.inf` near line 2054

These are seeds into the real behavior inside `amdkmdag.sys`.

### 5K / Tiled Config Descriptors

The driver contains explicit 5K and tiled-display related descriptors and strings, including:

- `DalEnable5kTiledMode`
- `DalEnableTiledDisplay`
- `Dal5K60PipeSplit`
- `DalAllowTiledDisplayGLSync`
- `DalPipeSplitPolicy`
- `DalForceSingleDispPipeSplit`
- `DalNumOfPathPerDpMstConnector`

These were recovered from the descriptor tables in `amdkmdag.sys` and traced into the DAL settings pipeline in IDA.

### Boot Camp Policy and Timing Path

#### `sub_141429530`

This large DAL startup routine creates services, including `TimingService`, then reads `KMD_BootCampPlatform` and immediately calls a TimingService virtual when Boot Camp is enabled.

Confirmed behavior:

- creates `"TimingService"`
- reads `KMD_BootCampPlatform`
- if set, calls the first TimingService virtual with `(5120, 2160)`

Source:

- IDA / Hex-Rays trace of `sub_141429530`
- string reference to `KMD_BootCampPlatform`

#### `sub_141AB7B70`

This is the TimingService virtual reached from the Boot Camp path above.

Confirmed behavior:

- walks a timing/mode list
- compares the first two dwords of an entry to the two call arguments
- removes entries matching the hardcoded pair `(5120, 2160)`

This is important because it shows that Boot Camp mutates timing state rather than simply toggling a boolean.

#### `sub_141AB98B0`

This is the TimingService constructor.

Confirmed behavior:

- reads `KMD_BootCampPlatform`
- caches it in global `dword_144C4E008`

#### `sub_141A376F0`

Confirmed behavior:

- handles a special timing classification path
- always triggers the relevant path for types `5` and `6`
- additionally includes type `14` only when the cached Boot Camp global is set
- sets output flag `0x10` when the path succeeds

#### `sub_141A7C0A0` and `sub_141A7C4A0`

These functions load and export a large DAL settings object.

Confirmed behavior:

- `sub_141A7C0A0` reads `KMD_BootCampPlatform` and sets a persistent byte at offset `+9476` when Boot Camp is enabled
- `sub_141A7C4A0` later exports the settings object into another structure
- during export, if that Boot Camp byte is set, the driver forces the exported pipe-split policy field to `1`
- the corresponding exported boolean for `DalForceSingleDispPipeSplit` is handled separately

This is the strongest direct config effect found so far: Boot Camp overrides the effective pipe-split policy, not just the user registry.

### ModeManager 5K/Tiled Config Consumption

#### `sub_141A3DA00`

This ModeManager-related constructor reads config descriptors tied to tiled/5K behavior.

Confirmed behavior:

- reads `DalEnable5kTiledMode`
- latches it into object state at `a1 + 128`

This is further confirmation that the Windows driver has an explicit 5K tiled feature switch independent of the INF alone.

## Confirmed Secondary-Tile / Lower-Level Firmware Control Path

### `dal3::WindowsDM::EnableSecondaryTileIfRequired` = `sub_141A174B0`

This is the most important high-level function identified so far.

Confirmed behavior:

- inspects the connected pin flags
- looks up a display/path object through `sub_141A33040`
- only proceeds when the detected display object already has type `128`
- if so, calls `sub_141B716F0`

This is critical: the secondary-tile path does not run generically. It only runs after detection already classifies the display/path in a special way.

### `sub_141B716F0`

This is the direct lower-level command dispatch found in the Windows driver.

Confirmed behavior:

- builds opcode `1265`
- sends a 1-byte payload with value `1`
- dispatches through `sub_141EDFC70`
- retries after 10 ms on failure
- optionally waits 100 ms after success

In short:

- Windows **does** send an explicit lower-level command toward the display controller / firmware layer
- the traced command is:
  - opcode: `1265`
  - payload: `1`

This is the clearest concrete answer to the question "does the Windows driver send commands to lower-level firmware?"

### `sub_141B6CA70`

This is the special bring-up path for one of the display connection classes.

Confirmed behavior:

- checks the display/path type
- when the type is `128`, sets two controller flags through virtual calls at offsets `66544` and `66552`
- then performs a special link-training path

### `sub_141B8D860`

This is the link-training function called from the special bring-up path.

Confirmed behavior:

- contains the log string `perform_link_training_with_retries`
- performs repeated link-training attempts
- handles fallback / retry behavior

This is important because Linux DC has a directly analogous conceptual path with the same name.

### `sub_141B6A720`

This is a higher-level bring-up wrapper that routes by display/path type.

Confirmed behavior:

- dispatches type `128` into the special path
- after successful bring-up, if a link-side flag is set, it calls `sub_141B716F0(..., 1)` again

This suggests the secondary-tile command is not only a one-shot setup action from `EnableSecondaryTileIfRequired`; it is also tied to the successful completion of a special training/bring-up path.

## Confirmed Windows Interpretation

The confirmed Windows picture is:

- Boot Camp changes policy and timing behavior.
- The Windows driver has explicit 5K/tiled config descriptors.
- The driver sends an explicit lower-level command (`1265`, payload `1`) to enable the secondary tile.
- However, that lower-level command is only sent after detection already reports a special panel/topology state (type `128`).

This means:

- Boot Camp is not enough by itself to explain 5K.
- The actual secondary-tile path depends on a special display/path classification that already exists by the time the command is sent.

## Confirmed Linux Mapping

### Pipe-Split Equivalents

Linux AMDGPU/DC has a direct equivalent to the Windows pipe-split policy concept.

Confirmed from source:

- `display/dc/dc.h` defines:
  - `enum pipe_split_policy`
  - `MPC_SPLIT_AVOID = 1`
- `display/amdgpu_dm/amdgpu_dm.c` sets:
  - `force_single_disp_pipe_split = false`
  - `pipe_split_policy = MPC_SPLIT_AVOID`
  - under the `DC_DISABLE_PIPE_SPLIT` debug mask
- `display/dc/resource/dcn20/dcn20_resource.c` consumes:
  - `dc->debug.pipe_split_policy`
  - `dc->debug.force_single_disp_pipe_split`

This is the strongest Windows-to-Linux mapping discovered so far:

- Windows Boot Camp forces the exported pipe-split policy to `1`
- Linux defines `MPC_SPLIT_AVOID = 1`

### MST / Tile Discovery Gates

Linux only enters MST topology discovery when DPCD reports MST capability.

Confirmed from source:

- `display/dc/link/protocols/link_dp_capability.c`
  - reads `DP_MSTM_CAP`
  - computes `link->dpcd_caps.is_mst_capable`
- `display/dc/link/link_detection.c`
  - calls `discover_dp_mst_topology()` only when `is_mst_capable`

Current runtime evidence shows Linux does **not** see a tiled panel topology on the target machine:

- `TILE` property is empty
- no `5120x2880` mode
- only one visible internal `eDP` connector

### Quirk Insertion Points

The most relevant Linux insertion points identified so far are:

- DMI-based quirk table:
  - `display/amdgpu_dm/amdgpu_dm_quirks.c`
- panel EDID quirk handling:
  - `display/amdgpu_dm/amdgpu_dm_helpers.c`
- link capability and MST gating:
  - `display/dc/link/protocols/link_dp_capability.c`
  - `display/dc/link/link_detection.c`
- link training:
  - `display/dc/link/protocols/link_dp_training.c`
  - `display/dc/link/link_dpms.c`

### eDP Supported Link Rates

Linux already has explicit eDP supported-link-rate handling over AUX / DPCD.

Confirmed from source:

- `display/dc/link/protocols/link_dp_capability.c`
  - reads `DP_SUPPORTED_LINK_RATES`
  - populates `edp_supported_link_rates[]`
- `display/amdgpu_dm/amdgpu_dm_debugfs.c`
  - exposes `ilr_setting`
  - reads `DP_SUPPORTED_LINK_RATES` through `dm_helpers_dp_read_dpcd()`

This matters because the special Windows path also performs capability / link-setting work that looks like panel-specific link-rate negotiation rather than only mode selection.

## Informed Inferences

This section contains informed inferences, not hard proof.

### Inference: Firmware / Preboot State Is Probably Required

Because the Windows secondary-tile command path only runs after detection already reports a special type `128`, it is likely that:

- firmware or preboot state
- hidden AUX-visible topology
- or Apple-specific capability exposure

exists before the Windows driver sends the final secondary-tile command.

This is consistent with reports that OCLP / OS X-like boot paths can make 5K work on Linux.

### Inference: Linux Is Missing Detection-Time State, Not Just a Modeline

The current Linux runtime state strongly suggests that the missing piece is not simply:

- a 5120x2880 modeline
- or a missing bandwidth calculation

Instead, Linux appears to be missing the prerequisite topology/capability state that would let it enter the equivalent of the Windows special bring-up path.

### Inference: A Linux Fix Probably Needs Two Pieces

The likely Linux fix shape is:

1. A machine/panel-specific quirk that makes DC treat the internal iMac panel path more like the Windows special path.
2. A follow-on training / secondary-tile activation step once that special path is engaged.

## Open Questions

These are still unresolved:

- What exactly populates the Windows special display/path type `128`?
- Is the missing Windows prerequisite state coming from Apple firmware, from a preboot handoff, from hidden AUX endpoints, or from Apple vendor blocks in the panel EDID?
- Does the internal panel actually expose multiple AUX-visible endpoints or DPCD instances?
- If so, are both links already present under certain boot conditions?
- Are the Apple CTA vendor-specific blocks with OUI `00-10-FA` sufficient to derive any of the missing state, or are they only advisory?

## Working Linux Enablement Direction

Current working direction:

- start kernel-first
- instrument first
- add a narrow Apple/iMac quirk rather than a broad global hack
- mirror the Windows-equivalent pipe-split policy
- then determine whether Linux can surface the missing special internal-panel state on its own

If that state never appears under a cold Linux boot, the next branch of work will likely need:

- preboot-assisted state comparison
- or explicit firmware-handoff assumptions

## Next Experiments

### 1. AUX / DPCD Probe on the Linux iMac

The most useful immediate experiment is to check whether multiple AUX endpoints exist and whether they expose sane DPCD state.

This tests the suggestion that the panel may already have multiple AUX-visible paths and that, if those DPCD registers are reachable, it may be possible to confirm hidden topology state before trying any risky link forcing.

What to look for:

- more than one AUX device
- valid DPCD on hidden paths
- nonzero `DP_SUPPORTED_LINK_RATES`
- `DP_MSTM_CAP` values
- `LINK_BW_SET` and `LINK_RATE_SET`

### 2. Compare Cold Linux vs Any Preboot-Assisted Boot

If available later:

- compare AUX/DPCD dumps between
  - a normal Linux boot
  - a preboot-assisted / OCLP-style boot path

The most valuable comparison points are:

- connector count
- AUX device count
- DPCD revision
- supported link rates
- MST capability
- any extra panel-visible path

The 2026-03-20 cold-boot AUX probe now gives a solid baseline for this comparison:

- only one meaningful internal AUX path
- no internal MST capability
- no exposed supported-link-rates table
- no hidden second internal AUX endpoint visible from userspace

### 3. Linux Kernel Quirk Prototype

After the AUX evidence is collected:

- add a narrow quirk for `iMac19,1` + `APP/AE25`
- log detection / DPCD / training decisions
- force the Windows-equivalent pipe-split policy
- do **not** inject a fake 5K mode first
- do **not** globally force MST for all eDP panels

## Appendix A: AUX / DPCD Probe Script

This script is preserved exactly as previously prepared in chat.

```bash
#!/usr/bin/env bash
set -u

OUTFILE="$PWD/imac_aux_probe_$(date +%Y%m%d_%H%M%S).txt"
exec > >(tee -a "$OUTFILE") 2>&1

run() {
  echo "\$ $*"
  "$@"
  rc=$?
  echo "[exit $rc]"
  echo
  return 0
}

run_root() {
  if [ "$(id -u)" -eq 0 ]; then
    run "$@"
  elif command -v sudo >/dev/null 2>&1; then
    echo "\$ sudo $*"
    sudo "$@"
    rc=$?
    echo "[exit $rc]"
    echo
    return 0
  else
    echo "[skip] need root for: $*"
    echo
    return 0
  fi
}

dump_text_file() {
  local f="$1"
  echo "----- $f -----"
  if [ -r "$f" ]; then
    cat "$f"
  else
    echo "[unreadable]"
  fi
  echo
}

dump_aux_region() {
  local dev="$1"
  local start_hex="$2"
  local count_dec="$3"
  local label="$4"
  local start_dec=$((start_hex))

  echo "=== $dev : $label (offset $(printf '0x%03x' "$start_dec"), len $count_dec) ==="
  if [ -e "$dev" ]; then
    if [ "$(id -u)" -eq 0 ]; then
      dd if="$dev" bs=1 skip="$start_dec" count="$count_dec" status=none 2>/dev/null | hexdump -Cv
    elif command -v sudo >/dev/null 2>&1; then
      sudo dd if="$dev" bs=1 skip="$start_dec" count="$count_dec" status=none 2>/dev/null | hexdump -Cv
    else
      echo "[skip] need root or sudo to read $dev"
    fi
  else
    echo "[missing]"
  fi
  echo
}

echo "Report: $OUTFILE"
echo "Started: $(date -Is)"
echo

echo "=== SYSTEM ==="
run uname -a
if [ -r /etc/os-release ]; then
  dump_text_file /etc/os-release
fi
if [ -r /sys/devices/virtual/dmi/id/product_name ]; then
  dump_text_file /sys/devices/virtual/dmi/id/product_name
fi
if [ -r /sys/devices/virtual/dmi/id/product_version ]; then
  dump_text_file /sys/devices/virtual/dmi/id/product_version
fi
if [ -r /sys/devices/virtual/dmi/id/sys_vendor ]; then
  dump_text_file /sys/devices/virtual/dmi/id/sys_vendor
fi

echo "=== DEBUGFS ==="
if ! grep -qE '[[:space:]]/sys/kernel/debug[[:space:]]' /proc/mounts; then
  run_root mount -t debugfs none /sys/kernel/debug
else
  echo "/sys/kernel/debug already mounted"
  echo
fi

echo "=== DRM CONNECTORS ==="
shopt -s nullglob
for d in /sys/class/drm/card*-*; do
  [ -d "$d" ] || continue
  echo "--- $d ---"
  for f in status enabled modes dpms; do
    if [ -e "$d/$f" ]; then
      echo "[$f]"
      cat "$d/$f" 2>/dev/null || echo "[unreadable]"
    fi
  done
  if [ -e "$d/edid" ]; then
    echo "[edid]"
    stat -c 'present, %s bytes' "$d/edid" 2>/dev/null || echo "present"
  else
    echo "[edid]"
    echo "missing"
  fi
  echo
done

echo "=== DEBUGFS CONNECTOR FILES ==="
run find /sys/kernel/debug/dri -maxdepth 2 -type f \( -name ilr_setting -o -name internal_display -o -name odm_combine_segments \) 2>/dev/null

find /sys/kernel/debug/dri -maxdepth 2 -type f \( -name ilr_setting -o -name internal_display -o -name odm_combine_segments \) -print0 2>/dev/null |
while IFS= read -r -d '' f; do
  dump_text_file "$f"
done

echo "=== AUX DEVICES ==="
run ls -l /dev/drm_dp_aux*
run ls -l /sys/class/drm_dp_aux_dev

if [ -d /sys/class/drm_dp_aux_dev ]; then
  for s in /sys/class/drm_dp_aux_dev/*; do
    [ -e "$s" ] || continue
    echo "--- $s ---"
    [ -f "$s/name" ] && dump_text_file "$s/name"
    [ -e "$s/device" ] && run readlink -f "$s/device"
  done
fi

echo "=== AUX DPCD DUMPS ==="
for a in /dev/drm_dp_aux*; do
  [ -e "$a" ] || continue
  dump_aux_region "$a" 0x000 16 "DPCD 0x000-0x00f"
  dump_aux_region "$a" 0x010 16 "DPCD 0x010-0x01f (SUPPORTED_LINK_RATES)"
  dump_aux_region "$a" 0x021 1  "DPCD 0x021 (MST_CAP)"
  dump_aux_region "$a" 0x100 1  "DPCD 0x100 (LINK_BW_SET)"
  dump_aux_region "$a" 0x115 1  "DPCD 0x115 (LINK_RATE_SET)"
done

echo "Finished: $(date -Is)"
echo "Report saved to: $OUTFILE"
```

## Appendix B: How To Read The AUX Probe Results

Use this checklist when reviewing the script output:

- Count visible connectors:
  - is there only one internal `eDP-*`, or more than one candidate path?
- Count AUX devices:
  - one AUX device suggests plain single-path visibility
  - multiple AUX devices are immediately interesting
- For each AUX device:
  - `0x000-0x00f`
    - should look like sane DPCD, not all zeros or obvious garbage
  - `0x010-0x01f`
    - nonzero values suggest supported link-rate reporting
  - `0x021`
    - MST capability bit may be relevant
  - `0x100`
    - current `LINK_BW_SET`
  - `0x115`
    - current `LINK_RATE_SET` for eDP-style link-rate-set operation
- Compare outputs across AUX devices:
  - if one path has sane DPCD and another has a different or richer capability set, that may be the hidden topology clue
- If available later, compare:
  - cold Linux boot
  - preboot-assisted boot

## Bottom Line

The key durable conclusion at this stage is:

- the Windows driver does send a concrete lower-level command toward the display/controller layer (`1265`, payload `1`)
- but it only does so after detection already reports a special panel/topology state
- Linux currently does not expose that state
- so the immediate problem is likely missing detection-time capability/topology exposure, not just a missing 5K modeline

## Update: Apple boot.efi (Monterey Recovery Base System)

### Artifact

- Binary analyzed in IDA with MCP:
  - `C:\proj\reverse\WT6A_INF\RE_Files\boot.efi`
- Source path used to obtain it:
  - `C:\proj\reverse\OpenCore-1.0.7-RELEASE\Utilities\macrecovery\com.apple.recovery.boot\macOS Base System\System\Library\CoreServices\boot.efi`
- Build string found in entry point:
  - `bootbase.efi 540.120.3~19 (Official), built 2022-06-17T21:13:26-0700`

### Confirmed Findings

- `boot.efi` performs real Apple boot-display setup after the handoff:
  - `ConnectGraphicsDevices` at `0x13227`
  - `OpenBootGraphics` at `0x12755`
  - `QueryBootDisplayState` at `0x13A3E`
  - `DrawBootGraphics` at `0x1290A`
  - `FDELoginUIInitializeUsers` at `0x27BF0`
- The entry point `_ModuleEntryPoint` at `0x74B9` calls the graphics chain in this order:
  - `ConnectGraphicsDevices(0, 0)` if graphics are not already connected
  - `OpenBootGraphics()`
  - `QueryBootDisplayState(0, 0)`
  - `DrawBootGraphics(1)`
- `OpenBootGraphics` first tries GOP and then falls back to an older graphics protocol:
  - GOP GUID at `0xAADA0` decodes to `9042A9DE-23DC-4A38-96FB-7ADED080516A`
  - fallback GUID at `0xAAE30` is also graphics-related
- `ConnectGraphicsDevices` logs `Start ConnectAll`, `End ConnectAll`, `Start ConnectDisplay`, and `End ConnectDisplay`, and reads the Apple NVRAM variable `gfx-saved-config-restore-status`.
- 2026-05-20 recheck:
  - `ConnectGraphicsDevices` locates Apple graphics-connect protocol `8ECE08D8-A6D4-430B-A7B0-2DF318E7884A`, calls slot `+0x18` for `ConnectAll` and slot `+0x08` for `ConnectDisplay`, then reads `gfx-saved-config-restore-status`.
  - Binary search across `RE_Files/FirmwarePE32_All` finds this `8ECE08D8-...` graphics-connect GUID in `AppleBds.efi` and `SlingShot.efi`, not in CoreEG2/GOP.
  - `AppleBds.efi` contains strings `START:ConnectGfxDevs`, `START:ConnectController`, `END:ConnectController`, `END:ConnectGfxDevs`, and `CoreEG2`.
  - Local disassembly shows the AppleBds protocol table slot `+0x08` at `AppleBds.efi+0x11159` calls a `ConnectGfxDevs`-looking routine at `+0x712F`, which eventually calls UEFI `BootServices->ConnectController(..., Recursive=TRUE)` for selected graphics handles.
  - Follow-up IDA pass names this table `AppleBdsGraphicsConnectProtocol` at `0x21A30`; slot `+0x08` is `AppleBdsGraphicsConnectDisplaySlot` and slot `+0x18` is `AppleBdsGraphicsConnectAllSlot`.
  - The selected-handle logic enumerates `EFI_PCI_IO_PROTOCOL_GUID`, reads PCI config space through `EFI_PCI_IO.Pci.Read(width=Uint32, offset=0, count=16)`, filters for PCI base class `0x03` display controllers, and prioritizes candidates whose config data contains Apple vendor/subsystem value `0x106B`.
  - Only priority-`1` records enter the logged `START:ConnectController` loop first; after connect, AppleBds enumerates GOP handles and maps their device paths back to the same PCI controller.
  - This makes Apple BDS / boot-manager connection order the best next target if we need to prove the OCLP/plain boot difference.
  - Apple-vs-generic boot classification in `AppleBds.efi` does not appear to be a literal `BootCamp`/`Windows` string check. No explicit `BootCamp`, `Windows`, or `Microsoft` string was found in the AppleBds string table.
  - The selected boot path is resolved through Simple FS by `AppleBdsResolveSimpleFsAndPath` (`0x1B9B2`), then classified by `AppleBdsGetAppleBootVolumeIdentity` (`0x1BBCE`).
  - That classifier first probes root `GetInfo` GUID `900C7693-8C14-58BA-B44E-974515D27C78`, then root `GetInfo` GUID `FA99420C-88F1-11E7-95F6-B8E8562CBAFA`, then falls back to `APPLE_PARTITION_INFO_PROTOCOL_GUID = 68425EE5-1C43-4BAA-84F7-9AA8A4D8E11E`.
  - The partition-info fallback accepts Apple-specific evidence such as `Apple_Boot`, `Apple_HFS`, `Apple_HFSX`, `Apple_Recovery`, or matching Apple partition type GUID encodings.
  - If no Apple identity flag is found, `AppleBdsEvaluateExtendedBootPolicy` (`0x1C153`) forces extended-boot result type `1`, the normal generic EFI path. Current interpretation: BootCamp/systemd-boot/plain EFI are recognized as the absence of Apple-blessed volume metadata, not as a separate named positive mode in AppleBds.
  - `AppleBdsBootManagerLoop` (`0x3FE1`) and `AppleBdsLoadAndStartBootOption` (`0xBA39`) use `APPLE_BOOT_POLICY_PROTOCOL_GUID = 62257758-350C-4D0A-B0BD-F6BE2E1E272C` when an Apple-policy/media-only boot entry needs the actual boot file path synthesized before `LoadImage`.
  - OCLP correction: this Apple volume classifier is not enough to explain the observed OCLP handoff. OCLP/OpenCore may be launched from generic EFI but still change graphics state through OpenCore's UEFI graphics/driver connection machinery, especially `ConnectDrivers` plus `ReconnectGraphicsOnConnect`/GOP provisioning. Keep Apple boot-target classification separate from OCLP's possible graphics reconnect trigger.
  - `gfx-saved-config-restore-status` is only referenced by `ConnectGraphicsDevices` in this IDB, so `boot.efi` appears to consume/report restore status rather than construct the saved display-config blob itself.
  - `QueryBootDisplayState` locates `APPLE_SHARED_GRAPHICS_PROTOCOL_GUID = 63FAECF2-E7EE-4CB9-8A0C-11CE5E89E33C` and calls protocol slot `+0x30`.
  - In CoreEG2, slot `+0x30` is now named `CoreEg2SharedGetBootDisplayState`; it only maps current CoreEG2 display-object state at `display+0x20C` / `+524` into a small boot display-state code.
  - `BootGraphicsOpenAndDrawSplashPreview` is the renamed wrapper that may call `ConnectGraphicsDevices(0, 1)`, `OpenBootGraphics`, `QueryBootDisplayState`, and `DrawBootGraphics`; current evidence says this is splash/UI plumbing.
- `ConsumeBlackModeNVRAM` at `0x126DC` reads `BlackMode` from Apple NVRAM and clears it after use.
- `DrawBootGraphics` reads Apple NVRAM variable `S` and chooses assets like `NetBoot`, `NetBoot2X`, `NetBootBlack`, and `NetBootBlack2X`.
- `FDELoginUIInitializeUsers` contains the `HiDPIScale2x` path and a large amount of login/FDE screen setup.

### Recovery / Boot Policy Findings

- `CheckSupportedBoardIds` at `0x1BD72` is an early platform gate used from `_ModuleEntryPoint`.
  - It parses boot policy/config data and checks the current platform identifier against `SupportedBoardIds`.
  - If this fails, `boot.efi` opens graphics just far enough to show the unsupported-platform message:
    - `This version of Mac OS X is not supported on this platform!`
- `SetRecoveryBootModeVariables` at `0xD34B` sets and clears recovery-related NVRAM variables:
  - `"r"` under Apple namespace
  - `internet-recovery-mode = "RecoveryModeDisk"`
  - `"R"` under the Apple namespace used elsewhere in boot
- `RebootToRecoveryBootMode` at `0x7049` calls `SetRecoveryBootModeVariables()` and then forces a reboot/reset path.
- This means Apple `boot.efi` does contain explicit platform gating and explicit recovery-mode control, not just UI code.

### Important Negative Findings

- I did **not** find direct 5K/tiled-display logic in this Apple `boot.efi` pass.
- Searching for `5K`, `DisplayPort`, `MST`, `eDP`, `AUX`, and similar terms did not produce a display-topology path.
- A 2026-05-20 constant scan did not find a direct `0x4F1` / decimal `1265` reference in `boot.efi`; the `0xF00` hits were unrelated compression/event-code uses, not the GOP grouped internal-display mask.
- The shared-graphics call from `boot.efi` to CoreEG2 slot `+0x30` is reporting existing CoreEG2 state, not performing DP/AUX/tile initialization.
- The `tile` strings present in `boot.efi` are UI carousel/login tiles, not panel tiling or dual-stream display tiling.

### Interpretation

- The current evidence supports this layered model:
  - OCLP PR `#777` gets firmware onto an Apple-approved diagnostics/recovery boot path
  - Apple `boot.efi` then performs normal Apple graphics and recovery boot handling
  - the actual decision that preserves or collapses the special 5K-capable panel state likely happens before, or right at the edge of, this Apple boot path
- Put differently:
  - `boot.efi` is clearly part of the chain
  - but it still does not look like the place where explicit 5K/tiled mode is being directly programmed
- This remains consistent with the Windows finding:
  - Windows `amdkmdag.sys` performs active runtime secondary-tile/controller work
  - but only after the machine/panel has already reached a special topology state

### Practical RE Implication

- The next Apple-side targets are more likely to be:
  - the diagnostics / `Product.efi` handoff details
  - boot-policy / boot-classification behavior around supported Apple boot paths
  - any firmware-facing protocol or Apple-specific display protocol touched before `OpenBootGraphics`
- The current Apple `boot.efi` findings strengthen the case that a future non-OCLP solution may be:
  - a tiny EFI shim that reproduces the accepted Apple boot classification before launching `systemd-boot`
  - plus, if still necessary, a Linux runtime patch that mirrors the Windows secondary-tile enablement

## Update: AppleBootPolicy DXE

### Artifact

- Binary extracted from the iMac19,1 SPI dump and analyzed in IDA with MCP:
  - `C:\proj\reverse\WT6A_INF\RE_Files\FirmwarePE32_Flat\AppleBootPolicy.efi`

### Confirmed Findings

- `AppleBootPolicy.efi` is small (`33` functions in the current IDB) and tightly scoped.
- The real entrypoint body is `InstallAppleBootPolicyProtocols` at `0x422E`.
- On load, it installs two AppleBootPolicy-related protocols:
  - auxiliary protocol GUID `4c8a2451-c207-405b-9694-99ea13251341`
  - main service protocol GUID `62257758-350c-4d0a-b0bd-f6be2e1e272c`
- The auxiliary protocol interface at `0x50D0` has version `0x10000` and two callbacks:
  - `GetBootPolicyMode` at `0x30C7`
  - `SetBootPolicyMode` at `0x30E9`
- The main service table at `0x5030` begins with version `3` and then exposes resolver-style callbacks:
  - `ResolveCanonicalBootPathFallback` at `0x198C`
  - `ResolveBootPathFromHandle` at `0x1AD5`
  - `sub_1C43` at `0x1C43`
- `InstallAppleBootPolicyProtocols` seeds `dword_510C` by inspecting the loaded image device path and then publishes the service table through Boot Services.

### Hardcoded Boot Path Search Order

- `ResolveCanonicalBootPathFallback` probes these canonical Apple / EFI loader paths when richer metadata is not available:
  - `\\System\\Library\\CoreServices\\boot.efi`
  - `\\EFI\\BOOT\\APPLE\\X64\\BOOT.efi`
  - `\\EFI\\BOOT\\BOOTX64.EFI`
  - `\\boot.efi`
- If one of those paths exists, it synthesizes a file device path for it using `BuildFileDevicePathForHandle` at `0x4471`.
- `BuildFileDevicePathForHandle` constructs a `MEDIA_FILEPATH_DP` node for the requested path and appends it to the handle device path when possible.

### Metadata-Guided Resolution

- `ResolveBootPathFromHandle` first tries to resolve a boot path through metadata records using two Apple-specific GUIDs:
  - `900c7693-8c14-58ba-b44e-974515d27c78`
  - `3533cf0d-685f-5ebf-8dc6-7393485bafa2`
- If that richer metadata path fails, it falls back to the canonical path probing above.
- `ResolveMatchingBootRecordPath` at `0x1303` is the larger helper that matches those metadata records and resolves a concrete file path.
- Standard EFI GUIDs confirmed in this module:
  - `EFI_FILE_INFO_GUID = 09576e92-6d3f-11d2-8e39-00a0c969723b`
  - `EFI_SIMPLE_FILE_SYSTEM_PROTOCOL_GUID = 964e5b22-6459-11d2-8e39-00a0c969723b`

### Important Negative Findings

- I did **not** find direct 5K, tiled-display, DisplayPort, AUX, MST, or panel-topology logic in `AppleBootPolicy.efi`.
- The module behaves like a boot-target policy / resolution service, not like a graphics-init or panel-programming module.

### Interpretation

- `AppleBootPolicy.efi` is still highly relevant to the 5K problem because it appears to embody firmware-side boot-path classification:
  - Apple-style `boot.efi`
  - Apple removable-media fallback path
  - generic EFI fallback path
- This strongly supports the idea that Apple firmware distinguishes between accepted boot paths and unsupported ones before the OS starts.
- But `AppleBootPolicy.efi` itself does not look like the place where the panel is directly switched into or preserved in 5K-capable state.

### Practical RE Implication

- `AppleBootPolicy.efi` now looks like the policy side of the chain.
- The most likely next Apple firmware targets for the actual 5K-preservation decision are:
  - `PolicyInitDxe.efi`
  - `AppleGraphicsConsole.efi`
  - possibly `AppleBacklightController.efi`, `MuxGraphicsSwitch.efi`, or `CoffeelakeGraphics.efi`
- In other words:
  - `AppleBootPolicy.efi` helps explain *which boot paths are treated specially*
  - the neighboring graphics/policy DXEs are more likely to explain *how the special display state is preserved or consumed*

## Update: PolicyInitDxe

### Artifact

- Binary extracted from the iMac19,1 SPI dump and analyzed in IDA with MCP:
  - `C:\proj\reverse\WT6A_INF\RE_Files\FirmwarePE32_Flat\PolicyInitDxe.efi`

### Confirmed Findings

- `PolicyInitDxe.efi` is also small (`32` functions in the current IDB) and centered around protocol installation plus state-object initialization.
- The real entrypoint body is `InitializePolicyInitDxe` at `0x2527`.
- Like `AppleBootPolicy.efi`, it installs a versioned getter/setter interface:
  - interface object at `0x4070`
  - getter `GetPolicyInitMode` at `0x12B1`
  - setter `SetPolicyInitMode` at `0x12D3`
- The GUID installed by `InitializePolicyInitDxe` begins with the same auxiliary policy-mode GUID seen in `AppleBootPolicy.efi`:
  - `4c8a2451-c207-405b-9694-99ea13251341`
- `InitializePolicyInitDxe` also:
  - stores the EFI system table / boot services pointers
  - allocates and initializes a primary state buffer at `qword_4448`
  - subscribes to at least one EFI event
  - locates more private protocols
  - publishes several GUID-tagged policy records / objects

### Message-Sink / Event Pattern

- Four helpers all locate the same private protocol GUID at `0x40A8`:
  - `SendTaggedPolicyMessageWithBuffer` at `0x143B`
  - `SendTaggedPolicyMessageFlag` at `0x15A7`
  - `SendTaggedPolicyMessageTriple` at `0x16FA`
  - `SendTaggedPolicyMessageInOut` at `0x186C`
- These helpers package small fixed-format records and send them through that private protocol.
- The tags used in those messages include:
  - `aea6b965-dcf5-4311-b4b8-0f12464494d2` (`0x40D8`)
  - `b5af1d7a-b8cf-4eb3-8925-a820e16b687d` (`0x40E8`)
  - `1810ab4a-2314-4df6-81eb-67c6ec058591` (`0x40F8`)
  - `627ee2da-3bf9-439b-929f-2e0e6e9dba62` (`0x4108`)
- `SyncInitialPolicyState` at `0x1A88` and `PublishPolicyChangeNotification` at `0x1BA0` use those helpers to mirror / publish state changes.

### Internal State / Policy Database Pattern

- `InitializePolicyInitDxe` allocates and initializes state objects and then builds multiple GUID-tagged records under a policy object model.
- It also subscribes callbacks:
  - `MirrorActivePolicyBuffer` at `0x1B45`
  - `PublishPolicyChangeNotification` at `0x1BA0`
  - `sub_1F7A` at `0x1F7A`
- `sub_1F7A` is the most hardware-adjacent path in this module:
  - it is event-driven
  - it queries another private protocol
  - it reads from memory-mapped hardware space
  - it may call `UpdatePolicyBufferFromEvent` at `0x1C2A`
- Even so, the module still looks like policy orchestration and state propagation more than direct panel-mode programming.

### Important Negative Findings

- I did **not** find explicit `5K`, tiled-display, MST, AUX, eDP, or direct panel-topology code in this pass.
- No strong display-specific strings appeared.
- The dominant patterns are:
  - versioned policy-mode interfaces
  - protocol lookup
  - event registration
  - tagged policy-record creation and synchronization

### Interpretation

- `PolicyInitDxe.efi` looks like a bridge layer between boot policy and other firmware consumers.
- It appears to take the current boot/policy mode, instantiate internal policy state, and notify other firmware components through private protocols and events.
- That is a meaningful step closer to the firmware decision chain, but it still does not look like the module that directly preserves or collapses the 5K-capable panel state.

### Practical RE Implication

- `PolicyInitDxe.efi` strengthens the current model:
  - `AppleBootPolicy.efi` classifies / resolves boot paths
  - `PolicyInitDxe.efi` turns policy state into internal objects / notifications
  - the actual display consequence is likely in a neighboring graphics-facing DXE
- The highest-value next firmware target is now:
  - `AppleGraphicsConsole.efi`
- After that:
  - `AppleBacklightController.efi`
  - `MuxGraphicsSwitch.efi`
  - `CoffeelakeGraphics.efi`

## Update: MuxGraphicsSwitch

### Artifact

- Binary extracted from the iMac19,1 SPI dump and analyzed in IDA with MCP:
  - `C:\proj\reverse\WT6A_INF\RE_Files\FirmwarePE32_Flat\MuxGraphicsSwitch.efi`

### Confirmed Findings

- `MuxGraphicsSwitch.efi` is small (`17` functions in the current IDB) but much more relevant than `AppleGraphicsConsole.efi`.
- The main entrypoint is `InitializeMuxGraphicsSwitch` at `0x1730`.
- It includes these firmware interfaces and tables:
  - `EFI_PCI_IO_PROTOCOL_GUID`
  - `EFI_SMM_COMMUNICATION_PROTOCOL_GUID`
  - `EDKII_PI_SMM_COMMUNICATION_REGION_TABLE_GUID`
  - `EFI_SMM_LOCK_BOX_COMMUNICATION_GUID`
  - `EFI_S3_BOOT_SCRIPT_DATA_GUID`
  - `EFI_S3_BOOT_SCRIPT_DATA_BOOT_TIME_GUID`
  - `EFI_S3_BOOT_SCRIPT_TABLE_BASE_GUID`
  - `EFI_S3_BOOT_SCRIPT_SMM_PRIVATE_DATA_GUID`
  - `EFI_DXE_SMM_READY_TO_LOCK_PROTOCOL_GUID`
  - `EFI_SMM_READY_TO_LOCK_PROTOCOL_GUID`

### Why This Matters

- This is the first firmware module in the investigation that clearly crosses into the lower firmware boundary through SMM communication.
- That makes it far more relevant to the “preserve special display state before OS boot” theory than the earlier policy-only modules.

### SMM Communication Path

- `GetSmmCommunicationBuffer` at `0x13DD` locates the `EDKII_PI_SMM_COMMUNICATION_REGION_TABLE_GUID` configuration table and finds an SMM communication region.
- `SendMuxSmmMessageWithBuffer` at `0x1471`:
  - locates `EFI_SMM_COMMUNICATION_PROTOCOL`
  - builds a message buffer headed by `EFI_SMM_LOCK_BOX_COMMUNICATION_GUID`
  - sends a type-`1` message
- `SendMuxSmmMessageFlag` at `0x15DD`:
  - uses the same SMM communication path
  - sends a type-`4` message
- `PublishMuxSmmPrivateData` at `0x2062`:
  - sends a type-`3` SMM message
  - then publishes `EFI_S3_BOOT_SCRIPT_DATA_BOOT_TIME_GUID`
  - and `EFI_S3_BOOT_SCRIPT_SMM_PRIVATE_DATA_GUID`
- `SyncMuxBootScriptState` at `0x1F01` is gated by `EFI_DXE_SMM_READY_TO_LOCK_PROTOCOL_GUID` and sends SMM messages tagged with:
  - `EFI_S3_BOOT_SCRIPT_DATA_GUID`
  - `EFI_S3_BOOT_SCRIPT_TABLE_BASE_GUID`

### Display / Mux-Relevant Behavior

- `AdjustMuxSmmStateForMultiDisplay` at `0x10E0` is the most interesting routine so far:
  - it enumerates all `EFI_PCI_IO_PROTOCOL` handles
  - for each one, it calls the protocol’s `GetLocation`-style method and a PCI config-space read method
  - the code uses the PCI class-code byte (`BYTE3(v9[2])`) and counts devices where the base class is `0x03` (display controller)
  - it also remembers one “fallback” PCI object corresponding to the all-zero location path
  - if it sees at least two display-class devices and has that fallback object, it reads/modifies/writes PCI config offset `0x54` on the fallback object
  - the write specifically clears bits `0x30`
- This is not a proof of 5K enablement by itself, but it is exactly the kind of preboot mux / topology / multi-display adjustment that could affect whether the panel ends up in the right state for later OS bring-up.

### Internal Sequencing / What Runs Before and After

- `InitializeMuxGraphicsSwitch` installs its debug/policy protocol first and caches:
  - `gST`
  - `gBS`
  - HOB / configuration-table derived state
- It then allocates a boot-services-side state buffer at `qword_31D0`.
- Before boot lock, it creates a DXE event for `SyncMuxBootScriptState` and registers it on:
  - `EFI_DXE_SMM_READY_TO_LOCK_PROTOCOL_GUID`
- It immediately signals that event once after registration.
- If `EFI_SMM_BASE2_PROTOCOL` is available, it then gets `gSmst` and allocates a separate SMM-side state buffer at `qword_31C8`.
- In SMM, it registers:
  - `PublishMuxSmmPrivateData` on `EDKII_SMM_EXIT_BOOT_SERVICES_PROTOCOL_GUID`
  - `PublishMuxSmmPrivateData` on `EDKII_SMM_LEGACY_BOOT_PROTOCOL_GUID`
  - `MirrorMuxStateBuffer` on `EFI_SMM_READY_TO_LOCK_PROTOCOL_GUID`
- `MirrorMuxStateBuffer` calls `SyncMuxBootScriptState` if the DXE and SMM state buffers differ, then mirrors `24` bytes of state and marks the SMM copy valid.
- `SyncMuxBootScriptState` is therefore the earlier “ready-to-lock” phase.
- `PublishMuxSmmPrivateData` is the later “exit boot services / legacy boot” phase.

### Stronger Interpretation

- This sequencing makes `MuxGraphicsSwitch.efi` the first module in the chain that clearly:
  - observes multi-display / multi-GPU presence before the OS
  - adjusts low-level PCI state in response
  - persists mux/display-related state through SMM / boot-script infrastructure
- That does **not** prove it is the sole source of iMac 5K behavior.
- But it is now the strongest Apple firmware lead for a preboot prerequisite that Windows later builds on and plain Linux likely misses.

### Interpretation

- `MuxGraphicsSwitch.efi` now looks like a serious candidate for the firmware-side handoff layer relevant to the iMac 5K problem.
- The most important shift is:
  - earlier modules mostly classified boot paths or propagated abstract policy state
  - this one actually packages and transfers graphics/mux-related state into SMM and boot-script structures
- This fits the broader working model:
  - Apple firmware preserves or prepares special display state before the OS
  - Windows `amdkmdag.sys` later performs additional runtime secondary-tile / bring-up work
  - plain Linux likely misses the preboot piece

### Current Limitation

- I still do **not** have direct proof here of:
  - explicit `5K`
  - tiled display
  - MST
  - AUX dual-link logic
- So this is not yet a full explanation of iMac 5K.
- But it is the strongest firmware-side lead so far for the missing preboot handoff.

### Best Follow-On Targets After `MuxGraphicsSwitch`

- Based on the GUIDs and firmware module list, the most likely consumers / adjacent modules on the other side of this path are:
  - `SmmLockBox`
  - `SmmS3SaveState`
  - `PiSmmCommunicationSmm`
- Supporting SMM infrastructure also present in the same firmware image:
  - `PiSmmCore`
  - `PiSmmIpl`
  - `SmmPlatform`
  - `PchInitSmm`
  - `PowerMgmtSmm`
- Practical implication:
  - if `MuxGraphicsSwitch.efi` is the DXE-side producer of the mux/display handoff
  - then `SmmLockBox` / `SmmS3SaveState` / `PiSmmCommunicationSmm` are the best next candidates for the SMM-side consumer / persistence logic

## Firmware RE: `SmmLockBox.efi`

### Artifact

- Binary extracted from the iMac19,1 SPI dump and analyzed in IDA with MCP:
  - `C:\proj\reverse\WT6A_INF\RE_Files\FirmwarePE32_Flat\SmmLockBox.efi`

### Why This Matters

- `MuxGraphicsSwitch.efi` was already sending SMM messages tagged with `EFI_SMM_LOCK_BOX_COMMUNICATION_GUID`.
- `SmmLockBox.efi` is now confirmed as the direct SMM-side receiver for that path.
- This is the first time in the project that the firmware producer/consumer pair has been tied together concretely.

### Confirmed Findings

- `SmmLockBox.efi` installs `EFI_LOCK_BOX_PROTOCOL_GUID`.
- It registers an SMI handler on `EFI_SMM_LOCK_BOX_COMMUNICATION_GUID`.
- It also registers protocol callbacks on:
  - `EFI_SMM_READY_TO_LOCK_PROTOCOL_GUID`
  - `EFI_SMM_END_OF_DXE_PROTOCOL_GUID`
  - `EDKII_END_OF_S3_RESUME_GUID`
  - `EFI_SMM_SX_DISPATCH2_PROTOCOL_GUID`
- The main initializer is `sub_1B3B`.
- The lock-box communication setup routine is `sub_1963`.
- The SMI dispatcher is `sub_184F`.

### SMM State Timeline

- `sub_1B3B` runs at module entry and does the SMM-side setup:
  - locates `EFI_SMM_BASE2_PROTOCOL` and gets `gSmst`
  - locates `EFI_SMM_ACCESS2_PROTOCOL`
  - allocates internal SMRAM-backed state buffers
  - snapshots memory-region metadata used later for range validation
  - publishes a private record under `EFI_SMM_LOCK_BOX_COMMUNICATION_GUID` if one is not already present
- `sub_254B`, registered on `EFI_SMM_END_OF_DXE_PROTOCOL_GUID`, captures and filters the memory map and memory-attributes state that later gate valid communication buffers.
- `gSmst_0`, registered on `EFI_SMM_READY_TO_LOCK_PROTOCOL_GUID`, sets `byte_3260 = 1`.
- `sub_1954`, also registered on `EFI_SMM_READY_TO_LOCK_PROTOCOL_GUID`, sets `byte_3108 = 1`.
- `sub_2213`, registered on `EFI_SMM_END_OF_DXE_PROTOCOL_GUID`, locates `EFI_SMM_SX_DISPATCH2_PROTOCOL_GUID` and registers `sub_2204` for sleep type `3`.
- `sub_2204` sets `byte_3261 = 1`.
- `sub_2297`, registered on `EDKII_END_OF_S3_RESUME_GUID`, clears `byte_3261 = 0`.

### What The Message Opcodes Do

- `sub_184F` is the main dispatcher for the lock-box communication GUID.
- It validates the caller buffer and then dispatches on the first DWORD:
  - opcode `1` -> `sub_1290`
  - opcode `2` -> `sub_1548`
  - opcode `3` -> `sub_177E`
  - opcode `4` -> `sub_1480`
  - opcode `5` -> replay all flagged entries back to their destination pointers
- The best current interpretation is:
  - opcode `1`: create/save a new lock-box entry
  - opcode `2`: update an existing entry, optionally growing backing storage
  - opcode `3`: read/restore an entry
  - opcode `4`: set entry attributes/flags
  - opcode `5`: replay active entries whose replay flag is set

### Lock-Box Entry Semantics

- `sub_1290` copies in a request structure, validates the source range, rejects duplicates, allocates backing storage in SMRAM, copies the payload, stores a 16-byte key, and links the new entry into the lock-box list.
- `sub_1548` finds an existing entry by 16-byte key, validates the source range, optionally reallocates larger backing storage, and copies new payload bytes into the saved buffer.
- `sub_177E` is the public read/restore path and delegates to `sub_22F2`.
- `sub_22F2` supports both:
  - "report required size only"
  - "copy the saved bytes out"
- `sub_1480` updates entry flags but prevents invalid flag combinations.

### Access Control / Validation

- `sub_23AC` is the core range validator used by the dispatcher and the create/update/read helpers.
- It enforces:
  - address-range limits based on the firmware-discovered address width
  - no overlap with forbidden memory-map regions
  - stricter checks after `ReadyToLock`
  - additional filtering based on memory attributes and reserved-region lists
- The read path in `sub_22F2` has a second gate:
  - if an entry has flag bit `2` set
  - and the firmware is in the `ReadyToLock` state
  - and the sleep-state marker is not active
  - then reads are denied

### Cross-Link To `MuxGraphicsSwitch.efi`

- This is the strongest direct bridge so far:
  - `MuxGraphicsSwitch.efi` sends SMM messages using `EFI_SMM_LOCK_BOX_COMMUNICATION_GUID`
  - `SmmLockBox.efi` receives those messages on the exact same GUID
  - the message types used by `MuxGraphicsSwitch` line up with real handlers here
- The current mapping is:
  - `MuxGraphicsSwitch` type `1` -> create/save lock-box data
  - `MuxGraphicsSwitch` type `4` -> set entry flags
  - `MuxGraphicsSwitch` type `3` -> read/restore the saved data later
- That makes `SmmLockBox` a persistence and controlled replay layer for mux/display-related state, not the final display-programming logic by itself.

### Interpretation

- The firmware chain now looks more structured:
  - DXE side (`MuxGraphicsSwitch`) detects multi-display conditions, tweaks PCI state, and packages state into lock-box messages
  - SMM side (`SmmLockBox`) validates, stores, protects, and later replays that state across boot-phase transitions
- This still does not prove that the stored payload is the exact 5K-enabling state.
- But it strongly supports the broader theory that Apple firmware preserves critical preboot display state through a protected handoff layer that plain Linux boot never participates in.

### Best Next Targets After `SmmLockBox`

- `SmmS3SaveState.efi`
  - because `MuxGraphicsSwitch` also uses `EFI_S3_BOOT_SCRIPT_*` GUIDs and `SmmLockBox` now looks like the staging area for that state
- `PiSmmCommunicationSmm.efi`
  - to understand the lower-level transport path, if needed
- Practical reading:
  - `SmmLockBox` looks like the protected mailbox and replay mechanism
  - `SmmS3SaveState` is the strongest candidate for "what consumes the saved boot-script/display payload next"

## Firmware RE: `SmmS3SaveState.efi`

### Artifact

- Binary extracted from the iMac19,1 SPI dump and analyzed in IDA with MCP:
  - `C:\proj\reverse\WT6A_INF\RE_Files\FirmwarePE32_Flat\SmmS3SaveState.efi`

### Why This Matters

- `MuxGraphicsSwitch.efi` already told us that Apple firmware saves state under:
  - `EFI_S3_BOOT_SCRIPT_DATA_GUID`
  - `EFI_S3_BOOT_SCRIPT_TABLE_BASE_GUID`
  - `EFI_S3_BOOT_SCRIPT_DATA_BOOT_TIME_GUID`
  - `EFI_S3_BOOT_SCRIPT_SMM_PRIVATE_DATA_GUID`
- `SmmLockBox.efi` then showed that those GUID-tagged records live in the lock-box communication list.
- `SmmS3SaveState.efi` is the first module that clearly consumes and republishes those exact boot-script records.

### Confirmed Findings

- The main initializer is `InitializeSmmS3SaveState` at `0x255D`.
- It installs `EFI_S3_SMM_SAVE_STATE_PROTOCOL_GUID` via `InstallS3SaveStateProtocol` at `0x1F3F`.
- It uses the same lock-box record list keyed by `EFI_SMM_LOCK_BOX_COMMUNICATION_GUID`.
- It references and actively manipulates all of:
  - `EFI_S3_BOOT_SCRIPT_DATA_GUID`
  - `EFI_S3_BOOT_SCRIPT_DATA_BOOT_TIME_GUID`
  - `EFI_S3_BOOT_SCRIPT_TABLE_BASE_GUID`
  - `EFI_S3_BOOT_SCRIPT_SMM_PRIVATE_DATA_GUID`

### Relationship To `SmmLockBox`

- `GetS3SaveStateLockBoxRecord` at `0x2074` is the same list-head lookup pattern we already saw in `SmmLockBox`.
- `FindS3SaveStateEntryByGuid` at `0x2182` searches that list for a specific GUID-tagged entry.
- `CreateS3SaveStateEntry` at `0x21CE` allocates a new lock-box entry, copies payload bytes into backing storage, and links it into the same lock-box list.
- `CopyS3SaveStateEntryToCaller` at `0x24C7` reads a saved entry back out of the same list, with the same lock/phase checks.
- `UpdateS3SaveStateEntry` at `0x232D` grows and overwrites an existing `EFI_S3_BOOT_SCRIPT_DATA_GUID` entry in place.

### Boot-Script State Flow

- `AllocateBootScriptBytes` at `0x310B` behaves like the shared boot-script buffer allocator used by many callers in the firmware.
- Before the ready-to-lock transition, it allocates or grows the working boot-script buffer in pages.
- `FinalizeBootScriptTerminator` at `0x2EB6` appends the `0xFF 0x03` boot-script terminator and updates the boot-script length.
- `CaptureBootScriptReadyToLockState` at `0x2F27`:
  - finalizes the current boot-script buffer
  - saves the whole `EFI_S3_BOOT_SCRIPT_DATA_GUID` payload into the lock-box list
  - saves `EFI_S3_BOOT_SCRIPT_TABLE_BASE_GUID`
  - marks both entries replayable / resizable
- `PublishBootScriptBootTimeState` at `0x306E`:
  - snapshots the current `EFI_S3_BOOT_SCRIPT_DATA_GUID`
  - creates `EFI_S3_BOOT_SCRIPT_DATA_BOOT_TIME_GUID`
  - creates `EFI_S3_BOOT_SCRIPT_SMM_PRIVATE_DATA_GUID`
  - marks the private-data entry replayable
- `MirrorBootScriptStateToSmmCopy` at `0x3013` mirrors the DXE working copy into the SMM working copy around the `ReadyToLock` handoff.

### Phase / Event Timeline

- `InitializeSmmS3SaveState` registers:
  - `RegisterS3SaveStateSleepCallback` on `EFI_SMM_END_OF_DXE_PROTOCOL_GUID`
  - a resume clearer on `EDKII_END_OF_S3_RESUME_GUID`
  - a DXE event callback on `EFI_DXE_SMM_READY_TO_LOCK_PROTOCOL_GUID`
  - SMM callbacks on:
    - `EDKII_SMM_EXIT_BOOT_SERVICES_PROTOCOL_GUID`
    - `EDKII_SMM_LEGACY_BOOT_PROTOCOL_GUID`
    - `EFI_SMM_READY_TO_LOCK_PROTOCOL_GUID`
- `RegisterS3SaveStateSleepCallback` installs `MarkS3SaveStateSleepActive` for sleep type `3`.
- `ClearS3SaveStateSleepOnResume` clears that state after resume.
- The access checks in `CopyS3SaveStateEntryToCaller` mirror the earlier lock-box gating:
  - once the protocol is in the locked state
  - and if the entry is flagged specially
  - reads are only allowed in the expected sleep/phase window

### Strongest Interpretation

- `SmmLockBox` is the protected mailbox / persistence layer.
- `SmmS3SaveState` is the layer that turns that stored data into real S3 boot-script state and boot-time snapshots.
- The important implication is that the mux/display-related state saved by `MuxGraphicsSwitch` is not just stashed and forgotten.
- It enters the same boot-script pipeline that other firmware modules use to preserve hardware state across boot transitions.

### Relevance To The iMac 5K Problem

- This is still not a direct "5K enable" function.
- But it is very relevant because it gives the missing structure to the preboot theory:
  - `MuxGraphicsSwitch` detects multi-display state and tweaks PCI state
  - `SmmLockBox` stores that state under protected GUID-tagged records
  - `SmmS3SaveState` republishes those records into the firmware boot-script pipeline
- That is a strong fit for the idea that Apple-approved boot paths preserve a special internal display/mux state before the OS, and that plain Linux never starts from the same conditions.

### Current Limitation

- I still do not have proof that the specific `EFI_S3_BOOT_SCRIPT_*` payload in this module contains the exact panel/tile configuration needed for 5K.
- I also still do not have proof that the Windows AMD driver talks directly to this Apple firmware path.
- The current best model remains:
  - Apple firmware preserves or prepares the right preboot state
  - the Windows AMD driver then performs additional runtime bring-up on top of that

### Best Next Target

- `PiSmmCommunicationSmm.efi` is still useful if transport details become necessary.
- But from a 5K perspective, the highest-value next RE is likely not generic transport.
- It is whichever graphics / mux / Apple-private module actually writes the critical display-specific content into the boot-script buffer that `SmmS3SaveState` later captures and republishes.

## Firmware-Wide Triage Pass

### Why I Added This

- At this point, the manual module-by-module hunt had already paid off:
  - `MuxGraphicsSwitch.efi` looked like the DXE-side graphics/mux state producer
  - `SmmLockBox.efi` looked like the protected state store / replay layer
  - `SmmS3SaveState.efi` looked like the boot-script consumer / publisher
- But the remaining question was no longer "does Apple firmware preserve state at all?"
- It was "which other modules sit next to that path and are most likely to seed the display-specific content relevant to 5K?"
- To widen the search cleanly, I added a full-dump triage script instead of relying on ad hoc manual string checks.

### Script

- Python helper added locally:
  - `C:\proj\reverse\WT6A_INF\scripts\firmware_module_triage.py`

### What The Script Does

- Walks the extracted SPI dump tree under:
  - `C:\proj\reverse\WT6A_INF\RE_Files\imac19_1_spi_1.bin.dump\3 BIOS region`
- Finds every UEFI file with:
  - `PE32 image section`
  - or `TE image section`
- Copies those images into a flat working directory:
  - `C:\proj\reverse\WT6A_INF\RE_Files\FirmwarePE32_All`
- Extracts ASCII + UTF-16LE strings from each image
- Scores modules against the problem-specific pattern buckets:
  - display / graphics / panel / backlight
  - tile / MST / AUX / DPCD / DP / eDP
  - lock-box / boot-script / S3 / ready-to-lock
  - boot / diagnostics / recovery
  - Apple / platform / policy / mux
- Emits a ranked JSON report:
  - `C:\proj\reverse\WT6A_INF\RE_Files\firmware_module_triage.json`

### Run Result

- Scanned modules: `246`
- Flattened images: `246`

### Top Ranked Candidates

- `MuxGraphicsSwitch` score `47`
- `CoffeelakeGraphics` score `33`
- `ApplePlatformSecurityPolicy` score `32`
- `AppleBootPolicy` score `31`
- `AcpiPlatform` score `30`
- `SmmLockBox` score `29`
- `AppleBacklightController` score `28`
- `SmmS3SaveState` score `28`
- `AppleGraphicsConsole` score `27`
- `AppleDmgBootDxe` score `26`
- `PlatformInit` score `24`
- `SmmPlatform` score `24`
- `AppleBootUI` score `23`
- `CpuS3DataDxe` score `22`
- `S3SaveStateDxe` score `22`

### Important Interpretation

- The scoring is heuristic and intentionally broad.
- It does not mean a high score is automatically a 5K-enablement module.
- For example:
  - `CoffeelakeGraphics` scores well because it contains generic graphics / panel / platform strings
  - but earlier RE already suggested it is more likely Intel iGPU bring-up than the direct AMD/iMac 5K path
- The value of the triage pass is not "this gives the answer automatically."
- The value is that it shows which *new* modules are most adjacent to the already-proven firmware handoff path.

### New High-Value Targets From The Triage Pass

- `AcpiPlatform`
  - especially interesting because its extracted strings include:
    - `ICMDisabledInWindows`
    - `InstallWindowsUEFI`
    - Apple `AAPL,current-*` properties
    - several GPU / PEG / IGPU object names
  - that makes it look much more like a platform bridge than generic ACPI boilerplate
- `SmmPlatform`
  - high overlap with the same S3 / phase-transition area as the lock-box and boot-script chain
- `PlatformInit`
  - may be a bridge between Apple platform state and later graphics/policy consumers
- `S3SaveStateDxe`
  - likely the DXE counterpart to the `SmmS3SaveState` path

### Current Best Reading

- The firmware side is no longer just a vague theory.
- There is now a concrete, multi-stage Apple preboot handoff pipeline:
  - graphics / mux detection and adjustment (`MuxGraphicsSwitch`)
  - protected persistence and replay (`SmmLockBox`)
  - boot-script / boot-time publication (`SmmS3SaveState`)
- The next most promising untapped modules are the ones that sit near that pipeline while also referencing Apple platform or Windows-install state.
- Right now, `AcpiPlatform` is the strongest new target because it is the first broad-scan hit that mentions both:
  - Apple platform / GPU object paths
  - and Windows-specific platform state

### Practical Next Step

- Continue RE on:
  - `AcpiPlatform.efi`
- Then, depending on what it shows:
  - `SmmPlatform.efi`
  - `PlatformInit.efi`
  - `S3SaveStateDxe.efi`

## Firmware RE: `AcpiPlatform.efi`

### Artifact

- Binary analyzed in IDA with MCP and efiXplorer:
  - `C:\proj\reverse\WT6A_INF\RE_Files\FirmwarePE32_All\AcpiPlatform.efi`

### Why This Module Matters

- The broad triage pass flagged this as unusually relevant because it mixes:
  - Apple platform / GPU object names
  - Windows-specific variable names
  - ACPI/NVS publication
  - `EndOfDxe`
  - Apple SMC access
- That combination makes it much more interesting than generic ACPI boilerplate.

### Key GUIDs Observed

- `EFI_END_OF_DXE_EVENT_GROUP_GUID`
- `S3_MEMORY_VARIABLE_GUID`
- `EFI_GLOBAL_NVS_AREA_PROTOCOL_GUID`
- `PCH_NVS_AREA_PROTOCOL_GUID`
- `EDKII_VARIABLE_LOCK_PROTOCOL_GUID`
- `APPLE_SMC_IO_PROTOCOL_GUID`
- `EFI_ACPI_SDT_PROTOCOL_GUID`
- `EFI_ACPI_TABLE_PROTOCOL_GUID`
- `EFI_PCI_ROOT_BRIDGE_IO_PROTOCOL_GUID`

### Key Strings Observed

- `AAPL,current-available`
- `AAPL,current-extra`
- `AAPL,current-extra-in-sleep`
- `AAPL,max-port-current-in-sleep`
- `UsbMux`
- `PEG0GFX0`
- `SsdtIGPU`
- `ICMDisabledInWindows`
- `EnableUsbDebugInWindows`
- `InstallWindowsUEFI`
- `S3MemoryVariable`

### Renamed Functions

- `InitializeAcpiPlatform` at `0x19FD`
- `SaveAndLockS3MemoryVariable` at `0x16A4`
- `PatchAndReinstallMcfgOnNotify` at `0x1323`

### Confirmed Findings

- `InitializeAcpiPlatform` is the main entry point for the module.
- It installs a debug-mask protocol and then allocates and publishes a global NVS buffer of size `389` bytes through `EFI_GLOBAL_NVS_AREA_PROTOCOL_GUID`.
- It enumerates ACPI table storage entries and rewrites multiple tables before reinstalling them.
- It reads Windows-related runtime variables:
  - `EnableUsbDebugInWindows`
  - `InstallWindowsUEFI`
  - and another Windows variable strongly matching `ICMDisabledInWindows`
- It locates `APPLE_SMC_IO_PROTOCOL_GUID` and performs Apple SMC transactions before registering its late callbacks.
- It registers:
  - an `EndOfDxe` callback (`SaveAndLockS3MemoryVariable`)
  - a protocol-notify path (`PatchAndReinstallMcfgOnNotify`)

### ACPI / Platform Patching Behavior

- The string-heavy ACPI/GPU path names all funnel into `InitializeAcpiPlatform`.
- The module walks ACPI storage tables and applies object-specific gating / patching for names including:
  - `UsbMux`
  - `Xhci`
  - `IGNoHda`
  - `IGHda`
  - `TbtOnPCH`
  - `TbtPEG01`
  - `TbtPEG1 `
  - `TbtPEG12`
  - `PEG0SSD0`
  - `PEG1SSD0`
  - `PEG2SSD0`
  - `PEG0GFX0`
  - `SsdtIGPU`
- It also patches ACPI property payloads carrying:
  - `AAPL,current-available`
  - `AAPL,current-extra`
  - `AAPL,current-extra-in-sleep`
  - `AAPL,max-port-current-in-sleep`
- Those current-limit values are fetched from an Apple-private protocol and then injected into the ACPI payload.

### Windows-Specific Behavior

- `InstallWindowsUEFI` is actively read and written, not just tested:
  - if the variable contains `2`, the module rewrites it to `3`
  - otherwise it rewrites it with different attributes
  - if the post-read state is not `3`, it clears an internal enable byte in the global NVS area
- `EnableUsbDebugInWindows` also sets a bit in the same NVS/status byte.
- A second Windows-specific variable influences later ACPI/GPU patching and matches the earlier extracted string `ICMDisabledInWindows`.

### S3 / Late-Phase Behavior

- `SaveAndLockS3MemoryVariable` runs from the `EFI_END_OF_DXE_EVENT_GROUP_GUID` callback path.
- It computes a memory range summary from the HOB list, stores it into the `S3MemoryVariable` NVRAM variable under `S3_MEMORY_VARIABLE_GUID`, and then locks that variable using `EDKII_VARIABLE_LOCK_PROTOCOL_GUID`.
- This places `AcpiPlatform` in the same late-phase / preserved-state neighborhood as the earlier:
  - `MuxGraphicsSwitch`
  - `SmmLockBox`
  - `SmmS3SaveState`

### MCFG / PCI Topology Behavior

- `PatchAndReinstallMcfgOnNotify` locates:
  - `EFI_ACPI_SDT_PROTOCOL_GUID`
  - `EFI_ACPI_TABLE_PROTOCOL_GUID`
  - `EFI_PCI_ROOT_BRIDGE_IO_PROTOCOL_GUID`
- It searches ACPI tables for `MCFG`, clones the table, probes PCI topology via root-bridge I/O, patches the cloned copy, and reinstalls it.
- This is another sign that `AcpiPlatform` is responsible for publishing Apple-specific hardware/platform view to later consumers, not just static ACPI data.

### Relevance To The 5K Investigation

- This module still does **not** expose direct tiled-display or 5K panel programming logic.
- But it is very relevant because it shows Apple firmware doing Windows-aware, platform-aware, and GPU-path-aware staging before the OS boots.
- In other words:
  - it helps explain how the Apple-approved Windows boot path can carry extra platform state into the OS environment
  - even if the final 5K-capable panel state is still produced elsewhere

### Current Best Interpretation

- `AcpiPlatform` looks like a firmware-side platform-state publisher.
- It is not the final 5K switch.
- But it is one of the clearest pieces of evidence so far that Apple firmware actively reshapes the machine description for Windows-era consumers:
  - ACPI objects
  - global NVS contents
  - S3-resume variable state
  - Apple SMC-derived data

### Best Next Targets After `AcpiPlatform`

- `ApplePlatformSecurityPolicy.efi`
  - best choice if the goal is to understand which boot paths are treated as Apple-approved / Windows-approved
- `S3SaveStateDxe.efi`
  - best choice if the goal is to connect the DXE-side platform state to the earlier SMM boot-script persistence path

## Firmware RE: `ApplePlatformSecurityPolicy.efi`

### Artifact

- Binary analyzed in IDA with MCP:
  - `C:\proj\reverse\WT6A_INF\RE_Files\FirmwarePE32_All\ApplePlatformSecurityPolicy.efi`

### High-Level Shape

- This module is tiny:
  - 4 total functions
  - 1 large initializer
  - essentially no strings
- That makes it much more of a protocol / policy wiring module than a feature-rich logic module.

### Renamed Function

- `InitializeApplePlatformSecurityPolicy` at `0x10A5`

### Confirmed Findings

- The module installs `EFI_DEBUG_MASK_PROTOCOL_GUID`.
- It also publishes a small Apple-private policy protocol that exposes:
  - a flags getter
  - a record getter
- The exported record getter returns entries from an internal table:
  - count in `dword_23E0`
  - base pointer in `qword_23D8`
  - entry size `0x110`
- The initializer seeds either one or two such records depending on a hardware-dependent check around physical memory / ROM state.

### Private GUIDs Decoded

- Custom protocol / data GUIDs recovered from the data section:
  - `CC34B651-889B-5050-9276-8D0EB6F70BCC`
  - `E4518E76-19D8-4475-9094-73BDABDC3B0C`
  - `75FAB4B4-6AC1-429A-A000-6B0B95E71CA1`
  - `AC5E4829-A8FD-440B-AF33-9FFE013B12D8`
  - `B2CB10B1-714A-4E0C-9ED3-35688B2C99F0`
  - `64ED8B7E-B1D0-4E9E-9D2A-696718CB8347`

### What Those GUIDs Appear To Be Doing

- `E4518E76-19D8-4475-9094-73BDABDC3B0C`
  - looks like the policy protocol this module exports
  - it is located at startup to detect whether the module is already active
  - consumers found in:
    - `AppleCrypto.efi`
    - `PciBusDxe.efi`
- `AC5E4829-A8FD-440B-AF33-9FFE013B12D8`
  - looks like a broader Apple platform / property provider protocol
  - it is heavily shared across the firmware, including:
    - `AcpiPlatform.efi`
    - `AppleBacklightController.efi`
    - `ApplePlatformInfoDatabaseDxe.efi`
    - `PolicyInitDxe.efi`
    - `TamperResistantBoot.efi`
    - `PciBusDxe.efi`
    - many other platform drivers
- `75FAB4B4-6AC1-429A-A000-6B0B95E71CA1`
  - looks like a smaller security / tamper / password-related protocol
  - shared with:
    - `FirmwarePassword.efi`
    - `PasswordUI.efi`
    - `TamperResistantBoot.efi`
- `CC34B651-889B-5050-9276-8D0EB6F70BCC`
  - appears in:
    - `AppleBds.efi`
    - `BootPicker.efi`
    - `PciBusDxe.efi`
    - `ApplePlatformSecurityPolicy.efi`
  - likely another boot/policy-related private protocol or notify GUID

### Internal Behavior

- The module parses a HOB record and copies a 32-bit value from it into `dword_23FC`.
- It exposes getter/setter helpers for that same 32-bit state through the installed protocol object.
- It queries the broad platform provider protocol (`AC5E...`) twice:
  - one query keyed by `64ED8B7E-B1D0-4E9E-9D2A-696718CB8347`
  - repeated per-record queries keyed by `B2CB10B1-714A-4E0C-9ED3-35688B2C99F0`
- It also registers an event / notify callback on the smaller security-related protocol (`75FAB4B4-...`).
- That callback reads two bytes through a method at offset `+0x10` of the located protocol object and conditionally sets policy bit `0x4`.
- A timer/event callback later clears that bit again.

### Strongest Interpretation

- `ApplePlatformSecurityPolicy.efi` does not look like the place where Apple directly decides "5K vs 4K" or directly classifies diagnostics vs normal boot.
- It does look like a compact policy bridge:
  - input from Apple-private platform data
  - input from tamper / password / secure-boot-adjacent state
  - output as a small queryable policy protocol for other firmware modules
- In other words, it is a policy publisher, not the original source of the boot-path decision.

### Why This Still Matters

- It confirms the Apple firmware environment is not a single monolithic "boot approval" check.
- Instead it is already split into:
  - platform data providers
  - security / tamper inputs
  - policy publication
- That makes the firmware side look structurally similar to what we already saw on the display side:
  - one module gathers state
  - another module republishes it in a form used by later consumers

### Best Next Targets After `ApplePlatformSecurityPolicy`

- `TamperResistantBoot.efi`
  - best next choice if the goal is to understand the Apple-approved / security-approved boot path and the smaller security protocol `75FAB4B4-...`
- `ApplePlatformInfoDatabaseDxe.efi`
  - best next choice if the goal is to identify the broad platform-data provider `AC5E4829-...` that both `AcpiPlatform` and `ApplePlatformSecurityPolicy` depend on

## Firmware RE: `TamperResistantBoot.efi`

### Artifact

- Binary analyzed in IDA with MCP:
  - `C:\proj\reverse\WT6A_INF\RE_Files\FirmwarePE32_All\TamperResistantBoot.efi`

### Renamed Functions

- `InitializeTamperResistantBoot` at `0x3E35`
- `DispatchTamperBootCommand` at `0x1458`
- `ReadVariableAlloc` at `0x1FF2`
- `LoadTamperRecordCache` at `0x292F`
- `ValidateCommandAgainstTamperCache` at `0x21D0`
- `QueryPlatformProviderBlob` at `0x2670`
- `IsTamperFlagFSet` at `0x13EB`

### Private GUIDs Recovered

- `F68DA75E-1B55-4E70-B41B-A7B7A5B758EA`
  - primary vendor GUID used for the module's private NVRAM state
- `AC5E4829-A8FD-440B-AF33-9FFE013B12D8`
  - same broad Apple platform/provider protocol already seen from:
    - `AcpiPlatform`
    - `ApplePlatformSecurityPolicy`
    - many other Apple platform modules
- `75FAB4B4-6AC1-429A-A000-6B0B95E71CA1`
  - same smaller security/tamper protocol family already seen in:
    - `ApplePlatformSecurityPolicy`
    - `FirmwarePassword`
    - `PasswordUI`
- `62BF9B1C-8568-48EE-85DC-DD3057660863`
  - vendor GUID for `BootFeatureUsage`
- `5D62B28D-6ED2-40B4-A560-6CD79B93D366`
  - second private vendor GUID used for pending command variables
  - only shared with `PasswordUI.efi`

### Strongest Finding

- `TamperResistantBoot` is not just a passive secure-boot flag checker.
- It is a real command / mailbox engine for tamper- and password-related boot state.
- The main exported security protocol `75FAB4B4-...` is small.
- But behind it, the module:
  - reads queued commands from private NVRAM variables
  - mutates persistent boot/tamper records
  - updates feature bits
  - and republishes the resulting state back into NVRAM

### Module Entry Behavior

- `InitializeTamperResistantBoot` installs `EFI_DEBUG_MASK_PROTOCOL_GUID`.
- It also installs protocol `75FAB4B4-6AC1-429A-A000-6B0B95E71CA1`.
- Before publishing that protocol, it performs an initialization / migration sequence:
  - reads a versioned state variable under `F68DA75E-...`
  - if missing, clears a set of boot-related variables and seeds version state
  - loads a cached tamper record list
  - processes one direct queued command plus a sequence of indexed queued commands under `5D62B28D-...`
  - then clears those queued command variables

### NVRAM / Variable Handling

- `ReadVariableAlloc` is a generic helper that does:
  - `GetVariable` size query
  - pool allocation
  - second `GetVariable`
- `LoadTamperRecordCache` reads variable `C` under `F68DA75E-...` and caches it as:
  - `qword_8158`
  - `dword_8150 = size / 0x45`
- `IsTamperFlagFSet` reads one-byte variable `F` under `F68DA75E-...`.
- `DispatchTamperBootCommand` reads and writes:
  - `C`
  - `B`
  - `D`
  - `T`
  - `F`
  - `BootOrder`
  - `BootNext`
  - `ResetNVRAM`
  - `BootFeatureUsage`

### Command Dispatcher Behavior

- `DispatchTamperBootCommand` switches on a command ID in the first DWORD of the message.
- Observed command families include:
  - creating / seeding a new tamper boot record set
  - validating a candidate record against the cached set
  - merging and rewriting the cached record list
  - updating `BootFeatureUsage`
  - writing per-record subvariables back to NVRAM
- The persistent record format is fixed-size:
  - `0x45` bytes per entry

### Relationship To Password / UI Flow

- The queued-command vendor GUID `5D62B28D-...` is shared only with `PasswordUI.efi`.
- That is the strongest clue in this module:
  - the command stream this module consumes is likely produced by the password / UI side
  - not by the generic graphics or mux path

### Relationship To Earlier Policy Modules

- `TamperResistantBoot` exports the same `75FAB4B4-...` protocol family that `ApplePlatformSecurityPolicy` listens to.
- That means the earlier policy module is not the root source of the security decision.
- Instead:
  - `TamperResistantBoot` appears to own or enforce the underlying tamper/password boot state
  - `ApplePlatformSecurityPolicy` consumes a smaller distilled view of that state

### Relationship To The Broader Apple Platform Provider

- `QueryPlatformProviderBlob` locates `AC5E4829-A8FD-440B-AF33-9FFE013B12D8`.
- It then queries it with key `B2CB10B1-714A-4E0C-9ED3-35688B2C99F0`.
- So `TamperResistantBoot` also depends on the same broad Apple platform/provider protocol as:
  - `AcpiPlatform`
  - `ApplePlatformSecurityPolicy`

### Relevance To The 5K Investigation

- This module still does not look like the direct source of 5K display state.
- But it is important because it clarifies the firmware security side:
  - the Apple-approved boot path is influenced by a real tamper/password state machine
  - that state is fed into `ApplePlatformSecurityPolicy`
  - which in turn participates in broader boot policy publication
- So the "approved path" story is now much less abstract:
  - there is a concrete tamper/password command engine behind it

### Current Best Interpretation

- `TamperResistantBoot` is primarily a tamper/password boot-state engine, not a graphics engine.
- It is likely adjacent to the Apple-approved boot-path decision.
- It is probably not the place where the 5K panel is directly preserved.
- The strongest remaining unexplained layer is now the broad Apple platform/provider protocol:
  - `AC5E4829-A8FD-440B-AF33-9FFE013B12D8`

### Best Next Target After `TamperResistantBoot`

- `ApplePlatformInfoDatabaseDxe.efi`
  - best next choice if the goal is to identify what `AC5E4829-...` actually provides to:
    - `AcpiPlatform`
    - `ApplePlatformSecurityPolicy`
    - `TamperResistantBoot`

## Firmware RE: `ApplePlatformInfoDatabaseDxe.efi`

### Artifact

- Binary analyzed in IDA with MCP:
  - `C:\proj\reverse\WT6A_INF\RE_Files\FirmwarePE32_All\ApplePlatformInfoDatabaseDxe.efi`

### Why This Module Matters

- Earlier firmware RE showed that all of these depend on the same Apple-private provider GUID:
  - `AcpiPlatform`
  - `ApplePlatformSecurityPolicy`
  - `TamperResistantBoot`
- This module is the provider for that shared GUID:
  - `AC5E4829-A8FD-440B-AF33-9FFE013B12D8`

### Renamed Functions

- `InitializeApplePlatformInfoDatabase` at `0x17E6`
- `LocateGuidHobList` at `0x1722`
- `FindGuidHobRecord` at `0x178A`
- `AllocCopyPoolBuffer` at `0x1686`
- `GetPlatformInfoBlob` at `0x1154`
- `GetPlatformInfoBlobCopy` at `0x1100`
- `GetPlatformInfoBlobSize` at `0x1121`
- `GetPlatformInfoStringByIndex` at `0x125D`
- `GetPlatformInfoStringCopy` at `0x123C`
- `GetPlatformInfoStringSize` at `0x120C`
- `MatchPlatformInfoBlob` at `0x1441`
- `MatchPlatformInfoBlobExact` at `0x1420`
- `GetPlatformInfoDebugMask` at `0x1656`
- `SetPlatformInfoDebugMask` at `0x1678`

### Confirmed Findings

- `InitializeApplePlatformInfoDatabase` installs `EFI_DEBUG_MASK_PROTOCOL_GUID`.
- It then installs protocol:
  - `AC5E4829-A8FD-440B-AF33-9FFE013B12D8`
- That confirms this module is the common provider used by the other firmware modules we already traced.

### Data Source

- The module does **not** appear to compute platform data dynamically.
- Instead, it loads its database from GUIDed HOB records:
  - `LocateGuidHobList` finds the HOB configuration table
  - `FindGuidHobRecord` searches it by GUID
- `InitializeApplePlatformInfoDatabase` loads:
  - one large packed database blob into `qword_20C8`
  - one special GUID value into `qword_20B8`
  - one optional second GUID value into `qword_20C0`

### Important GUIDs Recovered

- Provider protocol installed by this module:
  - `AC5E4829-A8FD-440B-AF33-9FFE013B12D8`
- HOB / metadata GUIDs used by this module:
  - `08A0A8A0-E581-4BDF-B813-ED59116423DC`
  - `C78F061E-0290-4E4F-8DDC-5BDAAC837DE5`
  - `B8E65062-FB30-4078-ABD3-A94E09CA9DE6`

### Cross-References To Other Firmware Modules

- `C78F061E-0290-4E4F-8DDC-5BDAAC837DE5` also appears in:
  - `AcpiPlatform.efi`
  - `PlatformDataRegion.efi`
  - `PlatformInit.efi`
- `B8E65062-FB30-4078-ABD3-A94E09CA9DE6` appears in a smaller set of modules and in this provider.
- That strongly suggests the actual producers of the database/HOB content sit in:
  - `PlatformDataRegion`
  - and/or `PlatformInit`

### What The Provider Exposes

- The provider object contains a compact method table with blob and string lookup helpers.
- `GetPlatformInfoBlob`:
  - resolves a record by GUID + filter mask
  - returns raw payload bytes or required size
- `GetPlatformInfoStringByIndex`:
  - fetches a matching blob
  - walks double-NUL-separated string data inside it
  - returns a selected string entry by index
- `MatchPlatformInfoBlob` and `MatchPlatformInfoBlobExact`:
  - compare a caller-supplied blob against the stored record payload
- `GetPlatformInfoDebugMask` / `SetPlatformInfoDebugMask` expose the module-local debug mask

### Record Layout Insight

- The packed database entries are not arbitrary.
- `GetPlatformInfoBlob` walks a record table under `qword_20C8` with per-entry layout roughly:
  - 8-byte key / tag
  - flags / filter mask
  - payload size
  - inline payload bytes
- The special values `qword_20B8` and `qword_20C0` are used as preferred keys / selectors during lookup.
- So this is a small prebuilt platform-info database, not a single fixed struct.

### Strongest Interpretation

- `ApplePlatformInfoDatabaseDxe` is the shared firmware-side platform property database.
- It is upstream of:
  - `AcpiPlatform`
  - `ApplePlatformSecurityPolicy`
  - `TamperResistantBoot`
- But it is still **not** the module where the critical 5K state is created.
- It is a record provider built from HOB data prepared elsewhere.

### Relevance To The 5K Investigation

- This is a meaningful result even without direct 5K logic:
  - it explains where the other modules get their Apple-private platform properties
  - it narrows the search for the actual source of that state
- The likely producers now move one step earlier in the pipeline:
  - `PlatformDataRegion.efi`
  - `PlatformInit.efi`

### Current Best Interpretation

- The firmware stack now looks layered:
  - some earlier platform module produces GUIDed HOB records
  - `ApplePlatformInfoDatabaseDxe` publishes those records through `AC5E4829-...`
  - `AcpiPlatform`, `ApplePlatformSecurityPolicy`, and `TamperResistantBoot` consume them
- So if we want the earliest source of Apple-specific preboot platform state, this provider is probably one layer too late.

### Best Next Target After `ApplePlatformInfoDatabaseDxe`

- `PlatformDataRegion.efi`
  - best next choice if the goal is to find who actually seeds the HOB-backed platform database
- `PlatformInit.efi`
  - strong alternative if the HOB producer logic turns out to be initialized there instead

## Firmware-Wide String Sweep Pivot

### Why This Matters

- After the Apple policy/provider chain, I widened the search across the extracted firmware modules instead of continuing only module-by-module.
- This was worthwhile.
- The firmware image contains embedded AMD GOP / display EFI drivers with much more direct display, DPCD, and Boot Camp-related breadcrumbs than the Apple policy modules.

### Search Method

- Searched the flattened firmware module set in:
  - `RE_Files/FirmwarePE32_All`
- Looked for:
  - `5k`
  - `5120`
  - `2880`
  - `tile`
  - `tiled`
  - `mst`
  - `dpcd`
  - `aux`
  - `display-dual-link`
  - `display-link-type`
  - `AAPL,boot-display`
  - `ATY,EFI-dpcd`
  - `bootcamp`
  - `windowsuefi`

### Key Result

- The broad search did **not** reveal a neat literal `5K` or `5120x2880` switch inside the Apple policy modules.
- But it did reveal two anonymous AMD EFI drivers that are much closer to display bring-up:
  - `1751907B-A9BB-429F-AEEA-732989091141.efi`
  - `963307C0-554F-471B-A120-9E21A73F5F5D.efi`
- A follow-up family-match sweep revealed a third anonymous AMD EFI driver that is an even better fit for this iMac:
  - `F0140FC0-EC2A-451E-ADC5-D671AC453A8C.efi`

### Why These AMD EFI Drivers Matter

- `1751907B-A9BB-429F-AEEA-732989091141.efi` contains:
  - source path `C:\EFI\EFI_Sample\EDK\Drivers\ati\radeon\northernislands\nibootcampsupport.c`
  - connector / display property strings:
    - `display-link-type`
    - `display-dual-link`
    - `display-link-component-bits`
    - `display-pixel-component-bits`
    - `display-bpc`
    - `AAPL,boot-display`
    - `display-connect-flags`
    - `display-type`
  - internal/external connector names:
    - `DP_INT`
    - `DP1`..`DP6`
    - `TMDSA`
    - `TMDSB`
    - `HDMIC`..`HDMIF`
    - `VGAA`
  - AMD EFI state/property strings:
    - `ATY,EFIEnabledMode`
    - `ATY,EFIBootMode`
    - `ATY,EFIDisplay`
    - `ATY,PlatformInfo`
    - `DPLinkRate`
    - `DPLanes`
    - `DPLinkBit`
    - `AmdAcpiVar`

- `963307C0-554F-471B-A120-9E21A73F5F5D.efi` contains:
  - many of the same connector/property strings
  - explicit DPCD-related state strings:
    - `dpcd-training-result`
    - `ATY,EFI-dpcd-training-result`
    - `ATY,EFI-dpcd-DPPreferBitRate`
    - `ATY,EFI-dpcd-post-training`
    - `ATY,EFI-dpcd-training-fail`
  - connector names including `DP_INT`

- `F0140FC0-EC2A-451E-ADC5-D671AC453A8C.efi` contains:
  - the same `nibootcampsupport.c` source-path breadcrumb
  - the same internal connector / display property set:
    - `DP_INT`
    - `display-link-type`
    - `display-dual-link`
    - `AAPL,boot-display`
    - `ATY,EFIDisplay`
  - explicit DPCD training state strings:
    - `dpcd-training-result`
    - `ATY,EFI-dpcd-training-result`
    - `ATY,EFI-dpcd-DPPreferBitRate`
    - `ATY,EFI-dpcd-post-training`
    - `ATY,EFI-dpcd-training-fail`
  - model strings that match the 2019 5K iMac Polaris family:
    - `Radeon Pro 580X`
    - `Radeon Pro 575X`
    - `Radeon Pro 570X`
  - board / ASIC strings:
    - `ATY,Sinu`
    - `ATY,Florin`
    - `AMD Radeon Ellesmere Video Adapter`

### Strongest Interpretation

- These two anonymous AMD EFI modules are more likely to contain the real firmware-side internal-panel / DP bring-up logic than:
  - `ApplePlatformSecurityPolicy`
  - `TamperResistantBoot`
  - `ApplePlatformInfoDatabaseDxe`
- The Apple modules were still useful:
  - they proved that firmware preserves important platform state before OS handoff
  - they showed real Windows-aware and tamper-aware boot policy plumbing
- But the actual display-specific 5K lead is now much more likely to be in the AMD GOP side.

### Current Best Next Firmware Target

- `F0140FC0-EC2A-451E-ADC5-D671AC453A8C.efi`
  - current best target overall because it combines:
    - the exact iMac Polaris SKU family (`580X/575X/570X`)
    - `nibootcampsupport.c`
    - `DP_INT`
    - DPCD training result strings
    - the same display dual-link / connector property vocabulary
- `1751907B-A9BB-429F-AEEA-732989091141.efi`
  - strongest first choice because it contains:
    - `nibootcampsupport.c`
    - `display-dual-link`
    - `AAPL,boot-display`
    - `DP_INT`
- `963307C0-554F-471B-A120-9E21A73F5F5D.efi`
  - best follow-up because it contains the clearest DPCD training result strings

### Relevance To The 5K Investigation

- This is the first firmware-wide result that feels close to the real target.
- It does not prove 5K yet, but it does move the search from:
  - abstract Apple boot-policy state
  - into:
    - AMD internal display connector enumeration
    - AMD Boot Camp support code
    - AMD DPCD / DP training state
- That is much nearer to the place where a 5K internal tiled/dual-link panel decision would likely live.

## Polaris AMD GOP Deep Dive

### Current Best-Match Module

- The best firmware-side AMD target for this iMac is now:
  - `F0140FC0-EC2A-451E-ADC5-D671AC453A8C.efi`
- Why:
  - contains `Radeon Pro 580X / 575X / 570X`
  - contains `nibootcampsupport.c`
  - contains `DP_INT`
  - contains `ATY,EFI-dpcd-*` property names

### Confirmed Structure

- `sub_9520`
  - module/device initialization entry for one display device instance
  - opens:
    - `EFI_DEVICE_PATH_PROTOCOL_GUID`
    - `EFI_PCI_IO_PROTOCOL_GUID`
  - then calls:
    - `sub_B590`
    - `sub_BD00`
    - `sub_CB70`

- `sub_BD00`
  - initializes the main per-device state block
  - seeds default connector/display arrays
  - installs a protocol/interface
  - initializes connector slots and default per-display state

- `sub_2740`
  - references:
    - `C:\\EFI\\EFI_Sample\\EDK\\Drivers\\ati\\radeon\\northernislands\\nibootcampsupport.c`
  - caller is via the larger initialization/dispatch structure used by `sub_BD00`
  - calls `sub_C5D0`
  - this is the first concrete Boot Camp-specific firmware helper found in the AMD GOP

- `sub_C5D0`
  - low-level helper called only from `sub_2740` so far
  - toggles per-connector register state via repeated `sub_EF20` / `sub_EE10`
  - waits for a status bit with timeout using `sub_F720`
  - this looks like real link/control register programming, not just metadata

### Internal / External Display Property Path

- `sub_36A0`
  - publishes connector properties into the GOP/device property structure
  - confirmed properties:
    - `display-link-type`
    - `display-dual-link`
    - `display-link-component-bits`
    - `display-pixel-component-bits`
    - `backlight-PWM-freq`
  - when one connector record field is set, it explicitly stores `display-dual-link = 2`
  - this is an important result:
    - dual-link state is being decided in firmware/GOP and exported as a connector property

- `sub_5030`
  - enumerates internal connector/display masks
  - if connector state is `3840`, it uses `DP_INT`
  - maps/display-publishes internal and external outputs such as:
    - `TMDSA`
    - `TMDSB`
    - `HDMIC`..`HDMIF`
    - `DP_INT`
    - `DP1`..`DP6`

- `sub_1EBB0`
  - similar display publishing path for DP/TMDS outputs
  - explicitly handles:
    - `DP1`..`DP6`
    - `DP_INT`
  - uses `sub_DBB0` / `sub_F650` to publish `ATY,EFIDisplay` entries

### DPCD Training Property Path

- `sub_1DD20`
  - higher-level DP/link preparation routine
  - performs a sequence of DP helper calls
  - ends by calling `sub_22740`

- `sub_22740`
  - builds and publishes the DPCD training result property set
  - confirmed string/property anchor:
    - `ATY,EFI-dpcd-training-result`
  - also associated with:
    - `dpcd-training-result`
    - `ATY,EFI-dpcd-DPPreferBitRate`
    - `ATY,EFI-dpcd-post-training`
    - `ATY,EFI-dpcd-training-fail`
  - behavior:
    - derives values from the DPCD buffer `a2`
    - pushes multiple DP-related fields via `sub_1BFA0` and `sub_1BB90`
    - performs extra transaction/delay steps
  - this is the clearest firmware-side proof so far that the GOP is doing DP training work and publishing the result for later consumers

### Strongest Interpretation

- The Apple platform-policy modules were useful to establish that preboot state is preserved.
- But this AMD GOP module is much closer to the real 5K target.
- Specifically, it shows that firmware/GOP is already responsible for:
  - internal DP connector classification (`DP_INT`)
  - dual-link property publication
  - DPCD training result capture/publication
  - Boot Camp-specific low-level helper logic

### Why This Is Important For Linux

- This strongly supports the idea that plain Linux is missing more than a modeline.
- The pre-OS GOP/firmware path appears to prepare:
  - connector properties
  - internal/external classification
  - DPCD training state
  - possibly Boot Camp-specific register programming
- That is exactly the kind of state Linux would not see if the Apple-approved preboot path did not run.

### Best Next RE Questions

- What exact condition makes `sub_36A0` set `display-dual-link = 2`?
- What exact condition makes the internal display become `DP_INT` / `3840` in `sub_5030` / `sub_1EBB0`?
- What does `sub_C5D0` actually toggle for the Boot Camp helper path?
- Does the DPCD training path feed any later boot variable, ACPI object, or handoff buffer that Linux could mimic?

### Polaris AMD GOP Deep Dive: Saved-State Replay And Shared Graphics Protocol

This pass went deeper into the same Polaris GOP module and produced a much stronger picture of how the pre-OS display state is carried forward.

#### Key Function Renames Added In IDA

- `0x3E00` -> `GopApplyConnectorConfigurationWithOverride`
- `0xA320` -> `GopReplaySavedDisplayConfiguration`
- `0x167C0` -> `GopAdjustViewportForTiledDisplayLayout`
- `0xB460` -> `GopLocateSharedGraphicsProtocol`
- `0xB590` -> `GopQueryHibernateWakeState`
- `0xB4C0` -> `GopFetchPlatformInfoBlob`
- `0xB770` -> `GopRegisterDeviceWithSharedGraphicsProtocol`
- `0xD2E0` -> `GopComputePreferredDpLinkRate`
- `0xD200` -> `GopComputePreferredDpLaneCount`
- `0xD9E0` -> `GopPublishDpLinkCapabilities`
- `0x1D590` -> `GopProgramDpTrainingSet`
- `0x16640` -> `GopProgramDisplayWindowPreAdjust`
- `0x16700` -> `GopProgramDisplayWindowPostAdjust`

#### Override Block Is Caller-Supplied, Not Locally Inferred

- `GopApplyConnectorConfigurationWithOverride`
  - this is the only caller of `GopLoadDisplayOverrideBlock`
  - flow:
    - `sub_4330` copies the base connector record
    - `GopLoadDisplayOverrideBlock` copies the caller-supplied override block
    - `GopPublishDisplayConnectorProperties` publishes `display-link-type`, `display-dual-link`, and related properties
  - conclusion:
    - `display-dual-link = 2` is not inferred inside the property publisher
    - it comes from an upstream override block supplied by another part of the firmware stack

#### Saved Display Configuration Replay Is A Major Pre-OS Path

- `GopReplaySavedDisplayConfiguration`
  - consumes a multi-display saved-config blob
  - allocates arrays sized by display count
  - copies:
    - 48-byte per-display records
    - 80-byte per-display override blocks
  - for each display:
    - preserves the special connector mask from the live connector record:
      - `mask = state[0x17C] & 0xF00`
    - stores that mask back into the saved state and live per-display slots
    - republishes connector properties via `GopPublishDisplayConnectorProperties`
  - it explicitly scans the connector's mode records for type `11`

- Important disassembly-level observations from `GopReplaySavedDisplayConfiguration`
  - it copies the 80-byte block from the input record into a replay buffer and stores a pointer to it
  - it preserves `0xF00` / `3840` in the replayed connector state
  - it calls `GopPublishDisplayConnectorProperties` on the reconstructed connector data
  - it then calls `sub_167C0` / `GopAdjustViewportForTiledDisplayLayout` with:
    - display count
    - two booleans derived from paired per-display fields

- interpretation:
  - this is not just replaying width/height
  - it is replaying connector identity, override state, and a tile-like layout decision

#### The Special Internal Display Class Is Persisted Explicitly

- `sub_1A160`
  - maps connector masks to dedicated per-connector flag bytes
  - special cases include:
    - `256`
    - `512`
    - `768`
    - `1024`
    - `1280`
    - `1536`
    - `3840`
  - `3840` maps to its own dedicated state byte at `a1 + 18926`

- implication:
  - the internal `DP_INT` path is not a generic alias for ordinary DP
  - it has a dedicated state slot inside the GOP

#### Shared Preboot Graphics Protocol

- `GopLocateSharedGraphicsProtocol`
  - calls `LocateProtocol` on:
    - `63FAECF2-E7EE-4CB9-8A0C-11CE5E89E33C`

- `GopRegisterDeviceWithSharedGraphicsProtocol`
  - calls method `+2` on that interface during device bring-up

- `GopFetchPlatformInfoBlob`
  - calls method `+3`
  - publishes the returned blob as:
    - `ATY,PlatformInfo`

- `sub_B690`
  - calls method `+4`
  - used by the saved-config path

- `GopQueryHibernateWakeState`
  - calls method `+5`
  - publishes:
    - `EFI,hibwake`

- important cross-module note:
  - the same GUID was found in:
    - `CoreEG2.efi`
    - `CoffeelakeGraphics.efi`
    - the other AMD GOP modules for neighboring GPU families
  - interpretation:
    - this looks like a shared preboot graphics-services protocol, not something local to this one GOP

#### DP Training / Link Capability Details

- `GopComputePreferredDpLaneCount`
  - derives preferred lane count from DPCD content and bandwidth helpers

- `GopComputePreferredDpLinkRate`
  - computes preferred DP link-rate tier
  - uses staged thresholds based on DPCD-derived bandwidth math

- `GopPublishDpLinkCapabilities`
  - publishes:
    - `DPLinkRate`
    - `DPLanes`
    - `DPLinkBit`

- `GopProgramDpTrainingSet`
  - chooses one of several training-program variants based on connector class
  - uses `GopComputePreferredDpLinkRate`
  - uses `GopComputePreferredDpLaneCount`

#### Tile-Like Layout Logic

- `GopAdjustViewportForTiledDisplayLayout`
  - called from:
    - `sub_172E0`
    - `GopReplaySavedDisplayConfiguration`
  - if display count is greater than one:
    - one boolean halves the working width
    - the other boolean halves the working height
  - then it chooses between:
    - `GopProgramDisplayWindowPreAdjust`
    - register writes for centered viewport offsets
    - `GopProgramDisplayWindowPostAdjust`

- interpretation:
  - this is the strongest evidence so far that the GOP is handling multi-segment / tiled-style layout decisions before OS boot
  - it does not prove a classic MST tile topology by itself
  - but it absolutely shows pre-OS layout splitting behavior that plain Linux never sees

#### Consolidated Interpretation

- The Polaris GOP is doing much more than basic GOP mode exposure.
- It contains a real pre-OS replay path for saved display state.
- That replay path preserves:
  - the special internal connector class `0xF00`
  - per-display override blocks
  - mode type `11`
  - tile-like width/height split decisions
- It also depends on a shared graphics-services protocol that supplies:
  - platform info
  - saved-state services
  - hibernation-wake state

This is now one of the strongest firmware-side explanations for why plain Linux only sees 4K:

- the Apple/AMD preboot graphics stack appears to prepare and replay richer internal-display state before the OS starts
- the Windows AMD driver then finishes runtime enablement
- plain `systemd-boot -> Linux` likely misses this entire replay/graphics-services path

#### Best Next Targets From Here

- `CoreEG2.efi`
  - likely the best candidate to decode the shared protocol `63FAECF2-E7EE-4CB9-8A0C-11CE5E89E33C`

- deeper tracing inside this Polaris GOP if needed:
  - identify the exact structure format of the 80-byte override block
  - resolve what mode type `11` means semantically
  - decode the shared-protocol method `+4` used by the saved-config path
  - decode the Boot Camp helper `GopBootCampPrepareLinkState` / `GopApplyBootCampLinkRegisterSequence` at register level

### CoreEG2: Shared Protocol Provider And Internal Apple Complex Display Path

This was the next major breakthrough after the Polaris GOP work.

#### Shared Protocol Provider Confirmed

- In `CoreEG2.efi`, module entry installs the same private protocol GUID that the Polaris GOP locates:
  - `63FAECF2-E7EE-4CB9-8A0C-11CE5E89E33C`

- Key renamed routine:
  - `0xC4B5` -> `CoreEg2ModuleEntryInstallDisplayProtocols`

- This module entry does all of the following:
  - installs `EFI_DEBUG_MASK_PROTOCOL_GUID`
  - installs multiple `EFI_DRIVER_BINDING_PROTOCOL_GUID` instances
  - installs `EFI_COMPONENT_NAME_PROTOCOL_GUID`
  - installs the private protocol `63FAECF2-E7EE-4CB9-8A0C-11CE5E89E33C`
  - installs a second private protocol:
    - `356A1A6B-6C51-4964-9B6E-D623A0566E11`

- conclusion:
  - `CoreEG2` is the provider side of the protocol that the Polaris AMD GOP later `LocateProtocol`s
  - this closes the loop on the shared preboot graphics-services path

#### Shared Protocol Table

- Protocol interface object installed for `63FAECF2-E7EE-4CB9-8A0C-11CE5E89E33C`:
  - interface table at `0xE0F8`

- renamed entries:
  - `0x6742` -> `CoreEg2SharedProtocolMethod2`
  - `0x6768` -> `CoreEg2SharedProtocolMethod3`
  - `0x67AA` -> `CoreEg2SharedProtocolMethod4`

- those methods forward into a backend object stored at:
  - `qword_E668`

- strong interpretation:
  - `CoreEG2` exposes a compact protocol veneer, while the real stateful behavior lives in the backend object

#### Shared Backend Object Behavior

- `CoreEg2PublishDisplayPropertiesToSharedService`
  - renamed from `0x20B3`
  - repeatedly calls backend method `qword_E668 + 24`
  - pushes property-style keys and values into the shared service
  - observed property-style strings include:
    - `link-type`
    - data-justification / dither / pixel-format style keys

- `CoreEg2RestoreBootGammaFromNvram`
  - renamed from `0x1CCC`
  - reads `BootGamma` from NVRAM
  - applies gamma-related state to the display path
  - then uses backend method `qword_E668 + 16`

- another direct backend use appears in the display-device open path:
  - `CoreEg2` pushes `AAPL,AuxPowerConnected` through backend method `qword_E668 + 16`

- conclusion:
  - this shared service is not abstract fluff
  - it is carrying real per-display preboot state and properties

#### Complex Internal Display Path

- Renamed:
  - `0x7A5B` -> `CoreEg2ComplexDisplayInit`
  - `0x2F22` -> `CoreEg2InitFramebufferAndActivateDisplays`

- The embedded instrumentation strings inside `CoreEg2ComplexDisplayInit` are extremely relevant:
  - `START:ComplexDisplayInit`
  - `START:CDSrchDevForIntDisp`
  - `END:CDSrchDevForIntDisp`
  - `START:SetupSecondConIntAppleCD`
  - `END:SetupSecondConIntAppleCD`
  - `START:CDApplyDefaultConfig`
  - `END:CDApplyDefaultConfig`
  - `START:CDLinkTrainDPConnector1`
  - `END:CDLinkTrainDPConnector1`
  - `START:CDLinkTrainDPConnector2`
  - `END:CDLinkTrainDPConnector2`

- interpretation:
  - this is the strongest firmware-side indication so far of an Apple-specific complex-display path with:
    - an internal display search
    - a second internal connector
    - separate DP training for connector 1 and connector 2

This fits the iMac 5K theory extremely well.

#### Why This Matters

The emerging layered model is now:

1. `CoreEG2`
   - provides shared preboot graphics/display services
   - owns complex internal Apple display initialization logic
   - appears to handle second-internal-connector / dual-DP-style bring-up

2. Polaris AMD GOP
   - consumes that shared protocol
   - replays saved display state
   - publishes `DP_INT`, `display-dual-link`, `AAPL,boot-display`, and `ATY,EFI-dpcd-*`

3. Windows `amdkmdag.sys`
   - sees the panel in a special internal state
   - then performs runtime secondary-tile / tiled-display / pipe-split enablement

This is now much stronger than the original “maybe the firmware enables 5K somehow” theory. There is concrete evidence for:

- a shared preboot graphics-services layer
- Apple-specific complex internal-display handling
- a second internal connector path
- separate DP training stages before the OS starts

#### Best Next Steps After CoreEG2

- stay in `CoreEG2` long enough to decode:
  - what `CoreEg2SharedProtocolMethod2/3/4` semantically mean
  - where `qword_E668` is constructed
  - how `SetupSecondConIntAppleCD` decides to run

- then correlate that back to:
  - the Polaris GOP saved-state / `DP_INT` / `display-dual-link` path
  - the Windows `type 128` / `EnableSecondaryTileIfRequired` path

#### CoreEG2 Deep Dive: Shared-Service Registration And Command Bridge

- Additional renamed routines:
  - `0x8D08` -> `CoreEg2GenericInitializeGraphicsDevice`
  - `0x571C` -> `CoreEg2SendSecondaryDisplayOpcode1265AndReloadState`
  - `0x2674` -> `CoreEg2TearDownActiveDisplaySequence`
  - `0x395D` -> `CoreEg2FindOrCreateDisplayByConnector`
  - `0x9AEA` -> `CoreEg2ConfigureAndActivateDisplayPipeline`
  - `0x9D72` -> `CoreEg2SelectNextDisplayPipelineSlot`
  - `0x78AD` -> `CoreEg2RegisterDevicePathPropertyDatabase`
  - `0x7975` -> `CoreEg2RegisterPlatformInfoDatabase`
  - `0x78C6` -> `CoreEg2RegisterMlbLedProtocol`
  - `0x7883` -> `CoreEg2RegisterSmcIoProtocol`
  - `0x761A` -> `CoreEg2RegisterUiThemeProtocol`

- Important correction to the earlier interpretation of `qword_E668`:
  - `qword_E668` is not merely an opaque internal CoreEG2 backend object
  - it is first populated by the registration callback tied to the provider labeled:
    - `Platform Info Database`
  - this provider uses GUID:
    - `AC5E4829-A8FD-440B-AF33-9FFE013B12D8`

- practical interpretation:
  - CoreEG2's private protocol `63FAECF2-E7EE-4CB9-8A0C-11CE5E89E33C` is a thin wrapper over a broader Apple platform-info / property service
  - the same service is used to publish dynamic display state, not just static blobs

- other registration callbacks recovered from the same notifier table:
  - `EFI_DEVICE_PATH_PROPERTY_DATABASE_PROTOCOL_GUID` -> `CoreEg2RegisterDevicePathPropertyDatabase`
  - `APPLE_SMC_IO_PROTOCOL_GUID` -> `CoreEg2RegisterSmcIoProtocol`
  - `D5B0AC65-9A2D-4D2A-BBD6-E871A95E0435` -> `CoreEg2RegisterUiThemeProtocol`
  - `A662CBA3-8085-41C6-907D-F452FF73EF3F` -> `CoreEg2RegisterMlbLedProtocol`

- confirmed backend behavior:
  - `CoreEg2PublishDisplayPropertiesToSharedService` repeatedly calls `qword_E668 + 24`
  - `CoreEg2RestoreBootGammaFromNvram` uses `qword_E668 + 16`
  - `sub_5F83` uses `qword_E668 + 16` for:
    - `AAPL,AuxPowerConnected`
  - `CoreEg2ComplexDisplayInit` uses `qword_E668 + 16` for:
    - `ComplexDisplaySetup`
    - `GenericComplexDisplayFailureInfo-%d`

- concrete property names recovered from `CoreEg2PublishDisplayPropertiesToSharedService`:
  - `LinkType`
  - `DataJustify`
  - `Dither`
  - `PixelFormat`
  - `InverterFrequency`
  - per-timing keys built as `T001`, `T002`, etc.

- this means the shared service is a real preboot display-property transport layer
  - it is not only diagnostic logging
  - it carries both display capability/state and complex-display setup payloads

#### CoreEG2 Deep Dive: Exact Complex-Display Selection Logic

- `CoreEg2ComplexDisplayInit` does not just vaguely "search for internal displays"
  - it searches a display list for a candidate with type `0x0B`
  - then enters the block instrumented as:
    - `START:SetupSecondConIntAppleCD`

- inside `SetupSecondConIntAppleCD`, firmware first issues a low-level command:
  - opcode `0x4F1`
  - payload size `1`
  - then stalls for `10000` microseconds

- after that, it scans for a second connector candidate and accepts it only if the candidate's identity block matches all of:
  - byte at offset `+0x0B` equals `0xAE`
  - big-endian word at offset `+0x08` equals `0x0610`
  - `(flags at +0x0A) & 3 == 2`

- this same Apple identity gate is then rechecked against the current connector object
  - if both sides match, firmware copies the second-connector state into the primary complex-display object via `sub_205C`
  - it also records the connector-list node in `[rbx+0x20]`

- informed inference:
  - this is very likely the firmware-side pairing of the two internal links used by the 5K iMac panel path
  - the identity check is too specific to be generic external-DP handling

#### CoreEG2 Deep Dive: From Complex Setup To The Same Command Family Seen In Windows

- after the second-connector path and default-config construction succeed, `CoreEg2ComplexDisplayInit` publishes:
  - `"ComplexDisplaySetup"`
  - length `0x105`
  - through the Device Path Property Database provider (`qword_E668 + 16`)

- later in the same function, after complex display setup/training, firmware calls:
  - `CoreEg2SendSecondaryDisplayOpcode1265AndReloadState`

- that routine does all of the following:
  - sends lower-level opcode `1265`
  - stalls `10000` microseconds
  - fetches connector-1 state with `sub_4D99(..., 1)`
  - clones that state into the active display object with `sub_205C`
  - frees the temporary connector snapshot

- this is the strongest firmware/runtime bridge found so far
  - Windows `amdkmdag.sys` also sends opcode `1265` in the secondary-tile path
  - here, Apple firmware / AMD GOP-side code is already using that same command family before OS boot

- practical consequence:
  - the Windows driver's `opcode 1265` path is not an isolated runtime invention
  - it appears to continue a command model that already exists in the Apple firmware + AMD GOP environment

#### CoreEG2: What `1265` / `0x4F1` Looks Like So Far

- Additional opcode-focused findings from the same lower command interface:
  - `0x600` (`1536`) appears in `sub_4D99`
    - used with a 1-byte state buffer
    - likely a selector / handshake command
    - firmware polls it up to 5 times with 1ms stalls
  - `0x0A0` (`160`) appears in `sub_4D99`
    - used to fetch a structured 128-byte blob
    - this blob is later treated like connector / EDID-side state
  - `0x400` (`1024`) appears in `sub_2CB7`
    - used as a capability / signature query
    - the returned bytes are compared against a fixed signature
  - `0x42F` (`1071`) appears in `sub_2CB7`
    - used to obtain a small state byte that maps to display-rotation / orientation modes

- this makes the interface shape much clearer:
  - it is a lower AMD display-link / controller command mailbox
  - not the shared platform-info database
  - not Apple boot policy

- all `1265` / `0x4F1` send sites in `CoreEG2` are on that same command interface:
  - `0x7BD7`
  - `0x7DA0`
  - `0x8B58`
  - `0x8C08`
  - `0x2500`
  - `0x576F`

- importantly, the byte argument is not constant:
  - `SetupSecondConIntAppleCD` sends `0x4F1` with byte `1`
  - several later / cleanup / publish paths send `0x4F1` with byte `0`

- informed interpretation:
  - `0x4F1` / `1265` is probably not a one-shot "enable 5K" command
  - it looks more like a mode selector or phase switch for the complex internal dual-connector path
  - likely something in the family of:
    - enter secondary/internal-complex mode
    - exit/finalize/commit complex mode
    - switch active internal connector context

- why this matters for Windows:
  - the Windows driver also sends `1265` in the special internal-display path
  - that now looks consistent with firmware and runtime using the same lower mailbox, but at different phases
  - firmware appears to use both `1` and `0` values around the second-connector path
  - Windows likely uses the same command family after OS takeover when it re-enters or finalizes the special internal-display state

- current confidence:
  - high confidence that `1265` belongs to the lower AMD internal display-link/controller mailbox
  - medium confidence that it is a selector / phase-switch command for complex internal display mode
  - low confidence on the exact vendor name / official semantic label

#### CoreEG2 Deep Dive: Generic Initialization And Saved-State Replay

- `CoreEg2GenericInitializeGraphicsDevice` has two major branches:
  - restore saved configuration when available
  - fallback to live device search / setup when not

- confirmed saved-state behavior:
  - it queries the Device Path Property Database provider for `SavedConfig`
  - it uses `EFI_SIMPLE_BOOT_FLAG_VARIABLE_GUID` as a retry/loop guard
  - it then calls a display restore callback from the active display backend
  - on success, it walks display/link structures, recreates display objects with `CoreEg2FindOrCreateDisplayByConnector`, retrains DP if needed, reactivates, reinstalls, and finalizes the display path

- fallback live path:
  - if no saved state is restored, it searches devices with `sub_98A3`
  - configures them with `CoreEg2ConfigureAndActivateDisplayPipeline`
  - activates and installs them

- this lines up closely with the Polaris AMD GOP findings:
  - saved configuration replay
  - re-created connector state
  - re-published display properties
  - later viewport/layout adjustment for multi-segment displays

#### Updated Interpretation

- firmware-side evidence is now strong enough to move beyond the earlier vague theory

- confirmed:
  - Apple firmware contains a dedicated complex internal-display path with:
    - second-connector setup
    - separate DP training stages
    - a complex-display config payload
    - a post-setup low-level opcode `1265` command
  - the same `1265` command family also appears later in Windows `amdkmdag.sys`
  - CoreEG2 and the Polaris AMD GOP are linked through a shared Apple platform-info / display-property service

- informed inference:
  - the 5K iMac path depends on preboot construction and replay of a paired internal-display state
  - Windows is then able to recognize that special state and finish runtime 5K enablement
  - plain Linux likely misses the earlier paired/internal-display construction step entirely

- immediate consequence for Linux strategy:
  - reproducing only the Windows runtime command sequence is probably not enough
  - the missing prerequisite is increasingly likely to be:
    - preboot complex-display construction
    - plus the later runtime command/tile enablement

#### CoreEG2: Resolving The Lower Mailbox Owner

- `EG2Connector` is created by `sub_5C01`
  - allocates a 120-byte object
  - opens `EFI_DEVICE_PATH_PROTOCOL`
  - opens a private protocol at `unk_E350`
  - stores that private protocol pointer at connector offset `+0x20`

- this means the low-level mailbox used by:
  - `0x600` / `1536`
  - `0x0A0` / `160`
  - `0x400` / `1024`
  - `0x42F` / `1071`
  - `0x4F1` / `1265`
  is not implemented directly by `CoreEG2`; it is implemented by the private protocol object bound to the connector handle

- raw GUID bytes at `unk_E350`:
  - `19 10 61 cf 12 a2 75 40 83 a7 39 f7 36 4d ea 6b`
  - canonical GUID:
    - `CF611019-A212-4075-83A7-39F7364DEA6B`

- cross-module byte scan of extracted firmware modules shows this same protocol GUID appears in:
  - `CoreEG2.efi`
  - `F0140FC0-EC2A-451E-ADC5-D671AC453A8C.efi`
  - `CoffeelakeGraphics.efi`
  - `1751907B-A9BB-429F-AEEA-732989091141.efi`
  - `963307C0-554F-471B-A120-9E21A73F5F5D.efi`
  - `F3834774-48E5-4583-B1A1-48C3E8634F79.efi`

- the important practical result is that the same private protocol GUID is present in the Polaris AMD GOP module
  - this strongly suggests the mailbox behind opcode `1265` is an AMD preboot display protocol
  - `CoreEG2` now looks more like the Apple firmware orchestrator that drives the mailbox, not the module that owns it

- `sub_3117` in `CoreEG2` reinforces that model:
  - installs EDID/device-path protocols for a display handle
  - then opens `unk_E350` on the underlying connector handle
  - uses that backend to configure display state

- updated architecture model:
  - `CoreEG2`:
    - Apple-side complex-display coordinator
    - complex internal display search / setup / training orchestration
  - Polaris AMD GOP:
    - likely owner/implementer of the lower connector mailbox protocol
    - exports or services the operations later used by `CoreEG2`
  - Windows `amdkmdag.sys`:
    - runtime consumer of the same command family after OS takeover

#### Polaris AMD GOP: Private Device/Connector Protocol Stack

- the earlier mailbox-owner question is now much clearer inside the Polaris GOP module

- `sub_1750` is the main GOP bring-up path for a PCI display handle
  - calls `GopCreateDisplayDeviceContext`
  - registers the device with the shared graphics protocol
  - fetches platform info
  - initializes connector records
  - installs per-connector private protocols through `sub_73F0`
  - then continues into saved-config / live setup

- `GopCreateDisplayDeviceContext`
  - allocates the main device context
  - opens:
    - `EFI_DEVICE_PATH_PROTOCOL`
    - `EFI_PCI_IO_PROTOCOL`
  - initializes the internal device state blocks

- `sub_9900` installs a private device protocol on the GPU handle
  - GUID:
    - `UNKNOWN_PROTOCOL_GUID_3`
  - protocol object copied from template `unk_2B710`
  - template layout:
    - revision `0x10001`
    - method `0x9C10`
    - method `0x9CB0` = `GopApplySavedDisplayConfiguration`
    - method `0xA320` = `GopReplaySavedDisplayConfiguration`

- `sub_2A70` / `sub_2AF0` initialize six inline connector records in the device context
  - each connector record is 616 bytes
  - `sub_2AF0` copies:
    - `unk_2A660` into the connector protocol object at `connector + 48`
    - `unk_2A708` into a secondary helper block at `connector + 216`
  - this means the connector-protocol method table is template-based, not assembled ad hoc

- `sub_73F0` / `sub_7480` install the private connector protocol on each connector handle
  - `sub_7480` calls `InstallMultipleProtocolInterfaces` with:
    - `UNKNOWN_PROTOCOL_GUID_2`
    - interface pointer = `connector + 48`
    - `EFI_DEVICE_PATH_PROTOCOL`
  - then it opens `UNKNOWN_PROTOCOL_GUID_3` from the parent device handle
  - so each connector protocol instance is explicitly linked back to the device-level private protocol

- `sub_75F0` / `sub_7690` are the corresponding teardown path

- `UNKNOWN_PROTOCOL_GUID_2`
  - GUID bytes resolve to:
    - `CF611019-A212-4075-83A7-39F7364DEA6B`
  - this is the same GUID seen from the `CoreEG2` side
  - practical consequence:
    - `CoreEG2` is opening a private AMD GOP connector protocol, not an Apple-only mailbox

- recovered connector-protocol method table from `unk_2A660`
  - revision `0x10002`
  - function pointers currently identified:
    - `0x3160` = validation/stub helper
    - `0x3210` = connector selection / override association
    - `0x3E00` = `GopApplyConnectorConfigurationWithOverride`
    - `0x4790` = connector state / saved-config related operation
    - `0x4C70` = connector/output programming helper
    - `0x4E00` = immediate connector programming/update helper
    - `0x5030` = `GopPublishPrimaryDisplayOutputs`
    - `0x59D0` = alternate output publication / connector mask apply path
    - `0x6000` = enable/apply current connector mask path
    - `0x6180` = disable/clear current connector mask path

- especially relevant cross-links:
  - `GopApplySavedDisplayConfiguration`
  - `GopReplaySavedDisplayConfiguration`
  - `GopApplyConnectorConfigurationWithOverride`
  - `GopPublishPrimaryDisplayOutputs`
  - all sit in the same private AMD GOP protocol family that Apple `CoreEG2` later drives

- informed inference:
  - the preboot complex internal-display setup is not purely Apple code
  - Apple `CoreEG2` orchestrates the flow, but the actual display-device/connector execution path lives in AMD GOP private protocols
  - this is the clearest firmware-side analogue yet to the lower command path later seen in Windows `amdkmdag.sys`

#### CoreEG2 To Polaris GOP: Exact Interface Offset Mapping

- switching back to `CoreEG2` clarified how the Apple side actually calls into the AMD GOP connector protocol

- the important `CoreEG2` call pattern is now:
  - display object -> opened AMD connector protocol pointer at `display + 0x20`
  - from that interface:
    - direct method at `iface + 0x28`
    - secondary backend pointer at `iface + 0x30`
    - dispatcher method at `(*((void **)iface + 6? actually [iface+0x30]) + 0x20)`

- concrete `CoreEG2` call sites:
  - `CoreEg2SendSecondaryDisplayOpcode1265AndReloadState`
    - opcode `0x4F1` / `1265`
    - path:
      - `mov rcx, [rdi+20h]`
      - `mov r8, [rcx+30h]`
      - `mov r11, [r8+20h]`
      - call with `edx = 0x4F1`
  - `sub_4D99`
    - same secondary backend dispatcher for:
      - `0x600` / `1536`
      - `0x0A0` / `160`
    - fallback branch calls the direct connector protocol method at `iface + 0x28`
  - `sub_2CB7`
    - same secondary backend dispatcher for:
      - `0x400` / `1024`
      - `0x42F` / `1071`

- this is the strongest exact cross-module alignment so far:
  - the direct `iface + 0x28` call in `CoreEG2` matches the static connector-protocol method slot we recovered from the Polaris GOP template
  - the opcode family goes through a second-level backend pointer at `iface + 0x30`
  - therefore:
    - the AMD connector protocol has both:
      - direct exported methods
      - a dynamically attached secondary backend object used for opcode-style mailbox commands

- practical consequence:
  - the remaining unknown is narrower now
  - we do not need to guess where `1265` lands conceptually
  - we need to find where the Polaris GOP initializes `iface + 0x30` and what object owns the dispatcher method at `+0x20`

#### Correction: `1265` Dispatcher Lives In `graphicsconnector.c`, Not `framebuffer.c`

- important correction from the next Polaris GOP pass:
  - the earlier `616`-byte `framebuffer.c` object family is **not** the receiver for the `1265` mailbox path
  - that was a false lead for the secondary-dispatcher ownership question

- the actual receiver path is the `408`-byte `graphicsconnector.c` object family built by:
  - `GopInitializeGraphicsConnectorObjects` (`0x65E0`)
  - `GopInitializeGraphicsConnectorObject` (`0x6660`)

- `GopInitializeGraphicsConnectorObject` proves the backend ownership:
  - connector object base:
    - `a1 + 408 * index + 144`
  - protocol interface copied to `connector + 48` from `unk_2B0C0`
  - for `0xF00`-class / complex-display connectors:
    - backend template copied from `unk_2B120` to `connector + 224`
    - `*(QWORD *)(connector + 96) = connector + 224`
  - because the exported protocol begins at `connector + 48`, this means:
    - protocol offset `+0x30` points at the backend object at `connector + 224`

- this exactly matches the Apple `CoreEG2` call pattern:
  - `display + 0x20` -> opened AMD connector protocol
  - `iface + 0x30` -> graphicsconnector backend object
  - backend `+0x20` -> real opcode dispatcher

- recovered backend template from `unk_2B120`:
  - backend `+0x20` = `0x7C70` -> `GopGraphicsConnectorBackendDispatchOpcode`
  - backend `+0x28` = `0x8BA0` -> `GopGraphicsConnectorBackendApplyOutputMask`
  - backend `+0x30` = `0x8470` -> unsupported/stub path

- practical consequence:
  - `CoreEg2SendSecondaryDisplayOpcode1265AndReloadState`
  - `CoreEG2ComplexDisplayInit`
  - and the other `1536` / `160` / `1024` / `1071` callers
  - all land in the AMD GOP `graphicsconnector.c` backend, not in Apple-only code and not in the earlier framebuffer private protocol family

#### `GopGraphicsConnectorBackendDispatchOpcode` Transport Model

- `GopGraphicsConnectorBackendDispatchOpcode` (`0x7C70`) is the key receiver for the `CoreEG2` opcode family

- before transport, it:
  - validates the connector object signature (`1852785479`)
  - requires connector readiness via `connector + 388`
  - normalizes the connector class mask from `connector + 380`
  - maps masks like `0x100`, `0x200`, `0x300`, `0x400`, `0x500`, `0x600`, `0xF00` into the active connector-state slot
  - waits for mask readiness through `GopWaitForConnectorMaskReady`

- after that, the dispatcher splits into two different command transports:

- `a4 == 0`:
  - low-byte connector command path
  - helper functions:
    - `GopWriteLowByteConnectorCommand`
    - `GopReadLowByteConnectorCommand`
    - special case `GopReadPagedCommand160`
  - the opcode forwarded to these helpers is only the low byte of `a2`
  - special case:
    - opcode `160` (`0xA0`) has dedicated paged-read logic and cached-buffer handling

- `a4 == 1`:
  - full-opcode extended window path
  - helper functions:
    - `GopWriteExtendedOpcodeWindow`
    - `GopReadExtendedOpcodeWindow`
  - this path passes the full opcode value (`a2`), not just the low byte

- low-level mailbox window helpers now identified:
  - `GopReadConnectorMailboxWindow`
  - `GopWriteConnectorMailboxWindow`
  - these use the register window around:
    - `+35392`
    - `+35394`
    - `+35395`
    - `+35396`
  - this looks like the concrete connector mailbox / register-window transport underneath the backend dispatcher

- the direct exported connector protocol method at `iface + 0x28` is now also identified:
  - `0x7800` -> `GopGraphicsConnectorProtocolTransfer`
  - `0x7750` -> `GopGraphicsConnectorProtocolUnsupportedStub`

- current remaining unknown:
  - whether the Apple `CoreEG2` send sites reach opcode `1265` through:
    - the low-byte path (`a4 == 0`)
    - or the full-opcode extended path (`a4 == 1`)
  - that is now the narrowest missing detail for understanding the exact semantics of `0x4F1` in firmware

#### `CoreEG2` `1265` Send-Site Decode: Full-Opcode Path Confirmed

- the next `CoreEG2` pass resolved the biggest remaining ambiguity:
  - Apple firmware sends `0x4F1` / `1265` through the **full-opcode extended transport**
  - not through the low-byte-only path

- exact call shape from `CoreEg2SendSecondaryDisplayOpcode1265AndReloadState` (`0x571C`):
  - receiver:
    - opened AMD connector protocol at `display + 0x20`
    - backend pointer at `iface + 0x30`
    - dispatcher at `backend + 0x20`
  - arguments:
    - opcode = `1265`
    - `a3 = 0`
    - `a4 = 1`
    - data pointer = address of a 1-byte stack buffer
    - data length pointer = address of an `int` initialized to `1`
    - destination = `0`
    - destination length = `0`
  - payload byte value at this site:
    - `0`
  - immediate behavior after the send:
    - firmware stalls `10 ms`
    - calls `sub_4D99(display, 1)` to reload connector-1 state

- exact call shape from `CoreEg2ComplexDisplayInit` setup path (`0x7DAE`):
  - same receiver path
  - same full-opcode transport (`a4 = 1`)
  - same 1-byte payload buffer and length `1`
  - payload byte value at this site:
    - `1`
  - this happens at:
    - `START:SetupSecondConIntAppleCD`

- additional `CoreEg2ComplexDisplayInit` cleanup/failure paths:
  - `0x8B66`
  - `0x8C16`
  - both send:
    - opcode `1265`
    - full-opcode transport (`a4 = 1`)
    - 1-byte payload value `0`

- practical interpretation:
  - `1265` is not just “do 5K”
  - it is a stateful full-opcode control with a meaningful 1-byte mode value
  - current best static interpretation:
    - payload `1` = enter / arm / select the complex second-internal-display phase
    - payload `0` = exit / finalize / reload / revert to the post-setup active state

- this is a much stronger decode than before because it combines:
  - exact Apple firmware caller
  - exact AMD GOP receiver
  - exact transport type
  - exact payload size
  - exact payload values observed in setup vs cleanup/reload

- important consequence for Linux enablement:
  - copying only a raw `1265` send is very unlikely to be sufficient
  - the opcode is part of a larger complex-display state machine
  - Linux will likely need:
    - the preboot complex-display construction
    - or a faithful recreation of the same state transitions before a runtime equivalent of the Windows path can succeed

#### Windows `amdkmdag.sys` vs Firmware `CoreEG2`: `1265` Comparison

- the Windows runtime helper now lines up with the firmware path much more closely than before

- Windows secondary helper:
  - `sub_141B716F0`
  - prototype:
    - `__int64 __fastcall(__int64, char)`
  - behavior:
    - initializes opcode `1265`
    - initializes a 1-byte payload buffer with value `1`
    - sends through `sub_141EDFC70(..., 1265, &payload, 1)`
    - retries once after `10 ms` if the first attempt fails
    - if successful and caller requested it, waits another `100 ms`

- exact implication:
  - Windows also uses a **1-byte payload value of `1`**
  - that matches the firmware **setup / enter** form
  - it does **not** match the firmware **cleanup / reload** form, which uses payload `0`

- lower Windows dispatch wrapper:
  - `sub_141EDFC70`
  - this is a thin lower-controller wrapper
  - it forwards:
    - lower session/controller id from `a2 + 0x30`
    - opcode
    - data pointer
    - data length
  - to a vfunc at offset `+0x190` on a lower controller interface

- caller 1:
  - `sub_141A174B0`
  - identified earlier as the `dal3::WindowsDM::EnableSecondaryTileIfRequired` path
  - it only emits `1265` if:
    - connected pin flags match the special path
    - the detected display object type is `128`

- caller 2:
  - `sub_141B6A720`
  - post-bring-up path
  - after a successful display-specific bring-up, if the lower display object has a nonzero flag at `+1460`, it also emits the same `1265` helper

- comparison against firmware:
  - firmware `CoreEG2`
    - sends `1265` with payload `1` at `START:SetupSecondConIntAppleCD`
    - sends `1265` with payload `0` in cleanup / reload paths
  - Windows `amdkmdag.sys`
    - the traced helper `sub_141B716F0` sends payload `1`
    - so far, the traced Windows 5K/tile paths do **not** show the matching payload-`0` cleanup form

- informed interpretation:
  - Windows appears to be replaying the **enter / arm secondary-path** side of the same state machine
  - firmware appears to own the earlier setup and later cleanup/reload transitions
  - this strengthens the model that:
    - firmware builds the complex internal two-connector state
    - Windows resumes from that prepared state and reasserts the secondary-path enable with payload `1`

- practical consequence for Linux:
  - a Linux runtime patch that only mimics the Windows helper would most likely reproduce only the **payload-1 reassertion**
  - it would still be missing the preboot setup and cleanup/reload choreography that firmware performs around it

#### Windows Lower-Level Command Interface Map

- the current best answer to “what is the lower level?” is:
  - not an obviously separate `.sys` file
  - it is a lower command interface **inside `amdkmdag.sys`**
  - the traced wrappers call into a lower controller object via vtable methods

- write wrapper:
  - `sub_141EDFC70`
  - caller passes:
    - lower transport/context object
    - upper display object
    - opcode
    - payload pointer
    - payload length
  - wrapper then:
    - loads `transport->+8`
    - dereferences to a lower controller/interface object
    - calls its vfunc at offset `+0x190`
    - passes:
      - controller object
      - session/object id from `upper + 0x30`
      - opcode
      - payload pointer
      - payload length

- read wrapper:
  - `sub_141EDDCD0`
  - same transport/context chain
  - same session/object id from `upper + 0x30`
  - calls sibling vfunc at offset `+0x188`

- interpretation:
  - this is an internal lower controller mailbox / command channel
  - not a one-off helper
  - there is a paired read/write API:
    - write at `+0x190`
    - read at `+0x188`

- direct write-wrapper callers currently confirmed:
  - `sub_141B716F0`
    - opcode `1265`
    - payload length `1`
    - payload byte `1`
    - retries after `10 ms`
  - `sub_141B71E40`
    - opcode `0x170`
    - payload length `1`
    - payload is a control byte assembled from several flags
    - and, in one branch:
    - opcode `0x116`
    - payload length `1`
  - `sub_141B86600`
    - opcode `0x2006`
    - payload length `1`
  - `sub_141AFF4A0`
    - thin generic wrapper
    - forwards arbitrary opcode/data to `sub_141EDFC70`
  - `sub_141BB92D0`
    - second thin generic wrapper
    - also forwards arbitrary opcode/data to `sub_141EDFC70`

- generic chunked write path:
  - `sub_141BB93B0`
  - splits larger payloads into chunks
  - repeatedly calls `sub_141BB92D0`
  - this means the lower controller interface is used for more than tiny control bytes

- direct read-wrapper callers currently confirmed:
  - `sub_141B86600`
    - reads opcode `0x170` / `368`
    - reads opcode `0x2006` / `8198`
  - `sub_141AFF430`
    - generic read wrapper
  - `sub_141BB9340`
    - generic chunked read wrapper
  - `sub_141EFE220`
    - another direct read caller

- practical takeaway:
  - the Windows `1265` path is part of a broader lower controller protocol family
  - the visible opcode surface is already wider than just `1265`:
    - `0x116`
    - `0x170`
    - `0x2006`
    - plus generic chunked traffic
  - this strengthens the model that `1265` is one state transition inside a larger internal display-controller mailbox API

#### What That "Lower Level" Looks Like in Practice

- the next layer up from the raw read/write wrappers is **not** another obvious Windows `.sys`
- instead, it is a **per-link worker / broker layer inside `amdkmdag.sys`**
- strongest evidence:
  - `sub_141B0F930`
    - waits on five events/timers
    - dispatches per-link work items
    - logs timeout / CPIRQ style events through the same link object
  - `sub_141D59EC0`
    - logs:
      - `[Link %d] --> TIMEOUT`
      - `[Link %d] --> CPIRQ`
    - processes the event/state machine for that link object
  - helper operations on the same link object:
    - `sub_141D5B220`
      - logs `mod_hdcp_reset_connection`
    - `sub_141D5B430`
      - logs `mod_hdcp_remove_display`
    - `sub_141D5B5D0`
      - logs `mod_hdcp_add_display`

- interpretation:
  - the lower mailbox used by `1265` is sharing infrastructure with a **per-link HDCP / link-management backend**
  - this does **not** prove that `1265` itself is an HDCP command
  - but it does show that the receiver lives in a link-centric control layer, not just a generic display-mode helper
  - supporting binary-wide clue:
    - separate string thunk helpers expose:
      - `DisplayManager_NoDCE::AddHdcpCtx`
      - `DisplayManager_NoDCE::RemoveHdcpCtx`
      - `DisplayManager_NoDCE::CleanupAllHdcpCtx`
      - `DisplayManager_NoDCE::NotifyHDCPAuthStatusChanged`
    - so this is part of a real HDCP/display-manager subsystem in `amdkmdag`, not an isolated leaf

- the wrapper-publishing side that feeds this worker layer:
  - `sub_141AFF6B0`
    - builds an object with per-index child workers
    - installs generic read/write/config callback pointers:
      - `sub_141AFF430`
      - `sub_141AFF4A0`
      - `sub_141AFF510`
      - `sub_141AFF5E0`
      - `sub_141AFF2A0`
  - `sub_141B10190`
    - allocates the worker state block
    - initializes timers, events, and a system thread
  - `sub_141B0F700`
    - starts the worker thread

- practical takeaway:
  - the "lower level" in Windows currently looks like:
    - `amdkmdag` display logic
    - callback/wrapper object family
    - per-link worker / broker object
    - lower controller mailbox vfuncs at `+0x188`, `+0x190`, `+0x198`
  - so the `1265` path is riding the same internal link-control infrastructure that also services HDCP / CPIRQ / link-state work

#### Windows Display Entry Construction: Where Type 128 Actually Comes From

- the next useful breakthrough on the Windows side is that the `type 128` object is not created by the `1265` helper path itself
- it is constructed much earlier as part of the legacy connector/display-entry build path

- constructor chain:
  - `sub_141B6CF40`
    - small dispatcher:
    - if `*(a2 + 24) == 1`, takes the sibling constructor path `sub_141B6CF90`
    - otherwise falls through to `sub_141B6D1F0`
  - `sub_141B6D1F0`
    - string evidence inside function:
      - `dc_link_construct_legacy`
      - `BIOS object table - link_id: %d`
      - `Connector[%d] description:signal %d`
    - practical interpretation:
      - this is the constructor for the same per-display entry family later returned by:
        - `sub_141A33040`
        - `sub_141AFF680`

- important field mapping recovered from `sub_141B6D1F0`:
  - `entry + 336`
    - parent adapter/display object
  - `entry + 344`
    - lower transport/context pointer
    - this is the exact path later used by:
      - `sub_141EDFC70`
      - `sub_141EDDCD0`
      - `sub_141EDF080`
  - `entry + 472`
    - BIOS/object-table connector/link identifier
    - not the runtime signal/type
    - used by `sub_141B707B0`
  - `entry + 56`
    - runtime signal/type classification
    - this is the field checked by:
      - `dal3::WindowsDM::EnableSecondaryTileIfRequired`
      - `sub_141B6CA70`

- exact signal/type mapping recovered from the constructor switch on the low byte of `entry + 472`:
  - object id `0x01` / `0x03` -> signal `1`
  - object id `0x02` / `0x04` -> signal `2`
  - object id `0x0C` -> signal `4`
  - object id `0x0E` -> signal `8`
  - object id `0x13` -> signal `32`
  - object id `0x14` -> signal `128`

- this is the strongest Windows-side explanation so far for the later secondary-tile path:
  - the special runtime `128` check is simply the already-constructed legacy connector/display entry for BIOS connector object id `0x14`
  - so `EnableSecondaryTileIfRequired` is not inventing a special display class
  - it is reacting to one that came out of the earlier BIOS/object-table based construction path

- important Linux normalization / correction:
  - Linux DC uses the same numeric identities for the ordinary eDP path:
    - `AMD_Linux/display/include/grph_object_id.h`
      - `CONNECTOR_ID_EDP = 20` / `0x14`
    - `AMD_Linux/display/include/signal_types.h`
      - `SIGNAL_TYPE_EDP = (1 << 7) = 128`
    - `AMD_Linux/display/dc/bios/bios_parser_common.c`
      - `CONNECTOR_OBJECT_ID_eDP -> CONNECTOR_ID_EDP`
  - corrected interpretation:
    - the Windows `object id 0x14 -> signal/type 128` path is **not** an exotic private Apple-only signal class
    - it maps cleanly onto the ordinary AMD `eDP` connector/signal path that Linux also uses
  - consequence:
    - Linux likely does get the same high-level connector classification (`eDP`) as Windows
    - the real divergence is more likely later:
      - in the firmware-saved complex-display state on that eDP path
      - or in the Windows runtime completion path (`1265`, retraining, secondary-path reassertion)

#### Shared Lower Helper Behind The Display Entry

- `sub_141B707B0`
  - recovered role:
    - looks up a connector-state handle from:
      - transport/provider at `entry->+344`
      - BIOS/object id at `entry->+472`
      - storage/context at `transport->+104`
  - concrete behavior:
    - first vfunc at `+32` maps object id to an intermediate byte-sized selector
    - second vfunc at `+72` fills a small record
    - final helper `sub_141BB61C0(...)` converts that into a state handle

- direct callers currently confirmed:
  - `sub_141B6D1F0`
    - constructor-time HPD / connector-state setup
  - `sub_141B6E610`
    - maps a returned connector-state code to a 0..5 ordinal
  - `sub_141B704F0`
    - queries connector readiness / state and includes the special `signal == 128` fast path
  - `sub_141B70650`
    - applies per-signal timing/state values using the same handle
  - `sub_141B71450`
    - toggles a flag on the same display entry and reuses the same handle path
  - `sub_141CEFB20`
    - string evidence:
      - `dce110_edp_wait_for_hpd_ready`
    - this ties the same object-id/state-handle path directly into an eDP HPD wait flow

- interpretation:
  - the Windows `1265` path and the connector-state / HPD / readiness path are neighbors on the same display-entry object family
  - the display entry owns:
    - the lower mailbox transport at `+344`
    - the BIOS/object-table id at `+472`
    - the runtime signal/type at `+56`
  - this puts the `1265` path in the same concrete link/display-entry class that also handles:
    - connector object lookup
    - HPD state
    - readiness
    - eDP waiting

#### Windows Type-128 Bring-Up: HPD/AUX Arm Comes Before `1265`

- new IDA pass on `B416406/amdkmdag.sys.i64` adds an important ordering correction

- renamed/confirmed functions:
  - `BringUpDpOrEdpWithLinkTraining` = `0x141B6CA70`
  - `QueryConnectorReadyAndArmEdp128` = `0x141B704F0`
  - `PrepareLinkTrainingStateAndArmEdp128` = `0x141BB7090`
  - `Dce110EdpWaitForHpdReady` = `0x141CEFB20`
  - `Dce110EdpPostHpdReadyDelay` = `0x141CEF6A0`
  - `ProgramDpPreTrainingAuxState` = `0x141B8E090`
  - `ChunkedLowerControllerWrite` = `0x141BB93B0`

- in `BringUpDpOrEdpWithLinkTraining`, the `signal/type == 128` branch does **not** go straight to link training:
  - loads parent Display Core object from `link + 0x150`
  - calls function pointer at Display Core `+0x103F0` with `(link, 1)`
  - calls function pointer at Display Core `+0x103F8` with `(link, 1)`
  - then calls `ProgramDpPreTrainingAuxState`
  - then calls `PerformLinkTrainingWithRetries`

- the same two `signal/type == 128` callbacks are repeated inside `PrepareLinkTrainingStateAndArmEdp128`
  - this function is called from `PerformLinkTrainingWithRetries` before each training attempt
  - so the HPD/AUX arm is not an incidental one-time setup step
  - Windows reasserts it as part of every eDP/type-128 training attempt

- mapping the callback table against the DCE/eDP templates shows the likely field meanings:
  - Display Core `+0x103F0` maps to `Dce110EdpWaitForHpdReady(link, true)`
  - Display Core `+0x103F8` maps to `Dce110EdpPostHpdReadyDelay(link)`
  - the first helper logs the exact string:
    - `dce110_edp_wait_for_hpd_ready`
  - it looks up the connector-state handle by object id, locks/opens it, polls readiness through `sub_141BB5630`, and waits up to about `300 ms` when enabling
  - the second helper enforces an additional settle delay for connector object id `0x14`

- `ProgramDpPreTrainingAuxState` then writes pre-training state through the same lower chunked mailbox path:
  - `0x300` / `768`
  - `0x303` / `771`
  - `0x310` / `784`
  - optional `0x340` / `832`
  - all through `ChunkedLowerControllerWrite`

- the corrected Windows ordering is therefore:
  1. classify the connector as object id `0x14` / signal `128`
  2. wait/arm the eDP HPD/AUX readiness path
  3. write pre-training lower state
  4. perform link training with retries
  5. after successful bring-up, optionally reassert `1265` payload `1`

- practical correlation with the current Linux failure:
  - our Linux quirk can create a logical secondary tiled `DP-1`
  - but latest captures show the physical secondary sink disappears and DPCD writes fail during link training
  - that matches a missing "make the type-128 internal path HPD/AUX ready" step much better than a missing late compositor/tile step
  - `1265` remains relevant, but it now looks downstream of this readiness/training sequence rather than the first thing to replay

#### Consequence For Linux Replication

- this makes the Windows story more concrete:
  - the firmware/preboot side creates the special internal display entry class
  - the Windows runtime side sees `signal/type 128` already present on that entry
  - only then does it emit opcode `1265`

- so a Linux-only reproduction that merely sends the Windows `1265` command would still be missing the earlier object construction / classification state
- this is another strong point in favor of the current strategy:
  - preserve or recreate the complex preboot internal-display state first
  - then, if needed, reassert the Windows-style runtime `payload = 1` transition

#### Linux Crosswalk: `create_links()` And `dc_link_construct_legacy()`

- after recovering the Windows constructor chain, the Linux source tree now lines up very closely with it

- Windows side:
  - `sub_141B7A8F0`
    - enumerates BIOS connectors
    - allocates 936-byte display-entry objects through `sub_141B6CEE0`
    - stores them into:
      - `a1 + 8 * index + 960`
    - also creates virtual links with signal `512`
  - `sub_141B6CEE0`
    - allocates one 936-byte connector/display entry
    - calls `ConstructConnectorDisplayEntry`
  - `ConstructLegacyConnectorDisplayEntry`
    - performs the legacy BIOS/object-table based setup
    - classifies object id `0x14` as runtime signal/type `128`

- Linux side:
  - `display/dc/core/dc.c:create_links()`
    - enumerates BIOS connectors
    - calls:
      - `dc->link_srv->create_link(&link_init_params)`
    - appends each resulting `dc_link *` into:
      - `dc->links[dc->link_count]`
    - also creates virtual links
  - `display/dc/link/link_factory.c`
    - logs:
      - `BIOS object table - link_id: %d`
      - `BIOS object table - is_internal_display: %d`
      - `Connector[%d] description: signal: %s`
    - creates:
      - `ddc_service`
      - HPD / HPD RX state
      - `link_enc`
      - optional `panel_cntl`
    - maps connector ids to connector signals:
      - `CONNECTOR_ID_DISPLAY_PORT` -> `SIGNAL_TYPE_DISPLAY_PORT`
      - `CONNECTOR_ID_EDP` -> `SIGNAL_TYPE_EDP`
      - `CONNECTOR_ID_LVDS` -> `SIGNAL_TYPE_LVDS`
      - etc.

- another strong cross-link:
  - Windows:
    - `sub_141CEFB20`
    - string:
      - `dce110_edp_wait_for_hpd_ready`
  - Linux:
    - `display/dc/hwss/dce110/dce110_hwseq.c:dce110_edp_wait_for_hpd_ready()`
    - called from:
      - `link_detection.c`
      - `link_dpms.c`
      - `link_edp_panel_control.c`

- interpretation:
  - the Windows object family we just reconstructed is not some private oddity unrelated to DC
  - it lines up with the standard DC link-construction path Linux still uses:
    - enumerate connectors
    - create per-link objects
    - classify connector ids
    - create DDC / HPD / link encoder / panel control state
  - the critical difference for the iMac 5K case is therefore likely not the existence of this path
  - it is what Apple firmware / Apple GOP causes the BIOS/object-table / internal-display classification to look like before this path runs

- practical implication for Linux:
  - the best kernel-side comparison point is now:
    - `display/dc/core/dc.c:create_links()`
    - `display/dc/link/link_factory.c`
    - `display/dc/hwss/dce110/dce110_hwseq.c:dce110_edp_wait_for_hpd_ready()`
  - if Linux never gets the equivalent of the Windows `object id 0x14 -> signal/type 128` path, it will never naturally reach the later runtime `1265` behavior either

#### Polaris AMD GOP Second Pass: How `0xF00` Becomes `DP_INT`

- a second focused pass on `F0140FC0-EC2A-451E-ADC5-D671AC453A8C.efi` filled in the missing firmware-side bridge between the internal connector mask and the published boot display type

- `GopPublishPrimaryDisplayOutputs` (`0x5030`)
  - publishes all human-readable boot display outputs
  - now confirmed:
    - if current display type is `3840`
      - it calls `GopActivateConnectorStateByMask(..., 0xF00)`
      - and publishes:
        - `ATY,EFIDisplay = "DP_INT"`
  - other DP classes map the same way:
    - `256 -> DP1`
    - `512 -> DP2`
    - `768 -> DP3`
    - etc.
  - this means `3840` is not a side flag; it is the actual firmware-visible published type for the special internal DP path

- `GopLoadGraphicsConnectorOverrideBlock` (`0x44F0`)
  - loads the caller-provided connector override block into object offsets `+456..+496`
  - marks:
    - `+412 = 1`
    - `+416 = a1 + 456`
  - this confirms the earlier observation that:
    - `display-dual-link`
    - and related connector property flags
    come from a real saved/override structure, not a late local guess

- `GopSelectConnectorMaskFromSavedState` (`0x3210`, renamed from `sub_3210`)
  - this is the key backward link from saved-state replay to the connector object
  - it:
    - finds the live connector object by matching the saved handle
    - copies connector field `+380` into current published mask `+556`
    - then filters that mask by saved mode type
  - exact mode-type behavior:
    - mode `11`
      - keep only `0xF00`
    - mode `8`
      - keep only `0x10`
    - mode `6/7/10`
      - keep `0xF083`
  - this is a strong indication that:
    - mode type `11`
    is the saved-state form of the special internal/tiled DP family

- `GopInitializeGraphicsConnectorObject` (`0x6660`)
  - creates one connector object at:
    - `a1 + 408 * connector_index + 144`
  - the important branch is:
    - if `(connector_field_380 & 0xF00) != 0`
  - then the GOP:
    - installs the `graphicsconnector.c` backend template `unk_2B120` into object offset `+224`
    - points protocol field `+96` at that backend object
    - creates a connector mode entry with:
      - type `11`
      - DP-style bandwidth values
    - initializes dedicated backend fields at `+224..+248`
  - important nuance:
    - this init path does **not** directly assign raw `0xF00`
    - the base connector masks are assigned first as:
      - connector 1 -> `0x120`
      - connector 2 -> `0x220`
      - connector 3 -> `0x301`
      - connector 5 -> `0x402`
    - the later logic then uses:
      - `connector_mask & 0xF00`
      to classify them into the DP-family / type-11 path
    - the exact `0xF00` value therefore looks like a **grouped / combined internal-display mask**, not a raw per-connector seed from init
  - this still means the DP-family / special-internal path is not treated as ordinary DP:
    - it gets its own backend object
    - and its own mode-class path

- `GopReplaySavedDisplayConfiguration` (`0xA320`)
  - for each saved connector record:
    - loads the live connector object
    - takes:
      - `connector_field_380 & 0xF00`
    - stores it into:
      - current published type
      - per-connector replay metadata
      - the global replay cache (`Source[139]`)
  - it also:
    - republishes connector properties
    - restores viewport dimensions
    - calls `GopAdjustViewportForTiledDisplayLayout(...)`
    - if processing a secondary connector record, calls a pairing/sync helper `sub_10A50(...)`
  - this is one of the strongest proofs so far that the preboot path is explicitly replaying a multi-segment internal display layout

- `GopAdjustViewportForTiledDisplayLayout` (`0x167C0`)
  - when more than one segment is active:
    - halves width and/or height depending on the segment topology flags
  - then reprograms the display window registers
  - this remains the clearest firmware-side tiled-layout handling routine we have

- `GopGraphicsConnectorBackendApplyOutputMask` (`0x8BA0`)
  - this is the key backend-side bridge from connector mask to published display type
  - it:
    - picks the saved state block whose `+3612` field matches `(connector_mask & 0xF00)`
    - calls the DP/backend configurator `sub_8510(...)`
    - then, if `(connector_mask & 0xF00) == 0xF00`, explicitly writes:
      - current published display type = `3840`
    - and activates that state with:
      - `sub_E0B0(..., 0xF00)` or `sub_DD50(..., 0xF00)`
  - this is the closest firmware-side analogue yet to the Windows:
    - special connector object creation
    - followed by later runtime special-type handling
  - with the init-path nuance above, the most likely interpretation is:
    - exact `0xF00`
    - means a grouped internal multi-connector DP state
    - while `0x100/0x200/0x300/0x400`
    - are the underlying per-connector DP-family seeds

- `GopGetConnectorStateSlotByMask` (`0x1CF50`)
  - maps connector masks to dedicated state slots
  - importantly:
    - `0xF00 -> a1 + 6208`
    - `0x100 -> a1 + 8024`
    - `0x200 -> a1 + 9840`
    - `0x300 -> a1 + 11656`
    - `0x400 -> a1 + 13472`
    - `0x500 -> a1 + 15288`
    - `0x600 -> a1 + 17104`
  - so the special internal class has its own dedicated slot, not just a flag layered on top of ordinary DP slots

- `sub_1D010` (`0x1D010`)
  - gives the slot-family code for each connector mask
  - importantly:
    - `0xF00 -> 2`
    - `0x100 -> 8`
    - `0x200 -> 128`
    - `0x300 -> 512`
    - `0x400 -> 1024`
    - `0x500 -> 2048`
    - `0x600 -> 64`
  - again, `0xF00` is handled as its own special class

- `sub_8510` (`0x8510`)
  - this backend helper configures the DP state slot selected by `(connector_mask & 0xF00)`
  - it:
    - initializes the slot
    - computes preferred DP link rate
    - can retry a lower helper `sub_CCA0(...)`
    - and has a special branch:
      - if slot byte `+25` is set and connector class is `0xF00`
      - mark global flag `a1 + 18932 = 1`
  - that flag later gates acceptance of higher explicit link-rate values
  - so the special internal class is getting extra DP training / capability handling beyond ordinary DP masks

- `GopBuildDisplayConnectFlags` (`0x14860`) and `GopApplySavedDisplayConfiguration` (`0x9CB0`)
  - `GopApplySavedDisplayConfiguration` publishes the saved config, then:
    - if the saved output mask has `0x10`
      - choose mode type `8`
    - else if the saved output mask has any `0xF00` bits
      - choose mode type `11`
    - else if it has `0xF083`
      - choose mode type `10` or `6`
  - this again reinforces that:
    - `0xF00`
    - mode type `11`
    - and the special internal replay path
    are the same family

- `GopBootCampPrepareLinkState` (`0x2740`)
  - this firmware helper is real Boot Camp logic, not just dead strings
  - if `a2 == 1 || a2 == 2`
    - it applies:
      - `GopApplyBootCampLinkRegisterSequence(..., 0xF4000000)`
    - and may notify a lower link object through a virtual call
  - this is another strong sign that:
    - firmware-side Boot Camp support is actively preparing link state before the OS driver takes over

- 2026-05-20 BootCamp firmware support refinement:
  - the GOP now has the BootCamp support path named/commented in IDA:
    - `GopDriverBindingStart` (`0x1750`)
    - `GopDriverBindingStop` (`0x1B80`)
    - `GopInitializeDisplayDeviceState` (`0xBD00`)
    - `GopInstallBootCampSupportProtocol` (`0x2850`)
    - `GopUninstallBootCampSupportProtocol` (`0x28A0`)
    - `GOP_BOOTCAMP_SUPPORT_PROTOCOL_GUID`
  - the protocol GUID is:
    - `A3249A94-CB66-43FC-92F3-8BA1889FD6C7`
  - GOP driver-binding start publishes that protocol after the GOP device, connector, and framebuffer protocol stack is installed
  - the interface is embedded inside GOP display state at:
    - `state + 0x320`
  - method slot `+0x08` is:
    - `GopBootCampPrepareLinkState`
  - `GopBootCampPrepareLinkState(a1, a2, a3)`:
    - validates that `a1 - 0x320` has the GOP state signature
    - returns success without hardware writes when `a3 == 1`
    - runs the real BootCamp path when `a2 == 1 || a2 == 2`
  - the real path calls:
    - `GopApplyBootCampLinkRegisterSequence(state, 0xF4000000)`
  - `GopApplyBootCampLinkRegisterSequence` is GPU MMIO handoff plumbing, not DPCD/AUX:
    - it reads `0x5428`
    - toggles CRTC blank-control bit `0x100` at endpoint bank `+0x6E74`
    - waits for blank status bit `0`
    - writes global registers `0x2024` and `0x2C04`
    - pokes DCP primary-surface address/high at endpoint bank `+0x6810` / `+0x681C`
    - writes `0x2024` again using the saved `0x5428` value and payload bits
    - clears the same CRTC blank-control bit
  - importantly, the helper checks GOP multi-display/replay state:
    - if `state+2216` is set, it iterates `state+2212` endpoints and uses bank bytes at `state+2276+i`
    - otherwise it uses the current endpoint selected by `state+2210`
  - so the BootCamp helper can operate across both tile endpoints, but only if the GOP already has replayed/multi-display state
  - static writer search tightens this dependency:
    - `state+2216` and `state+2212` are set by `GopReplaySavedDisplayConfiguration(non-NULL)`
    - the replay cleanup / `NULL` path clears both fields
    - no independent BootCamp-support writer for the paired replay latch was found
  - output bank mapping is now decoded through the GOP connector-mask table:
    - mask `0x100` -> output bank byte `0`
    - mask `0x200` -> output bank byte `1`
    - grouped mask `0xF00` -> output bank byte `0`
  - so the BootCamp handoff can only touch both tiles when the earlier replay state already resolved endpoints into the split `0x100/0x200` connector objects
  - firmware-wide GUID search found this same protocol GUID in:
    - the current Polaris GOP module
    - another neighboring AMD GOP module
    - `PlatformInit.efi`
  - raw PE scanning of `PlatformInit.efi` shows:
    - the GUID bytes live at `.data` RVA `0x6090`
    - code has direct references to that exact GUID address around RVA `0x11C9` and `0x124A`
  - the PlatformInit caller side is now recovered:
    - `PlatformInitCallGopBootCampSupport` (`PlatformInit.efi+0x1170`) is registered as an `EVT_SIGNAL_EXIT_BOOT_SERVICES` callback
    - it calls `LocateHandleBuffer(ByProtocol, GOP_BOOTCAMP_SUPPORT_PROTOCOL_GUID, NULL, &count, &handles)`
    - for each handle it calls `ProtocolsPerHandle`
    - it matches the exact `A3249A94-...` GUID in that protocol list
    - it opens the protocol with `OpenProtocol(..., EFI_OPEN_PROTOCOL_GET_PROTOCOL)`
    - it checks `*(u32 *)iface == 0x10000`
    - it calls method slot `+0x08` as:
      - `(iface, 1, 2, 0, 0, 0)`
  - therefore GOP receives:
    - `a2 == 1`
    - `a3 == 2`
    - so it definitely takes the real `GopApplyBootCampLinkRegisterSequence(..., 0xF4000000)` path and not the `a3 == 1` no-op return
  - PlatformInit registers this through its main entry path:
    - `_ModuleEntryPoint` jumps into the large init routine at `0x3F90`
    - that routine creates an `EVT_SIGNAL_EXIT_BOOT_SERVICES` event for `PlatformInitExitBootServicesCallback`
    - then creates another `EVT_SIGNAL_EXIT_BOOT_SERVICES` event for `PlatformInitCallGopBootCampSupport`
  - the known suppressor is:
    - `PlatformInitSkipBootCampSupportCall` (`PlatformInit.efi+0x65D0`)
  - the only writer found so far is:
    - `PlatformInitSetSkipBootCampSupportCall` (`PlatformInit.efi+0x13D1`)
    - this is a protocol-notify callback for GUID `0DFD015E-A1BB-4E63-8372-89BD526DE956`
    - if it fires before `ExitBootServices`, the GOP BootCamp call is skipped
  - the source of that skip notify is now recovered:
    - `boot.efi` locates `EFI_OS_INFO_PROTOCOL_GUID = C5C5DA95-7D5C-45E6-B2F1-3FD52BB10077`
    - it checks the interface version
    - it calls slot `+0x08` with:
      - `"macOS 11.0"`
    - it then calls slot `+0x10` with:
      - `"Apple Inc."`
    - it then calls slot `+0x18` with Apple boot-info flags
  - in `EfiOSInfo.efi`:
    - slot `+0x08` is the OSName setter
    - slot `+0x10` is the OSVendor setter
    - the OSVendor setter stores the vendor string and enters a common tail
    - if the stored vendor string is exactly `"Apple Inc."`, the module transiently installs and uninstalls `0DFD015E-A1BB-4E63-8372-89BD526DE956`
    - this transient protocol pulse is what wakes PlatformInit's protocol-notify callback and sets `PlatformInitSkipBootCampSupportCall = 1`
  - therefore:
    - Apple `boot.efi` deliberately suppresses the PlatformInit/GOP BootCamp handoff
    - generic EFI / BootCamp-style OS handoff should leave the skip flag clear unless something else calls EfiOSInfo OSVendor with `"Apple Inc."`
    - the GOP BootCamp handoff is best understood as a non-Apple OS handoff helper, not the Apple 5K initialization path
  - current interpretation:
    - BootCamp firmware support is real and published by GOP
    - PlatformInit actively calls it at OS handoff with the real/non-noop argument shape
    - the visible GOP method is probably a pre-OS scanout/link handoff helper after display state exists
    - it is not yet the missing complete grouped-live/tile-pair initializer

- 2026-05-20 dynamic probe update:
  - EDK II probe shim v1.3 now logs, read-only:
    - `GOP_BOOTCAMP_SUPPORT_PROTOCOL_GUID`
    - `EFI_OS_INFO_PROTOCOL_GUID`
    - `EFI_OS_INFO_APPLE_VENDOR_NOTIFY_GUID`
    - GOP BootCamp support interface pointer/version/method
    - GOP private state at `iface - 0x320` when signature `0x30315652` matches
    - replay fields `state+0x8A2`, `state+0x8A4`, `state+0x8A8`
    - endpoint masks at `state+0x8B0`
    - output slot bytes at `state+0x8E0..0x8E7`
  - this should answer whether OCLP/plain firmware handoff differs because the GOP BootCamp protocol is absent, because replay state is absent, or because replay is live with the wrong single-endpoint shape

- current interpretation after this pass:
  - the firmware-side special internal display family is:
    - grouped connector mask `0xF00`
    - mode type `11`
    - dedicated connector-state slot `a1 + 6208`
    - published boot display type `3840`
    - published connector name `DP_INT`
  - underlying connector init still begins from the DP-family seeds:
    - `0x100`
    - `0x200`
    - `0x300`
    - `0x400`
    and the exact `0xF00` value seems to arise later when firmware combines or replays the internal multi-connector state
  - this is the strongest firmware analogue yet to the Windows:
    - special connector object id `0x14`
    - runtime signal/type `128`
  - the numeric values differ, but the role is very similar:
    - both are special preclassified internal display objects
    - both are prerequisites for the later `1265` / secondary-path state machine

- practical consequence:
  - Linux probably does **not** just need a late runtime opcode replay
  - it likely needs the equivalent of this preboot classification chain:
    - special internal connector class
    - saved-state replay
    - tiled viewport/layout setup
    - and only then, if needed, the Windows-style runtime reassertion

#### Polaris AMD GOP: How The Special Internal State Gets Prepared Before `0xF00`

- a deeper pass on the GOP detection path clarifies an important nuance:
  - the module has a canonical `0xF00` / `3840` internal-display family
  - but the local bring-up code starts from ordinary DP-family seeds and only later hands richer state to a shared Apple graphics service

- `sub_1A5D0`
  - this is the DP-family probe helper behind `sub_1A870`
  - it explicitly probes:
    - `0x100`
    - `0x200`
    - `0x300`
    - `0x400`
  - for each family it:
    - picks the dedicated state slot
    - issues paged command `160` (`0xA0`) through `GopReadPagedCommand160(...)`
    - and records success/failure in:
      - bytes `a1+18920 .. a1+18923`
      - bytes `a1+2309 .. a1+2312`
  - this means the GOP first learns which underlying DP-family seeds are actually alive on hardware

- `sub_1A870`
  - orchestrates the probing:
    - lazy-initializes the connector-presence cache
    - if needed, calls a lower wake/prepare path (`sub_19050` / `sub_19470`)
    - then probes the four DP-family masks via `sub_1A5D0`
  - after this, the GOP knows which DP-family connectors are present

- `sub_14240`
  - this is the aggregation helper used by `GopBuildDisplayConnectFlags`
  - it takes a topology/type code from the panel descriptor path and converts the active DP-family presence bytes into a grouped mask
  - notably:
    - for codes `4` and `8`
      - it ORs together any active:
        - `0x100`
        - `0x200`
        - `0x300`
        - `0x400`
      - and adds low bits `| 3`
    - other codes map to:
      - `4`
      - `8`
      - `32`
      - `64`
  - this is the first concrete place where per-link DP seeds are transformed into a richer grouped topology decision

- `GopBuildDisplayConnectFlags`
  - does **not** directly write raw `0xF00`
  - instead it:
    - iterates live connector candidates from `unk_2C050`
      - currently:
        - `0x100`
        - `0x200`
        - `0x300`
        - `0x400`
        - `0x1`
        - `0x2`
    - maps each candidate through `GopMapConnectorMaskToOutputSlot`
    - compares it against stored panel/topology descriptors at:
      - `a1+1411`
      - `a1+1435`
    - uses:
      - `sub_14240(a1, *(u8 *)(a1+1492))`
      - `sub_14240(a1, *(u8 *)(a1+1588))`
      to decide whether the current live candidate matches the stored internal-display topology
  - when a match is found, it fills:
    - the complex display record at `sub_1A080(...)`
    - geometry / timing fields
    - display-connect / pixel-combine metadata
    - and the mode-family field at `a1+720`
  - so this routine selects the complex internal topology, but the exact `0xF00` mask still does not appear here as a direct local assignment

- data-table clue:
  - the connector map at `unk_2C530` already contains a first-class entry for:
    - `0xF00`
  - alongside:
    - `0x100`
    - `0x200`
    - `0x300`
    - `0x400`
    - `0x500`
    - `0x600`
    - TMDS / analog classes
  - so the GOP design absolutely expects a canonical grouped internal-display mask
  - it is just not obviously seeded in the ordinary early candidate loop

- `sub_9A20`
  - in the saved-config apply path, this routine scans all connectors and mode entries
  - when a mode-type `11` connector is found and the shared-protocol callback reports success:
    - it marks the per-output presence byte
    - and also marks the special family byte returned by `sub_1A160(...)`
  - `sub_1A160` includes a dedicated slot for:
    - `3840 -> a1 + 18926`
  - this is another sign that the `3840` family is treated separately from ordinary DP-family bytes

- `sub_21D30`
  - this is the custom packet interpreter behind the GOP's shared-graphics protocol path
  - it is reached from:
    - `sub_139E0`
    - `sub_12C30`
    - `sub_1E110`
  - these are exactly the routines used when the GOP publishes complex display metadata, boot mode packets, and timing/layout information
  - this is now the strongest likely boundary where:
    - local GOP bookkeeping
    turns into
    - Apple shared-service state

- key inference after this pass:
  - the GOP locally performs:
    - DP-family probing
    - panel/topology matching
    - complex-display record construction
  - but the fully authoritative grouped internal-display state may only emerge after the GOP hands that metadata through the shared Apple graphics protocol
  - this lines up well with the earlier `CoreEG2` finding:
    - Apple firmware orchestrates the complex display path
    - AMD GOP provides the lower connector/backend machinery
    - the exact preboot state is likely finalized at the protocol boundary between them

- practical consequence:
  - the remaining “who writes the canonical grouped internal state?” question now points more strongly at:
    - `CoreEG2`
    - and the shared protocol methods driven by:
      - `sub_B690`
      - `sub_139E0`
      - `sub_21D30`
- so the next highest-value RE step is likely back in `CoreEG2`, not more blind local GOP hunting

## CoreEG2: Display Object Lifecycle And Complex-Display Publish Path

- second-pass conclusion:
  - the authoritative preboot state does not come from EDID alone
  - `CoreEG2` creates a dedicated 616-byte display object, attaches ordinary EDID to it, then separately builds and publishes a complex-display property block
  - only after that block exists and the Apple internal-panel signature matches does firmware send `1265` with payload `1`

- `CoreEg2CreateDisplayObject` (`0x29BB`, formerly `sub_29BB`)
  - allocates a 616-byte display object
  - copies EDID into:
    - `display + 152` = EDID buffer pointer
    - `display + 144` = EDID length
  - links the object into the global display list
  - seeds callback/publisher state used later by the complex-display path:
    - `display + 536`
    - `display + 104`
    - list links at `+48/+56` and `+64/+72`
  - on failure, it immediately falls back into `CoreEg2PublishDisplayPropertiesToSharedService`

- `sub_205C`
  - simple helper used by the create/reload paths
  - confirms that:
    - `display + 152/+144` are the copied EDID buffer and size
  - this matters because it separates ordinary panel EDID from the later complex-display block

- `CoreEg2SetSignalForDisplay` (`0x3B3E`, formerly `sub_3B3E`)
  - assigns the runtime signal/type for the display object
  - when signal values `8/9` are chosen, it parses a specific EDID descriptor and builds the complex-display property block at:
    - `display + 560 .. +608`
  - it sets:
    - `display + 584 = 7`
    - `display + 592` = 28-byte temporary property array
    - `display + 600 = 0x100010000`
    - `display + 608 = display + 560`
  - then it serializes and publishes the resulting properties through the shared backend `qword_E668`:
    - `T1..T7`
    - `LinkType`
    - `D`
    - `DataJustify`
    - `Dither`
    - the boolean at `asc_D94A`
    - `PixelFormat`
    - `InverterFrequency`
  - this is the first strong proof that the complex-display state is assembled in the Apple orchestrator, not only inside the AMD GOP

- `CoreEg2PublishDisplayPropertiesToSharedService` (`0x20B3`)
  - first checks:
    - `display + 604 == 1`
  - when true, it republishes the complex-display property block through `qword_E668 + 24`
  - then it checks:
    - `display + 552 != 0`
    - EDID signature:
      - byte `+11 == 0xAE`
      - big-endian word at `+8 == 0x0610`
      - flags at `+10` satisfy `& 3 == 2`
  - only when both the complex block and Apple panel signature exist does it:
    - send opcode `1265`
    - payload length `1`
    - payload byte `1`
    - through the AMD backend
    - then stall `10 ms`
  - after publishing, the function tears the display object down and frees:
    - `display + 552` as a 384-byte block
    - `display + 120`
    - `display + 152`
    - finally the 616-byte display object itself

- important field interpretation after the second pass:
  - `display + 152/+144`
    - ordinary EDID pointer/length
  - `display + 560..608`
    - complex-display property working set built during signal assignment
  - `display + 584`
    - property count used during publish (`7` on the complex-display path)
  - `display + 592`
    - serialized 28-byte helper buffer used while publishing `T*` properties
  - `display + 604`
    - flag that tells publish logic the complex-display property set is live
  - `display + 552`
    - separate 384-byte complex-display blob whose presence gates the Apple-panel `1265` branch

- `CoreEg2FindOrCreateDisplayByConnector` (`0x395D`)
  - searches existing display objects by connector and internal/external class
  - otherwise:
    - fetches connector info with `sub_4D99`
    - creates the display object
    - then calls `CoreEg2SetSignalForDisplay`
  - this is the object-lifecycle bridge between connector probing and the complex-display property path

- `sub_4D99`
  - still an important connector-fetch helper
  - when called on the secondary AMD backend path, it:
    - performs selector/handshake opcode `1536`
    - then fetches a 128-byte connector blob with opcode `160`
  - this is the same backend family used later by opcode `1265`

- `CoreEg2ComplexDisplayInit`
  - after second-connector Apple-panel matching succeeds, firmware allocates `0x180` bytes and stores that pointer at:
    - `display + 552`
  - it then calls `sub_52F2(display, iterator)`
  - later in the function it publishes/activates the paired internal-display state and uses opcode `1265` payloads `1` and `0` to enter and leave that state machine

- `sub_52F2`
  - fills the newly allocated complex-display blob and related working fields:
    - `display + 416 .. +492`
  - derives:
    - `display + 424` from internal/secondary connector state
    - `display + 428` from `sub_52AA`
  - emits detailed failure records through `sub_A8BB`
  - important nuance:
    - the function only checks `display + 552` for presence
    - it does not allocate the blob itself; that happens in `CoreEg2ComplexDisplayInit`

- `sub_52AA`
  - helper that derives the complex-display link-type/category
  - if `display + 604 != 1`, it falls back to EDID-driven classification
  - otherwise it returns the special complex-display category directly

- `CoreEg2ConfigureAndActivateDisplayPipeline`
  - the lower backend gets the complex-display pointer only if:
    - `display + 604 != 0`
  - in that case it passes:
    - `display + 600`
  - otherwise it passes `NULL` and the pipeline stays on the ordinary path
  - so `display + 604` is not cosmetic; it gates whether the complex-display data is actually consumed during final pipeline setup

- `sub_8C38`
  - retry loop used during generic display bring-up
  - sequence:
    - `sub_98A3` -> select or create display object
    - `CoreEg2ConfigureAndActivateDisplayPipeline`
    - `CoreEg2InitFramebufferAndActivateDisplays`
    - `CoreEg2InstallDisplayProtocols`
    - `sub_AD18`
    - `sub_AB65`
  - this is where the generic path hands the chosen display object into the publish/config path

- `sub_AD18`
  - packages current display timing and state into a 136-byte block
  - consults the display-config table from `CoreEg2LocateDisplayConfigTable`
  - if `a3 != 0`, also copies connector-specific state from the lower backend into the package
  - this looks like another handoff point between live backend state and the Apple shared display database

- `CoreEg2InstallDisplayProtocols` (`0x3117`, formerly `sub_3117`)
  - installs:
    - `EFI_EDID_DISCOVERED_PROTOCOL`
    - `EFI_DEVICE_PATH_PROTOCOL`
    - `EFI_GRAPHICS_OUTPUT_PROTOCOL`
    - `EFI_EDID_ACTIVE_PROTOCOL`
  - opens the AMD private connector protocol `CF611019-A212-4075-83A7-39F7364DEA6B`
  - confirms that once the display object is accepted, Apple firmware exposes it through standard EFI display protocols on top of the private AMD backend

- strongest updated interpretation:
  - AMD GOP still provides the low-level connector/backend and the `1265` transport
  - but `CoreEG2` is the place where:
    - the display object is created
    - the complex-display property set is assembled
    - the Apple internal-panel signature is checked
    - and the final publish-plus-`1265` sequence is decided
  - so the special internal 5K-capable state is now best understood as:
    - Apple `CoreEG2` orchestration
    - over AMD GOP connector/backend services
    - with the final state handed out through shared firmware property services before OS boot

- practical consequence for Linux:
  - the decisive preboot prerequisite is not merely:
    - panel EDID
    - or a single opcode
  - it is:
    - creation of the special display object
    - construction/publication of the complex-display property block
    - then the Apple-panel-gated `1265 payload=1` transition
  - this makes a future minimal EFI shim more likely to succeed if it preserves or recreates the complex-display publish path, rather than trying to inject only the Windows-style runtime opcode

### What Actually Decides Whether the Dual-Connector 5K Path Is Applied

- firmware does not appear to ask a simple question like:
  - "is 5120x2880 supported?"
- instead, it decides whether the display qualifies for the Apple complex internal-display path

- the current best decision chain is:
  - `CoreEg2ComplexDisplayInit`
    - searches for the internal display path
    - then runs `SetupSecondConIntAppleCD`
  - `SetupSecondConIntAppleCD`
    - sends opcode `1265` with payload `1` through the AMD connector backend
    - looks for a second-connector candidate in the connector list
    - requires a mode entry with:
      - `type == 11`
    - fetches connector state through `sub_4D99`
      - handshake/select with opcode `1536`
      - then fetches a 128-byte connector/state blob with opcode `160`
  - second-connector acceptance requires the Apple panel signature gate:
    - EDID byte at `+11 == 0xAE`
    - big-endian word at `+8 == 0x0610`
    - flags at `+10`, low bits `== 2`
  - only after that successful match does firmware:
    - clone the second connector state into the primary complex-display object
    - allocate the `0x180`-byte complex-display blob at `display + 552`
    - call `sub_52F2` to populate the complex-display fields
    - run `CoreEg2SetSignalForDisplay`

- the key "go live" point is in `CoreEg2SetSignalForDisplay`
  - for signal values `8/9`, it writes:
    - `[display + 600] = 0x100010000`
  - this implicitly makes:
    - `[display + 604] = 1`
  - that flag is the real gate for whether the complex-display path is consumed later

- final activation happens only if both are true:
  - `display + 604 == 1`
  - `display + 552 != 0` and the Apple panel signature still matches
- then:
  - `CoreEg2PublishDisplayPropertiesToSharedService` republishes the complex-display property set
  - `CoreEg2ConfigureAndActivateDisplayPipeline` passes the complex-display pointer into the lower backend
  - firmware sends the Apple-panel-gated `1265 payload=1` transition

- if those checks fail:
  - the second-connector path is abandoned
  - cleanup paths send `1265 payload=0`
  - the pipeline falls back to the ordinary single-display path
  - the special grouped internal state never becomes authoritative

- strongest current interpretation:
  - the decision is based on:
    - active internal DP-family connector state
    - a second-connector candidate with mode type `11`
    - Apple internal panel identity
    - and successful construction of the complex-display blob
  - not on a direct "5K mode present" test

### Signal Values 11 vs 8/9

- `CoreEg2ComplexDisplayInit` first creates the primary display object and immediately calls:
  - `CoreEg2SetSignalForDisplay(display, 11)`
- that means signal `11` is the initial complex-display candidate/root path, not the final "paired 5K path is live" state

- the actual source of later signal values is:
  - `CoreEg2FindOrCreateDisplayByConnector`
  - it passes the connector-record field:
    - `*(connector_entry + 4)`
  - into `CoreEg2SetSignalForDisplay`
  - so signal `8/9/10/11/12` are coming from firmware-discovered connector entries, not arbitrary local constants

- important split:
  - signal `11/12` branch in `CoreEg2SetSignalForDisplay`
    - marks the object as a special candidate (`display+44 = 1`)
    - queries backend state
    - fills ordinary capability/timing fields
    - does **not** by itself make the complex-display path live
  - signal `8/9` branch in `CoreEg2SetSignalForDisplay`
    - parses the special property records behind `display+98`
    - and only on this branch does firmware write:
      - `[display + 600] = 0x100010000`
    - which is the store that makes:
      - `[display + 604] = 1`

- practical interpretation:
  - signal `11` is the "complex-display candidate/root object" path
  - signal `8/9` is the branch that actually turns the complex-display property block into the live paired-display state
  - therefore the boot path matters even earlier than blob allocation:
    - it has to preserve or expose connector records that later drive the `8/9` branch

- additional refinement from a second pass:
  - inside `CoreEg2SetSignalForDisplay`, the `8/9` logic is selected by:
    - `(signal & ~1) == 8`
  - so `CoreEG2` itself treats `8` and `9` as one signal family
  - there is no later branch in that path that meaningfully distinguishes `8` from `9`
  - this makes them look less like literal mode IDs and more like closely related connector-role / path-role enums coming from the live connector records

- where `8/9` come from:
  - `CoreEg2FindOrCreateDisplayByConnector` passes:
    - `*(connector_entry + 4)`
  - into `CoreEg2SetSignalForDisplay`
  - those connector entries are not created ad hoc inside `CoreEG2`; they come from the firmware-discovered connector topology and the private connector protocol chain
  - `CoreEG2` therefore consumes `8/9`; it does not invent them

- strongest current interpretation of `8/9`:
  - they are firmware/backend connector-role values for the paired Apple internal path
  - likely representing closely related sub-link / segment roles within the complex internal display
  - not direct "5K" or "5120x2880" markers

### How Control Reaches CoreEG2, And Where Boot Can Fall Back To 4K

- `CoreEg2ModuleEntryInstallDisplayProtocols` is the UEFI module entrypoint
  - it is the first code run when DXE dispatch loads `CoreEG2`
  - it:
    - records `ImageHandle` / `SystemTable`
    - installs `EFI_DEBUG_MASK_PROTOCOL_GUID`
    - locates the display-config table
    - installs multiple `EFI_DRIVER_BINDING_PROTOCOL` instances
  - so firmware does not call `CoreEG2ComplexDisplayInit` directly from a boot policy module
  - instead:
    - DXE loads the image
    - `CoreEG2` registers driver binding callbacks
    - later UEFI driver-binding dispatch invokes the matching `Supported()` / `Start()` style routines on graphics handles

- one relevant driver-binding pair is:
  - `sub_58C0`
    - behaves like a `Supported()`/probe callback
    - opens the private graphics protocol `unk_E3C0`
    - records the matching handle in `qword_E608/qword_E610`
  - `sub_5968`
    - behaves like the matching `Start()` path
    - creates a per-handle CoreEG2 node
    - opens `EFI_DEVICE_PATH_PROTOCOL` and the same private protocol `unk_E3C0`
    - and, when the discovered connector counts match expected counts, calls:
      - `CoreEg2ComplexDisplayInit`

- so the control chain is approximately:
  - DXE dispatcher
  - `CoreEg2ModuleEntryInstallDisplayProtocols`
  - `EFI_DRIVER_BINDING_PROTOCOL` support/start callbacks
  - `sub_5968`
  - `CoreEg2ComplexDisplayInit`

- the best current boot divergence points are:

- 1. driver-binding / device-match stage
  - if the private graphics protocol / handle topology does not match what `CoreEG2` expects
  - `sub_5968` never reaches `CoreEg2ComplexDisplayInit`
  - then the complex internal-display path never even starts

- 2. early generic graphics-init gate
  - `CoreEg2GenericInitializeGraphicsDevice` checks:
    - `SkipNvramDisplay`
    - `sub_74C1()`
    - `dword_E670 & 0x10`
  - those can bypass or suppress parts of display state restoration before the complex path is attempted
  - this is one of the strongest candidates for where an "unsupported" boot path can lose the richer preboot state

- 3. complex-display candidate stage
  - `CoreEg2ComplexDisplayInit` creates the root display object and seeds it with signal `11`
  - if it cannot find a valid internal display object / connector set, it does not proceed to the paired path

- 4. second-connector setup stage
  - `SetupSecondConIntAppleCD` must find:
    - a second connector candidate
    - a mode entry with `type == 11`
    - a valid connector-state blob from `sub_4D99`
  - if any of that fails, the complex paired path is abandoned

- 5. Apple panel identity gate
  - the second connector must still match:
    - EDID byte `+11 == 0xAE`
    - big-endian word `+8 == 0x0610`
    - flags `(+10 & 3) == 2`
  - if that fails, cleanup sends `1265 payload=0` and the special path is not armed

- 6. live paired-path activation
  - only if the firmware-discovered connector records later drive the `8/9` signal family does `CoreEg2SetSignalForDisplay` write:
    - `[display + 600] = 0x100010000`
  - that is the write that makes:
    - `[display + 604] = 1`
  - if this never happens, the complex-display blob may exist only as a candidate, but the lower pipeline never consumes it

- best current interpretation of "right 5K path" vs "unsupported 4K path":
  - right 5K path:
    - CoreEG2 driver binding matches the graphics topology
    - generic init does not suppress state restoration
    - second internal connector is discovered
    - Apple panel signature matches
    - signal `8/9` path goes live (`display+604 = 1`)
    - complex-display properties are published and `1265 payload=1` is sent
  - unsupported 4K path:
    - one of the above stages fails
    - the pipeline falls back to ordinary single-display handling
    - cleanup / fallback sends `1265 payload=0`
    - Windows/Linux later see only the collapsed ordinary internal display

### CoreEG2 Deep Dive: What Actually Happens On Success Versus Fallback

- the biggest clarification from the deeper `CoreEG2` pass is that the split is now visible inside one function:
  - `CoreEg2ComplexDisplayInit`
  - if it succeeds, firmware takes the paired internal-display path all the way through setup, training, activation, and complex-display publication
  - if it fails at any major stage, it ends by returning:
    - `CoreEg2GenericInitializeGraphicsDevice(a1)`
  - this is currently the strongest static model for:
    - "right 5K path" vs "collapsed ordinary path"

- `sub_4D99` is the connector-record bridge between the AMD backend and `CoreEG2`
  - it fetches connector state through the AMD private connector backend using opcode `160`
  - it returns:
    - a primary 128-byte connector record
    - plus optional chained 128-byte records
      - count carried in byte `126` of the primary record
  - it validates the record set with `sub_523C`
    - requires total size to be a multiple of `0x80`
    - rejects records whose first 8 bytes match the sentinel `00 FF FF FF FF FF FF 00`
  - on failure, it publishes `FetchEDIDFailureInfo-%d`

- `CoreEg2CreateDisplayObject`
  - allocates the 616-byte display object
  - copies the fetched connector data into the display object's persistent EDID / descriptor storage via `sub_205C`
  - creates a per-display path/property blob at `display + 120`
  - links the object into CoreEG2's global display lists
  - asks the registered device-path property database (`qword_E658`) to parse the copied record into working state at `display + 104`
  - if any of that parser setup fails, it immediately destroys the object and never reaches the paired path

- `CoreEg2SetSignalForDisplay`
  - signal `11` is the initial complex-display candidate path
  - signal `8/9` is the paired internal path family that actually makes the complex state live
  - the `8/9` branch scans the stored connector/descriptor data for an Apple-specific 18-byte sequence:
    - `00 00 00 01 00 06 10`
      embedded in the record block it walks
  - this branch populates the structured per-display property area and writes:
    - `[display + 600] = 0x100010000`
  - that is the store that makes:
    - `[display + 604] = 1`
  - only after that does later code treat the complex-display block as live input rather than as a candidate

- `sub_52F2` is more than a copy helper
  - it walks the parser-provided iterator/callback set from `display + 104`
  - matches timing descriptors against the connector records
  - populates the working display-setup block at `display + 416`
  - sets:
    - `display + 424`
    - `display + 428`
      from `sub_52AA(display)`
  - and repeatedly retries feasibility checks with `sub_4D21`
  - so this stage looks like:
    - timing/classification validation
    - not just blob assembly

- `sub_56A5`
  - is the timing admissibility helper used during that process
  - it accepts either:
    - a bounded timing range from the record itself
    - or a fallback ceiling of:
      - pixel clock `<= 0x23C34600`
      - width `<= 0x1000`
  - it logs:
    - `#[EDID:EG2:PR] %lu %u %u`

- success path inside `CoreEg2ComplexDisplayInit`
  - allocates a 384-byte complex-display blob and stores it at:
    - `display + 552`
  - fills it through `sub_52F2`
  - duplicates the validated setup block into two internal segments inside that blob
  - calls the parent backend with the prepared 384-byte block
  - performs two explicit link-training phases:
    - `START:CDLinkTrainDPConnector1`
    - `START:CDLinkTrainDPConnector2`
  - then activates the display, restores boot gamma/background/backlight, installs display protocols, and finally publishes:
    - `ComplexDisplaySetup`
      - size `261` / `0x105`

- one important activation detail:
  - `CoreEg2ConfigureAndActivateDisplayPipeline`
    passes:
    - `display + 600`
      as the third argument to the lower backend only when:
      - `display + 604 == 1`
  - so the complex-display state is not just metadata
  - it changes the actual lower pipeline activation call

- another important correction:
  - the function previously labeled `CoreEg2PublishDisplayPropertiesToSharedService`
    is better understood as:
    - a destroy/teardown path that also publishes the current complex-display properties before freeing the object
  - it is called from:
    - object-destruction paths
    - a display-protocol notify callback that tears down the object and re-enters generic init
  - so its `1265 payload=1` send should be read as part of a destroy/reload transition, not the primary success-side handoff

- current best high-level interpretation
  - `CoreEG2ComplexDisplayInit` is the preboot 5K split
  - success means:
    - paired connector discovered
    - Apple panel signature matched
    - connector/timing records validated
    - `display + 604` made live
    - 384-byte complex-display blob applied
    - both connectors link-trained
    - `ComplexDisplaySetup (0x105)` published
  - failure means:
    - `CoreEg2GenericInitializeGraphicsDevice`
      becomes the fallback path
    - which is the strongest static explanation so far for why plain unsupported boots can collapse to the ordinary internal display path

- one more important nuance after decompiling `CoreEg2GenericInitializeGraphicsDevice`:
  - the fallback path is not "do nothing"
  - it is a full generic restore/bring-up pipeline
  - it can:
    - honor `SkipNvramDisplay`
    - query `MSLD`
    - load saved config from the shared backend
    - restore a generic display state
    - retry link training with `sub_4A9A`
    - and then activate/install the ordinary display objects
  - so the current best model is:
    - successful complex path -> paired internal 5K-style preboot state
    - failed complex path -> still-initialized but generic single-display path
  - that matches the current Linux evidence better than a theory of:
    - "Linux gets no useful preboot display initialization at all"

### CoreEG2 Deep Dive: Connector Record Set And ComplexDisplaySetup Packing

- renamed helper functions from the latest pass:
  - `sub_4D99` -> `CoreEg2FetchConnectorRecordSet`
  - `sub_523C` -> `CoreEg2ValidateConnectorRecordSet`
  - `sub_56A5` -> `CoreEg2IsTimingCompatibleWithConnectorRecord`
  - `sub_126B` -> `CoreEg2ReadPciConfigDword`
  - `sub_63D9` -> `CoreEg2FetchPlatformRecordByPciId`

- `CoreEg2FetchConnectorRecordSet`
  - fetches connector state from the AMD connector backend using opcode `160`
  - returned layout is:
    - object header:
      - dword total size
      - qword pointer to contiguous record bytes
    - contiguous record bytes:
      - first record is always `128` bytes
      - byte `126` of the first record is a record count
      - additional chained records are fetched in `128`-byte increments
  - total size is:
    - `128 + (record_count << 7)`
  - the validator requires:
    - total size is a multiple of `0x80`
    - first 8 bytes are not the sentinel `00 FF FF FF FF FF FF 00`

- important practical implication:
  - the "connector blob" consumed by `CoreEG2` is not just one EDID block
  - it is a record set with a primary 128-byte record plus optional chained records
  - `CoreEG2` later copies the whole set into the display object and mines it for:
    - Apple panel signature
    - timing classes
    - complex-display candidate information

- `CoreEg2SetSignalForDisplay` exact complex-path trigger
  - the paired-path branch is selected by:
    - `(signal & ~1) == 8`
  - that branch scans the stored record set at:
    - `display + 152`
  - it walks 18-byte descriptors starting at offset `0x36`
  - the descriptor must contain:
    - `00 00 00 01 00 06 10`
  - only then does it populate the complex property block and write:
    - `[display + 600] = 0x100010000`
    - `[display + 608] = display + 560`
    - `[display + 584] = 7`
  - this is the concrete point where:
    - `display + 604 == 1`
      becomes true

- property publication performed by the `8/9` branch
  - once the branch is taken, `CoreEG2` pushes a structured property set through the shared backend
  - published keys include:
    - `LinkType`
    - `D`
    - `DataJustify`
    - `Dither`
    - `PixelFormat`
    - `InverterFrequency`
  - this confirms the complex path is not just a boolean gate
  - it exports a richer per-display configuration block before activation

- `ComplexDisplaySetup` packet packing in `CoreEg2ComplexDisplayInit`
  - before publishing, the stack buffer is filled with `0xAA`
  - dword `0` is then set to version-like value:
    - `1`
  - the packet then packs:
    - primary connector id
    - secondary connector id
    - selected lane / rate style fields from `display + 340 / +344`
    - display-configuration flag derived from `qword_E6D0 + 12 == 129`
    - the working setup block from:
      - `display + 352 .. +404`
    - and a large tail copied from the 384-byte complex-display blob at:
      - `display + 552`
  - published key:
    - `ComplexDisplaySetup`
  - size:
    - `261` / `0x105`

- current best read of `ComplexDisplaySetup`
  - it is the success-side firmware handoff packet for the paired internal display
  - it carries:
    - connector identities
    - selected link/training parameters
    - display pipeline setup fields
    - complex-display blob contents
  - this is probably the single richest preboot artifact to compare against any future Linux instrumentation

### Early CoreEG2 Gate: SMC Query + Boot-Time Policy Bitfield

- `CoreEg2GenericInitializeGraphicsDevice` contains the tightest early branch we have found so far:
  - it reads `dword_E670`
  - then calls `sub_74C1()`
  - and does:
    - if `sub_74C1()` returns nonzero:
      - continue
    - else if `(dword_E670 & 0x10) != 0`:
      - jump to the fallback/abort path at `loc_9668`

- this is currently the best candidate for the "approved boot path keeps 5K-capable state, unsupported boot falls back" branch

- `sub_74C1()` is not a generic display-capability probe
  - it uses the latched SMC protocol pointer `qword_E628`
  - it issues:
    - `SMC Get`
    - key `0x4D534C44`
    - ASCII big-endian: `MSLD`
  - it copies back at most one byte and returns true only when that byte equals `1`

- so the early gate is effectively:
  - if `SMC["MSLD"] == 1`:
    - continue
  - else if module-entry policy bit `0x10` is set:
    - take the fallback path

- `qword_E628`
  - is the SMC-backed provider used by `sub_74C1()`
  - it is latched by `CoreEg2RegisterUiThemeProtocol`
  - so this part of the decision is Apple platform-state driven, not plain EDID or connector-state driven

- `dword_E670`
  - is written exactly once at module entry in `CoreEg2ModuleEntryInstallDisplayProtocols`
  - it is not read from NVRAM directly
  - the module fetches it through helper `sub_78F0` from a provider interface at `qword_E660`
  - the queried selector GUID decodes to:
    - `1823A1A6-0EAF-4F3A-93D3-BF61F5EEFA0D`
  - after entry, `dword_E670` is only read by:
    - `CoreEg2GenericInitializeGraphicsDevice`
    - `CoreEg2ComplexDisplayInit`
    - `CoreEg2SetSignalForDisplay`

- refined provider chain for `dword_E670`
  - `CoreEg2` module entry walks a 6-entry protocol-registration table
  - for each entry it tries to `LocateProtocol`
  - if found immediately, it calls the matching register callback
  - otherwise it registers a notify event and will call that callback later when the protocol appears
  - the relevant entry for `qword_E660` is:
    - provider GUID `AC5E4829-A8FD-440B-AF33-9FFE013B12D8`
    - register callback `CoreEg2RegisterMlbLedProtocol`
  - the callback name is misleading
    - the important fact is the GUID, not the string label
  - this same GUID was already traced earlier to:
    - `ApplePlatformInfoDatabaseDxe`
  - so the best current interpretation is:
    - `dword_E670` ultimately comes from the Apple platform-info database provider
    - not from SMC
    - and not from raw EDID

- refined provider chain for `sub_74C1()`
  - `sub_74C1()` uses `qword_E628`
  - `qword_E628` is latched from the protocol-registration entry whose provider GUID is:
    - `APPLE_SMC_IO_PROTOCOL_GUID`
  - so the `MSLD` check is a real Apple SMC/platform-state query

- strongest current interpretation:
  - `sub_74C1()` asks Apple SMC whether a specific preboot/platform state (`MSLD`) is active
  - `dword_E670` is a boot-time CoreEG2 policy/config bitfield
  - if Linux boots through a path that leaves `MSLD != 1` while also setting or inheriting the wrong `dword_E670` bits, `CoreEG2` can fall off the complex-display path before the paired internal setup is even attempted

- first Linux `MSLD` probe status
  - report file:
    - `iMac_info/msld_probe_20260321_101612.txt`
  - results:
    - `applesmc` is loaded
    - `dmesg` shows `applesmc: key=789 fan=1 temp=108 index=100 acc=0 lux=2 kbd=0`
    - but no applesmc key-index sysfs directory was found at the initial expected path
  - interpretation:
    - this run does **not** prove `MSLD` is absent
    - it only proves the first probe script did not find the sysfs key-enumeration interface on this kernel/path layout
  - action:
    - use the updated `scripts/probe_msld_linux.sh`, which now searches the key interface more broadly under `/sys` and also checks `APP0001:*`

- second Linux `MSLD` probe result
  - report file:
    - `iMac_info/msld_probe_20260321_102147.txt`
  - confirmed runtime findings on plain Linux boot:
    - applesmc sysfs path:
      - `/sys/bus/acpi/devices/APP0001:00`
    - `key_count = 789`
    - `MSLD` found at key index `332`
    - type:
      - `ui8`
    - length:
      - `1`
    - raw data:
      - `00`
    - unsigned byte value:
      - `0`
  - interpretation:
    - on the current plain Linux boot, the Apple SMC state queried by `CoreEG2::sub_74C1()` is definitively:
      - `MSLD = 0`
    - this is one of the strongest confirmed differences between the observed Linux runtime state and the firmware-side check used by `CoreEG2`
  - important nuance after combining this with the `dword_E670` work:
    - if the current `iMac19,1 -> j138 -> dword_E670 = 1` mapping is correct, then:
      - `(dword_E670 & 0x10) == 0`
    - therefore the exact early branch:
      - `if !sub_74C1() && (dword_E670 & 0x10) != 0`
      - would **not** fall through to the fallback path solely because `MSLD` is `0`
  - practical conclusion:
    - `MSLD = 0` on plain Linux is real and important
    - but it is probably **not** by itself the direct reason the iMac ends up on the 4K path
    - this pushes the highest-value remaining suspects later in the pipeline:
      - second internal connector discovery

- `LidPoller.efi` follow-up on `MSLD`
  - live IDA/MCP analysis on:
    - `RE_Files/FirmwarePE32_All/LidPoller.efi`
  - `LidPoller` contains two `MSLD` (`0x4D534C44`) key references in the same polling routine:
    - function:
      - `sub_12BF`
  - `sub_12BF` behavior:
    - checks that the module's SMC/protocol handle is present
    - raises TPL
    - queries key metadata via method `qword_20F8 + 48`
    - if the key length is `<= 1`, reads the key value via method `qword_20F8 + 8`
    - copies back the first data byte and treats value `1` as the active state
    - then calls `sub_1596(v5 != 1)` to notify registered listeners
  - `sub_120F` behavior:
    - stores the protocol pointer into `qword_20F8`
    - creates a timer event that repeatedly calls `sub_12BF`
  - `sub_1596` behavior:
    - walks a callback/listener list
    - invokes enable/disable style callbacks depending on whether `MSLD == 1`
  - conclusion:
    - `LidPoller.efi` is a **consumer/poller** of `MSLD`
    - it does **not** currently look like the module that sets the key
  - updated likely-setter ranking:
    - strongest remaining candidate:
      - `RE_Files/FirmwarePE32_All/AppleSmc.efi`
    - possible secondary candidate:
      - `RE_Files/FirmwarePE32_All/52C05B14-0B98-496C-BC3B-04B50211D680.efi`
    - why:
      - `AppleSmc.efi` contains `MSLD` and the Apple SMC protocol implementation context
      - `52C05...efi` contains `MSLD` but does not appear to reference `APPLE_SMC_IO_PROTOCOL_GUID` directly, so it currently looks more like a higher-level consumer/helper than the actual protocol owner
      - Apple panel signature match
      - complex-display blob construction / publish
      - and the later runtime reassertion that Windows performs with opcode `1265` payload `1`

- refined database-side decode of the `dword_E670` selector
  - reopening `ApplePlatformInfoDatabaseDxe` clarified that its exported APIs (`GetPlatformInfoBlob`, `GetPlatformInfoStringByIndex`, `MatchPlatformInfoBlob`) all funnel through `sub_14F6`
  - `sub_14F6` is table-driven:
    - top-level records are `GUID + offset + size`
    - the selected record points to a secondary table of policy entries
    - the final returned pointer/size depends on a small mask/value match plus platform override selection
  - the selector GUID used by `CoreEG2`:
    - `1823A1A6-0EAF-4F3A-93D3-BF61F5EEFA0D`
    - does **not** appear as code in `ApplePlatformInfoDatabaseDxe`
    - it appears in the raw firmware/HOB data in `imac19_1_spi_1.bin`
  - in the raw BIOS-region dump, that selector's top-level record is:
    - selector GUID `1823A1A6-0EAF-4F3A-93D3-BF61F5EEFA0D`
    - offset `0x7A0`
    - size `0x30`
  - following that offset resolves to a 48-byte secondary table containing exactly two 24-byte entries:
    - token `j139`
      - mask/value field `0xFFFFFFFF00000000`
      - data size `4`
      - returned dword `5`
    - token `j138`
      - mask/value field `0xFFFFFFFF00000000`
      - data size `4`
      - returned dword `1`
  - strongest current interpretation:
    - `dword_E670` is not an arbitrary blob and not a universal constant
    - it is a small platform-selected 32-bit policy value coming from the Apple platform-info database
    - for this selector, the platform database currently exposes two known candidate values:
      - `5`
      - `1`
  - stronger token-to-model mapping from the raw BIOS-region data:
    - in the same platform database block, `j139` is paired with UTF-16 string `iMac19,2`
    - `j138` is paired with UTF-16 string `iMac19,1`
    - this is the strongest current evidence that the selector table is model-token keyed, not arbitrary
  - practical conclusion for the target machine:
    - target system identity is already confirmed as `iMac19,1`
    - so the most likely resolved token is `j138`
    - and the most likely `CoreEG2` policy value is therefore:
      - `dword_E670 = 1`
  - remaining caution:
    - this is still inferred from the platform database content, not observed live at runtime
    - a later board-variant override is still theoretically possible, but the current evidence strongly favors `iMac19,1 -> j138 -> 1`

## AMD GOP Deep Dive: How Ordinary DP Records Become `DP_INT`

- live IDA/MCP pass on:
  - `RE_Files/FirmwarePE32_All/F0140FC0-EC2A-451E-ADC5-D671AC453A8C.efi`
- key producer-side functions:
  - `GopReadPagedCommand160` at `0x1CC00`
  - `GopRefreshConnectorRecordCacheByMask` at `0x1A5D0`
  - `GopPopulateConnectorRecordCaches` at `0x1A870`
  - `GopCopyConnectorRecordSetByMask` at `0x1A9E0`
  - `GopBuildDisplayConnectFlags` at `0x14860`
  - `GopProbeConnectorRecordCachesFromDisplayTable` at `0x9A20`
  - `GopApplySavedDisplayConfiguration` at `0x9E3B`
  - `GopPublishPrimaryDisplayOutputs` at `0x5030`
  - `GopPublishDetectedDisplayOutputs` at `0x1EBB0`

### `opcode 160` is a paged record fetch, not a simple scalar mailbox read

- `GopGraphicsConnectorProtocolTransfer` and `GopGraphicsConnectorBackendDispatchOpcode` both special-case `opcode 160`
- instead of the normal low-byte connector command path, `160` is routed into:
  - `GopReadPagedCommand160`
- `GopReadPagedCommand160` behavior:
  - selects the connector register bank
  - reads the first page
  - treats byte `126` of the first page as the continuation count / “more pages” indicator
  - reads subsequent windows in `0x100`-byte chunks
  - uses low-byte commands to page-switch between the first page and later pages
- interpretation:
  - the important connector state used by the 5K preboot path is a structured paged record set, not a one-byte capability bit

### The GOP explicitly caches per-mask connector record sets

- `GopPopulateConnectorRecordCaches` pre-populates ordinary page caches and then forces refresh for the DP-family masks:
  - `0x100`
  - `0x200`
  - `0x300`
  - `0x400`
- `GopRefreshConnectorRecordCacheByMask`:
  - maps each DP-family mask to a connector-state slot and a “cache valid” byte
  - calls `GopReadPagedCommand160`
  - records success/failure into:
    - bytes `a1 + 18920 .. 18923`
    - bytes `a1 + 2309 .. 2312`
- `GopCopyConnectorRecordSetByMask`:
  - copies the first `0x80` bytes of the cached record set to the caller
  - if byte `126` is nonzero and the caller buffer is large enough, copies the next `0x80` bytes too
  - latches a special panel-identity marker when bytes `8..9 == 0x0610`

### `GopBuildDisplayConnectFlags` is the promotion step

- `GopBuildDisplayConnectFlags` does not simply inspect one live connector and declare success
- it:
  - ensures the paged connector caches are populated
  - iterates candidate masks from the internal table at `unk_2C050`
  - maps each mask to an output slot
  - fetches the cached record buffer and validity flag for that slot
  - compares the record set against stored descriptor/timing families
  - builds a display-configuration block only when one family matches
- important internal helpers/fields:
  - `sub_14240()` aggregates the available DP-family seeds from bytes `a1 + 18920 .. 18923`
  - `sub_147B0()` compares a stored selector against the current candidate-mask table
  - `sub_1A080()` returns the per-slot display-config storage block used to materialize the selected result
- important outputs when a match succeeds:
  - writes the per-slot display-config block
  - latches multiple display/link/pixel-format fields from the selected descriptor
  - calls `GopPackDisplayConfigFromTimingRecord`
  - later publication code consumes the resulting current output type

### The grouped internal path is real on the GOP side, not just in `CoreEG2`

- `GopGraphicsConnectorBackendApplyOutputMask` at `0x8BA0` makes the promotion explicit:
  - it reads the active connector mask from `display+384`
  - if `(mask & 0xF00) == 0xF00`, it sets the current published type to:
    - `3840`
  - then it activates connector state for mask:
    - `0xF00`
- `GopPublishPrimaryDisplayOutputs`:
  - if the current display type becomes `3840`, it:
    - activates connector state by mask `0xF00`
    - publishes:
      - `ATY,EFIDisplay = DP_INT`
- `GopPublishDetectedDisplayOutputs` performs the same publication for type `3840`
- interpretation:
  - the special internal path is represented in firmware as:
    - grouped mask `0xF00`
    - current display type `3840`
    - published identity `DP_INT`

### Saved-config replay also treats `0xF00` as special mode type `11`

- `GopProbeConnectorRecordCachesFromDisplayTable` walks saved display-table entries
- when a saved entry has:
  - mode type `11`
  - and a matching shared-graphics-protocol record fetch succeeds
- it marks the corresponding connector-mask availability byte using `sub_1A160()`
- `sub_1A160()` maps:
  - `0x100 -> a1 + 18920`
  - `0x200 -> a1 + 18921`
  - `0x300 -> a1 + 18922`
  - `0x400 -> a1 + 18923`
  - `0x500 -> a1 + 18924`
  - `0x600 -> a1 + 18925`
  - `0xF00 / 3840 -> a1 + 18926`

- `GopApplySavedDisplayConfiguration` then makes the grouped-mask semantics explicit:
  - after `GopBuildDisplayConnectFlags`, it writes the current selected output mask back into the saved display-table entry
  - when the current output mask contains `0xF00`, it chooses:
    - mode type `11`
  - when it contains `0x10`, it chooses:
    - mode type `8`
  - when it contains `0xF083`, it chooses:
    - mode type `10` or `6`

- interpretation:
  - on the AMD GOP side, mode type `11` is the saved-config representation of the grouped internal `0xF00` path
  - this lines up with `CoreEG2ComplexDisplayInit`, which also treats mode type `11` as the complex internal-display candidate/root

### Current best firmware-side model after combining GOP + `CoreEG2`

- the GOP first probes and caches ordinary DP-family connector record sets
- `GopBuildDisplayConnectFlags` promotes a subset of those cached records into a richer display-config result when the stored descriptor family matches
- that richer result can become:
  - grouped output mask `0xF00`
  - current type `3840`
  - `ATY,EFIDisplay = DP_INT`
  - saved-config mode type `11`
- `CoreEG2` then consumes this richer state as the “complex internal display” candidate path
- if later paired-display conditions succeed, `CoreEG2` builds the complex-display blob, drives both connectors, and sends `1265 = 1`
- if not, firmware can still fall back to the generic single-display path

### Why this matters for Linux

- this strengthens the idea that Linux may boot with a valid but generic eDP state even when the firmware/GOP never fully promotes the internal panel into the grouped `0xF00 / 3840 / DP_INT / mode-type-11` path
- it also means that simply reproducing a Windows runtime opcode without the earlier firmware-side grouped-state promotion is unlikely to be sufficient

### Low-level connector-slot state behind the grouped path

- the paged connector record set is only one layer
- beneath it, the GOP also maintains a per-mask connector-state slot filled by:
  - `sub_CCA0`
- `sub_CCA0` behavior:
  - reads sparse extended opcode windows into `slot + 12`
  - special windows observed:
    - `0x100` size `0x0A`
    - `0x240` size `0x09`
    - `0x600` size `0x01`
  - on success, if:
    - `slot[25] != 0`
    - and current output type is `3840`
  - then it sets:
    - `a1 + 18932 = 1`

- `sub_8510` shows how the grouped path depends on that slot state:
  - it always binds the connector-state slot for the current grouped mask
  - it derives preferred link settings from slot fields:
    - `slot + 13`
      - link-rate multiplier
      - used as `2700 * slot[13]`
    - `slot + 14`
      - lane count in low 5 bits
      - extra capability/flag in bit `0x80`
  - for grouped path `0xF00`, if:
    - `slot[25] != 0`
  - then the GOP enables a second special gate:
    - `a1 + 18932 = 1`

- once that grouped-path gate is active, `sub_8510` inspects an extra secondary sub-block inside the connector-state slot:
  - chosen by bit `slot[17] & 1`
  - offset:
    - `+0x400` or
    - `+0x500`
  - required byte pattern in that sub-block:
    - `[+12] == 0`
    - `[+13] == 16`
    - `[+14] == 250`
  - if the pattern matches, the GOP latches:
    - `a1 + 18952 = slot[25] & 1`

- interpretation:
  - the grouped internal path is not accepted solely because the top-level record mask says `0xF00`
  - the GOP also requires a valid low-level connector-state snapshot for a secondary sub-block associated with the grouped path
  - this is strong evidence that the paired 5K path depends on extra per-segment/per-connector low-level state, not just one promoted connector identity

- additional grouped-path consequences:
  - `GopAdjustViewportForTiledDisplayLayout` halves width and/or height when multiple segments are active and programs adjusted display windows
  - `sub_1D980` has a dedicated branch for `a1 + 716 == 0x4000`
    - it computes a midpoint split
    - programs display registers around that split
    - and then commits the new configuration

- current interpretation of the two layers together:
  - paged connector record set:
    - higher-level panel/topology identity consumed by `CoreEG2`
  - connector-state slot:
    - low-level link/training/paired-segment state consumed by the AMD GOP backend

## CoreEG2 Deep Dive: What `signal 8/9` Actually Reads From The Record Set

- key functions:
  - `CoreEg2FindOrCreateDisplayByConnector` at `0x395D`
  - `CoreEg2CreateDisplayObject` at `0x29E5`
  - `CoreEg2SetSignalForDisplay` at `0x3B3E`
  - `CoreEg2IsTimingCompatibleWithConnectorRecord` at `0x56A5`
  - `CoreEg2ComplexDisplayInit` at `0x7A5B`
  - `CoreEg2GenericInitializeGraphicsDevice` at `0x8D08`
  - `sub_98A3` at `0x98A3`

### `CreateDisplayObject` does not interpret the Apple-specific fields yet

- `CoreEg2CreateDisplayObject`:
  - validates the fetched connector record set with `CoreEg2ValidateConnectorRecordSet`
  - allocates the 616-byte display object
  - copies the record bytes into:
    - `display + 152`
  - stores the record-set size into:
    - `display + 144`
- interpretation:
  - the fetched record set becomes the high-level source of truth for later signal/panel interpretation
  - Apple-specific meaning is applied later, not during object creation

### `signal 11/12` and `signal 8/9` are different phases, not just different names

- `CoreEg2ComplexDisplayInit` forces:
  - `CoreEg2SetSignalForDisplay(display, 11)`
  - on the primary complex-display candidate
- `CoreEg2FindOrCreateDisplayByConnector` passes the connector-node signal value directly into:
  - `CoreEg2SetSignalForDisplay(display, signal)`
- so:
  - `11/12` is the complex-display candidate/root family
  - `8/9` is a later connector-role family already attached to live connector nodes

### The `signal 8/9` branch scans a specific 18-byte descriptor family

- in `CoreEg2SetSignalForDisplay`, the branch:
  - `(signal & ~1) == 8`
  - is the only branch that makes the complex-display state live
- it scans `display + 152` starting at offset:
  - `0x36`
- step size:
  - `0x12` / 18 bytes
- required byte pattern at the start of the descriptor:
  - `00 00 00 01 00 06 10`
- if no such 18-byte descriptor is found before offset `0x7E`, the branch fails

- once the descriptor is found, `CoreEg2SetSignalForDisplay` decodes fields from it:
  - byte `+7`
    - bit `0`
      - feeds `display + 564` (`LinkType` publish side)
    - bit `4`
      - feeds `display + 565` (`D`)
    - bit `5`
      - feeds `display + 567` (`DataJustify`)
  - byte `+8`
    - bit `0`
      - feeds `display + 568` with `6` or `8`
    - bit `4`
      - feeds `display + 569` with `6` or `8`
  - byte `+9`
    - bit `3`
      - feeds `display + 566` (`Dither`)

- it then writes:
  - `display + 600 = 0x100010000`
  - `display + 608 = display + 560`
  - `display + 584 = 7`
  - allocates `display + 592` for a 28-byte property block
- this is the store that implicitly makes:
  - `display + 604 = 1`

### The `signal 8/9` branch publishes a structured property bundle, not just a mode bit

- after decoding the descriptor, `CoreEg2SetSignalForDisplay` publishes properties named:
  - `LinkType`
  - `D`
  - `DataJustify`
  - `Dither`
  - one unnamed boolean under `asc_D94A`
  - `PixelFormat`
  - `InverterFrequency`
- interpretation:
  - `signal 8/9` is a descriptor-driven property bundle for the paired-display path
  - it is not merely a raw “enable 5K” toggle

### Timing compatibility is checked against connector-record metadata

- `CoreEg2IsTimingCompatibleWithConnectorRecord` compares the candidate timing against fields in the connector record:
  - record qword range at offsets `+8 .. +16`
  - mode/type field at offset `+24`
- special cases:
  - if record type is `2`, the candidate timing must fall within the qword range
  - if record type is `0`, firmware accepts only a narrower default timing envelope
- interpretation:
  - the paged connector record set does not just identify the panel
  - it also carries timing-compatibility metadata used during complex-display blob construction

### Where `signal 8/9` comes from

- `CoreEg2GenericInitializeGraphicsDevice` and `sub_98A3` both walk controller/connector lists
- they call:
  - `CoreEg2FindOrCreateDisplayByConnector(controller, *(connector_node + 4))`
- `CoreEG2` therefore consumes a pre-existing role/type field from each connector node; it does not invent that value inside the display object
- new controller-side clarification:
  - `sub_5C01` is the controller-node constructor on the `CoreEG2` side
  - it opens the private connector protocol (`unk_E350`) on the handle being attached
  - it reads:
    - connector entry count from `protocol + 24`
    - connector entry pointer array from `protocol + 32`
  - it then wraps each imported entry into a 32-byte child node
  - so the value later passed to `CoreEg2FindOrCreateDisplayByConnector(..., *(connector_node + 4))` is really the imported GOP descriptor field at `entry + 4`
- this moves the true role assignment point out of `CoreEG2` and back into the AMD GOP private connector protocol

### Raw GOP connector seeds versus replay-selected roles

- `GopInitializeGraphicsConnectorObject` (`0x6660`) constructs the per-connector entry array that `CoreEG2` later imports through the private protocol
- each entry is a 32-byte record rooted at `connector + 104`, and the role/type field is written at `entry + 4` (`connector + 32*n + 108`)
- static seed mapping from the raw connector mask:
  - if `(mask & 0x10) != 0`, create a role `8` entry
  - if `(mask & 0xF00) != 0`, create a role `11` entry
  - if `(mask & 0xF083) != 0` and `(mask & 0x20) == 0`, create additional role `10` and role `6` entries
- connector indices on this iMac-class Polaris layout seed masks:
  - connector `1` -> `0x120`
  - connector `2` -> `0x220`
  - connector `3` -> `0x301`
    - connector `5` -> `0x402`
- important consequence:
  - the raw connector initializer clearly seeds `8`, `11`, `10`, and `6`
  - `9` is still not a direct raw seed in the static iMac path recovered so far

### Where role `8` shows up

- `GopSelectConnectorMaskFromSavedState` and `GopApplySavedDisplayConfiguration` are the clearest places where role `8` is selected later
- in `GopApplySavedDisplayConfiguration`:
  - if the selected output mask has bit `0x10`, the chosen role is `8`
  - if the selected output mask has `0xF00`, the chosen role is `11`
  - if the selected output mask has `0xF083`, the chosen role is `10` or `6`
- interpretation:
  - role `8` currently looks replay-driven / selected-mode-driven, not like a raw physical connector seed
  - this is a strong hint that the `CoreEG2` `signal 8/9` gate may correspond to a replayed logical paired-display role layered on top of the raw GOP connector seeds
- remaining unknown:
  - where role `9` is introduced, or whether `CoreEG2` simply supports the `8/9` family while the iMac 5K path primarily lands on `8`

### Direct bridge: `CoreEG2` generic restore path calls the GOP saved-display apply method

- `CoreEg2GenericInitializeGraphicsDevice` does not just enumerate connectors blindly
- it first invokes the controller private protocol method at offset `+0x38`
- this method now lines up with the GOP device protocol template:
  - template base `unk_2B710`
  - `+0x38` -> `GopApplySavedDisplayConfiguration`
  - `+0x40` -> `GopReplaySavedDisplayConfiguration`
- this is the strongest recovered bridge so far between Apple `CoreEG2` and the AMD GOP replay path
- practical consequence:
  - `CoreEG2` asks the GOP to apply the saved display configuration first
  - only after that does it read the selected connector entry and call:
    - `CoreEg2FindOrCreateDisplayByConnector(..., entry->role)`
- this makes the current best model much tighter:
  - raw role `8` exists in the connector entry array for mask `0x10`
  - `GopApplySavedDisplayConfiguration` selects role `8` whenever the chosen output mask has bit `0x10`
  - `CoreEG2` can then immediately consume that selected raw role `8`
  - role `9` still looks more like a later derived grouped-state encoding than the primary initial 5K entrypoint

### `CoreEG2` treats `8/9` as one family and `11/12` as another

- `CoreEg2FindOrCreateDisplayByConnector` does **not** fetch connector records based on the full raw role
- it collapses the input role to:
  - `a2 - 11 < 2`
- practical meaning:
  - `11/12` share one record-fetch / display-object family
  - all other roles share the other family
- later, `CoreEg2SetSignalForDisplay` also does not distinguish `8` from `9` at the main paired-display gate
- it uses:
  - `(signal & ~1) == 8`
- practical meaning:
  - `8` and `9` are treated as one logical family for the descriptor-driven paired-display activation branch
  - so far, the recovered code does not give `9` a distinct setup path from `8`
- interpretation:
  - `11/12` look like the complex-display candidate/root family
  - `8/9` look like the paired-display activation family
  - among those, `8` is currently the strongest concrete live entrypoint because it exists as a raw GOP seed and is selected by the saved-display apply path

### Important correction: `CoreEg2ComplexDisplayInit` itself only calls `SetSignal(..., 11)`

- direct xrefs to `CoreEg2SetSignalForDisplay` are only:
  - `CoreEg2FindOrCreateDisplayByConnector`
  - `CoreEg2ComplexDisplayInit`
- in `CoreEg2ComplexDisplayInit`, the direct call is:
  - `CoreEg2SetSignalForDisplay(display, 11)`
- that means the complex constructor itself enters the **candidate/root** family directly, not the `8/9` activation family
- separately:
  - the `display + 600 = 0x100010000` store that makes `display + 604 = 1`
  - appears only once, inside the `CoreEg2SetSignalForDisplay` `8/9` branch
- and `CoreEg2ConfigureAndActivateDisplayPipeline` only passes `display + 600` to the lower backend when:
  - `display + 604 != 0`
- interpretation:
  - the recovered preboot 5K path is now best modeled as **two-stage**
  - stage 1:
    - `CoreEg2ComplexDisplayInit` creates the role-`11` complex-display candidate/root object
    - builds the 384-byte complex blob at `display + 552`
    - performs second-connector setup and Apple panel validation
  - stage 2:
    - a later `8/9`-family display object or mode transition makes `display + 604 = 1`
    - only then can `CoreEg2ConfigureAndActivateDisplayPipeline` pass the complex-display header at `display + 600` into the lower backend
- practical consequence:
  - the preboot 5K path is not a single monolithic flag
  - it is a staged relationship between a role-`11` root/candidate object and a later role-`8/9` activation object/state

### Likely live boot-time sequence inside `CoreEG2`

- `CoreEg2GenericInitializeGraphicsDevice` first invokes the GOP controller method at `protocol + 0x38`
  - this lines up with `GopApplySavedDisplayConfiguration`
- after that, it does **not** iterate every connector first
  - it reads the controller protocol's currently selected entry index
  - then fetches that selected imported connector entry
  - then calls:
    - `CoreEg2FindOrCreateDisplayByConnector(..., selected_entry->role)`
- on the current recovered iMac 5K path, the strongest interpretation is:
  - the saved-display apply logic first selects the grouped `0xF00` / role-`11` candidate path
  - `CoreEG2` creates the role-`11` root object from that
- only after that initial selected-path handling does `CoreEg2GenericInitializeGraphicsDevice` fall through to:
  - `sub_98A3(controller, 1)`
- `sub_98A3` then walks the connector children and can create/find additional display objects from the remaining imported roles
- practical interpretation:
  - initial boot restore likely reconstructs the role-`11` root/candidate object first
  - later search/bring-up can discover the role-`8` activation-side object
  - this matches the staged `11 -> 8/9` model much better than any one-shot “5K enabled” flag

### The `6/8/10/11/12` family is also consumed later as grouped-layout state

- several lower pipeline helpers reuse the same numeric family after connector-role selection:
  - `sub_22070` maps current `a1 + 720` values:
    - `6 -> 1`
    - `8 -> 2`
    - `10 -> 3`
    - `11 -> 4`
    - `12 -> 5`
  - `sub_22140` (`0x22140`) uses `a1 + 720` to choose grouped-path register encodings when `a1 + 716 == 0x4000`
  - `sub_22520` (`0x22520`) encodes `10/12/16` into upper grouped-layout bits after calling `sub_22070`
  - `sub_1D980` (`0x1D980`) handles the special `a1 + 716 == 0x4000` path and converts divider-derived values `6/8/10/12` into grouped display-programming state
- interpretation:
  - the numeric family `6/8/10/11/12` is reused as a lower display-layout / grouped-path state encoding
  - this explains why the same values appear in both connector-role selection and later hardware programming code
  - it does **not** mean every one of those values is necessarily a raw connector seed in the private protocol array

### Derived grouped-state family: where `9` actually shows up

- `sub_17AB0` derives a small grouped-state selector from the currently selected connector mask stored at:
  - `device + 6928 + 4 * current_display_index + 2224`
- current mapping in `sub_17AB0`:
  - `0x100 -> 2`
  - `0x200 -> 3`
  - `0x300 -> 1`
  - `0x400 -> 5`
  - `0x1   -> 1`
  - `0x2   -> 5`
- `sub_17B90` then maps that grouped selector into the later numeric state family:
  - `1 -> 9`
  - `2 -> 10`
  - `3 -> 11`
  - `4 -> 12`
  - `5 -> 13`
  - default -> `3`
- interpretation:
  - `9` is now explained as a **derived grouped-state value**, not as a raw connector-entry seed written into the private connector protocol array
  - specifically, the current code derives `9` from selected mask family `0x300` or `0x1`
  - `10` derives from `0x100`
  - `11` derives from `0x200`
  - `13` derives from `0x400` or `0x2`
  - `12` is still present in the mapping table but is not obviously produced by the current `sub_17AB0` cases, so it may be reserved, model-specific, or produced by a path not yet recovered

### Current refined model

- AMD GOP:
  - builds and promotes paged connector record sets into grouped state (`0xF00 -> 3840 -> DP_INT`)
- AMD GOP connector initializer:
  - seeds raw per-connector role entries (`11`, `10`, `6`) in the private connector protocol array
- AMD GOP saved-state selection/apply:
  - can later select replay-driven logical roles such as `8`
- `CoreEG2`:
  - stores the fetched record set in the display object
  - uses `signal 11` to create a complex-display candidate
  - consumes connector-node role values imported from the GOP private protocol array
  - uses the `signal 8/9` family together with a specific 18-byte descriptor family in the record set to make the paired-display state live
- this means the preboot 5K path is gated by:
  - grouped internal connector promotion on the GOP side
  - replay/selection of the correct logical connector role on the GOP side
  - and a descriptor-driven property bundle on the `CoreEG2` side

### New structural clarification: `CoreEG2` only caches two display-object families per controller

- `CoreEg2FindOrCreateDisplayByConnector` does **not** key display objects by the full imported connector role
- it reduces the role to:
  - family `1` for `11/12`
  - family `0` for everything else
- it then looks for an existing display object by:
  - same controller node
  - same reduced family bit
- practical consequence:
  - `11/12` share one display-object bucket
  - `6/8/9/10/...` share the other bucket
- this strongly suggests:
  - `11/12` are the special/root complex-display family
  - later activation-side roles like `8/9` are expected to reuse a shared non-`11/12` display object rather than allocating one object per raw role

### New structural clarification: imported connector roles are sorted in descending order

- when `CoreEG2` imports the GOP private connector descriptors through protocol `unk_E350`, it wraps each descriptor in a 32-byte child node
- those child nodes are inserted into the controller list in **descending** order of `descriptor->role`
- practical effect on the recovered role set:
  - a connector with roles `11`, `10`, `8`, `6` is searched as:
    - `11 -> 10 -> 8 -> 6`
  - not in raw seed order
- this makes the role-`11` family the first-class firmware path even more clearly:
  - the search walkers see `11` before any later paired-display activation-side role such as `8`

### New structural clarification: the two role families fetch connector state through different backends

- `CoreEg2FetchConnectorRecordSet(controller, family)` splits the fetch path by the reduced family bit:
  - family `1` (`11/12`)
    - uses the **secondary AMD backend**
    - performs the `1536` handshake first
    - then fetches paged connector data with opcode `160`
  - family `0` (everything else)
    - uses the **direct connector protocol method** at interface offset `+0x28`
    - fetches connector data with opcode `160`
- practical consequence:
  - the `11/12` family is not just a later logical label
  - it is already tied to the special secondary-backend discovery lane in preboot

### New 5K-relevant latch: `byte_E5E0`

- during `CoreEg2FetchConnectorRecordSet(..., family=1)`, if the `1536` handshake succeeds and the backend type is not `1`, firmware sets:
  - `byte_E5E0 = 1`
- current interpretation:
  - this is a global latch that says the special secondary-backend / complex-display discovery lane was reached successfully
  - it is **not** set by the generic direct connector path

### New 5K-relevant fallback/rescue path: `sub_8C38`

- `sub_8C38` only runs when:
  - `byte_E5E0 == 1`
- both `sub_5968` and `sub_5C01` call:
  - `CoreEg2ComplexDisplayInit(controller)`
  - then `sub_1301()`
- if `CoreEg2ComplexDisplayInit` succeeds:
  - they return success immediately
- if `CoreEg2ComplexDisplayInit` fails:
  - and `dword_E540 == dword_E544` after `sub_1301()`
  - they call `sub_8C38()`
- this means:
  - `sub_8C38` is **not** the primary success path
  - it is a delayed rescue/retry path that only exists after firmware already reached the special `11/12` lane once

### What `sub_1301()` is counting

- `sub_1301()` lazily initializes `dword_E540`
- it enumerates `EFI_PCI_IO_PROTOCOL` handles
- it counts display-class PCI functions matching the Apple-specific condition recovered in that loop
- then increments `dword_E544` each time the caller finishes a `CoreEg2ComplexDisplayInit` attempt
- practical meaning:
  - `dword_E540 == dword_E544` is a barrier:
    - all expected controller-start attempts have completed
  - only then does firmware trigger the delayed `sub_8C38` rescue path

### New working model for preboot sequencing

- primary path:
  - imported role list is searched with `11` first
  - `CoreEg2ComplexDisplayInit` attempts the complex paired-display path
  - this is the main 5K-capable preboot lane
- if the special lane was reached far enough to set `byte_E5E0`, but complex init still fails:
  - firmware waits until all controller-start attempts finish
  - then runs `sub_8C38`
  - `sub_8C38` performs a delayed `sub_98A3(..., 0)` search/activate loop
- interpretation:
  - Apple firmware has both:
    - a direct complex-display constructor path
    - and a special delayed rescue path after partially successful complex-display discovery
- this makes the preboot 5K logic look more like a staged state machine than a one-shot mode decision

### Important new correction: complex-init failure falls directly into generic init

- in the failure tail of `CoreEg2ComplexDisplayInit`:
  - firmware logs `GenericComplexDisplayFailureInfo-%d`
  - sends `1265` with payload `0`
  - performs local cleanup of the failure-side temporary buffers
- then, on error exit, it does **not** simply return the failure to the caller
- instead, it tail-jumps directly to:
  - `CoreEg2GenericInitializeGraphicsDevice(controller)`
- practical consequence:
  - the firmware fallback chain is now:
    1. try the complex paired-display path (`CoreEg2ComplexDisplayInit`)
    2. on failure, immediately fall into generic init on the same controller
    3. only later, if the special backend latch was set and all controller attempts have completed, use the delayed `sub_8C38` rescue path
- interpretation:
  - `CoreEg2GenericInitializeGraphicsDevice` is not an unrelated alternate path
  - it is the **immediate fallback continuation** of failed complex-display bring-up
  - the delayed `sub_8C38` path is a second-tier recovery layer on top of that
- informed inference:
  - because the complex-display constructor links the display object before this failure tail
  - and the traced failure tail does not show a display-object destroy
  - `CoreEg2GenericInitializeGraphicsDevice` likely re-enters with the role-`11` candidate object still present in the global display list
  - this makes the fallback look like a reuse/continuation path rather than a full restart from scratch

### New sequencing detail: search iteration naturally falls from one display-object family to the next

- `sub_98A3` marks the chosen display object by setting:
  - `display + 16 = 1`
- if setup later fails, `CoreEg2TearDownActiveDisplaySequence`:
  - clears `display + 408`
  - clears the linked controller-child backpointers
  - but does **not** clear `display + 16`
- practical consequence:
  - once `sub_98A3` has returned a given display object once, later search iterations skip that same object
  - after a failed setup/teardown, the next search pass naturally moves on to another display object/family instead of immediately retrying the same one
- interpretation:
  - this makes the generic fallback loop look intentionally staged:
    - try one display-object family
    - if it fails, tear it down
    - then advance to the next unvisited family
- informed inference:
  - if the first search-side attempt reuses the role-`11/12` candidate family and fails, later iterations can naturally fall through to the shared non-`11/12` family, where role `8/9`-style activation becomes plausible

### New consequence: the role-`10` gate likely decides whether the non-`11/12` family ever reaches role `8`

- imported connector roles are searched in descending order
- the shared non-`11/12` display-object bucket covers:
  - role `10`
  - role `8`
  - and the other non-`11/12` values
- therefore, on a connector that exposes both `10` and `8`:
  - `10` is encountered first
- `sub_98A3` contains a special extra backend check only for role `10`
- practical consequence:
  - if role `10` is accepted first, the shared non-`11/12` bucket is consumed before a later role `8` entry can claim it
  - if role `10` is rejected by that extra backend check, search can continue and the same non-`11/12` bucket may later be claimed under role `8`
- interpretation:
  - the role-`10` gate is likely one of the most important remaining static decision points for understanding whether firmware eventually lands on the role-`8/9` activation-side path

### New clarification: the role-`10` gate looks like a DP-to-HDMI adapter path, not the internal 5K path

- `sub_98A3` only applies the extra backend check to role `10`
- that check issues backend opcode:
  - `0x80 / 128`
- if the reply length is nonzero, the returned bytes must match the 16-byte signature at `unk_E120`
- the recovered bytes at `unk_E120` start with:
  - `DP-HDMI ADAPTOR`
- practical consequence:
  - role `10` now looks much more like a DP-to-HDMI adapter-specific path than an internal paired-display role
- interpretation:
  - on the internal Apple 5K panel path, role `10` is likely rejected by this gate
  - that makes it much more plausible that firmware search naturally falls through from the role-`11/12` family to the later role-`8` activation-side family when the non-`11/12` bucket is reached

### New role-`8/9` activation details: the last real gate is inside `CoreEg2SetSignalForDisplay`

- `CoreEg2FindOrCreateDisplayByConnector` always returns the display object if it exists or can be created
- it does **not** check the return value from:
  - `CoreEg2SetSignalForDisplay(display, role)`
- practical consequence:
  - a display object can exist for role `8`
  - but the paired-display activation path is only really armed if the role-`8/9` branch inside `CoreEg2SetSignalForDisplay` succeeds

### Exact role-`8/9` descriptor gate

- in `CoreEg2SetSignalForDisplay`, the paired-display branch is selected by:
  - `(signal & ~1) == 8`
- inside that branch, firmware scans the imported connector record set at:
  - `display + 152`
- it starts at offset:
  - `0x36`
- and advances in steps of:
  - `0x12` bytes
- it looks for a descriptor prefix:
  - `00 00 00 01 00 06 10`
- if no such descriptor is found before offset `0x7E`:
  - the role-`8/9` branch fails with `EFI_NOT_FOUND`
  - and the paired-display state is **not** armed

### What happens when the role-`8/9` descriptor gate succeeds

- firmware decodes bitfields from the matched 18-byte descriptor:
  - byte `+7`:
    - bit `0` -> `display+564 = !bit0`
    - bit `4` -> `display+565`
    - bit `5` -> `display+567`
  - byte `+9`:
    - bit `3` -> `display+566`
  - byte `+8`:
    - bit `0` -> `display+568 = 6 or 8`
    - bit `4` -> `display+569 = 6 or 8`
- then it performs the critical activation-side stores:
  - `qword [display+600] = 0x100010000`
  - `qword [display+608] = display + 560`
  - `dword [display+584] = 7`
- this is the point where:
  - `display+604` becomes `1`
  - and the paired-display header becomes available to the later lower backend

### The 28-byte side block for the role-`8/9` branch

- after the descriptor gate succeeds, firmware allocates a 28-byte side block at:
  - `display + 592`
- it tries to populate that block from a GUID-selected provider record:
  - `4DD18C5B-FD83-075A-5D2C-151B227417AB`
- if that query fails, firmware falls back to defaults equivalent to:
  - `[0] = 0`
  - `[1] = 1`
  - `[2] = 200`
  - `[3] = 200`
  - `[4] = 1`
  - `[5] = 0`
  - `[6] = 400`
- this side block is not the main gate
- the main gate is still:
  - finding the role-`8/9` descriptor
  - and performing the `display+600` write that makes `display+604` live

### Refined implication for the preboot 5K path

- it is no longer enough to say:
  - firmware eventually reaches role `8`
- the stronger requirement is:
  - firmware must reach role `8/9`
  - and the imported connector record set must contain the specific 18-byte descriptor family expected by `CoreEg2SetSignalForDisplay`
- if that descriptor is missing:
  - the role-`8/9` activation branch silently fails
  - even though a display object may still exist

### New upstream source for the early role gate: device-path property database

- `CoreEg2CreateDisplayObject` calls a provider through:
  - `qword_E658`
  - to populate the side structure at `display + 104`
- `CoreEg2SetSignalForDisplay` immediately reads the early gate byte from that structure at:
  - `*(*(*(display+104)+8)+20)`
- tracing the registration side shows:
  - `qword_E658` is set by `CoreEg2RegisterDevicePathPropertyDatabase`
  - the associated registration record points at:
    - `EFI_DEVICE_PATH_PROPERTY_DATABASE_PROTOCOL_GUID`
- practical consequence:
  - the first role-dependent gate before the `8/9` descriptor scan is driven by a **device-path property database** result
  - not by the Apple platform-info database path

### Saved-display apply can preselect role `8`, but only as a replay lane

- `GopApplySavedDisplayConfiguration` chooses the logical role from the selected output mask at:
  - `device + 9152`
- current mapping in the saved-apply path:
  - if `(mask & 0x10) == 0x10`:
    - select role `8`
  - else if `(mask & 0xF00) != 0`:
    - select role `11`
  - else if `(mask & 0xF083) != 0`:
    - select role `10` or `6`
- practical consequence:
  - the GOP **can** directly preselect role `8`
  - but only when the saved output mask already contains the dedicated `0x10` bit

### Why role `8` now looks like a replay/restoration lane, not the normal first-discovery lane

- the live connect-flags builder (`GopBuildDisplayConnectFlags`) uses `sub_14240()` to aggregate active DP-family seeds from bytes:
  - `a1 + 18920 .. 18923`
- for the important descriptor classes, `sub_14240()` returns:
  - `0x100/0x200/0x300/0x400` combinations
  - then ORs in the low bits
- that is the same grouped DP-family material that later feeds:
  - `0xF00 -> published type 3840 -> DP_INT`
- by contrast, the direct role-`8` selection needs:
  - a saved output mask already carrying `0x10`
- interpretation:
  - the natural live-discovery path on the iMac still looks like:
    - grouped DP-family / `0xF00` root first
    - then later paired-display activation
  - while direct role `8` selection looks more like:
    - replay/restoration of a previously established state

### Supporting clue from the graphicsconnector backend

- `GopGraphicsConnectorBackendApplyOutputMask` only interprets the grouped high bits:
  - `0xF00 -> 3840`
  - `0x100 -> 256`
  - `0x200 -> 512`
  - `0x300 -> 768`
  - `0x400 -> 1024`
  - `0x500 -> 1280`
  - `0x600 -> 1536`
- it does **not** have a corresponding direct published-output path for `0x10`
- practical consequence:
  - the high-level published boot-display/type path is grouped-mask driven
  - this is another reason role `8` looks like a later logical replay/activation choice, not the primary output-publication path

### Stronger replay-side evidence: saved replay preserves the grouped/root family, not direct role `8`

- `GopReplaySavedDisplayConfiguration` imports each saved display entry and records:
  - `connector + 380 & 0xF00`
- it writes that grouped value back into:
  - the active output-mask slot at `device + 6928 + ... + 2224`
  - and the connector-side cached field at `connector + 384`
- it does **not** preserve the low `0x10` bit there
- practical consequence:
  - the replay lane preserves the grouped/root family
  - not a direct role-`8` activation mask
- interpretation:
  - this is strong evidence that the natural saved/replay path restores the role-`11`/grouped-root side first
  - and any eventual role-`8` activation is likely derived later by the Apple `CoreEG2` state machine rather than replayed directly as the primary restored output identity

### New GOP-side narrowing: role `8` looks like the default activation lane, not a published boot identity

- `GopRefreshConnectorRecordCacheByMask` only refreshes the DP-family masks:
  - `0x100`
  - `0x200`
  - `0x300`
  - `0x400`
- it uses paged opcode `160` to fill those caches and mark which DP-family probes respond
- this means the initial cache build is still rooted in the grouped DP-family state, not a low-bit role-`8` mask

- `GopGetConnectorStateSlotByMask` gives dedicated connector-state slots only to the grouped/high-mask families:
  - default / unrecognized:
    - `device + 8024`
  - `0x100`:
    - also falls into the default slot
  - `0x200`:
    - `device + 9840`
  - `0x300`:
    - `device + 11656`
  - `0x400`:
    - `device + 13472`
  - `0x500`:
    - `device + 15288`
  - `0x600`:
    - `device + 17104`
  - `0xF00`:
    - `device + 6208`

- `sub_1D010`, which seeds the connector-state code for that slot, has the same shape:
  - default / unrecognized:
    - state code `8`
  - `0x200`:
    - `128`
  - `0x300`:
    - `512`
  - `0x400`:
    - `1024`
  - `0x500`:
    - `2048`
  - `0x600`:
    - `64`
  - `0xF00`:
    - `2`

- practical consequence:
  - low `0x10` does not appear to get a dedicated connector-state slot or unique low-level state code
  - it collapses into the default/non-root activation lane
  - this makes role `8` look even more like the fallback/default activation path than a separate boot-visible root identity

- `GopPublishPrimaryDisplayOutputs` reinforces that split:
  - it calls `GopActivateConnectorStateByMask(mask)` and publishes only:
    - TMDS / HDMI combinations
    - `0xF00 -> DP_INT`
    - `0x100 -> DP1`
    - `0x200 -> DP2`
    - `0x300 -> DP3`
    - `0x400 -> DP4`
    - and the remaining grouped DP families
  - there is no direct publication path for low `0x10`

- practical consequence:
  - firmware does not treat low `0x10` as a first-class published boot-display identity
  - the grouped/root family is what gets published and activated preboot
  - role `8` is much more likely to be a later logical activation lane used by `CoreEG2` once it has already consumed the imported record set

### New descriptor clue from live build

- `GopBuildDisplayConnectFlags` decodes a matched descriptor block and uses:
  - `descriptor + 0x24`, bit `0x10`
  - to control the published `display-connect-flags` property
- this strongly suggests the low-bit activation behavior is synthesized from descriptor content during live build
- combined with the replay behavior above, the best current model is:
  - grouped/root state is what gets restored and published
  - role-`8` activation is derived later from descriptor bits, not replayed directly as the primary mask

### New `CoreEG2` detail: what the `8/9` branch actually decodes and publishes

- `CoreEg2SetSignalForDisplay` treats `8` and `9` as one family:
  - `(signal & ~1) == 8`
- the branch scans the imported connector record set at:
  - `display + 152`
  - starting at offset `0x36`
  - stepping by `0x12`
- the required descriptor prefix is:
  - `00 00 00 01 00 06 10`

- once that descriptor is found, `CoreEG2` derives the first paired-display fields directly from descriptor bytes:
  - byte `+7`:
    - bit `0` inverted -> `display + 564`
    - bit `4` -> `display + 565`
    - bit `5` -> `display + 567`
  - byte `+9`:
    - bit `3` -> `display + 566`
  - byte `+8`:
    - bit `0` -> `display + 568`
      - encoded as `6` or `8`
    - bit `4` -> `display + 569`
      - encoded as `6` or `8`

- this is the exact point where the paired complex-display path becomes live:
  - `display + 560` gets a header marker:
    - `0x10000`
  - `display + 600` is written with:
    - `0x100010000`
  - `display + 608` is set to:
    - `display + 560`
  - `display + 584` is set to:
    - `7`
- practical consequence:
  - the paired-display state is not just one bit
  - it is a descriptor-driven property block rooted at `display + 560`
  - and `display + 604 == 1` is just the visible high dword of that `0x100010000` store

### Optional provider-backed side block for the paired path

- after arming the paired path, `CoreEg2SetSignalForDisplay` allocates:
  - `0x1C` bytes
  - stored at `display + 592`
- it then calls `sub_78F0`, which is a size-checked query wrapper over:
  - `qword_E660`
  - the Apple platform-info provider already traced earlier
- this means the paired path has an optional 28-byte provider-backed side block on top of the raw descriptor bits

- if that provider query fails, `CoreEG2` fills the side block with defaults:
  - `[0x00] = 0`
  - `[0x04] = 1`
  - `[0x08] = 0xC8`
  - `[0x0C] = 0xC8`
  - `[0x10] = 1`
  - `[0x14] = 0`
  - `[0x18] = 0x190`

- practical consequence:
  - the paired path can still proceed without the provider query succeeding
  - so this side block looks like a refinement layer, not the root gate

### `CoreEG2` publishes a richer named property set after the descriptor match

- after the paired path is armed, `CoreEg2SetSignalForDisplay` publishes named fields through the shared display service:
  - `LinkType`
  - `A`
  - `D`
  - `DataJustify`
  - `Dither`
  - `L`
  - `PixelFormat`
  - `InverterFrequency`

- these properties are written through:
  - `qword_E668`
  - the shared display property service already seen elsewhere in `CoreEG2`

- important interpretation:
  - the `8/9` branch is not just a final boolean gate
  - it is the place where `CoreEG2` translates imported connector-record descriptors into the concrete property model of the paired internal display path

### What still looks upstream of this branch

- `CoreEg2ValidateConnectorRecordSet` is intentionally weak:
  - it mostly checks record-set size/alignment/sentinel shape
  - it does **not** enforce Apple panel or paired-display rules
- so the real 5K/pairing split still happens later, in:
  - `CoreEg2ComplexDisplayInit`
  - plus the descriptor-driven `8/9` branch of `CoreEg2SetSignalForDisplay`

### New source split: how `CoreEG2` fetches the two display-object families

- `CoreEg2FindOrCreateDisplayByConnector` reduces roles into only two families:
  - `11/12`
  - everything else
- `CoreEg2FetchConnectorRecordSet(controller, family_bit)` then fetches the backing record set differently for those two families

- for family bit `1` (`11/12`):
  - it uses the secondary AMD backend at `controller->backend->dispatch`
  - first with opcode:
    - `0x600 / 1536`
    - as a selector / handshake
  - then with opcode:
    - `0xA0 / 160`
    - to fetch the first 128-byte page
  - it handles continuation via byte `126` and fetches chained pages when present
  - if the first `1536` handshake succeeds while the controller is not yet in mode `1`, it sets:
    - `byte_E5E0 = 1`

- for family bit `0` (non-`11/12`):
  - it does **not** use the special secondary backend
  - it uses the direct connector protocol method at interface offset `+0x28`
  - still with opcode:
    - `0xA0 / 160`
    - but through the ordinary direct connector path

- practical consequence:
  - the `11/12` root family is special at the source of record acquisition, not only later in `CoreEG2`
  - this makes the firmware state machine look like:
    1. special root-family fetch through the secondary backend
    2. try complex-display construction
    3. on failure, fall through to the generic/non-root family path
    4. later descriptor-driven `8/9` activation can still happen on top of that generic path

### Record-set validation is not where the 5K split happens

- `CoreEg2ValidateConnectorRecordSet` only checks:
  - record size is at least `0x80`
  - low bits of the size field have the expected alignment shape
  - the record-set pointer trailer matches a sentinel-like layout
- it does **not** check:
  - Apple panel signature
  - `11/12` vs `8/9`
  - dual-link / paired-display conditions
- practical consequence:
  - both the special and generic families can pass validation
  - the meaningful 5K split still happens later, during:
    - `CoreEg2ComplexDisplayInit`
    - and especially the descriptor-driven `8/9` parser in `CoreEg2SetSignalForDisplay`

### New generic-fallback selector model: how firmware advances after complex-display failure

- after a failed complex-display or setup/activation attempt, `CoreEg2GenericInitializeGraphicsDevice` does **not** simply retry the same object
- it tears down the current display state with:
  - `CoreEg2TearDownActiveDisplaySequence`
- then runs two ordered search passes:
  1. `START:SearchDeviceForDisplay`
     - calls `sub_98A3(controller_root, 1)`
  2. `START:SearchDeviceForDisplay-Lid`
     - only if the first pass found nothing
     - calls `sub_98A3(controller_root, 0)`

### What `sub_98A3` really does

- `sub_98A3` walks the controller/device list, then each controller’s connector-role list in descending imported-role order
- for each connector candidate it calls:
  - `CoreEg2FindOrCreateDisplayByConnector(controller_family, role)`
- if a display object is returned:
  - and its internal `display + 16` visited byte is still `0`
  - `sub_98A3` sets `display + 16 = 1`
  - and returns that display
- if `display + 16` was already `1`, it keeps searching

- practical consequence:
  - this is the concrete mechanism that lets generic fallback advance past a previously tried family
  - combined with `CoreEg2TearDownActiveDisplaySequence` not clearing `display + 16`, firmware naturally progresses to a different display-object family instead of retrying the same one

### The two search passes are not identical

- in `sub_98A3`, the first pass argument `a2 = 1` adds a skip rule:
  - if the current controller state field at `controller + 0x10` is `1`
  - and `sub_74C1()` returns true
  - that controller is skipped in this pass
- the second pass (`a2 = 0`) does **not** apply that skip

- practical consequence:
  - generic init first tries a stricter search
  - then falls back to a broader “Lid” search if nothing was accepted
- important note:
  - despite the string name, this pass is still part of the iMac firmware path
  - it should be interpreted as a broader search class, not literally as laptop-only logic

### Role `10` remains a filtered special case inside generic search

- `sub_98A3` accepts non-`10` roles immediately when a display object is returned
- for role `10` only, it performs the extra backend query:
  - opcode `0x80`
  - 16-byte reply
  - compared against `byte_E120`
- that reply matches:
  - `DP-HDMI ADAPTOR`
- if it does **not** match, search continues to the next connector role

- practical consequence:
  - role `10` is still a filtered DP-HDMI adapter lane
  - not the most likely internal iMac paired-display activation lane

### What this means for the 4K-vs-5K split

- the current best model for the firmware state machine is now:
  1. fetch root family `11/12` through the special secondary backend
  2. try complex-display construction and paired setup
  3. on failure, tear down current state
  4. enter generic init
  5. generic init searches for a fresh display object, first with a stricter pass, then with a broader pass
  6. later non-root roles, especially the `8/9` family, can still arm the paired-display state if their descriptor-driven parse succeeds

- practical consequence:
  - the fallback path is a deliberate staged firmware state machine
  - not just “complex-display failed, so we use plain 4K forever”
  - this keeps open the possibility that the Linux-visible preboot state may already be a generic-but-still-structured state from which Windows later finishes the job

### New structural correction: `CoreEG2` keeps only two display-object families

- `CoreEg2FindOrCreateDisplayByConnector(controller, role)` does **not** create one display object per imported role
- instead it collapses roles into one family bit:
  - `role 11/12` -> root family
  - everything else -> non-root family
- it searches the global display-object list for an existing object with:
  - the same controller
  - the same family bit
- if found, it reuses that object
- if not found, it fetches a connector record set, creates the object, and then calls:
  - `CoreEg2SetSignalForDisplay(display, role)`

- practical consequence:
  - `CoreEG2` is not choosing between many independent role-specific display objects
  - it has a root object family and a non-root object family, and later role attempts can re-signal the same non-root object

### New implication for the generic search loop

- because `CoreEg2FindOrCreateDisplayByConnector` always calls `CoreEg2SetSignalForDisplay` before returning, even a previously created family object can be reprogrammed with a later imported role value
- `sub_98A3` then decides only whether to return that object, based on the visited byte at `display + 16`
- this means the generic search loop is really:
  - walk imported connector roles in descending order
  - keep re-signaling the same family object as needed
  - only return the object when it is still fresh for that family

- practical consequence:
  - the firmware is closer to a role-driven state machine on top of two persistent display-object families than to a simple connector enumeration

### New generic-init detail: restore-first path before the search passes

- `CoreEg2GenericInitializeGraphicsDevice` first checks for saved configuration through the shared backend `qword_E668`
- if saved config is present and restore succeeds:
  - it does **not** enter the later `sub_98A3(...,1)` / `sub_98A3(...,0)` search loop immediately
  - instead it reconstructs a display choice from the saved config table and calls:
    - `CoreEg2FindOrCreateDisplayByConnector(controller_family, saved_role)`
- it then:
  - sets `display + 532 = 1`
  - assigns `display + 408`
  - computes `display + 524 = sub_2CB7(display)`
  - optionally link-trains when the role is `11/12`
  - activates the display and installs protocols
  - reports success through `sub_AD18(controller_root, display, 1)`

- practical consequence:
  - generic init is really:
    1. skip/fail gates
    2. try saved-config restore
    3. only then fall back to the two search passes
  - this makes saved preboot state even more important than it looked earlier

### What `sub_AD18` appears to report

- `sub_AD18(controller_root, display, from_restore)` packages a 140-byte status/report blob and sends it through the shared service `qword_E668 + 16`
- it includes:
  - controller identity
  - display object identity fields around `display + 340/344`
  - active timing/config fields from `display + 352..404`
  - and, when available, active complex-display side data from `display + 408`
- the third argument distinguishes:
  - `1` = success after restore-driven path
  - `0` = success after generic search/activation path

- practical consequence:
  - firmware explicitly remembers whether a successful activation came from restored state or from the generic search path
  - that strongly supports the idea that the preboot state machine has multiple success lanes, not just one paired path and one dumb fallback

### New GOP-side correction: raw imported roles on this iMac are `11`, `10`, and `6`

- in `GopInitializeGraphicsConnectorObject`, the physical connector index maps to these raw mask seeds:
  - connector `1` -> `0x120`
  - connector `2` -> `0x220`
  - connector `3` -> `0x301`
  - connector `5` -> `0x402`
  - connectors `4` and `6` -> `0`

- from those masks, the constructor appends descriptor-role entries at `connector + 104 + 0x20*n`:
  - `role 11`
    - appended for any connector with `mask & 0xF00`
    - present on connectors `1`, `2`, `3`, `5`
  - `role 10`
    - appended for connectors where `(mask & 0xF083) != 0` and `(mask & 0x20) == 0`
    - present on connectors `3` and `5`
  - `role 6`
    - appended by the same non-`0x20` family when descriptor count is still below `3`
    - also present on connectors `3` and `5`
  - `role 5`
    - would come from `mask & 0x0C`
    - not present in this iMac mask set
  - `role 8`
    - would come from `mask & 0x10`
    - also not present in this raw constructor path for this iMac mask set

- practical consequence:
  - the raw role list imported into `CoreEG2` from the GOP private connector protocol is, on this machine, fundamentally `11`, `10`, `6`
  - not a raw `8/9`-first list

### Important model correction: generic search likely falls to `6`, not directly to `8`

- `sub_5C01` imports connector-role descriptors from the GOP connector protocol and sorts them in descending role order
- on this iMac config that means the meaningful per-connector raw order is:
  - `11`
  - `10`
  - `6`
- `sub_98A3` in `CoreEG2` walks those imported raw roles
- `role 10` is then specially filtered by the `DP-HDMI ADAPTOR` check

- practical consequence:
  - after `11/12` root-family failure, the generic search path is much more likely to progress toward `6`
  - not toward a raw boot-time `8`

### New interpretation of `role 8`

- `GopApplySavedDisplayConfiguration` explicitly synthesizes `role 8` when the selected saved connector mask collapses to `0x10`
- this role is chosen from saved-state logic, not from the raw iMac connector constructor path above
- `CoreEg2FindOrCreateDisplayByConnector` can still receive `role 8` from restore logic because it only uses the role value to select the family and then re-signal the display object

- practical consequence:
  - `role 8` now looks like a restore-driven synthetic activation role
  - not the primary raw generic-search role on this machine
  - this makes saved preboot state even more central to the full paired-display path than before

### Scoped negative result: no active raw or saved-path `role 9` assignment found in the GOP pass

- a targeted sweep of these GOP ranges found no literal `role 9` assignment:
  - `GopInitializeGraphicsConnectorObject`
  - `GopApplySavedDisplayConfiguration`
  - `GopReplaySavedDisplayConfiguration`
- `CoreEG2` still treats `8/9` as one family with `(signal & ~1) == 8`, but in the GOP module the active paths we traced for this iMac only concretely assign:
  - raw constructor roles: `11`, `10`, `6`
  - saved/apply synthesized role: `8`

- practical consequence:
  - `role 9` currently looks more like a generic sibling/variant in `CoreEG2` than a proven active preboot role on this exact iMac path
  - we should avoid assuming that the iMac 5K boot path necessarily depends on `9`

### New strongest GOP-side gate: `GopBuildDisplayConnectFlags` is the real grouped-vs-generic selector

- `GopBuildDisplayConnectFlags` iterates candidate `(mask, descriptor)` pairs from `unk_2C050`
- for each candidate it:
  - maps the candidate mask to an output slot
  - checks the live connector-record cache flag for that slot via `sub_19FA0`
  - loads the live cached record blob via `sub_19EC0`
  - rejects candidates when the slot is absent or the candidate-vs-live bit conditions fail

- if a candidate survives those slot checks, the firmware then tries to match one of **two** stored descriptor families:
  1. first family:
     - record byte `127` must match `a1 + 1402`
     - timing block must exist at `a1 + 1411`
     - aggregated active-mask helper `sub_14240(a1, *(a1 + 1492))` must intersect the current candidate mask
     - this family is allowed when:
       - only one live cache is active
       - or `sub_147B0(a1)` says the current candidate matches the stored preferred mask
  2. second family:
     - record byte `127` must match `a1 + 1410`
     - timing block must exist at `a1 + 1435`
     - aggregated active-mask helper `sub_14240(a1, *(a1 + 1588))` must intersect the current candidate mask
     - this family is only allowed when `v8 <= 1`, meaning the multi-cache case is **not** active

- if neither family matches, `GopBuildDisplayConnectFlags` returns `EFI_NOT_FOUND`

- practical consequence:
  - this is the strongest static selector we have for:
    - grouped internal/paired path
    - versus generic fallback
  - the first family can survive a multi-cache environment if the preferred-mask check passes
  - the second family is much stricter and looks more like a simpler/single-cache path

### What `sub_14240` is doing for that selector

- `sub_14240` converts stored selector bytes into connector-mask families
- for selector values `4` or `8`, it aggregates the live DP-family cache bytes at:
  - `a1 + 18920`
  - `a1 + 18921`
  - `a1 + 18922`
  - `a1 + 18923`
- those become grouped masks like:
  - `0x100`
  - `0x200`
  - `0x300`
  - `0x400`
  - plus low bits `| 3`
- other selector values map to smaller masks like:
  - `16 -> 4`
  - `32 -> 8`
  - `2/34 -> 32`
  - `128/160 -> 64`

- practical consequence:
  - the grouped-vs-generic decision is driven by live connector-cache population plus stored selector bytes
  - not just by a single raw connector enum

### Why this likely matters for the iMac 5K path

- the first descriptor family in `GopBuildDisplayConnectFlags` is the only one that can survive the multi-cache case when the preferred-mask helper matches
- that fits the 5K hypothesis much better than the simpler second family, because the paired internal path is exactly where we expect multiple related connector caches to matter
- if plain Linux boot reaches only the simpler/generic family, the firmware can still hand over a valid display state, but not the richer grouped internal state that later feeds the paired path

### New correction: `sub_1F3C0` is a mode/preset classifier, not the selector-family source

- the earlier offset overlap with `1492/1588` was misleading
- `sub_1F3C0` builds a large stack-local preset table and is wrapped by:
  - `sub_20710`
  - `sub_20770`
  - `sub_20830`
- those callers use it to classify/encode mode metadata for outgoing packets
- the offsets seen there are local table entries, not the device-object fields that `GopBuildDisplayConnectFlags` reads at:
  - `a1 + 1402`
  - `a1 + 1410`
  - `a1 + 1411`
  - `a1 + 1435`
  - `a1 + 1492`
  - `a1 + 1495`
  - `a1 + 1588`

- practical consequence:
  - `sub_1F3C0` is still relevant to advertised mode classification
  - but it is not the source of the paired-vs-generic selector families

### Device-context initializer path: the GOP object is seeded from a copied ROM/blob plus device-specific tables

- `GopCreateDisplayDeviceContext` allocates the main per-device object and then calls:
  - `sub_BD00`
  - `sub_CB70`
- `sub_BD00` is the big device-context initializer:
  - sets defaults
  - seeds connector/cache state arrays
  - calls `sub_12670`
  - calls `sub_B860`
  - calls `sub_141E0`
- `sub_12670`:
  - allocates and copies a device-provided binary blob into `a1 + 680`
  - validates it with `sub_12590`
  - stores a secondary pointer at `a1 + 696`
- `sub_12590` only validates the blob header/checksum; it does **not** parse the selector families
- `sub_CB70` installs device-specific tables for the ASIC family, not the live paired-display decision itself

- practical consequence:
  - the selector-family bytes used later by `GopBuildDisplayConnectFlags` are probably coming from:
    - device-specific handler/table state
    - ROM/blob-driven parsing
    - or both
  - but not from `sub_1F3C0`

### Static candidate table `unk_2C050`: the grouped `0xF00` path is a promotion, not a raw table entry

- dumping `unk_2C050` shows that `GopBuildDisplayConnectFlags` iterates ordinary seed masks:
  - `0x100`
  - `0x200`
  - `0x300`
  - `0x400`
  - plus low-bit cases `0x1` and `0x2`
- there is no raw `0xF00` candidate in that table
- `dword_2C530` also confirms:
  - ordinary masks like `0x100`, `0x200`, `0x300`, `0x400`
  - grouped output types like `3840 / 0xF00` exist in the mapping layer, but not as the initial seed candidates

- practical consequence:
  - the internal grouped path is built on top of ordinary DP-family seeds
  - it is not simply chosen as a raw connector mask from the start

### New concrete preferred-mask detail: `sub_147B0` uses `a1 + 1495` as a stored preferred grouped candidate

- `sub_147B0(a1)` compares a preferred-mask value extracted from `a1 + 1495` against the current candidate mask from `unk_2C050`
- this helper is only used by `GopBuildDisplayConnectFlags`
- it is what lets the first descriptor family survive the multi-cache case

- practical consequence:
  - the first descriptor family is not just “any matching cache”
  - it has a stored preferred candidate, and that preference matters when multiple caches are active

### New grouped-path latch: connector-state refresh explicitly sets `a1 + 18932`

- `sub_CCA0` refreshes the connector-state slot by reading extended opcode windows
- after that refresh, it does:
  - if `slot[25] != 0`
  - and current selected mask is `3840 / 0xF00`
  - then set `a1 + 18932 = 1`
- `GopGetConnectorStateSlotByMask` shows the grouped slot mapping:
  - `3840 / 0xF00 -> a1 + 6208`
  - `1536 / 0x600 -> a1 + 17104`
  - `0x200/0x300/0x400/0x500` each have their own slots

- xrefs to `a1 + 18932` now tie it directly to the richer DP path:
  - `sub_8510`
  - `sub_1D3E0`
  - `GopTrainDisplayPortAndPublishDpcd`
  - `GopPublishDpcdTrainingProperties`

- practical consequence:
  - `a1 + 18932` is a real grouped-path-live latch on the GOP side
  - not just another generic state byte

### New strongest 5K-specific finding: grouped `0xF00` unlocks extra DP training behavior

- `GopGraphicsConnectorBackendApplyOutputMask`:
  - calls `sub_8510`
  - then, when `(selected_mask & 0xF00) == 0xF00`, it publishes mask `3840`
  - then activates the `0xF00` connector-state path through the normal activation functions
- inside `sub_8510`, after state-slot refresh:
  - `a1 + 18932` is used as the grouped-path latch
  - `a1 + 18944`, `a1 + 18948`, `a1 + 18952` are derived from grouped slot state
- failure-side grouped branch:
  - records generic DP training fallback values into:
    - `a1 + 18936`
    - `a1 + 18940`
  - sets `a1 + 18944 = 1` when slot bit `0x80` is absent
  - clears `a1 + 18948`
  - if grouped path is active and `a1 + 18932` is set, it additionally checks the grouped sub-block:
    - choose `+0x400` or `+0x500` based on `slot[17] & 1`
    - require:
      - `[sub+12] == 0`
      - `[sub+13] == 16`
      - `[sub+14] == 250`
    - then store `a1 + 18952 = slot[25] & 1`

- `GopTrainDisplayPortAndPublishDpcd`:
  - only takes the extra `sub_1D8D0` branch when:
    - `a1 + 18932 != 0`
    - and `a1 + 18952 != 0`
- `GopPublishDpcdTrainingProperties`:
  - if `a1 + 18932 != 0`, publishes extra property id `266`
  - if `a1 + 18944 != 0`, clears one training/status bit from the outgoing lane-count/status byte
  - if `a1 + 18952 != 0`, sets the extra grouped-path flag byte used for property `266`
- `sub_8510` also accepts special custom link-rate values only when grouped path is live:
  - `21600`
  - `24300`
  - `32400`
  - `43200`
  - otherwise those values are rejected as unsupported

- practical consequence:
  - this is the strongest static evidence so far that the grouped `0xF00` path is a real preboot 5K-capable DP lane
  - not just a cosmetic connector classification
  - the grouped path unlocks extra link-training and DPCD publication behavior that the generic path does not get

### New grouped subtype split: `0xF00` alone is not yet the full paired-live state

- `sub_1D3E0(a1, mask)` classifies the current selected mask family for outgoing packetization
- for grouped masks:
  - if `(mask & 0xF00) != 0`, it returns `19`
  - but if `a1 + 18932` is set, it upgrades that return value to `20`
- this classifier is used by `sub_12DB0`, which builds a packet for `sub_21D30(..., 0x4C)` and stores:
  - `BYTE2(Buffer[2]) = sub_1D3E0(...)`

- practical consequence:
  - the GOP does **not** treat every grouped `0xF00` state as equivalent
  - there is now a concrete distinction between:
    - grouped-but-not-fully-live (`19`)
    - grouped-and-paired/live (`20`)
  - that looks like one of the best static candidates for:
    - generic grouped preinit
    - versus the richer fully paired internal 5K preinit path

### Lower backend mapping: grouped `0xF00` is a first-class connector-backend state, not just a bitmask coincidence

- `GopGraphicsConnectorBackendDispatchOpcode` maps the currently selected mask family to a backend state code before dispatching commands
- current mapping includes:
  - `0x100 -> 8`
  - `0x200 -> 128`
  - `0x300 -> 512`
  - `0x400 -> 1024`
  - `0x500 -> 2048`
  - `0x600 -> 64`
  - `0xF00 -> 2`
- this happens before the dispatcher chooses low-byte vs extended-window transport

- practical consequence:
  - the grouped internal path is not just “multiple ordinary DP seeds present”
  - it has its own explicit lower-backend state code (`2`)
  - that makes the 5K-capable path look even more like a dedicated hardware/mailbox mode

### Lower backend readiness: grouped `0xF00` gets special wait/poll treatment

- `GopWaitForConnectorMaskReady(a1, mask)` derives a ready bit from the selected mask and polls register `0x1223C`
- even though `sub_1D0B0` mostly maps ordinary masks, `GopWaitForConnectorMaskReady` explicitly treats the grouped path specially:
  - it enters the active poll loop whenever:
    - `(mask & 0x20) != 0`
    - or `(mask & 0xF00) == 0xF00`
- this means the grouped path is treated like a state that may need extra hardware convergence time before use

- practical consequence:
  - grouped `0xF00` is special all the way down:
    - selector family
    - grouped-live latch
    - subtype `19 -> 20`
    - extra DP training properties
    - dedicated backend state code
    - and dedicated ready/poll handling

### Narrow result on `slot[25]`: the GOP does not set it in software; it is sourced from the lower connector mailbox

- `slot[25]` is only used locally in two places:
  - `GopRefreshConnectorStateSlotFromExtendedWindows`
  - `GopPrepareDpTrainingStateForSelectedMask`
- no local software writer for `slot[25]` was found in the GOP module
- the byte comes from the first connector-state window refresh:
  - `GopRefreshConnectorStateSlotFromExtendedWindows` reads 16 bytes from extended window offset `0x000`
  - that data lands at `slot + 12 .. slot + 27`
  - therefore `slot[25]` is byte `13` of the very first 16-byte extended read
- that read only happens after the lower connector backend has already:
  - mapped the selected grouped mask `0xF00` to backend state code `2`
  - selected the connector register bank
  - and passed the grouped-mask ready/poll check

- practical consequence:
  - from static RE, `slot[25]` is best understood as a lower hardware/mailbox-sourced grouped-live indicator
  - not a higher-level software policy bit written by the GOP itself
  - this strengthens the model that the fully paired/live 5K preinit state depends on a deeper connector-engine condition, not just GOP-side bookkeeping

### New strongest cross-module correlation: `CoreEG2`’s special root-class byte matches the GOP grouped-live subtype

- in `CoreEg2SetSignalForDisplay`, the `11/12` root-family branch asks the lower backend for a classification/result block
- it stores the first returned byte into `display + 300` (`0x12C`)
- that branch only accepts values in the range:
  - `0x10 .. 0x14`
  - anything outside that range aborts the root-path decode
- later, `sub_9E99` takes its special paired-display iterator path only when:
  - `display + 300 >= 0x14`
  - `display + 318 != 0`
  - `display + 320 == 10`
  - `display + 328 == 0x258` (`600`)

- because `CoreEg2SetSignalForDisplay` already rejects values above `0x14`, the `>= 0x14` check in `sub_9E99` really means:
  - `display + 300 == 0x14`

- practical consequence:
  - this is the clearest Apple-side match yet to the GOP grouped subtype split:
    - GOP grouped default subtype: `19`
    - GOP grouped live subtype: `20`
    - CoreEG2 special root-class byte: `0x14` (`20`)
  - static RE now strongly supports the idea that `CoreEG2`’s paired-display success path is keyed off the same lower grouped-live state that the GOP distinguishes as subtype `20`

### What the `CoreEG2` paired-success branch actually does after seeing class `0x14`

- when the `sub_9E99` special branch is taken:
  - it uses an alternate iterator/provider path from the display object’s protocol table
  - walks returned timing/property records
  - checks them with `CoreEg2IsTimingCompatibleWithConnectorRecord`
  - and on success copies the resulting record block into:
    - `display + 0x1A0 .. 0x1ED`
- after that it derives additional policy from the copied block:
  - `display + 0x1A8`
  - `display + 0x1AC = sub_52AA(display)`

- `sub_52AA` is also informative:
  - if `display + 604 == 1`, it returns `0`
  - otherwise it derives a value from imported connector-record bytes
- practical consequence:
  - the `display + 300 == 0x14` branch is not just another validation check
  - it is the gateway into a distinct paired/root property-record import path on the Apple side

### New consumer-side result: the paired/root record block is passed directly into the lower pipeline object

- `CoreEg2ConfigureAndActivateDisplayPipeline` is the first hard consumer of the success-side record block:
  - it calls `CoreEg2ResolvePostSignalDisplayRecord`
  - then passes:
    - `display + 0x1A0` as the main record/config block
    - `display + 0x258` only when `display + 0x25C != 0`
  - into the active pipeline object method at `pipeline_vtbl + 0x38`
- so the paired/root path is not just cached metadata in `CoreEG2`
- it becomes the actual lower pipeline configuration payload

- practical consequence:
  - `display + 0x25C` is the real gate for whether the lower pipeline sees the extra complex-display blob
  - `display + 0x1A0 .. 0x1ED` is always the main display-record block handed to the pipeline once the record-build step succeeds

### Record-layout copy is shared between the ordinary iterator path and the special paired/root path

- both:
  - `CoreEg2BuildDisplayRecordFromConnectorIterator`
  - `CoreEg2ResolvePostSignalDisplayRecord`
- copy the same packed record layout into:
  - `display + 0x1A0 = 0x10001`
  - `display + 0x1B0`
  - `display + 0x1B8 .. 0x1ED`
- so the paired/root success branch does not invent a different record format
- it supplies a different source iterator and then fills the same downstream pipeline-facing structure

- practical consequence:
  - the main difference between generic and paired/root success is:
    - which iterator/provider produced the record
    - and how `display + 0x1A8`, `display + 0x1AC`, and `display + 0x1F0` are derived afterward
  - not the existence of a separate consumer format

### `display + 0x1A8`, `display + 0x1AC`, and `display + 0x1F0` are success-side policy outputs, not raw imported bytes

- after the record copy, both builders derive:
  - `display + 0x1A8`
  - `display + 0x1AC = sub_52AA(display)`
- `sub_52AA` returns:
  - `0` whenever `display + 0x25C == 1`
  - otherwise a table-driven value from imported connector-record bytes
- in the paired/root success branch, extra flag checks can override the defaults:
  - `display + 0x1A8 = 3/4`
  - `display + 0x1AC = 4/3`
  - `display + 0x1F0 = 2`
- those overrides depend on:
  - provider-side capability bytes from the display object
  - imported connector-record flag bytes around `record + 0x1C .. 0x1E`
  - local display flags at `display + 0x146` / `display + 0x150`

- practical consequence:
  - the paired/root branch is not only selecting a special record source
  - it can also switch the final policy/mode class that the lower pipeline receives
  - `display + 0x1F0 = 2` currently looks like one of the best Apple-side markers of the fully paired/live path after record import succeeds

### `sub_4D21` is a bandwidth-fit gate used only on the ordinary iterator path

- `CoreEg2BuildDisplayRecordFromConnectorIterator` calls `sub_4D21` only when:
  - no complex-display blob is already present
  - and the signal family is `11/12`
- `sub_4D21` compares derived pixel/data rate against a bandwidth budget and then forces retries/downgrades of `display + 0x1AC`
- if the budget does not hold, the builder reports the attempted and downgraded values through `sub_A8BB`

- practical consequence:
  - the ordinary root/iterator path contains an explicit bandwidth-fit loop
  - the special paired/root class-`0x14` success path in `CoreEg2ResolvePostSignalDisplayRecord` bypasses that exact loop and instead uses its own post-copy policy override path

### Exact Apple-side lower query for the paired/root class byte

- in `CoreEg2SetSignalForDisplay`, the root-family (`11/12`) setup issues a lower-backend query with:
  - selector `5`
  - request length / mode `1`
  - output byte buffer at local `v48`
  - output block at local `v50`
- the call only succeeds if:
  - the lower method returns success
  - and `(v48[0] & 6) == 2`
- after that:
  - the first returned class byte is stored at `display + 0x12C`
  - only values in `0x10 .. 0x14` are accepted
  - the later special paired/root resolver path effectively requires the top value `0x14`

- practical consequence:
  - the Apple-side paired/root gate is now narrowed to a concrete lower query:
    - selector `5`
    - success-bit pattern `(out[0] & 6) == 2`
    - class byte `0x14`
  - the best remaining static question is now on the GOP side:
    - which lower protocol method implements selector `5`
    - and what exact lower grouped-live state makes it return class `0x14`

### Better GOP-side publication story: grouped-live subtype `20` is explicitly emitted into the shared graphics path

- `GopClassifySelectedMaskSubtype` remains the only local producer of subtype `19/20`
- `sub_12DB0` packs that subtype directly into packet `0x4C`:
  - `BYTE2(Buffer[2]) = GopClassifySelectedMaskSubtype(...)`
  - then sends it via `sub_21D30(..., 0x4C)`
- `sub_21D30` is the custom shared-graphics packet interpreter/boundary already tied to Apple-facing complex-display publication

- practical consequence:
  - the GOP does not keep subtype `20` purely local
  - it explicitly exports it through the shared graphics service
  - this is now one of the strongest candidate bridges between:
    - lower grouped-live latch / subtype `20`
    - and the Apple-side paired/root class byte `0x14`

### Where grouped `0xF00` publishes its packet stream

- when published display type becomes `3840`, `GopPublishPrimaryDisplayOutputs`:
  - activates connector state by mask `0xF00`
  - labels the path as `ATY,EFIDisplay = DP_INT`
- `GopActivateConnectorStateByMask(a1, 0xF00)`:
  - refreshes the connector-state slot
  - normalizes the backend mask/state fields
  - calls:
    - `sub_1DEF0` -> grouped packet `0x0C`
    - `sub_1DE20` -> grouped packets `0x01` and `0x0D`
- `GopTrainDisplayPortAndPublishDpcd` also checks the grouped-live latch `a1+18932` and, when active with extra flag `a1+18952`, emits grouped packet `0x10`

- practical consequence:
  - the fully paired/live grouped path has a distinct publication stream:
    - packet `0x0C`
    - packet `0x01`
    - packet `0x0D`
    - optional packet `0x10`
    - and packet `0x4C` carrying subtype `19/20`
  - this is much closer to an Apple-visible grouped-live state machine than a single hidden latch

### Grouped-live latch is also consumed by DPCD training-property publication

- `GopPublishDpcdTrainingProperties` checks:
  - `a1 + 18932`
  - `a1 + 18952`
- when grouped-live is active it publishes extra training/property state before the normal DPCD properties:
  - explicit use of grouped packet path
  - extra grouped-only property emission

- practical consequence:
  - subtype `20` is not the only grouped-live export
  - the GOP also surfaces grouped-live state through its DPCD/training publication path
  - this strengthens the theory that Apple can later distinguish:
    - generic grouped/generic DP state
    - versus the fully paired/live grouped 5K-preinit state

### Important correction: provider split, export path, and paired-root go/no-go gate

2026-05-20 correction: this older note had `qword_E668` and `qword_E660` conflated.

- `qword_E660` is the Platform Info Database provider (`AC5E4829-A8FD-440B-AF33-9FFE013B12D8`) used for selector/key reads such as `dword_E670`.
- `qword_E668` is the Device Path Property Database provider (`91BD12FE-F6C3-44FB-A5B7-5122AB303AE0`) used for named per-device-path properties such as `SavedConfig`, `ComplexDisplaySetup`, AUX power, backlight/gamma, and failure records.
- `CoreEg2SharedProtocolMethod2/3/4` are just thin wrappers around that provider:
  - `qword_E668 + 8`
  - `qword_E668 + 16`
  - `qword_E668 + 24`
- the direct `qword_E668` callers in `CoreEG2` are dominated by named property/failure publication helpers:
  - `FailureNoDisplayFound-%d`
  - `InstallDisplayFailureInfo-%d`
  - `ActivateDisplayFailureInfo-%d`
  - `FetchEDIDFailureInfo-%d`
  - `FramebufferFailureInfo-%d`
  - `TimingFailureInfo-%d`
  - `LinkTrainFailureInfo-%d`
  - `EG2DisplayWAInfo-%d`
  - `GraphicsBacklightSetup-%d`
  - success-side `ComplexDisplaySetup`
- the most important success-side proof is in `CoreEg2ComplexDisplayInit`:
  - the named `ComplexDisplaySetup` blob is published through `qword_E668 + 16`
  - but only after:
    - second-connector setup
    - paired blob construction
    - dual link training
    - activation
    - and protocol installation

- practical consequence:
  - the Device Path Property Database provider now looks like the Apple-side export/telemetry/property channel for display state
  - it does **not** look like the original decision point that turns grouped-live GOP state into the paired-root class byte
  - the real paired-root gate remains the lower connector-backend selector-`5` query in `CoreEg2SetSignalForDisplay`

### Narrowed conclusion after correlating CoreEG2 with the grouped-live GOP path

- the Apple paired/root success path currently looks like:
  1. lower connector backend selector `5` returns valid status with `(reply[0] & 6) == 2`
  2. returned class byte is accepted in `0x10 .. 0x14`
  3. later special success effectively wants class `0x14`
  4. complex-display setup is built, trained, activated, and only then exported through the Device Path Property Database provider
- the GOP grouped-live subtype `20` and grouped packet stream still matter, but the current best interpretation is:
  - they are upstream state that likely influence the lower selector-`5` query result
  - not something `CoreEG2` directly consumes through `qword_E668` to make the class-`0x14` decision

- practical consequence:
  - the next best static target is back on the GOP/private-backend side:
    - find which selector-`5` handler or path returns class `0x14`
    - and correlate that with the grouped-live subtype `20` / grouped packet stream

### Stronger bridge: selector `5` is a raw extended-window read into the GOP connector-state slot

- `CoreEG2` does not ask the GOP for a high-level abstract “5K class”
- the lower selector-`5` query uses the same backend dispatcher as `1265`, but in read mode:
  - backend method: `GopGraphicsConnectorBackendDispatchOpcode`
  - transport: full-opcode path (`a4 == 1`)
  - operation: `GopReadExtendedOpcodeWindow(opcode = 5, len = 1, ...)`
- `GopRefreshConnectorStateSlotFromExtendedWindows` caches extended windows directly into the connector-state slot:
  - window `0x00 .. 0x0F` -> `slot + 12 .. slot + 27`
- therefore a direct read of selector/opcode `5` corresponds to the same raw cached byte at:
  - `slot + 12 + 5`
  - i.e. `slot[17]`

### Important Apple-side correction: selector `5` is only the status gate, not the final class byte

- in `CoreEg2SetSignalForDisplay`, the root-family setup performs **two** lower reads, not one:
  1. selector/opcode `5`, length `1`
     - output buffer: single byte at local `v48[0]`
     - success condition:
       - backend call succeeds
       - and `(v48[0] & 6) == 2`
     - practical meaning:
       - this is only a readiness/status gate
       - it is **not** the later paired/root class byte
  2. selector/opcode `0`, length `16`
     - output buffer: 16-byte block at local `v50`
     - if one returned fallback flag byte (`var_C2`) is negative, `CoreEG2` retries with selector/opcode `0x2200`
     - after that:
       - first byte of the returned block is stored at `display + 0x12C`
       - second byte of the block becomes the later link-rate/code byte used by `CoreEG2`
     - because this is a full-opcode extended-window read at base `0`, it lines up with the same first 16-byte window that the GOP refresh path caches at:
       - `slot + 12 .. slot + 27`
     - practical interpretation:
       - paired/root class byte at `display + 0x12C` is very likely the same low-level cached byte as `slot[12]`
       - the selector-`5` status gate is very likely the sibling byte `slot[17]`

- practical consequence:
  - the earlier shorthand “selector `5` returns class `0x14`” was too compressed
  - the better model is:
    - selector `5` proves the lower grouped path is in the right state family
    - the subsequent 16-byte classification read provides the actual class byte and neighboring mode fields
  - the paired/root Apple success path still effectively wants the top class value `0x14`, but that value comes from the **second** lower read

- practical consequence:
  - the Apple paired/root gate is reading a live low-level connector-state byte
  - it is **not** just reading the shared AC5E provider or a synthesized display property
  - this makes the remaining question much sharper:
    - what values can `slot[17]` take on this iMac
    - and which combination with the grouped-live latch (`slot[25] -> a1+18932`) yields the successful paired path

### Why `slot[17]` matters even without dynamic tracing

- the grouped training path already consumes `slot[17]` directly:
  - in `GopPrepareDpTrainingStateForSelectedMask`, grouped-live `0xF00` uses `slot[17] & 1` to choose whether the active grouped sub-block is under:
    - `+0x400`
    - or `+0x500`
- it then validates the chosen grouped sub-block against the exact byte pattern:
  - `[+0] == 0`
  - `[+1] == 16`
  - `[+2] == 250`
- grouped-live still separately depends on:
  - `slot[25]`
  - published type `3840`
  - and the resulting latch `a1 + 18932`
- the same grouped training path later also uses:
  - `slot[25] & 1`
  - to populate an extra grouped state bit after the sub-block validation succeeds

- practical consequence:
  - the Apple selector-`5` byte and the GOP grouped-live path are now proven to come from the same local slot-family
  - the likely split is no longer “policy blob vs hardware state”
  - it is now much closer to:
    - generic grouped/generic eDP state
    - versus a specific low-level connector-state-byte pattern that unlocks the fully paired/live path

### One caveat to revalidate later on the Apple side

- the current Apple notes still describe:
  - selector-`5` status gate `(reply[0] & 6) == 2`
  - and a later accepted “class” in `0x10 .. 0x14`
- after proving selector `5` is a direct 1-byte extended-window read, those two details may not both refer to the same returned byte
- practical consequence:
- the exact Apple-side interpretation of the selector-`5` result should be rechecked in `CoreEG2`
- but the new GOP-side result remains solid:
  - selector `5` reads the same low-level slot byte family that grouped 5K training already uses
  - and the local GOP code only consumes the low bit of `slot[17]`, so Apple may be using richer information from that byte than the GOP’s own grouped training helper does

### Better phrasing of the current 5K-vs-4K split

- the current best static model is now:
  1. GOP grouped path probes raw DP-family state and promotes it into grouped mask `0xF00`
  2. grouped-live depends on additional slot state:
     - `slot[17]` low-level grouped selector byte
     - `slot[25]` grouped-live latch source
     - grouped subtype `19 -> 20`
  3. Apple selector `5` reads that same low-level slot-family as a 1-byte status gate
  4. only if that gate is good does Apple ask for the 16-byte classification block
  5. the returned class byte at `display + 0x12C` must reach the top class `0x14`
  6. only then does `CoreEG2ResolvePostSignalDisplayRecord` enter the fully paired/root success lane

- practical consequence:
  - the fallback to 4K/generic preinit is now most plausibly:
    - grouped state never becomes fully live
    - or selector `5`/classification block never reaches the `0x14` paired/root class
  - this is a much tighter explanation than the earlier generic “policy difference” model

### Static lower-boundary reached: selector/class bytes are raw lower-transport results

- `GopReadExtendedOpcodeWindow` does not synthesize selector/opcode results locally
- it goes through `sub_1BA90`, which:
  - kicks the lower connector transaction engine
  - polls transport completion/status bits
  - and returns the raw lower response block
- practical consequence:
  - the Apple-facing bytes now mapped as:
    - selector `5` status byte
    - classification window `0` first byte (`display + 0x12C`)
  - should be treated as raw connector microprotocol results, not values invented by higher-level GOP logic

- this means the current static conclusion is meaningful:
  - the 5K-vs-4K split is no longer hidden in generic Apple policy code
  - it sits at the point where the lower connector microprotocol either:
    - returns the good grouped-live status and top paired/root class
    - or does not

## 2026-03-26 Probe-Boot Milestone: Firmware Really Hands Linux A Tiled State

### Probe boot summary

- using the diagnostics-style front door with the observational shim, firmware no longer handed Linux the old plain-4K-style preboot state
- in `Probe_results/bootprobe.txt`:
  - both GOP handles reported current resolution `5120x2880`
  - Apple shared display protocol `63FAECF2-...` was present
  - Apple platform provider `AC5E4829-...` was present
  - AMD connector protocol `CF611019-...` was present on 6 handles
- practical consequence:
  - the diagnostics/OCLP-style boot path really is preserving a richer preboot display state
  - the problem is no longer best described as “firmware never pre-initialized 5K”

### What Linux saw after that boot

- in `Probe_results/live_linux_5k_20260326_135819/xrandr-verbose.txt`:
  - current mode became `2560x2880`
  - only `eDP-1` was connected
- in `Probe_results/live_linux_5k_20260326_135819/drm_info.txt`:
  - `eDP-1` gained a real immutable `TILE` property
  - tile topology was:
    - group id `1`
    - number of tiles `2x1`
    - tile location `0,0`
    - tile size `2560x2880`
- in `Probe_results/live_linux_5k_20260326_135819/sys_class_drm/card1-eDP-1/edid.txt`:
  - EDID changed to DisplayID tiled-topology model `44582`
  - tiled block explicitly says:
    - `Num horizontal tiles: 2`
    - `Num vertical tiles: 1`
    - `Tile location: 0, 0`
    - `Tile resolution: 2560x2880`

### Why this is important

- baseline plain-Linux boots had:
  - single-panel-style EDID
  - no `TILE` property
  - preferred mode `3840x2160`
  - physical size about `600x340`
- probe-booted Linux instead had:
  - tiled EDID
  - `TILE` property present
  - preferred/current mode `2560x2880`
  - physical size about `300x340`

- practical consequence:
  - firmware is now proven to hand Linux at least tile `0,0` of the real paired/tiled panel state
  - Linux takeover is the side collapsing the preboot 5K-capable state into a single visible half-width tile
  - the remaining bug is now much more likely in Linux AMDGPU/DC takeover/runtime handling than in the firmware probe path itself

### New best working model after the probe boot

- direct Linux boot:
  - falls back before the tiled/pair-aware state is preserved into OS-visible EDID/topology
- diagnostics/probe boot:
  - preserves the tiled panel state far enough that Linux sees a real `2x1` tiled display descriptor
- current Linux takeover:
  - keeps only tile `0,0` as connected `eDP-1`
  - exposes `2560x2880`
  - does not surface the second tile as a second connected path
  - and does not aggregate both halves into a logical `5120x2880` desktop

### Implication for next work

- the firmware-side half of the problem is now far more constrained:
  - we do not need to guess whether the diagnostics-style path matters
  - it demonstrably does
- the highest-value next work is Linux-side:
  - understand why amdgpu/DC preserves only tile `0,0`
  - identify where the second tile path disappears
  - determine whether Linux needs:
    - MST/internal dual-link takeover support
    - tile aggregation support on this internal path
    - or a smaller runtime nudge on top of the already-good preboot state

### 2026-03-26 debugfs follow-up: Linux sees the tiled panel but does not engage aggregation

- in `Probe_results/live_linux_debugfs_20260326_141221/debugfs_dri`:
  - `eDP-1/internal_display` reported `Internal: 0`
  - `eDP-1/is_mst_connector` reported `no`
  - `eDP-1/mst_progress_status` reported `disabled`
  - `eDP-1/odm_combine_segments` returned `-95`
  - `eDP-1/link_settings` reported a real active DP link:
    - current `4 0x14 16`
    - verified `4 0x14 16`
    - reported `4 0x14 16`

- practical consequence:
  - Linux DC is not treating the active internal tile as an MST connector
  - and it is also not enabling ODM combine on the active `eDP-1` path
  - so even though the EDID now contains a real `2x1` tiled topology block, Linux is still operating the panel as a single visible tile with no active aggregation mechanism

- this gives a much tighter Linux-side hypothesis:
  - the diagnostics/probe boot gets us far enough to expose a tiled internal EDID/topology
  - but during takeover, amdgpu/DC leaves the active path as plain eDP with no MST progress and no ODM-combine support
  - that is consistent with the observed final mode `2560x2880` on tile `0,0`

### 2026-03-26 full DC state: kernel still has a 5120x2880 framebuffer, but userspace active state is only 2560x2880

- in `Probe_results/amdgpu_state.txt`:
  - only one active CRTC exists:
    - `crtc-0`
    - active mode `2560x2880`
  - only one connector is attached:
    - `eDP-1 -> crtc-0`
  - all other CRTCs are inactive and all `DP-*` connectors are detached
- the active primary plane is:
  - `plane-5 -> crtc-0`
  - framebuffer `96`
  - allocated by `kwin_wayland`
  - size `2560x2880`
- in `Probe_results/amdgpu_framebuffer.txt`:
  - framebuffer `97`
  - allocated by `[fbcon]`
  - still exists with size `5120x2880`

- practical consequence:
  - the kernel/console path did at least preserve a full-width `5120x2880` framebuffer object
  - but by the time the captured DRM state was taken, the active KMS configuration had already been reduced to a single `2560x2880` tile owned by `kwin_wayland`
  - this makes the current Linux-side collapse look even more like a takeover/userspace modeset issue than a pure firmware or early-kernel detection failure

- one weak but interesting hint in the same state dump:
  - detached `plane-4` still carried geometry matching the right half:
    - `crtc-pos=2560x2880+0+0`
    - `src-pos=2560x2880+2560+0`
  - because it had `fb=0` and `crtc=(null)`, this is not proof of an active second tile
  - but it is suggestive that the driver state may have recently contained a right-half mapping before the final single-tile configuration settled

### 2026-03-26 pre-GUI capture: Linux already drives the panel as a true two-head split before the compositor starts

- booting the same probe path with `systemd.unit=multi-user.target` produced a very different live DRM state:
  - two active CRTCs:
    - `crtc-0` at `2560x2880`
    - `crtc-1` at `2560x2880`
  - both active planes (`plane-4` and `plane-5`) scan out from the same `[fbcon]` framebuffer `97`
  - framebuffer `97` is full-width `5120x2880`
- the left/right split is explicit in the source rectangles:
  - `plane-5 -> crtc-0` uses `src-pos +0`
  - `plane-4 -> crtc-1` uses `src-pos +2560`
- connector mapping in the atomic state:
  - `eDP-1 -> crtc-0`
  - `DP-1 -> crtc-1`

- practical consequence:
  - before any graphical compositor starts, Linux is already driving the panel as two simultaneous `2560x2880` heads over a shared `5120x2880` fbcon framebuffer
  - so the collapse to a single visible `2560x2880` tile does **not** happen during early firmware handoff or initial amdgpu console takeover
  - it happens later, when the graphical userspace stack reprograms the display state

- this also explains the earlier Wayland capture much better:
  - pre-GUI:
    - two active CRTCs
    - shared `5120x2880` fbcon buffer
    - explicit left/right source split
  - later Wayland capture:
    - only `crtc-0` remains
    - only `eDP-1` remains attached
    - active framebuffer becomes a `2560x2880` `kwin_wayland` surface

- best current project conclusion:
  - the diagnostics/probe boot already gives Linux enough state for a functional two-half pre-GUI display path
  - the current failure is most likely in the graphical userspace/KMS takeover path:
    - compositor modeset choice
    - connector selection
    - or amdgpu/DC userspace-facing enumeration after the text-console state is already good

## 2026-04-29 Windows RE Checkpoint: `1265` Is A DP AUX/DPCD Sink-Specific Write

### Main correction

- the Windows lower "opcode" path has now been descended past the display-entry wrapper
- the old `opcode` name was useful while tracing call sites, but in the Windows backend it resolves to a DP AUX/DPCD transaction address
- the famous `1265` value is therefore:
  - decimal `1265`
  - hex `0x4F1`
  - a DP AUX/DPCD address in the sink device-specific DPCD range

- practical consequence:
  - the likely receiver of `1265` is not UEFI runtime firmware
  - and not a generic Windows-only firmware mailbox
  - it is most likely the internal panel/TCON sink-side microcontroller over the selected link's AUX channel

### Exact Windows call chain

- write path:
  1. `LowerControllerDpcdAuxWriteAddress` = `0x141EDFC70`
  2. `WindowsDM_LowerDpcdAuxWriteThunk` = `0x1419FFFB0`
  3. `DcLinkDpcdAuxWrite` = `0x141B7C780`
  4. `DceAuxWriteTransaction16` = `0x141BAF1F0`
  5. `DceAuxTransferWithRetriesEntry` = `0x141BAE870`
  6. `DceAuxTransferWithRetries` = `0x141C11640`
  7. `DceAuxTransferRawHw` = `0x141C11370`
  8. `DceAuxSubmitChannelRequest` = `0x141C10410`
  9. `DceAuxGetChannelStatus` = `0x141C0FFA0`
  10. `DceAuxReadChannelReply` = `0x141C10170`

- read path:
  1. `LowerControllerDpcdAuxReadAddress` = `0x141EDDCD0`
  2. `WindowsDM_LowerDpcdAuxReadThunk` = `0x141A003B0`
  3. `DcLinkDpcdAuxRead` = `0x141B7BF30`
  4. `DceAuxReadTransaction16` = `0x141BAF2C0`
  5. then the same DCE AUX transfer engine

### Why this is solid

- `sub_141A2A050` initializes `WindowsDM + 0x110` as a self-pointer to the `WindowsDM` object
- `dc_construct_ctx` / `ConstructDcContextFromInitData` copies `init_data + 0x28` into `dc_context + 0x8`
- therefore the former lower wrapper:
  - dereferences `dc_context + 0x8`
  - reaches `WindowsDM + 0x110`
  - follows that back to the main `WindowsDM` vtable
  - dispatches:
    - read through vtable slot `+0x188`
    - write through vtable slot `+0x190`

- the write vtable slot is `WindowsDM_LowerDpcdAuxWriteThunk`
- it locks the Windows display manager mutex and calls:
  - `DcLinkDpcdAuxWrite(dc, link_id, address, payload, len)`
- that indexes:
  - `dc->links[link_id]`
  - then the per-link DDC/AUX service at `link + 0x140`
- the per-link service was created by:
  - `CreateDceAuxDdcService`
  - `ConstructDceAuxDdcService`

### AUX request decode

- `DceAuxWriteTransaction16` builds a request structure where:
  - byte `+0` = address space
  - byte `+1` = write/read operation
  - byte `+2` = final/chained flag
  - dword `+4` = address
  - dword `+8` = length
  - qword `+16` = payload buffer
  - qword `+24` = reply/status storage
  - dword `+32` = retry delay

- `DceAuxActionFromTransaction` maps those fields to standard AUX actions:
  - DP write -> `0x80`
  - DP read -> `0x90`
  - I2C-over-AUX write/read/MOT -> `0x00`, `0x10`, `0x40`, `0x50`

- the `1265` helper passes address-space `0`, so it becomes a native DP AUX transaction, not I2C-over-AUX
- the DCE AUX submit function then programs the normal AUX MMIO register set:
  - `AUX_SW_DATA`
  - `AUX_SW_CONTROL`
  - `AUX_SW_STATUS`
  - `AUX_INTERRUPT_CONTROL`
  - `AUX_ARB_CONTROL`

### Surrounding addresses are now meaningful too

- the pre-training writes around `1265` are not arbitrary firmware commands:
  - `0x300` is `DP_SOURCE_OUI`
  - `0x303` is immediately after the source OUI in the source device-specific DPCD block
  - `0x310` is AMD's `DP_SOURCE_TABLE_REVISION`
  - `0x340` is AMD's `DP_SOURCE_MINIMUM_HBLANK_SUPPORTED`
- `ProgramDpPreTrainingAuxState` therefore writes source-side DPCD identity/capability state before link training
- this lines up with the existing Linux source definitions in:
  - `include/drm/display/drm_dp.h`
  - `drivers/gpu/drm/amd/display/include/dpcd_defs.h`

### Linux source crosswalk for the descended Windows backend

- the descended Windows backend lines up with the normal AMD DC DCE AUX implementation:
  - Windows `DceAuxSubmitChannelRequest` matches Linux `submit_channel_request`
    - `drivers/gpu/drm/amd/display/dc/dce/dce_aux.c`
  - Windows `DceAuxGetChannelStatus` matches Linux `get_channel_status`
  - Windows `DceAuxReadChannelReply` matches Linux `read_channel_reply`
  - Windows `DceAuxTransferWithRetries` matches the same retry/status model used by Linux AUX transfer paths
- the register names and fields also match:
  - `AUX_ARB_CONTROL`
  - `AUX_SW_DATA`
  - `AUX_SW_CONTROL`
  - `AUX_SW_STATUS`
  - `AUX_INTERRUPT_CONTROL`
  - `AUX_SW_GO`
  - `AUX_SW_DONE`
  - `AUX_SW_REPLY_BYTE_COUNT`

- practical consequence:
  - this is no longer an opaque proprietary Windows firmware call at the final layer
  - it is AMD's normal hardware AUX engine
  - the Apple/private part is the chosen sink DPCD address and the required link/topology state around it

### Exact pre-training DPCD payloads recovered

- `ProgramDpPreTrainingAuxState` first reads DPCD `0x300` length `3`
- if the current bytes are not `00 00 1A`, it writes:
  - address `0x300`
  - length `3`
  - payload `00 00 1A`
- this is the source OUI write

- it then writes DPCD `0x303` length `9`
- the payload is constructed locally:
  - byte `0` = low byte of `dc_context + 0x28`
  - byte `1` = high byte of `dc_context + 0x28`
  - bytes `2..5` = zero
  - byte `6` = low byte of `dc_context + 0x50`
  - bytes `7..8` = zero
- this looks like the source device-specific identity/version block following `DP_SOURCE_OUI`

- it then writes DPCD `0x310` length `3`
- payload is:
  - if `dc_context + 0x50 >= 12`: `05 1D 03`
  - otherwise: `04 1D 03`
- this is the AMD `DP_SOURCE_TABLE_REVISION` area

- optional final write:
  - condition: `dc_context + 0x50 >= 12` and `dc_object + 0x4C != 0`
  - address `0x340`
  - length `1`
  - payload = low byte of `dc_object + 0x4C`
- this lines up with AMD's `DP_SOURCE_MINIMUM_HBLANK_SUPPORTED`

- practical consequence:
  - the Windows DP/eDP bring-up does a normal source DPCD publication sequence before link training and before any later sink-private `0x4F1` write
  - so a future Linux implementation should mirror this as DPCD/AUX operations on the correct secondary internal link, not as a separate firmware mailbox

### Runtime DPCD ordering around `0x4F1`

- the DP/eDP runtime order is now concrete in `amdkmdag.sys`:
  - `BringUpDisplayBySignalType` dispatches signal/type `128` into `BringUpEdpSignal128Thunk`
  - it dispatches signal/type `32` directly into `BringUpDpOrEdpWithLinkTraining`
  - that thunk reaches `BringUpDpOrEdpWithLinkTraining`
  - for signal/type `128`, before training, the driver calls the eDP HPD/AUX arming callbacks:
    - DisplayCore vfunc `+0x103F0`
    - DisplayCore vfunc `+0x103F8`
  - it then calls `ProgramDpPreTrainingAuxState`
  - then `PerformLinkTrainingWithRetries`
  - each training attempt repeats the same type-128 HPD/AUX arming in `PrepareLinkTrainingStateAndArmEdp128`
  - only after bring-up returns success does `BringUpDisplayBySignalType` mark the display live at `link/display + 0x378`
  - if the lower display object flag at `+0x5B4` is set, it sends `DPCD 0x4F1 = 1`

- the `0x4F1` helper itself is simple:
  - address `0x4F1`
  - payload byte `1`
  - length `1`
  - if the first AUX write fails, sleep `10 ms` and retry once
  - if the caller requested settling and the write succeeded, sleep `100 ms`

- there is also an upper WindowsDM helper:
  - `WindowsDM_EnableSecondaryTileIfRequired`
  - resolves the selected `dc->links[index]`
  - requires the connected pin descriptor to carry byte `+6` bit `2` or bit `3`
  - requires the selected link signal/type to be `128`
  - then sends the same `DPCD 0x4F1 = 1` helper

- follow-up on the lower display object flag:
  - the post-bring-up caller tests lower object flag `+0x5B4`
  - an IDA displacement search for `+0x5B4` found the meaningful DC-side read at `BringUpDisplayBySignalType`
  - no nearby direct setter in the same `0x141B...` bring-up cluster was found
  - current inference: the flag is probably populated earlier through object construction, copied descriptor/property state, or a table-backed object rather than a local post-training assignment

- companion DPCD addresses around this path:
  - `0x170`: one-byte control/status latch written by `BuildLinkStateAndWriteControlOpcodes`
  - `0x116`: conditional one-byte write when link state `+0x2D0 == 1`
  - `0x375`: conditional one-byte chunked write when link state `+0x2D4` is true
  - `0x2006`: read/write status area used by the HPD RX IRQ helper after `0x170` bit0 is present
  - `0x205`: `DP_SINK_STATUS`, polled by the stream-enable path after `0x4F1` when the pin has capability tag `0x40`

- interpretation:
  - `0x4F1` is still best understood as a sink-private panel/TCON DPCD latch
  - it is not the first thing Windows does
  - Windows first makes the special type-128/eDP path HPD/AUX-ready, publishes source DPCD state, trains the link, then asserts `0x4F1`
  - this matches the Linux goal: do not treat `0x4F1` as a firmware mailbox; treat it as a DPCD write that only matters on the correct live internal AUX/link instance

### Updated meaning of `1265`

- `1265` / `0x4F1` sits inside the sink device-specific DPCD region:
  - `DP_SINK_OUI` starts at `0x400`
  - `0x4F1` is still within that vendor/private sink area
- this strongly suggests `0x4F1` is an Apple/LG/internal-panel private DPCD byte
- Windows sends payload `1` after the special type-128/eDP bring-up succeeds
- firmware/CoreEG2 also sends `0x4F1` payload `1` on the paired-display success path and payload `0` on cleanup/reload paths

- best current interpretation:
  - `0x4F1 = 1` probably tells the internal 5K sink/TCON to enter or expose the paired/secondary-tile operating state
  - `0x4F1 = 0` probably clears or deasserts that state
  - but the write is not sufficient alone, because Windows only sends it after HPD/AUX readiness, pre-training DPCD source setup, and link training have already reached the right path

### Other Windows `0x4F1` immediate sites checked

- an immediate search found eight `0x4F1` / `1265` uses in `amdkmdag.sys`
- non-DPCD uses:
  - `0x140094920`: structure offset check at `object + 0x4F1`
  - `0x1400999D0`: structure offset write at `object + 0x4F1`
  - `0x141895E00`: HDCP log line number `1265`, not a DPCD address

- real additional `0x4F1` writers:
  - `WriteSinkDpcd4F1Payload1Retry` = `0x1414B03F0`
  - `StreamEnableWrite4F1AndPollSinkStatus` = `0x1414E7810`

- `WriteSinkDpcd4F1Payload1Retry`:
  - resolves a display/link object
  - gets a lower interface from `resolved_object + 0x18`
  - calls vfunc `+0x8` with:
    - address `0x4F1`
    - payload byte `1`
    - length `1`
  - retries once after `10 ms`

- `StreamEnableWrite4F1AndPollSinkStatus`:
  - checks display/link capability bits before sending
  - calls object at `a1 + 0x90`, vfunc `+0x8`, with:
    - address `0x4F1`
    - payload byte `1`
    - length `1`
  - retries once after `10 ms`
  - then, in a related condition, polls address `0x205`
    - Linux names this `DP_SINK_STATUS`
    - it loops up to `0x32` times with `1 ms` sleeps waiting for returned byte `1`

- practical consequence:
  - Windows has more than one runtime site that asserts sink-private DPCD `0x4F1 = 1`
  - this is not just an isolated helper bolted onto `EnableSecondaryTileIfRequired`
  - it is also present in stream/link enable style code paths
  - the immediate polling of `DP_SINK_STATUS` after one of these paths further supports the interpretation that this is normal AUX/DPCD traffic to the sink/TCON

### What this means for Linux replication

- we probably do not need to call UEFI runtime firmware from Linux
- the thing Windows does is expressible through the normal AMD AUX engine once the correct link and AUX service exist
- the hard part remains:
  - getting the secondary/internal type-128 path live enough that Linux has a valid AUX engine for the right half
  - then reproducing the source DPCD setup and sink-private `0x4F1` state transition at the right time

- this changes the static target from:
  - "find unknown firmware opcode receiver"
- to:
  - "identify the exact AUX link instance and DPCD transaction sequence Windows uses for the secondary internal tile"

### Link/session identity for the `0x4F1` write

- the link/session id passed into the lower AUX path is not hidden in the payload
- `SendSecondaryTileOpcode1265Payload1` calls:
  - `LowerControllerDpcdAuxWriteAddress(display_entry + 0x158, display_entry, 0x4F1, &payload, 1)`
- `LowerControllerDpcdAuxWriteAddress` reads the link/session id from:
  - `display_entry + 0x30`
- `ConstructLegacyConnectorDisplayEntry` initializes that field from constructor input:
  - `display_entry + 0x30 = input + 0x14`
- WindowsDM then indexes the link array through:
  - `DcGetLinkByIndex(dc, index)`
  - implemented as `dc + 0x3C0 + 8 * index`

- practical consequence:
  - the `0x4F1` write goes to whichever `dc->links[index]` entry was classified as object id `0x14` / signal `128`
  - so a Linux reproduction must not just write DPCD `0x4F1` somewhere
  - it must target the AUX engine belonging to the secondary/type-128 internal link
  - if plain Linux never constructs that link/AUX service, a raw `0x4F1` write from the visible `eDP-1` AUX path may hit the wrong sink side or simply do nothing useful

### How the type-128 link becomes AUX-capable in Windows

- `CreateLinksFromBiosObjectTable` builds the normal `dc->links[]` array from the BIOS object table
- for each physical connector it passes a small constructor input block:
  - `input + 0x10` = physical connector index from the BIOS object table
  - `input + 0x14` = runtime `dc->links[]` index
- `ConstructLegacyConnectorDisplayEntry` stores:
  - `display_entry + 0x30 = input + 0x14`
  - `display_entry + 0x150 = dc`
  - `display_entry + 0x158 = dc_context`
- the BIOS object service then maps the physical connector index to a connector object id:
  - `display_entry + 0x1D8 = bios_object_service->get_connector_object_id(index)`
- when that connector object id is display-family `3` with low byte `0x14`, Windows maps it to:
  - `display_entry + 0x38 = 128`
  - this is the same signal/type tested by the secondary-tile enable path before the DPCD `0x4F1` write

- the constructor also asks the lower connector-state/GPIO layer for a state handle:
  - `display_entry + 0x398 = LookupConnectorStateHandleByObjectId(dc_context + 0x58, object_id, dc_context + 0x68)`
- if that handle exists, Windows opens/reads it and records HPD/GPIO-style state:
  - `display_entry + 0x3C = hpd id/state`
  - `display_entry + 0x40 = extra connector state for type `0x13` / `0x14``
- for type `0x14`, one additional gate exists:
  - if `dc + 0x1C5` is false, Windows clears `display_entry + 0x3C`
  - the exact meaning of `dc + 0x1C5` is still unresolved, but it affects whether the type-128 connector keeps a normal HPD id

- once the connector is accepted, Windows creates the per-link DDC/AUX service:
  - `display_entry + 0x140 = CreateDceAuxDdcService({ object_id, dc_context, display_entry })`
- `ConstructDceAuxDdcService` asks the BIOS object service for I2C/DDC line information:
  - indirect call at service vtable `+0x18`
  - logs `"BIOS object table - i2c_line: %d"`
  - logs `"BIOS object table - i2c_engine_id: %d"`
- it then creates a DDC/GPIO pin-pair service from the firmware-provided line/mask info:
  - `CreateDdcGpioPinPairService(dc_context + 0x68, line_id, 1 << pin_index, optional_flag)`
- connector object low byte `0x14` or `0x0E` sets DDC/AUX service flag bit `1`
  - so the type-128 path is explicitly treated as AUX-capable, not as a random firmware side channel

### Transaction-time liveness gates before any DPCD write

- every raw DCE AUX transfer still has to pass runtime ownership/setup:
  - `DceAuxOpenEngineForDdcService`
- that function requires all of the following:
  - `DceAuxIsEngineAvailable` must not report DMCU ownership of the AUX engine
  - `OpenDdcPinPairForAuxMode(ddc_service, 3, 0)` must succeed
  - `DceAuxAcquireEngine` must acquire SW ownership of the hardware AUX engine
- `DceAuxAcquireEngine` mirrors Linux `dce_aux.c` closely:
  - reads `AUX_ARB_CONTROL`
  - rejects `AUX_REG_RW_CNTL_STATUS == 2`
  - enables `AUX_EN` if needed
  - optionally toggles AUX reset and waits for `AUX_RESET_DONE`
  - sets `AUX_SW_USE_AUX_REG_REQ`
  - requires final `AUX_REG_RW_CNTL_STATUS == 1`
- only after those gates pass can Windows submit the DPCD transaction through:
  - `DceAuxSubmitChannelRequest`

### Current static conclusion from the lower Windows path

- the receiver of `0x4F1` is not an EFI module and not a UEFI runtime service
- it is the sink/TCON side reached over the secondary/type-128 link's DPCD AUX path
- Windows can send `0x4F1` only after all of this exists:
  - BIOS object table exposes a physical connector object with low byte `0x14`
  - Windows maps that object to signal/type `128`
  - connector-state/GPIO lookup succeeds well enough for construction
  - per-link DDC/AUX service is created from firmware-provided I2C/DDC line info
  - DDC pins can be opened in AUX mode
  - the DCE AUX engine can be acquired by SW

- likely Linux failure points, in descending order of suspicion:
  - plain Linux never receives/models the secondary object-id-`0x14` connector as a second `dc->links[]` entry
  - Linux constructs only the visible eDP path and therefore has no correct secondary AUX service to target
  - the secondary DDC/AUX line info exists in firmware tables, but Linux does not associate it with a connector/link
  - the AUX engine/pin acquisition path exists but is never reached for the secondary tile
  - the sink-private `0x4F1` and source DPCD pre-training writes are missing even when the link exists

### Where the lower connector object info comes from

- the lower provider behind `dc_context + 0x58` is the AMD ATOM/VBIOS object-table service
- Windows creates it in `ConstructDisplayCore`:
  - if init data did not supply a service, it calls `CreateAtomBiosObjectTableService`
  - this tries `CreateAtomObjectTableServiceV3` first
  - if that fails, it falls back to `CreateAtomObjectTableServiceLegacy`
- both services read the mapped ATOM BIOS image through:
  - `AtomBiosPtrAtOffset`
  - a bounds-checked `vbios_base + offset` helper

- the vtable slots used by the link constructor are now identified:
  - vfunc `+0x00`: connector count
  - vfunc `+0x08`: connector index -> decoded connector object id
  - vfunc `+0x18`: connector object id -> I2C/DDC/AUX line info
  - vfunc `+0x20`: connector object id -> HPD id/info
  - vfunc `+0x48`: HPD id -> GPIO register/mask info
  - vfunc `+0xF8` on the newer parser: connector object id -> internal-display flags

- the important object id translation is now aligned with Linux naming:
  - Windows object low byte `0x14` maps to signal value `128`
  - Linux `CONNECTOR_ID_EDP` is `20` (`0x14`)
  - Linux `SIGNAL_TYPE_EDP` is `(1 << 7)` = `128`
- so the Windows "type 128" path is not a mystery connector type
  - it is the second eDP-class connector object exposed by the ATOM/VBIOS object table
  - the special part is not the numeric signal value
  - the special part is that Windows successfully constructs and uses the secondary eDP/AUX path

### Linux cross-check from source

- Linux has the same broad construction model:
  - `dc/core/dc.c:create_links`
  - reads `bios->funcs->get_connectors_number`
  - loops connector indices
  - calls `bios->funcs->get_connector_id`
  - passes `connector_index` and `link_index` to `link_create`
- Linux `dc/link/link_factory.c:construct_phy` then:
  - logs `"BIOS object table - link_id: %d"`
  - checks for a supported source encoder before DDC/HPD setup
  - creates `link->ddc = link_create_ddc_service`
  - fails if `link->ddc` or `link->ddc->ddc_pin` is missing
  - maps `CONNECTOR_ID_EDP` to `SIGNAL_TYPE_EDP`
- one Linux-specific gate to keep in mind:
  - for `CONNECTOR_ID_EDP`, `construct_phy` rejects a second eDP link when `smart_mux_version` is set and an eDP was already counted
  - we do not yet know whether this triggers on the iMac, but it is now a concrete static candidate to log/check

- practical implication for the next Linux patch is sharper now:
  - do not start by replaying `0x4F1`
  - first prove whether plain boot sees the second ATOM connector object `0x14`
  - if it sees it, identify which `construct_phy` gate drops it
  - if it does not see it, compare the VBIOS/object-table visibility between probe boot and plain boot

### Post-DDC Windows link construction gates

- after Windows creates the per-link DDC/AUX service, it still has normal link-construction gates:
  - DDC/AUX service pointer must be non-null
  - the service's backing DDC pin pair must be non-null
  - for connector object low byte `0x14` or `0x0E`, Windows may create a panel-control object
  - object-table vfunc `+0x10` gets source/encoder object index `0`
  - `GetDdcHwInstFromDdcService` derives the DDC hardware channel from the created DDC/AUX service
  - `GetHpdSourceFromConnectorState` derives HPD source from connector-state/GPIO records
  - `TranslateEncoderObjectToTransmitter` maps the source/encoder object to a transmitter enum
  - resource/link-encoder creation must succeed
- if link-encoder creation fails, Windows aborts link construction and destroys the DDC/AUX service it just created

- useful Windows/Linux difference:
  - Windows creates the DDC/AUX service before link encoder creation
  - Linux checks for a supported source encoder before DDC/HPD setup in `construct_phy`
  - so one plausible Linux-only failure is an early unsupported-encoder/transmitter gate that prevents the secondary eDP DDC/AUX service from ever being built

### Type-128 readiness byte and runtime 5K gates

- the previously unresolved `display_entry/link + 0x58` byte is now explained:
  - `RefreshType128LinkReadyFlags` = `0x141B78FE0`
  - it calls `CollectType128LinksMax2` = `0x141B7B3D0`
  - that collector scans `dc->links[]` and selects only links whose runtime signal/type is `128`
  - it stops after two matching links, which fits the paired internal-panel use case

- `RefreshType128LinkReadyFlags` then updates each selected link:
  - if `DisplayCore + 0x1BE` is set, it forcibly clears `link + 0x58 = 0`
  - otherwise it calls `QueryConnectorReadyAndArmEdp128`
  - the returned connector-state/HPD readiness boolean is stored as `link + 0x58`

- the source of `DisplayCore + 0x1BE` was also traced:
  - `PopulateDcInitDataFromWindowsDM` sets `dc_init_data + 0x7E`
  - that value comes from adapter/config flags at `adapter_object + 0x70`, bit `0x80000`
  - `ConstructDisplayCore` copies `dc_init_data + 0x78..0xA7` into `DisplayCore + 0x1B8..`
  - therefore `dc_init_data + 0x7E` becomes `DisplayCore + 0x1BE`
  - practical interpretation: this is a Windows/config policy bit that disables type-128 readiness refresh when set

- `QueryConnectorReadyAndArmEdp128` = `0x141B704F0` does more than a passive read:
  - for signal/type `8`, it returns ready immediately
  - for signal/type `128`, it first calls two DisplayCore callback slots:
    - `DisplayCore + 0x103F0`
    - `DisplayCore + 0x103F8`
  - the DCE implementations behind those slots are:
    - `Dce110EdpWaitForHpdReady` = `0x141CEFB20`
    - `Dce110EdpPostHpdReadyDelay` = `0x141CEF6A0`
  - only after those callbacks does it resolve connector-state/HPD by object id and return the readiness boolean

- `Dce110EdpWaitForHpdReady` confirms the type-128 meaning:
  - it decodes the connector object low byte with `GetDisplayObjectLowByteIfDisplayFamily`
  - it only performs the HPD wait when that low byte is `20` (`0x14`)
  - it resolves the connector-state handle by object id
  - it waits for the HPD/readiness state to match the requested value
  - timeout is `300 ms` when enabling and `500 ms` otherwise

- the link-detect path now has a concrete type-128 order:
  - `DcLinkDetectEntryWithEncoderSequencing` wraps `DcLinkDetectHelper`
  - `DcLinkDetectHelper` calls `QueryConnectorReadyAndArmEdp128`
  - if that readiness query fails, detection returns false before EDID/sink creation
  - if a sink object is already present on signal/type `128`, Windows takes a fast path:
    - `ProgramDpPreTrainingAuxState`
    - `30 ms` wait
    - follow-up DP/eDP helper
    - return success without full rediscovery

- link training repeats the same type-128 arming:
  - `BringUpDpOrEdpWithLinkTraining` calls the `+0x103F0/+0x103F8` callbacks before pre-training DPCD setup
  - `PrepareLinkTrainingStateAndArmEdp128` repeats those callbacks before each training attempt
  - so Windows does not treat the secondary eDP path as "ready once and forever"
  - it reasserts HPD/AUX readiness before training and retry logic

### Pin descriptor bits that authorize `0x4F1`

- the 16-byte pin descriptor helper is now named:
  - `CopyPinDescriptor16` = `0x141A45630`
  - it copies `pin + 0x40` into a caller buffer
- the companion property-record lookup helper is now named:
  - `FindPinPropertyRecordByTag` = `0x141A45660`
  - it scans the pin record list at `pin + 0x38`
  - records are keyed by a 32-bit tag id at record offset `0`

- `WindowsDM_EnableSecondaryTileIfRequired` gates the `0x4F1` write on the connected pin descriptor:
  - byte `+6`, bit `2` (`0x04`)
  - or byte `+6`, bit `3` (`0x08`)
  - then the selected `dc->links[index]` must have signal/type `128`
  - only then does it call `SendSecondaryTileOpcode1265Payload1`

- those same descriptor bits appear in the independent stream-enable path:
  - `StreamEnableWrite4F1AndPollSinkStatus` checks descriptor byte `+6`, bit `2`
  - if not set, it checks descriptor byte `+6`, bit `3`
  - if either is set, it writes DPCD `0x4F1 = 1`
  - it retries once after `10 ms` if the first write fails

- `StreamEnableWrite4F1AndPollSinkStatus` has one more related descriptor bit:
  - descriptor byte `+6`, bit `5`
  - this gates the follow-up DPCD `0x205` poll
  - it polls `DP_SINK_STATUS` up to `0x32` times with `1 ms` sleeps until the byte becomes `1`

- `sub_141EDFFD0`, the display-info builder fed by this pin graph, maps some descriptor bits into output fields:
  - descriptor byte `+6`, bit `3` triggers property tag `0x3B`
  - this gives the bit a real descriptor/property-table backing, not just a local branch in the 0x4F1 helper
  - descriptor byte `+6`, bit `2` is the companion tag `0x3A`, and is now confirmed in multiple 0x4F1 paths

Additional refinement:

- the descriptor bit setter is now named:
  - `SetPinDescriptorCapabilityBitById` = `0x141A40B50`
  - it maps a capability/property tag directly into the 16-byte pin descriptor bitset at `pin + 0x40`
- the relevant tag-to-bit mapping is now concrete:
  - tag `0x3A` / decimal `58` sets descriptor byte `+6`, bit `2`
  - tag `0x3B` / decimal `59` sets descriptor byte `+6`, bit `3`
  - tag `0x40` / decimal `64` sets descriptor byte `+6`, bit `5`
- therefore the Windows `0x4F1` authorization is no longer just an anonymous bit test:
  - `0x4F1 = 1` is gated by pin capability tag `0x3A` or `0x3B`
  - the later `DPCD 0x205` poll is gated by pin capability tag `0x40`
- the upstream builders are now named:
  - `ApplyVendorProductCapabilityTableToPin` = `0x141A44600`
  - `ParseDisplayDescriptorAndSetPinCapabilities` = `0x141A44AC0`
  - `ApplyPinCapabilityEdidFixups` = `0x141A456E0`
- `ApplyVendorProductCapabilityTableToPin` extracts panel vendor/product information from the EDID-like blob and matches it against a capability table at `pin + 0x50`
  - exact vendor/product, vendor + wildcard product, and full wildcard entries are supported
  - a matched record is converted to a tag, inserted into the pin property list, and passed to `SetPinDescriptorCapabilityBitById`
- the capability table object is now named in IDA:
  - `ConstructVendorProductCapabilityTables` = `0x141ABCBD0`
  - `VendorProductCapabilityTablesGetCount` = `0x141ABC9D0`
  - `VendorProductCapabilityTablesGetRecord` = `0x141ABC8D0`
  - `InitCapabilityTableDescriptor` = `0x141ABCBA0`
  - `MapCapabilityTableIdToPinPropertyTag` = `0x141A41A90`
- the static table pointer at `0x144C4E080` resolves to table data at `0x144C4F720`
  - this table has `361` records
  - each record is four DWORDs:
    - EDID manufacturer id
    - EDID product id
    - capability-table id
    - value
  - the capability-table id is translated through `MapCapabilityTableIdToPinPropertyTag`
- the relevant Apple panel records are a very strong clue:
  - `APP / AE25` maps to source id `0x38`, translated tag `0x3A`, value `0`
  - `APP / AE26` maps to source id `0x39`, translated tag `0x3B`, value `0`
  - the same odd/even pairing appears for several Apple panel ids:
    - `AE01` -> tag `0x3A`, `AE02` -> tag `0x3B`
    - `AE0D` -> tag `0x3A`, `AE0E` -> tag `0x3B`
    - `AE19` -> tag `0x3A`, `AE1A` -> tag `0x3B`
    - `AE25` -> tag `0x3A`, `AE26` -> tag `0x3B`
  - our plain Linux EDID is `APP / AE25`
  - the probe/diagnostics tiled EDID model was `44582` / `0xAE26`
- interpretation of this split:
  - `APP / AE25` looks like the plain primary/internal panel identity
  - `APP / AE26` looks like the paired/tiled companion identity
  - Windows capability tag `0x3A` sets descriptor byte `+6`, bit `2`
  - Windows capability tag `0x3B` sets descriptor byte `+6`, bit `3`
  - both byte `+6`, bit `2` and byte `+6`, bit `3` are exact gates for `DPCD 0x4F1 = 1`
  - so the table statically connects both the plain `APP / AE25` identity and the tiled/probe `APP / AE26` identity to the later secondary-tile DPCD latch
  - the important missing piece in plain Linux is therefore not just EDID product id; it is the Windows-style capability processing plus the correct live type-128/AUX link instance
- `ApplyPinCapabilityEdidFixups` then iterates the resulting property list:
  - tags `0x3A` and `0x3B` call `DoubleEdidPhysicalWidthForTiledCaps`
  - that helper doubles EDID byte `21` / horizontal physical size when it is smaller than byte `22` / vertical physical size
  - it then recomputes the EDID block checksum with `RecomputeEdidBlockChecksum`
- interpretation:
  - tags `0x3A` and `0x3B` appear to describe a tiled/half-width internal panel EDID condition
  - those same tags both authorize the later sink-side `0x4F1` write
  - this links the EDID/tile interpretation path and the AUX/TCON enable path through the same pin capability IDs

### DAL mode-manager 5K tiled option

- a second Windows 5K mechanism was found outside the AUX writer:
  - `ConstructDalModeManagerWith5kTiledOption` = `0x141A3DA00`
  - it reads DAL option string `DalEnable5kTiledMode`
  - if the option DWORD is `1`, it sets the mode-manager flag at object `+0x80`
  - because the public interface pointer is object `+0x28`, this is visible to methods as interface `+0x58`

- the flag is consumed by:
  - `ModeManagerAddDerivedModesIncluding5kTile` = `0x141A3C4E0`

- the relevant mode synthesis branch is very specific:
  - requires mode-manager flag bit `7` at object `+0x50`
  - requires source timing flag bit `7` in the timing record
  - requires the source timing to be `1920x2160`
  - first adds a derived `1280x1440` mode
  - if mode-manager flag bit `5` is also set and `DalEnable5kTiledMode` is true, it adds `2560x2880`

- important implication:
  - Windows has a mode-list/tile-geometry path separate from the DPCD/AUX `0x4F1` path
  - `0x4F1` appears to be a sink/TCON side enable or latch on a live type-128/eDP link
  - `DalEnable5kTiledMode` controls whether the DAL mode solver exposes/adds the `2560x2880` half-tile mode derived from a `1920x2160` source timing
  - this matches our Linux probe-boot symptom where the system could land on a visible `2560x2880` half/tile state

- nearby evidence:
  - `DoubleEdidPhysicalWidthForTiledCaps` patches tiled-capability EDID physical size for tags `0x3A/0x3B`
  - `InjectLenovoYoga27Mv270QumFallbackEdid` contains a hardcoded fallback EDID for a Lenovo Yoga 27 / `MV270QUM` panel and injects 256 bytes when the EDID buffer is blank
  - that fallback EDID is probably not the Apple iMac panel path, but it proves this same capability layer contains panel-specific internal-display EDID fixups

### 5K60 pipe-split timing classification

- another independent Windows-side 5K mechanism was found in the mode timing path:
  - `MarkTimingPipeSplitForHighResAnd5k60` = `0x1415D5380`
  - `ModeListContainsPipeSplitTiming` = `0x1415CAF60`

- `MarkTimingPipeSplitForHighResAnd5k60` checks option IDs from the DAL option table:
  - option id `0x587` = `DalSupportMultiPipe`
  - the explicit `5120x2880@60` branch checks option id `0x878`
  - corrected table mapping in this build:
    - `0x875` = `DalVSR21x9`, default `0`
    - `0x876` = `Dal5K60PipeSplit`, default `1`
    - `0x878` = `DalEnableLTTPR`, default `1`
  - therefore the earlier label "`0x878` = `Dal5K60PipeSplit`" was wrong for this binary

- the explicit 5K60 branch is:
  - timing width field `>= 0x1400` / `5120`
  - timing height field `>= 0x0B40` / `2880`
  - refresh field equals `60`
  - option id `0x878` returns enabled
  - then the timing record is marked with low bits `2` at timing record field `+0x24` / `record[9]`

- `ModeListContainsPipeSplitTiming` later treats `record[9] & 3 > 1` as evidence that the mode list contains a pipe-split timing

- the same helper also marks:
  - very tall/high-res tiled-looking modes when `DalSupportMultiPipe` is enabled
  - 8K-ish timings `>= 7680x4320`
  - the timing record's flag bit `7` at field `+0x1C` when the timing aspect ratio matches the display dimensions within the helper's 90%-110% window

- interpretation:
  - Windows has an explicit DAL-level `5120x2880@60 -> pipe split` classifier
  - the option-name mapping is slightly non-obvious: the `5120x2880@60` branch checks `0x878`, while `Dal5K60PipeSplit` itself is `0x876` and feeds the mode-solution flags through the mode manager
  - this is separate from `DalEnable5kTiledMode`, which injects/derives a `2560x2880` half-tile mode
  - it is also separate from the AUX/TCON `0x4F1` write
  - together they explain why Windows can understand both the logical 5K mode and the physical split/tile plumbing

- mode-manager flag sources relevant to the `2560x2880` derived tile path:
  - flag bit `5` comes from `AdapterServiceAllowsDownscaleBeyondNative` = `0x141460F50`
  - flag bit `6` comes from option id `0x863` = `DalHighResDownScaleSupport`
  - flag bit `7` comes from option id `0x461` = `DalScalerBoundaryOption`
  - flag bit `9` comes from option id `0x876` = `Dal5K60PipeSplit`
  - neighboring option id `0x875` is `DalVSR21x9`
  - the `2560x2880` injection requires bit `5`, bit `7`, a `1920x2160` source timing with its own tile flag, and `DalEnable5kTiledMode`

### Downscale/VSR gate behind mode-manager flag bit 5

- the unresolved mode-manager flag bit `5` is now traced:
  - `CreateAdapterServiceInterface` = `0x141468DF0`
  - `ConstructAdapterServiceObject` = `0x141468B10`
  - `InitializeAdapterServiceForConnector` = `0x141467800`
  - the adapter service returns its public interface as object `+0x28`
  - the constructor installs public vtable `off_141F9CB00`
  - public vtable entry `+0x3C8` points to `AdapterServiceAllowsDownscaleBeyondNative` = `0x141460F50`

- `AdapterServiceAllowsDownscaleBeyondNative` is a small predicate:
  - it calls adjacent vfunc `+0x3D0`
  - that target is `AdapterServiceGetDownscaleLimitPercent` = `0x141460D70`
  - it returns true only when the returned percent is greater than `100`

- `AdapterServiceGetDownscaleLimitPercent` resolves the percent from multiple inputs:
  - option id `0x421` = `DalOverrideDownscaleLimit`
  - connector/pin default downscale limit percent from the current CBasePin-like object at adapter-service `+0x148`
  - valid nonzero limits are clamped/accepted only in the `100..400` range
  - if both the default and override are present, the override wins

- the CBasePin default is now traced one level deeper:
  - `ConstructCBasePinWithCapabilityName` = `0x14152EED0`
  - `CreatePinCapabilityNameByConnectorType` = `0x1415B3590`
  - `CBasePinGetDownscaleLimitProperty27` = `0x14152D450`
  - CBasePin vfunc `+0x110` calls the pin's `Name()` / capability object with property id `27`
  - `ConstructDfpCapabilityNameType82` = `0x1415FF5C0` initializes property index/id `27` to `200`
  - `ConstructDfpCapabilityNameType8D` = `0x1415FEC20` also initializes property index/id `27` to `200`
  - this means the default path makes `AdapterServiceAllowsDownscaleBeyondNative` true (`200 > 100`) unless a disable option clamps it back down

- two options can force the limit back to `100`, disabling the flag-bit-5 predicate:
  - option id `0x58C` = `DALDisableDownscalingKickers`
  - option id `0x735` = `DalDisableVSR`
  - the `DalDisableVSR` branch also checks pin descriptor byte `+6`, bit `3`, which is capability tag `0x3B`

- interpretation:
  - mode-manager flag bit `5` means the adapter/pin allows a downscale/VSR-style limit greater than native scale, not simply that a hardcoded 5K option is enabled
  - this explains why the same bit gates a generic derived `3200x1800` mode and the later `2560x2880` 5K half-tile injection
  - the `DalDisableVSR + tag 0x3B -> force 100` branch ties the 5K/tiled pin-capability path back into the mode-manager scaling gate
  - for Linux, this is more evidence that Windows models the panel as a tiled/internal display with derived half-tile modes, not as a simple single eDP mode list

### Source timing flag bit 7

- the `ModeManagerAddDerivedModesIncluding5kTile` source timing flag bit `7` is also no longer mysterious:
  - it reads timing record field `+0x1C`, bit `7`
  - `MarkTimingPipeSplitForHighResAnd5k60` sets that same field/bit when timing aspect ratio matches the display dimensions within the 90%-110% window
  - the derived tile path then requires this bit together with mode-manager bit `7`, base timing `1920x2160`, and optionally mode-manager bit `5` plus `DalEnable5kTiledMode`

- interpretation:
  - the `1920x2160 -> 1280x1440 -> 2560x2880` path is guarded by both global mode-manager policy and a per-timing aspect/tile suitability bit
  - this per-timing bit likely prevents the 5K half-tile derivation from applying to arbitrary 1920x2160 timings

### Refined Windows-side sequence

- the current best static ordering is now split into two cooperating paths:

- link / sink enable path:
  - ATOM/VBIOS exposes connector object low byte `0x14`
  - Windows constructs a display/link entry for it and maps it to signal/type `128`
  - Windows builds the DDC/AUX service and connector-state/HPD handle for that object
  - startup refresh selects up to two type-128 links and sets `link + 0x58` from HPD/readiness
  - detect and training repeatedly call the DCE eDP HPD wait/delay callbacks for object `0x14`
  - pre-training DPCD source state is written
  - link training runs
  - pin descriptor bits `+6 & 0x0C` authorize `0x4F1 = 1`
  - stream enable may also poll `DP_SINK_STATUS` at DPCD `0x205`

- mode / tiled-geometry path:
  - the EDID/vendor-product/pin capability layer sets tags such as `0x3A/0x3B`
  - those tags trigger tiled EDID physical-width fixups and become the `0x4F1` authorization bits
  - the DAL mode manager reads `DalEnable5kTiledMode`
  - when the mode-manager flags and source timing match, it synthesizes a `2560x2880` half-tile mode from a `1920x2160` timing
  - the timing/mode-solution path consumes `Dal5K60PipeSplit` as a mode-solution flag
  - when a mode is `5120x2880@60`, Windows marks that timing as pipe-split / multi-pipe through the separate `0x878`-gated branch

- practical Linux implication:
  - if plain Linux has no second object-id-`0x14` link, it cannot naturally reproduce this sequence
  - if Linux creates the second link but has no connector-state/HPD/AUX readiness equivalent, training and DPCD writes will still be unreliable
  - if Linux gets the link ready but skips the source DPCD setup and `0x4F1` timing, the sink/TCON may still fail to expose the paired/tiled state
  - if Linux gets only the raw half/tile mode but lacks the aggregation/tiled-monitor representation, userspace can see a broken `2560x2880`-style state instead of a single coherent `5120x2880` monitor
  - therefore the next Linux target is not just "send 0x4F1"; it is "make the secondary eDP/type-128 path live, mirror the DPCD sequence at the same stage, and expose the correct tiled mode model"

### IDA database update

- IDA database saved:
  - `B416406/amdkmdag.sys.i64`
- key new names/comments were applied for:
  - WindowsDM lower read/write thunks
  - DC link DPCD/AUX read/write wrappers
  - DCE AUX transaction builder
  - DCE AUX transfer retry loop
  - DCE AUX submit/status/reply helpers
  - generic register read/write/update/wait helpers
  - additional runtime `0x4F1` writers and the follow-up `DP_SINK_STATUS` poll
  - DCE AUX engine acquire/release/availability helpers
  - DDC/GPIO pin-pair service creation/open/close helpers
  - BIOS object table link construction and type-128 connector comments
  - ATOM/VBIOS object table service constructors and vfuncs for connector count/object id/I2C/HPD/GPIO/internal-display flags
  - source-object/encoder/transmitter/link-encoder construction gates after DDC/AUX creation
  - type-128 link readiness byte refresh at `link + 0x58`
  - DCE eDP HPD wait and post-HPD settle callbacks behind DisplayCore slots `+0x103F0/+0x103F8`
  - pin descriptor byte `+6` bits that gate both secondary-tile and stream-enable `0x4F1` writes
  - `DalEnable5kTiledMode` mode-manager option and the `2560x2880` half-tile mode synthesis path
  - pin capability tags `0x3A/0x3B/0x40` and their relationship to EDID fixups, `0x4F1`, and `DPCD 0x205`
  - `Dal5K60PipeSplit` and the `5120x2880@60` timing classifier that marks pipe-split modes
  - adapter-service downscale/VSR gate behind mode-manager flag bit `5`
  - CBasePin property `27` default downscale limit path that initializes the relevant percent to `200`

### 2026-04-29 focused Windows RE follow-up

- corrected DAL option-table mapping around the 5K-related mode flags:
  - option id `0x875` = `DalVSR21x9`, default `0`
  - option id `0x876` = `Dal5K60PipeSplit`, default `1`
  - option id `0x878` = `DalEnableLTTPR`, default `1`
  - earlier notes that labeled `0x878` as `Dal5K60PipeSplit` were wrong for this binary

- the `5120x2880@60` pipe-split branch in `MarkTimingPipeSplitForHighResAnd5k60` still checks option id `0x878`:
  - in this build that means the branch is gated by the option-table entry named `DalEnableLTTPR`
  - `Dal5K60PipeSplit` is instead consumed earlier when `sub_141429530` builds the ModeMgr config flags
  - `sub_141429530` reads option id `0x876` and stores it as ModeMgr flag bit `9`
  - `CreateModeSolutionWith5KPipeSplitOptionFlags` then folds ModeMgr flag bit `9` into bit `1` of the mode-solution constructor flags
  - `ConstructModeSolutionWithPipeSplitFlags` stores those mode-solution flags at object `+0xB8`

- `dc_sink +0x5B4` source now traced:
  - `DcLinkDetectHelper` creates a sink with `CreateDcSinkForLink` and assigns it to the link/display entry at `+0x28`
  - `BringUpDisplayBySignalType` later reads that same pointer and tests `dc_sink +0x5B4` before calling `SendSecondaryTileOpcode1265Payload1`
  - `CreateDcSinkForLink` allocates a zeroed `0x690`-byte sink, so `+0x5B4` defaults to `0`
  - `ReadEdidAndPopulateDcSink` calls `ParseEdidAndPinCapsIntoDcSinkInfo` with destination `dc_sink +0x508`
  - inside that parser, local parsed-info `+0xAC` becomes `dc_sink +0x5B4`
  - if connected pin descriptor byte `+6`, bit `3` is set, the parser finds pin property tag `59` / `0x3B` and copies that record's value field into parsed-info `+0xAC`
  - therefore `dc_sink +0x5B4` is not a mysterious lower-firmware flag; it is the sink copy of pin capability/property tag `0x3B`'s value

- implication for the `0x4F1` paths:
  - the post-bring-up `BringUpDisplayBySignalType` reassertion path requires the tag `0x3B` property value to be nonzero
  - the other two known `0x4F1` writers do not require that value; they gate on descriptor byte bits directly
  - this means a tag can authorize `0x4F1` by presence, while the post-bring-up reassertion path uses the tag value as an additional policy/value gate
  - the static Apple `APP/AE26 -> tag 0x3B, value 0` record therefore may set the descriptor bit but still leave this specific post-bring-up `dc_sink +0x5B4` reassertion disabled unless another path overrides the value

- additional IDA names applied:
  - `DpcdAuxWriteIfDisplayPresent` = `0x141AFF4A0`
  - `DpcdAuxReadIfDisplayPresent` = `0x141AFF430`
  - `ChunkedLowerControllerDpcdWrite` = `0x141BB92D0`
  - `ChunkedLowerControllerDpcdRead` = `0x141BB9340`
  - `CreateModeSolutionWith5KPipeSplitOptionFlags` = `0x141A3C920`
  - `ConstructModeSolutionWithPipeSplitFlags` = `0x141ACF560`
  - `IsDalOptionSupportedByDriverVersion` = `0x141460680`
  - `CreateDcSinkForLink` = `0x141B9CE40`
  - `ConstructDcSinkForLink` = `0x141B9CC40`
  - `ReadEdidAndPopulateDcSink` = `0x141BB2E70`
  - `ReadEdidIntoDcSinkThroughPin` = `0x141EDDD50`
  - `ParseEdidAndPinCapsIntoDcSinkInfo` = `0x141EDFFD0`
  - `ReadMccsCapsIntoDcSink` = `0x141EDEE30`
  - `ReplaceDcLinkSink` = `0x141B6FEC0`
  - `ClearDcLinkSink` = `0x141B6FF00`

- current interpretation:
  - no new evidence was found that the Windows 5K runtime path requires a UEFI runtime service or Apple firmware call
  - the Windows path still looks like two cooperating mechanisms:
    - a live internal/type-128 AUX link path that performs source DPCD setup, link training, and sink-private DPCD `0x4F1`
    - a DAL mode/timing path that exposes `2560x2880` half-tile and `5120x2880@60` pipe-split/tiled modes
  - the next narrow RE target, if we keep descending, is no longer the `+0x5B4` setter; it is the relationship between tag `0x3A`/`0x3B` presence/value and which of the three `0x4F1` writers actually runs for the Apple panel

### 2026-04-29 0x4F1 writer reachability follow-up

- the known Windows `0x4F1` writers now split into distinct lifecycle phases:
  - detect/status phase:
    - `DetectDisplayMaybeAssert4F1ForCapableSink` / `ProbeDisplayStatusMaybeAssert4F1`
    - calls `WriteSinkDpcd4F1Payload1Retry`
    - checks connected pin descriptor byte `+6`, bit `2`
    - this is capability/property tag `0x3A`
    - after writing `DPCD 0x4F1 = 1`, it waits `100 ms` and re-reads display status
  - stream-enable phase:
    - `StreamEnableWrite4F1AndPollSinkStatus`
    - reached by `EnableValidatedStreamMaybeAssert4F1`
    - checks descriptor byte `+6`, bit `2` or bit `3`
    - therefore accepts either tag `0x3A` or tag `0x3B`
    - writes `DPCD 0x4F1 = 1`, retries once after `10 ms`
    - if descriptor byte `+6`, bit `5` / tag `0x40` is present, it polls `DPCD 0x205` up to `50` times waiting for byte value `1`
  - WindowsDM / secondary-tile helper phase:
    - `WindowsDM_EnableSecondaryTileIfRequired`
    - checks descriptor byte `+6`, bit `2` or bit `3`
    - additionally requires the selected DC link/display object to have signal/type `128`
    - calls `SendSecondaryTileOpcode1265Payload1`
    - this helper is referenced through tables / virtual dispatch rather than by a simple direct code caller
  - post-bringup reassertion phase:
    - `BringUpDisplayBySignalType`
    - after successful signal bring-up it checks `dc_sink +0x5B4`
    - `dc_sink +0x5B4` is copied from tag `0x3B`'s value field
    - this path requires the tag value to be nonzero, not merely the descriptor bit

- the tag `0x3B` detect override is newly important:
  - `DetectDisplayAndApplyApple5kOverrides` has a special branch on connected pin descriptor byte `+6`, bit `3`
  - if this bit is set, the underlying display object reports physically connected, the current detect result is not connected, and the reported signal type is `11`, Windows forces the detect result back to connected and clears status byte `+112`
  - `SignalTypeToString` confirms signal type `11` = `DisplayPort`, `12` = `DisplayPortMst`, and `13` = `EDP`
  - this means tag `0x3B` is not only a permission bit for the later `0x4F1` write
  - it also affects whether a physically-present DP-side Apple/tiled display path survives the ordinary display-detect state machine

- practical implication:
  - tag `0x3A` appears to be enough for the detect-time `0x4F1` assertion
  - tag `0x3B` appears to be the stronger Apple paired/tiled path:
    - it authorizes stream-enable and WindowsDM `0x4F1`
    - it can force a physically-present DisplayPort signal object back to connected
    - its value field feeds `dc_sink +0x5B4`, which controls post-bringup reassertion
  - the static Apple table entry for `APP/AE26` maps source id `0x39` / decimal `57` to property tag `0x3B`
  - the static Apple table entry for `APP/AE25` maps source id `0x38` / decimal `56` to property tag `0x3A`
  - this keeps matching our Linux observations: plain boot seeing only the weaker/single-side shape and probe/diagnostics boot exposing richer tiled state is consistent with Windows having a special tag `0x3B` path that can preserve or force the paired side through detection

- new IDA names/comments applied in this pass:
  - `DetectMstDisplayMaybeAssert4F1` = `0x1414A85E0`
  - `DetectDisplayMainMaybeApple5k` = `0x1414A96C0`
  - `DetectDisplayAndApplyApple5kOverrides` = `0x1414B3E80`
  - `DetectDisplayEarlyExitForApple5kCaps` = `0x1414B31D0`
  - `ProbeDisplayStatusMaybeAssert4F1` = `0x1414B2490`
  - `DetectMstLinkEntryMaybeApple5k` = `0x141430050`
  - `EnableStreamValidatedWrapperMaybe4F1` = `0x1414F4C00`
  - `SignalTypeToString` = `0x1413E6480`
  - `IsDpMstOrEdpSignalType` = `0x1413E3150`
  - `IsEmbeddedPanelSignalLvdsOrEdp` = `0x1413E3190`
  - `IsDviSignalType1To3` = `0x1413E3B00`
  - `IsSignalTypeClass6To10` = `0x1413E72B0`

- current Linux-facing hypothesis after this pass:
  - the first Linux patch should not only synthesize two DRM connectors/tiles
  - it should also emulate the Windows decision model:
    - carry an Apple-panel capability equivalent to tag `0x3B`
    - keep the second/tiled side connected through early detection
    - perform the `0x4F1 = 1` DPCD transition at detect/stream-enable time on the correct live AUX path
    - optionally poll `DPCD 0x205` only if we can justify the equivalent of tag `0x40`

### 2026-04-29 Apple AE static capability table tightening

- exact byte-pattern checks in the static vendor/product capability table confirm:
  - `APP/AE25` has source capability id `0x38`, value `0`
  - `APP/AE26` has source capability id `0x39`, value `0`
  - through `MapCapabilityTableIdToPinPropertyTag`, source id `0x38` becomes property tag `0x3A`
  - source id `0x39` becomes property tag `0x3B`
  - there is no static `APP/AE25` or `APP/AE26` source id `0x40` record
  - there is no static `APP/AE??` source id `0x40` record in this table pattern

- implication:
  - for our observed `AE25` / `AE26` panel IDs, the static table explains the descriptor bits that authorize `DPCD 0x4F1 = 1`
  - the static table does not explain the tag `0x40` / descriptor byte `+6`, bit `5` path that would enable the post-`0x4F1` `DPCD 0x205` poll
  - therefore Linux should not blindly add a `0x205` poll just because Windows has that code path
  - a `0x205` poll should be added only if runtime EDID/descriptor parsing proves tag `0x40` exists for this exact boot state

- later Apple AE entries are also useful:
  - `APP/AE31` -> source id `0x38`, value `1`
  - `APP/AE32` -> source id `0x39`, value `1`
  - `APP/AE35` -> source id `0x38`, value `1`
  - `APP/AE36` -> source id `0x39`, value `1`
  - those nonzero `0x39` values would feed `dc_sink +0x5B4` and therefore enable the post-bringup `0x4F1` reassertion path
  - by contrast, `AE26` has tag `0x3B` present but value `0`, so for our observed tiled/probe model the presence-based writers are more important than the `dc_sink+0x5B4` value-based reassertion path

### 2026-04-29 pre-training DPCD placement follow-up

- the source-DPCD setup is not an isolated final-enable helper; `ProgramDpPreTrainingAuxState` has three meaningful callers:
  - `BringUpDpOrEdpWithLinkTraining` calls it after the type-128 HPD/AUX arm callbacks and before `PerformLinkTrainingWithRetries`
  - `DcLinkDetectHelper` calls it in the signal/type-128 fast path when a sink already exists, then waits `30 ms` and calls the follow-up DP/eDP helper
  - `RetrieveDpLinkCapsAfterSourceDpcdSetup` calls it during `retrieve_link_cap`, before the normal receiver-cap reads from DPCD `0x000` and extended caps from `0x2200`

- exact `retrieve_link_cap` order:
  - call lower helper `sub_141BBD5B0`
  - read DPCD `0x600` length `1`
  - if that read fails or returns byte `2`, wait `1000 us`
  - call `ProgramDpPreTrainingAuxState`
  - wait `30 ms`
  - then read receiver caps at DPCD `0x000` length `16`, with up to three tries
  - if extended caps are advertised, read DPCD `0x2200` length `16`

- exact `ProgramDpPreTrainingAuxState` behavior:
  - if display-core flag `+0x10604` is set, it uses a cached fast path:
    - write `12` bytes from `display_core +0x10605` to DPCD `0x300`
    - this covers DPCD `0x300..0x30B`
    - it skips the generated `0x310` and optional `0x340` writes
  - otherwise it uses the generated path already recovered:
    - read DPCD `0x300` length `3`
    - if current bytes are not `00 00 1A`, write `00 00 1A` to DPCD `0x300`
    - write DPCD `0x303` length `9`
    - write DPCD `0x310` length `3`, payload `04 1D 03` or `05 1D 03` depending on link generation
    - optionally write DPCD `0x340` length `1` if the link generation is high enough and object field `+0x4C` is nonzero

- the lower wrappers still preserve native DPCD address semantics:
  - `ChunkedLowerControllerDpcdReadWithRangeFixup` expands only a special high DPCD range before chunked reads
  - that special expansion range is `0xF0000..0xF0007`
  - the chunk table also lives in high DPCD space around `0xF0000..0xF00FF`
  - therefore low addresses such as `0x300`, `0x303`, `0x310`, `0x340`, `0x4F1`, and `0x2006` are not remapped or reinterpreted by this wrapper layer

- direct lower AUX/DPCD write sites around this path are now bounded:
  - `SendSecondaryTileOpcode1265Payload1` writes DPCD `0x4F1 = 1`, retries after `10 ms`, and optionally waits `100 ms`
  - `BuildLinkStateAndWriteControlOpcodes` writes DPCD `0x170`, conditional `0x116`, and conditional `0x375`
  - `HandleDpcd2006StatusAfterHpdRx` reads `0x170`, then reads and acknowledges DPCD `0x2006`
  - generic chunked writes handle the source-DPCD block and ordinary DP training/control writes

- Linux-facing implication after this pass:
  - reproducing Windows more closely means publishing the source-DPCD identity/capability block on the correct live internal AUX path before link caps/training have fully settled, not only immediately before a later `0x4F1` write
  - the minimal Linux experiment should prefer the generated AMD-default path unless Linux has an explicitly valid `vendor_signature` override:
    - `0x300 = 00 00 1A`
    - `0x303` with the AMD source-specific bytes Linux can derive or conservatively zero where Windows does
    - `0x310 = 04/05 1D 03` depending on the same link-generation condition
    - defer `0x340` unless Linux exposes an equivalent minimum-HBlank value
  - this still does not make `0x4F1` a firmware mailbox; it remains a sink-private DPCD transition that has to occur on the right live link/tile after the source-DPCD and HPD/AUX readiness sequence

### 2026-04-29 Linux source-DPCD cross-check

- local Linux/AMD DC source already contains the public equivalent of much of `ProgramDpPreTrainingAuxState`:
  - `drivers/gpu/drm/amd/display/dc/link/protocols/link_dp_capability.c`
  - function `dpcd_set_source_specific_data`

- Linux currently writes:
  - if `link->dc->vendor_signature.is_valid`, one raw 12-byte write to DPCD `0x300`, matching the Windows cached fast path shape
  - otherwise, DPCD `0x300` / `DP_SOURCE_OUI` with AMD OUI `00 00 1A` if needed
  - otherwise, DPCD `0x303` / `DP_SOURCE_OUI + 0x03` with `struct dpcd_amd_device_id`
    - device id low byte
    - device id high byte
    - four zero bytes
    - `dce_version`
    - two DAL version bytes, currently zero
  - otherwise, optional DPCD `0x340` / `DP_SOURCE_MINIMUM_HBLANK_SUPPORTED`

- Linux call placement is also close to Windows:
  - `link_dp_capability.c` calls `dpcd_set_source_specific_data(link)` before the main receiver-cap read loop, then waits `30 ms`
  - `link_dpms.c` calls it during link enable because mode switches can lose OUI/source data
  - `link_dpms.c` also calls it on the eDP fast-boot path when `link->is_dds`
  - `link_detection.c` calls it for an eDP resume/OLED local-sink fast path

- important mismatch:
  - Windows explicitly writes DPCD `0x310` / `DP_SOURCE_TABLE_REVISION`
  - current local Linux source defines `DP_SOURCE_TABLE_REVISION` but no call site writes it
  - Windows writes `0x310` length `3` with payload:
    - `04 1D 03` for older/lower DCE generation
    - `05 1D 03` when `dc_context +0x50 >= 12`

- implication update:
  - the missing Linux behavior is not "no source-DPCD publication at all"
  - the likely missing pieces are:
    - ensuring this source-DPCD publication runs on the Apple secondary/tiled AUX path that plain boot currently fails to keep alive
    - adding/experimenting with the Windows-specific `0x310` table-revision write
    - only then asserting sink-private `0x4F1 = 1` on that same live AUX path

### 2026-04-29 stream-enable `0x4F1` interface provenance

- another full immediate sweep for decimal `1265` / hex `0x4F1` still finds only eight immediate sites:
  - two early sites are ordinary structure offset `+1265` uses
  - one site is an HDCP/MFT diagnostic source-line number in `HdcpBindMftIdSession`
  - the remaining sites belong to the three real display DPCD writers already under investigation

- the HDCP/MFT false positive is now named/commented in IDA:
  - `HdcpBindMftIdSession` = `0x141895E00`
  - its `1265` value is passed as a printed line number for `"BindMFTID:: Invalid input sessionID..."`
  - it is not a DPCD address and not part of the Apple 5K path

- the detect-time and stream-enable `0x4F1` writers share the same underlying per-display DPCD/AUX interface:
  - the display-map stores fixed `64`-byte entries
  - `FindDisplayEntryByObjectId` compares the object id against the entry descriptor at `entry +0x08`
  - it returns the 64-byte map entry, not the raw display object
  - `WriteSinkDpcd4F1Payload1Retry` loads `display_map_entry +0x18`
  - it calls that interface's vfunc `+8` with:
    - address `0x4F1`
    - payload byte `1`
    - length `1`
  - it retries once after `10 ms`

- the stream-enable path inherits that same interface instead of constructing a separate transport:
  - `BuildStreamForDisplayEntryPath` creates stream init-data
  - it resolves the same 64-byte display-map entry
  - init-data `+0x18` is assigned from `display_map_entry +0x18`
  - `ConstructDpStreamObjectWithDpcdInterface` copies init-data `+0x18` into the stream object at parent `+0xB8`
  - the stream vtable method runs on the subobject at parent `+0x28`
  - therefore `StreamEnableWrite4F1AndPollSinkStatus` sees parent `+0xB8` as method argument `+0x90`
  - its `0x4F1` write and optional `0x205` poll go through that inherited per-display DPCD/AUX interface

- functions renamed/commented in IDA during this pass:
  - `FindDisplayEntryByObjectId` = `0x1414AEA40`
  - `BuildStreamForDisplayEntryPath` = `0x1414B55D0`
  - `ConstructStreamObjectBySignalKind` = `0x141448C10`
  - `ConstructDpStreamObjectWithDpcdInterface` = `0x1414E8140`
  - `ConstructBaseStreamObjectFromInit` = `0x1414D3E90`
  - `AttachStreamInterfaceToDisplayEntry` = `0x1414AC4A0`
  - `GetAttachedStreamInterfaceForSignal` = `0x1414AD430`
  - `ConstructDisplayObjectAndStreams` = `0x1414B7220`
  - `CreateDisplayCapabilityServiceInterface` = `0x141552020`
  - `ConstructDisplayCapabilityService` = `0x1415DC180`
  - `GetDisplayMapEntryByIndex` = `0x1413E6E10`
  - `InsertDisplayMapEntry64` = `0x1413E6F20`
  - `InsertInterfaceIntoDisplayMap` = `0x1414AC950`
  - `MarkDisplayMapEntryLive` = `0x1414B69C0`
  - `CreateDisplayTransportInterface` = `0x141551C20`
  - `CreateFullDisplayObjectInterface` = `0x14154BA90`
  - `CreateMinimalDisplayObjectInterface` = `0x141546D10`

- implication update:
  - the stream-enable writer is not a second kind of Apple firmware call
  - it is the same DPCD/AUX transport object propagated from the display/link entry into the stream object
  - for Linux, a fake or emulated DRM connector alone is not enough to reproduce the Windows path
  - the secondary/tiled path needs a real live AUX/link object equivalent, because both the source-DPCD setup and the `0x4F1` transition are tied to that per-display transport identity

### 2026-04-29 display-map `+0x18` live AUX backend follow-up

- important correction/refinement:
  - `display_map_entry +0x18` is not simply the raw interface pointer copied into the map by `InsertInterfaceIntoDisplayMap`
  - insertion stores the base display-map record and descriptor, but `+0x18` is made live later
  - this matters because the `0x4F1` writers dereference `entry +0x18` as an object with DPCD/AUX read/write vfuncs

- two writers to the live `entry +0x18` slot are now identified:
  - `UpdateDisplayMapEntryLiveSinkObject` = `0x1414B3A20`
    - resolves the 104-byte per-object record by object id
    - checks that the incoming display/sink object belongs to that record's object list
    - calls sink/display vfunc `+0x258`
    - if true, stores the active sink/display object at record `+0x18`
    - if false, clears record `+0x18`
  - `PopulateDisplayMapAuxObjectsForHotplugRange` = `0x1414B8600`
    - enumerates object ids from the controller
    - inserts display-map interfaces with `InsertInterfaceIntoDisplayMap`
    - creates `CreateDpcdAuxInterfaceForObjectId(...)`
    - stores the returned AUX object at `display_map_entry +0x18`
    - optionally creates a companion object at `entry +0x20`

- the object behind `entry +0x18` is now concretely an AUX/DPCD object in the populated path:
  - `CreateDpcdAuxInterfaceForObjectId` = `0x141551400`
  - `ConstructDpcdAuxInterfaceForObjectId` = `0x141551090`
  - constructor stores the controller/source object at full object `+0x898`
  - constructor obtains the per-object AUX handle by calling controller vfunc `+0x170` with the object id and stores it at full object `+0x30`
  - it also caches a source/controller query at full object `+0x890`

- the vtable used by the `0x4F1` writers is now decoded:
  - `entry +0x18` vfunc `+0` = `DpcdAuxReadUpTo16Bytes` at `0x14154F320`
  - `entry +0x18` vfunc `+8` = `DpcdAuxWriteUpTo16Bytes` at `0x14154F020`
  - the stream-enable `DPCD 0x205` poll reaches vfunc `+0`
  - the detect-time and stream-enable `DPCD 0x4F1 = 1` writes reach vfunc `+8`

- `DpcdAuxWriteUpTo16Bytes` proves this is normal DP AUX/DPCD transport, not a firmware mailbox:
  - rejects writes longer than `16` bytes with log text `"Attempting to write more than 16 bytes to aux.\n"`
  - builds a DPCD AUX write request with `BuildDpcdAuxWriteRequest`
  - submits it with `SubmitDpcdAuxRequest`
  - maps the AUX transaction status with `MapAuxTransactionStatusToDriverStatus`
  - `DpcdAuxReadUpTo16Bytes` is the symmetric read path using `BuildDpcdAuxReadRequest`

- there is also a special sink-OUI retry branch:
  - `ReadDpcdCapsIntoAuxInterface` reads DPCD capability blocks through the same vfuncs
  - it reads DPCD `0x500..0x508`
  - it stores the 24-bit sink OUI from `0x500..0x502`
  - if that cached OUI is `0x90CC24`, reads/writes use:
    - `DpcdAuxWriteRetryNakSpecialOui`
    - `DpcdAuxReadRetryNakSpecialOui`
  - those special paths retry AUX `NAK` up to `7` times and log:
    - `"write dpcd %5xh NAK - retry, treat as defer\n"`
    - `"read dpcd %5xh NAK - retry, treat as defer\n"`

- functions renamed/commented in IDA during this pass:
  - `UpdateDisplayMapEntryLiveSinkObject` = `0x1414B3A20`
  - `PopulateDisplayMapAuxObjectsForHotplugRange` = `0x1414B8600`
  - `CreateDpcdAuxInterfaceForObjectId` = `0x141551400`
  - `ConstructDpcdAuxInterfaceForObjectId` = `0x141551090`
  - `DpcdAuxWriteUpTo16Bytes` = `0x14154F020`
  - `DpcdAuxReadUpTo16Bytes` = `0x14154F320`
  - `DpcdAuxWriteRetryNakSpecialOui` = `0x14154EEB0`
  - `DpcdAuxReadRetryNakSpecialOui` = `0x14154F1B0`
  - `MapAuxTransactionStatusToDriverStatus` = `0x14154F4B0`
  - `ReadDpcdCapsIntoAuxInterface` = `0x14154DF40`
  - `BuildDpcdAuxWriteRequest` = `0x14146FFB0`
  - `BuildDpcdAuxReadRequest` = `0x141470030`
  - `SubmitDpcdAuxRequest` = `0x1414707B0`
  - `GetDpcdAuxRequestStatus` = `0x141470280`

- Linux-facing implication update:
  - we no longer need to treat the Windows `0x4F1` sender as opaque
  - the `0x4F1` receiver is the panel/TCON side reached through the live per-object AUX handle stored at display-map `+0x18`
  - Windows does not appear to invoke Apple firmware to send `0x4F1`; it builds and submits an AUX DPCD write transaction
  - the gating problem for plain Linux is therefore earlier:
    - create/retain the same secondary/internal object id path
    - obtain a real AUX handle for that object
    - publish source-DPCD data on that AUX path
    - then send `DPCD 0x4F1 = 1` through that exact live AUX object

### 2026-04-29 `entry +0x18` population call path

- the live AUX/backend slot is populated from two lifecycle phases:
  - topology-manager construction
  - later connection-state updates

- topology-manager construction path:
  - `ConstructDisplayTopologyManager` = `0x14143A840`
  - builds a `ConstructDisplayObjectRecordTable` object at manager `+0x98`
  - `ConstructDisplayObjectRecordTable` = `0x1414B46D0`
  - this table allocates `104` bytes per controller object id
  - each record stores:
    - object-id descriptor at record `+0x00`
    - live object/AUX slot at record `+0x18`, initially zero
    - companion slot at record `+0x20`, initially zero
    - object-list count at record `+0x30`
    - per-object lists/metadata after that
  - the constructor also reads controller/DPCD-like addresses `0x4E1`, `0x501`, and `0x521` into fields around manager/table `+0x78..+0x80`

- hotplug-range population path:
  - `ConstructDisplayTopologyManager` calls `PopulateDisplayMapAuxObjectsForHotplugRange`
  - this is the path that creates `CreateDpcdAuxInterfaceForObjectId(...)`
  - it writes the returned AUX object to `display_map_entry +0x18`
  - if feature/quirk `776` is not set, it also builds the companion `entry +0x20` object with `sub_14154C410`

- connection-state update path:
  - `UpdateDisplaySinkConnectionAndLiveAuxState` = `0x1414A7D90`
  - it calls sink/display object vfunc `+0x418` with the new connection flag
  - updates a display-index state table
  - calls `UpdateDisplayMapEntryLiveSinkObject`
  - if the sink/display object is usable, the record's `+0x18` slot is set live
  - if not usable, the slot is cleared

- interpretation:
  - Windows does not just keep a passive descriptor and then spray DPCD writes at a global AUX bus
  - there is a construction phase that enumerates object ids and creates per-object AUX backends
  - there is also a connection-state phase that keeps each object's live backend/sink pointer current
  - this is probably the closest Windows analogue to the Linux question "who makes the secondary Apple/tiled AUX path exist?"
  - if plain Linux falls back to 4K, the failure can happen before `0x4F1`:
    - no secondary object id is enumerated
    - no per-object AUX backend is created
    - the backend is created but never marked live/usable
    - source-DPCD and `0x4F1` are sent only on the primary object

### 2026-04-29 lower AUX backend under `AdapterService`

- continued descending from `ConstructDpcdAuxInterfaceForObjectId`:
  - the constructor stores the incoming service/controller object at full AUX object `+0x898`
  - the call at `0x1415511A1` invokes that object's vfunc `+0x170`
  - the concrete public vtable is `AdapterService` public vtable `off_141F9CB00`
  - vfunc `+0x170` is now named `AdapterServiceGetAuxHandleForObjectId` = `0x141465AA0`

- this means the "lower layer" behind Windows `0x4F1` is not Apple EFI/runtime firmware:
  - it is AMD DAL `AdapterService`
  - `AdapterService` owns object-id enumeration, object descriptors, connector/pin capability records, and per-object AUX handle creation
  - the final DPCD write still goes through normal AUX request build/submit machinery

- `AdapterServiceGetAuxHandleForObjectId` flow:
  - calls `AdapterServiceQueryObjectDescriptor` = `0x1414663A0` via public vtable `+0x100`
  - that delegates into the adapter object/descriptor table at the service's internal object pointer
  - extracts descriptor fields including an object/source id and source bit position
  - builds a source mask as `1 << source_bit`
  - calls the AUX transaction factory at service base `+0x1A8`
  - returns the resulting per-object AUX handle

- adjacent `AdapterService` vtable shape:
  - public vtable `+0x170` is the object-id AUX handle path used by `ConstructDpcdAuxInterfaceForObjectId`
  - public vtable `+0x178` and `+0x180` are sibling handle builders using adjacent descriptor queries
  - public vtable `+0x190` is now named `AdapterServiceReleaseAuxHandle` = `0x1414657D0`
  - `ConstructDisplayTopologyManager` also temporarily calls vtable `+0x170` while logging `"Display Index: %d, Ddc: %d"`, then releases the handle through vtable `+0x190`
  - so Windows uses the same object-id to AUX-handle machinery both for real DPCD access and for topology/DDC bookkeeping

- the AUX transaction factory path:
  - `CreateAuxTransactionPathFactory` = `0x14152BF40`
  - returns public interface object `+0x28`
  - factory vfunc `+0x08` is `AuxFactoryCreateHandleForSourceMask` = `0x14152AE80`
  - factory vfunc `+0x10` is `AuxFactoryReleaseHandle` = `0x14152AE00`
  - this creates `ConstructDualPinAuxTransactionHandle` = `0x1415B0C20`

- the final handle is a dual-pin AUX transaction handle:
  - `ConstructDualPinAuxTransactionHandle` stores the owning adapter/factory object
  - calls `ResolveSourceMaskToPinIndex` = `0x141529FB0`
  - asks the adapter pin graph for a pin type `3` endpoint
  - asks the same pin graph for a pin type `4` endpoint
  - if either endpoint is missing, the handle self-invalidates
  - `SubmitDpcdAuxRequest` calls this handle's vfunc `+0x08`
  - vfunc `+0x08` is now named `DualPinAuxHandleSubmitTransaction` = `0x1415B0980`
  - that submit path drives/flushes both underlying pin endpoints

- `AdapterService` initialization relevant to this path:
  - `CreateAdapterServiceInterface` = `0x141468DF0`
  - `ConstructAdapterServiceObject` = `0x141468B10`
  - `InitializeAdapterServiceForConnector` = `0x141467800`
  - constructor returns public interface as object `+0x28`
  - public vtable is `off_141F9CB00`
  - initializes a `CBasePin`-like capability object at base `+0x170`
  - initializes a connector/chip-family-specific object at base `+0x180`
  - initializes the AUX transaction factory at base `+0x1A8` when the capability pin allows this AUX path

- object descriptor source is now traced further down:
  - `AdapterServiceQueryObjectDescriptor` does not invent the object fields itself
  - it delegates into an object-info descriptor adapter at service base `+0x168`
  - that adapter is built by `ConstructObjectInfoDescriptorAdapter` = `0x14152D250`
  - the wrapped backend is created by `CreateVbiosObjectInfoParserForAdapter` = `0x1415297E0`
  - this creates one of two VBIOS object-info parsers:
    - `ConstructVbiosObjectInfoParserV1` = `0x1415A8E40`
    - `ConstructVbiosObjectInfoParserV2` = `0x1415AEF60`

- the VBIOS object-info parsers are explicit about their source:
  - read `"romHeaderOffset"`
  - read `"romHeader"`
  - read `"masterDataTable"`
  - read `"objectInfoTableOffset"`
  - read `"objectInfoTable"`
  - failures log those exact strings
  - this makes the object-id/source-mask/AUX selection path VBIOS-derived, not Apple-runtime-derived

- additional descriptor adapter queries:
  - `ObjectInfoDescriptorAdapterQueryA` = `0x14152D030`
  - `ObjectInfoDescriptorAdapterQueryB` = `0x14152CFB0`
  - `ObjectInfoDescriptorAdapterQueryC` = `0x14152CF50`
  - correction: the previously named `ObjectInfoDescriptorAdapterQueryApplePanelByte` is not Apple-panel policy
  - it is now renamed `DescriptorAdapterGetDalMinBacklightLevel` = `0x14152D0B0`
  - it walks DAL option table `off_141F92210`
  - it filters table entry tag/value `0x2A1` / decimal `673`
  - Python table decode shows entry `77` is `DalMinBacklightLevel`
  - when the filter matches and the capability object reports a nonzero byte, it returns that byte

- IDA names/comments added in this pass:
  - `AdapterServiceGetAuxHandleForObjectId` = `0x141465AA0`
  - `AdapterServiceQueryObjectDescriptor` = `0x1414663A0`
  - `CreateAuxTransactionPathFactory` = `0x14152BF40`
  - `AuxFactoryCreateHandleForSourceMask` = `0x14152AE80`
  - `ConstructDualPinAuxTransactionHandle` = `0x1415B0C20`
  - `DualPinAuxHandleSubmitTransaction` = `0x1415B0980`
  - `DualPinAuxHandleSetTransactionFlags` = `0x1415B0940`
  - `DualPinAuxHandleConfigureTransaction` = `0x1415B09E0`
  - `ResolveSourceMaskToPinIndex` = `0x141529FB0`
  - `AdapterServiceReleaseAuxHandle` = `0x1414657D0`
  - `AuxFactoryReleaseHandle` = `0x14152AE00`
  - `CreateVbiosObjectInfoParserForAdapter` = `0x1415297E0`
  - `ConstructVbiosObjectInfoParserV1` = `0x1415A8E40`
  - `ConstructVbiosObjectInfoParserV2` = `0x1415AEF60`
  - `ConstructObjectInfoDescriptorAdapter` = `0x14152D250`
  - `ObjectInfoDescriptorAdapterQueryA` = `0x14152D030`
  - `ObjectInfoDescriptorAdapterQueryB` = `0x14152CFB0`
  - `ObjectInfoDescriptorAdapterQueryC` = `0x14152CF50`
  - `DescriptorAdapterGetDalMinBacklightLevel` = `0x14152D0B0`
  - `ConstructTopologyDisplayPathBuilder` = `0x1414B90D0`
  - `PopulateTopologyDisplayMapInterfaces` = `0x1414B88A0`

- Linux-facing implication:
  - Windows does not send `0x4F1` through a global/default AUX bus
  - it first maps a display object id to a VBIOS-derived adapter descriptor
  - that descriptor selects a source mask/pin index
  - only then does Windows create the live per-object AUX transaction handle
  - a blind Linux write of `DPCD 0x4F1 = 1` on the primary AUX path can therefore be correct syntax but wrong recipient
  - a faithful Linux path needs the equivalent of:
    - enumerate/recognize the secondary internal display object
    - map it to the correct source mask/pin/link
    - create or expose the matching AUX path
    - publish source-DPCD data on that path
    - send `0x4F1 = 1` through that path

### 2026-04-29 correction: DAL option table vs VBIOS object descriptor

- `off_141F92210` was decoded from raw `amdkmdag.sys` with Python:
  - VA `0x141F92210`
  - file offset `0x1F47010`
  - section `PAGED2PR`
  - count `0x121` / decimal `289`
  - entry size `0x18`
  - entry layout observed as:
    - `+0x00`: option name pointer
    - `+0x08`: low dword option id/tag, high dword default/value
    - `+0x10`: type (`0` bool, `1` dword, `2` byte-like)

- exact `0x2A1` table decode:
  - index `77`
  - name pointer `0x141F9A9C8`
  - name string `DalMinBacklightLevel`
  - raw qword1 `0x0000000C000002A1`
  - option id/tag `0x2A1`
  - default/value high dword `0xC`
  - type `1`

- this corrects the previous suspicion:
  - tag `673` / `0x2A1` is not an Apple 5K object descriptor
  - it is `DalMinBacklightLevel`
  - functions touching `off_141F92210` mostly implement option-table loading and option lookup
  - renamed in IDA:
    - `GetDalOptionTableCount` = `0x141468D30`
    - `FindDalOptionTableIndexById` = `0x141468D40`
    - `LoadDalOptionTableValues` = `0x1414643F0`
    - `AdapterServiceGetDalOptionValue` = `0x141467360`
    - `AdapterServiceIsDalBoolOptionEnabled` = `0x141467530`
    - `DescriptorAdapterGetDalMinBacklightLevel` = `0x14152D0B0`

- the real object-id to AUX selector path is separate:
  - `AdapterServiceGetAuxHandleForObjectId` calls AdapterService vfunc `+0x100`
  - that reaches `ObjectInfoDescriptorAdapterQueryC`
  - `ObjectInfoDescriptorAdapterQueryC` delegates to VBIOS parser public vfunc `+0x60`
  - VBIOS parser vfunc `+0x60` is:
    - `VbiosParserV1GetObjectDescriptor` = `0x1415A7E60`
    - `VbiosParserV2GetObjectDescriptor` = `0x1415AE3A0`

- VBIOS descriptor parser behavior:
  - both variants locate a VBIOS object record for the requested object id
  - both look for encoder record type `1`
  - both fill a descriptor struct consumed by `AdapterServiceGetAuxHandleForObjectId`
  - V1 helper `ParseVbiosV1EncoderRecordToObjectDescriptor` = `0x1415A7830`
  - V2 helper `ParseVbiosV2EncoderRecordToObjectDescriptor` = `0x1415AC210`

- descriptor fields relevant to AUX creation:
  - descriptor `+0x00`: one-bit flag from encoder record byte 2 bit 7
  - descriptor `+0x04`: low nibble of encoder record byte 2
  - descriptor `+0x08`: bits 4..6 of encoder record byte 2
  - descriptor `+0x0C`: encoder record byte 3
  - descriptor `+0x1C`: source/encoder selector passed directly to the AUX factory
  - descriptor `+0x3C`: source-bit/pin-bit field
  - `AdapterServiceGetAuxHandleForObjectId` computes `source_mask = 1 << descriptor[+0x3C]`
  - it calls the AUX factory with descriptor `+0x1C` and that source mask

- Linux-facing implication update:
  - the next static target is not DAL option id `0x2A1`
  - the next static target is the VBIOS object record for the secondary/internal 5K path
  - if Linux sees only the primary 4K path on plain boot, it may be failing to enumerate or bind the VBIOS object whose descriptor has the second source-bit at `+0x3C`
  - a useful Linux-side probe would dump AMD VBIOS object table records and compare them with the object ids seen during probe/USB boot

### 2026-04-29 exact iMac19,1 VBIOS object table decode

- Python scan of the extracted SPI tree found the exact ATOMBIOS used by Linux:
  - Linux dmesg reports `ATOM BIOS: 113-D0008A14GP-003`
  - extracted ROM path:
    - `RE_Files/imac19_1_spi_1.bin.dump/3 BIOS region/3 B116D52B-E69F-4352-B657-790ECD7F28EB/2 2AD3DE79-63E9-9B4F-B64F-E7C6181B0CEC/1 EE4E5898-3914-4259-9D6E-DC7BD79403CF/1 Volume image section/0 A881D567-6CB0-4EEE-8435-2E72D33E45B5/174 64956079-E961-4FCD-B48A-F307F6C08DDE/0 Raw section/body.bin`
  - PCI ROM signature `55 aa`
  - PCI vendor/device `1002:67df`
  - ATOMBIOS string at `0x1A7`
  - ROM header at `0x234`
  - master data table at `0x51C2`
  - object info table at `0x588A`
  - object info table revision `1.3`
  - helper script saved as `scripts/decode_imac5k_vbios_objects.py`

- This means the relevant Windows grammar for this machine is the V1 parser path:
  - `VbiosParserV1GetObjectDescriptor` = `0x1415A7E60`
  - `VbiosV1FindObjectRecordByObjectId` = `0x1415A3C00`
  - `ParseVbiosV1EncoderRecordToObjectDescriptor` = `0x1415A7830`

- Object-id helper functions renamed/commented in IDA:
  - `ObjectIdGetClassNibble` = `0x14143BDB0`
  - `ObjectIdCoreFieldsEqual` = `0x14143BCB0`
  - `ObjectIdCopyCoreFields` = `0x14143BEE0`
  - `ObjectIdBuildCoreFields` = `0x14143BE40`
  - `ObjectIdClearCoreFields` = `0x14143C040`
  - `DisplayObjectIdGetLowByteIfClass3` = `0x14143BF80`
  - `VbiosV1FindObjectRecordByObjectId` = `0x1415A3C00`
  - `VbiosV2FindObjectRecordByObjectId` = `0x1415AC490`
  - `VbiosV1PtrAtOffsetChecked` = `0x1415A3DE0`
  - `VbiosV2PtrAtOffsetChecked` = `0x1415AC440`
  - `VbiosV1DecodeObjectIdFromRecordRef` = `0x1415A4590`
  - `VbiosV2DecodeObjectIdFromRecordRef` = `0x1415A9DB0`

- Exact object-id packing:
  - low byte: object type/id
  - bits `8..11`: enum/index
  - bits `12..15`: graph object class
  - class `3` is connector/sink
  - low byte `0x14` is eDP
  - low byte `0x13` is DisplayPort

- Exact display paths in `113-D0008A14GP-003`:
  - path tag `0x0002`: connector `0x3114` = connector class, enum 1, type `0x14` eDP; encoder chain `0x2120`
  - path tag `0x0008`: connector `0x3113` = connector class, enum 1, type `0x13` DisplayPort; encoder chain `0x2220`
  - path tag `0x0080`: connector `0x3213` = connector class, enum 2, type `0x13` DisplayPort; encoder chain `0x221E`
  - path tag `0x0200`: connector `0x3313` = connector class, enum 3, type `0x13` DisplayPort; encoder chain `0x2221`

- Exact connector AUX/DDC/HPD records:
  - `0x3114` eDP:
    - I2C/AUX record `01 04 93 00`
    - HPD record `02 04 02 00`
    - descriptor index from record byte 2 = `3`
    - descriptor `+0x1C` selector = `0x4875`
    - descriptor `+0x3C` mask shift = `0`
    - computed source mask = `0x1`
  - `0x3113` DP enum 1:
    - I2C/AUX record `01 04 92 00`
    - HPD record `02 04 01 00`
    - descriptor index = `2`
    - descriptor `+0x1C` selector = `0x4871`
    - descriptor `+0x3C` mask shift = `0`
    - computed source mask = `0x1`
  - `0x3213` DP enum 2:
    - I2C/AUX record `01 04 90 00`
    - HPD record `02 04 03 00`
    - descriptor index = `0`
    - descriptor `+0x1C` selector = `0x4869`
    - descriptor `+0x3C` mask shift = `0`
    - computed source mask = `0x1`
  - `0x3313` DP enum 3:
    - I2C/AUX record `01 04 91 00`
    - HPD record `02 04 04 00`
    - descriptor index = `1`
    - descriptor `+0x1C` selector = `0x486D`
    - descriptor `+0x3C` mask shift = `0`
    - computed source mask = `0x1`

- Important correction to the earlier source-mask hypothesis:
  - on this exact ROM, the four connector paths do not appear to be separated primarily by descriptor `+0x3C`
  - all four decoded V1 connector descriptors produce mask shift `0` / source mask `0x1`
  - the path distinction is carried by descriptor `+0x1C`:
    - DP enum 2: `0x4869`
    - DP enum 3: `0x486D`
    - DP enum 1: `0x4871`
    - eDP enum 1: `0x4875`
  - `AdapterServiceGetAuxHandleForObjectId` passes descriptor `+0x1C` and source mask into `AuxFactoryCreateHandleForSourceMask`
  - `ConstructDualPinAuxTransactionHandle` then resolves that pair into a pin index and opens pin type `3` plus pin type `4` endpoints

- Linux-facing implication update:
  - the exact ROM already exposes one eDP object and three DP objects
  - the probe/diagnostics boot state where Linux sees `eDP-1` plus `DP-1` is consistent with one of the DP objects becoming live alongside the eDP object
  - the missing piece is no longer "is there a second object in the ROM?"
  - the missing piece is "which DP object does Apple/Windows treat as the paired 5K half, and what policy makes Linux plain boot ignore or disconnect it?"
  - the current best static candidates for the paired half are the DP objects with descriptors:
    - `0x3113` / selector `0x4871`
    - `0x3213` / selector `0x4869`
    - `0x3313` / selector `0x486D`
  - comparing Linux probe-boot `DP-1` HPD/AUX behavior against these ROM records should identify the exact object.

- Linux source cross-check:
  - `display/dc/bios/bios_parser.c:bios_parser_get_connector_id()` reads `ATOM_OBJECT_TABLE.asObjects[i].usObjectID` directly from the connector table
  - for this V1.3 ROM, connector table order is:
    - index `0`: `0x3114` eDP
    - index `1`: `0x3113` DP enum 1
    - index `2`: `0x3213` DP enum 2
    - index `3`: `0x3313` DP enum 3
  - therefore Linux `DP-1` / connector index `1` should correspond to ROM object `0x3113`
  - the probe-boot live secondary half is therefore most likely:
    - connector object `0x3113`
    - I2C/AUX record `01 04 92 00`
    - HPD record `02 04 01 00`
    - Windows descriptor selector `0x4871`

- Live Linux VBIOS cross-check:
  - user-provided live dump: `amdgpu_vbios.rom`
  - size: `60416`
  - SHA-256: `a351440349690942c5e1cb8e1346417d1e98c279c68cb9778abebaa744e99fe8`
  - SPI candidate `113-D0008A14GP-003` has the same size and SHA-256
  - byte-for-byte comparison result: identical
  - therefore plain Linux is not failing because it receives a different VBIOS image from VFCT/ACPI
  - the remaining failure class is runtime/preboot state, connector detect policy, AUX/DPCD state, or Linux handling of the existing ROM topology

### 2026-04-29 reconciliation: Linux `DP-1` signal 32 vs Windows signal 128 notes

- Older Windows notes emphasized a signal/type `128` gate because `WindowsDM_EnableSecondaryTileIfRequired` only calls `SendSecondaryTileOpcode1265Payload1` from that early helper when:
  - connected pin descriptor byte `+6` has bit `2` or bit `3`
  - and `dc->links[index]` / display entry signal field is `128`

- That is not the only `0x4F1` call path:
  - `BringUpDisplayBySignalType` dispatches signal `32` directly to `BringUpDpOrEdpWithLinkTraining`
  - signal `128` goes through `BringUpEdpSignal128Thunk`, which is only a thunk to the same `BringUpDpOrEdpWithLinkTraining`
  - inside `BringUpDpOrEdpWithLinkTraining`, signal `128` receives extra eDP-style HPD-ready callbacks before link training
  - but the common source-DPCD publication and link-training path is shared by signal `32` and signal `128`
  - after successful bring-up, `BringUpDisplayBySignalType` can still call `SendSecondaryTileOpcode1265Payload1` if the lower display object flag at `+0x5B4` / decimal `1460` is set

- This makes the Linux observation coherent:
  - Linux probe boot reports live secondary `DP-1` as signal `32`
  - that can still correspond to Windows' common DP/eDP bring-up path
  - the stricter signal `128` gate applies to the early `WindowsDM_EnableSecondaryTileIfRequired` helper, not to every possible `0x4F1` send
  - the likely paired 5K half remains `DP-1` / ROM object `0x3113` unless later evidence points to a different DP object

### 2026-04-29 extra Windows RE passes before next kernel patch

- pass 1 focused on the detect/keep-connected path around Apple capability tag `0x3B`:
  - `DetectDisplayAndApplyApple5kOverrides` = `0x1414B3E80`
  - `DetectDisplayEarlyExitForApple5kCaps` = `0x1414B31D0`
  - `ProbeDisplayStatusMaybeAssert4F1` = `0x1414B2490`
  - `DetectDisplayMaybeAssert4F1ForCapableSink` = `0x1414B2AF0`

- the tag `0x3B` DisplayPort keep-connected override is now precise:
  - Windows fetches the connected pin/display descriptor through vfunc `+0x218`
  - it tests descriptor byte `+6`, bit `3`
  - earlier mapping says this is Apple/source capability id `0x39` -> property tag `0x3B`
  - if that bit is set, and:
    - the underlying display object reports physically present through vfunc `+0x258`
    - the current detect result says not connected at result byte `+114`
    - the current signal type returned by vfunc `+0x288` is `11` / DisplayPort
  - Windows forces:
    - result byte `+114 = 1`
    - status byte `+112 = 0`
  - this is a direct Windows analogue of "keep the DP-side tile alive even though normal detection would drop it"

- detect-time `0x4F1` behavior remains more limited:
  - `DetectDisplayMaybeAssert4F1ForCapableSink` checks descriptor byte `+6`, bit `2`
  - this maps to tag `0x3A`
  - if present, it calls `WriteSinkDpcd4F1Payload1Retry`
  - `WriteSinkDpcd4F1Payload1Retry` resolves the display object by object id, takes display-map entry `+0x18`, and calls that DPCD/AUX interface vfunc `+8`
  - therefore detect-time `0x4F1` is still per-display-object, not global

- stream-enable `0x4F1` behavior is broader and important for Linux:
  - `StreamEnableWrite4F1AndPollSinkStatus` = `0x1414E7810`
  - reached by `EnableValidatedStreamMaybeAssert4F1` = `0x141446B50`
  - checks descriptor byte `+6`, bit `2` or bit `3`
  - therefore tag `0x3A` or tag `0x3B` is enough
  - writes DPCD `0x4F1`, payload `1`, length `1`
  - retries once after `10 ms` on failure
  - this path does not require signal/type `128`
  - optional DPCD `0x205` polling is still gated by descriptor byte `+6`, bit `5` / tag `0x40`
  - since the static `APP/AE25`/`APP/AE26` table has no tag `0x40`, Linux should not add the `0x205` poll without runtime evidence

- pass 2 focused on the signal-32 DP-side bring-up path:
  - `BringUpDisplayBySignalType` dispatches signal `32` directly to `BringUpDpOrEdpWithLinkTraining`
  - signal `128` reaches the same helper through `BringUpEdpSignal128Thunk`
  - both signal classes call `ProgramDpPreTrainingAuxState` before `PerformLinkTrainingWithRetries`
  - this confirms source-DPCD setup is common to the DP-side `DP-1` path and not only an eDP/signal-128 path

- new signal-32-only pretraining detail:
  - `SetDpcdMstCtrlEnableBit` = `0x141B8E9C0`
  - `ClassifyDpLinkRateEncodingRange` = `0x141B8C880`
  - if `ClassifyDpLinkRateEncodingRange(link_settings)` returns class `2` and signal is `32`, Windows:
    - reads DPCD `0x111`
    - sets bit `0`
    - writes DPCD `0x111` back
  - DPCD `0x111` is `DP_MSTM_CTRL` in normal DP naming
  - this appears to be a generic DP/MST-style setup, not Apple-only
  - it is still relevant because the live OCLP/probe secondary half is Linux `DP-1` / signal `32`

- other DPCD sideband/status operations found in the same lower-write family:
  - `BuildLinkStateAndWriteControlOpcodes` writes:
    - DPCD `0x170`, length `1`
    - conditionally DPCD `0x116`, length `1`
    - conditionally DPCD `0x375`, length `1`
  - `HandleDpcd2006StatusAfterHpdRx` later:
    - reads DPCD `0x170`, length `1`
    - if bit `0` is present, reads DPCD `0x2006`, length `3`
    - may write DPCD `0x2006`, length `1` to acknowledge/clear status bits
  - these look like normal DP status/HPD maintenance around the link, not a separate Apple firmware channel

- new IDA names/comments applied:
  - `SetDpcdMstCtrlEnableBit` = `0x141B8E9C0`
  - `ClassifyDpLinkRateEncodingRange` = `0x141B8C880`
  - `WriteDpcd32FForDpEdpSignal` = `0x141B8C700`
  - comments added at the signal-32 branch in `BringUpDpOrEdpWithLinkTraining`
  - comments added at the tag `0x3B` detect override and stream-enable `0x4F1` writer

- kernel-facing implications from these two passes:
  - for OCLP/probe-boot finalization:
    - keep Linux `DP-1` / ROM object `0x3113` alive through detect and first modeset
    - treat it as the paired internal DisplayPort-side tile when the Apple panel/tile fingerprint matches
    - make sure stream-enable/finalization can send `DPCD 0x4F1 = 1` on `DP-1`'s AUX path, not only on `eDP-1`
    - log/read DPCD `0x111` and `0x205` on both halves before adding any writes/polls
  - for Apple-free plain boot:
    - the missing piece is likely a Linux equivalent of the tag `0x3B` DisplayPort keep-connected policy plus the source-DPCD/`0x4F1` sequence on the `0x3113` path
    - these passes still found no unique Apple EFI/runtime call and no obvious Apple-specific MMIO path around `0x4F1`
    - the observed Windows actions remain ordinary per-link DPCD/AUX operations layered on top of a correct connector/sink detection decision

### 2026-04-29 final Windows RE tightening before next Linux patch

- IDA correction:
  - the actual start of `StreamEnableWrite4F1AndPollSinkStatus` is `0x1414E7810`
  - earlier notes that used `0x1414E7820` were pointing inside the same function, not a separate helper

- the main link-enable convergence point has been renamed in IDA:
  - `EnableLinkAndBringUpDisplayPath` = `0x141B67540`
  - callers include:
    - HPD/RX IRQ handling
    - link enable paths
    - validation/re-enable paths
  - this function eventually calls `BringUpDisplayBySignalType`
  - therefore the observed Windows sequence is:
    - detect keeps/creates a live display object
    - link-enable convergence calls signal-specific bring-up
    - DP/eDP bring-up publishes source DPCD
    - link training runs
    - post-success `0x4F1` may be reasserted if the lower display object carries the tag/value-derived flag

- the signal-32 DP-side sequence is now tighter:
  - `BringUpDpOrEdpWithLinkTraining` calls `ClassifyDpLinkRateEncodingRange`
  - classifier behavior:
    - link-rate field `6..30` -> class `1`
    - link-rate field `1000..2000` -> class `2`
    - otherwise `0`
  - for class `2` and signal `32`, Windows calls `SetDpcdMstCtrlEnableBit`
  - `SetDpcdMstCtrlEnableBit`:
    - reads DPCD `0x111`
    - sets bit `0`
    - writes DPCD `0x111`
  - this happens before `ProgramDpPreTrainingAuxState`
  - `ProgramDpPreTrainingAuxState` then writes:
    - `0x300..0x302` = source OUI `00 00 1A` unless already present
    - `0x303..0x30B` = AMD source device-specific block
    - `0x310..0x312` = AMD source table revision payload `04/05 1D 03`
    - optional `0x340` byte when the high-generation condition and source field are present
  - link training starts after those writes

- the Windows mode-model side maps directly onto our GNOME/KDE symptom:
  - `ModeManagerAddDerivedModesIncluding5kTile` = `0x141A3C4E0`
  - it can synthesize `2560x2880` from a `1920x2160` source timing only when:
    - mode-manager flag bit `7` is set
    - the source timing record has bit `7` set
    - the base timing is `1920x2160`
    - mode-manager flag bit `5` is set
    - `DalEnable5kTiledMode` is true
  - it first adds `1280x1440`
  - then, under the extra bit5 + `DalEnable5kTiledMode` gate, adds `2560x2880`

- the timing classifier side is separate from the half-tile injection:
  - `MarkTimingPipeSplitForHighResAnd5k60` = `0x1415D5380`
  - it sets timing record flag bit `7` when the timing aspect ratio matches the display dimensions within the `90%..110%` window
  - it marks high-res / `5120x2880@60`-class timings as pipe-split by setting timing record low pipe bits to `2`, gated by multi-pipe options
  - this means Windows does not rely only on a raw `2560x2880` tile mode
  - it has both:
    - a physical half-tile derived mode path
    - a logical `5120x2880@60` pipe-split classification path

- implication for the current Linux problem:
  - the OCLP/probe path proving two live `2560x2880` scanouts is only half of the Windows model
  - Linux also needs to expose the pair as one coherent tiled/internal display model, or at least as correctly coordinated tile connectors
  - this matches the current GNOME/KDE failures:
    - KDE sees two outputs
    - GNOME sees one 5K-ish logical monitor but scans/stretchs incorrectly
  - the next Linux patch should therefore avoid being only an AUX write experiment
  - it should also log and fix the tile identity/order/properties and the logical `5120x2880` aggregation presented to DRM userspace

- IDA comments added at:
  - `0x141B67540`
  - `0x141A3C4E0`
  - `0x1415D5380`
  - `0x1414E7810`

### 2026-04-29 Windows mode-model chain and GNOME/KMS correlation

- additional IDA names/comments applied:
  - `UpdateDisplayViewSolutionsWithDerived5kModes` = `0x141A3CBA0`
  - `GetOrCreateModeSolutionForDisplay` = `0x141A3CAF0`
  - comments added at:
    - `ConstructModeSolutionWithPipeSplitFlags` = `0x141ACF560`
    - `ApplyPinCapabilityEdidFixups` = `0x141A456E0`
    - `DoubleEdidPhysicalWidthForTiledCaps` = `0x141A43030`

- the mode-model producer/consumer chain is now explicit:
  - `GetOrCreateModeSolutionForDisplay` looks up an existing per-display mode solution
  - if absent, it calls `CreateModeSolutionWith5KPipeSplitOptionFlags`
  - `CreateModeSolutionWith5KPipeSplitOptionFlags` passes option flags into `ConstructModeSolutionWithPipeSplitFlags`
  - `ConstructModeSolutionWithPipeSplitFlags` stores those flags at mode-solution object `+0xB8`
  - bit `1` in those constructor flags comes from ModeMgr flag bit `9`
  - ModeMgr flag bit `9` is the `Dal5K60PipeSplit` path
  - `UpdateDisplayViewSolutionsWithDerived5kModes` then validates the display timing list and calls `ModeManagerAddDerivedModesIncluding5kTile`
  - so the derived `2560x2880` half-tile timing is not just a UI decoration; it is inserted into the real per-display view/mode-solution list

- the Windows EDID/capability path is also tied to the same tiled model:
  - `ApplyPinCapabilityEdidFixups` dispatches capability/property tags
  - tags `0x3A` and `0x3B` call `DoubleEdidPhysicalWidthForTiledCaps`
  - `DoubleEdidPhysicalWidthForTiledCaps` doubles EDID byte `21` / horizontal physical size if it is smaller than byte `22` / vertical physical size and smaller than `0x80`
  - it then recomputes the EDID checksum
  - therefore tags `0x3A/0x3B` simultaneously explain:
    - the userspace-visible tiled/half-panel EDID geometry fixup
    - the sink-private `DPCD 0x4F1 = 1` permission path
    - the tag `0x3B` DisplayPort keep-connected detect override

- local Linux capture correlation:
  - `kernel_obs_gnome/imac5k_drm_info.txt` and `kernel_obs_4_gnome/imac5k_drm_info.txt` show superficially correct DRM `TILE` blobs:
    - group id `1`
    - single monitor `yes`
    - number of tiles `2x1`
    - tile locations `0,0` and `1,0`
    - tile size `2560x2880`
  - however, the GNOME KMS state in `kernel_obs_gnome/imac5k_state.txt` shows two independent `gnome-shell` framebuffers:
    - one `2560x2880` FB on `crtc-0`
    - one `2560x2880` FB on `crtc-1`
    - both with source origin `0,0`
  - by contrast, the better Xorg-style state in `kernel_obs_3/imac5k_state.txt` shows one shared `5120x2880` FB:
    - `crtc-0` scans source `0,0`
    - `crtc-1` scans source `2560,0`
  - this is the exact userspace/KMS shape Windows' mode model implies:
    - physical modes remain `2560x2880` per tile
    - the logical desktop/FB must be `5120x2880`
    - the right tile must scan from source x-offset `2560`

- current GNOME/Mutter context:
  - a late-2025 Mutter fix exists for tiled-monitor handling:
    - Launchpad bug `2131575`, "SRU backends/monitor-manager: Improve tiled monitor handling"
    - upstream MR referenced there: GNOME Mutter `!4685`
  - the bug text explicitly covers tiled monitors such as Apple Studio Display, Dell UP2715K/UP3218K, and LG 27MD5KL
  - the failure descriptions include stretched content, black/unusable output, and main/non-main tile handling
  - this is very close to our observed GNOME symptom class
  - therefore the next confidence step should include checking the exact CachyOS `mutter` version and whether it contains that tiled-monitor fix, before assuming every remaining GNOME failure is in `amdgpu`

- high-confidence kernel target after this pass:
  - for the OCLP/probe path, the kernel side must first provide a stable pair of connected tile heads with:
    - identical tile group
    - single-monitor flag set
    - correct `2x1` tile geometry
    - physical tile modes present on both connectors
    - DP-1 / ROM object `0x3113` preserved through userspace takeover
    - `DPCD 0x4F1 = 1` sent on the same DP-1 AUX path after the Windows-like source-DPCD/link-training stage
  - but if those properties are present and GNOME still allocates two independent `2560x2880` framebuffers, that specific failure is likely in Mutter's tiled monitor handling or in how our connector pair differs from the assumptions Mutter makes
  - a kernel patch can still work around that only if we identify the missing/misleading kernel-exposed field that causes Mutter to choose the wrong logical layout
  - the static Windows evidence now argues against a blind kernel-only "just add 0x4F1" patch for the GNOME issue

### 2026-04-29 final Windows RE pass for Linux "keep 5K"

This pass was intentionally narrow: only re-check the Windows paths that affect an already-probed / already-preinitialized 5K tiled state, i.e. the state Linux sees after the diagnostics/OCLP probe boot.

- the Windows stream-enable writer remains the strongest runtime analogue for Linux's "keep 5K through KMS takeover" problem:
  - `EnableValidatedStreamMaybeAssert4F1` = `0x141446B50`
  - `StreamEnableWrite4F1AndPollSinkStatus` = `0x1414E7810`
  - if the stream is actually enabled immediately, Windows descends into `StreamEnableWrite4F1AndPollSinkStatus`
  - that function checks the live sink/pin descriptor through the stream/display object
  - descriptor byte `+6`, bit `2` or bit `3` gates the sink-private write:
    - bit `2` corresponds to the capability/tag family previously associated with `0x3A`
    - bit `3` corresponds to the capability/tag family previously associated with `0x3B`
  - when either bit is present, Windows writes:
    - DPCD address `0x4F1`
    - payload `1`
    - length `1`
    - retry once after `10 ms` on failure
  - if descriptor byte `+6`, bit `5` is present, Windows then polls:
    - DPCD address `0x205`
    - length `1`
    - up to `50` iterations
    - waits for byte value `1`

- the detect side has one extra nuance that is important for Linux:
  - `DetectDisplayAndApplyApple5kOverrides` = `0x1414B3E80`
  - `DetectDisplayEarlyExitForApple5kCaps` = `0x1414B31D0`
  - `DetectDisplayEarlyExitForApple5kCaps` reads the same descriptor-style byte `+6`
  - if bit `2` or bit `3` is present, the helper returns early as "handled"
  - therefore the `0x3A/0x3B` Apple/tiled capability family is not merely a late permission to send `0x4F1`
  - it also participates in keeping the special display object out of the ordinary disconnect/fallback detect path
  - this is now the closest Windows-side static analogue of our Linux "preserve the secondary tiled head instead of letting generic detect collapse it" quirk

- the source-DPCD/link-training path is still common, not Apple-firmware-specific:
  - `ProgramDpPreTrainingAuxState` = `0x141B8E090`
  - `BringUpDpOrEdpWithLinkTraining` = `0x141B6CA70`
  - `RetrieveDpLinkCapsAfterSourceDpcdSetup` = `0x141B84040`
  - before link training, Windows writes the AMD source identity/capability block on the same live AUX object:
    - `0x300..0x302` = AMD OUI `00 00 1A`
    - `0x303..0x30B` = AMD source-specific bytes
    - `0x310..0x312` = AMD table-revision payload `04 1D 03` or `05 1D 03`
    - optional `0x340` = minimum hblank byte
  - Linux already has most of the public equivalent in `dpcd_set_source_specific_data`
  - the meaningful mismatch remains that local Linux does not appear to write `DP_SOURCE_TABLE_REVISION` / `0x310`

- the post-bring-up `0x4F1` helper is still downstream of successful link enable:
  - `BringUpDisplayBySignalType` = `0x141B6A720`
  - it calls the DP/eDP link-training path for signal `32`
  - it calls the signal/type `128` eDP thunk for the special internal path
  - only after successful bring-up does it mark the display live
  - only then does it optionally call `SendSecondaryTileOpcode1265Payload1` if the lower display object carries a nonzero flag at `+0x5B4`

- the mode-model side still says Windows intentionally exposes half-tile modes into the display view/mode-solution list:
  - `UpdateDisplayViewSolutionsWithDerived5kModes` = `0x141A3CBA0`
  - `ModeManagerAddDerivedModesIncluding5kTile` = `0x141A3C4E0`
  - the derived-mode function can synthesize a real `2560x2880` half-tile mode from the `1920x2160` base timing when the 5K/tiled gates are set
  - this maps cleanly onto the Linux observation that the working pre-GUI state is not one magic monolithic `5120x2880` link, but two live `2560x2880` scanout halves that must be exposed/consumed as one tiled monitor

Linux implementation implications after this final pass:

- for the probe/OCLP "keep 5K" case, the first kernel target should remain preservation, not cold-boot synthesis:
  - keep the secondary Apple/tiled connector/link alive through detect and the first atomic takeover
  - preserve a correct tile group on both connectors
  - do not allow a proposed userspace commit that drops the second `2560x2880` stream to silently become the new steady state

- the next experimental kernel changes should be ordered like the Windows lifecycle:
  1. preserve / reconnect the secondary tiled head if the preboot state exposed it
  2. ensure `dpcd_set_source_specific_data` runs on that secondary link's AUX path
  3. optionally add the Windows-matching `0x310` source-table-revision write on the same path
  4. only after the secondary link is live/trained, assert sink-private DPCD `0x4F1 = 1` on that same secondary AUX path

- what this pass did **not** find:
  - no Apple EFI runtime service call in the Windows keep path
  - no special Apple MMIO register sequence replacing normal AMD AUX/link training
  - no evidence that a blind `0x4F1` write to the primary/eDP AUX path is sufficient
  - no evidence that a one-connector fake `5120x2880` abstraction is the Windows model at the hardware layer; Windows still has a derived half-tile / pipe-split mode path plus per-link AUX/link control

Practical confidence update:

- the next Linux work can be done with reasonably high confidence without another broad Windows RE pass
- the highest-value Linux patch direction is now:
  - strengthen the preserved secondary-head/tile-group path
  - add logging around source-DPCD publication and secondary-link stream enable
  - add an optional, tightly gated secondary-link `0x310` + `0x4F1` sequence only after the link object is proven live
- if GNOME still stretches/loses the right tile after both KMS streams stay alive and the `TILE` properties are correct, the remaining issue is likely in compositor tiled-monitor handling or in the exact tile ordering/source-rect contract, not in a missing Windows firmware call

### 2026-04-30 final IDA sanity check before kernel patch

- IDA MCP health check confirmed the active database:
  - `C:\proj\reverse\WT6A_INF\B416406\amdkmdag.sys.i64`
  - module `amdkmdag.sys`
  - Hex-Rays ready

- a direct IDA immediate search for `0x4F1` found many raw/data hits, but the meaningful code-side hits still reduce to the same small set:
  - structure-offset uses:
    - `0x1400942B2`: write to `[rsi+0x4F1]`
    - `0x140094929`: compare `[rcx+0x4F1]`
    - `0x140099B8D`: write to `[rbx+0x4F1]`
  - non-DPCD diagnostic/log use:
    - `0x141895EBC`: `HdcpBindMftIdSession` uses `0x4F1` as a printed/source-line style constant
  - real DPCD/AUX writers:
    - `WriteSinkDpcd4F1Payload1Retry` at `0x1414B03F0`
    - `StreamEnableWrite4F1AndPollSinkStatus` at `0x1414E7810`
    - `SendSecondaryTileOpcode1265Payload1` at `0x141B716F0`

- all three real Windows runtime `0x4F1` writers write payload `1`:
  - `WriteSinkDpcd4F1Payload1Retry`:
    - resolves the display object by object id
    - uses display-map entry `+0x18`
    - calls the AUX/DPCD write vfunc with address `1265`, payload byte `1`, length `1`
    - retries once after `10 ms`
  - `StreamEnableWrite4F1AndPollSinkStatus`:
    - uses the stream/link AUX object at `a1+0x90`
    - gates on descriptor byte `+6` bit `2` or bit `3`
    - writes address `1265`, payload byte `1`, length `1`
    - retries once after `10 ms`
    - optionally polls DPCD `0x205` when descriptor byte `+6` bit `5` is set
  - `SendSecondaryTileOpcode1265Payload1`:
    - writes address `1265`, payload byte `1`, length `1`
    - retry after `10 ms`
    - optional `100 ms` settle delay if requested by caller

- this final check did **not** find a Windows runtime-driver counterpart that clearly writes `DPCD 0x4F1 = 0`
  - the known payload-`0` cleanup/deassert behavior remains a firmware/CoreEG2-side observation
  - for the first Linux `keep 5K` patch, this means we should add only a cautious, tightly gated `0x4F1 = 1` assertion on the live secondary AUX path
  - do not add a Linux runtime deassert path unless later testing shows it is needed for suspend/resume or mode teardown

- final RE conclusion before kernel work:
  - no broad Windows RE blocker remains
  - the next patch should focus on preserving the secondary tiled head and only then replaying the Windows-like source-DPCD / `0x4F1 = 1` sequence on that exact link

### 2026-04-30 Linux patch applied: secondary-link DPCD finalization

- implemented the first Windows-matching kernel experiment in:
  - `drivers/gpu/drm/amd/display/amdgpu_dm/amdgpu_dm.c`
  - `drivers/gpu/drm/amd/display/amdgpu_dm/amdgpu_dm.h`

- the patch remains narrowly gated by the existing iMac5K/probe-boot state:
  - only runs when the iMac5K quirk is enabled
  - only runs after the secondary tiled head has been discovered
  - only runs after Linux has observed two live `2560x2880` tile streams
  - only targets the connector already marked as the secondary tiled head

- the secondary-link finalization sequence now does:
  1. `dpcd_set_source_specific_data(link)` on the secondary link
  2. write `DP_SOURCE_TABLE_REVISION` / DPCD `0x310` with the Windows-style source table revision bytes
  3. only if that succeeds, write sink-private DPCD `0x4F1 = 1`
  4. retry the `0x4F1` write once after `10 ms`, matching the Windows runtime pattern

- the patch intentionally does not:
  - write `0x4F1` on the primary/eDP link
  - write `0x4F1 = 0`
  - call any Apple EFI/firmware mailbox path
  - synthesize the cold-boot secondary head yet

- expected dmesg breadcrumbs:
  - `IMAC5K: commit_streams_end secondary DPCD finalize gated off ...` if the quirk has not reached the preserved two-tile state yet
  - `IMAC5K: commit_streams_end secondary DPCD finalize found no secondary tile stream target ...` if the global gates pass but no committed stream maps to the marked secondary connector
  - `IMAC5K: commit_streams_end secondary source-DPCD ... 0x310=... status=...`
  - `IMAC5K: commit_streams_end secondary 0x4F1 latch deferred ... source-DPCD not programmed` if the `0x310` write fails
  - `IMAC5K: commit_streams_end secondary 0x4F1 latch ... payload=1 status=... asserted=...`

### 2026-04-30 post-test Windows RE pass: source-DPCD is too late in the first Linux patch

- latest GNOME/probe-boot test directory:
  - `C:\proj\reverse\WT6A_INF\kernel_obs_5_gnome`

- the test confirms the latest kernel was loaded:
  - `Linux version 7.0.1-1-imac-5k-g0478e908d18f`
  - commit `0478e908d` / `try to keep 5K mode by emulating Windows behavior`

- Linux reached a better exposed topology:
  - `eDP-1` connected, tile group `1`, tile location `0,0`, tile size `2560x2880`
  - `DP-1` connected, tile group `1`, tile location `1,0`, tile size `2560x2880`
  - two active CRTCs at `2560x2880`
  - fbcon still has a `5120x2880` framebuffer

- but the secondary physical/AUX path failed before the new finalization could help:
  - `dpcd_set_link_settings` failed on `DP_DOWNSPREAD_CTRL`
  - `dpcd_set_link_settings` failed on `DP_LANE_COUNT_SET`
  - `dpcd_set_link_settings` failed on `DP_LINK_BW_SET`
  - `enabling link 1 failed: 15`
  - the new patch then logged `secondary source-DPCD DP-1 ... 0x310=04 1d 03 status=-1`
  - it correctly deferred `0x4F1` because source-DPCD was not successfully programmed

- IDA MCP was rechecked and the active Windows database still resolves the important labels:
  - `BringUpDpOrEdpWithLinkTraining` = `0x141B6CA70`
  - `ProgramDpPreTrainingAuxState` = `0x141B8E090`
  - `PrepareLinkTrainingStateAndArmEdp128` = `0x141BB7090`
  - `RetrieveDpLinkCapsAfterSourceDpcdSetup` = `0x141B84040`
  - `DetectDisplayEarlyExitForApple5kCaps` = `0x1414B31D0`
  - `DetectDisplayAndApplyApple5kOverrides` = `0x1414B3E80`
  - `StreamEnableWrite4F1AndPollSinkStatus` = `0x1414E7810`

- the key corrected ordering from Windows is:
  1. detect/keep the Apple tiled DP-side path connected if capability tag `0x3A` or `0x3B` applies
  2. for type/signal `128`, run HPD/AUX-ready callbacks before training
  3. publish source-DPCD before receiver-cap reads and before link training
  4. run link training with retries
  5. only after successful stream/link enable, assert sink-private `DPCD 0x4F1 = 1`

- `RetrieveDpLinkCapsAfterSourceDpcdSetup` proves source-DPCD publication is not only a late stream-enable detail:
  - it calls `ProgramDpPreTrainingAuxState`
  - waits `30 ms`
  - only then reads receiver caps at DPCD `0x000`
  - then reads extended caps at DPCD `0x2200` if advertised

- `BringUpDpOrEdpWithLinkTraining` also proves the same ordering during display bring-up:
  - for signal `32` and link-rate class `2`, it calls `SetDpcdMstCtrlEnableBit` before source-DPCD and training
  - for signal `128`, it runs the DisplayCore `+0x103F0/+0x103F8` HPD/AUX-ready callbacks
  - then it calls `ProgramDpPreTrainingAuxState`
  - only then does it call `PerformLinkTrainingWithRetries`

- `PrepareLinkTrainingStateAndArmEdp128` repeats the type-128 HPD/AUX callbacks inside each link-training attempt:
  - this matters because Windows does not train a stale logical object
  - it re-arms or waits for the live internal HPD/AUX path each retry

- `DetectDisplayEarlyExitForApple5kCaps` and `DetectDisplayAndApplyApple5kOverrides` remain important for Linux:
  - descriptor byte `+6` bit `2` / tag `0x3A` can make the detect helper return handled
  - descriptor byte `+6` bit `3` / tag `0x3B` can also make the detect helper return handled
  - tag `0x3B` has a stronger override:
    - if the underlying display object reports physically present
    - ordinary detect says disconnected
    - signal type is `11` / DisplayPort
    - Windows forces the detect result back to connected and clears status byte `+112`

- this now maps almost exactly to the Linux failure:
  - Linux preserved a logical/emulated `DP-1` sink and correct tile metadata
  - but when link training tried to write DPCD on the secondary path, AUX writes failed
  - therefore the missing Linux behavior is earlier than the final `0x4F1` latch
  - the next Linux patch should not wait until `commit_streams_end` to publish `0x310`

- next Linux experiment should be ordered more like Windows:
  1. keep the secondary Apple/tiled DP-side connector physically/live-detected before generic detect drops it
  2. publish source-DPCD, including the Windows `0x310 = 04 1D 03`, during secondary link capability/detect or immediately before link training
  3. if needed, add a signal-32 equivalent of Windows' `SetDpcdMstCtrlEnableBit` read/modify/write at DPCD `0x111`
  4. let link training run
  5. assert `0x4F1 = 1` only after the secondary AUX path is writable and/or the stream enable succeeds

- IDA comments added in this pass:
  - `0x141B8D9E2`: each training retry re-runs the type-128 HPD/AUX-ready callbacks
  - `0x141B8413D`: source-DPCD also runs during receiver-cap retrieval, before cap reads
  - `0x1414B332B`: tag `0x3A/0x3B` detect early-exit supports preserving the Apple/tiled connector
  - `0x1414B4174`: tag `0x3B` can force a physically-present DisplayPort object back to connected
  - `0x141B6A897`: post-bring-up `0x4F1` is downstream of successful link enable

### 2026-04-30 Linux patch update: move source-DPCD earlier, keep 0x4F1 conservative

- kernel source changed:
  - `C:\proj\reverse\WT6A_INF\linux-imac-5k\drivers\gpu\drm\amd\display\amdgpu_dm\amdgpu_dm.c`

- implemented the part of the Windows ordering we are confident about:
  - secondary iMac 5K head now tries to publish source-DPCD before link training, not only at `commit_streams_end`
  - early call sites are tagged in dmesg as:
    - `detect_live_pretrain`
    - `detect_connected_pretrain`
    - `boot_detect_pretrain`
    - `commit_streams_pretrain`
  - each call reads/logs DPCD `0x111` (`DP_MSTM_CTRL`) read-only for now
  - each call invokes `dpcd_set_source_specific_data(link)`
  - each call writes DPCD `0x310` (`DP_SOURCE_TABLE_REVISION`) as `04 1D 03` on this DCE generation, or `05 1D 03` on DCE 12+
  - `commit_streams_pretrain` is forced even if an earlier detect-time write succeeded, because DP power/mode transitions can discard source-DPCD state before link training

- kept the risky/private panel latch conservative:
  - `0x4F1 = 1` is still never sent unless source-DPCD was successfully programmed
  - it is now also deferred unless the secondary link reports both `link_status.link_active` and `link_state_valid`
  - this matters because AMD DC can tolerate `DC_FAIL_DP_LINK_TRAINING` and continue stream programming; in that case `link_state_valid` alone is not a strong enough success signal

- expected next-test dmesg tells us which branch we are on:
  - if early `0x310` succeeds and link training succeeds, `0x4F1` should be allowed afterward
  - if early `0x310` succeeds but link training still fails, `0x4F1` should now log a defer with `link_active=0`
  - if early `0x310` still fails, the missing piece is earlier than source-DPCD publication and likely related to Windows' type-128 HPD/AUX-ready callback path or the tag `0x3B` physically-present override

- next Windows RE pass should focus narrowly on:
  - what the type-128 `+0x103F0/+0x103F8` callbacks do before AUX/link training
  - whether those callbacks map to AMD DC HPD, AUX power, encoder routing, or a BIOS/firmware command
  - whether the signal-32 `DP_MSTM_CTRL` / DPCD `0x111` write is ever relevant to this iMac 5K internal secondary head, or only to a neighboring external/MST path

### 2026-04-30 Windows RE pass: type-128 readiness is eDP HPD wait, not a new mailbox opcode

- IDA MCP confirmed the two type-128 callback targets:
  - DisplayCore vtable `+0x103F0` = `Dce110EdpWaitForHpdReady` at `0x141CEFB20`
  - DisplayCore vtable `+0x103F8` = `Dce110EdpPostHpdReadyDelay` at `0x141CEF6A0`

- `Dce110EdpWaitForHpdReady` behavior:
  - only applies when the connector object low byte is `0x14`
  - Linux maps connector id `20` / `0x14` to `CONNECTOR_ID_EDP`
  - for power-up it looks up the connector-state/HPD handle
  - optionally waits an extra panel timing delay from the link object
  - polls readiness every `10 ms`
  - times out after `300 ms` on power-up, `500 ms` on power-down
  - logs `dce110_edp_wait_for_hpd_ready: wait timed out!` if readiness never arrives

- `Dce110EdpPostHpdReadyDelay` behavior:
  - also only applies to connector id `0x14` / eDP
  - computes elapsed time from a stored event timestamp
  - waits until panel timing offset `link/panel + 0x594 + 500 ms` has elapsed
  - this is a settle delay after HPD-ready, before downstream AUX/link operations

- type-128 readiness is used earlier than link training:
  - `QueryConnectorReadyAndArmEdp128` at `0x141B704F0` calls both callbacks before querying the connector-state HPD/readiness handle
  - `DcLinkDetectHelper` calls `QueryConnectorReadyAndArmEdp128`; if readiness cannot be resolved, detect fails before EDID/sink creation
  - `RefreshType128LinkReadyFlags` at `0x141B78FE0` collects up to two signal/type-128 links, runs the same query, and stores the result in link byte `+0x58`
  - `PrepareLinkTrainingStateAndArmEdp128` repeats the same two callbacks inside each link-training attempt

- important Linux correlation from the latest test:
  - `eDP-1` logged as `type=14 signal=128`
  - `DP-1` logged as `type=10 signal=32`
  - Windows' type-128 readiness path maps to Linux `SIGNAL_TYPE_EDP`
  - Linux currently only calls `edp_power_control()` and `edp_wait_for_hpd_ready()` in detect/training when the link/stream signal is `SIGNAL_TYPE_EDP`
  - therefore the preserved secondary `DP-1` tile is likely missing Windows' eDP HPD-ready/settle sequencing because Linux classifies it as plain DP signal 32

- the `0x111` question is now lower priority:
  - `SetDpcdMstCtrlEnableBit` reads DPCD `0x111`, sets/clears bit 0, and writes it back
  - it is used by signal-32 class-2 and signal-64/MST bring-up paths
  - it is not part of the type-128/eDP readiness path
  - IDA rename applied: `sub_141B6C9A0` -> `BringUpDpMstWithMstCtrlPreEnable`

- the `1265`/`0x4F1` question is now also cleaner:
  - `LowerControllerDpcdAuxWriteAddress` at `0x141EDFC70` forwards `(link/session id, DPCD address, payload, length)` through the lower display-miniport AUX write vfunc at `+0x190`
  - `LowerControllerDpcdAuxReadAddress` at `0x141EDDCD0` is the matching read wrapper through vfunc `+0x188`
  - so `1265` is the DPCD address `0x4F1`, not an opaque firmware mailbox selector in this path
  - Windows writes `0x4F1 = 1` via the same DPCD/AUX abstraction used for normal DP receiver registers

- next Linux implication:
  - if the current early `0x310` patch still cannot write AUX on `DP-1`, the next likely patch should not be another late latch tweak
  - instead, it should emulate the Windows type-128 readiness sequence for the iMac secondary tile even though Linux sees it as signal 32:
    - before secondary detect/link training, call or clone the safe parts of eDP HPD-ready wait/settle logic for the marked secondary connector
    - log connector id, link encoder connector id, HPD state, local sink presence, and AUX write/read result before and after the wait
    - only after this wait should source-DPCD `0x310` and normal link training run

### 2026-04-30 Linux patch update: clone the safe eDP readiness wait for the secondary tile

- kernel source changed:
  - `C:\proj\reverse\WT6A_INF\linux-imac-5k\drivers\gpu\drm\amd\display\amdgpu_dm\amdgpu_dm.c`
  - `C:\proj\reverse\WT6A_INF\linux-imac-5k\drivers\gpu\drm\amd\display\amdgpu_dm\amdgpu_dm.h`

- implemented a new iMac-only helper:
  - `amdgpu_dm_imac5k_wait_secondary_aux_ready()`
  - it is called before secondary source-DPCD writes
  - it does not call Linux's normal `edp_wait_for_hpd_ready()` directly, because that helper asserts if the connector object is not `CONNECTOR_ID_EDP`
  - instead it uses public `dc_link_get_hpd_state()` to poll the secondary link safely even though Linux currently classifies it as DP signal 32

- timing now mirrors the safe part of Windows' type-128 readiness path:
  - poll HPD every `10 ms`
  - wait up to `300 ms`
  - apply a one-time `30 ms` post-ready settle delay before the first secondary source-DPCD write
  - keep polling on forced `commit_streams_pretrain` rewrites, but avoid adding the settle delay repeatedly on later modesets
  - treat either Linux's raw HPD read or cached `link->hpd_status` as ready, because Windows' connector-state readiness is not necessarily the same as physical HPD

- new dmesg line to watch:
  - `IMAC5K: <tag> secondary AUX-ready wait DP-1 link=1 signal=32 link_conn=<id> enc_conn=<id> live_sink=<0/1> hpd_initial=<0/1> hw_hpd=<initial>/<final> cached_hpd=<initial>/<final> hpd_ready=<0/1> elapsed_ms=<n> post_delay_ms=<n> waited_before=<0/1>`

- expected interpretation:
  - `hpd_ready=0` means Linux's secondary DP object is not live at the HPD/link layer, so we need to trace how Windows' connector-state provider differs from Linux's physical HPD view
  - `hpd_ready=1` plus `0x310 status=0` means the readiness timing fixed the AUX write window
  - `hpd_ready=1` plus `0x310 status=-1` means HPD is not the only missing piece; next RE/kernel work should look at AUX routing/encoder setup rather than more DPCD payloads

### 2026-04-30 Windows RE pass: stop staring at `0x4F1`, trace the live AUX object lifecycle

This pass deliberately treats `1265` as already understood:

- decimal `1265` is DPCD address `0x4F1`
- the Windows runtime writers submit it as an AUX/DPCD transaction
- the remaining problem is not the byte value
- the remaining problem is which object-specific AUX/link backend receives the transaction, and how Windows keeps that backend live

Fresh IDA pass on the connector-state path:

- `ConstructLegacyConnectorDisplayEntry` builds the runtime link/display entry from the ATOM/VBIOS object id:
  - connector object low byte `0x13` maps to signal `32` / DisplayPort
  - connector object low byte `0x14` maps to signal `128` / eDP
  - it looks up a connector-state/GPIO handle for the object id
  - it creates the DDC/AUX service before final link-encoder construction
  - for `0x14` / eDP-like objects it may also create a panel-control object

- the connector-state provider is a real backend, not just `link->hpd_status`:
  - `LookupConnectorStateHandleByObjectId` maps object id to a connector/GPIO descriptor
  - `LookupConnectorGpioPinHandle` creates the concrete GPIO/DDC pin handle
  - `OpenConnectorStateHandleMode(..., 4)` opens that handle for HPD/readiness access
  - `QueryConnectorStateReady` calls the opened handle's readiness method
  - `CloseConnectorStateHandle` releases that opened mode afterward

- IDA names/comments added in this pass:
  - `CloseConnectorStateHandle` = `0x141BB55E0`
  - `OpenConnectorStateHandleMode` = `0x141BB5700`
  - `QueryConnectorStateReady` = `0x141BB5630`
  - `CreateConnectorGpioPinHandle` = `0x141BB5730`
  - `LookupConnectorGpioPinHandle` = `0x141BB61C0`
  - `GetConnectorHpdGpioId` = `0x141BB6430`
  - `GetConnectorDdcGpioIdMaybe` = `0x141BB64E0`
  - `SetConnectorHpdFilterDelays` = `0x141BB6490`
  - `ConstructAuxTransactionPathFactory` = `0x14152B8B0`

The display-map / stream side now ties back to the same object-specific AUX identity:

- `PopulateDisplayMapAuxObjectsForHotplugRange` inserts display-map records and stores `CreateDpcdAuxInterfaceForObjectId(...)` at display-map entry `+0x18`
- `UpdateDisplayMapEntryLiveSinkObject` later stores or clears the active sink/display object at the same entry `+0x18` based on the object's usable/connected state
- `BuildStreamForDisplayEntryPath` copies display-map entry `+0x18` into stream init-data `+0x18`
- `ConstructDpStreamObjectWithDpcdInterface` stores that value in the DP stream object at parent `+0xB8`
- `StreamEnableWrite4F1AndPollSinkStatus` then reaches it as the stream/link DPCD interface and submits `DPCD 0x4F1 = 1`

So Windows has a coherent object path:

1. ATOM/VBIOS connector object id
2. connector-state/GPIO handle
3. DDC/AUX service and link encoder
4. display-map entry
5. live sink/display object at entry `+0x18`
6. stream object carrying the same DPCD/AUX interface
7. source-DPCD, link training, then optional sink-private `0x4F1`

Important lower AUX detail:

- `AdapterServiceGetAuxHandleForObjectId` still maps object id through the VBIOS descriptor parser
- it passes descriptor `+0x1C` selector and computed source mask into `AuxFactoryCreateHandleForSourceMask`
- `ConstructDualPinAuxTransactionHandle` resolves that selector/mask to a pin index
- it then opens both pin type `3` and pin type `4` endpoints
- if either endpoint is missing, the AUX handle self-invalidates
- `DualPinAuxHandleSubmitTransaction` drives/flushes both underlying pin endpoints

This matters for Linux because our current preservation patch can keep a logical DRM connector/sink alive even if the lower AUX path is not truly live:

- preserving tile metadata and an emulated sink can make userspace see `DP-1`
- it does not prove Linux still has the correct object-specific AUX route for ROM object `0x3113`
- an AUX write failure on `DP-1` after this point likely means we preserved the display-model object but did not preserve or recreate the lower connector-state/AUX backend

Static Linux comparison to keep in mind:

- Linux `construct_phy()` has the same broad pieces but orders some gates differently:
  - it reads `get_connector_id`
  - it checks the source encoder/transmitter before DDC/HPD setup
  - only then creates `link->ddc`
  - then derives DDC line, HPD source, link encoder, signal type, device tag, integrated display path info, and HPD filter
- Windows, by contrast, creates the DDC/AUX service before final link-encoder construction and keeps a separate connector-state handle path for readiness
- that makes these high-value Linux failure candidates:
  - source-encoder/transmitter gate prevents a secondary path from ever receiving a DDC/AUX service
  - Linux creates `DP-1` but with a stale/wrong DDC/AUX pin pair for the paired 5K half
  - Linux keeps an emulated/preserved sink after detect, but the real DDC/AUX service was already destroyed or invalidated
  - Linux does not have the equivalent of the Windows object-id -> live display-map `+0x18` refresh before stream enable

Current conclusion after this pass:

- more DPCD payload guessing is not the right next move
- if the next Linux test still shows `0x310 status=-1` after the readiness wait, the kernel patch should instrument and compare lower routing:
  - ROM object id / connector index for `DP-1`
  - `link->ddc` pointer and `ddc_pin` presence before and after detect preservation
  - DDC line / AUX hardware instance
  - HPD source and IRQ source
  - link encoder transmitter / preferred engine
  - whether the preserved secondary sink is backed by a real `dc_link`/DDC service or only by `dc_em_sink`

That is the closest static analogue to Windows' working behavior: not "send opcode 1265", but "make sure the same object-specific AUX backend survives from VBIOS object discovery through stream enable."

### 2026-04-30 Windows RE pass: DCE11 AUX selector map identifies the paired-tile route

This pass ties the Windows lower AUX backend to the iMac VBIOS objects instead of treating `DP-1` as only a logical DRM name.

Live Linux VBIOS dump from `amdgpu_vbios.rom`:

- part number: `113-D0008A14GP-003`
- object-info revision: `1.3`
- display path `0`: connector `0x3114`, connector type `0x14` / eDP, encoder chain `0x2120`
- display path `1`: connector `0x3113`, connector type `0x13` / DisplayPort, encoder chain `0x2220`
- connector `0x3114`:
  - I2C/AUX bytes: `01 04 93 00`
  - HPD bytes: `02 04 02 00`
  - Windows descriptor selector: `0x4875`
  - decoded descriptor index: `3`
- connector `0x3113`:
  - I2C/AUX bytes: `01 04 92 00`
  - HPD bytes: `02 04 01 00`
  - Windows descriptor selector: `0x4871`
  - decoded descriptor index: `2`

IDA MCP then confirmed the Windows DCE11 selector resolver:

- `Dce11ResolveAuxSelectorToPinIndex` at `0x141695990`
- literal selector-to-pin map:
  - `0x4869 -> 0`
  - `0x486D -> 1`
  - `0x4871 -> 2`
  - `0x4875 -> 3`
  - `0x4879 -> 4`
  - `0x487D -> 5`
  - `0x4881 -> 6`
  - `0x4899 -> 7`
- therefore the iMac paired DisplayPort tile object `0x3113` / selector `0x4871` resolves to Windows lower AUX pin index `2`
- the primary eDP object `0x3114` / selector `0x4875` resolves to Windows lower AUX pin index `3`

The surrounding Windows object chain is now:

1. `AdapterServiceGetAuxHandleForObjectId` parses the VBIOS connector descriptor for an object id.
2. It passes descriptor selector and source mask to `AuxFactoryCreateHandleForSourceMask`.
3. `AuxFactoryCreateHandleForSourceMask` constructs a `ConstructDualPinAuxTransactionHandle`.
4. `ConstructDualPinAuxTransactionHandle` calls `ResolveSourceMaskToPinIndex`.
5. On DCE11, `ResolveSourceMaskToPinIndex` reaches `Dce11ResolveAuxSelectorToPinIndex`.
6. The resolved pin index is used to open both type-`3` and type-`4` pin endpoints.
7. If either endpoint is absent, the Windows AUX handle invalidates itself.

IDA addresses for the object-to-stream handoff:

- `AdapterServiceGetAuxHandleForObjectId` at `0x141465AA0`
  - calls the adapter descriptor lookup for the object id
  - derives the selector/source-mask pair
  - calls the AUX factory with that per-object selector
- `ConstructDpcdAuxInterfaceForObjectId` at `0x141551090`
  - calls adapter vfunc `+0x170`, which reaches `AdapterServiceGetAuxHandleForObjectId`
  - stores the returned lower AUX handle at the DPCD interface object
- `CreateConnectorGpioPinHandle` at `0x141BB5730`
  - creates the concrete connector GPIO/DDC pin handle for descriptor type `3` or `4`
- `CreateDdcGpioPinHandle` at `0x141BB5230`
  - initializes provider-specific GPIO/DDC pin metadata
  - this supports the interpretation that the Windows lower path is physical GPIO/DDC/AUX routing, not a separate opaque runtime firmware mailbox
- `PopulateDisplayMapAuxObjectsForHotplugRange` at `0x1414B8600`
  - stores `CreateDpcdAuxInterfaceForObjectId(...)` into display-map entry `+0x18`
- `BuildStreamForDisplayEntryPath` at `0x1414B55D0`
  - copies display-map entry `+0x18` into stream init-data `+0x18`
- `ConstructDpStreamObjectWithDpcdInterface` at `0x1414E8140`
  - carries that DPCD/AUX interface into the stream object
- therefore later source-DPCD, link-training, and `0x4F1` writes remain tied to the original object-specific lower AUX route

Linux-side correlation from source:

- Linux `link_factory.c::translate_encoder_to_transmitter()` maps `ENCODER_ID_INTERNAL_UNIPHY1 + ENUM_ID_1` to `TRANSMITTER_UNIPHY_C`
- it maps `ENCODER_ID_INTERNAL_UNIPHY1 + ENUM_ID_2` to `TRANSMITTER_UNIPHY_D`
- the VBIOS eDP encoder `0x2120` is `UNIPHY1 enum 1`, so Linux should build it on `UNIPHY_C`
- the VBIOS paired DisplayPort encoder `0x2220` is `UNIPHY1 enum 2`, so Linux should build it on `UNIPHY_D`

Current conclusion:

- the static target for the secondary half is now exact enough to instrument:
  - connector object id `0x3113`
  - selector `0x4871`
  - Windows AUX pin index `2`
  - Linux transmitter `TRANSMITTER_UNIPHY_D`
  - VBIOS I2C/AUX uid bytes `01 04 92 00`
  - VBIOS HPD bytes `02 04 01 00`
- this strongly suggests the next Linux check should not merely ask whether `DP-1` exists
- it should ask whether the preserved/live `DP-1` object still owns the expected lower route:
  - `link->link_id`
  - connector index
  - source encoder object id
  - `link->ddc_hw_inst`
  - `dal_ddc_get_line(get_ddc_pin(link->ddc))`
  - `link->hpd_src`
  - `link->irq_source_hpd`
  - `link->irq_source_hpd_rx`
  - `link->link_enc->transmitter`
  - `link->link_enc->preferred_engine`
  - `link->link_enc_hw_inst`

This explains why the preserved Linux connector could still fail AUX writes: keeping the display/tile object alive is not enough if the lower DDC/AUX service for object `0x3113` was never constructed, was constructed against the wrong pin, or was invalidated before stream enable.

### 2026-04-30 Windows RE pass: corrected live-slot model, two tables not one

This pass corrected an important ambiguity in the previous notes.

There are two related Windows topology structures:

- a 64-byte display-map entry table
- a 104-byte per-object detection/record table

The 64-byte display-map table:

- `GetDisplayMapEntryByIndex` indexes entries as `base + (index << 6)`
- `InsertInterfaceIntoDisplayMap` inserts these 64-byte entries
- `PopulateDisplayMapAuxObjectsForHotplugRange` can seed entry `+0x18` with `CreateDpcdAuxInterfaceForObjectId(...)`
- `BuildStreamForDisplayEntryPath` copies the 64-byte entry `+0x18` into stream init-data `+0x18`
- `ConstructDpStreamObjectWithDpcdInterface` stores that stream init-data at stream object `+0xB8`
- later stream-enable DPCD writes reach that same interface as the stream object's DPCD/AUX backend

The 104-byte per-object detection/record table:

- built by `ConstructDisplayObjectRecordTable` at `0x1414B46D0`
- one record per object id from the adapter/object provider
- record layout observed so far:
  - `+0x00`: object id core fields
  - `+0x08`: base/display interface pointer
  - `+0x10..+0x13`: policy/state flags
  - `+0x14`: HPD reschedule count
  - `+0x18`: currently live display/sink object
  - `+0x20`: candidate display object pointer array
  - `+0x30`: candidate count
  - `+0x38/+0x40`, `+0x48/+0x50`, `+0x58/+0x60`: timer source / timer handle pairs
- `FindDisplayObjectRecordByObjectId` at `0x1414AF460` searches this table using record size `104`

How the 104-byte record gets candidates:

- `ConstructDisplayTopologyManager` at `0x14143A840` builds display objects, then iterates them
- at `0x14143B57E`, each object is added to the detection record table unless bits `4`, `5`, or `10` in the object's vfunc `+0x60` flags are set
- `AddDisplayObjectToDetectionRecord` at `0x1414B4450`:
  - finds the 104-byte record by object id
  - initializes record `+0x08` from the display object's vfunc `+0xA8` if needed
  - assigns timer/IRQ sources through `DetectionRecordAssignTimerSource`
  - accepts at most two candidate display objects
  - appends the candidate at `record + 0x20 + 8 * count`
  - increments candidate count at `record + 0x30`

How the 104-byte record becomes live:

- `HandleDisplayDetectionResultAndLiveState` at `0x1414A8AE0` receives a detection result
- detection result byte `+0x72` is the connected flag
- detection result byte `+0x70` is a change/rescan flag
- if connected state changed, rescan is requested, or the object is embedded-panel-like, it calls `ApplyDetectionResultAndUpdateLiveDisplayObject`
- `ApplyDetectionResultAndUpdateLiveDisplayObject` at `0x1414A8110` attaches stream interfaces for each signal when the object is connected or embedded-panel-like
- it then calls `UpdateDisplaySinkConnectionAndLiveAuxState`
- `UpdateDisplaySinkConnectionAndLiveAuxState` at `0x1414A7D90`:
  - writes the connected state back to the display object through vfunc `+0x418`
  - calls `UpdateDisplayMapEntryLiveSinkObject`
- `UpdateDisplayMapEntryLiveSinkObject` at `0x1414B3A20`:
  - finds the 104-byte record for the display object's id
  - verifies the object is already in the candidate array at `record +0x20`
  - if the object's vfunc `+0x258` says usable/connected, stores the object at `record +0x18`
  - otherwise clears `record +0x18`

Timer/private-neighborhood note:

- `ConstructDisplayObjectRecordTable` reads three nearby controller/DPCD-like addresses:
  - `0x4E1` into record-table `+0x78`
  - `0x501` into record-table `+0x7C`
  - `0x521` into record-table `+0x80`
- these are not the later sink latch write at `0x4F1`
- `DetectionMgrHandleHpdInterrupt` uses record-table `+0x80` as the delay for the longer HPD timer path
- so `0x521` appears to be HPD/detection timing policy, not the panel-pairing latch itself

Follow-up use of `0x4E1` / `0x501`:

- `HandleDisplayDetectionResultAndLiveState` calls `UpdateDisplayBaseInterfacePrivateTimingPolicy` after `ApplyDetectionResultAndUpdateLiveDisplayObject`
- `UpdateDisplayBaseInterfacePrivateTimingPolicy` at `0x1414B37F0` calls the display object's base/interface vfunc `+0x70`
- inputs to that vfunc come from:
  - record-table `+0x78`, sourced from private-neighborhood read `0x4E1`
  - record-table `+0x7C`, sourced from private-neighborhood read `0x501`
  - optional display/pin property `42`
- branch behavior:
  - if a lower descriptor byte `+4` is negative, call vfunc `+0x70` with `0x4E1` value and optional property `42`
  - if display descriptor byte `+0x0C` bit `1` is set, call vfunc `+0x70` with `(0, 0)`
  - otherwise call vfunc `+0x70` with the `0x4E1` and `0x501` values
- current interpretation:
  - `0x4E1`, `0x501`, and `0x521` are part of Windows' private detection/timing policy neighborhood
  - they are not the same operation as the later `0x4F1 = 1` sink latch
  - but they do happen before/around live-object maintenance, so Linux may need equivalent timing/readiness behavior even if it never writes those exact private bytes

Apple/tiled tag reachability nuance:

- `DetectDisplayAndApplyApple5kOverrides` initializes detection result byte `+0x72` from the display object's current connected/usable state
- `DetectDisplayEarlyExitForApple5kCaps` can return handled before the normal detect path when descriptor byte `+6` bit `2` or bit `3` is present
- the later explicit descriptor byte `+6` bit `3` branch can set detection result `+0x72 = 1` and clear `+0x70`
- however, if the early helper sees the same descriptor bit first, the later explicit `0x3B` force-connected branch is bypassed
- the robust static conclusion is therefore:
  - the `0x3A/0x3B` tag family participates in preserving or forcing the Apple/tiled DP-side path through detection
  - the exact branch reached depends on detect phase and descriptor context
  - Linux should mirror the outcome, not blindly copy only one branch: the secondary `0x3113` path must not be dropped by ordinary DP disconnect handling before the stream/DPCD backend is attached

IDA names added in this pass:

- `FindDisplayObjectRecordByObjectId` = `0x1414AF460`
- `AddDisplayObjectToDetectionRecord` = `0x1414B4450`
- `ApplyDetectionResultAndUpdateLiveDisplayObject` = `0x1414A8110`
- `HandleDisplayDetectionResultAndLiveState` = `0x1414A8AE0`
- `DetectionMgrHandleInterrupt` = `0x1414B35A0`
- `DetectionMgrHandleHpdInterrupt` = `0x1414AFA70`
- `DetectionMgrNotifyPlugUnplug` = `0x1414AF740`
- `DetectionRecordScheduleTimer` = `0x1414AF5A0`
- `DetectionRecordCancelTimer` = `0x1414AF4F0`
- `DetectionRecordAssignTimerSource` = `0x1414AF6E0`
- `ResolveDetectionTimerIrqSource` = `0x1414AF1F0`
- `UpdateDisplayBaseInterfacePrivateTimingPolicy` = `0x1414B37F0`

Linux implication:

- the Windows "live" state has two levels:
  - stream/DPCD backend identity from the 64-byte display-map path
  - currently live usable candidate from the 104-byte detection record path
- Linux preserving a DRM connector/tile is not enough if it does not also preserve the equivalent of:
  - candidate object creation for the `0x3113` route
  - connected/usable result propagation
  - live object selection before stream attachment and DPCD writes
- if Linux has a preserved `DP-1` object but its AUX writes still fail, the failure may be between "candidate exists" and "record live slot equivalent is usable", not only at the final source-DPCD or `0x4F1` stage

### 2026-04-30 Windows RE pass: detect wrapper gate, DPCD object quirks, and final display-map liveness

This pass tightened three pieces that matter before another Linux patch:

1. when the Windows detect wrapper actually refreshes the live object state
2. how the per-object DPCD/AUX object is built for `0x3113`
3. how the display-map entry becomes live enough for stream construction

#### Detect wrapper gate

`DetectDisplayMainMaybeApple5k` at `0x1414A96C0` now has a clearer control flow:

- `NormalizeDetectPassForPendingDisplay` = `0x1414A9AA0`
  - if display object is null or pass is `1`, returns the original pass
  - otherwise reads the display index through vfunc `+72`
  - if the display index is already in the pending set at topology manager `+392`, returns pass `1`
  - otherwise inserts the display index and returns the original pass
- `ClearPendingDetectDisplayIfNeeded` = `0x1414A9A40`
  - removes the display index from the same pending set when pass is not `1`
- `DetectionRecordTableGetBusyFlag` = `0x1414B42E0`
  - returns record-table byte `+0x74`
- `DetectionRecordTableSetBusyFlag` = `0x1414B4300`
  - writes record-table byte `+0x74`
- `IsDetectPass3Or4Or6` = `0x141545160`
  - true for detect passes `3`, `4`, and `6`
- `DetectDisplayStatusMaybeAssert4F1` = `0x1414B3CE0`
  - alternate detect/status helper that also reaches the detect-time `0x4F1` writer

The important nuance is the gate after `DetectDisplayAndApplyApple5kOverrides`:

- the 120-byte detection-result buffer has:
  - result `+0x72`: connected flag
  - result `+0x73`: special side-path flag
- if result `+0x73` is set, the wrapper takes a special branch and does not call `HandleDisplayDetectionResultAndLiveState`
- otherwise `HandleDisplayDetectionResultAndLiveState` runs only if:
  - `DetectDisplayAndApplyApple5kOverrides` returned nonzero, or
  - display-object flags bit `0` is set and the normalized detect pass is not `1`

This changes the interpretation of the Apple tag path:

- `DetectDisplayEarlyExitForApple5kCaps` can short-circuit ordinary detect when descriptor byte `+6` bit `2` or bit `3` is present
- that early-exit path can preserve state without necessarily forcing the live-slot update itself
- when normal detect reaches the explicit descriptor byte `+6` bit `3` branch, Windows forces a signal-`11` DisplayPort object connected and clears result `+0x70`
- therefore the robust Windows behavior is not "always force connected"
- it is "do not let the Apple/tiled DP-side object fall through ordinary disconnect handling unless the detect phase/context says that is safe"

Linux implication:

- for the OCLP/probe-boot keep-5K case, a late emulated sink is not quite equivalent
- Linux should preserve or reconstruct the equivalent of the Windows detect-wrapper outcome:
  - keep `0x3113` / `DP-1` from being dropped by normal DP detect
  - make sure it reaches stream construction with a live lower AUX path
  - only then do source-DPCD and `0x4F1` make sense

#### AdapterService lower AUX construction

`InitializeAdapterServiceForConnector` at `0x141467800` now looks like the constructor for the whole lower display-service backend:

- builds the base pin object at AdapterService `+0x170`
- creates the signal-specific display base interface at AdapterService `+0x180`
  - `CreateAdapterDisplayBaseInterfaceBySignalType` = `0x1414676A0`
  - signal type `11` -> `sub_14152F670`
  - signal type `12` -> `sub_14152C620`
  - signal type `13` -> `sub_14152D3D0`
- if base-pin capability flags say AUX is disabled, AdapterService `+0x1A8` is set to null
- otherwise `CreateAuxTransactionPathFactory` creates the AUX path factory at AdapterService `+0x1A8`
- creates the VBIOS object-info parser at AdapterService `+0x198`
- creates the object descriptor adapter at AdapterService `+0x168`
- creates descriptor-derived capability flags at AdapterService `+0x178`
- creates vendor/product capability tables at AdapterService `+0x188`

New function names added:

- `AdapterServiceGetBasePinSignalType` = `0x1414672C0`
- `AdapterServiceGetBasePinEnumValue` = `0x141467280`
- `CreateAdapterDisplayBaseInterfaceBySignalType` = `0x1414676A0`
- `ConstructDisplayDescriptorCapabilityFlags` = `0x14152FBD0`

`AdapterServiceQueryObjectDescriptor` at `0x1414663A0` is vtable `+0x100` and forwards object-id descriptor queries to the provider at AdapterService `+0x140`, provider vfunc `+0x58`.

`ConstructAuxTransactionPathFactory` at `0x14152B8B0` now has two critical fields:

- factory `+0xA8`: underlying pin-map/capability object
- factory `+0xB0`: source-mask/selector to pin-index resolver

For the iMac VBIOS this resolver is the already-decoded DCE11 map:

- `0x3113` / selector `0x4871` -> Windows AUX pin index `2`
- `0x3114` / selector `0x4875` -> Windows AUX pin index `3`

`ConstructDualPinAuxTransactionHandle` then opens two concrete endpoints for the resolved pin index:

- endpoint type `3`
- endpoint type `4`

If either endpoint is missing, the AUX handle invalidates itself.

Linux implication:

- the next useful kernel instrumentation is not "does DP-1 exist?"
- it is "does DP-1 own the correct lower route?"
- for this machine, the target route remains:
  - ROM object `0x3113`
  - connector type `0x13`
  - selector `0x4871`
  - Windows pin index `2`
  - Linux transmitter `UNIPHY_D`
  - HPD bytes `02 04 01 00`
  - I2C/AUX bytes `01 04 92 00`

#### Per-object DPCD AUX object quirks

`CreateDpcdAuxInterfaceForObjectId` = `0x141551400`

`ConstructDpcdAuxInterfaceForObjectId` = `0x141551090`

This object is the public DPCD/AUX interface later stored at display-map entry `+0x18`.

The constructor does these relevant things:

- calls controller vfunc `+0x170` to obtain the per-object lower AUX handle from AdapterService
- reads option `DalDPSkipPowerOff` into object field `+0x884`
- reads option `DalDPAuxPowerUpWaDelay` into object field `+0x888`
- reads option `DPDelay4I2CoverAUXDEFER` into object field `+0x880`
- if `DPDelay4I2CoverAUXDEFER` is absent, defaults it to true when object-id low byte is `0x13`
- therefore the iMac paired tile object `0x3113` gets that AUX-defer policy by default
- reads option `DPTranslatorDelay4I2CoverAUXDEFER` into object field `+0x87C`, defaulting to `5`
- sets object byte `+0x88C` for object-id low byte `0x14` or `0x0E`
- therefore `0x3114` is in the internal/eDP-like bucket, but `0x3113` is not

DPCD vtable entries decoded:

- `DpcdAuxReadUpTo16Bytes` = `0x14154F320`
  - vfunc `+0`
  - used for normal DPCD reads and the later `0x205` status poll
- `DpcdAuxWriteUpTo16Bytes` = `0x14154F020`
  - vfunc `+8`
  - used by the known `DPCD 0x4F1 = 1` writers
- `ReadDpcdCapsIntoAuxInterface` = `0x14154DF40`
  - vfunc `+0x10`
  - reads core DPCD and OUI blocks

`ReadDpcdCapsIntoAuxInterface` behavior:

- if `DalDPSkipPowerOff` is set, writes DPCD `0x600 = 1` before the first core DPCD read, retrying up to five attempts
- if `DalDPAuxPowerUpWaDelay` is set, waits `500 ms`
- reads DPCD `0x000..0x00F`
- reads DPCD `0x200`
- reads DPCD `0x400..0x408`
- reads DPCD `0x409..0x40B`
- reads DPCD `0x509..0x50B`
- reads DPCD `0x500..0x508`
- stores the sink OUI in the DPCD object

Special OUI behavior:

- if stored OUI is `0x90CC24`, both read and write calls use special NAK retry wrappers:
  - `DpcdAuxReadRetryNakSpecialOui` = `0x14154F1B0`
  - `DpcdAuxWriteRetryNakSpecialOui` = `0x14154EEB0`
- those wrappers retry AUX status `5` / NAK up to seven times and log it as "treat as defer"

This matters because Linux can fail before `0x4F1` for two different reasons:

- no real per-object AUX backend exists for `0x3113`
- the backend exists, but Linux gives up on AUX readiness/NAK behavior earlier than Windows

The first remains the stronger suspect from our current logs, but the OUI/defer behavior is now a concrete secondary suspect.

#### Display-map entry liveness

`PopulateDisplayMapAuxObjectsForHotplugRange` = `0x1414B8600`

This path is called from `ConstructDisplayTopologyManager` and is the only direct xref to `CreateDpcdAuxInterfaceForObjectId`.

For each object id in the hotplug/object range:

1. create a display-map interface with `CreateDisplayMapInterfaceForObjectIdMaybeSkipInternal`
2. insert it into the display map
3. create the per-object DPCD AUX interface with `CreateDpcdAuxInterfaceForObjectId`
4. store that DPCD AUX interface at display-map entry `+0x18`
5. recursively expand the object path through `ExpandDisplayPathAndConstructStreams`

New function name:

- `CreateDisplayMapInterfaceForObjectIdMaybeSkipInternal` = `0x141552480`

Its filter is interesting:

- it only accepts class-3 display/sink object ids
- `0x3113` and `0x3114` both qualify
- if global option bit `17` is enabled, it skips object-id low byte `0x14` and `0x0E`
- it does not skip low byte `0x13`
- therefore this display-map population path can favor the paired DP-side `0x3113` object over the primary/internal `0x3114` object under that option

New function name:

- `ExpandDisplayPathAndConstructStreams` = `0x1414B7AC0`

This function:

- walks class-2 intermediate transport objects recursively
- creates display transport interfaces when needed
- appends transport/display entries to the accumulated path
- eventually calls `ConstructDisplayObjectAndStreams`

`ConstructDisplayObjectAndStreams` then marks display-map entries live:

- `MarkDisplayMapEntryLive` = `0x1414B69C0`
- display-map byte `+0x10` is marked present/constructed
- display-map byte `+0x11` is marked live
- if an entry was already live, the interface flags are ORed with bit `0`
- all sub-stream/path interfaces are also marked live

This gives a three-level Windows liveness model:

1. display-map entry has per-object DPCD/AUX backend at `+0x18`
2. display-map entry bytes `+0x10/+0x11` show constructed/live path state
3. 104-byte detection record `+0x18` selects the currently live sink/display candidate

Linux implication:

- a visible DRM connector plus tile metadata is only level zero
- the closer Windows analogue is:
  - object path exists
  - object path has the correct per-object AUX backend
  - object path is marked live before stream construction
  - detection result does not tear it down before source-DPCD/link enable
- this supports doing one more Linux patch around early link/AUX preservation and readiness, not a random late `0x4F1` replay

#### IDA updates from this pass

Renames added:

- `NormalizeDetectPassForPendingDisplay` = `0x1414A9AA0`
- `ClearPendingDetectDisplayIfNeeded` = `0x1414A9A40`
- `DetectionRecordTableGetBusyFlag` = `0x1414B42E0`
- `DetectionRecordTableSetBusyFlag` = `0x1414B4300`
- `IsDetectPass3Or4Or6` = `0x141545160`
- `DetectDisplayStatusMaybeAssert4F1` = `0x1414B3CE0`
- `AdapterServiceGetBasePinSignalType` = `0x1414672C0`
- `AdapterServiceGetBasePinEnumValue` = `0x141467280`
- `CreateAdapterDisplayBaseInterfaceBySignalType` = `0x1414676A0`
- `ConstructDisplayDescriptorCapabilityFlags` = `0x14152FBD0`
- `CreateDisplayMapInterfaceForObjectIdMaybeSkipInternal` = `0x141552480`
- `ExpandDisplayPathAndConstructStreams` = `0x1414B7AC0`

Comments added at:

- detect wrapper gate: `0x1414A975E`, `0x1414A97E9`, `0x1414A97F7`, `0x1414A9850`
- Apple/tiled detect branches: `0x1414B3F48`, `0x1414B4174`
- AdapterService descriptor/AUX construction: `0x1414663E9`, `0x141467B9A`, `0x141467D3A`, `0x14146800E`
- AUX selector and endpoint construction: `0x14152BA61`, `0x14152BD87`, `0x1415B0D0E`, `0x1415B0D4A`, `0x1415B0DB7`
- DPCD object construction and power/readiness: `0x1415511A1`, `0x14155132F`, `0x1415513D1`, `0x14154DFFB`, `0x14154E03D`, `0x14154E07D`, `0x14154E0C4`, `0x14154E251`, `0x14154E415`
- display-map interface/liveness: `0x1415524B4`, `0x141552658`, `0x1414B8706`, `0x1414B875E`, `0x1414B8883`, `0x1414B7B71`, `0x1414B7C96`, `0x1414B752D`, `0x1414B75EB`, `0x1414B6A3F`, `0x1414B6A68`, `0x1414B6A71`

Current static confidence after this pass:

- high confidence: `0x4F1` is a sink-private DPCD write through the per-object DPCD AUX interface
- high confidence: `0x3113` is the paired 5K DP-side object, with selector `0x4871` and Windows AUX pin index `2`
- high confidence: Windows has special detect and display-map liveness paths that prevent `0x3113` from being treated like a normal detachable DP port
- medium/high confidence: Linux's next patch should target early construction/preservation of the `0x3113` lower AUX/link path and Windows-like AUX readiness/retry behavior before attempting final `0x4F1`
- lower confidence: a blind compositor/KDE/GNOME-only fix will be sufficient, because the Windows evidence still points below userspace

### 2026-04-30 Windows RE pass: bit-17 is a base-pin capability, not a global option

This was the narrow follow-up after the display-map pass.

The previous notes called the `>> 17` predicate a global option bit because the call sites look like:

- adapter/controller vfunc `+0x248`
- returned dword `>> 17`

That is now corrected.

The vtable entry is:

- `AdapterServiceGetBasePinProperty13Flags` = `0x1414670C0`
- public AdapterService vtable `off_141F9CB00 + 0x248`
- implementation delegates to the base-pin object at AdapterService `+0x170`
- it calls base-pin vfunc `+0x98`

The base-pin getters are:

- `CBasePinGetProperty6Flags` = `0x14152D880`
  - main `CBasePin` vtable `off_141FA90E8 + 0x90`
  - returns capability property index `6`
- `CBasePinGetProperty13Flags` = `0x14152D810`
  - main `CBasePin` vtable `off_141FA90E8 + 0x98`
  - returns capability property index `13`
- `PinCapabilityGetIndexedDwordProperty` = `0x141697CB0`
  - returns `property[index]` for index `< 32`

`ConstructCBasePinWithCapabilityName` builds the base-pin capability object from AdapterService init fields.

`ConstructBaseDfpPinCapabilityName` = `0x141697DF0`

The important property initializers are:

- capability property `6` comes from constructor input `a2[8]`
- capability property `13` comes from constructor input `a2[9]`

Tracing those inputs back through `sub_141429530`:

- source field `adapter_init_source + 0x6C` is translated by `TranslateRawDceFlagsToPinProperty6` = `0x14146C890`
- source field `adapter_init_source + 0x70` is translated by `TranslateRawDceFlagsToPinProperty13` = `0x14146C9F0`

So the `>> 17` predicate is:

```text
adapter init source +0x70 raw bit 0x80000
  -> TranslateRawDceFlagsToPinProperty13()
  -> base-pin capability property 13 bit 17 / 0x20000
  -> AdapterService vfunc +0x248
  -> display-map and detect-wrapper policy
```

This is board/ASIC/base-pin capability state, not an ordinary DAL registry option.

#### Property 13 high-value mapping

`TranslateRawDceFlagsToPinProperty13` maps raw source field `+0x70` like this:

- always starts with property bit `0x10000` set
- raw `0x10`:
  - sets property bits `0x1`, `0x2`, `0x4`, `0x8`
  - clears property bit `0x10000`
- raw `0x40` -> property `0x10`
- raw `0x80` -> property `0x20`
- raw `0x100` -> property `0x40`
- raw `0x200` -> property `0x80`
- raw `0x800` -> property `0x100`
- raw `0x1000` -> property `0x200`
- raw `0x2000` -> property `0x400`
- raw `0x4000` -> property `0x800`
- raw `0x8000` -> property `0x1000`
- raw `0x10000` -> property `0x2000`
- raw `0x20000` -> property `0x4000`
- raw `0x40000` -> property `0x8000`
- raw `0x80000` -> property `0x20000`
- raw `0x200000` -> property `0x40000`

The confirmed `0x3113`-relevant consumer is property `13` bit `17`:

- `CreateDisplayMapInterfaceForObjectIdMaybeSkipInternal` checks property `13` bit `17`
- when set, low-byte `0x14` and low-byte `0x0E` class-3 display objects are skipped
- low-byte `0x13` is not skipped
- therefore ROM object `0x3113` survives this population path while `0x3114` / type `0x14` can be skipped under the same policy

The same property `13` bit `17` also gates detect-pass busy-flag handling:

- `DetectDisplayMainMaybeApple5k` checks property `13` bit `17`
- when set and detect pass is `3`, it sets the detection record-table busy flag before detect
- after detect, the same predicate clears the busy flag

That means bit `17` participates in both:

- display-map object selection
- detect-state scheduling/serialization

#### Property 6 high-value mapping

The neighboring capability property is also important because it can suppress AUX creation.

`TranslateRawDceFlagsToPinProperty6` maps raw source field `+0x6C` like this:

- raw `0x800` -> property `0x2`
- raw `0x8000` -> property `0x1`
- raw `0x10000` -> property `0x4`
- raw `0x40000` -> property `0x8`
- raw `0x100000` -> property `0x10`
- raw `0x400000` -> property `0x20`
- raw `0x800000` -> property `0x40`
- raw `0x1000000` -> property `0x80`
- raw `0x2000000` -> property `0x100`
- raw `0x4000000` -> property `0x200`

The confirmed consumer is:

- `InitializeAdapterServiceForConnector`
- it reads `CBasePinGetProperty6Flags`
- if property `6` bit `4` / `0x10` is set, AdapterService does **not** create the AUX transaction factory at `+0x1A8`

So the exact suppressor is:

```text
adapter init source +0x6C raw bit 0x100000
  -> TranslateRawDceFlagsToPinProperty6()
  -> base-pin capability property 6 bit 4 / 0x10
  -> AdapterService skips AUX factory creation
```

This is the lower-layer "AUX can never work" gate.

#### What this changes

This removes one ambiguity from the previous section:

- the `0x3113`-preserving display-map filter is not a random global Windows setting
- it is a base-pin capability path derived during adapter initialization
- the same capability family also controls whether the lower AUX factory exists at all

Static Linux implication:

- for probe/OCLP boot, if Linux already sees `DP-1` but AUX writes fail, we should log whether its lower route has the equivalent of Windows property `6` clear and property `13` bit `17` set
- for plain boot, if Linux falls back to 4K, it may be missing this base-pin capability outcome entirely
- a clean Linux quirk can model the outcome directly:
  - do not treat `0x3113` as a normal detachable DP object during early detect
  - do not suppress its AUX/DDC backend
  - keep the type `0x13` object path selectable/live while not relying on type `0x14` alone

This also makes the next kernel instrumentation more exact:

- log the VBIOS object id and connector type for each `dc_link`
- log whether link construction suppressed DDC/AUX for the secondary path
- log the resolved HPD/DDC/AUX instance for `0x3113`
- add an iMac19,1 quirk that explicitly preserves the Windows-equivalent `0x3113` capability outcome before stream construction

#### IDA updates from this pass

Renames added:

- `AdapterServiceGetBasePinProperty13Flags` = `0x1414670C0`
- `CBasePinGetProperty6Flags` = `0x14152D880`
- `CBasePinGetProperty13Flags` = `0x14152D810`
- `PinCapabilityGetIndexedDwordProperty` = `0x141697CB0`
- `ConstructBaseDfpPinCapabilityName` = `0x141697DF0`
- `TranslateRawDceFlagsToPinProperty6` = `0x14146C890`
- `TranslateRawDceFlagsToPinProperty13` = `0x14146C9F0`

Comments added at:

- `0x1414670F5`
- `0x14152D868`
- `0x14152D8D8`
- `0x141697CCE`
- `0x141697F80`
- `0x141697F9D`
- `0x141429A33`
- `0x141429A7C`
- `0x14146CBF5`
- `0x14146C92B`
- `0x1415525E8`
- `0x141552658`
- `0x1414A9729`
- `0x1414A9A0B`

### 2026-04-30 Windows RE pass: upstream source of base-pin property6/property13 source flags

This pass was intentionally narrow after the previous base-pin capability result.

The open question was:

- where do the `adapter_init_source +0x6C` and `adapter_init_source +0x70` raw fields come from?
- are they direct VBIOS/ATOM connector-record bits, or are they synthesized by the Windows KMD/DAL init path?

The answer is now clearer:

- `InitializeDalFromKmdAdapter` = `0x1401A0380`
- it builds a 152-byte DAL init structure on the stack
- it calls `PopulateDalAdapterInitSourceFlags` = `0x14019F1A0`
- then it sets DAL init field `+0x18` to `display_interface + 0x140`
- that pointer is the `adapter_init_source` consumed later by `CreateDal2CoreWrapper` and `ConstructDal2CoreAndServices`

So:

```text
KMD display interface object
  +0x140 = adapter_init_source base passed into DAL2/Core
  +0x1AC = adapter_init_source +0x6C raw property6 source flags
  +0x1B0 = adapter_init_source +0x70 raw property13 source flags
```

The raw flags are not simply copied from one ATOM connector object record.

`PopulateDalAdapterInitSourceFlags` first zeroes the eight bytes covering `adapter_init_source +0x6C` and `+0x70`, then synthesizes them from KMD adapter predicates, ASIC/core feature predicates, and DAL feature-object predicates.

#### Property13 bit17 upstream source

The previously decoded policy chain was:

```text
adapter_init_source +0x70 raw bit 0x80000
  -> TranslateRawDceFlagsToPinProperty13()
  -> base-pin capability property13 bit17 / 0x20000
  -> display-map and detect-wrapper policy
```

The upstream writer is:

- `PopulateDalAdapterInitSourceFlags` sets a local byte at display-interface `+0x1D0`
- that byte is set when:
  - KMD adapter vfunc `+0x250` or vfunc `+0x260` returns true
  - and the DAL feature object vfunc `+0x4D8` returns false
- later, if that byte is set, Windows ORs `0x80000` into `adapter_init_source +0x70`

Therefore property13 bit17 is a synthesized Windows KMD/DAL adapter policy result.

It is still board/ASIC/display-stack capability state, but it is not a plain "read this one connector-record bit" result.

This matters because property13 bit17 was the bit that:

- lets the display-map path skip low-byte `0x14` / `0x0E` class-3 display objects
- does not skip low-byte `0x13`
- therefore preserves the paired DP-side `0x3113` path in that population/filter path
- also participates in the detect-pass busy-flag scheduling around pass `3`

#### Property6 AUX-suppressor upstream source

The previously decoded suppressor chain was:

```text
adapter_init_source +0x6C raw bit 0x100000
  -> TranslateRawDceFlagsToPinProperty6()
  -> base-pin capability property6 bit4 / 0x10
  -> AdapterService skips AUX transaction factory creation
```

The upstream writer is also `PopulateDalAdapterInitSourceFlags`.

The raw `0x100000` bit can be set through multiple paths:

- core/hardware predicate vfunc `+0x220` and DAL feature-object test id `300` set raw mask `0x100000`
- DAL feature-object test id `300` and core/hardware predicate vfunc `+0x2F8` set raw mask `0x140000`
- DAL feature-object vfunc `+0x520` sets raw mask `0x940000`

All three masks include raw bit `0x100000`.

So the AUX-factory suppressor is also synthesized from Windows-side KMD/DAL/ASIC feature state, not directly from the iMac connector object id alone.

We still do not know, statically, which of these predicates is true on the exact iMac runtime path.

But the Windows logic tells us what outcome matters:

- the `0x3113` route must not be filtered like an ordinary detachable DP object
- the lower AUX/DDC backend for `0x3113` must exist and remain usable
- if a Linux route has the right connector object but AUX writes fail, the missing piece is the lower AdapterService-equivalent route/pin/backend state, not the final `0x4F1` byte

#### Linux implication after this pass

This makes the next kernel work more justifiable, not less:

- do not hunt for a single ATOM/VBIOS flag that exactly equals Windows property13 bit17
- do not expect Linux to already expose Windows property6/property13 as named fields
- instead, on the tightly gated iMac19,1 panel path, explicitly model the Windows outcome:
  - keep ROM object `0x3113` / selector `0x4871` selectable/live
  - do not suppress its AUX/DDC backend
  - preserve or reconstruct its lower route to Windows AUX pin index `2` / Linux `UNIPHY_D`
  - only then try Windows-like source-DPCD publication and the sink-private `0x4F1 = 1` latch

This is also a warning against more blind kernel tests:

- if the next Linux patch still cannot write DPCD `0x310` on `DP-1`, the failure is probably before link training and before `0x4F1`
- the next logs should prove whether `DP-1` owns the correct lower route, AUX instance, DDC pin, HPD source, and encoder/transmitter identity

#### IDA updates from this pass

Renames added:

- `PopulateDalAdapterInitSourceFlags` = `0x14019F1A0`
- `InitializeDalFromKmdAdapter` = `0x1401A0380`
- `CreateDisplayInterfaceAndInitializeDal` = `0x14019EC00`
- `ConstructDisplayInterfaceCommon` = `0x14019E4A0`
- `CreateDal2CoreWrapper` = `0x14142BBF0`
- `ConstructDal2CoreAndServices` = `0x141429530`
- `SelectDalGeneration` = `0x14142BF50`
- `IsForcedDalGenerationOptionSet` = `0x14142BDB0`

Comments added at:

- `0x1401A0574`
- `0x14019F2E4`
- `0x14019F4A5`
- `0x14019F2CB`
- `0x14019F8D6`
- `0x14019F603`
- `0x14019F643`
- `0x14019F663`
- `0x141429A33`
- `0x141429A7C`

### 2026-04-30 Windows RE pass: lower AUX handle must prove the route, not just the connector

This was a final narrow pass before the next Linux patch.

The goal was to avoid another blind kernel experiment by checking exactly what Windows needs below the logical display object before source-DPCD and `0x4F1` writes can work.

The important functions are:

- `ConstructDpcdAuxInterfaceForObjectId`
- `AuxFactoryCreateHandleForSourceMask`
- `ConstructDualPinAuxTransactionHandle`
- `Dce11ResolveAuxSelectorToPinIndex`

The Windows path is:

```text
display object id
  -> descriptor selector/source mask
  -> AdapterService vfunc +0x170
  -> AUX factory
  -> DCE11 selector resolver
  -> concrete AUX pin index
  -> open lower endpoint type 3
  -> open lower endpoint type 4
  -> only then keep the DPCD/AUX handle alive
```

The DCE11 selector map is literal:

- `0x4869 -> AUX pin index 0`
- `0x486D -> AUX pin index 1`
- `0x4871 -> AUX pin index 2`
- `0x4875 -> AUX pin index 3`
- `0x4879 -> AUX pin index 4`
- `0x487D -> AUX pin index 5`
- `0x4881 -> AUX pin index 6`
- `0x4899 -> AUX pin index 7`

So the iMac routes remain:

- paired DP-side tile:
  - ROM object `0x3113`
  - descriptor selector `0x4871`
  - Windows AUX pin index `2`
  - Linux transmitter expectation `TRANSMITTER_UNIPHY_D`
- primary eDP tile:
  - ROM object `0x3114`
  - descriptor selector `0x4875`
  - Windows AUX pin index `3`
  - Linux transmitter expectation `TRANSMITTER_UNIPHY_C`

The key RE correction is that Windows does not consider the route live merely because a display-map entry or connector object exists.

`ConstructDualPinAuxTransactionHandle` invalidates the handle unless both lower endpoint opens succeed:

- endpoint type `3` for the resolved pin index
- endpoint type `4` for the same resolved pin index

That means Linux can easily have a preserved `DP-1` connector and still fail DPCD writes if its lower DDC/AUX service is stale, wrong, or not routed to the equivalent pin/encoder backend.

#### Kernel implication from this pass

The next Linux patch should not yet force a guessed hardware route.

The safe, high-value change is route instrumentation at every `IMAC5K:` checkpoint:

- connector/link id and signal type
- internal-display flag
- `ddc_hw_inst`
- `ddc_pin` line/channel/hw-supported fields
- DDC transaction type and `aux_mode`
- HPD source and IRQ sources
- link encoder connector id, encoder id, transmitter, preferred engine
- current link state/rate/lane count

The expected successful secondary route should look like:

```text
connector = DP-1
link_id.id = CONNECTOR_ID_DISPLAY_PORT / 19
signal = SIGNAL_TYPE_DISPLAY_PORT / 32
ddc/aux route = Windows AUX pin index 2 equivalent
link encoder transmitter = TRANSMITTER_UNIPHY_D
```

If the next test still shows `0x310 status=-1`, the new route logs should tell us which layer is wrong:

- no `ddc_pin`
- wrong DDC/AUX channel
- `aux_mode=0`
- wrong transmitter
- missing/stale link encoder
- HPD not ready
- link object exists but no usable lower AUX backend

This keeps the next Linux test diagnostic rather than speculative.

#### IDA comments added from this pass

- `0x141695B0A`: selector `0x4871` maps to AUX pin index `2`
- `0x141695B17`: selector `0x4875` maps to AUX pin index `3`
- `0x1415B0D0E`: per-object AUX handle resolves selector/source-mask to concrete pin index
- `0x1415B0D4A`: opens lower endpoint type `3`
- `0x1415B0DB7`: opens lower endpoint type `4`
- `0x1415511A1`: DPCD AUX interface obtains object-specific lower AUX handle
- `0x141467BD4`: property6 bit4 suppresses the AUX factory entirely

### 2026-04-30 Linux patch update: route-identity logging before more 5K forcing

After the lower AUX handle RE pass, I updated the Linux branch to log the Windows-equivalent route fields rather than force a guessed route.

Files touched:

- `drivers/gpu/drm/amd/display/amdgpu_dm/amdgpu_dm.c`

New helper:

- `amdgpu_dm_log_link_route`

It prints three `IMAC5K:` lines for each relevant route:

1. connector/link identity:
   - connector name
   - `link_index`
   - `link_id` type/enum/id
   - signal type
   - connection type
   - endpoint type
   - internal-display flag
   - local sink pointer/signal/EDID length/name
2. lower DDC/AUX identity:
   - `ddc`
   - `ddc_pin`
   - `ddc_hw_inst`
   - `dal_ddc_get_line(ddc_pin)`
   - `ddc_pin->hw_info.ddc_channel`
   - DDC transaction type
   - `aux_mode`
   - HPD cached/hardware state
   - HPD source and IRQ sources
3. link encoder identity:
   - link encoder pointer
   - link encoder connector id
   - encoder id
   - transmitter
   - whether transmitter is `TRANSMITTER_UNIPHY_D`
   - preferred engine / `eng_id`
   - `link_enc_hw_inst`
   - link active/state-valid/rate/lane count

The helper is called from:

- `amdgpu_dm_dump_links_and_sinks`
  - uses `drm_info` only when the iMac5K quirk is active
  - otherwise preserves the old `drm_dbg_kms` behavior
- `amdgpu_dm_imac5k_log_connector`
- `amdgpu_dm_imac5k_program_secondary_source_dpcd`
  - immediately before the HPD/AUX wait and source-DPCD write

This should answer the next diagnostic question in one boot:

```text
Does Linux DP-1 really have the Windows-equivalent 0x3113 lower route?
```

The expected good secondary route is still:

- `DP-1`
- `CONNECTOR_ID_DISPLAY_PORT` / `19`
- `SIGNAL_TYPE_DISPLAY_PORT` / `32`
- DDC/AUX route matching Windows AUX pin index `2`
- `TRANSMITTER_UNIPHY_D`
- `aux_mode=1`
- non-null `ddc_pin`

If `0x310` still fails, these logs should show whether the problem is:

- missing DDC pin
- wrong DDC/AUX line/channel
- `aux_mode=0`
- wrong transmitter
- missing link encoder
- HPD not actually ready
- link object alive but lower AUX backend not alive

### 2026-04-30 Windows RE pass: endpoint type 3/4 are DDC data/clock, not new mystery display opcodes

This was the targeted pass requested before doing more Linux changes.

The earlier pass established that Windows only keeps the object-specific DPCD/AUX handle alive if `ConstructDualPinAuxTransactionHandle` can create both lower endpoints:

- endpoint type `3`
- endpoint type `4`
- both for the same resolved DCE11 AUX/DDC pin index

This pass descended into those endpoints and correlated their constants with the public Linux register headers.

The important correction is:

- endpoint type `3` = DDC DATA GPIO endpoint
- endpoint type `4` = DDC CLOCK GPIO endpoint
- both are required for a valid AUX/DDC transaction backend

The selector values we found earlier are not arbitrary opaque values. They are literal DCE11 GPIO/DDC register addresses:

```text
0x4871 = DC_GPIO_DDC3_A
0x4875 = DC_GPIO_DDC4_A
```

The Linux headers confirm the same mapping:

- `mmDC_GPIO_DDC3_MASK = 0x4870`
- `mmDC_GPIO_DDC3_A    = 0x4871`
- `mmDC_GPIO_DDC3_EN   = 0x4872`
- `mmDC_GPIO_DDC3_Y    = 0x4873`
- `mmDC_GPIO_DDC4_MASK = 0x4874`
- `mmDC_GPIO_DDC4_A    = 0x4875`
- `mmDC_GPIO_DDC4_EN   = 0x4876`
- `mmDC_GPIO_DDC4_Y    = 0x4877`
- `mmDC_I2C_DDC3_SETUP = 0x16E3`
- `mmDC_I2C_DDC4_SETUP = 0x16E5`

So the route mapping is now stronger:

- paired DP-side tile:
  - ROM object `0x3113`
  - descriptor selector `0x4871`
  - Windows AUX pin index `2`
  - DCE11 register group `DC_GPIO_DDC3_*`
  - Linux DDC line `GPIO_DDC_LINE_DDC3`
  - Linux `ddc_pin->hw_info.ddc_channel` numeric `2`
    - this is the BIOS `i2c_line` value, not `CHANNEL_ID_DDC3`
  - Linux transmitter expectation `TRANSMITTER_UNIPHY_D`
- primary eDP tile:
  - ROM object `0x3114`
  - descriptor selector `0x4875`
  - Windows AUX pin index `3`
  - DCE11 register group `DC_GPIO_DDC4_*`
  - Linux DDC line `GPIO_DDC_LINE_DDC4`
  - Linux `ddc_pin->hw_info.ddc_channel` numeric `3`
    - this is the BIOS `i2c_line` value, not `CHANNEL_ID_DDC4`
  - Linux transmitter expectation `TRANSMITTER_UNIPHY_C`

For pin index `2`, Windows constructs:

- type `3` / DATA:
  - registers `0x4870..0x4873`
  - data bit mask `0x100`
  - pull-down mask `0x1000`
  - receive mask `0x4000`
  - AUX pad mode mask `0x10000`
  - setup register `0x16E3`
- type `4` / CLOCK:
  - registers `0x4870..0x4873`
  - clock bit mask `0x1`
  - pull-down mask `0x10`
  - receive mask `0x40`
  - AUX pad mode mask `0x10000`
  - setup register `0x16E3`

For pin index `3`, the same data/clock split applies over `0x4874..0x4877` / setup `0x16E5`.

The low-level register helpers are also now identified:

- `DceRegisterReadViaGpuService`
  - packet length `64`
  - command/class field `19`
  - operation `1`
  - register index in packet dword `3`
  - returned value in packet dword `6`
- `DceRegisterWriteViaGpuService`
  - packet length `64`
  - command/class field `19`
  - operation `2`
  - register index in packet dword `3`
  - value in packet dword `6`

This is not the old `1265` confusion. `1265` decimal is DPCD address `0x4F1`; this pass is about the lower GPIO/DDC register backend needed before any DPCD transaction to that sink can work.

#### Linux implication from this pass

The next Linux validation should expect the secondary `DP-1` route to say:

```text
dal_ddc_get_line(ddc_pin) = GPIO_DDC_LINE_DDC3 / numeric 2
ddc_pin->hw_info.ddc_channel = BIOS i2c_line / numeric 2
link encoder transmitter = TRANSMITTER_UNIPHY_D
aux_mode = 1
```

The primary/eDP route should be:

```text
dal_ddc_get_line(ddc_pin) = GPIO_DDC_LINE_DDC4 / numeric 3
ddc_pin->hw_info.ddc_channel = BIOS i2c_line / numeric 3
link encoder transmitter = TRANSMITTER_UNIPHY_C
aux_mode = 1
```

If Linux has the preserved `DP-1` object but its DDC line is not `DDC3`, or its transmitter is not `UNIPHY_D`, then the failure is not primarily GNOME/KDE/tile presentation. It is a lower route mismatch. In that case, forcing a 5K mode in userspace will keep producing half-correct results because DPCD/source-DPCD writes are going to the wrong backend or no backend.

If Linux does show `DDC3 + UNIPHY_D + aux_mode=1` for `DP-1` and DPCD writes still fail, the next suspect becomes the exact timing/open-mode sequence around the DATA/CLOCK GPIO endpoints and AUX pad mode, not object discovery.

#### IDA names/comments added from this pass

Renamed:

- `AuxEndpointFactoryCreateProxyForTypePin`
- `ConstructAuxEndpointProxy`
- `AuxEndpointProxyOpenLowerEndpoint`
- `AuxEndpointFactoryOpenLowerByTypePin`
- `GetOrCreateDdcDataEndpointByPinIndex`
- `GetOrCreateDdcClockEndpointByPinIndex`
- `CreateDce11DdcDataEndpointObject`
- `CreateDce11DdcClockEndpointObject`
- `ConstructDce11DdcGpioEndpoint`
- `Dce11GpioMaskedRegisterWrite`
- `DceRegisterReadViaGpuService`
- `DceRegisterWriteViaGpuService`
- `Dce11DdcGpioEndpointApplyOpenMode`
- `Dce11DdcDataEndpointProgramAuxTransaction`

Comments added or refined at:

- `0x1415B0D4A`
- `0x1415B0DB7`
- `0x141696623`
- `0x141696C97`
- `0x1416966F1`
- `0x141696D65`
- `0x141B29F20`
- `0x141B29D80`
- `0x1416D02E0`
- `0x1416D0540`

### 2026-04-30 Linux patch update: Windows-route verifier before secondary DPCD writes

Based on the DDC DATA/CLOCK endpoint RE pass, I added a targeted route verifier to the Linux branch.

File touched:

- `drivers/gpu/drm/amd/display/amdgpu_dm/amdgpu_dm.c`

New verifier behavior:

- decodes the Windows-equivalent route in dmesg as `IMAC5K: ... Windows-route ...`
- logs the expected route and actual route side by side
- checks the secondary route before writing source-DPCD `0x310`
- checks the secondary route again before writing panel latch DPCD `0x4F1`
- skips those writes if the route is not the Windows-equivalent backend
- prints explicit "failed after Windows-route ok" lines if the route is correct but the DPCD write still fails

The expected secondary route is now hard-coded from the Windows RE:

```text
object id 0x3113
selector 0x4871
Windows AUX pin 2
Linux DDC line GPIO_DDC_LINE_DDC3 / numeric 2
Linux hw_info.ddc_channel BIOS i2c_line / numeric 2
transmitter TRANSMITTER_UNIPHY_D
aux_mode = 1
```

The expected primary route is logged for comparison:

```text
object id 0x3114
selector 0x4875
Windows AUX pin 3
Linux DDC line GPIO_DDC_LINE_DDC4 / numeric 3
Linux hw_info.ddc_channel BIOS i2c_line / numeric 3
transmitter TRANSMITTER_UNIPHY_C
aux_mode = 1
```

### 2026-04-30 kernel_obs_6_gnome result: route was correct, verifier was wrong

The `kernel_obs_6_gnome` logs corrected one important assumption in the Linux verifier.

Observed early route state:

- `DP-1` / paired secondary tile:
  - `ddc_line=2`
  - `ddc_channel=2`
  - `aux_mode=1`
  - transmitter `3` / `TRANSMITTER_UNIPHY_D`
  - live sink and HPD were ready during the early detect/pretrain window
- `eDP-1` / primary tile:
  - `ddc_line=3`
  - `ddc_channel=3`
  - `aux_mode=1`
  - transmitter `2` / `TRANSMITTER_UNIPHY_C`

This is the Windows-equivalent route:

- `0x3113` / selector `0x4871` maps to DDC3 / numeric `2`
- `0x3114` / selector `0x4875` maps to DDC4 / numeric `3`

The apparent `Windows-route mismatch` was caused by our verifier treating `ddc_pin->hw_info.ddc_channel` as `enum channel_id`. In Linux it is assigned from `i2c_info.i2c_line`, so it is zero-based and matches `enum gpio_ddc_line`.

Kernel patch correction:

- changed expected `ddc_channel` for `DP-1` from `CHANNEL_ID_DDC3` / `3` to BIOS/GPIO line `2`
- changed expected `ddc_channel` for `eDP-1` from `CHANNEL_ID_DDC4` / `4` to BIOS/GPIO line `3`
- renamed the log field to `expected_hw_channel` / `actual_hw_channel`

What the run still tells us:

- the preserved `DP-1` lower route is probably correct
- source-DPCD `0x310` was not tested in that boot, because our verifier skipped it before the write
- later GNOME commits kept two `2560x2880` tile streams alive, but `DP-1` had already lost live sink/HPD by then
- the next useful kernel boot should show whether early `source-DPCD` succeeds now that the good route is no longer falsely rejected

### 2026-04-30 kernel_obs_7_gnome result: early source-DPCD works, later link training loses AUX

The `kernel_obs_7_gnome` run validates the corrected route model.

Early secondary `DP-1` state:

```text
detect_live_pretrain secondary Windows-route ok
expected_obj=0x3113
selector=0x4871
win_aux_pin=2
actual_ddc=GPIO_DDC_LINE_DDC3/2
actual_hw_channel=GPIO_DDC_LINE_DDC3/2
actual_tx=TRANSMITTER_UNIPHY_D/3
aux_mode=1
```

Immediately after that, source-DPCD succeeds on the live secondary AUX path:

```text
detect_live_pretrain secondary source-DPCD DP-1
live_sink=1
0x111=00 status_111=1
0x310=04 1d 03 status=1
programmed=1
```

This is an important milestone:

- Linux does have the Windows-equivalent lower route for `0x3113`
- Linux can write source-DPCD `0x310` on that route while the physical secondary sink is still live
- the previous `0x310` failure was caused by our bad route gate, not by an inherently missing AUX backend

The remaining failure is later:

```text
preserving secondary tiled head on DP-1 using emulated sink after the physical sink disappeared
detect_preserve_secondary ... type=0 ... local_sink=0 ... hpd=0/0
commit_streams_pretrain ... AUX/HPD not ready
dpcd_set_link_settings ... DP_DOWNSPREAD_CTRL failed
dpcd_set_link_settings ... DP_LANE_COUNT_SET failed
dpcd_set_link_settings ... DP_LINK_BW_SET failed
enabling link 1 failed: 15
commit_streams_end secondary 0x4F1 latch deferred ... source-DPCD not programmed
```

There is also a small Linux bookkeeping issue:

- the `commit_streams_pretrain` call currently uses `force=true`
- that clears `imac5k_source_dpcd_programmed` before trying to reprogram
- by that time HPD/AUX is already gone, so the reprogram attempt is skipped and the earlier successful `0x310` state is forgotten

But fixing only that bookkeeping is probably not enough. The larger failure is that the physical secondary `DP-1` sink/AUX path disappears before normal link training. The emulated sink keeps the DRM connector/tile alive, but not the lower live AUX/HPD state that link training needs.

Next kernel directions from this run:

1. Keep the successful early `0x310` state from being cleared by a failed forced reprogram attempt.
2. Consider a tightly gated detect-time `DPCD 0x4F1 = 1` immediately after successful early source-DPCD while `DP-1` still has `live_sink=1` and HPD ready.
3. Continue investigating how to preserve or refresh the physical `0x3113` AUX/HPD backend through the first two-stream commit, because later link training is failing ordinary DP DPCD writes after the sink disappears.

This result shifts confidence away from "wrong Linux route" and toward "right route, but Linux lets the secondary physical sink/backend disappear before stream enable."

### 2026-04-30 Windows RE pass after kernel_obs_7: use the live detect window, not a late stale sink

After `kernel_obs_7_gnome`, the RE question changed:

- Linux now proves the `0x3113` / `0x4871` / DDC3 / UNIPHY_D route is correct.
- Linux now proves source-DPCD `0x310 = 04 1d 03` succeeds while `DP-1` still has a live physical sink and HPD.
- Linux then loses the physical `DP-1` sink before two-stream link training.
- Later training fails ordinary DPCD writes because the route is now logical/emulated rather than physically live.

The targeted Windows pass confirms that Windows has a detect/status-time path that can act in exactly that early live window.

Relevant IDA functions:

- `DetectDisplayMaybeAssert4F1ForCapableSink` = `0x1414B2AF0`
- `WriteSinkDpcd4F1Payload1Retry` = `0x1414B03F0`
- `DetectDisplayEarlyExitForApple5kCaps` = `0x1414B31D0`
- `PrepareLinkTrainingStateAndArmEdp128` = `0x141BB7090`
- `PerformLinkTrainingWithRetries` = `0x141B8D860`
- `QueryConnectorReadyAndArmEdp128` = `0x141B704F0`

The important detect/status sequence is:

```text
DetectDisplayAndApplyApple5kOverrides
  -> ProbeDisplayStatusMaybeAssert4F1
     -> DetectDisplayMaybeAssert4F1ForCapableSink
        -> checks pin descriptor byte +6, bit 2 / tag 0x3A
        -> WriteSinkDpcd4F1Payload1Retry
           -> FindDisplayEntryByObjectId(object id)
           -> use display_entry +0x18 DPCD/AUX interface
           -> write DPCD 0x4F1 = 1
           -> retry once after 10 ms if needed
        -> wait 100 ms
        -> re-read display status
```

This is now a strong Linux analogue:

- in `kernel_obs_7`, Linux successfully writes `0x310` during `detect_live_pretrain`
- at that exact point, `DP-1` still has `live_sink=1` and HPD ready
- Windows has a path that can send `0x4F1 = 1` during detect/status while the object-specific AUX interface is still live
- Linux currently waits until `commit_streams_end`, by which time `DP-1` has `local_sink=0` and `hpd=0/0`

So the next kernel experiment should not be a late `0x4F1` replay after the sink disappears. The Windows-shaped experiment is:

1. On the marked iMac secondary `DP-1` / `0x3113` path, wait for the same early live AUX window already proven by `detect_live_pretrain`.
2. Write source-DPCD `0x310 = 04 1d 03`.
3. If that succeeds and the Windows-equivalent route is still valid, immediately write sink-private `DPCD 0x4F1 = 1`.
4. Retry the `0x4F1` write once after `10 ms`, matching Windows.
5. Wait `100 ms`, matching Windows' detect/status path.
6. Re-read/log basic status afterward:
   - HPD / live sink
   - `0x205` read only, not poll/write yet unless runtime evidence proves tag `0x40`
   - perhaps a cheap DPCD read such as `0x000` or the same status byte used by Linux link detection

The pass also re-confirmed the training distinction:

- `PrepareLinkTrainingStateAndArmEdp128` is called before each Windows training retry.
- For signal/type `128`, it re-runs the eDP HPD wait and post-HPD settle callbacks.
- For our proven target `0x3113`, Windows/VBIOS classify it as DP `0x13` / signal `32`, not eDP `0x14` / signal `128`.
- Therefore the type-128 readiness path is not the direct fix for `DP-1`; the direct Windows analogue for this run is the detect/status-time `0x4F1` assertion and display-map liveness policy around `0x3113`.

IDA comments added at:

- `0x1414B2D1F`
- `0x1414B03F0`
- `0x1414B332B`
- `0x141B8D9E2`

Current conclusion:

- high confidence: route construction is correct for probe/OCLP boot
- high confidence: early source-DPCD works
- medium/high confidence: Linux should test early `0x4F1 = 1` immediately after successful early `0x310`, not wait for `commit_streams_end`
- medium confidence: if early `0x4F1` keeps HPD/live sink through training, the remaining work shifts back to tiled presentation
- if early `0x4F1` succeeds but the sink still disappears, the next RE target is the `0x3113` display-map/live-record policy rather than DPCD payload contents

What to look for in the next dmesg:

- `Windows-route ok for source-DPCD`
  - then a DPCD failure means the route is right and the remaining problem is timing/open-mode/AUX pad sequencing
- `Windows-route mismatch for source-DPCD`
  - then the next fix should not be compositor/tile policy; it should be lower route construction or selection
- `Windows-route ok for 0x4F1-latch`
  - means the latch write is being attempted only on the proven secondary route
- `source-DPCD write failed after Windows-route ok`
  - means the route matches Windows but the AUX transaction still fails
- `0x4F1 latch write failed after Windows-route ok`
  - means source-DPCD likely happened, but the final panel latch write still did not complete

This patch intentionally does not force DDC3 or rewrite link routing yet. It turns the next kernel boot into a clean fork:

```text
wrong route -> fix Linux route construction/selection
right route but DPCD fails -> investigate timing/open-mode/AUX pad sequencing
right route and DPCD succeeds -> continue with tiled presentation/compositor behavior
```

### 2026-05-01 targeted RE + Linux patch: move 0x4F1 into the proven live window

The follow-up RE pass focused only on reducing kernel iterations for the current near-term target: keep the 5K probe/OCLP state alive through Linux/DRM takeover.

Additional Windows details confirmed:

- `WriteSinkDpcd4F1Payload1Retry` still does only the expected thing:
  - resolve the current display object id
  - find its display-map entry
  - use the per-object DPCD/AUX backend at display-map `+0x18`
  - write `DPCD 0x4F1 = 1`
  - retry once after `10 ms`
- `DetectDisplayMaybeAssert4F1ForCapableSink` then waits `100 ms` and re-reads status.
- `ProbeDisplayStatusMaybeAssert4F1` refreshes/validates the display object's backend against display-map `+0x18/+0x20` before status handling. This reinforces that keeping a logical sink alive is not enough if the lower AUX backend is stale.
- `sub_1414B0F70` first checks adapter/HPD-style connected status, but if the pin descriptor allows it, it falls back to a tiny DPCD read through the lower object. This supports logging a cheap `DPCD 0x000` read after the early Linux latch, not relying only on HPD/local-sink state.

IDA comments were added at:

- `0x1414B0F70`
- `0x1414B24CD`
- `0x1414B2536`
- `0x1414B1043`

Linux patch applied in `linux-imac-5k`:

- factored secondary `0x4F1` writes into a helper
- after successful early `detect_live_pretrain` / `detect_connected_pretrain` / `boot_detect` source-DPCD on the secondary head, immediately writes `DPCD 0x4F1 = 1`
- keeps the Windows retry pattern: one retry after `10 ms`
- keeps the Windows detect/status settle: `100 ms`
- logs post-latch HPD, cached HPD, `local_sink`, link-active/state-valid, `DPCD 0x000`, and `DPCD 0x205`
- preserves a previous successful `source_dpcd_programmed` flag when a later forced source-DPCD retry is skipped because AUX/HPD is gone

Expected useful dmesg fork:

- `detect_live_pretrain secondary 0x4F1 latch ... status=0 asserted=1` followed by HPD/local sink staying alive: early latch likely solved the backend-loss phase.
- `status=0 asserted=1` but HPD/local sink still disappears: next target is Windows' `0x3113` display-map/live-record preservation policy, not the DPCD payload.
- `0x4F1 latch write failed after Windows-route ok`: source-DPCD succeeds but the panel-latch write has stricter timing/open-mode requirements.
- `dpcd000` read succeeds even while HPD says disconnected: Linux may need a Windows-like DPCD-reachable secondary-internal policy instead of treating HPD loss as final disconnect.

### 2026-05-01 kernel_obs_8_gnome result: early 0x4F1 succeeds, but Linux still drops the real secondary sink

Capture set:

- `C:\proj\reverse\WT6A_INF\kernel_obs_8_gnome`
- kernel: `7.0.1-1-imac-5k-g102100b1c509`

The new patch definitely ran. The added `previously_programmed`, `dpcd000`, and `dpcd205` fields are present in dmesg.

Important early live-window result:

```text
detect_live_pretrain secondary source-DPCD DP-1
  live_sink=1
  0x310=04 1d 03
  status=1
  programmed=1

detect_live_pretrain secondary 0x4F1 latch DP-1
  payload=1
  status=1
  asserted=1
  settle_ms=100
  live_sink=1
  hpd_hw=1/1
  dpcd000=12 status000=1
  dpcd205=00 status205=1
```

This proves:

- the Windows-equivalent `0x3113` / DDC3 / UNIPHY_D secondary AUX route is correct
- Linux can write source-DPCD `0x310` while the physical secondary sink is live
- Linux can also write sink-private `DPCD 0x4F1 = 1` in that same early live window
- after the `100 ms` Windows-style settle, the sink is still DPCD-readable (`DPCD 0x000 = 0x12`)

But the later failure remains:

```text
preserving secondary tiled head on DP-1 using emulated sink after the physical sink disappeared
detect_preserve_secondary ... type=0 local_sink=0 hpd=0/0
commit_streams_pretrain ... live_sink=0 hpd_ready=0
DP_DOWNSPREAD_CTRL failed
DP_LANE_COUNT_SET failed
DP_LINK_BW_SET failed
enabling link 1 failed: 15
```

So `0x4F1` is not sufficient to keep Linux's secondary physical backend alive. The immediate failure is Linux's disconnect/update path:

- the physical `DP-1` sink exists and is writable during detect
- then `link_detection` clears `link->local_sink` and sets `link->type = dc_connection_none`
- our preservation path keeps userspace-visible topology by switching `aconnector->dc_sink` to `dc_em_sink`
- but link training still uses the underlying `dc_link`, which now has no `local_sink`, HPD low, and no connected link type

Userspace/DRM state after GNOME:

- two connected connectors remain: `eDP-1` and `DP-1`
- both expose TILE blobs in the same group:
  - `eDP-1`: `1:1:2:1:0:0:2560:2880`
  - `DP-1`: `1:1:2:1:1:0:2560:2880`
- two CRTCs remain active at `2560x2880`
- GNOME allocates two independent `2560x2880` framebuffers
- fbcon still has a `5120x2880` framebuffer object, but GNOME is not using it

The next kernel attempt should therefore stop adding DPCD payloads and instead emulate Windows' live-record/display-map preservation outcome:

1. In the iMac5K secondary-preserve path, do not replace the real physical `DP-1` sink with only `dc_em_sink` if we still have the previous physical `aconnector->dc_sink`.
2. Restore or retain `dc_link->local_sink` from that previous physical sink for the `0x3113` secondary head.
3. Restore the link type to connected/single for that internal secondary route instead of leaving `link->type = dc_connection_none`.
4. Keep the emulated sink only as a fallback for userspace EDID/topology if the physical sink object is truly gone.
5. Add logs proving whether `commit_streams_pretrain` now sees `local_sink != NULL`, `type=1`, and whether ordinary training DPCD writes stop failing.

This maps closely to the Windows RE model:

- Windows does not merely keep a logical tiled connector alive
- it keeps the display-map entry's object-specific lower AUX backend live enough for stream construction/training
- Linux currently keeps the logical connector alive but lets the lower `dc_link` live state disappear

### 2026-05-01 Linux patch update: remove iMac5K emulated-sink preservation path

Follow-up decision after `kernel_obs_8_gnome`:

- Windows has logical display-map/tile policy, but no evidence of a Linux-style `dc_em_sink` replacement before training.
- The successful early `0x310` and `0x4F1` writes prove the real `0x3113` secondary AUX backend exists and is usable.
- Therefore the next Linux attempt should stop preserving the secondary head by swapping to an emulated sink.

Patch direction applied:

- removed the iMac5K-specific emulated sink seeding path from the active flow
- removed the now-unused iMac5K EDID clone/tile-patch helper used only for that emulated sink
- changed secondary preservation to require the previous real `aconnector->dc_sink`
- when HPD later disappears, restore that real sink into `dc_link->local_sink`
- restore `link->type = dc_connection_single`
- restore `dpcd_sink_count = 1` if Linux cleared it
- keep the existing connector EDID/topology rather than fabricating a remote sink
- update AUX-ready wait so a successful `DPCD 0x000` read can count as liveness for this internal secondary path, matching the Windows status fallback more closely than pure HPD

Expected useful dmesg for the next boot:

```text
preserving secondary tiled head on DP-1 using real previous sink after HPD disappeared
restored secondary real link backend ... old_type=0 new_type=1 ... local_sink=<non-null>
commit_streams_pretrain secondary AUX-ready wait ... live_sink=1 ... aux_dpcd_ready=1 dpcd000=12
```

If those appear and ordinary `DP_DOWNSPREAD_CTRL` / `DP_LANE_COUNT_SET` / `DP_LINK_BW_SET` writes still fail, the next issue is not emulation or HPD policy; it is likely a lower link-output/power/training sequencing difference from Windows.

### 2026-05-01 targeted Windows RE pass: mode-solution grouping does not expose a simple tile-X literal

Question for this pass:

- do we fully understand how Windows presents the two physical halves as one 5K display?
- can the Windows driver statically reveal the exact left/right source-position or shift used for the secondary tile?

Result:

- We do **not** yet have a complete end-to-end Windows model, but the important pieces are now well separated:
  - sink/link bring-up: `0x3113` / DDC3 / UNIPHY_D, source-DPCD `0x310`, sink-private `DPCD 0x4F1 = 1`
  - mode policy: `DalEnable5kTiledMode`, derived `2560x2880` half-tile mode injection, and `5120x2880@60` pipe-split marking
  - presentation/placement: still only partially recovered statically
- Rechecked:
  - `ModeManagerAddDerivedModesIncluding5kTile` = `0x141A3C4E0`
  - `MarkTimingPipeSplitForHighResAnd5k60` = `0x1415D5380`
  - `ModeListContainsPipeSplitTiming` = `0x1415CAF60`
  - `CreateModeSolutionWith5KPipeSplitOptionFlags` = `0x141A3C920`
  - `ConstructModeSolutionWithPipeSplitFlags` = `0x141ACF560`
  - grouped view-solution builder now named `CreateDisplayGroupViewSolution` = `0x141A3BBC0`

Confirmed again:

- Windows can synthesize `2560x2880` as a physical half-tile timing from a `1920x2160` source timing only when:
  - mode-manager flag bit `7` is set
  - source timing record flag bit `7` is set
  - base timing is `1920x2160`
  - mode-manager flag bit `5` is set
  - `DalEnable5kTiledMode` / object byte `+0x80` is true
- Windows separately marks `5120x2880@60` / high-resolution timings as pipe-split by setting timing-record low bits in record field `+0x24`.
- `CreateModeSolutionWith5KPipeSplitOptionFlags` passes a bit derived from `Dal5K60PipeSplit` into `ConstructModeSolutionWithPipeSplitFlags`, which stores it at solution object `+0xB8`.

New/clarified negative finding:

- The grouped view-solution builder has an optional vtable call that passes `ModeManager + 0x34` divided by `2`.
- This looked suspiciously like a possible "half width" / tile offset path at first glance.
- But `ModeManager + 0x34` is initialized from constructor input `+0x24` and defaults to `6` if zero.
- Therefore this half-value is almost certainly a grouped-solution policy/count/threshold value, **not** the visible `2560` pixel right-tile source X offset.
- The setter now named `SetViewSolutionHalfPolicyValue` = `0x141AC99B0` only stores that integer at grouped-solution object `+0x3F4`.

Interpretation for the current Linux/GNOME shift problem:

- Static Windows RE still argues that Windows exposes one logical 5K display backed by split/pipe/tile machinery.
- But this pass did not find a simple Windows-side literal equivalent to "right tile source X = 2560".
- The best concrete placement evidence remains the Linux pre-GUI KMS state:
  - one shared `5120x2880` framebuffer
  - left tile scans source X `0`
  - right tile scans source X `2560`
- The bad GNOME state still shows two independent `2560x2880` framebuffers with source origin `0,0` on both halves.
- So the immediate presentation target for Linux remains:
  - preserve or recreate two live physical `2560x2880` streams
  - expose the tiled pair coherently enough that userspace allocates one logical `5120x2880` monitor/framebuffer, or kernel-side scanout setup maps the second tile to source X `2560`

IDA annotations added:

- `CreateDisplayGroupViewSolution` = `0x141A3BBC0`
- `ValidateDisplayGroupAgainstViewSolution` = `0x141A3B800`
- `SetViewSolutionHalfPolicyValue` = `0x141AC99B0`
- `ConstructViewSolutionClass0` = `0x141ACD520`
- `ConstructViewSolutionClass1` = `0x141ACD290`
- `ConstructViewSolutionClass2EnabledState` = `0x141ACD1B0`
- `ConstructViewSolutionClass3WithSlots` = `0x141ACD390`
- `ConstructViewSolutionClass4` = `0x141ACD320`
- `ConstructViewSolutionClass5EnabledState` = `0x141ACD460`

Practical conclusion:

- More broad RE is unlikely to magically produce a one-line recipe.
- One more *very targeted* RE branch could still help if we chase the later DCE/DCN plane/viewport programming path that consumes the mode solution.
- However, for the next Linux test, the highest-confidence fix remains the current Windows-like live real-sink preservation attempt, not another opcode or emulated-sink change.

### 2026-05-01 targeted Windows RE pass: viewport consumer confirms split-pipe placement model

Question for this narrow pass:

- after the mode-solution object is built, who consumes the plane/view rectangles?
- does Windows carry a simple static equivalent of "right tile source X = 2560"?
- or is the visible tile placement derived later from pipe/plane topology?

Main result:

- The later viewport consumer is now identified as `ResourceBuildScalingParamsAndViewport` = `0x141BABA80`.
- This function logs the final-ish DC resource state with:
  - viewport height/width/x/y
  - recout height/width/x/y
  - HACTIVE/VACTIVE
  - plane `src_rect`, `dst_rect`, and `clip_rect`
- It consumes the pipe-context split links at pipe entry offsets:
  - `+0x2F8`
  - `+0x300`
  - `+0x308`
  - `+0x310`
- It is downstream from OS plane conversion and much closer to real hardware programming than the ModeManager/view-solution layer.

Viewport computation:

- `ComputeViewportForSplitPipe` = `0x141BA96A0`
- `GetSplitPipeCountAndIndex` = `0x141BA9CC0`
- `CountSecondarySplitPipeChain` = `0x141BA9DB0`
- `CountSamePlaneSplitSiblings` = `0x141BA9E40`

`ComputeViewportForSplitPipe` does not read a visible 5K-specific tile-X literal. Instead it:

- starts from plane/stream source and clip rectangles
- asks `GetSplitPipeCountAndIndex` for the total split count and this pipe's split index
- localizes viewport `x`, `width`, and `height` for the current pipe
- handles both same-plane sibling split chains and the secondary split-chain path

Split topology construction:

- `AttachPlaneStateAndCloneSplitPipeLinks` = `0x141BADE20`
- `FindHeadPipeForStreamState` = `0x141BA8960`
- `RecyclePipeContextForSplitClone` = `0x141BA8610`
- `BuildValidateContextAndScalingParams` = `0x141B754A0`

The clone/relink path copies plane state onto the stream and preserves the split-pipe graph:

- clone inherits the previous split-context link at `+0x2F8`
- the old pipe gets the back-link at `+0x300`
- the orthogonal split chain is propagated through `+0x308` / `+0x310`
- after this topology exists, `ResourceBuildScalingParamsAndViewport` recomputes each pipe's viewport

Plane conversion / non-finding:

- `ConvertOsPlaneConfigToInternalPlaneConfig` = `0x14144D7F0`
- `ApplyPlaneViewportSourceWaDuringSetup` = `0x1414F6740`
- `ClampViewportDimensionForOsWa` = `0x14144D730`

These functions copy OS plane source/destination/clip rectangles and contain an OS workaround that clamps viewport **dimensions** to surface limits/alignment. This path did not reveal a 5K-specific `x = 2560` placement constant.

Interpretation for Linux:

- The Windows model appears to be "build a valid split-pipe topology, then derive per-pipe viewport placement from split count/index", not "store a magic right-tile source-X value in the mode solution".
- This matches the good Linux pre-GUI state conceptually: two live scanout halves backed by one logical `5120x2880` framebuffer.
- It also explains why the bad GNOME state is wrong: GNOME ends up with two independent `2560x2880` framebuffers whose source origins are both `0,0`, so the right half is no longer part of a single split-pipe/logical-framebuffer presentation.

Kernel implication:

- The next Linux work should avoid a broad hard-coded tile-position hack as the primary model.
- The closest Windows-like target is:
  - keep the two physical half-panel links/sinks alive
  - preserve or recreate the DC pipe split/ODM relationship between the halves
  - log Linux DC pipe topology, plane `src_rect`, `dst_rect`, and stream rectangles at the first userspace modeset
  - only use a forced right-tile source offset as a tightly gated fallback if the Linux DC/KMS representation cannot express the split topology cleanly

IDA annotations added:

- `ResourceBuildScalingParamsAndViewport` = `0x141BABA80`
- `ComputeViewportForSplitPipe` = `0x141BA96A0`
- `GetSplitPipeCountAndIndex` = `0x141BA9CC0`
- `CountSecondarySplitPipeChain` = `0x141BA9DB0`
- `CountSamePlaneSplitSiblings` = `0x141BA9E40`
- `AttachPlaneStateAndCloneSplitPipeLinks` = `0x141BADE20`
- `FindHeadPipeForStreamState` = `0x141BA8960`
- `RecyclePipeContextForSplitClone` = `0x141BA8610`
- `BuildValidateContextAndScalingParams` = `0x141B754A0`
- `ConvertOsPlaneConfigToInternalPlaneConfig` = `0x14144D7F0`
- `ApplyPlaneViewportSourceWaDuringSetup` = `0x1414F6740`
- `ClampViewportDimensionForOsWa` = `0x14144D730`

### 2026-05-01 Linux patch update: remove iMac5K fallbacks and log Windows-like viewport topology

Decision after the viewport-consumer RE pass:

- Do not keep Linux-only iMac5K fallbacks that hide the real DC state.
- Keep the model as close as possible to Windows:
  - real secondary DP sink/link only
  - canonical tile order: primary/eDP tile `0`, secondary/DP tile `1`
  - one shared tile group exposed to DRM userspace
  - pipe/viewport placement must come from DC split/ODM topology, not from an arbitrary right-tile X hack

Kernel changes made in `linux-imac-5k`:

- Removed the earlier hard-swap tile experiment.
  - `IMAC5K_PRIMARY_TILE_X = 0`
  - `IMAC5K_SECONDARY_TILE_X = 1`
- Added explicit secondary TILE property application from the primary tile group.
  - This makes the secondary DP half join the same userspace tile group as the primary eDP half.
  - This is the Linux analogue of Windows presenting one logical 5K display group.
- Removed the iMac5K stream-drop suppression path.
  - A proposed one-stream/tile-drop userspace state is now logged and allowed through.
  - This avoids masking the real failure with a Linux-only "keep previous state" workaround.
- Removed the dead connector-detect path that could report connection through a preserved emulated sink.
- Removed the iMac5K-specific `imac5k_em_sink_seeded` marker and stale comments that described the old emulated-sink experiment.
- Added DC pipe/viewport logging to mirror the Windows `resource_build_scaling_params` evidence:
  - per stream: timing, source rect, destination rect, link, signal, plane count, OTG/encoder
  - per pipe: `top_pipe`, `bottom_pipe`, `prev_odm_pipe`, `next_odm_pipe`
  - per pipe: scaler viewport, recout, hactive/vactive
  - per pipe: plane source/destination/clip rects

Expected useful dmesg shape:

```text
IMAC5K: applied secondary TILE property on DP-1 from primary eDP-1 loc=1,0 ...
IMAC5K: observed tiled stream drop attempt ... allowing commit, no emulated/suppressed iMac5K path
IMAC5K: <tag> pipe[...] ... top=... bottom=... prev_odm=... next_odm=... viewport=... recout=... plane_src=... plane_dst=...
```

How to read the next boot:

- If the good probe/OCLP pre-GUI state shows a pipe relationship where one half has `next_odm`/`prev_odm` or `top`/`bottom` linkage and GNOME later loses it, the next fix is to preserve/recreate that split relationship in DC.
- If both halves are already independent pipes with no split/ODM relation even in the good state, then Windows' abstraction does not map directly onto Linux ODM; the fix likely moves up to DRM tiled-monitor presentation and shared framebuffer construction.
- If the secondary tile loses `local_sink` or AUX before stream commit, we are still failing earlier at link/sink preservation and should not chase compositor behavior yet.

### 2026-05-01 next Windows RE targets before another kernel attempt

Reason for another RE pass:

- The latest static gap is no longer `0x4F1`, source-DPCD, or the lower AUX route.
- Those are already understood well enough for the current Linux patch series.
- The unresolved Windows-to-Linux bridge is:
  - Windows marks/chooses a 5K pipe-split mode solution.
  - Later Windows has a concrete split-pipe graph consumed by `ResourceBuildScalingParamsAndViewport`.
  - We still need the bridge between those two facts.

Target 1: split-pipe graph construction bridge

- Primary function: `AttachPlaneStateAndCloneSplitPipeLinks` = `0x141BADE20`
- Immediate helpers:
  - `FindHeadPipeForStreamState` = `0x141BA8960`
  - `RecyclePipeContextForSplitClone` = `0x141BA8610`
  - `BuildValidateContextAndScalingParams` = `0x141B754A0`
  - `ResourceBuildScalingParamsAndViewport` = `0x141BABA80`
- Known callers of `AttachPlaneStateAndCloneSplitPipeLinks`:
  - `sub_141A11E80`
  - `sub_141AD9000`
  - `sub_141ADA460`
  - `sub_141BA8240`
  - `sub_141BD9DE0`
- Questions:
  - Which caller is used for the normal modeset/validate path, not a side path?
  - What condition decides that a second pipe should be cloned/linked for the same stream/plane?
  - Does that condition consume the mode-solution pipe-split flag stored at solution object `+0xB8`?
  - Do the Windows links map more closely to Linux `top_pipe`/`bottom_pipe` or `prev_odm_pipe`/`next_odm_pipe`?
- Stop condition:
  - Stop when we can name the exact upstream flag/field that causes the pipe graph to be cloned, or when we prove this graph is generic DC resource allocation rather than 5K-specific policy.

Target 2: mode-solution flag to validate-context handoff

- Starting functions:
  - `CreateModeSolutionWith5KPipeSplitOptionFlags` = `0x141A3C920`
  - `ConstructModeSolutionWithPipeSplitFlags` = `0x141ACF560`
  - `CreateDisplayGroupViewSolution` = `0x141A3BBC0`
  - `ValidateDisplayGroupAgainstViewSolution` = `0x141A3B800`
- Known state:
  - `CreateModeSolutionWith5KPipeSplitOptionFlags` passes a bit derived from `Dal5K60PipeSplit`.
  - `ConstructModeSolutionWithPipeSplitFlags` stores it at solution object `+0xB8`.
  - `ModeManager + 0x34 / 2` was checked and is not the tile-X offset.
- Questions:
  - Where is solution `+0xB8` read after construction?
  - Does it flow into validate context, pipe count, stream count, or split-pipe clone logic?
  - Is this flag only eligibility/policy, or does it directly force split-pipe resources?
- Stop condition:
  - Stop when every xref/use of solution `+0xB8` is accounted for, or when it is only copied into later objects and the next consuming object is identified.

Target 3: plane/view rectangle source and one-logical-framebuffer model

- Starting functions:
  - `ValidatePlaneConfigurations` string xref path around `0x141A19DF0`
  - `sub_141A1ABB0`
  - `sub_141A18790`
  - `ConvertOsPlaneConfigToInternalPlaneConfig` = `0x14144D7F0`
  - `ApplyPlaneViewportSourceWaDuringSetup` = `0x1414F6740`
  - `ClampViewportDimensionForOsWa` = `0x14144D730`
  - `SetupPlaneConfigurations` string xref path around `0x141EF2CB0` / `0x141EFF490`
  - `SetViewPorts` string path around `0x141EFF490`
- Known state:
  - The conversion path copies OS plane `src_rect`, `dst_rect`, and `clip_rect`.
  - It only revealed dimension clamping, not a 5K tile-X literal.
- Questions:
  - Does Windows receive one 5120-wide OS plane and split it into two pipe viewports internally?
  - Or does Windows receive two OS plane records that are already grouped by the display manager?
  - Where does the right-half plane get source origin/viewport localization?
- Stop condition:
  - Stop when we can state whether Windows' input to DC is one logical 5120-wide plane or two 2560-wide plane records.

Target 4: display-map/live-record policy only if link state still matters

- Starting facts:
  - `0x3113` / selector `0x4871` / DDC3 / UNIPHY_D is correct.
  - Linux can write source-DPCD `0x310` and sink-private `0x4F1 = 1` while the physical sink is live.
  - Early `0x4F1` is not sufficient by itself; Linux later drops the real secondary sink.
- Questions:
  - Does Windows keep a live display-map record even after a status path that Linux would treat as DP disconnected?
  - Is there another Windows status bit/field beyond HPD that keeps `0x3113` trainable?
- Stop condition:
  - Defer this target unless the next kernel logs still show `local_sink`/AUX loss after real-sink preservation.

Recommended RE order:

1. Start with Target 1, because it is the closest to the latest viewport-consumer finding and most likely to map to Linux DC pipe fields.
2. If Target 1 finds a flag source, immediately pivot to Target 2 to connect it to `Dal5K60PipeSplit` / solution `+0xB8`.
3. If Target 1 shows the split graph is generic and not mode-solution-driven, pivot to Target 3 to understand the OS plane model.
4. Do not spend more time on Target 4 until runtime evidence says the real sink still disappears after the latest Linux patch.

### 2026-05-07 Windows RE pass: split-pipe bridge is validate/resource construction, not plane attach

IDA functions renamed/commented during this pass:

- `WindowsDM_SetPathMode_BuildStreamsAndCommit` = `0x141A22470`
- `CollectActiveDcStreamsFromDisplayObjects` = `0x141ED0DB0`
- `DcCommitStreams_BuildValidateAndCommit` = `0x141B7CE10`
- `DcValidateWithContext_RebuildStreamsAndPlanes` = `0x141BAD500`
- `DcValidateContext_AddStreamViaResourceCallback` = `0x141BADD00`
- `DcValidateFinalize_RebuildViewportsAndResources` = `0x141BAE330`
- `DcBuildScalingParamsAndViewportForAllPipes` = `0x141BA8B20`
- `AttachAllPlanesForStreamFromPlaneSet` = `0x141BA8240`
- `ConstructDcStreamStateFromDisplayObject` = `0x141B64660`
- `ConstructViewSolutionBase` = `0x141ACD0A0`
- `ConstructViewSolutionBaseFlagsAndModeMetadata` = `0x141ACCA30`

Target 1 result:

- `AttachPlaneStateAndCloneSplitPipeLinks` is generic attach/relink code.
- It does not decide that 5K should be split.
- It finds the head pipe for an existing stream, walks existing pipe links, clones/relinks pipes for attached planes, and preserves the already-existing split graph.
- The important pipe links consumed later by `ResourceBuildScalingParamsAndViewport` remain:
  - `+0x2F8` / `+0x300`
  - `+0x308` / `+0x310`
- Therefore the 5K policy decision is upstream of plane attach.

Normal Windows commit bridge found:

- `WindowsDM_SetPathMode_BuildStreamsAndCommit` creates/updates display objects and stream states.
- It calls `CollectActiveDcStreamsFromDisplayObjects`.
- `CollectActiveDcStreamsFromDisplayObjects` walks display objects and appends every non-disabled `display_object + 0x38` stream pointer into the stream array.
- That stream array is passed to `DcCommitStreams_BuildValidateAndCommit`.
- `DcCommitStreams_BuildValidateAndCommit` creates a validate context and calls `DcValidateWithContext_RebuildStreamsAndPlanes`.

Validate/resource path:

- `DcValidateWithContext_RebuildStreamsAndPlanes` diffs old versus new stream sets.
- Removed streams are detached with `sub_141BAC270` and `sub_141BA7500`.
- Added streams go through `DcValidateContext_AddStreamViaResourceCallback`.
- `DcValidateContext_AddStreamViaResourceCallback` stores the stream in the validate context, then calls the resource callback at table offset `+0x78`.
- After stream/resource creation, planes are attached with `AttachAllPlanesForStreamFromPlaneSet` and `AttachPlaneStateAndCloneSplitPipeLinks`.
- Finalization calls `DcValidateFinalize_RebuildViewportsAndResources`.
- `DcValidateFinalize_RebuildViewportsAndResources` calls `DcBuildScalingParamsAndViewportForAllPipes`.
- `DcBuildScalingParamsAndViewportForAllPipes` iterates context pipe slots and calls `ResourceBuildScalingParamsAndViewport` for each active pipe.

Current interpretation:

- Windows appears to choose/group the 5K mode in the display-manager mode/view solution layer.
- It then commits real DC stream objects.
- The concrete split-pipe graph is created by the resource add-stream/build-pipe callback, before plane attach and before viewport derivation.
- The viewport code is a consumer of the split graph, not the owner of the 5K policy.
- This is consistent with the earlier finding that `ComputeViewportForSplitPipe` derives per-pipe viewport placement from split count/index rather than reading a static `x = 2560` literal.

Target 2 partial result:

- `CreateModeSolutionWith5KPipeSplitOptionFlags` still creates the mode solution with bit1 derived from `Dal5K60PipeSplit`.
- `ConstructModeSolutionWithPipeSplitFlags` stores that option flag at mode-solution `+0xB8`.
- A second unrelated `+0xB8` exists in the grouped view-solution class:
  - `ConstructViewSolutionBaseFlagsAndModeMetadata` sets grouped view flags at `view_solution + 0xB8`.
  - This is not the same object as the mode-solution `+0xB8`.
- No direct static reader of the mode-solution `+0xB8` was proven in this pass.
- Best current read: mode-solution `+0xB8` is an eligibility/policy marker in the mode/view solution layer, while the actual pipe graph is materialized later by the DC resource add-stream path.

Useful kernel implication:

- For "keep 5K" on Linux, the closest Windows-equivalent model is not an emulated fake sink.
- It is:
  - keep both real display objects/connectors/streams alive,
  - expose/group them as one logical tiled display to DRM userspace,
  - ensure amdgpu DC builds two real stream/pipe paths before userspace's first atomic commit,
  - let viewport placement come from the split/tile graph rather than hardcoding a right-tile viewport in userspace policy.

Next RE target:

- Continue from the resource callback at validate-context add-stream:
  - `DcValidateContext_AddStreamViaResourceCallback` calls callback table offset `+0x78`.
  - By signature and by comparison with Linux `dc_resource.c`, this callback is very likely the Windows equivalent of `res_pool->funcs->add_stream_to_ctx(dc, new_ctx, stream)`.
  - In Linux, `add_stream_to_ctx` maps pool resources, maps PHY clock resources, optionally adds DSC, and then builds mapped resources. That is exactly the layer where pipe slots and stream resources become concrete.
  - Identify the concrete callback implementation for this ASIC/resource pool.
  - Inside that callback, find where pipe slots are allocated and where `+0x2F8/+0x300/+0x308/+0x310` links first become non-null.
- If callback resolution is too indirect statically, pivot to Target 3 and prove whether DC receives one grouped 5120-wide plane or two 2560-wide plane records.

### Windows DC Resource Callback: DCE112 Add-Stream And Seamless Boot Reuse

This pass continued exactly from `DcValidateContext_AddStreamViaResourceCallback` and resolved the callback chain far enough to compare it against Linux's DCE 11.2 resource path.

IDA functions renamed/commented during this pass:

- `ClassifyDisplayCoreVersionFromAsicId` = `0x141BAB280`
- `CreateDce112ResourcePool_Alloc` = `0x141C0F7D0`
- `Dce112ResourceConstruct` = `0x141C0D120`
- `Dce112AddStreamToCtx_MapBuildResources` = `0x141C0F5E0`
- `ResourceMapPoolResourcesForStream` = `0x141BA6800`
- `ResourceMapPhyClockForStream` = `0x141C0F670`
- `Dce112BuildMappedResourceForStream` = `0x141C0E340`
- `MarkSeamlessBootStreamIfResourcesMatch` = `0x141BA6DF0`
- `IsType128StreamMatchingBootResources` = `0x141B7B830`
- `TryMapSeamlessBootPipeResources` = `0x141BA6E80`
- `MapFirstFreePipeResourcesForStream` = `0x141BA7B50`
- `FindOrCreateSplitClonePipeForPlaneAttach` = `0x141BA8810`
- `FindTailPipeInTopPipeChain` = `0x141BA8910`
- `IsDpOrEdpSignalTypeForBootResourceMatch` = `0x141B7B4E0`
- `EdpLinkRateOptimizationWouldBreakSeamlessBoot` = `0x141B8C4E0`
- `WindowsDM_SetPowerState_ManageDcSeamlessFlags` = `0x141A1F9C0`

Resource-pool version mapping:

- `ClassifyDisplayCoreVersionFromAsicId` maps ASIC family/revision/device id to a Windows DC resource-pool case.
- VI/Polaris-family selection is under family selector `0x82`.
- The relevant case `5/6` constructor is `Dce112ResourceConstruct`.
- `Dce112ResourceConstruct` installs resource function table `off_144B10880`.
- `off_144B10880 + 0x78` resolves to `Dce112AddStreamToCtx_MapBuildResources`.
- This is the concrete Windows callback used by `DcValidateContext_AddStreamViaResourceCallback` for this DCE11-style pool.

Windows DCE112 add-stream sequence:

- `Dce112AddStreamToCtx_MapBuildResources` calls:
  - `ResourceMapPoolResourcesForStream`
  - `ResourceMapPhyClockForStream`
  - `Dce112BuildMappedResourceForStream`
- This maps directly onto Linux `dce112_add_stream_to_ctx`:
  - `resource_map_pool_resources`
  - `resource_map_phy_clock_resources`
  - `build_mapped_resource`
- This is a strong Windows/Linux structural match, not just a naming guess.

Important seamless-boot reuse branch:

- Before adding streams, `DcValidateWithContext_RebuildStreamsAndPlanes` calls `MarkSeamlessBootStreamIfResourcesMatch` for every added stream.
- If a later stream has `stream + 0x933` set, Windows swaps that stream to index `0` before calling add-stream.
- `MarkSeamlessBootStreamIfResourcesMatch` sets `stream + 0x933` only when:
  - DC config flag at `dc + 0x1BC` is enabled,
  - BIOS/object-table accelerated mode check is false,
  - `IsType128StreamMatchingBootResources` returns true.
- `IsType128StreamMatchingBootResources` starts with a hard gate:
  - `display_object_or_link_type == 128`
- It then checks that the stream matches the boot/preboot resource identity:
  - the boot DIG is enabled,
  - the DIG frontend maps to a resource-pool stream encoder,
  - the boot resource index is valid,
  - boot timing fields match the new stream timing,
  - several DP/eDP signal-specific fields match,
  - eDP link-rate optimization would not require changing what firmware already set.

Where `dc + 0x1BC` comes from:

- `WindowsDM_SetPathMode_BuildStreamsAndCommit` sets `dc + 0x1BC = 1` when incoming path-mode-set flags have bit2 set:
  - `if ((path_mode_set_flags >> 2) & 1) dc->config_allow_seamless_boot_reuse = 1`
  - exact write: `0x141A22DAF`
- `WindowsDM_SetPowerState_ManageDcSeamlessFlags` clears `dc + 0x1BC` during the power-state transition:
  - exact write: `0x141A200C8`
- So Windows' seamless/preboot resource reuse is not just an ASIC default.
- It is enabled by a path-mode policy flag and then consumed later by DC validation.

Pipe allocation result:

- If `stream + 0x933` is set, `ResourceMapPoolResourcesForStream` tries `TryMapSeamlessBootPipeResources` first.
- `TryMapSeamlessBootPipeResources` maps the new stream to the existing boot/preboot pipe index and fills the pipe context from that resource slot.
- If this fails, Windows clears the flag and falls back to `MapFirstFreePipeResourcesForStream`.
- This is the exact Windows analogue of Linux's `stream->apply_seamless_boot_optimization` and `acquire_resource_from_hw_enabled_state`.

Split-pipe links:

- `ResourceMapPoolResourcesForStream` allocates the head pipe and stream resources.
- `AttachPlaneStateAndCloneSplitPipeLinks` / `FindOrCreateSplitClonePipeForPlaneAttach` create or reuse clone/split pipes during plane attach.
- The concrete pipe-link fields remain:
  - `pipe + 0x2F8` / `pipe + 0x300`
  - `pipe + 0x308` / `pipe + 0x310`
- `ResourceBuildScalingParamsAndViewport` later consumes these links to derive per-pipe viewport placement.

Linux comparison and immediate implication:

- Linux already has this same mechanism in `dc/core/dc_resource.c`:
  - `mark_seamless_boot_stream`
  - `acquire_resource_from_hw_enabled_state`
  - seamless stream swap-to-front before `dc_state_add_stream`
- But Linux enables it through `amdgpu_device_seamless_boot_supported`.
- In the current tree that function returns false for dGPUs unless `amdgpu.seamless=1` is forced.
- This iMac's Polaris GPU is a dGPU, so default Linux leaves:
  - `init_data.flags.seamless_boot_edp_requested = false`
  - `init_data.flags.allow_seamless_boot_optimization = false`
- Latest logs in `kernel_obs_8_gnome/dmesg.txt` confirm every relevant iMac stream has:
  - `boot_odm=0`
  - `seamless=0`

Updated interpretation:

- Windows is not merely doing a fresh two-tile modeset from scratch.
- For a type-128 internal DP-family stream, Windows tries to prove the new stream matches the firmware/preboot stream and then reuse the exact boot pipe/resource assignment.
- Linux currently has the equivalent generic machinery, but it is disabled by policy on this dGPU.
- For the next kernel attempt, the most Windows-faithful low-risk change is to enable and instrument seamless boot/resource reuse for this exact iMac/panel path, then check whether the probe-boot 5K state survives userspace takeover before adding more synthetic behavior.

### Windows Grouped View Solution And SetPathMode Metadata Branch

This pass continued statically from the Windows `SetPathMode` grouped-display branch and corrected one important working assumption.

IDA functions renamed/commented during this pass:

- `ConstructDisplayObjectWithPinAndModeState` = `0x141ED9780`
- `SetDisplayObjectBasePinEndpoint` = `0x141ED7C00`
- `SelectDisplayModeBlockByRoleOrSignal128` = `0x141ED7A60`
- `CopyPathModeRecordToSelectedModeObject` = `0x141ED9630`
- `SetSelectedModeEnabledFlag` = `0x141ED9670`
- `GetSelectedModeObject` = `0x141ED9720`
- `BuildDcStreamFromSelectedModeObject` = `0x141ED6880`
- `ComputeStreamDstRectFromPathScaling` = `0x141ED3C70`
- `StoreSelectedModeSrcDstRects` = `0x141ED94E0`
- `UpdateDisplayCachedSrcDstRectsAndDirtyFlag` = `0x141ED81B0`
- `SetStreamColorSpaceAndUpdateInfoPacketOnLivePipe` = `0x141B7D400`
- `BuildAviInfoPacketFromStreamColorSpace` = `0x141BACCA0`
- `CreateDisplayGroupViewSolution` = `0x141A3BBC0`
- `SetViewSolutionGlobalEnableFlag` = `0x141ACBF30`
- `ValidateRendererModesForViewSolution` = `0x141ACA150`
- `SelectMatchingModeForViewSolution` = `0x141ACA990`
- `PopulateViewSolutionPerRendererEntries` = `0x141ACAE00`
- `CacheViewSolutionActiveModeDimensions` = `0x141ACAE80`

Windows/Linux seamless matcher comparison:

- `IsType128StreamMatchingBootResources` is structurally very close to Linux `dc_validate_boot_timing`.
- Both paths:
  - start from the stream's display/sink object at stream offset `0`,
  - use the stream link object at stream offset `+8`,
  - compare boot hardware timing against the new stream timing,
  - reject DSC,
  - do extra DP/eDP link/pixel-format checks for DP-family signals.
- Important Linux limitation:
  - Linux's current `dc_validate_boot_timing` hard-gates on `sink->sink_signal == SIGNAL_TYPE_EDP`.
  - The observed secondary tile appears in Linux as a DP-side stream/sink path, not plain eDP.
  - So force-enabling seamless boot may help preserve the primary firmware/preboot stream, but by itself it probably will not mark the secondary DP tile as seamless.

Display object and selected-mode object flow:

- `ConstructDisplayObjectWithPinAndModeState` constructs the large per-display object and allocates:
  - endpoint/pin object at `display + 0x90`
  - lower state object at `display + 0x388`
  - selected mode object at `display + 0x94BC0`
- `SetDisplayObjectBasePinEndpoint` stores the connected `CBasePin` and pin name into the endpoint object.
- `CopyPathModeRecordToSelectedModeObject` copies the incoming `0xCC`-byte path-mode record into:
  - `selected_mode + 4`
- `BuildDcStreamFromSelectedModeObject` then turns that selected mode object into a DC stream:
  - copies timing fields from selected mode offsets `+0x6C..+0x98`
  - sets stream source rectangle from selected mode width/height at `+8/+12`
  - computes the stream destination rectangle through `ComputeStreamDstRectFromPathScaling`
  - writes/stores the combined source/destination rectangle block through `StoreSelectedModeSrcDstRects`
  - chooses stream signal/type from `SelectDisplayModeBlockByRoleOrSignal128`
- This means the per-stream source/destination geometry is ordinary selected-mode plumbing; it is not hidden in the post-commit grouped branch.

Grouped view solution layer:

- `CreateDisplayGroupViewSolution` receives a list of display ids and creates a grouped view-solution object from their per-display mode solutions.
- For each display id, it calls `GetOrCreateModeSolutionForDisplay`.
- If no mode solution exists, `CreateModeSolutionWith5KPipeSplitOptionFlags` constructs one and carries the mode-manager flags, including the `Dal5K60PipeSplit`-derived bit.
- `UpdateDisplayViewSolutionsWithDerived5kModes` still feeds the important derived-mode path:
  - gets/creates the display's mode solution,
  - validates it against the timing list,
  - calls `ModeManagerAddDerivedModesIncluding5kTile`,
  - thereby injecting the `2560x2880` half-tile mode when the known gates are satisfied.
- `ConstructViewSolutionBaseFlagsAndModeMetadata` counts renderers in the grouped solution:
  - one renderer sets single-view flags and type fields,
  - two renderers set grouped/multi-renderer flags:
    - `view_solution + 0xB8` gets bit `0x10`
    - `view_solution + 0xBC = 3`
    - `view_solution + 0xC0 = 3`
  - more than two renderers take the larger grouped class path.
- This is the higher layer that explains how Windows can keep a grouped logical view while `SetPathMode` later commits multiple real DC streams.

Important correction about the `SetPathMode` grouped branch:

- After `CollectActiveDcStreamsFromDisplayObjects`, Windows compares active display objects whose endpoint objects share the same owner/list pointer:
  - `display + 0x90 -> +0x98`
- That is a real grouped-display test.
- However, the operations in that branch are not the missing tile-positioning mechanism.
- First, if the grouped endpoint lacks byte `+0xCC`, Windows clears:
  - `stream + 0x230`
- `ConstructDcStreamStateFromDisplayObject` shows `stream + 0x230` is copied from display metadata count at `display + 0x528`, with small records copied to `stream + 0x234`.
- This looks like duplicate stream metadata/audio suppression for grouped displays, not framebuffer/tile X positioning.
- Second, after a successful DC commit, if the connected pin descriptor has byte `+6` bit `2` or bit `3`, Windows calls:
  - `SetStreamColorSpaceAndUpdateInfoPacketOnLivePipe(stream, 1)`
  - then repeats the same call for the grouped peer stream.
- That helper:
  - finds the live pipe carrying the stream,
  - stores a color-space enum at `stream + 0x380`,
  - rebuilds a 20-byte AVI/info packet at `stream + 0x3B0`,
  - pushes the packet through stream-encoder callbacks.
- This is a grouped-stream metadata/info-packet refresh after the topology already exists.
- It is not where Windows creates the second tile, chooses the right-tile source offset, or exposes one logical 5K display.

Current static conclusion:

- Windows appears to solve the 5K presentation in three cooperating layers:
  - the display/mode-solution layer injects and classifies the `2560x2880` half-tile and 5K pipe-split mode policy,
  - the grouped view-solution layer binds two display/render objects into one logical view,
  - the DC validation/resource layer commits multiple real streams/pipes and may reuse firmware/preboot resources when the seamless matcher succeeds.
- The post-commit grouped branch is supporting metadata synchronization, not the core 5K topology decision.

Kernel implication:

- For Linux, the Windows-faithful model is still:
  - two real internal streams/connectors must survive,
  - DRM/userspace should see one tiled/grouped logical monitor,
  - amdgpu DC should preserve or recreate the two stream/pipe paths before the first userspace modeset,
  - source/destination/viewport placement should come from the stream/tile/pipe graph rather than an emulated fake sink fallback.
- Seamless boot enablement remains useful for the probe/OCLP preservation path, but it is likely incomplete for the secondary DP-side tile unless Linux also handles the non-eDP secondary stream and grouped logical display state.

Next RE target:

- The remaining static target is no longer `SetStreamColorSpaceAndUpdateInfoPacketOnLivePipe`.
- Better targets are:
  - where a two-display id list is assembled for `CreateDisplayGroupViewSolution`,
  - where the resulting grouped view solution is attached back to the two display endpoint objects,
  - where the selected `2560x2880` derived mode is chosen for both physical display objects before `CopyPathModeRecordToSelectedModeObject`.

### Windows Active Display Set View Context And Tile Metadata

This pass followed the next branch from the grouped-view notes and found the concrete active-display-set validation layer used after selected modes are copied but before the final DC commit.

IDA names/comments added during this pass:

- `CreateOrGetDisplaySetViewCommitContext` = `0x141A75A70`
- `ValidateDisplaysShareTiledGroupMetadata` = `0x141A75760`
- `LookupCachedViewCommitContextByModeIds` = `0x141A75440`
- `CacheViewCommitContextByModeIds` = `0x141A75180`
- `ReleaseCachedViewCommitContext` = `0x141A75080`
- `ConstructMultiDisplayViewCommitContext` = `0x141ADB980`
- `ConstructCloneViewCommitContextForceFakeSink` = `0x141AD8D00`
- `ValidateViewContextAddStreamsPlanesAndFinalize` = `0x141AD9000`
- `FindViewContextEntryByDisplayIndex` = `0x141AD94C0`
- `FindViewContextEntryByDisplayIndexEx` = `0x141AD9500`
- `SetViewContextDisplaySourceSize` = `0x141ADB740`
- `SetViewContextRotationPolicy` = `0x141ADB5D0`
- `ApplyPathModeTimingToViewContextStream` = `0x141ADAC70`
- `SetViewContextTargetTimingBounds` = `0x141ADAB90`
- `FinalizeViewContextValidation` = `0x141ADA010`
- `ValidateActiveDisplaySetViaViewCommitContext` = `0x141A27FD0`
- `TryMarkCompatibleActiveDisplaySetForSpecialCommit` = `0x141A27A20`
- `UpdateSpecialCommitRatioFlags` = `0x141A244D0`

Mode-manager construction and ownership:

- `CreateDalModeManager` returns the public mode-manager interface at object `+0x28`.
- `ConstructDalModeManagerWith5kTiledOption` installs the mode-manager vtable at `0x144CB9D40`.
- The main Windows display-manager constructor stores that interface at WindowsDM object `+0xC8`.
- The constructor also installs a separate display-set/view-context interface at WindowsDM object `+0x50`, vtable `0x144AF4D08`.

Active display set path:

- `WindowsDM_SetPathMode_BuildStreamsAndCommit` copies each incoming `0xCC` path-mode record into the display object's selected-mode object.
- It then builds a real DC stream for each active display object.
- Before final `DcCommitStreams_BuildValidateAndCommit`, it calls `TryMarkCompatibleActiveDisplaySetForSpecialCommit`.
- That function requires exactly two live display objects for this path and calls `ValidateActiveDisplaySetViaViewCommitContext`.
- `ValidateActiveDisplaySetViaViewCommitContext`:
  - builds an array of active display indices,
  - calls the WindowsDM `+0x50` interface to create/get a view commit context,
  - feeds each active display's selected source size, rotation, timing, and target timing bounds into that context,
  - finalizes validation.

View commit context:

- `CreateOrGetDisplaySetViewCommitContext` accepts:
  - an array of active display indices,
  - a display count, maximum `6`,
  - a clone/fake-sink flag.
- It first tries `LookupCachedViewCommitContextByModeIds`.
- The cache key is the selected mode id at:
  - `SelectDisplayModeBlockByRoleOrSignal128(display) + 0x688`
- For each display in the active set, it records:
  - display index,
  - selected display mode block,
  - link signal/type from `display->link + 0x38`,
  - a small selected-mode flag from `sub_141ED7570`.
- Non-clone mode constructs `ConstructMultiDisplayViewCommitContext`.
- Clone/fake-sink mode constructs `ConstructCloneViewCommitContextForceFakeSink` and forces each context stream's signal/type field to `512`.
- The 5K path we care about is the non-clone multi-display context, because it preserves real stream objects.

What the multi-display context validates:

- `ConstructMultiDisplayViewCommitContext` stores per-display entries:
  - real stream pointer from `sub_1419FB920`,
  - plane state from `sub_141B9D530`,
  - display index,
  - signal/type,
  - selected-mode flag.
- `ValidateViewContextAddStreamsPlanesAndFinalize` then performs the same DC resource sequence we have seen elsewhere:
  - add each real stream to a DC context,
  - attach plane state,
  - clone/link split-pipe state through `AttachPlaneStateAndCloneSplitPipeLinks`,
  - finalize resources/viewports through `DcValidateFinalize_RebuildViewportsAndResources`.
- This is a strong static confirmation that Windows keeps the two real stream/pipe paths and validates them together. It does not solve the internal 5K panel by replacing them with a fake sink.

Tile/group metadata source:

- `ValidateDisplaysShareTiledGroupMetadata` is the first concrete consumer of tile-placement metadata found in this pass.
- It checks each display object's endpoint object:
  - `display + 0x90` is the endpoint object pointer.
  - endpoint `+0x98` is a shared tiled/group token.
  - endpoint `+0xA4` and `+0xA8` are row/column-like tile index fields.
  - endpoint `+0xAC` and `+0xB0` are grid dimensions; their product must equal the active display count.
  - endpoint `+0xCC` is the same byte later used by the post-commit grouped branch.
- The validator computes a per-tile slot from the row/column-like fields and verifies that all slots in the grid are present.
- It also prefers the display whose endpoint `+0xCC` byte is set when returning the representative/primary display index.
- This means Windows' left/right ordering is not guessed during `SetPathMode`; it is already encoded in endpoint tile metadata.

Secondary tile enable helper:

- The diagnostic string `dal3::WindowsDM::EnableSecondaryTileIfRequired` points to:
  - `WindowsDM_EnableSecondaryTileIfRequired` = `0x141A174B0`
- It checks the connected pin descriptor byte `+6`:
  - bit `2`,
  - or bit `3`.
- If either bit is set and the link/display type is `128`, it calls:
  - `SendSecondaryTileOpcode1265Payload1(link, payload)`
- This is the same decimal `1265` / DPCD `0x4F1` helper already traced earlier.
- `BringUpDisplayBySignalType` also calls the same helper after successful bring-up when the lower display object flag at `+0x5B4` is set.

EDID / sink-info bridge found in the follow-up:

- `WindowsDM_ReadType128EdidWithPinCapabilityFixups` = `0x141A21FE0`
  - for special type-128 internal DP-family sinks, it builds a temporary pin-cap object from the lower descriptor,
  - applies `ApplyPinCapabilityEdidFixups`,
  - returns patched EDID bytes.
- `ReadEdidIntoDcSinkThroughPin` calls WindowsDM vtable offset `+0x158` after constructing the pin object from EDID bytes.
  - In the WindowsDM class, vtable `+0x158` is `WindowsDM_EnableSecondaryTileIfRequired`.
  - So EDID read can also trigger the secondary-tile `0x4F1` assertion for capable paths.
- `ParseEdidAndPinCapsIntoDcSinkInfo` copies pin property tag `0x3B` into parsed sink-info output `+0xAC`.
  - When the caller output is `dc_sink + 0x508`, this becomes `dc_sink + 0x5B4`.
  - `BringUpDisplayBySignalType` later uses that lower display/sink field to gate the post-bring-up `0x4F1` helper.
- `AttachConstructedPinToDisplayEndpoint` = `0x141A0E040`
  - constructs a `CBasePin` from selected EDID/mode data,
  - attaches it to the display endpoint object,
  - sets endpoint name and pin pointer,
  - but does not populate endpoint tiled group fields `+0x98..+0xCC`.

Updated static conclusion:

- Windows has two distinct cooperating paths:
  - a secondary-tile enable path that asserts DPCD `0x4F1 = 1` for type-128 internal DP-family links when the Apple pin descriptor says the secondary tile is required,
  - a view-context validation path that uses endpoint tile metadata to validate a grouped/tiled active display set with real streams and split-pipe/plane resources.
- The endpoint tile metadata is now the best static candidate for the source of left/right tile placement.
- For Linux, the most Windows-faithful target is not a fake sink fallback:
  - preserve or recreate two real streams,
  - populate/use DRM tile/group metadata with the correct tile coordinates and grid dimensions,
  - keep both streams in the same atomic commit/resource context,
  - assert the secondary-tile DPCD enable only as the panel-side prerequisite, not as the whole solution.

Next RE target:

- Find who populates the endpoint metadata at `display + 0x90 + 0x98..0xCC`.
- Best candidates are the pin capability / display-info construction paths that consume Apple pin descriptor byte `+6`, capability tags `0x3A/0x3B/0x40`, and tiled EDID/display-id data.
- If this producer is found, it should tell us the exact tile coordinate convention Windows uses for the iMac pair.

### Windows 5K Mode-Manager And Detect Path Pass

This pass followed the endpoint metadata producer target, but the first useful result was a more complete Windows-side 5K recipe around detection, pin-cap EDID fixups, and mode/view solution construction.

IDA names/comments added or confirmed during this pass:

- `BuildAndCacheDisplayStatusSnapshot` = `0x141A28E50`
- `UpdateDisplayColorDataAndSyncPipeColorState` = `0x141ED7E50`
- `UpdateEndpointFreesyncAndCapabilityFlags` = `0x141A0BB50`
- `InitializeModeManagerViewTimingLists` = `0x141A3D4A0`
- `EnsureModeManagerMasterViewListCapacity` = `0x141A3C320`
- `ModeManagerAddViewTimingToMasterList` = `0x141A3C3D0`
- `ConstructDalModeManagerWith5kTiledOption` = `0x141A3DA00`
- `ModeManagerAddDerivedModesIncluding5kTile` = `0x141A3C4E0`
- `CreateModeSolutionWith5KPipeSplitOptionFlags` = `0x141A3C920`
- `UpdateDisplayViewSolutionsWithDerived5kModes` = `0x141A3CBA0`
- `MarkTimingPipeSplitForHighResAnd5k60` = `0x1415D5380`
- `DoubleEdidPhysicalWidthForTiledCaps` = `0x141A43030`
- `ApplyPinCapabilityEdidFixups` = `0x141A456E0`
- `DetectDisplayAndApplyApple5kOverrides` = `0x1414B3E80`
- `DetectDisplayMainMaybeApple5k` = `0x1414A96C0`
- `DetectDisplayEarlyExitForApple5kCaps` = `0x1414B31D0`
- `ApplyDetectionResultAndUpdateLiveDisplayObject` = `0x1414A8110`
- `UpdateDisplaySinkConnectionAndLiveAuxState` = `0x1414A7D90`
- `UpdateDisplayMapEntryLiveSinkObject` = `0x1414B3A20`

Negative result from the endpoint-metadata search:

- `UpdateDisplayColorDataAndSyncPipeColorState` is not the endpoint tile-coordinate producer.
  - It selects the display mode block, pushes `"color_data"` through the display object's interface, then syncs color state across the three pipe-state slots.
- `BuildAndCacheDisplayStatusSnapshot` is also downstream of the endpoint metadata producer.
  - It reads endpoint fields and connected pin properties,
  - builds a cached 0x88-byte status snapshot at display object `+0x94B18`,
  - and reads Apple pin tags such as `0x59/0x78/0x79`, but does not write endpoint tile fields `+0x98..+0xCC`.
- `UpdateEndpointFreesyncAndCapabilityFlags` writes endpoint feature/status fields such as `+0x44..+0x53` and `+0x120`, but not the tiled group block.
- Therefore the exact endpoint `+0x98..+0xCC` producer is still open, but the static search narrowed away several tempting downstream paths.

Important new Windows mode-manager gate:

- The string `DalEnable5kTiledMode` is referenced through `off_144C4E150`.
- `ConstructDalModeManagerWith5kTiledOption` reads that option and stores it at mode-manager `+0x80`.
- `ModeManagerAddDerivedModesIncluding5kTile` consumes mode-manager `+0x80`.
- The 5K half-tile mode is injected only when all of these are true:
  - mode-manager flag bit 7 is set,
  - source timing record flag bit 7 is set,
  - the source timing is exactly `1920x2160`,
  - mode-manager flag bit 5 is set,
  - `DalEnable5kTiledMode` is enabled.
- When all gates pass, Windows adds a derived physical half-tile mode:
  - `2560x2880`
- The same branch also adds:
  - `1280x1440`
- This is a major clue for Linux:
  - Windows does not merely rely on the raw panel EDID mode list.
  - It has a policy path that synthesizes the final `2560x2880` per-tile mode from an existing `1920x2160` timing.
  - The user-visible 5K view is then built above these real per-tile modes.

Timing flag source:

- `MarkTimingPipeSplitForHighResAnd5k60` sets timing record flag bit 7 when the timing aspect ratio matches display dimensions within a 90%-110% window.
- The same function checks DAL option id `1415` / `0x587`, which is `DalSupportMultiPipe`.
- If multi-pipe is allowed, it marks timing record `[9]` low bits as `2` for pipe-split/multi-pipe when one of these high-res conditions applies:
  - `5120x2880 @ 60`,
  - 8K-ish timings,
  - other high-res thresholds.
- It also checks option id `2168` in the `5120x2880 @ 60` path.
- The string table identifies `Dal5K60PipeSplit`; `CreateModeSolutionWith5KPipeSplitOptionFlags` stores the corresponding mode-solution policy flag at solution `+0xB8` bit 1.

View-solution bridge:

- `UpdateDisplayViewSolutionsWithDerived5kModes` gets or creates the mode solution for the display, validates it against the timing list, then calls `ModeManagerAddDerivedModesIncluding5kTile`.
- `GetOrCreateModeSolutionForDisplay` creates a mode solution through `CreateModeSolutionWith5KPipeSplitOptionFlags` if no cached solution exists.
- `CreateDisplayGroupViewSolution` builds grouped view solutions for multiple display ids.
- `ConstructViewSolutionBaseFlagsAndModeMetadata` counts renderers in the solution list:
  - one renderer sets single-display flags,
  - two renderers sets grouped/multi-renderer state,
  - grouped solutions keep multiple real renderers rather than collapsing into one fake sink.
- This matches the later `ValidateActiveDisplaySetViaViewCommitContext` path:
  - Windows builds one grouped logical/view context,
  - but validates/commits real per-display streams and planes underneath.

Pin-cap EDID fixup:

- `ApplyPinCapabilityEdidFixups` dispatches EDID fixups based on connected pin capability/property tags.
- Tags `0x3A` and `0x3B` call `DoubleEdidPhysicalWidthForTiledCaps`.
- `DoubleEdidPhysicalWidthForTiledCaps`:
  - checks base EDID byte `0x15` horizontal physical size against byte `0x16` vertical physical size,
  - if horizontal size is less than vertical size and less than `0x80`, doubles the horizontal physical size,
  - recomputes EDID block 0 checksum.
- This is a specific tiled-half EDID geometry fixup. It is separate from, but complementary to, the derived `2560x2880` mode injection.

Detect/live-backend path:

- `DetectDisplayAndApplyApple5kOverrides` calls `DetectDisplayEarlyExitForApple5kCaps`.
- `DetectDisplayEarlyExitForApple5kCaps` has explicit Apple/tiled handling:
  - connected pin descriptor byte `+6` bit 2 or bit 3 can short-circuit ordinary detect cleanup,
  - this is the same tag family used by the `0x4F1` helper and EDID fixups.
- In the normal detect path, `DetectDisplayAndApplyApple5kOverrides` calls `ProbeDisplayStatusMaybeAssert4F1`, then later has a tag-`0x3B` keep-connected branch:
  - if descriptor byte `+6` bit 3 is set,
  - and the display is physically present,
  - and the signal type is `11`,
  - then Windows forces result `+0x72` connected and clears result `+0x70`.
- `ApplyDetectionResultAndUpdateLiveDisplayObject` flows result byte `+0x72` into `UpdateDisplaySinkConnectionAndLiveAuxState`.
- `UpdateDisplaySinkConnectionAndLiveAuxState` calls `UpdateDisplayMapEntryLiveSinkObject`.
- `UpdateDisplayMapEntryLiveSinkObject` is the exact live-backend writer:
  - if the display object's connected/live vfunc says usable, it stores the active object at display-map entry `+0x18`,
  - otherwise it clears display-map entry `+0x18`.
- This explains why a Linux fake/emulated secondary sink is insufficient:
  - Windows preserves a real live backend object for the secondary DP-side path,
  - later status and `0x4F1` paths resolve through that live object.

Updated static model after this pass:

- Windows' 5K path has at least four cooperating layers:
  - detect/live-backend preservation keeps the secondary DP-side object usable,
  - pin-cap EDID fixups mark the half-panel geometry correctly,
  - mode-manager policy injects the `2560x2880` half-tile mode from `1920x2160`,
  - view-solution and active-display-set validation group two real renderers/streams into one higher-level view.
- DPCD `0x4F1 = 1` is still necessary panel-side state, but it is not the whole solution.
- The Linux target for the OCLP/probe-boot keep-5K case should mirror these properties:
  - keep the real secondary DP sink/link alive,
  - expose a real `2560x2880` mode on each half or preserve the already-derived mode,
  - preserve/emit tiled group metadata,
  - allow userspace to see one grouped logical `5120x2880` monitor rather than two unrelated displays.

Best next RE targets:

- Continue looking for the producer of endpoint tiled fields `display+0x90+0x98..0xCC`.
- Trace consumers/producers around `CreateDisplayGroupViewSolution` and `ValidateDisplayGroupAgainstViewSolution` to see where the grouped display-id list is generated.
- Trace how mode-manager flag bits 5, 7, and 9 are derived from init data/options in `sub_141A2A050`, especially the `DalEnable5kTiledMode`, `Dal5K60PipeSplit`, and `DalSupportMultiPipe` gates.
- Compare those Windows gates with Linux DC/amdgpu equivalents:
  - tiled EDID handling,
  - per-tile mode derivation,
  - multi-pipe/pipe-split policy,
  - live sink/link preservation during first userspace modeset.

### 2026-05-07 Windows RE pass: endpoint tile metadata producer resolved

This pass resolved the previous open target: who populates endpoint tiled fields `display+0x90+0x98..0xCC`.

IDA names/comments added or confirmed:

- `UpdateEndpointModesAndTiledMetadataAfterPinAttach` = `0x141A0F2F0`
- `PopulateEndpointTileMetadataFromDisplayIdTopology` = `0x141A206C0`
- `MarkModeRecordIfTileAspectCompatible` = `0x141A12400`
- `ParseDisplayId20TiledTopologyBlock40` = `0x141AEF130`
- `ParseDisplayId13TiledTopologyBlock18` = `0x141AFB700`
- `FindDisplayId20DataBlockByTag` = `0x141AEEF60`
- `FindDisplayId13DataBlockByTag` = `0x141AFB460`
- `QueryNextEdidExtensionForTiledTopology` = `0x141A388A0`
- `RefreshDisplayTileActiveFromMetadataFlags` = `0x1419F7BE0`
- `SetDisplayOverrideSinkInfoAndMetadataFlag` = `0x141ED7940`
- `SetDisplayTileActiveFlag` = `0x141ED75E0`
- `DisplayHasOverrideSinkInfoFlag403` = `0x141ED7610`
- `ApplyEdidPayloadAndRebuildDcSinkForDisplay` = `0x141ED7630`

Exact producer path:

- `sub_141A07ED0` calls WindowsDM interface slot `+0x120` immediately after `AttachConstructedPinToDisplayEndpoint`.
- WindowsDM constructor `sub_141A30570` installs vtable `off_144AF4760` at `WindowsDM+0x60`.
- Vtable slot `+0x120` / index 36 is `UpdateEndpointModesAndTiledMetadataAfterPinAttach`.
- Inside that function, `v43` is `endpoint+0x98`.
- It calls `PopulateEndpointTileMetadataFromDisplayIdTopology(a1-0x60, display_id, endpoint+0x98)`.
- If DisplayID tiled topology parsing fails, it clears the full 0x38-byte endpoint tile block with zeroes.
- If parsing succeeds, it scans peer displays with the same tile group token and elects one representative/primary tile by writing byte `endpoint_tile+0x34`.

Endpoint tile block layout as written by Windows:

- `endpoint_tile+0x00` qword: tile group token / identity, built from DisplayID topology identity bytes.
- `endpoint_tile+0x0C` dword: tile coordinate/index field.
- `endpoint_tile+0x10` dword: tile coordinate/index field.
- `endpoint_tile+0x14` dword: grid dimension.
- `endpoint_tile+0x18` dword: grid dimension.
- `endpoint_tile+0x1C` dword: per-tile active width-like field from DisplayID topology.
- `endpoint_tile+0x20` dword: per-tile active height-like field from DisplayID topology.
- `endpoint_tile+0x24` dword: computed total/offset field, `v12 + v8 * v11`.
- `endpoint_tile+0x28` dword: computed total/offset field, `v13 + v9 * v10`.
- `endpoint_tile+0x2C` dword: product of grid dimensions.
- `endpoint_tile+0x30` dword: zeroed by this path.
- `endpoint_tile+0x34` byte: primary/representative flag, seeded from DisplayID topology flags bit 2 and later overwritten by peer election.

DisplayID tiled-topology parsers:

- One EDID/display-info parser vtable uses `ParseDisplayId20TiledTopologyBlock40`.
  - It searches for DisplayID data block tag decimal `40` / hex `0x28`.
- Another parser vtable uses `ParseDisplayId13TiledTopologyBlock18`.
  - It searches for DisplayID data block tag decimal `18` / hex `0x12`.
- Both parsers fill the same 64-byte topology output consumed by `PopulateEndpointTileMetadataFromDisplayIdTopology`.
- If the current extension object does not contain the tiled topology block, both parsers can call `QueryNextEdidExtensionForTiledTopology` to ask the next EDID/DisplayID extension object.

Mode-list bridge:

- `MarkModeRecordIfTileAspectCompatible` compares candidate mode aspect ratio against the DisplayID tiled topology's total tile size.
- The tolerance is roughly 90%-110%.
- If the mode is compatible, it sets `mode_record[7] |= 0x80`.
- This explains the earlier mode-manager bit-7 gate for derived 5K half-tile modes.
- In other words, the derived `2560x2880` half-tile mode is not just enabled by global policy; the individual display/mode record must first be marked compatible with the DisplayID tiled topology.

Display active flags nearby:

- `SetDisplayOverrideSinkInfoAndMetadataFlag` sets `display+0x403` when an alternate/live sink-info block is present.
- `ApplyEdidPayloadAndRebuildDcSinkForDisplay` sets `display+0x402` after EDID payload/sink rebuild succeeds.
- `RefreshDisplayTileActiveFromMetadataFlags` sets `display+0x400` if either `display+0x403` or `display+0x402` is true.
- `WindowsDM_HandleDisplayTileActiveStateEvent` can still separately set `display+0x400` from a tile-active event payload.

Updated static conclusion:

- Windows does not infer the 5K left/right tile order from connector enumeration or from a magic viewport constant.
- It parses DisplayID tiled-topology blocks from the panel EDID data, converts them into endpoint tile metadata at `endpoint+0x98`, marks compatible mode records with bit `0x80`, validates that active displays share the same tile group token/grid, and only then builds grouped view/stream state.
- This makes the Linux target sharper:
  - preserve both real tile EDIDs / DisplayID tiled topology blocks,
  - preserve or synthesize correct DRM tile group metadata from those blocks,
  - keep the real secondary sink/link live,
  - ensure both half-tile mode records are accepted as tile-compatible,
  - let DC split/pipe topology derive the final per-pipe viewports instead of hardcoding a right-half offset.

Best next RE targets:

- Trace `CreateDisplayGroupViewSolution` and `ValidateDisplayGroupAgainstViewSolution` to see exactly how Windows chooses the grouped display-id set after endpoint metadata is populated.
- Trace `GetOrCreateModeSolutionForDisplay` and the mode-solution construction path enough to map mode-record bit `0x80` into the final `2560x2880` half-tile selection.
- If the actual iMac EDID/DisplayID bytes are available from probe logs, parse the tiled-topology block bytes directly and compare them against the Windows field mapping above.

### 2026-05-07 Windows RE pass: grouped view context and real EDID correlation

This pass followed the endpoint tile metadata producer downstream and correlated it with the captured iMac EDID bytes.

IDA comments added:

- `TryMarkCompatibleActiveDisplaySetForSpecialCommit`:
  - exactly two active display objects/streams are required before the special grouped path runs,
  - it chooses/marks a representative display state byte at `display_state+0xF2`,
  - grouping happens after both display objects already have real stream pointers.
- `ValidateActiveDisplaySetViaViewCommitContext`:
  - collects active display indices from live display objects,
  - creates/reuses a display-set view commit context through the WindowsDM `+0x50` subobject vtable,
  - feeds each real display's selected mode size into the grouped view context,
  - validates after clock/timing compatibility checks across the two active streams.
- `CreateOrGetDisplaySetViewCommitContext`:
  - non-clone mode constructs `ConstructMultiDisplayViewCommitContext`,
  - the multi-display context preserves one stream/plane entry per physical tile.
- `ValidateDisplaysShareTiledGroupMetadata`:
  - `endpoint+0x98` must be nonzero for every active display in the group,
  - all active displays must share the same DisplayID-derived tile group token,
  - the tile slot is derived from DisplayID tile coordinate fields, not connector enumeration order.

Downstream grouping model:

- `TryMarkCompatibleActiveDisplaySetForSpecialCommit` requires exactly two active display objects with stream pointers.
- It does not create a fake single sink.
- It marks one compatible display object as the representative/primary and then calls `ValidateActiveDisplaySetViaViewCommitContext`.
- `ValidateActiveDisplaySetViaViewCommitContext` builds a display-id list from active display objects and asks the display-set view-context layer for a commit context.
- `CreateOrGetDisplaySetViewCommitContext` then constructs `ConstructMultiDisplayViewCommitContext` in the non-clone case.
- Therefore Windows' "one 5K view" is a display-manager/view-solution abstraction above two real DC streams, not one synthetic DC stream replacing the two halves.

Actual iMac EDID / DisplayID block from probe logs:

- File checked:
  - `Probe_results\pre_gui_5k_20260326_150143\sys_class_drm\card1-eDP-1\edid.bin`
  - same bytes also appear in `Probe_results\live_linux_5k_20260326_135819\sys_class_drm\card1-eDP-1\edid.bin`
- EDID extension block 1 is DisplayID 1.3:
  - header starts `70 13 79 03 00`
  - first data block is tag `0x12`, revision `0`, length `0x16`
- Tiled Display Topology body:
  - `82 10 00 00 ff 09 3f 0b 00 00 00 00 00 41 50 50 13 ae 8b 8a e7 04`
- Decoded through the Windows parser's field mapping:
  - flags `0x82`
  - primary/representative seed is true because `(flags & 7) == 2`
  - single physical enclosure bit is set
  - tile size is `2560x2880`
  - DisplayID parser field0 count is `1`
  - DisplayID parser field1 count is `2`
  - tile location fields are `0,0`
  - vendor is `APP`
  - product is `0xAE13` / decimal `44563`
  - tiled serial is `0x04E78A8B` / decimal `82283147`
- Linux's DRM TILE string for this primary tile is:
  - `1:1:2:1:0:0:2560:2880`

Important Linux correlation:

- In the old probe/pre-GUI capture, only `eDP-1` had a populated TILE blob:
  - group id `1`
  - single monitor `1`
  - number of tiles `2x1`
  - tile location `0,0`
  - tile size `2560x2880`
- The DP-side connector had an active CRTC in some states but an empty EDID/TILE property.
- In later `kernel_obs_8_gnome`, both connectors had TILE blobs:
  - `eDP-1`: location `0,0`
  - `DP-1`: location `1,0`
- But the visible state still showed independent `2560x2880` scanouts/framebuffers in the bad GNOME path.
- This matches the Windows model:
  - tile metadata is necessary for grouping,
  - but it is not sufficient unless both real streams are committed together as one grouped view/pipe topology.

Updated Linux implication:

- We should not revert to a fake/emulated sink model.
- The Windows-faithful Linux target is:
  - keep the real secondary `DP-1` / `0x3113` route alive,
  - carry or reconstruct the missing peer tile metadata only for that real route,
  - keep both `2560x2880` streams active,
  - ensure userspace/DRM sees the pair as one tiled monitor,
  - ensure the first userspace modeset does not turn that into two unrelated `2560x2880` framebuffers.
- If the physical secondary EDID cannot be read reliably after boot handoff, reconstructing the right tile metadata from the primary DisplayID block is a Windows-shaped metadata reconstruction, not the same thing as emulating a sink, provided it is gated on the real secondary route being present.

Most useful next RE target:

- Trace the view-context method table used by `ConstructMultiDisplayViewCommitContext` enough to identify which method receives:
  - per-display selected mode size,
  - color/depth mode,
  - per-display timing object,
  - final validate call.
- The goal is to map Windows' grouped view context fields to Linux's atomic state:
  - one logical `5120x2880` tiled monitor/framebuffer,
  - two real `2560x2880` streams,
  - correct left/right source placement at the first graphical modeset.

### 2026-05-07 Windows RE pass: multi-display view-context methods

This pass opened the method table installed by `ConstructMultiDisplayViewCommitContext`.

Constructor details:

- `ConstructMultiDisplayViewCommitContext` = `0x141ADB980`
- It installs:
  - object vtable `off_144CC62F0`
  - view-context method table `off_144CC6220` at object `+0x28`
- It allocates one entry per display/tile:
  - `context+0xA8 + 40*i` receives the real DC stream pointer for display `i`
  - `context+0xB0 + 40*i` receives a plane state for display `i`
  - `context+0x19C + 16*i` stores the display index
  - `context+0x1A4 + 16*i` stores selected-mode/stream metadata flags
- This is another direct confirmation that Windows carries two physical tile streams through the grouped path.

Important view-context method table entries:

- slot `0`: `SetViewContextDisplaySourceSize` = `0x141ADB740`
- slot `1`: `SetViewContextRotationPolicy` = `0x141ADB5D0`
- slot `3`: `ApplyPathModeTimingToViewContextStream` = `0x141ADAC70`
- slot `7`: `SetViewContextTargetTimingBounds` = `0x141ADAB90`
- slot `9`: `ValidateMultiDisplayViewContextAddStreamsPlanesFinalize` = `0x141ADA460`
- slot `11`: `FinalizeViewContextValidation` = `0x141ADA010`

Source-size method:

- `SetViewContextDisplaySourceSize(display_id, width_height)` finds the view-context entry for that display id.
- It writes the selected source size into both:
  - the DC stream object fields around `stream+0x1E0..0x1E4`,
  - the paired plane/config object fields around `+0x628..+0x668`.
- This is per-display/tile source-size state, not a single fake 5K sink.

Timing method:

- `ApplyPathModeTimingToViewContextStream(display_id, timing)` finds the same per-display entry.
- It writes detailed timing fields into the stream object:
  - active dimensions,
  - blanking/sync parameters,
  - pixel clock,
  - timing flags.
- It then runs the normal stream setup/update path for that tile.

Final grouped validation method:

- `ValidateMultiDisplayViewContextAddStreamsPlanesFinalize` is the key bridge from grouped view state to DC resource state.
- It creates/uses one shared DC validation context.
- For each physical tile stream:
  - it copies/updates per-stream state into the validation context,
  - calls `DcValidateContext_AddStreamViaResourceCallback`,
  - calls `AttachPlaneStateAndCloneSplitPipeLinks`,
  - stops if any tile stream fails.
- Only after all tile streams/planes are present does it call:
  - `DcValidateFinalize_RebuildViewportsAndResources`
- This is the exact bridge that explains the later viewport finding:
  - Windows does not hardcode a final right-tile source-X literal in SetPathMode,
  - it builds a multi-stream validation context and lets the DC resource/viewport stage derive the final split placement.

Comparison with Linux states:

- A good Linux target is not merely:
  - both connectors have TILE blobs,
  - both connectors have `2560x2880` modes.
- The Windows-equivalent target is stricter:
  - both physical tile streams must be in the same validation/atomic commit,
  - both plane states must be attached under one grouped view/topology,
  - the resource/viewport rebuild must happen after the two streams are present together.
- This explains why `kernel_obs_8_gnome` could have two TILE blobs but still look wrong:
  - userspace/kernel state can still end up as two unrelated `2560x2880` scanouts instead of one grouped `5120x2880` view.

Current best Linux implementation direction:

- Preserve the real secondary route and TILE metadata work already done.
- Stop treating TILE metadata as the end of the problem.
- Add or adjust the atomic/DC path so that when the two iMac tile connectors are both active:
  - they are committed as one paired display set,
  - their streams are validated together,
  - the final plane/framebuffer state is one logical `5120x2880` view split across two `2560x2880` streams.
- If Linux cannot express that as a DC split/ODM graph on DCE 11, the next best analogue is a narrowly gated DRM tiled-monitor helper path that forces the first graphical modeset to allocate/use one logical framebuffer for the tile group.

### 2026-05-08 Windows RE pass: numbered remaining targets RE-1..RE-4

This pass labels the remaining Windows RE questions and resolves part of each enough to guide the next Linux patch.

Target labels:

- `RE-1`: confirm exactly what Windows passes into `ValidateMultiDisplayViewContextAddStreamsPlanesFinalize`: per-tile `2560x2880` source sizes, logical `5120x2880` view size, or both.
- `RE-2`: trace how Windows keeps the secondary lower display object live after detect / HPD instability.
- `RE-3`: confirm whether Windows reconstructs secondary tile metadata from the primary DisplayID block or requires a real secondary DisplayID/EDID object.
- `RE-4`: map final grouped-view validation to Linux concepts: two `dc_stream_state`s, two plane states, one logical tiled monitor, and correct right-half source placement.

`RE-1` findings:

- `ValidateActiveDisplaySetViaViewCommitContext` collects active display indices only from display objects that already have real stream pointers.
- It then creates or reuses a display-set view commit context through the `WindowsDM+0x50` subobject.
- For each active display, it reads the selected mode object:
  - width from selected mode `+0x08`,
  - height from selected mode `+0x0C`.
- It calls the view-context method-table slot `0`, `SetViewContextDisplaySourceSize`, once per active display with that per-display selected mode size.
- It calls method-table slot `3`, `ApplyPathModeTimingToViewContextStream`, once per active display with that display's timing object.
- It calls method-table slot `9`, `ValidateMultiDisplayViewContextAddStreamsPlanesFinalize`, only after the per-display source-size/timing setup and cross-display clock/timing checks.
- Therefore the final grouped path does not receive one fake `5120x2880` sink in place of the tile streams. It receives a multi-display context whose entries are real display/tile streams, each with its own selected mode/timing data. The logical 5K view is a display-manager/view-solution abstraction above those real streams.

`RE-2` findings:

- `DetectDisplayEarlyExitForApple5kCaps` has an Apple/tiled early-exit path when the pin descriptor byte family exposes the tag `0x3A/0x3B` bits.
- `DetectDisplayAndApplyApple5kOverrides` has a stronger `0x3B` keep-connected branch:
  - pin descriptor byte `+6` bit `3` set,
  - display object currently reports connected,
  - detection result would otherwise say disconnected,
  - object signal is `11` / DP-family,
  - branch forces detection result connected and clears the change flag at result `+0x70`.
- `ProbeDisplayStatusMaybeAssert4F1` begins by resolving the current display object id, looking up the display-map entry, and validating/refreshing the backend pointer at display-map `+0x18/+0x20` before status polling and possible `0x4F1`.
- This makes the Windows model deeper than "keep a sink pointer." It preserves or refreshes the lower display-object/backend/AUX identity before the private sink transition. Linux preserving only a DRM connector or stale `dc_sink` pointer is probably too shallow if the lower AUX/link backend has already been invalidated.

`RE-3` findings:

- `PopulateEndpointTileMetadataFromDisplayIdTopology` looks up the display object by display id and requires that display object to have an active DisplayID/EDID parser.
- It calls that display object's parser method at vtable slot `+0x158` to retrieve tiled topology into a local 64-byte structure.
- It then writes the endpoint tile metadata block from that display's own parsed topology.
- If parsing fails, `UpdateEndpointModesAndTiledMetadataAfterPinAttach` clears the endpoint tile metadata block rather than cloning it from a peer.
- `QueryNextEdidExtensionForTiledTopology` only asks the next extension in the same parser chain. It does not appear to search another display's primary-tile EDID.
- Therefore Windows appears to expect every grouped display endpoint to have its own tiled topology metadata. If Linux reconstructs missing secondary TILE metadata from the primary DisplayID block, that is a pragmatic Linux workaround, not a proven Windows behavior. It remains defensible only when gated on the real secondary route being present.

`RE-4` findings:

- `ValidateMultiDisplayViewContextAddStreamsPlanesFinalize` creates or uses one shared DC validation context.
- For each physical tile stream it:
  - updates/copies per-stream state,
  - calls `DcValidateContext_AddStreamViaResourceCallback`,
  - calls `AttachPlaneStateAndCloneSplitPipeLinks`.
- Only after all tile streams and planes are attached does it call `DcValidateFinalize_RebuildViewportsAndResources`.
- `AttachPlaneStateAndCloneSplitPipeLinks` clones/relinks pipe context pointers at split-chain fields equivalent to `+0x2F8/+0x300/+0x308/+0x310`.
- `DcValidateFinalize_RebuildViewportsAndResources` then calls `DcBuildScalingParamsAndViewportForAllPipes`.
- `ResourceBuildScalingParamsAndViewport` calls `ComputeViewportForSplitPipe`, which derives viewport placement from the pipe/split graph and split index/count.
- This confirms the Linux mapping:
  - two real `dc_stream_state`s,
  - two real plane states,
  - one shared atomic/DC validation context,
  - a preserved or rebuilt split/pipe relationship before viewport computation,
  - userspace-facing tiled group metadata that causes one logical 5K monitor,
  - no magic Windows literal for "right tile source X = 2560" at this layer.

Immediate Linux implications from this pass:

- Keep the current removal of fake/emulated sink behavior.
- Keep canonical tile order: primary/eDP `0,0`, secondary/DP `1,0`.
- Treat "secondary TILE property exists" as necessary but not sufficient.
- The suspicious Linux code is the current "preserve previous real sink after HPD disappeared" path. It is closer than the old emulated-sink path, but Windows suggests we may need to preserve/refresh the lower AUX/link backend earlier, during detect, before the secondary object is allowed to fall through ordinary disconnect handling.
- The next kernel test should show whether two real streams and plane states enter the same DC atomic/validation state and whether the pipe split/viewport graph is present before GNOME takes over.

### 2026-05-08 Windows RE-2 deep pass: display-map transport versus live sink record

This pass corrects an important structural ambiguity from earlier notes: there are two different `+0x18` slots.

Structure A: 64-byte display-map entry

- `FindDisplayEntryByObjectId` returns this structure.
- It compares the requested object id against the descriptor at entry `+0x08`.
- `InsertInterfaceIntoDisplayMap` creates this entry:
  - entry `+0x00`: display-map interface pointer,
  - entry `+0x08`: object-id descriptor,
  - entry `+0x10`: present/constructed byte,
  - entry `+0x11`: live byte,
  - entry `+0x18`: per-object DPCD/AUX transport interface,
  - entry `+0x20`: companion/helper object when enabled.
- `PopulateDisplayMapAuxObjectsForHotplugRange` seeds entry `+0x18` by calling `CreateDpcdAuxInterfaceForObjectId`.
- `BuildStreamForDisplayEntryPath` copies entry `+0x18` into stream init data at `+0x18`.
- `WriteSinkDpcd4F1Payload1Retry` also resolves `FindDisplayEntryByObjectId` and calls entry `+0x18` vfunc `+8` to write DPCD address decimal `1265` / hex `0x4F1`.
- Therefore this entry `+0x18` is the lower per-object AUX/DPCD transport. This is the one Linux must preserve as an AUX/DDC/link backend analogue.

Structure B: 104-byte display-object detection record

- `FindDisplayObjectRecordByObjectId` returns this structure.
- It compares the requested object id against the descriptor at record `+0x00`.
- `AddDisplayObjectToDetectionRecord` appends candidate display objects:
  - record `+0x18`: currently live display/sink object,
  - record `+0x20`: candidate display object array slot 0,
  - record `+0x28`: candidate display object array slot 1,
  - record `+0x30`: candidate count, capped below 2.
- `UpdateDisplayObjectRecordLiveSinkObject` updates this structure:
  - it verifies the display object is already in the candidate array,
  - if display-object vfunc `+0x258` reports connected/usable, record `+0x18 = display_object`,
  - otherwise it clears record `+0x18`.
- This record `+0x18` is not the DPCD/AUX transport. It is the current live display/sink object selected by detection state.

Corrected RE-2 model:

- Windows decouples the lower per-object AUX transport from the current live sink state.
- The 64-byte display-map entry's `+0x18` DPCD/AUX interface is created during topology/display-map construction and is later reused by detect-time `0x4F1` and stream construction.
- The 104-byte detection record's `+0x18` live display object can be updated or cleared as connection state changes.
- This explains how Windows can still perform object-specific DPCD transactions during detection/status handling without depending on a Linux-style `local_sink` pointer being currently non-null.

Special `0x3113` policy from this pass:

- `CreateDisplayMapInterfaceForObjectIdMaybeSkipInternal` only accepts class-3 display/sink object ids. Both `0x3113` and `0x3114` qualify.
- When AdapterService base-pin property13 bit 17 is set, it skips low-byte `0x14` and `0x0E` class-3 display objects.
- Low-byte `0x13` / object `0x3113` is deliberately not skipped.
- `ConstructDpcdAuxInterfaceForObjectId` enables default `DPDelay4I2CoverAUXDEFER` for low-byte `0x13`; therefore `0x3113` has a default AUX-defer tolerance policy.
- `AddDisplayObjectToDetectionRecord` also detects low-byte `0x13` and sets a record policy byte at `record+0x13`, while clearing `record+0x12`.
- These are concrete static signs that the paired DP-side tile object is not treated like a plain external DP connector.

Linux implication after the correction:

- The current Linux "preserve previous real sink after HPD disappeared" path is probably preserving the wrong abstraction level.
- The Windows-shaped goal is not only `connector_status_connected` or non-null `dc_sink`.
- The closer analogue is:
  - construct/identify the `0x3113` route early,
  - keep its AUX/DDC backend usable even when HPD/local-sink state wobbles,
  - allow DPCD/source-DPCD/`0x4F1` transactions on that backend when Windows would use the 64-byte display-map entry `+0x18`,
  - separately track whether a live sink/display object exists for modeset policy.
- For the Linux code, that points to decoupling "can use AUX on the secondary route" from "has current `link->local_sink`", at least inside the narrowly gated iMac5K path.

### 2026-05-08 RE-2 closure: display-map transport lifetime

Follow-up target: confirm whether Windows clears/destroys the 64-byte display-map entry `+0x18` AUX/DPCD transport when the live sink object wobbles or disappears.

The display-map maintenance helpers now have conservative IDA names:

- `InsertOrFindSortedDisplayMapEntry64` (`0x1414AC300`) inserts or returns a sorted 64-byte display-map entry. The raw vector helper copies the full 64-byte temporary record into the map, but does not contain HPD/detect policy.
- `PopulateProviderObjectInterfacesMarkPresent` (`0x1414B5D70`) inserts provider-reported interfaces and marks entry byte `+0x10` present.
- `InsertDisplayTransportEntryMarkPresent` (`0x1414B5EA0`) creates a class-2 transport interface, inserts it into the map, and marks entry byte `+0x10` present.
- `MarkDisplayMapEntryLive` (`0x1414B69C0`) marks entry bytes `+0x10/+0x11`; if the entry was already live, it ORs bit 0 into the base interface flags first.
- `ClearDisplayMapEntryRefCounts` (`0x1414ADD90`) clears entry dword `+0x0C` refcounts for all map entries.
- `DecrementDisplayMapEntryRefAndClearLiveByte` (`0x1414ABD50`) decrements entry dword `+0x0C`; when it reaches zero, it clears byte `+0x12`. This is liveness/reference bookkeeping, not the entry `+0x18` AUX pointer.
- `DestroyDisplayMapBaseInterfaces` (`0x1414AECC0`) releases the base display-map interface at entry `+0x00`. In this display-map cleanup path, no corresponding clear of entry `+0x18` was observed.

What this clarifies:

- The only confirmed seed for the class-3 display-object AUX/DPCD transport remains `PopulateDisplayMapAuxObjectsForHotplugRange`, which writes `CreateDpcdAuxInterfaceForObjectId(...)` to 64-byte display-map entry `+0x18`.
- Detect/status code validates/uses entry `+0x18/+0x20`, and stream construction copies entry `+0x18` into stream init data.
- The display-map ref/live-byte maintenance routines do not clear entry `+0x18` when a display object becomes not-live.
- The separate 104-byte detection record can still clear its own record `+0x18` live display/sink object. That does not destroy the lower object-id AUX transport.

RE-2 conclusion:

- RE-2 is now sufficiently closed for the next Linux design step.
- Windows preserves the lower object-id-routed AUX/DPCD backend separately from the current live sink/display-object decision.
- Therefore a Windows-shaped Linux fix should not be "keep a stale `dc_sink` alive at all costs" and should not fall back to an emulated sink.
- The closer Linux target is to preserve or reacquire the real secondary route's AUX/DDC backend and tile metadata independently of transient `link->local_sink` state, then build two real tile streams/planes from that backend when the reference probe/OCLP state exists.
- Earlier notes that described the same display-map `+0x18` as both transport and live sink should be treated as superseded by the corrected Structure A / Structure B split above.

### 2026-05-08 RE-1 closure: grouped validation inputs

Follow-up target: determine exactly what Windows passes into `ValidateMultiDisplayViewContextAddStreamsPlanesFinalize`: per-tile `2560x2880` source sizes, a logical `5120x2880` view size, or both.

The relevant path is now clear:

- `TryMarkCompatibleActiveDisplaySetForSpecialCommit` requires exactly two active display objects with stream pointers before entering this path.
- It marks one display object as the representative/primary by setting display state byte `+0xF2`.
- It then calls `ValidateActiveDisplaySetViaViewCommitContext`.
- `ValidateActiveDisplaySetViaViewCommitContext` calls the display-set view context factory with the final argument `0`.
- That `0` selects the normal `ConstructMultiDisplayViewCommitContext` path, not `ConstructCloneViewCommitContextForceFakeSink`.
- `ConstructCloneViewCommitContextForceFakeSink` does exist, and it sets stream field `+0x980` to `512`, but this is the clone/fake-sink variant and is not the active two-display grouped path we are mapping for iMac 5K.

Normal multi-display context construction:

- `ConstructMultiDisplayViewCommitContext` allocates one context entry per display/tile.
- Each entry stores a real per-tile DC stream pointer at context entry `+0x00` / container offset `a1 + 40*i + 168`.
- Each entry allocates a real plane state at context entry `+0x08` / container offset `a1 + 40*i + 176`.
- Each entry records the display index and mode/signal metadata in the parallel 16-byte display-index table.

Per-display source/timing setup:

- `ValidateActiveDisplaySetViaViewCommitContext` reads the selected mode object for each active display:
  - selected mode `+0x08`: width,
  - selected mode `+0x0C`: height.
- It calls `SetViewContextDisplaySourceSize` once per display with that per-display width/height.
- `SetViewContextDisplaySourceSize` writes the width/height into both the stream-side fields and plane/state fields of the matching view-context entry.
- It calls `ApplyPathModeTimingToViewContextStream` once per display with that display's timing object.
- It calls `SetViewContextTargetTimingBounds` for non-representative displays while adjusting compatible timing ratios.
- Only after this per-display setup succeeds does it call final grouped validation.

Selected-mode to stream geometry:

- `BuildDcStreamFromSelectedModeObject` sets the stream source rectangle origin to `0,0`.
- It sets stream source width/height from the selected mode/path-mode width and height for that display.
- It computes the destination rectangle from path scaling fields.
- It stores source/destination rectangles back into selected-mode state via `StoreSelectedModeSrcDstRects`.
- Therefore the per-stream input is not one synthetic `5120x2880` stream. Each tile stream is initialized as its own display-sized source, expected to be `2560x2880` for the iMac 5K halves.

Final validation:

- `ValidateMultiDisplayViewContextAddStreamsPlanesFinalize` operates on the shared multi-display view context.
- For each context entry, it copies/refreshes stream and plane state from the prepared template objects.
- It calls `DcValidateContext_AddStreamViaResourceCallback` once per real tile stream.
- It calls `AttachPlaneStateAndCloneSplitPipeLinks` once per stream/plane pair.
- It calls `DcValidateFinalize_RebuildViewportsAndResources` only after all tile streams and planes have been added to the same validation context.

RE-1 conclusion:

- Windows passes per-tile/per-display mode sizes and timings into the grouped view context, not a single fake `5120x2880` sink/stream.
- The logical 5K display is a grouping/view-solution outcome above the physical stream objects, plus the final DC validation/resource rebuild across those streams.
- Right-half placement is not passed into final validation as a simple source `x=2560` literal in this path. Each stream starts with source origin `0,0`; the right-half relationship is derived later from tile/group/split-pipe/viewport state.
- For Linux, the closest match remains:
  - two real streams,
  - two real plane states,
  - per-stream source size `2560x2880`,
  - correct DRM tile/group metadata so userspace sees one logical `5120x2880` monitor,
  - a shared DC/atomic validation path that keeps both tile streams in one commit and computes the final viewport/pipe relationship.
- This strengthens the decision to avoid the previous emulated/fake-sink path for the target solution.

### 2026-05-08 RE-3 closure: secondary tile metadata source

Follow-up target: confirm whether Windows reconstructs the secondary tile metadata from the primary DisplayID block, or whether each active endpoint needs its own real DisplayID/EDID topology parser.

Producer-side result:

- `PopulateEndpointTileMetadataFromDisplayIdTopology` has one code xref: `UpdateEndpointModesAndTiledMetadataAfterPinAttach`.
- It receives a display id, resolves that same display object through the display array, and requires:
  - the display object exists,
  - display byte `+0x400` says the display object is active/valid,
  - the display object's own parser object at display `+0x90` can return tiled topology through vfunc `+0x158`.
- If any of those fail, it returns false.
- The caller then zeroes the 0x38-byte endpoint tile block at `endpoint+0x98..0xCF`.
- No peer-copy or primary-tile reconstruction branch was found on this failure path.

Parser-chain result:

- `ParseDisplayId20TiledTopologyBlock40` searches for DisplayID 2.0 tiled-topology block tag `0x28`.
- `ParseDisplayId13TiledTopologyBlock18` searches for DisplayID 1.x tiled-topology block tag `0x12`.
- If a tiled-topology block is not in the current extension, both parsers call `QueryNextEdidExtensionForTiledTopology`.
- `QueryNextEdidExtensionForTiledTopology` now resolves through `GetNextEdidExtensionParserSlot`, which is simply parser-local storage at `parser+0x50`.
- Therefore this fallback walks the next EDID/DisplayID extension in the same display object's parser chain. It does not search another display object's EDID and does not use the primary tile as a metadata donor.

Peer/grouping result:

- After successful metadata population, `UpdateEndpointModesAndTiledMetadataAfterPinAttach` scans peer displays with the same tile group token.
- That peer scan only elects a single representative/primary tile by writing byte `endpoint_tile+0x34`.
- It may clear the representative flag on peers with the same group token.
- It does not copy the 0x38-byte tile metadata block from one endpoint to another.

Consumer-side result:

- `ValidateDisplaysShareTiledGroupMetadata` rejects grouped validation if any active display lacks its own nonzero `endpoint+0x98` tile group token.
- It checks that every active endpoint has the same DisplayID-derived group token.
- It computes occupied tile slots from each endpoint's own coordinate fields.
- It checks that the occupied slot count matches the grid product.
- This consumer logic expects every grouped endpoint to already have valid tile metadata; it does not fill missing secondary metadata from the representative/primary endpoint.

RE-3 conclusion:

- Static evidence says Windows does not reconstruct secondary tile metadata from the primary DisplayID block in this path.
- Windows expects each real tile endpoint to carry its own parsed DisplayID tiled-topology metadata.
- If Linux reconstructs the secondary `DP-1` TILE property from the primary `eDP-1` DisplayID, that is a Linux compatibility workaround, not a behavior copied from this Windows path.
- That workaround is still reasonable only if gated on the real secondary route/backend being present, because Windows also requires the real stream/display object before grouped validation.
- Best Linux target remains: preserve or reacquire the secondary route's own EDID/DisplayID/tile metadata if possible; only synthesize the missing TILE metadata as a last narrow quirk when the physical secondary route is proven real.

### 2026-05-08 RE-4 closure: Windows grouped validation mapped to Linux/DC

Follow-up target: map final grouped-view validation to Linux concepts: two `dc_stream_state`s, two plane states, one logical tiled monitor, and correct right-half source placement.

View context to DC validation:

- `ConstructMultiDisplayViewCommitContext` creates one entry per physical display/tile.
- Each entry holds:
  - a real prepared stream object,
  - a real prepared plane state,
  - display index / mode metadata.
- `ValidateMultiDisplayViewContextAddStreamsPlanesFinalize` is the grouped/special path.
- `ValidateViewContextAddStreamsPlanesAndFinalize` is the generic sibling.
- Both use the same DC resource sequence:
  - prepare or refresh stream state,
  - prepare or refresh plane state,
  - add each stream to one shared validation context,
  - attach plane state and clone/relink pipe-split links,
  - finalize resources and rebuild viewports only after all streams/planes are present.

Direct DCE 11.2 resource mapping:

- `DcValidateContext_AddStreamViaResourceCallback` stores the stream in the validation context stream array and increments context stream count.
- It then invokes resource callback `+0x78`.
- On this driver, that callback is `Dce112AddStreamToCtx_MapBuildResources`.
- `Dce112AddStreamToCtx_MapBuildResources` performs:
  - `ResourceMapPoolResourcesForStream`,
  - `ResourceMapPhyClockForStream`,
  - `Dce112BuildMappedResourceForStream`.
- This maps directly to the Linux DCE 11.2 path conceptually named `dce112_add_stream_to_ctx`.
- `ResourceMapPoolResourcesForStream` chooses or reuses a pipe slot for the stream:
  - it first marks/tries seamless-boot resources if the stream matches preboot resources,
  - otherwise maps the first free pipe,
  - otherwise recycles a split/clone pipe context.

Plane attach and split links:

- `AttachPlaneStateAndCloneSplitPipeLinks` finds the head pipe for the stream.
- It attaches the plane state to each relevant pipe in the chain.
- If a clone/split pipe is needed, it finds or creates one via `FindOrCreateSplitClonePipeForPlaneAttach` or `RecyclePipeContextForSplitClone`.
- It propagates pipe links equivalent to:
  - `pipe+0x2F8` / `pipe+0x300`,
  - `pipe+0x308` / `pipe+0x310`.
- These correspond to the linked pipe/split topology that later viewport code walks.

Viewport/resource finalization:

- `DcValidateFinalize_RebuildViewportsAndResources` is called once after the grouped stream/plane loop.
- It then calls `DcBuildScalingParamsAndViewportForAllPipes`.
- `DcBuildScalingParamsAndViewportForAllPipes` iterates active pipe slots and calls `ResourceBuildScalingParamsAndViewport`.
- `ResourceBuildScalingParamsAndViewport` calls `ComputeViewportForSplitPipe`.
- `ComputeViewportForSplitPipe` calls `GetSplitPipeCountAndIndex`.
- `GetSplitPipeCountAndIndex` derives split count and pipe index by walking the pipe-context links, including the `+0x308/+0x310` chain and same-plane split siblings.

Right-half placement result:

- No RE-4 function passes or stores a simple iMac-specific right-tile literal like `source_x = 2560`.
- The stream source rectangle can still begin at `0,0` per tile.
- The localized per-pipe viewport is derived from:
  - the grouped validation context,
  - the pipe/split graph,
  - split count,
  - current pipe index,
  - scaling/source/destination rectangles.
- Therefore Windows' right-half placement is a resource/viewport consequence of a correct grouped pipe graph, not a magic coordinate patched into the mode object.

Linux mapping:

- Windows display-manager/user side exposes a grouped/tiled logical view.
- Windows DC/resource side still validates real tile streams and real plane states.
- The closest Linux steady state is:
  - DRM exposes one tiled logical monitor to userspace,
  - DC receives both tile streams in the same `dc_state`,
  - both streams have real sinks/routes,
  - each stream has a plane state,
  - resource validation builds/preserves the pipe relationship before viewport/scaling computation.
- If Linux submits two independent one-stream commits, or if userspace creates two unrelated `2560x2880` scanouts, Windows' viewport model is already broken before the finalizer.
- If Linux cannot express the Windows split graph across the iMac's two tile connectors in DCE 11.2 DC, the fallback design should move up one layer: preserve two real streams/routes, but make DRM/userspace tiled-monitor grouping force a single logical `5120x2880` framebuffer/tiled presentation instead of two independent desktops.

RE-4 conclusion:

- RE-4 is sufficiently closed.
- Windows does not solve iMac 5K by presenting one fake `5120x2880` stream to DC.
- Windows also does not solve right-half placement by hardcoding a `2560` source offset in the visible path.
- It solves it by validating two real streams/planes together and letting resource/pipe/viewport code derive placement from the grouped split topology.
- For Linux kernel work, the next high-confidence target is therefore the shared atomic/DC validation shape:
  - keep or reacquire the real secondary route,
  - keep valid tile metadata,
  - make the first graphical modeset contain both tile streams together,
  - prevent independent one-stream collapse,
  - log whether Linux DC builds any pipe/split/ODM relationship before viewport/scaling finalization.

### 2026-05-08 remaining RE targets after RE-1..RE-4

The original RE-1..RE-4 questions are now closed enough to stop reusing those numbers. The remaining work is narrower:

- `RE-5`: resource/split trigger. Identify whether Windows creates a real DCE/DC split graph across the two iMac tile streams, or whether the "one 5K monitor" model lives above DC as a grouped/tiled view over two real streams.
- `RE-6`: secondary activation order. Reconfirm the exact Windows order around secondary `0x3113`: EDID/DisplayID, source-DPCD `0x310`, link readiness/training, sink-private `0x4F1`, and any status polling.
- `RE-7`: secondary live-sink recovery. Determine whether Windows preserves a real secondary sink through HPD/detect instability, reacquires it from the durable display-map AUX route, or keeps only the lower transport and rebuilds the display object later.
- `RE-8`: cold-boot enablement. Determine whether the Windows runtime driver can create the full 5K state from a plain 4K-like boot without Apple preinit, and if so which non-Apple register/DPCD/link sequence is required.
- `RE-9`: Linux correlation. Map the final Windows model to exact Linux `amdgpu_dm` / DCE11.2 / DRM tiled-monitor code paths and decide which current Linux changes are wrong, redundant, or still useful.

### 2026-05-08 RE-5 pass: DCE112 resource/split trigger

Target question:

- Does Windows create one hidden DCE split graph that stitches `eDP-1` and `DP-1` together below DC, or does it keep two real tile streams and expose the logical 5K monitor above DC through grouped/tiled view state?

Static result:

- `ValidateMultiDisplayViewContextAddStreamsPlanesFinalize` still loops over active display entries and calls the add-stream/resource callback once per real tile stream.
- `ConstructMultiDisplayViewCommitContext` allocates one real stream and one real plane per active display/tile. In the two-tile iMac case this is two stream/plane entries, not one fake `5120x2880` DC stream.
- On this driver, resource callback table slot `+0x78` is `Dce112AddStreamToCtx_MapBuildResources`.
- The same DCE112 resource callback table has slot `+0x68` set to `NULL`. Therefore `FindOrCreateSplitClonePipeForPlaneAttach` cannot ask this table for a special resource-specific cross-stream split pipe.
- `ResourceMapPoolResourcesForStream` maps one stream to a head pipe/resource set:
  - it may reuse a seamless-boot pipe only for a type-128 internal stream that matches boot/preboot resources,
  - otherwise it maps the first free pipe,
  - otherwise it recycles a split/clone pipe context.
- `MarkSeamlessBootStreamIfResourcesMatch` / `IsType128StreamMatchingBootResources` is explicitly gated on stream signal `128`. This can preserve the firmware-established primary/internal boot pipe, but it is not the direct mechanism for the DP-side `0x3113` tile, which is signal `32`.
- `AttachPlaneStateAndCloneSplitPipeLinks` walks the pipe chain for the same stream, attaches the plane state, and clones/relinks same-plane/split siblings.
- `GetSplitPipeCountAndIndex` / `CountSamePlaneSplitSiblings` only count linked pipe contexts that belong to the same plane pointer or the explicit secondary split chain. This is intra-stream/intra-plane split topology, not arbitrary grouping of two different tile streams.

RE-5 conclusion:

- Static evidence now argues that Windows does not stitch `0x3114` and `0x3113` into one hidden cross-connector DCE split stream.
- The Windows model remains two real streams and two real planes inside DC/resource validation.
- The "one 5K monitor" behavior is most likely owned above DC by grouped/tiled display-manager view state, with DC receiving synchronized real tile streams underneath.
- Practical RE-5 framing: for Linux we do not need to reproduce Windows' full display-manager abstraction; we need the DRM/KMS state consumed by GNOME/KDE to describe one tiled monitor made from two real tile streams.
- Therefore the Linux target should not be a forced DCE `next_odm`/split-pipe link between unrelated eDP and DP connectors unless runtime logs prove Linux had such a relationship in the good pre-GUI state.
- The higher-confidence Linux target is:
  - keep/reacquire the real `0x3113` secondary route,
  - keep both tile streams in the same atomic/DC validation transaction,
  - expose correct DRM tile-group metadata so GNOME/KDE see one logical tiled monitor,
  - avoid presenting two unrelated desktops with independent `2560x2880` framebuffers.

### 2026-05-08 RE-6 pass: secondary activation ordering

Target question:

- Is Linux still missing a major Windows-style secondary activation step before the 5K tiled presentation problem, or have we already reproduced the important DPCD sequence?

Static ordering confirmed:

- `ProbeDisplayStatusMaybeAssert4F1` begins by resolving the current display object id, finding the 64-byte display-map entry, and validating the durable per-object DPCD/AUX backend at entry `+0x18/+0x20`.
- `WriteSinkDpcd4F1Payload1Retry` uses that display-map entry `+0x18` transport and writes DPCD address decimal `1265` / hex `0x4F1`, payload `1`, length `1`, retrying once after `10 ms`.
- `RetrieveDpLinkCapsAfterSourceDpcdSetup` calls `ProgramDpPreTrainingAuxState` before the normal receiver-cap reads from DPCD `0x000` and before extended-cap reads from `0x2200`.
- `BringUpDpOrEdpWithLinkTraining` also calls `ProgramDpPreTrainingAuxState` before `PerformLinkTrainingWithRetries`.
- For signal `32` / DisplayPort, if `ClassifyDpLinkRateEncodingRange(...) == 2`, Windows first calls `SetDpcdMstCtrlEnableBit`, which reads DPCD `0x111`, sets bit `0`, and writes it back.
- For signal `128` / eDP, Windows runs the extra DisplayCore HPD/AUX-ready callbacks at vtable offsets `+0x103F0/+0x103F8` before source-DPCD and training, and repeats those inside training retries through `PrepareLinkTrainingStateAndArmEdp128`.
- `ProgramDpPreTrainingAuxState` writes:
  - DPCD `0x300`: source OUI `00 00 1A` if needed,
  - DPCD `0x303`: a 9-byte source block,
  - DPCD `0x310`: `04 1D 03` or `05 1D 03` depending on the configured link-rate class,
  - DPCD `0x340`: optional minimum-hblank byte when enabled.
- `StreamEnableWrite4F1AndPollSinkStatus` is a later stream-enable path. It writes DPCD `0x4F1 = 1` if pin capability tag `0x3A` or `0x3B` is present, and this path is not restricted to signal `128`.
- The follow-up `DPCD 0x205` poll is only gated by capability tag `0x40`; the static Apple `AE25` / `AE26` evidence has not proven tag `0x40`, so Linux should not blindly add that poll as required behavior.

RE-6 conclusion:

- The important activation order is now clear:
  1. preserve or reacquire the real object-specific AUX route for `0x3113`,
  2. if applicable for signal `32` / class `2`, set DPCD `0x111` bit `0`,
  3. publish source-DPCD `0x300/0x303/0x310` before caps/training,
  4. run normal link training/stream enable,
  5. assert sink-private DPCD `0x4F1 = 1` on the same real AUX route,
  6. poll `0x205` only if runtime evidence proves the `0x40` tag.
- Our Linux experiments already proved the real `0x3113` route can write source-DPCD `0x310 = 04 1D 03` and sink-private `0x4F1 = 1` while the physical secondary sink is live.
- Therefore RE-6 does not point to another mysterious firmware opcode or Apple runtime call.
- The only plausible still-missing Windows-like activation detail is the signal-32/class-2 `DPCD 0x111` bit0 step, but the current OCLP/probe GNOME failure is more strongly explained by Linux later dropping or presenting the secondary tile incorrectly despite successful early `0x310`/`0x4F1`.
- Next highest-value RE target is `RE-7`: what Windows does when the secondary display object's live sink/detect status wobbles after this activation sequence.

### 2026-05-08 RE-7 pass: secondary live-sink preservation after detect wobble

Target question:

- After the secondary `0x3113` route exists and early AUX writes succeed, does Windows preserve the real live sink, rebuild it from the durable AUX transport, or allow a fake/stale logical sink to stand in?

Relevant path:

- `DetectDisplayAndApplyApple5kOverrides` calls `DetectDisplayEarlyExitForApple5kCaps`, then the normal detect/status helpers, then applies an Apple/tiled tag `0x3B` override.
- `DetectDisplayEarlyExitForApple5kCaps` returns handled when pin descriptor byte `+6` has bit `2` or bit `3`, matching capability tags `0x3A/0x3B`.
- In the normal path, `ProbeDisplayStatusMaybeAssert4F1` first validates the current display object against the durable display-map DPCD/AUX backend before status handling and possible `0x4F1`.
- Detection results flow into `HandleDisplayDetectionResultAndLiveState`.
- `HandleDisplayDetectionResultAndLiveState` calls `ApplyDetectionResultAndUpdateLiveDisplayObject` only when:
  - `result.connected != current_object.connected`,
  - the result change/rescan byte is set,
  - or the signal is an embedded-panel signal.
- `ApplyDetectionResultAndUpdateLiveDisplayObject` eventually calls `UpdateDisplaySinkConnectionAndLiveAuxState`.
- `UpdateDisplaySinkConnectionAndLiveAuxState` calls `UpdateDisplayObjectRecordLiveSinkObject`.
- `UpdateDisplayObjectRecordLiveSinkObject` updates the 104-byte detection/display-object record:
  - if the object is connected/usable, it stores the current display object at record `+0x18`,
  - if not, it clears record `+0x18`.
- This live-object record is distinct from the durable 64-byte display-map entry whose `+0x18` is the DPCD/AUX transport.

The key Apple/tiled preservation branch:

- In `DetectDisplayAndApplyApple5kOverrides`, after normal detect/status work, Windows checks:
  - pin descriptor byte `+6`, bit `3` / tag `0x3B` is set,
  - the display object's current connected/physical-present vfunc returns `1`,
  - the new detection result says not connected,
  - the display signal/type is `11` / DisplayPort-family in this layer.
- If all are true, Windows forces:
  - `result.connected = 1`,
  - `result.changed = 0`.
- Then `HandleDisplayDetectionResultAndLiveState` sees no connection change and no rescan/change flag, so it avoids `ApplyDetectionResultAndUpdateLiveDisplayObject`.
- Avoiding that call prevents `UpdateDisplayObjectRecordLiveSinkObject` from clearing the live display-object record.

RE-7 conclusion:

- Windows does not solve this by keeping a fake/stale sink after disconnect.
- It also does not blindly reconstruct the live sink from primary-tile metadata.
- The tag `0x3B` path is subtler: if the secondary DisplayPort object is still physically/currently connected but a detect pass reports disconnected, Windows suppresses that false disconnect so the real live object is not torn down.
- This maps very directly to the Linux failure after successful early `0x310`/`0x4F1`: Linux should not replace `DP-1` with an emulated sink; it should narrowly suppress or defer the false disconnect/teardown for the proven real iMac `0x3113` route when the Windows-equivalent conditions hold.
- The preservation condition should be strict:
  - iMac19,1 / Apple internal 5K panel fingerprint,
  - connector route matches `0x3113` / DDC3 / UNIPHY_D,
  - the secondary route was physically live and AUX-writable earlier in the boot,
  - source-DPCD and/or `0x4F1` succeeded on that same route,
  - tile metadata/grouping exists or can be recovered from the real secondary EDID/DisplayID path.
- This makes the next kernel-side shape clearer: preserve the real secondary sink/link object through a transient false disconnect, then fix userspace-facing tiled grouping, rather than building an emulated sink fallback.

### 2026-05-08 RE-8 pass: cold-boot enablement boundary

Target question:

- Can the Windows runtime driver create the full iMac 5K state from a plain 4K-like boot without Apple preinit, and if so which non-Apple sequence does Linux need to reproduce?

Static evidence:

- `DetectDisplayMainMaybeApple5k` normalizes the detect pass, calls `DetectDisplayAndApplyApple5kOverrides`, and then either bypasses live-state update for a special handled path or calls `HandleDisplayDetectionResultAndLiveState`.
- That live-state update path is the same path from RE-7 that can clear or preserve the current live display object. Therefore cold-boot enablement and post-detect preservation share the same critical boundary: whether Windows gets a real current display object for the secondary route and avoids tearing it down on a false disconnect.
- `PopulateDisplayMapAuxObjectsForHotplugRange` builds display-map entries from firmware/object-table display object ids, creates a per-object display-map interface, creates a `DPCD/AUX` interface for the same object id, and stores that durable transport at the 64-byte display-map entry `+0x18`.
- `CreateDisplayMapInterfaceForObjectIdMaybeSkipInternal` accepts class-3 display object ids, so both `0x3114` and `0x3113` are in the accepted object-id family.
- The same helper has a policy that can skip low-byte object ids `0x14` and `0x0E` when a base-pin AdapterService property bit is set, but it does not skip low-byte `0x13`. This is important because the iMac secondary tile route is `0x3113`.
- The source-DPCD and sink-private activation sequence from RE-6 uses the durable display-map `DPCD/AUX` transport. It does not require an Apple EFI runtime call once that transport exists.
- No static evidence in this pass shows Windows calling an Apple GOP/EFI runtime service or a separate firmware mailbox to create the runtime 5K state. The visible Windows sequence is object-table route discovery, durable AUX creation, DisplayID/EDID detection, source-DPCD publication, link training/stream enable, and `0x4F1` assertion.

What static RE still cannot prove:

- Static RE does not prove that Windows can conjure a secondary tile if the `0x3113` route is completely absent from the runtime object table/link topology.
- The Windows path is better described as "construct and preserve the secondary object from discoverable AMD object-table/AUX/link ingredients" than "create a fake secondary display from nothing."
- This matters for Linux plain boot: Stage 2 should try to discover or construct the real `0x3113` AUX/link route early enough for normal detection, not just expose a fake logical 5K monitor with no real secondary backend.

RE-8 conclusion:

- The best current model is that Windows does not need Apple binaries at runtime, but it does need the non-Apple AMD display-object ingredients: `0x3113`, its AUX/DDC backend, its tile EDID/DisplayID metadata, and the DPCD activation sequence.
- This supports the long-term goal of replacing OCLP/Apple preinit, but the Linux implementation must become capable of making the lower `0x3113` route live from the AMD/VBIOS/object-table side or preserving it if firmware/probe boot already made it live.
- For the immediate GNOME target after OCLP/probe boot, RE-8 does not change the priority: keep the real secondary route alive and expose correct tiled-monitor metadata. Cold-boot synthesis remains a follow-on once the preserved two-tile state works.

### 2026-05-08 RE-9 pass: Linux correlation for GNOME-visible 5K

Target question:

- Given RE-1 through RE-8, which current Linux changes are aligned with Windows, which are risky, and what should the next kernel patch focus on so GNOME sees one 5K tiled monitor?

Current Linux changes that still match the RE model:

- The iMac quirk constants match the Windows object model: primary `0x3114` / selector `0x4875` / DDC4 / UNIPHY_C, and secondary `0x3113` / selector `0x4871` / DDC3 / UNIPHY_D.
- The secondary source-DPCD path is aligned with Windows RE-6: write AMD source-specific DPCD, especially `0x310 = 04 1D 03` for this DCE generation, before or around stream training/commit.
- The `0x4F1 = 1` write is correctly treated as a sink-private DPCD write over AUX to the panel/TCON, not as a firmware mailbox opcode.
- The current direction of preserving the real previous `dc_sink` is aligned with RE-7, provided it remains a strict false-disconnect suppression path and not a broad fake-sink fallback.
- Copying or applying TILE metadata to the secondary connector is aligned with RE-3/RE-5, because GNOME/KDE need a DRM tile group that describes two physical tiles as one monitor.

Linux gaps or suspicious areas after the RE pass:

- The preservation condition is still less precise than the Windows tag-`0x3B` behavior. Windows suppresses a disconnect only when the object is still physically/currently connected and the detection result is the questionable part. Linux should remember concrete evidence that the secondary route was real and AUX-writable earlier in the boot, such as successful source-DPCD or `0x4F1` on the `0x3113` route.
- If GNOME still sees stretched or half-visible output, the next suspect is not a hidden DCE split pipe. The stronger suspect is the DRM tiled-monitor contract: same tile-group id, `tile_is_single_monitor = true`, correct `2x1` topology, primary tile at `0,0`, secondary tile at `1,0`, both `2560x2880`, and tile properties installed before userspace builds its monitor layout.
- The current Linux path should avoid the old emulated-sink fallback. Windows preserves a real object; it does not solve the iMac 5K path by inventing a fake disconnected sink.
- Plain boot will not be fixed by only cloning TILE metadata if the real lower `0x3113` route never becomes live. Stage 2 needs the RE-8 object-table/AUX construction angle, not just a userspace-facing connector property.
- The optional Windows `DPCD 0x111` bit0 step remains a secondary activation candidate, but the evidence still says the main visible failure after OCLP/probe boot is grouping/preservation, not lack of a mysterious firmware enable opcode.

Concrete next kernel direction:

- First tighten the real-secondary preservation path to require explicit runtime evidence that `0x3113` was live/AUX-writable, then suppress only the false disconnect/teardown that would clear that real sink.
- In the same patch, make the TILE property application unambiguous for GNOME: same tile group on both connectors, `single_monitor` true, `2x1` layout, left tile `0,0`, right tile `1,0`, and enough logging to prove userspace received those properties before the first destructive modeset.
- Do not add a DCE cross-connector split-pipe hack unless runtime logs show Linux had that relationship in the good pre-GUI state.
- Do not reintroduce an emulated sink fallback for the main path. If a synthetic path is ever needed for cold boot, it should be a separate Stage 2 experiment backed by real `0x3113` AUX/link construction from the AMD object table.

IDA breadcrumbs saved for this pass:

- `0x1414B8706`: display-map interface creation from object-table ids.
- `0x1414B875E`: durable per-object DPCD/AUX transport stored at display-map entry `+0x18`.
- `0x1415524B4`: class-3 object-id acceptance for `0x3114` / `0x3113`.
- `0x141552658`: low-byte skip policy excludes `0x14` / `0x0E`, not `0x13`.
- `0x1414A9850`: detect/live-state update gate that maps to Linux real-sink preservation.

### 2026-05-08 RE-B: DPCD `0x111` / `0x205` checkpoint

This pass rechecked the two small DPCD questions in `amdkmdag.sys.i64` instead of relying on memory from the earlier `0x4F1` work.

IDA functions checked:

- `SetDpcdMstCtrlEnableBit` = `0x141B8E9C0`
- `BringUpDpOrEdpWithLinkTraining` = `0x141B6CA70`
- `ClassifyDpLinkRateEncodingRange` = `0x141B8C880`
- `ProgramDpPreTrainingAuxState` = `0x141B8E090`
- `StreamEnableWrite4F1AndPollSinkStatus` = `0x1414E7810`

Findings:

- `SetDpcdMstCtrlEnableBit` directly reads DPCD address `273` / `0x111`, sets or clears bit `0`, then writes `0x111` back.
- In `BringUpDpOrEdpWithLinkTraining`, the set-bit path runs before `ProgramDpPreTrainingAuxState` and before link training.
- The relevant gate is `ClassifyDpLinkRateEncodingRange(link_settings) == 2` and signal `32`.
- `ClassifyDpLinkRateEncodingRange` returns class `2` when the link-settings rate field is in `1000..2000`; class `1` is `6..30`.
- This keeps DPCD `0x111` bit `0` as a plausible Windows-like secondary `0x3113` step, but only if the real iMac secondary route is signal `32` and class `2` at runtime. It should not be a generic Linux DP write.
- `StreamEnableWrite4F1AndPollSinkStatus` confirms the DPCD `0x205` poll is real but separately gated.
- The stream-enable `0x4F1 = 1` write is authorized by descriptor byte `+6` bit `2` or bit `3`, matching the `0x3A` / `0x3B` family already tied to `APP/AE25` and `APP/AE26`.
- The follow-up `0x205` poll is gated by descriptor byte `+6` bit `5`, previously mapped to tag `0x40`.
- The known static Apple `AE25` / `AE26` evidence does not prove tag `0x40`, so `0x205` should remain diagnostic-only unless runtime data proves that tag.

Kernel implication:

- If we make another DPCD activation patch, the high-signal candidate is a tightly gated `0x111` read/modify/write on the proven real `DP-1` / `0x3113` route before source-DPCD and training.
- Do not add an unconditional `0x205` poll; Windows treats it as a capability/tag-specific status wait, not as a universal post-`0x4F1` dependency.

### 2026-05-08 RE-A: object-table AUX construction mapped to Linux link creation

Target question:

- Does Windows create a real durable secondary `0x3113` AUX route from AMD object-table data, and where should Linux have the equivalent route?

IDA functions checked:

- `ConstructDpcdAuxInterfaceForObjectId` = `0x141551090`
- `AdapterServiceGetAuxHandleForObjectId` = `0x141465AA0`
- `AdapterServiceQueryObjectDescriptor` = `0x1414663A0`
- `AuxFactoryCreateHandleForSourceMask` = `0x14152AE80`
- `ConstructDualPinAuxTransactionHandle` = `0x1415B0C20`
- `ResolveSourceMaskToPinIndex` = `0x141529FB0`
- `ConstructAuxTransactionPathFactory` = `0x14152B8B0`
- `CreateAuxPinMapFactoryByGeneration` = `0x1415AFB70`
- `ConstructDce11AuxPinMapFactory` = `0x1415FE160`
- `Dce11ResolveAuxSelectorToPinIndex` = `0x141695990`
- `InitializeAdapterServiceForConnector` = `0x141467800`

Windows route construction result:

- `ConstructDpcdAuxInterfaceForObjectId` calls AdapterService vfunc `+0x170` to get the lower AUX handle for the exact display object id.
- AdapterService vfunc `+0x170` is `AdapterServiceGetAuxHandleForObjectId`.
- `AdapterServiceGetAuxHandleForObjectId` queries the object descriptor and uses:
  - descriptor `+0x1C` as the AUX selector,
  - descriptor `+0x3C` as the source index used to build `source_mask = 1 << value`.
- For the iMac paired 5K paths:
  - secondary object `0x3113` uses selector `0x4871`,
  - primary object `0x3114` uses selector `0x4875`.
- The selector/source-mask pair is passed to `AuxFactoryCreateHandleForSourceMask`.
- That creates `ConstructDualPinAuxTransactionHandle`, which resolves the selector/source mask to a concrete pin index, then opens both endpoint type `3` and endpoint type `4`.
- Endpoint type `3` is the DDC DATA endpoint.
- Endpoint type `4` is the DDC CLOCK endpoint.
- The handle is invalidated if either concrete endpoint is absent.
- Therefore Windows requires a real lower DDC/AUX backend for `0x3113`; it is not satisfied by keeping only a logical display object.

Selector mapping result:

- `ConstructAuxTransactionPathFactory` creates the AUX path factory and stores the selector/source-mask resolver at factory `+0xB0`.
- Generation `11` uses `ConstructDce11AuxPinMapFactory`.
- `Dce11ResolveAuxSelectorToPinIndex` has literal mappings:
  - `0x4869 -> 0`
  - `0x486D -> 1`
  - `0x4871 -> 2`
  - `0x4875 -> 3`
  - `0x4879 -> 4`
  - `0x487D -> 5`
  - `0x4881 -> 6`
  - `0x4899 -> 7`
- Therefore:
  - secondary `0x3113` / selector `0x4871` maps to AUX pin index `2`;
  - primary `0x3114` / selector `0x4875` maps to AUX pin index `3`.

AdapterService factory precondition:

- `InitializeAdapterServiceForConnector` creates the AUX transaction factory at AdapterService `+0x1A8`.
- If base-pin property `6` bit `4` says AUX is disabled, AdapterService leaves this factory null.
- If the factory is null, `AdapterServiceGetAuxHandleForObjectId` cannot create object-specific lower AUX handles such as the `0x3113` secondary route.

Linux correlation:

- `dc/core/dc.c::create_links()` gets connector count from the BIOS parser, then loops through BIOS object-table connector ids and calls `create_link()`.
- `dc/bios/bios_parser2.c::bios_parser_get_connectors_number()` counts display paths with nonzero encoder object ids.
- `dc/bios/bios_parser2.c::bios_parser_get_connector_id()` returns each path's `display_objid`.
- `dc/link/link_factory.c::construct_phy()` stores that id in `link->link_id`, queries the source/encoder object, maps it to a transmitter, creates the DDC service, records `ddc_hw_inst`, builds HPD state, assigns connector signal, and creates the link encoder.
- `dc/link/protocols/link_ddc.c::ddc_service_construct()` calls `get_i2c_info()` for the connector object and creates the `ddc_pin`.
- `dc/bios/bios_parser2.c::bios_parser_get_i2c_info()` and `get_gpio_i2c_info()` map the ATOM I2C record through the GPIO pin LUT. This is the closest Linux equivalent to Windows' selector/source-mask to concrete DDC/AUX pin materialization.

RE-A conclusion:

- Windows builds the secondary iMac 5K tile as a real object-table route:
  - object id `0x3113`,
  - descriptor selector `0x4871`,
  - DCE11 AUX pin index `2`,
  - concrete DDC DATA and DDC CLOCK endpoints.
- The Linux plain-boot question should now be framed as: does Linux construct and keep a real `dc_link` for `0x3113` with the Windows-equivalent DDC3 / UNIPHY_D / HPD route?
- If yes, the problem is later detect/preservation/activation.
- If no, Stage 2 must create or expose that real route from AMD/VBIOS ingredients before normal detection.
- TILE metadata alone cannot solve Apple-free cold boot if the real secondary `0x3113` lower AUX/link backend never exists.
