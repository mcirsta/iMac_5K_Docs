# iMac19,1 5K Current Findings

Status: curated from `RE_5K_iMac19_1.md` on 2026-05-08.

This file is the current working truth set. The original `RE_5K_iMac19_1.md` is still the raw lab notebook and provenance trail, but it contains older hypotheses that were later corrected. Before kernel work, use this file first.

## How To Treat The Old Notebook

`RE_5K_iMac19_1.md` is useful for detailed evidence, but it is no longer safe as a direct implementation spec.

The newest reliable chain starts around these old-file anchors:

- `2026-03-26 Probe-Boot Milestone`: firmware/probe boot proves Linux can inherit a richer two-tile state.
- `2026-04-29 Windows RE Checkpoint`: decimal `1265` resolves to sink DPCD address `0x4F1`.
- `2026-05-01 Linux patch update`: fake/emulated sink preservation was removed from the intended path.
- `2026-05-07/08 Windows RE passes`: grouped display validation, real tile streams, real tile metadata, and display-map transport lifetime.
- `RE-1..RE-9`: current Windows-to-Linux model and next Linux direction.

Older sections remain valuable as history, but conclusions involving fake sinks, one synthetic 5K stream, or magical firmware opcodes should be considered superseded unless restated here.

## Current Goal

There are two separate goals.

Goal 1 is the near-term probe/OCLP boot target:

- Boot through the already-known diagnostics/probe path.
- Keep the real two-tile 5K state alive through the first Linux DRM/KMS userspace takeover.
- Make GNOME/KDE see one logical tiled 5120x2880 monitor, not two unrelated 2560x2880 monitors.

Goal 2 is the long-term Apple-free target:

- Boot only systemd-boot or another normal bootloader plus Linux.
- Have Linux discover or construct the real secondary tile route itself.
- Reach the same final Linux KMS state without OCLP, Apple diagnostics binaries, or a pre-systemd shim.

Goal 1 should be solved first because it gives the stable Linux model that Goal 2 must recreate.

## Current Hardware And Route Model

The iMac19,1 internal 5K panel is treated as two real physical tile paths.

- Primary tile: `0x3114`, Linux `eDP-1`, left tile.
- Primary route details: DDC4, DCE DDC A register `0x4875` / `mmDC_GPIO_DDC4_A`, Windows AUX pin index `3`, transmitter `UNIPHY_C`.
- Secondary tile: `0x3113`, Linux `DP-1`, right tile.
- Secondary route details: DDC3, DCE DDC A register `0x4871` / `mmDC_GPIO_DDC3_A`, Windows AUX pin index `2`, transmitter `UNIPHY_D`.
- Tile geometry: two columns, one row.
- Left tile location: `0,0`.
- Right tile location: `1,0`.
- Per-tile mode: `2560x2880`.
- Logical monitor: `5120x2880`.

The canonical tile order is primary/eDP left and secondary/DP right.

## Magic Number Registry

Use these names before treating the values as unexplained magic:

- `0x3113`: AMD DAL graphics object id for connector type DisplayPort, enum 1, object type connector. This is the secondary/right tile route.
- `0x3114`: AMD DAL graphics object id for connector type eDP, enum 1, object type connector. This is the primary/left tile route.
- `0x4871`: DCE DDC3 A register (`mmDC_GPIO_DDC3_A`). Windows also carries this as the AUX selector for secondary `0x3113`, then resolves it to AUX pin index `2`.
- `0x4875`: DCE DDC4 A register (`mmDC_GPIO_DDC4_A`). Windows also carries this as the AUX selector for primary `0x3114`, then resolves it to AUX pin index `3`.
- `32`: Linux/AMD `SIGNAL_TYPE_DISPLAY_PORT`.
- `128`: Linux/AMD `SIGNAL_TYPE_EDP`.
- `0x100`: DP DPCD `DP_LINK_BW_SET`.
- `0x101`: DP DPCD `DP_LANE_COUNT_SET`.
- `0x107`: DP DPCD `DP_DOWNSPREAD_CTRL`.
- `0x10A`: DP/eDP DPCD `DP_EDP_CONFIGURATION_SET`; bit 0 is ASSR / alternate scrambler reset.
- `0x111`: DP DPCD `DP_MSTM_CTRL`; bit 0 is MST enable.
- `0x205`: DP DPCD `DP_SINK_STATUS`.
- `0x316`: source-device-specific DPCD range; Windows treats bit 0 as a conditional stream/source flag for stream mode values `10`, `11`, or `12`.
- `0x4F1`: sink-device-specific/private DPCD range; on this panel Windows writes `1` as an Apple/tiled panel latch. Decimal `1265` is this address, not a firmware mailbox opcode.

AtomBIOS/AMD headers explain object ids, DDC/I2C routing, GPIO/DDC registers, and transmitter names. They do not explain Apple/private sink DPCD `0x4F1` or Apple capability tags; those remain Windows RE plus runtime-observation facts.

## Probe-Boot Observations

The diagnostics/OCLP-style boot path has proven that preboot firmware can hand Linux a richer display state.

Observed from probe runs:

- GOP handles reported `5120x2880` during preboot probing.
- Apple shared display protocol `63FAECF2-E7EE-4CB9-8A0C-11CE5E89E33C` was present.
- AMD connector protocol `CF611019-A212-4075-83A7-39F7364DEA6B` was present.
- Linux pre-GUI/fbcon state could contain two active `2560x2880` scanout halves.
- The pre-GUI framebuffer could be one full-width `5120x2880` framebuffer.
- The failure happens later when real DRM userspace takes over, not simply at firmware handoff.

Observed bad graphical states:

- KDE Wayland can see two displays instead of one tiled monitor.
- GNOME can expose a 5K-ish logical mode but still stretch or lose the right half.
- Some captures showed both TILE blobs present yet GNOME still using two independent `2560x2880` framebuffers.

Important interpretation:

- TILE metadata is necessary but not sufficient.
- The final userspace/atomic state must keep both tile streams together as one logical monitor/framebuffer.
- A successful pre-GUI state proves Linux can drive the hardware; the remaining problem is preserving and presenting that topology correctly.

## Firmware RE Model

The Apple firmware path is useful as evidence, but not the final desired dependency.

Current firmware-side model:

- OCLP does not appear to implement its own 5K display driver in `diags.efi`.
- OCLP enters an Apple diagnostics/product-driver path that preserves or triggers richer Apple/AMD preboot display initialization.
- `CoreEG2.efi` is the Apple-side orchestrator for complex display setup.
- The Polaris AMD GOP is the lower AMD-side provider for device and connector execution.
- `CoreEG2` installs or uses Apple shared display/platform provider protocols.
- The AMD GOP installs the private connector protocol `CF611019-A212-4075-83A7-39F7364DEA6B`.
- `CoreEG2` opens that AMD connector protocol and drives the lower connector backend.
- The important preboot success path constructs a complex-display blob and makes it live for the lower backend.
- `CoreEG2` publishes Apple shared graphics protocol `63FAECF2-E7EE-4CB9-8A0C-11CE5E89E33C`; the GOP locates this protocol and uses it as a cross-module backchannel.
- GOP shared-graphics calls can fetch platform info and connector-record sets from CoreEG2 display objects.
- In the GOP saved-config path, `GopProbeConnectorRecordCachesFromDisplayTable` can seed GOP connector-record caches from CoreEG2's cached display records. This is a plausible way an Apple-looking/OCLP-style path preserves richer state, but it is not the `dword_E670` platform-policy source.
- The GOP also publishes `APPLE_GOP_DEVICE_PROTOCOL_GUID = 53CFEB0D-9B9E-4D0A-90B6-024A79906A58`; CoreEG2 has a driver binding for this protocol and stores the opened GOP device interface in its device node.
- GOP installs the device protocol first, then the per-connector `CF611019-...` protocol and the framebuffer/root-display `F977B8AE-...` protocol. CoreEG2 waits until both expected child counts are present before running `CoreEg2ComplexDisplayInit`.
- CoreEG2's successful paired path builds a 384-byte complex-display replay record and calls GOP device-protocol slot `+0x40`, identified as `GopReplaySavedDisplayConfiguration`. Cleanup/failure calls the same slot with `NULL`.
- That 384-byte replay record is now decoded enough to model the successful 5K handoff:
  - header at `record+0x000`: version `0x10000`, endpoint count `2`, endpoint pointer `record+0x120`, global-timing pointer `record+0x018`
  - global/logical timing at `record+0x018`: version `0x10001`, normal logical mode `5120x2880`, pixel-clock-ish value `938250000`
  - tile timing blocks at `record+0x070` and `record+0x0C8`
  - endpoint descriptors at `record+0x120` and `record+0x150`, each 48 bytes, with connector id at `+0x04`, viewport at `+0x1C/+0x20`, and tile-timing pointer at `+0x28`
- Normal 5K success shape is endpoint0 = root connector at viewport `0,0`, endpoint1 = `root connector id + 1` at viewport `tile_width,0`. GOP then maps each connector id back to a live connector object and derives the backend mask from that object, not from the replay record.
- If `dword_E670 & 4` is set, CoreEG2 chooses the lower 4096x2304-style logical timing instead of the normal `5120x2880` timing. For iMac19,1 the platform DB evidence still says `dword_E670 = 1`, so the normal path should be the expected one.
- GOP replay consumer recheck: `GopReplaySavedDisplayConfiguration` seeds GOP multi-display/tile state, publishes connector/display properties, and calls `GopAdjustViewportForTiledDisplayLayout`. That viewport helper emits GOP register/window programming through `GopMmioRead32Indexed` / `GopMmioWrite32Indexed`; it does not perform connector mailbox or DPCD/link-training transactions.
- Training-order clarification: CoreEG2 performs explicit `CDLinkTrainDPConnector1` and `CDLinkTrainDPConnector2` calls after the replay call succeeds. Both calls go through the graphicsconnector backend slot `+0x28`, identified as `GopGraphicsConnectorBackendApplyOutputMask`, with the same `display+0x160` DP training request. Root and sibling are therefore trained/applied as two real connector backends, not as one synthetic 5K sink.
- `display+0x160` training request shape: `+0x08` is link-rate class, `+0x0C` is lane count, `+0x30..+0x32` are policy flags, and `+0x34` is optional custom link rate. `CoreEg2SetDpTrainingRequestFromTiming` maps `1620/2700/5400/8100` to classes `0/1/2/3`; unknown/custom rates use class `-65536`.
- GOP `GopPrepareDpTrainingStateForSelectedMask` parses that request into the connector-state slot. For grouped `0xF00`, it enables grouped-live/custom-rate behavior and accepts special custom rates `21600`, `24300`, `32400`, and `43200` only when grouped-live state is present.
- Non-NULL backend `+0x28` requests reach `GopTrainConnectorMaskWithDpTrainingLoop`, which retries `GopTrainDisplayPortAndPublishDpcd` up to 10 times. A single attempt programs DP training set `7`, sends a source/PHY training packet, optionally sends grouped training packet `0x10`, publishes DPCD training state, then programs training set `1`.
- Therefore the missing plain-boot step is earlier than the replay record but not earlier than all training: Linux first needs the secondary `0x3113` route alive/AUX-backed enough to validate and train; then it must perform the equivalent source-DPCD/pretraining/training sequence without losing HPD/AUX.

Current CoreEG2/GOP details that still matter:

- CoreEG2 reads platform display flags into `dword_E670` using provider key GUID `1823A1A6-0EAF-4F3A-93D3-BF61F5EEFA0D`.
- `dword_E670 & 1` gates the complex paired-display path before CoreEG2 tries the second/tiled connector path.
- That provider read is through protocol `AC5E4829-A8FD-440B-AF33-9FFE013B12D8`, published by `ApplePlatformInfoDatabaseDxe.efi`; the misleading old CoreEG2 callback label was corrected in IDA to `CoreEg2RegisterPlatformInfoDatabaseProvider`.
- The raw platform database maps selector `1823A1A6-...` to `j139 -> 5` and `j138 -> 1`; the same database maps `j139` to `iMac19,2` and `j138` to `iMac19,1`.
- Raw firmware recheck: the selector GUID byte pattern is not present in the PE32 module set. It appears in the platform-info raw freeform file `5E7BE016-33CF-2D42-8758-C69FA5CDBB2F`, raw section offset `0x78`, whose value block at `0x7A0` contains exactly the `j139 = 5` and `j138 = 1` rows. The AC5E provider GUID appears in many PE modules because it is the database service protocol; the selector value itself is static platform database data.
- Therefore, for the target `iMac19,1`, the best current evidence is `dword_E670 = 1`: bit0 should be set. Treat "plain boot has bit0 clear" as unproven and probably wrong unless a direct firmware trace/log proves it.
- The grouped `0xF00` path is real on the GOP side, not just an Apple-side label.
- The lower connector backend has a grouped/live subtype `20` / `0x14`.
- `CoreEG2`'s paired-root success path correlates with that grouped subtype.
- `CoreEG2` reaches GOP's `GopGraphicsConnectorBackendDispatchOpcode` for full-opcode connector backend operations.
- GOP backend mode `a4 == 1` preserves full opcodes up to `0xFFFFF`; CoreEG2 decimal `1265` is therefore sent as full `0x4F1`, not truncated to `0xF1`.
- GOP's extended opcode transport is also used for normal DP DPCD training addresses (`0x100`, `0x101`, `0x102`, `0x103`, `0x107`, `0x108`) and training-status readback (`0x202`).
- GOP has an optional extended `0x10A` write path gated by internal Apple/panel flags, which reinforces the current secondary-route ASSR/panel-policy direction.
- CoreEG2's own training-status checker reads DPCD `0x202`, length `2`, through the same backend transfer and expects `0x77/0x77` for four lanes, `0x77` for two lanes, or `0x07` for one lane.
- GOP connector id `1` maps to backend mask `0x100` / record-cache index `4`; connector id `2` maps to backend mask `0x200` / record-cache index `3`. CoreEG2's sibling rule is `root connector id + 1`, so this is the firmware-side root/sibling pair.
- CoreEG2's final paired-display decision still depends on live connector-record validation after `0x4F1 = 1`: sibling record bytes `+8/+9 == 0x0610`, byte `+0x0B == 0xAE`, and `(flags at +0x0A & 3) == 2`.
- The final firmware-side 5K commit point is not OCLP code replacing CoreEG2; it is CoreEG2 reaching `GopReplaySavedDisplayConfiguration(record)` during DXE driver-binding connect.
- `display + 0x25C` / `display + 604` is a key live flag/gate for whether complex-display data is consumed later.
- `CoreEG2ConfigureAndActivateDisplayPipeline` is a hard consumer of the success-side complex-display record.
- The strongest static boundary reached was that selector/class bytes come from raw lower-transport results.

What firmware RE does not prove:

- It does not prove Linux needs Apple EFI runtime calls after ExitBootServices.
- It does not prove `1265` is a magical mailbox opcode that Linux must replay.
- It does not prove a one-sink fake 5K abstraction is the hardware model.
- It does not prove the OCLP/plain difference is `dword_E670` bit0. The stronger current suspect is later CoreEG2/GOP sibling connector discovery/validation after the root-side `0x4F1` latch sequence.

## Windows Driver RE Model

The Windows driver model is now the strongest guide for Linux.

Windows does not appear to solve iMac 5K by presenting one fake `5120x2880` sink to DC.

Current Windows model:

- Windows uses two real tile display objects.
- Windows uses two real streams and two real plane states in grouped/multi-display validation.
- The logical one-monitor 5K behavior is produced above or around DC by grouped/tiled display-manager state.
- DC/resource validation still receives real per-tile streams underneath.
- Per-tile stream size is `2560x2880`.
- The final logical view is `5120x2880`.
- Right-half placement is not a simple hardcoded source-X literal found in the final validation pass.
- The grouped path validates that active displays share tiled group metadata.
- Each endpoint is expected to have valid tile metadata from its own DisplayID/EDID parser.

Important Windows functions and concepts:

- `ConstructMultiDisplayViewCommitContext`: allocates one context entry per physical display/tile.
- `ValidateMultiDisplayViewContextAddStreamsPlanesFinalize`: adds streams/planes for each tile and finalizes view/resource validation.
- `PopulateEndpointTileMetadataFromDisplayIdTopology`: fills endpoint tile metadata from the display object's own DisplayID parser.
- `ValidateDisplaysShareTiledGroupMetadata`: requires active endpoints to share the same tile group token and occupy valid slots.
- `DetectDisplayAndApplyApple5kOverrides`: applies special Apple/tiled detect behavior.
- `HandleDisplayDetectionResultAndLiveState`: update gate that can preserve or clear the live display object.
- `PopulateDisplayMapAuxObjectsForHotplugRange`: builds display-map AUX objects from object-table display ids.
- `CreateDisplayMapInterfaceForObjectIdMaybeSkipInternal`: accepts class-3 display ids, including `0x3113` and `0x3114`.

## DPCD And The `1265` Correction

Decimal `1265` is hexadecimal `0x4F1`.

Current meaning:

- `0x4F1` is a sink-private DPCD address reached over DP AUX.
- The receiver is the panel/TCON side, not a generic EFI mailbox endpoint.
- The important payload observed is one byte, value `1`.
- Windows writes `0x4F1 = 1` on the real object-specific AUX route.
- Firmware/CoreEG2 sends the same value through GOP's full-opcode connector backend: `GopGraphicsConnectorBackendDispatchOpcode` -> `GopWriteExtendedOpcodeWindow` -> connector mailbox transaction.
- The GOP backend preserves decimal `1265` as full `0x4F1`; it is not a low-byte mailbox command.
- Linux should still model this as a DPCD/AUX operation because GOP uses the same extended transport for ordinary DPCD training addresses.

Current DPCD ordering model:

1. Preserve or reacquire the real object-specific AUX route for `0x3113`.
2. Apply the Windows-like ASSR/panel policy for the exact secondary object low byte `0x13` / route `0x3113`: set DPCD `0x10A` bit `0` before training and pass the same policy into training-pattern programming.
3. If applicable for signal `32` / class `2`, set DPCD `0x111` bit `0`; `obs_10` rate `20` is class `1`, so this is currently diagnostic rather than a next-patch requirement.
4. Publish source-specific DPCD before normal caps/training.
5. For this DCE generation, write source-DPCD `0x310 = 04 1D 03`.
6. Run normal link training and stream enable.
7. Write sink-private DPCD `0x4F1 = 1`.
8. Keep `0x316`, `0x32F`, and `0x205` observational unless runtime evidence proves the corresponding Windows gates are active on this panel.

Linux experiments already proved that the real `0x3113` route can write source-DPCD `0x310 = 04 1D 03` and `0x4F1 = 1` while the route is live.

Therefore the current failure is not "we never found the magic firmware opcode." After `obs_10`, the most concrete remaining lower-link difference is that Linux still skips the pre-training `0x10A` ASSR step because `panel_mode_edp=0`, while Windows has an object-route policy for the secondary `0x3113` path.

## DisplayID And Tile Metadata

Windows appears to expect every grouped tile endpoint to have valid tile metadata.

Current metadata model:

- The primary eDP DisplayID block contains a tiled display topology block.
- Captured primary TILE data describes a `2x1` display.
- Primary tile location is `0,0`.
- Tile size is `2560x2880`.
- DisplayID tiled serial observed in notes: `0x04E78A8B`.
- Windows parser paths support DisplayID 1.x tiled topology tag `0x12` and DisplayID 2.0 tiled topology tag `0x28`.
- Windows does not appear to reconstruct the secondary endpoint's tile metadata from the primary endpoint in the path we traced.
- The Windows peer scan can elect a representative/primary tile, but it does not copy the full tile metadata block from one endpoint to another.
- If a DisplayID tiled topology block is not found in the current parser object, Windows can ask the next EDID/DisplayID extension parser in the same display object's parser chain. This is extension-chain recovery, not cross-display/primary-to-secondary recovery.

Linux implication:

- Best target is to preserve or reacquire the secondary route's own EDID/DisplayID/tile metadata.
- If the secondary EDID cannot be read reliably, reconstructing the secondary TILE property from the primary tile is a Linux workaround, not something proven to be the Windows method.
- That workaround must remain gated to the real `0x3113` route.
- The latest runtime logs already showed an early real secondary `DP-1` sink with `sink_edid_len=256`, followed later by `local_sink=0`. That makes "snapshot/preserve the real secondary EDID/tile metadata before teardown" a better Linux target than "invent secondary metadata from primary" when the probe/OCLP path has made the secondary route live.

## Current Linux Kernel Direction

Current Linux code direction that matches RE:

- Keep the iMac19,1 quirk narrowly gated.
- Match primary/secondary route identities to the Windows object model.
- Program source-DPCD early enough on the secondary route.
- Write `0x4F1 = 1` while the real secondary route is live.
- Preserve the real secondary `dc_sink` only when it is truly the previously live route.
- Apply DRM TILE properties so GNOME/KDE can group the two physical connectors.
- Avoid fake/emulated sink preservation in the main path.

The next kernel patch should focus on the Windows-like secondary ASSR/panel policy first.

- Remove the `panel_mode_edp` dependency for the exact proven secondary route: object id `0x3113`, DDC3 / hw instance `2`, UNIPHY_D.
- Force `DP_PANEL_MODE_EDP` / ASSR behavior for that route early enough that normal AMD link training writes DPCD `0x10A` bit `0` before training.
- Remove the no-write guard in the explicit secondary `0x10A` helper for that same route; if AUX is live and the old value lacks bit `0`, write it anyway and log old/new/verify.
- Keep `0x111`, `0x205`, `0x316`, and `0x32F` as diagnostics unless runtime evidence proves their Windows gates are active on the panel.
- Keep the already implemented preservation and TILE contract work, but do not expand it into fake sinks or generic HPD overrides.

Implemented after `RE-LINK-CLASSIFY`:

- `link_edp_panel_control.c::dp_get_panel_mode()` now returns `DP_PANEL_MODE_EDP` for the exact Windows-equivalent secondary route even when `link->dpcd_caps.panel_mode_edp == 0`; the expected log string is `iMac5K secondary 0x3113 forced ASSR panel mode`.
- `amdgpu_dm_imac5k_program_secondary_assr_10a()` no longer skips the write when `panel_mode_edp == 0`; for the verified route it now writes DPCD `0x10A` bit `0` and logs `windows_route_policy=1`.
- The old active `0x111` clear path is now read-only diagnostic logging because the latest RE says the Windows `0x111` write/clear is class-gated and `obs_10` rate `20` is class `1`.
- DPCD snapshots now also read/log `0x32F` as diagnostic state; Linux still does not write `0x316`, `0x32F`, `0x205`, or `0x111` in this patch.

Do not add a DCE cross-connector split/ODM hack unless runtime logs prove Linux had such a relation in the good pre-GUI state.

## Plain Boot Direction

Plain boot is a separate Stage 2 problem.

Detailed backlog note: `C:\proj\reverse\WT6A_INF\RE_plain_boot_0x3113_backlog.md`.

Current RE-8 conclusion:

- Windows can build durable per-object AUX transports from AMD object-table display ids.
- `0x3113` is accepted as a class-3 object id.
- The low-byte skip policy can skip `0x14` and `0x0E`, but it does not skip `0x13`.
- This means the secondary iMac tile route is not filtered out by that Windows path.
- No static evidence shows Windows requiring Apple EFI runtime services to perform runtime DPCD/source-DPCD/`0x4F1` operations once the route exists.

What this means for Linux:

- Long-term Linux should discover or construct the real `0x3113` route from AMD/VBIOS/object-table ingredients.
- Plain boot will not be solved by only cloning TILE metadata if the real lower route never becomes live.
- A synthetic connector without real AUX/link backing is not the target solution.

## Retired Or Superseded Ideas

These ideas should not drive new code unless fresh evidence revives them.

- Fake/emulated secondary sink as the main solution.
- One fake `5120x2880` sink replacing the two physical tile streams.
- A magical private firmware opcode named `1265`; the practical operation is DPCD `0x4F1`.
- Blindly writing `0x4F1` on the primary AUX path.
- Treating TILE metadata alone as sufficient after GNOME still produces bad state.
- Forcing a DCE `next_odm` / split-pipe link between eDP and DP without runtime evidence.
- Treating display-map entry `+0x18` as both durable transport and live sink record. It is the durable per-object AUX/DPCD transport in the 64-byte display-map entry; the live display object slot is a different record.
- Treating AdapterService property13 bit17 as a global DAL option. It is tied to base-pin/platform capability handling.
- Treating MSLD/lid policy as the central 5K gate. It may still be platform context, but it does not explain the current Linux 5K failure.
- Treating role `8/9` labels as direct raw connector identities without the later corrected CoreEG2/GOP sequencing context.

## Remaining Unknowns

Still unknown or not fully proven:

- Exactly why GNOME can sometimes see a 5K-ish monitor but still stretch or lose the right half.
- Whether the Linux DRM tile property update timing is late relative to Mutter/KWin monitor construction.
- Whether Linux needs to reject or defer the first destructive userspace commit that drops the secondary tile stream.
- Whether DPCD `0x111` bit `0` is required for the secondary signal-32 path on this panel.
- Whether DPCD `0x205` polling matters for this panel; only add it if runtime evidence proves the relevant tag/status path.
- How to make plain Linux discover or construct the real `0x3113` route without Apple preboot.
- Whether the secondary endpoint's own EDID/DisplayID remains readable after userspace handoff. Static Windows RE now says the Windows path expects per-endpoint parser output; Linux-side primary-derived TILE reconstruction is only a narrow compatibility recovery when the real `0x3113` route was already proven.

## Numbered RE Tasks From Here

`RE-10`: GNOME/Mutter tiled-monitor contract.

- Goal: determine exactly what Mutter requires to merge `eDP-1` and `DP-1` into one logical 5120x2880 monitor.
- Local source checked: `C:\proj\reverse\WT6A_INF\mutter-main`.
- Local version: Mutter `50.1` source export, not a git checkout.
- Primary files to inspect:
  - `src/backends/native/meta-kms-connector.c`
  - `src/backends/native/meta-output-kms.c`
  - `src/backends/meta-output.c`
  - `src/backends/meta-monitor.c`
  - `src/backends/meta-monitor-manager.c`
  - `src/backends/meta-monitor-config-manager.c`
  - `src/backends/meta-monitor-config-store.c`
- Specific questions:
  - how Mutter reads the DRM `TILE` property,
  - how it stores tile group id, tile location, tile size, and `single_monitor`,
  - what makes it instantiate a tiled monitor instead of two normal monitors,
  - whether both tile outputs must have matching EDID/vendor/product/serial fields,
  - whether late TILE property changes require a hotplug/config rebuild,
  - why a configuration can appear as 5K but render/scan out only half or stretch.

RE-10 current result from the local Mutter source:

- `meta-kms-connector.c::state_set_tile_info()` reads the DRM `TILE` blob string as eight integers:
  - group id,
  - flags,
  - max horizontal tiles,
  - max vertical tiles,
  - horizontal tile location,
  - vertical tile location,
  - tile width,
  - tile height.
- `meta-output-kms.c` copies this KMS connector tile info directly into `MetaOutputInfo::tile_info`.
- `meta-monitor-manager.c` creates a `MetaMonitorTiled` only from a tiled output whose tile location is `0,0`.
- `meta-monitor.c::find_tiled_monitor_outputs()` groups all outputs on the same GPU with the same tile group id.
- `meta-monitor.c::verify_tiles_filled()` is the hard gate:
  - every output in the group must have nonzero group id,
  - every tile location must be inside the advertised grid,
  - `max_h_tiles * max_v_tiles` must equal the number of outputs in the group,
  - all outputs must share the same group id,
  - all outputs must share the same max tile grid,
  - all outputs must share the same tile width and height,
  - no two outputs may occupy the same tile slot.
- If this validation fails, Mutter will not create `MetaMonitorTiled`; the outputs can fall through as ordinary independent monitors.
- `meta-monitor.c::is_crtc_mode_tiled()` defines a tiled CRTC mode as a mode whose width and height exactly match `tile_w` and `tile_h`.
- `create_tiled_monitor_mode()` creates one logical monitor mode by finding a matching per-tile CRTC mode on every tile output, then summing tile sizes into the logical mode.
- For our iMac target, Mutter's expected successful shape is:
  - `eDP-1` TILE: same group id, grid `2x1`, loc `0,0`, size `2560x2880`,
  - `DP-1` TILE: same group id, grid `2x1`, loc `1,0`, size `2560x2880`,
  - both outputs expose a real `2560x2880` CRTC mode with compatible refresh/mode flags,
  - one `MetaMonitorTiled`,
  - one logical monitor mode `5120x2880`,
  - two CRTC assignments: left tile at `0,0`, right tile at `2560,0`.
- Mutter parses and preserves the TILE `flags` field, and exposes it over DisplayConfig, but the core grouping gate found here does not require a specific `flags` value. The geometry/group-id/mode checks are the stronger requirements.
- `meta-kms-connector.c` treats tile-info changes as `META_KMS_RESOURCE_CHANGE_FULL`, so a changed TILE property should trigger a full resource/monitor rebuild if Mutter observes the change.
- `meta-output.c::meta_output_matches()` includes connector name, vendor, product, and serial when matching outputs across updates. For a stable tiled monitor update, the origin and main outputs must remain matchable by those identity fields.

RE-10 Linux implication:

- If Mutter sees the two outputs as independent monitors, first suspect TILE validation failure: missing peer, wrong group id, wrong grid, duplicate location, wrong tile size, or one tile missing the `2560x2880` mode.
- If Mutter sees one 5K logical monitor but the image is stretched or half missing, first suspect the later assignment/programming layer: Mutter should be asking for per-tile CRTCs at `0,0` and `2560,0`, so compare Mutter's requested CRTC layouts against amdgpu/DC's final stream/plane/viewport programming.
- A kernel-side log should print the exact TILE blob string that userspace reads, not only internal connector fields.
- A kernel-side log should also prove whether a full hotplug/resource update is emitted after TILE mutation, because Mutter only rebuilds from the new tile info after it notices a full resource change.

RE-10 update from `kernel_obs_9`:

- The latest run satisfies Mutter's tiled-monitor construction contract at the DRM object level:
  - `eDP-1` exposes TILE `1:1:2:1:0:0:2560:2880`, EDID identity `APP/iMac/84CC4178B8A27`, and preferred mode `2560x2880`.
  - `DP-1` exposes TILE `1:1:2:1:1:0:2560:2880`, the same EDID identity, and preferred mode `2560x2880`.
  - The two connectors are connected and assigned to separate active CRTCs.
- A previous DisplayConfig capture (`C:\proj\reverse\WT6A_INF\mutter_display_config.txt`) proves Mutter can expose this shape as one monitor with current/preferred logical mode `5120x2880`.
- Separate `gnome-shell` framebuffers of `2560x2880` per CRTC are not by themselves evidence that Mutter failed to group the monitor. Mutter normally creates one native stage view / onscreen framebuffer per active CRTC, even for one tiled `MetaMonitor`.
- The remaining GNOME-side thing to verify in a fresh capture is Mutter's internal CRTC/view layout, not merely framebuffer size. For a correct 2x1 tile, Mutter should assign the left view at logical/physical x `0` and the right view at the tile offset (`2560` in physical layout, or `2560 / scale` in logical layout).
- In `kernel_obs_9`, the final GNOME atomic commit keeps both tile streams alive, but each DC pipe shows viewport/source `+0+0`. Earlier fbcon/plymouth split scanout used one shared `5120x2880` framebuffer and therefore showed the right tile with viewport/source `+2560+0`. For GNOME, per-CRTC `+0+0` can be normal if Mutter rendered the correct right-half stage view into DP-1's own `2560x2880` buffer. The next thing to verify is therefore Mutter's final view/layout assignment and rendered per-view content, plus the secondary link-training/HPD-low failure path, not initial TILE parsing.

`RE-11`: Linux DRM/amdgpu handoff timing.

- Goal: map the Mutter requirements from `RE-10` onto what our kernel exposes before GNOME's first monitor rebuild and first atomic commit.
- Main source tree: `C:\proj\reverse\WT6A_INF\linux-imac-5k`.
- Primary files to inspect:
  - `drivers/gpu/drm/amd/display/amdgpu_dm/amdgpu_dm.c`
  - `drivers/gpu/drm/amd/display/amdgpu_dm/amdgpu_dm.h`
  - `drivers/gpu/drm/amd/display/amdgpu_dm/amdgpu_dm_quirks.c`
  - Linux DRM core tile helpers around `drm_connector_set_tile_property()`, `drm_mode_get_tile_group()`, and connector hotplug/property update paths.
- Specific questions:
  - are TILE fields installed before userspace sees the connector,
  - does userspace receive a hotplug/config change after TILE mutation,
  - does the first GNOME commit contain both tile streams,
  - if a commit drops one tile stream, should the iMac quirk reject/defer it after the proven two-tile state is armed.

RE-11 kernel logging update:

- `amdgpu_dm.c` now logs the exact userspace-facing DRM TILE blob for the iMac connectors as `IMAC5K: ... mutter_tile ... blob="group:single:num_h:num_v:loc_h:loc_v:tile_w:tile_h"`.
- The same log prints `mutter_shape_ok`, tile group id, tile group topology bytes, grid, location, and tile size. This directly matches Mutter's eight-integer parse contract.
- `amdgpu_dm_connector_get_modes()` now logs `IMAC5K: ... mutter_modes ...` plus each probed/current mode for the iMac pair, so a GNOME failure can be checked for a missing or mismatched per-tile `2560x2880` mode.
- `amdgpu_dm_atomic_commit_tail()` now logs `IMAC5K: ... mutter_atomic_* ...` on relevant modeset/resource commits. These lines show which connectors Mutter enabled, each CRTC mode, each primary/overlay plane's framebuffer size, source rectangle, and CRTC rectangle.
- The relevant DRM hotplug/resource event paths now log `IMAC5K: ... emitting DRM hotplug/resource event ...`, so we can tell whether Mutter was nudged to rescan after connector/TILE changes.
- The iMac false-disconnect gate now logs `IMAC5K: ... false_disconnect_gate ... reason=...` and requires a detected disconnect, a still-live real secondary sink, Windows-equivalent DDC3/UNIPHY_D/`0x3113` route identity, and prior AUX evidence from source-DPCD, `0x111`, or `0x4F1`.
- The iMac TILE rewrite now forces the userspace-facing `single_monitor` flag to true for both tiles and logs if it had to override an inherited value.
- TILE publication paths now log explicit skip/failure reasons when the primary tiled head or DRM tile group is not available.
- Atomic validation failure now emits `IMAC5K: atomic_check_fail ...` plus the same connector/CRTC/plane state dump used for Mutter handoff analysis.
- The old Stage 2 naming was changed from "plain-boot synthesis" to "plain-boot real-route probe" so it cannot be confused with the retired fake/synthetic connector idea.
- The intended diagnosis split is:
  - bad or missing `mutter_tile` / `mutter_modes` means GNOME was not given the tiled-monitor contract it requires,
  - good `mutter_tile` / `mutter_modes` but bad `mutter_atomic_plane` or later DC viewport logs means GNOME or amdgpu/DC is mapping the two tile scanouts incorrectly after grouping.

`RE-12`: Windows grouped-view finalization.

Status: complete enough for the Stage 1 "keep 5K through GNOME" kernel work.

Goal:

- Identify the last Windows point that guarantees both tile streams belong to one grouped view before viewport/resource finalization.
- Clarify whether Windows finalizes one fake 5120-wide stream or two real 2560-wide tile streams.
- Map the result to Linux atomic behavior: what must be present in the commit before amdgpu/DC should let userspace take over.

Main conclusion:

- Windows enters the grouped commit path only after it already has exactly two active display objects with real stream pointers.
- The iMac grouped path uses `CreateOrGetDisplaySetViewCommitContext(..., a4 = 0)`, which selects `ConstructMultiDisplayViewCommitContext`, not the clone/fake-sink constructor.
- `ConstructMultiDisplayViewCommitContext` allocates one context entry per physical tile and creates one real stream plus one real plane state for each tile.
- `ValidateActiveDisplaySetViaViewCommitContext` applies per-display selected mode width/height and timing to each tile stream before final validation.
- `ValidateMultiDisplayViewContextAddStreamsPlanesFinalize` is the final grouped transaction point: it loops all real tile stream/plane entries, adds each stream to the same DC validation context, attaches each plane, preserves/clones split-pipe links, and only then calls `DcValidateFinalize_RebuildViewportsAndResources`.
- `DcValidateFinalize_RebuildViewportsAndResources` rebuilds resources/viewports after both streams and both planes have been accepted into the shared validation context.
- The observed path does not expose a hidden single 5120-wide DC stream. The logical 5K monitor is a display-manager/grouping abstraction above two real per-tile streams.

Important Windows functions:

- `0x141A27A20` / `TryMarkCompatibleActiveDisplaySetForSpecialCommit`: requires exactly two active display objects with stream pointers before entering the special/grouped validation path.
- `0x141A27FD0` / `ValidateActiveDisplaySetViaViewCommitContext`: collects live active display indices, constructs the non-clone view commit context, writes per-display source size/timing, then invokes the final grouped validation slot.
- `0x141A75A70` / `CreateOrGetDisplaySetViewCommitContext`: `a4 != 0` selects clone/fake-sink construction; `a4 == 0` selects `ConstructMultiDisplayViewCommitContext`. The grouped iMac path reaches the `a4 == 0` branch.
- `0x141ADB980` / `ConstructMultiDisplayViewCommitContext`: installs the multi-display view context vtables and allocates one stream/plane pair per active tile.
- `0x141ADB740` / `SetViewContextDisplaySourceSize`: writes the selected per-display size into stream and plane fields. For the iMac path this is per-tile `2560x2880`, not one logical `5120x2880` stream.
- `0x141ADAC70` / `ApplyPathModeTimingToViewContextStream`: applies the selected timing into the matching tile stream.
- `0x141ADA460` / `ValidateMultiDisplayViewContextAddStreamsPlanesFinalize`: final grouped transaction over all tile streams/planes.
- `0x141BADD00` / `DcValidateContext_AddStreamViaResourceCallback`: adds each stream into the shared DC validation context and calls the resource add-stream callback.
- `0x141BADE20` / `AttachPlaneStateAndCloneSplitPipeLinks`: attaches the matching plane state and preserves/clones split-pipe linkage for that stream.
- `0x141BAE330` / `DcValidateFinalize_RebuildViewportsAndResources`: final resource/viewport rebuild after all grouped stream/plane entries succeeded.
- `0x141BA8B20` / `DcBuildScalingParamsAndViewportForAllPipes`: iterates active pipe contexts and rebuilds viewport/scaling state.
- `0x141BA96A0` / `ComputeViewportForSplitPipe`: derives viewport placement from the split-pipe graph count/index rather than from a simple hardcoded `source_x = 2560` literal in the visible finalization path.
- `0x141BA9CC0` / `GetSplitPipeCountAndIndex`: walks linked pipe contexts to determine split count and pipe index.
- `0x141A75760` / `ValidateDisplaysShareTiledGroupMetadata`: requires every active tile endpoint to have matching tile group metadata and valid grid occupancy before the grouped path is considered valid.

Tile metadata gate:

- `ValidateDisplaysShareTiledGroupMetadata` rejects the one-display case.
- It requires each active endpoint to have nonzero tile metadata.
- It requires the active endpoints to share the same DisplayID-derived tile group token.
- It checks that the tile grid product matches the active display count.
- It computes occupied tile slots from the endpoint tile position fields and verifies that the occupied slot count equals the grid product.
- This still supports the current model that Windows expects real per-endpoint tile metadata rather than inventing a missing secondary tile inside finalization.

Linux implication:

- The Windows-equivalent Linux handoff should not let the first GNOME/userspace modeset silently collapse the armed two-tile state into one active tile.
- If the iMac quirk has proven the two-tile state is armed, the first destructive one-tile atomic commit is suspicious. A useful kernel behavior is to log it clearly and either reject/defer it or force the grouped two-connector/two-CRTC commit shape, depending on what the next runtime trace proves.
- The desired Linux steady state for GNOME is therefore two real DRM connectors/streams with correct `TILE` grouping that Mutter can merge into one logical `5120x2880` monitor, not one synthetic/fake 5K connector.
- If `mutter_tile` and `mutter_modes` logs look correct but GNOME still displays a shifted/stretched half, the next suspect is the stream/plane/viewport mapping during grouped atomic commit, not EDID parsing.

`RE-13`: Secondary EDID/DisplayID recovery.

Status: complete enough for Stage 1.

Goal:

- Decide whether Windows reconstructs missing secondary tile metadata from the primary tile's DisplayID block.
- Separate Windows' real recovery behavior from a possible Linux-only compatibility workaround.
- Decide whether Linux should prioritize preserving the secondary route's own EDID/DisplayID parser output.

Main conclusion:

- Windows does not appear to reconstruct the secondary endpoint's 0x38-byte tile metadata block from the primary endpoint in the traced path.
- The metadata producer resolves the requested display id, calls that display object's own active DisplayID/EDID parser, and writes that endpoint's tile block only when the parser returns tiled topology successfully.
- If the parser cannot provide tiled topology, the caller clears the endpoint tile block instead of copying from a peer.
- Windows has extension-chain recovery inside the same display object's parser chain. That means "try the next EDID/DisplayID extension for this same endpoint," not "borrow the primary tile's topology for the secondary endpoint."
- The peer scan after successful metadata population only compares group tokens and elects one representative/primary tile by writing `endpoint_tile+0x34`. It does not copy the full metadata block.
- The grouped-view consumer expects nonzero matching group tokens and coordinate/grid fields to already be present on each active endpoint.

Important Windows functions:

- `0x141A206C0` / `PopulateEndpointTileMetadataFromDisplayIdTopology`: resolves the display object by display id and calls that display object's own DisplayID parser vfunc `+0x158` to fill a local topology record. On success it writes endpoint tile metadata; on failure it returns false.
- `0x141A0F2F0` / `UpdateEndpointModesAndTiledMetadataAfterPinAttach`: caller where `v43 = endpoint + 0x98`. On producer failure it zeroes `endpoint+0x98..0xCF`.
- `0x141A388A0` / `QueryNextEdidExtensionForTiledTopology`: fallback helper that calls the next parser slot at `parser+0x50`, still within the same display object's EDID/DisplayID parser chain.
- `0x141AEF130` / `ParseDisplayId20TiledTopologyBlock40`: DisplayID 2.0 tiled topology parser for tag decimal `40` / hex `0x28`; falls back only to `QueryNextEdidExtensionForTiledTopology`.
- `0x141AFB700` / `ParseDisplayId13TiledTopologyBlock18`: DisplayID 1.x tiled topology parser for tag decimal `18` / hex `0x12`; falls back only to `QueryNextEdidExtensionForTiledTopology`.
- `0x141A75760` / `ValidateDisplaysShareTiledGroupMetadata`: consumer that requires matching per-endpoint tile group tokens and uses each endpoint's own coordinate/grid fields.

Linux implication:

- Best Stage 1 behavior is to snapshot/preserve the real secondary `DP-1` / `0x3113` EDID and TILE metadata while the route is still live, then keep using that metadata if a later false disconnect clears `local_sink`.
- Reconstructing the secondary DRM `TILE` property from the primary `eDP-1` DisplayID remains a reasonable Linux compatibility fallback only when the real secondary route was already proven live and AUX-backed.
- That fallback should not be described as "Windows does this"; the Windows-shaped behavior is per-endpoint parser output plus extension-chain recovery.
- The `kernel_obs_8_gnome` logs support this direction: early boot had a real secondary `DP-1` sink with `sink_edid_len=256`, then later commits showed the route still identifiable but `local_sink=0`. That points to preserving/reusing the real secondary metadata captured earlier, not inventing metadata blindly.

`RE-14`: Plain-boot route construction.

- Goal: move from probe/OCLP preservation to Apple-free boot by making Linux discover or construct the real `0x3113` route.
- This is Stage 2 and should wait until Stage 1 gives GNOME one usable 5K monitor.
- Relevant Windows RE anchors:
  - `PopulateDisplayMapAuxObjectsForHotplugRange`
  - `CreateDisplayMapInterfaceForObjectIdMaybeSkipInternal`
  - object id `0x3113` as accepted class-3 display object
  - durable per-object DPCD/AUX transport at display-map entry `+0x18`

`RE-A`: Windows object-table/AUX construction mapped to Linux link creation.

- IDA checkpoint: `C:\proj\reverse\WT6A_INF\B416406\amdkmdag.sys.i64`.
- Goal: confirm whether Windows has a real, durable object-specific lower AUX route for the secondary tile and map the equivalent Linux construction path.
- `ConstructDpcdAuxInterfaceForObjectId` at `0x141551090` calls AdapterService vfunc `+0x170` at `0x1415511A1` to obtain the lower AUX handle for the exact display object id.
- AdapterService vfunc `+0x170` is `AdapterServiceGetAuxHandleForObjectId` at `0x141465AA0`.
- `AdapterServiceGetAuxHandleForObjectId` queries the object descriptor through AdapterService vfunc `+0x100` / `AdapterServiceQueryObjectDescriptor` at `0x1414663A0`.
- The descriptor fields used by the AUX path are:
  - descriptor `+0x1C`: AUX selector / DCE DDC A register, `0x4871` (`mmDC_GPIO_DDC3_A`) for secondary `0x3113`, `0x4875` (`mmDC_GPIO_DDC4_A`) for primary `0x3114`;
  - descriptor `+0x3C`: source index, converted to `source_mask = 1 << value`.
- `AdapterServiceGetAuxHandleForObjectId` passes selector and source mask to `AuxFactoryCreateHandleForSourceMask` at `0x14152AE80`.
- `AuxFactoryCreateHandleForSourceMask` constructs `ConstructDualPinAuxTransactionHandle` at `0x1415B0C20`.
- `ConstructDualPinAuxTransactionHandle` calls `ResolveSourceMaskToPinIndex` at `0x141529FB0`, then opens two concrete base-pin endpoints for the resolved pin:
  - endpoint type `3`: DDC DATA;
  - endpoint type `4`: DDC CLOCK.
- If either concrete endpoint is missing, the lower AUX handle invalidates itself. This means Windows requires a real lower DDC/AUX backend, not merely a logical display object.
- `ConstructAuxTransactionPathFactory` at `0x14152B8B0` builds the AUX factory and stores the selector/source-mask resolver at factory `+0xB0`.
- `CreateAuxPinMapFactoryByGeneration` at `0x1415AFB70` selects the generation-specific resolver. On generation `11`, it creates `ConstructDce11AuxPinMapFactory` at `0x1415FE160`.
- `Dce11ResolveAuxSelectorToPinIndex` at `0x141695990` has the literal mapping:
  - `0x4869 -> 0`,
  - `0x486D -> 1`,
  - `0x4871 -> 2`,
  - `0x4875 -> 3`,
  - `0x4879 -> 4`,
  - `0x487D -> 5`,
  - `0x4881 -> 6`,
  - `0x4899 -> 7`.
- Therefore the Windows secondary tile route is:
  - object id `0x3113`,
  - AUX selector / DDC A register `0x4871`,
  - AUX pin index `2`,
  - concrete DDC DATA + DDC CLOCK endpoints for that pin.
- The Windows primary tile route is:
  - object id `0x3114`,
  - AUX selector / DDC A register `0x4875`,
  - AUX pin index `3`.
- `InitializeAdapterServiceForConnector` at `0x141467800` creates the AUX transaction factory at AdapterService `+0x1A8`, unless base-pin property `6` bit `4` disables AUX factory creation. If this factory is absent, object-specific AUX handles such as the `0x3113` secondary route cannot be created.
- Linux counterpart:
  - `dc/core/dc.c::create_links()` asks the BIOS parser for connector count and connector ids, then calls `create_link()` for each object-table connector.
  - `dc/bios/bios_parser2.c::bios_parser_get_connectors_number()` counts display paths whose encoder object id is nonzero.
  - `dc/bios/bios_parser2.c::bios_parser_get_connector_id()` returns each path's `display_objid`.
  - `dc/link/link_factory.c::construct_phy()` stores that object id in `link->link_id`, asks `get_src_obj()` for the encoder, maps the encoder to transmitter, creates the DDC service, records `ddc_hw_inst`, HPD source, signal type, and link encoder.
  - `dc/link/protocols/link_ddc.c::ddc_service_construct()` calls `get_i2c_info()` for the connector object, maps the ATOM I2C record through the GPIO pin LUT, and creates `ddc_pin`.
  - `bios_parser2.c::bios_parser_get_i2c_info()` and `get_gpio_i2c_info()` are the Linux equivalent of descriptor-to-pin materialization for DDC/AUX routing.
- RE-A conclusion: Windows constructs the secondary tile as a real AMD object-table route with a real lower AUX/DDC backend. The closest Linux invariant to check is whether plain boot constructs and keeps a `dc_link` for `0x3113` with the Windows-equivalent DDC3 / UNIPHY_D / HPD route. If plain boot lacks that real route, TILE metadata alone cannot produce a working Apple-free 5K solution.
- Kernel implication:
  - Stage 1 should continue preserving only a proven real `0x3113` route with successful prior AUX evidence.
  - Stage 2 should instrument early link creation to print every object id, encoder object, transmitter, DDC line, HPD source, connector signal, and whether the resulting route matches `0x3113` / DDC3 / UNIPHY_D.
  - If `0x3113` is present but later dropped, fix the detect/preservation path.
  - If `0x3113` is absent from plain boot link creation, the cold-boot quirk must create or expose that real link from AMD/VBIOS ingredients before normal detection; it should not present a fake logical 5K monitor with no secondary AUX backend.

`RE-A-preserve`: Windows false-disconnect preservation gate.

- IDA checkpoint: `C:\proj\reverse\WT6A_INF\B416406\amdkmdag.sys.i64`.
- `DetectDisplayMainMaybeApple5k` calls `DetectDisplayAndApplyApple5kOverrides` to fill a 120-byte detection result, then normally calls `HandleDisplayDetectionResultAndLiveState` unless `result+0x73` takes the special side path.
- Relevant result fields:
  - `result+0x70`: changed/rescan flag,
  - `result+0x72`: detected connected flag,
  - `result+0x73`: special side-path flag that can bypass the normal live-state updater.
- `DetectDisplayAndApplyApple5kOverrides` contains the precise Apple/tiled false-disconnect branch at `0x1414B4106..0x1414B418A`.
- The branch requires:
  - descriptor/capability byte `+6` bit `3` set (`0x08`, tag family previously tracked as `0x3B`),
  - current display object vfunc `+0x258` reports connected,
  - new detection result `result+0x72` says disconnected,
  - current signal from vfunc `+0x288(-1)` is `11`, which `SignalTypeToString` maps to `DisplayPort`.
- If all conditions hold, Windows writes:
  - `result+0x72 = 1`,
  - `result+0x70 = 0`.
- `HandleDisplayDetectionResultAndLiveState` computes `changed = result.connected != current_connected`. Because the branch above restores `connected=1` and clears `changed/rescan`, the normal teardown path is avoided for the false disconnect.
- `ApplyDetectionResultAndUpdateLiveDisplayObject` is only called for changed, rescan, or embedded-panel cases. It calls `UpdateDisplaySinkConnectionAndLiveAuxState`.
- `UpdateDisplaySinkConnectionAndLiveAuxState` calls `UpdateDisplayObjectRecordLiveSinkObject`, which updates or clears the 104-byte display-object detection record's live object at `record+0x18`.
- Therefore Windows does not preserve the secondary tile by inventing a fake sink after disconnect. It prevents the false-disconnect result from reaching the live-object teardown path when the real DisplayPort object is still currently connected and tagged with the Apple/tiled `0x3B` capability family.
- Linux implication:
  - the current `false_disconnect_gate` should be interpreted as false-disconnect suppression for a proven real `0x3113` route,
  - it should require evidence that the route was live/AUX-writable and should not run for generic disconnected DP,
  - if possible, Linux should suppress/defer the teardown before clearing `dc_sink`/`local_sink`, not rebuild an emulated sink afterward.

`RE-B`: DPCD `0x111` and `0x205` clarification.

- IDA checkpoint: `C:\proj\reverse\WT6A_INF\B416406\amdkmdag.sys.i64`.
- `SetDpcdMstCtrlEnableBit` at `0x141B8E9C0` is literal and small: it reads DPCD address `0x111`, sets or clears bit `0` depending on the boolean argument, and writes DPCD `0x111` back through the same lower DPCD/AUX abstraction.
- The relevant call in `BringUpDpOrEdpWithLinkTraining` at `0x141B6CA70` happens before `ProgramDpPreTrainingAuxState` and before `PerformLinkTrainingWithRetries`.
- That call is gated by both:
  - `ClassifyDpLinkRateEncodingRange(link_settings) == 2`,
  - stream/link signal `32`.
- `ClassifyDpLinkRateEncodingRange` at `0x141B8C880` classifies link-settings field `+4` as:
  - `6..30` -> class `1`,
  - `1000..2000` -> class `2`,
  - otherwise class `0`.
- This keeps `0x111` as a plausible Windows-like missing secondary activation step for the iMac `0x3113` / signal-`32` route, but it is conditional. Linux should not set it globally; if we add it, gate it to the proven real secondary `0x3113` path and log old/new `0x111`.
- `ProgramDpPreTrainingAuxState` at `0x141B8E090` still follows with source-DPCD writes `0x300`, `0x303`, `0x310`, and optional `0x340`.
- `StreamEnableWrite4F1AndPollSinkStatus` at `0x1414E7810` confirms `0x205` is real but not unconditional. The stream-enable path writes DPCD `0x4F1 = 1` when descriptor byte `+6` has bit `2` or bit `3` (`0x3A` / `0x3B` capability family).
- The DPCD `0x205` poll is gated separately by descriptor byte `+6` bit `5`, mapped in earlier notes to tag `0x40`. When that bit is set, Windows polls DPCD `0x205` up to `50` times, sleeping `1 ms`, until the byte becomes `1`.
- Existing `APP/AE25` and `APP/AE26` static evidence explains tags `0x3A` / `0x3B`; it does not prove tag `0x40`. Therefore Linux should keep `0x205` as read-only diagnostic unless runtime evidence proves tag `0x40` for this exact boot state.
- RE-B conclusion: the next kernel-side DPCD experiment, if any, is a narrow `0x111` bit0 read/modify/write on the proven live `DP-1` / `0x3113` path before source-DPCD and training. Do not add an unconditional `0x205` poll.

`RE-CAP`: Capability/tag provenance.

- IDA checkpoint: `C:\proj\reverse\WT6A_INF\B416406\amdkmdag.sys.i64`.
- Goal: confirm whether descriptor byte `+6` bits `2/3/5` are real panel capability tags or just local magic bits in the DPCD writers.
- `ConstructVendorProductCapabilityTables` at `0x141ABCBD0` initializes the main vendor/product capability table through `InitCapabilityTableDescriptor`.
- `sub_141ABC720` at `0x141ABC720` initializes the main table from `off_144C4E080 -> 0x144C4F720`, with `361` records.
- Each main table record is four DWORDs:
  - EDID manufacturer id,
  - EDID product id,
  - capability-table source id,
  - value.
- `ApplyVendorProductCapabilityTableToPin` at `0x141A44600` extracts EDID manufacturer/product, supports exact manufacturer/product, manufacturer plus wildcard product, and full wildcard records, then:
  - maps record source id through `MapCapabilityTableIdToPinPropertyTag`,
  - inserts the resulting tag/value into the pin property list,
  - calls `SetPinDescriptorCapabilityBitById`.
- `MapCapabilityTableIdToPinPropertyTag` at `0x141A41A90` maps:
  - source id `0x38` -> pin property tag `0x3A`,
  - source id `0x39` -> pin property tag `0x3B`,
  - source id `0x3E` -> pin property tag `0x40`.
- `SetPinDescriptorCapabilityBitById` at `0x141A40B50` maps:
  - tag `0x3A` -> descriptor byte `+6`, bit `2`,
  - tag `0x3B` -> descriptor byte `+6`, bit `3`,
  - tag `0x40` -> descriptor byte `+6`, bit `5`.
- Raw table confirmation from `amdkmdag.sys`:
  - table record `149` at `0x144C50070`: manufacturer `0x1006` (`APP` in EDID byte order), product `0xAE25`, source id `0x38`, value `0`;
  - table record `150` at `0x144C50080`: manufacturer `0x1006`, product `0xAE26`, source id `0x39`, value `0`.
- The Apple odd/even pattern is broader than just this iMac:
  - `AE01/AE02`,
  - `AE0D/AE0E`,
  - `AE19/AE1A`,
  - `AE25/AE26`,
  - with the odd entry mapping to source id `0x38` / tag `0x3A`, and the even paired entry mapping to source id `0x39` / tag `0x3B`.
- Additional Apple entries `AE31/AE32` and `AE35/AE36` use the same `0x38/0x39` split but value `1`, proving the value field is independent from tag presence.
- No `APP/AE25` or `APP/AE26` table record uses source id `0x3E` / tag `0x40`; the few source-id-`0x3E` records belong to other manufacturer/product pairs. Therefore the Windows `0x205` poll is real, but static Apple iMac evidence still does not authorize treating it as required.

Consumers of these tags:

- `ApplyPinCapabilityEdidFixups` at `0x141A456E0` dispatches tags `0x3A` and `0x3B` to `DoubleEdidPhysicalWidthForTiledCaps`, tying the same capability family to the tiled/half-panel EDID geometry fixup.
- `DetectDisplayMaybeAssert4F1ForCapableSink` at `0x1414B2AF0` checks descriptor byte `+6` bit `2` / tag `0x3A` before the detect/status `0x4F1 = 1` write.
- `WindowsDM_EnableSecondaryTileIfRequired` at `0x141A174B0` checks descriptor byte `+6` bit `2` or bit `3`, then requires the selected link/display type `128` before calling `SendSecondaryTileOpcode1265Payload1`.
- `StreamEnableWrite4F1AndPollSinkStatus` at `0x1414E7810` checks descriptor byte `+6` bit `2` or bit `3` before the stream-enable `0x4F1 = 1` write; it separately checks descriptor byte `+6` bit `5` / tag `0x40` before polling DPCD `0x205`.
- `DetectDisplayEarlyExitForApple5kCaps` at `0x1414B31D0` returns handled when descriptor byte `+6` bit `2` or bit `3` is present, so this capability family participates in detect preservation/short-circuiting.
- `DetectDisplayAndApplyApple5kOverrides` at `0x1414B3E80` uses descriptor byte `+6` bit `3` / tag `0x3B` for the stronger false-disconnect suppression branch.
- `ParseEdidAndPinCapsIntoDcSinkInfo` at `0x141EDFFD0` copies the value field of pin property tag `0x3B` into parsed sink info `+0xAC`. When the caller output is `dc_sink + 0x508`, this becomes `dc_sink + 0x5B4`.
- `BringUpDisplayBySignalType` at `0x141B6A720` uses `dc_sink +0x5B4` to gate the post-bring-up `0x4F1` helper, so that particular path depends on tag `0x3B` value, while the detect/status, WindowsDM, and stream-enable paths depend on descriptor bit presence.

RE-CAP conclusion:

- The `0x3A/0x3B` family is proven as real vendor/product-derived Apple panel capability, not a random local bit.
- `APP/AE25` maps to tag `0x3A`; `APP/AE26` maps to tag `0x3B`.
- Tag presence sets descriptor bits that authorize EDID fixups, detect preservation, and multiple `0x4F1` writers.
- The value field is separate from presence. For `APP/AE25` and `APP/AE26`, value is `0`, so the post-bring-up `dc_sink+0x5B4` path may not fire from these exact static records, even though descriptor-bit presence can still authorize the other `0x4F1` paths.
- Tag `0x40` is the `0x205` poll gate, but it is not proven for `APP/AE25` / `APP/AE26`; Linux should keep `0x205` read-only diagnostic for now.

`RE-LINK`: secondary HPD-low / cached-link-settings preservation.

- `kernel_obs_9` proves the high-level tiled shape is not the immediate failure: both `eDP-1` and `DP-1` are connected, tiled, assigned to active CRTCs, and accepted by Mutter's expected DRM TILE shape.
- The lower-level failure clue is later in the same run: during GNOME commits, `DP-1` remains the Windows-equivalent `0x3113` route and has a live preserved sink, but HPD is low, AUX readiness fails, source-DPCD is skipped as already programmed, and the `0x4F1` latch is deferred because `link_active=0` even though `link_state_valid=1` and the previous trained rate/lane state is present.
- `EnableValidatedStreamMaybeAssert4F1` checks `LinkServiceHasCachedLinkSettings` (`0x141443B50`, link-service `+0x8C != 0`) before calling the fresh link-enable/link-training vfunc. If cached link settings exist, Windows skips that vfunc and continues stream setup.
- `TryEnableLinkWithHbr2Fallback` (`0x1414E5CB0`) first compares the requested link-settings tuple against the cached tuple at link-service `+0x8C/+0x90`. An exact match returns success without retraining; a mismatch disables/reprograms and performs normal training/fallback.
- `sub_1414D1C00` writes the requested tuple into the same `+0x8C` cache before the full training call, so this field should be described as cached link settings rather than unconditionally as proven trained settings.
- The stronger static conclusion is not "Windows proves HPD-low is always safe." The stronger static conclusion is: Windows has a no-fresh-training path based on cached matching link settings, and that path is not gated by a fresh HPD-ready/link-active check visible in the stream-enable function.
- `StreamEnableWrite4F1AndPollSinkStatus` performs the stream-enable call first, then writes `0x4F1 = 1` when descriptor byte `+6` has bit `2` or `3`. It retries the AUX write once after `10 ms` on failure. It does not contain an explicit pre-write `HPD high` or `link active` check comparable to Linux's current `link_active` defer.
- The optional `0x205` poll remains gated by descriptor byte `+6` bit `5`.
- Kernel implication: for the proven secondary `0x3113` route, the current Linux condition that defers `0x4F1` solely because `link_active=0` is stricter than the Windows stream-enable path we traced. The next kernel change should not be a generic fake-HPD workaround; it should narrowly allow the real `0x3113` route to proceed with cached-link-state stream finalization / `0x4F1` attempt when the real sink and prior AUX evidence exist, while logging whether the write succeeds or fails.
- Kernel caution: this is not a generic fake-HPD workaround. Windows' visible fake-HPD branch in `HandleDisplayDetectionResultAndLiveState` is HDMI-specific; the Apple 5K DP path is false-disconnect suppression plus trained-link preservation, gated by the Apple/tiled capability bits and real route identity.

`RE-LINK-ASSR`: missed Windows stream/link setup around DPCD `0x10A`, `0x111`, and `0x316`.

- IDA checkpoint: `C:\proj\reverse\WT6A_INF\B416406\amdkmdag.sys.i64`.
- This pass was targeted at the `kernel_obs_9` failure: Linux had two live tile streams and the correct real `0x3113` route, but the secondary commit happened with HPD low, AUX-ready failure, and no fresh secondary stream finalization beyond source-DPCD/`0x4F1`.
- New IDA names saved:
  - `ProgramRequestedLinkSettingsAndCache` at `0x1414D1C00`,
  - `ClassifyDpPanelModeForAssrPolicy` at `0x1414E1460`,
  - `ProgramDpcd10A_AssrBitForPanelMode` at `0x1414E1370`,
  - `ProgramAssrThenTrainLinkSettings` at `0x1414E1570`,
  - `ProgramDpcd316_StreamSourceFlag` at `0x1414DD030`.
- `StreamEnableWrite4F1AndPollSinkStatus` has a fuller order than the earlier notes captured:
  - maybe clear DPCD `0x111` bit `0`,
  - maybe verify/retry link capabilities,
  - try link enable using cached settings or full training/fallback,
  - program DPCD `0x316`,
  - call the real stream-enable vfunc,
  - then perform the tag-gated `0x4F1 = 1` write and optional `0x205` poll.
- `MaybeClearDpcd111Bit0BeforeStreamEnable` clears DPCD `0x111` bit `0` for DPCD revision `1.2+` when it is set. This means the earlier Windows `0x111` set path is not a simple persistent "enable and leave on" rule; Windows can clear it again before stream enable.
- `ProgramAssrThenTrainLinkSettings` runs before the actual link-training sequence. Its order is:
  - `ProgramRequestedLinkSettingsAndCache`,
  - `ClassifyDpPanelModeForAssrPolicy`,
  - `ProgramDpcd10A_AssrBitForPanelMode`,
  - then the real link-training routine.
- `ProgramDpcd10A_AssrBitForPanelMode` RMWs DPCD `0x10A`, which Linux names `DP_EDP_CONFIGURATION_SET`. Bit `0` is `DP_ALTERNATE_SCRAMBLER_RESET_ENABLE`. Policy values `1` or `2` set bit `0`; other nonzero policy values clear it.
- `ClassifyDpPanelModeForAssrPolicy` has a special object-id branch: if the class-3 display object low byte is `0x13` (the secondary `0x3113` route), it calls display-object vfunc `+0x1F0`; if that returns true, the policy is `1`, causing the `0x10A` ASSR bit to be set.
- `SetDpPhyPatternWithAssrPolicy` at `0x1414D8EE0` uses the same `ClassifyDpPanelModeForAssrPolicy` result in the DP training-pattern request block passed to the lower training backend. So the `0x3113` policy is not just a one-off `0x10A` write; it can also affect how the training-pattern backend is instructed.
- This maps suspiciously well to Linux. The generic AMD Linux code only treats `DISPLAY_PORT` as eDP panel mode when `link->is_internal_display` is true:
  - `link_edp_panel_control.c::dp_get_panel_mode()` returns `DP_PANEL_MODE_EDP` for `EDP`, or for `DISPLAY_PORT && is_internal_display`.
  - `kernel_obs_9` logs the secondary as `signal=32` / DisplayPort but `internal=0`.
  - Therefore Linux may never set `DP_EDP_CONFIGURATION_SET` / `0x10A` bit `0` for the secondary `0x3113` tile, while Windows has a direct `0x3113`-low-byte policy path that can set it.
- `ProgramDpcd316_StreamSourceFlag` also runs inside the Windows stream-enable path. It RMWs DPCD `0x316` in the source-specific `0x300..0x3ff` range and sets/clears bit `0` based on stream mode values `10`, `11`, or `12`. This is not the private `0x4F1` latch, but it is part of the same Windows stream-enable sequence and Linux currently does not log it in the iMac path.
- RE-LINK-ASSR conclusion: before another broad kernel guess, the next Linux-side change should at minimum log DPCD `0x10A`, `0x111`, `0x316`, `0x4F1`, and `0x205` on the real secondary `0x3113` route around detect, pretrain, and commit finalization. A likely targeted experiment is to set `DP_EDP_CONFIGURATION_SET` bit `0` for the proven secondary `0x3113` route before link training/source-DPCD, mirroring Windows' `0x3113` ASSR policy rather than relying on Linux's `is_internal_display` classification.

## Next Recommended Work

Near-term kernel patch:

- `amdgpu_dm_imac5k_should_suppress_secondary_false_disconnect()` now requires proof that the real secondary route was live/AUX-writable and only fires when the new detect result is disconnected.
- It uses per-connector flags for `0x10A`, source-DPCD, and `0x4F1` success.
- It keeps the route check against DDC3 / UNIPHY_D / `0x3113`.
- The RE-A route logging now reconstructs the raw AMD object id from `dc_link->link_id` and prints whether each link matches the Windows primary `0x3114/DDC4/UNIPHY_C` or secondary `0x3113/DDC3/UNIPHY_D` route.
- The old RE-B `0x111` set experiment has been removed from the active bring-up path. Newer Windows RE shows `StreamEnableWrite4F1AndPollSinkStatus` can clear `0x111` bit `0` before stream enable, so Linux now treats `0x111` as observable state and clears it if present on the proven secondary route, rather than using it as activation proof.
- The RE-A-preserve pass confirms Windows suppresses a false disconnect before the live object is cleared: Apple/tiled byte `+6` bit `3`, current object still connected, new result disconnected, and signal `11`/DisplayPort cause `result.connected` to be restored and `result.changed` to be cleared.
- The RE-CAP pass confirms `APP/AE25 -> 0x3A` and `APP/AE26 -> 0x3B` are vendor/product-derived pin capabilities, and confirms no static `APP/AE25`/`APP/AE26` evidence for tag `0x40`.
- The iMac TILE path now forces `tile_is_single_monitor = true` and correct `2x1` TILE fields for the real pair.
- Current logging now shows TILE fields, tile group id, stream count, tile stream count, mode lists, framebuffer size, and atomic plane source/CRTC rectangles.
- Do not reintroduce an emulated sink fallback.
- Do not add an unconditional `0x205` poll; keep `0x205` as read-only diagnostic unless runtime evidence proves the Windows tag-`0x40` gate on this panel.
- Add or adjust the lower-level secondary preservation logic so later userspace commits can use the cached `0x3113` rate/lane state when HPD is low, matching the Windows cached-link-settings skip path. Log any decision as cached-link-preserve versus fresh-retrain so the next iMac run tells us which path executed.
- Implemented kernel-side RE-LINK change: commit-time secondary `0x4F1` finalization now requires the real live secondary sink and Windows-equivalent route, but can proceed through a `Windows-like cached-link path` when `link_active=0` if `link_state_valid`, cached rate/lane state, and prior AUX evidence exist. This path force-reasserts `0x4F1` instead of returning early on a previous detect-time assertion, so the next run should show whether the stream-enable-time AUX write succeeds or fails.
- New dmesg strings to look for: `secondary 0x4F1 latch using Windows-like cached-link path`, `secondary 0x4F1 latch reasserting`, and final `force_write=1 cached_link_state=... windows_cached_path=... status=...`.
- Implemented kernel-side RE-LINK-ASSR change, but `obs_10` proves it is still too conservative: Linux recognizes the Windows-equivalent secondary route (`0x3113`, DDC hw instance `2`, UNIPHY_D) in `link_edp_panel_control.c::dp_get_panel_mode()`, but only returns `DP_PANEL_MODE_EDP` when the sink advertises `panel_mode_edp`. Runtime log `obs_10` shows the real secondary `0x3113` route has `panel_mode_edp=0`, so Linux still skips DPCD `0x10A` bit `0`.
- `amdgpu_dm.c` now also programs/logs DPCD `0x10A` bit `0` on the proven secondary route before source-DPCD publication, and records `dpcd10a_attempted` / `dpcd10a_asserted` per connector.
- New DPCD snapshots log `0x10A`, `0x111`, `0x316`, `0x4F1`, and `0x205` around source-DPCD and stream finalization. `0x316` remains read-only diagnostic because Windows' mode values `10/11/12` have not yet been mapped to a Linux stream-mode value.
- Route-preservation auxiliary evidence now uses `0x10A`, source-DPCD, or `0x4F1`; `0x111` no longer counts as durable activation evidence.
- `obs_10` confirms the latest kernel ran (`7.0.1-1-imac-5k-g21938f46a96a`) and the new logs executed. It preserves the real `DP-1` / `0x3113` secondary route, has valid early AUX, successfully publishes source-DPCD, reads `0x316 = 0`, and sees `0x4F1 = 1` early. It does not assert `0x10A` because `panel_mode_edp=0`; later GNOME commits occur with HPD/AUX low, so late `0x4F1` reassertion fails despite cached-link-state evidence.
- `obs_11` confirms the exact-route `0x10A` change worked: the secondary `DP-1` / `0x3113` route is live early, `0x10A` is changed from `0x00` to `0x01`, source-DPCD reads back as `0x310=04 1d 03`, and the early snapshot sees `0x4F1=0x01`.
- `obs_11` also shows the new failure point more precisely: the first userspace KMS takeover from `plymouthd` commits only the primary `eDP-1` tile (`stream_count=1`, `tile_streams=1`) while the secondary has been detected but has not yet become a DC stream. After that primary-only commit, `DP-1` false-disconnect suppression keeps the logical object and TILE metadata, but the lower link is already weak/dead (`hpd=0/0`, `link_active=0`, `link_state_valid=0`, rate/lanes zero). The later two-tile commit has correct TILE metadata and stream shape, but secondary AUX is no longer ready and link training/source-DPCD/late `0x4F1` cannot complete.
- `RE-LINK-ASSR-2` confirms the Windows route is stronger than the first Linux implementation. `ProgramAssrThenTrainLinkSettings` at `0x1414E1570` computes `ClassifyDpPanelModeForAssrPolicy`, writes DPCD `0x10A` through `ProgramDpcd10A_AssrBitForPanelMode`, and only then calls the actual training helper. Therefore `0x10A` is a pre-training setup step, not a late stream-enable cleanup.
- `SetDpPhyPatternWithAssrPolicy` at `0x1414D8EE0` passes the same `ClassifyDpPanelModeForAssrPolicy` result into the lower training-pattern request block. Therefore the Windows `0x3113` ASSR policy affects both DPCD `0x10A` and training-pattern programming.
- `TryEnableLinkWithHbr2Fallback` at `0x1414E5CB0` has a cached-link fast path: if requested link settings match cached settings at link-service `+0x8C/+0x90`, Windows treats link enable as successful without fresh training. If settings differ, the fresh path calls `ProgramAssrThenTrainLinkSettings`, so ASSR is applied before training.
- The lower AUX/DPCD write path does not currently look like a hidden firmware wakeup path. `DpcdAuxWriteUpTo16Bytes` at `0x14154F020` builds and submits a normal AUX write; for special OUI `0x90CC24`, `DpcdAuxWriteRetryNakSpecialOui` at `0x14154EEB0` retries NAK/defer up to seven times and then maps status. This supports fixing Linux earlier, while AUX is alive, rather than relying on late `0x4F1` reassertion after HPD/AUX have dropped.
- `RE-LINK-CLASSIFY` separates the Windows steps by portability:
  - DP/eDP standard or normal AMD DC equivalents: DPCD `0x10A` / `DP_EDP_CONFIGURATION_SET` bit `0` for ASSR, DPCD `0x111` / `DP_MSTM_CTRL` bit `0`, DPCD `0x205` / `DP_SINK_STATUS`, and ordinary link-training / training-pattern programming.
  - AMD/source-specific DP setup: source-DPCD publication at `0x300`, `0x303`, `0x310`, optional `0x340`, plus stream/source flags such as `0x316` and post-training `0x32F`.
  - Apple/panel-private setup: sink-private DPCD `0x4F1 = 1`, authorized by Apple/tiled capability bits `0x3A` / `0x3B` and meaningful only on the real secondary `0x3113` AUX route.
  - Windows lifecycle behavior to emulate conceptually, not literally: false-disconnect suppression and cached-link-state preservation.
- `BringUpDpOrEdpWithLinkTraining` at `0x141B6CA70` is the lower normal-DP bring-up order: optional `0x111` for signal `32` / link-rate class `2`, optional signal-`128` HPD/AUX callbacks, source-DPCD publication, then `PerformLinkTrainingWithRetries`.
- `ClassifyDpLinkRateEncodingRange` at `0x141B8C880` classifies link-setting field `+4`: `6..30` is class `1`, `1000..2000` is class `2`. `obs_10` secondary rate `20` is class `1`, so the Windows `0x111` class-`2` pre-enable path is not the strongest next Linux target for this exact run.
- `PrepareLinkTrainingStateAndArmEdp128` at `0x141BB7090` repeats the HPD/AUX arming callbacks only for signal/type `128`. The observed secondary `0x3113` route is signal `32`, so do not copy the type-`128` callback behavior blindly into Linux for `DP-1`.
- `WriteDpcd32FForDpEdpSignal` at `0x141B8C700` writes DPCD `0x32F` bit `0` after training only under extra link flags and only for signal `32` or `128`. Treat this as source/backlight/source-specific diagnostic for now, not as the core Apple 5K latch.

`RE-SEQUENCE`: Windows event order around detect, training, stream-enable, and `0x4F1`.

- IDA checkpoint: `C:\proj\reverse\WT6A_INF\B416406\amdkmdag.sys.i64`.
- This pass was done before another kernel compile because `obs_10` suggests a sequence problem: early secondary AUX/source-DPCD/`0x4F1` succeeds while the later GNOME stream finalization sees HPD/AUX low and late `0x4F1` reassertion fails.
- Windows does not have one single linear "5K opcode" path. It has at least three relevant phases: detect/status, initial normal-DP bring-up, and DisplayPortLinkService stream-enable.
- Detect/status phase:
  - `DetectDisplayAndApplyApple5kOverrides` calls `ProbeDisplayStatusMaybeAssert4F1` on the normal detect path.
  - `ProbeDisplayStatusMaybeAssert4F1` first resolves the display object id and validates the durable per-object transport from the 64-byte display-map entry at `+0x18/+0x20`.
  - `DetectDisplayMaybeAssert4F1ForCapableSink` checks Apple/tiled descriptor byte `+6` bit `2` / tag `0x3A`, writes DPCD `0x4F1 = 1` through the durable per-object AUX route, waits `100 ms`, then re-reads status.
  - Therefore the detect/status `0x4F1` write can happen before later bring-up and stream-enable. It is not evidence of a firmware mailbox, and it is not visibly gated by a later `link_active` state.
- Initial normal-DP bring-up phase:
  - `BringUpDisplayBySignalType` dispatches signal `32` to `BringUpDpOrEdpWithLinkTraining`.
  - `BringUpDpOrEdpWithLinkTraining` order is: optional `0x111` only for signal `32` / link-rate class `2`; signal-`128` HPD/AUX callbacks only for type `128`; `ProgramDpPreTrainingAuxState`; then `PerformLinkTrainingWithRetries`.
  - `ProgramDpPreTrainingAuxState` publishes AMD/source DPCD state before training: `0x300`, `0x303`, `0x310`, and optionally `0x340`.
  - Only after successful lower bring-up does `BringUpDisplayBySignalType` set the live flag at display object `+888` and optionally call the post-bring-up `0x4F1` helper if the lower object at `+40` has flag/value `+1460`.
  - This post-bring-up `0x4F1` helper is downstream of successful training; it is not the first opportunity to latch the panel.
- DisplayPortLinkService stream-enable phase:
  - `EnableValidatedStreamMaybeAssert4F1` skips fresh link training when `LinkServiceHasCachedLinkSettings()` is true. When cached settings exist, it continues stream setup without an explicit fresh HPD-high/link-active gate visible in that function.
  - The non-cached stream-enable route calls `StreamEnableWrite4F1AndPollSinkStatus`.
  - `StreamEnableWrite4F1AndPollSinkStatus` order is: maybe clear DPCD `0x111` bit `0`; maybe verify/retry link caps; call `TryEnableLinkWithHbr2Fallback`; program DPCD `0x316`; call the real stream-enable vfunc; then write tag-gated DPCD `0x4F1 = 1`; finally poll `0x205` only when descriptor byte `+6` bit `5` / tag `0x40` is present.
  - The stream-enable `0x4F1` write is attempted after the stream-enable vfunc and retried once after `10 ms`. No explicit HPD-high/link-active check comparable to Linux's late defer is visible in this function.
- Fresh DisplayPortLinkService training phase:
  - `TryEnableLinkWithHbr2Fallback` first compares requested link settings against cached settings at link-service `+0x8C/+0x90`.
  - On a cache match it returns success without fresh training.
  - On a mismatch it disables/reprograms and calls `ProgramAssrThenTrainLinkSettings`.
  - `ProgramAssrThenTrainLinkSettings` order is: `ProgramRequestedLinkSettingsAndCache`, `ClassifyDpPanelModeForAssrPolicy`, `ProgramDpcd10A_AssrBitForPanelMode`, then `PerformLinkTrainingWithAssrPatternPolicy`.
  - `PerformLinkTrainingWithAssrPatternPolicy` calls `SetDpPhyPatternWithAssrPolicy`, which again feeds `ClassifyDpPanelModeForAssrPolicy` into the lower training-pattern backend.
- Sequence conclusion:
  - `0x10A` is a pre-training requirement in the Windows fresh-link path, not a late stream-enable cleanup. The Linux fix must make the exact secondary `0x3113` route look like the Windows ASSR policy route before normal AMD link training.
  - Early `0x4F1` success and late `0x4F1` failure are not contradictory. Windows has an early detect/status writer while AUX is still live, and a later stream-enable writer that assumes the stream/link lifecycle can continue from cached or freshly trained state.
  - The next compile should therefore test the already-targeted `0x3113` ASSR/panel-policy change first. Do not move to broad fake HPD, fake sinks, or signal-`128` callback copying based only on the late AUX-low symptom.
- Linux cross-check after this pass:
  - `link_dp_training.c::perform_link_training_with_retries()` gets `panel_mode = dp_get_panel_mode(link)`, calls `edp_set_panel_assr(...)`, then calls `dp_set_panel_mode(...)` before AUX-based DP link training.
  - `link_dp_training.c::dp_set_hw_test_pattern()` also feeds `dp_get_panel_mode(link)` into the training-pattern request block, matching the Windows `SetDpPhyPatternWithAssrPolicy` shape.
  - Therefore the targeted `link_edp_panel_control.c::dp_get_panel_mode()` route quirk is the right place to make normal AMD Linux training inherit the Windows `0x3113` ASSR policy, while the `amdgpu_dm.c` explicit `0x10A` helper remains useful logging/early-programming insurance.

`RE-STATE`: probable Windows state machine around the three phases.

- IDA checkpoint: `C:\proj\reverse\WT6A_INF\B416406\amdkmdag.sys.i64`.
- There are at least two coupled state machines:
  - the display object / detect-live state, including the live flag at display object `+888`, the false-disconnect override in `DetectDisplayAndApplyApple5kOverrides`, and the live-object teardown avoided by clearing the detection changed flag;
  - the DisplayPortLinkService stream/link lifecycle state, including cached link settings at link-service `+0x8C/+0x90` and a stream-enable state field seen as `a1 + 0x58` inside `StreamEnableWrite4F1AndPollSinkStatus`.
- Display object state:
  - Phase 1 can preserve the object as live by converting an Apple/tiled false-disconnect result back to connected and clearing the changed/rescan flag.
  - This explains why a later layer may still have a `0x3113` object after HPD instability: Windows prevented teardown of the real object rather than creating a fake sink.
- Link/cache state:
  - `LinkServiceHasCachedLinkSettings` is literal: link-service `+0x8C != 0`.
  - `ProgramRequestedLinkSettingsAndCache` writes the requested tuple into `+0x8C/+0x90`.
  - `TryEnableLinkWithHbr2Fallback` compares the requested tuple with that cache. If it matches, it returns success without fresh training; if not, it calls the full ASSR-before-training path.
- Stream-enable state field:
  - In `StreamEnableWrite4F1AndPollSinkStatus`, state values `2` and `3` return success immediately.
  - State value `1` takes a special already-link-configured path: cache requested settings, program DPCD `0x316`, call a stream-enable vfunc, set the state to `3`, and return.
  - The normal path performs optional `0x111` clear, cap verify, cached-or-fresh link enable, `0x316`, stream-enable vfunc, tag-gated `0x4F1`, optional `0x205`, then sets the state to `2`.
  - Adjacent helper `sub_1414E7760` treats state `0` or `5` as already safe/no-op for that helper, and another helper can promote state `0` to `2` when a stream record is active. Exact names are still inferred, but the behavior proves a lifecycle state, not a one-shot flag.
- Current mapping to Linux/OCLP observations:
  - Phase 1 is working under probe/OCLP boot: the real `0x3113` route exists, exposes real tile metadata, accepts source-DPCD, and accepts early `0x4F1`.
  - Phase 2 was incomplete in `obs_10`: Linux had the route but skipped the Windows-like pre-training `0x10A` ASSR policy because `panel_mode_edp=0`.
  - Phase 3 was reached logically by GNOME/KMS, but with the secondary lower link in a weak state: cached link fields existed, the real sink was preserved, but HPD/AUX were low and the late `0x4F1` write failed.
  - Therefore the next test is about whether the new exact-route ASSR policy lets Phase 2 produce a stronger trained/cached link state before Phase 3, not about repeating Phase 1 detection.

`RE-ONE-STREAM-HANDOFF`: Windows grouped path versus Linux first userspace takeover.

- IDA checkpoint: `C:\proj\reverse\WT6A_INF\B416406\amdkmdag.sys.i64`.
- This pass was done after `obs_11`, because runtime proved the secondary `0x3113` route is alive early but is HPD/AUX-dead by the time the later two-tile userspace commit arrives.
- `WindowsDM_SetPathMode_BuildStreamsAndCommit` at `0x141A22470` processes each requested path, builds or replaces that display object's DC stream pointer at display object `+0x38`, then calls `CollectActiveDcStreamsFromDisplayObjects` before `DcCommitStreams_BuildValidateAndCommit`.
- `CollectActiveDcStreamsFromDisplayObjects` at `0x141ED0DB0` iterates all display objects and adds every non-disabled display object with a non-null `display+0x38` stream pointer to the commit array. Therefore Windows' DC commit stream count is derived from live display-object stream pointers, not just from the transient OS path record currently being updated.
- `TryMarkCompatibleActiveDisplaySetForSpecialCommit` at `0x141A27A20` only enters the special grouped path when exactly two active display objects have stream pointers. It clears/sets per-display state byte `display_state+0xF2` and then calls `ValidateActiveDisplaySetViaViewCommitContext`.
- `ValidateActiveDisplaySetViaViewCommitContext` at `0x141A27FD0` builds an active-display-index array from display objects that already have streams, creates the normal multi-display commit context with final argument `0`, feeds each active display's selected mode width/height and timing into the context, and finalizes through the view-context vfunc. This remains a two-real-stream model, not a hidden single fake 5K sink.
- `WindowsDM_ResetPathMode_ClearListedStreamsAndCommitRemaining` at `0x141A24B20` explicitly destroys stream pointers only for displays listed in the reset request, then collects and commits the remaining active display streams. The stream clear helper is now named `DestroyAndClearDisplayDcStream`, and the final pointer clear helper is `ClearDisplayDcStreamPointerAndModeCache`.
- Xrefs to `DestroyAndClearDisplayDcStream` are SetPathMode stream replacement, ResetPathMode, StopDevice/device teardown, and Windows7/DalWDDM parallel versions of the same paths. This supports treating stream removal as an explicit mode/reset/device lifecycle action, not as the normal false-disconnect preservation path.
- `DcValidateWithContext_RebuildStreamsAndPlanes` at `0x141BAD500` diffs old/new stream sets; if a secondary stream is omitted from the new set, it is treated as stale and removed. This is the closest Windows-side equivalent to Linux letting a one-stream commit power down the secondary route.
- `MarkSeamlessBootStreamIfResourcesMatch` at `0x141BA6DF0` is not a generic secondary preservation hook. It only marks streams matching type/signal `128` boot resources (`stream+0x933`), so it should not be copied blindly for the secondary signal-`32` `0x3113` route.
- RE-ONE-STREAM-HANDOFF conclusion: Windows' successful 5K path needs both tile display objects to have real stream pointers before the grouped validation/final commit. Windows can preserve an already-live peer stream across a partial SetPathMode-style update because it collects all active display-object streams, but it does not appear to rescue a secondary route after a destructive one-stream DC commit has already removed or powered it down.
- Linux implication from `obs_11`: the next kernel target is not another late `0x4F1` retry. The next target is preventing the first destructive primary-only userspace transition after the secondary `0x3113` route has been armed. The guard should trip when the real secondary route has early AUX evidence (`0x10A`, source-DPCD, or `0x4F1`) but the commit would proceed with only one tile stream before any successful two-tile commit. The log should make this explicit, for example `IMAC5K: blocking primary-only takeover while secondary 0x3113 is armed`.

`RE-16`: Windows partial modeset protection.

- IDA checkpoint: `C:\proj\reverse\WT6A_INF\B416406\amdkmdag.sys.i64`.
- RE-16 target: clarify how Windows handles a SetPathMode/ResetPathMode request that names only one tile while the peer tile is already live.
- `CollectActiveDcStreamsFromDisplayObjects` at `0x141ED0DB0` is the exact final commit collector used by both SetPathMode and ResetPathMode. Its predicate is:
  - `display = GetDisplayObjectByIndex(display_table, index)`
  - skip if `DisplayObjectIsDisabledForActiveCommit(display)` is true; this helper at `0x141ED9760` returns byte `display+0x95256`.
  - append `display+0x38` to the commit stream array if that stream pointer is non-null.
- Therefore the peer-preservation condition is now concrete: the unmentioned tile survives a partial modeset only if its display object is not disabled and still owns a live stream pointer at `display+0x38`.
- `WindowsDM_SetPathMode_BuildStreamsAndCommit` at `0x141A22470` destroys/replaces the stream only for the currently requested display object. The replacement path calls `DestroyAndClearDisplayDcStream` at `0x141A22FC7` only for that requested object, then installs the new stream at `0x141A23045` by writing `display+0x38 = new_stream`.
- After all requested paths are processed, SetPathMode calls the global collector at `0x141A233B0`. This means a partial SetPathMode that updates only the primary still commits `new_primary_stream + existing_secondary_stream` if the secondary was already live and not disabled.
- `SetSelectedModeEnabledFlag` at `0x141A2301D` writes a selected-mode enabled byte from the incoming path flags, but this is not the commit collector predicate. Active-commit membership is controlled by the disabled byte and `display+0x38`.
- `WindowsDM_ResetPathMode_ClearListedStreamsAndCommitRemaining` at `0x141A24B20` explicitly destroys stream pointers only for display IDs listed in the reset request. It then calls the same global collector at `0x141A24F29` and commits the remaining active streams at `0x141A24F6A`.
- `TryMarkCompatibleActiveDisplaySetForSpecialCommit` at `0x141A27A20` counts live display stream pointers and requires exactly two. If a one-tile transition has already cleared the secondary `display+0x38`, the grouped/special path cannot recover it later.
- RE-16 conclusion: Windows does not solve partial modesets by letting a one-tile commit retire the peer and rebuilding it later. It keeps the peer in the commit array by preserving the peer display object's live stream pointer across partial SetPathMode/ResetPathMode operations.
- Linux implication: after Linux reaches the two-real-stream handoff state, the Windows-like behavior is to carry/preserve the peer stream through partial userspace updates. A hard reject/defer remains useful before handoff, but after both route roles are live the better model is "never let a partial userspace request silently clear the peer tile"; either carry the peer state forward or fail loudly with a log that says the current atomic path cannot preserve the peer.

`RE-15`: who creates the secondary live stream before first userspace handoff.

- IDA checkpoint: `C:\proj\reverse\WT6A_INF\B416406\amdkmdag.sys.i64`.
- RE-15 target: trace the durable stream pointer at display object `+0x38`, especially for the secondary `0x3113` tile, and separate real live stream creation from temporary validation stream allocation.
- Stream allocation and live-stream installation are separate steps. `AllocateDcStreamForDisplayObject` at `0x1419FB8C0` allocates a `dc_stream_state` from the display object's selected/special mode block and stores the owner display object at stream `+0x3E0`, but the stream is not live until a later store writes it to `display+0x38`.
- Important checked false lead: `UpdateEndpointModesAndTiledMetadataAfterPinAttach` calls `AllocateDcStreamForDisplayObject` at `0x141A10094` during detect/mode-list/tile metadata processing, but this is a temporary validation stream. The same function frees it at `0x141A114C5` and never writes it into `display+0x38`.
- The main live-stream birth point is `WindowsDM_SetPathMode_BuildStreamsAndCommit`:
  - `0x141A22D75`: allocate candidate stream for the current display object.
  - `0x141A22DCF`: `BuildDcStreamFromSelectedModeObject` fills timing/source fields from the selected path-mode object.
  - `0x141A23045`: stores the newly built stream into `display_object+0x38`.
  - `0x141A23074`: adds that stream to the pending commit context with `AddDcStreamToCommitContextArray`.
  - `0x141A233B0`: calls `CollectActiveDcStreamsFromDisplayObjects`, which collects every non-disabled display object that already has `display+0x38 != NULL`.
  - `0x141A237C7`: commits the collected stream array through `DcCommitStreams_BuildValidateAndCommit`.
- The parallel SetPathMode implementations have the same durable stream install pattern:
  - `DalWDDM_SetPathMode_BuildStreamsAndCommit` at `0x141AAD910` writes the live stream at `0x141AAE3DF`, then collects active display streams at `0x141AAE777`.
  - `Windows7DM_SetPathMode_BuildStreamsAndCommit` at `0x141F02B40` writes the live stream at `0x141F0348A`, then collects active display streams at `0x141F036CA`.
- `BuildDcStreamFromSelectedModeObject` is only called by those three SetPathMode-like functions. No detect/status path has been found that builds and installs a durable `display+0x38` stream for the secondary tile before SetPathMode.
- `ValidateAndCommitExistingStreamPropertyChange` at `0x141A87BF0` can clone, validate, and write back stream pointers at `0x141A87F6B` / `0x141A884C0`, but both branches first require an existing `display+0x38` stream. This is an existing-stream property-change/revalidation path, not initial secondary creation.
- `RebalancePeerStreamsAfterNewStreamBuild` at `0x141A0A670` and `RebalancePeerStreamsAfterStreamReset` at `0x141A0A380` synchronize related peer streams in a grouped live set. They can replace a peer's `display+0x38` with a cloned/rebalanced stream, but only when the peer stream was already present in the live display set.
- `sub_141BD9B30` constructs a related/fake/split stream from an existing resource entry and validation context, sets stream state such as `+0x970`, and feeds DC validation. It does not install a display object's durable live stream at `display+0x38`, so it should not be treated as the answer to "who makes the secondary tile live?"
- RE-15 conclusion: Windows' stable grouped path depends on both real tile display objects already having durable `display+0x38` streams before grouped validation. Windows then preserves and rebalances those live peer streams across partial updates; it does not appear to synthesize the missing secondary live stream during final grouped validation if the peer was absent.
- Linux implication: our state machine should treat `secondary-route-seen` and `secondary-aux-armed` as necessary but insufficient. The handoff-safe state is closer to `secondary-stream-installed`: both tile connectors must have real DC streams in the atomic state before the first Plymouth/GNOME commit is allowed to retire the boot/pre-GUI configuration. A preserved connector plus TILE metadata is not enough.

`RE-19`: boot/pre-existing-stream adoption path for signal-32 objects.

- IDA checkpoint: `C:\proj\reverse\WT6A_INF\B416406\amdkmdag.sys.i64`.
- RE-19 target: ask whether Windows has another boot/preboot or pre-existing-stream adoption path that applies to signal-32 DisplayPort objects such as secondary `0x3113`, separate from the already-known type/signal-128 seamless helper.
- `MarkSeamlessBootStreamIfResourcesMatch` at `0x141BA6DF0` remains the only marker that sets `stream+0x933`, the flag consumed by the resource mapper to attempt boot/preboot pipe-resource reuse.
- `IsType128StreamMatchingBootResources` at `0x141B7B830` has a hard first gate: `stream->signal == 128`. If that test fails, it returns false before timing/resource identity comparisons. Therefore the secondary `0x3113` tile, observed as signal `32`, cannot directly enter this seamless resource-reuse path.
- `IsDpOrEdpSignalTypeForBootResourceMatch` at `0x141B7B4E0` accepts `32`, `64`, and `128`, but this helper is called only after the outer type-128 stream gate has already passed. It should not be read as a standalone signal-32 adoption path.
- `ResourceMapPoolResourcesForStream` at `0x141BA6800` calls `MarkSeamlessBootStreamIfResourcesMatch`, then only calls `TryMapSeamlessBootPipeResources` if `stream+0x933` is set. Otherwise it falls back to `MapFirstFreePipeResourcesForStream`, then `RecyclePipeContextForSplitClone`.
- `TryMapSeamlessBootPipeResources` at `0x141BA6E80` is the exact boot/preboot pipe-slot reuse path. It is only reached through the `stream+0x933` flag; xrefs show no independent signal-32 caller.
- `MapFirstFreePipeResourcesForStream` at `0x141BA7B50` is generic first-free resource assignment, not boot-state adoption. `RecyclePipeContextForSplitClone` at `0x141BA8610` is split/clone resource recycling, not preboot adoption for a missing peer display object.
- `WindowsDM_RecommendFunctionalVidPn_CheckSeamlessBootMatch` at `0x141A04000` is another consumer of `IsType128StreamMatchingBootResources` from the `recommendFunctionalVidPn` path. It uses the same type-128 matcher, so it does not add a separate signal-32 route.
- RE-19 conclusion: no separate Windows boot/pre-existing-stream adoption path for signal-32 `0x3113` was found in this pass. Windows can reuse boot/preboot pipe resources for type-128 streams and can keep already-live `0x3113` display/link objects through the normal live-stream/display-set machinery, but the DP-side secondary tile does not appear to get seamless boot pipe adoption by itself.
- Linux implication: forcing generic seamless boot is unlikely to solve the secondary `0x3113` tile directly. It may help preserve the primary/internal boot stream, but Stage 1 still needs the explicit route/AUX readiness plus two-real-stream handoff barrier; Stage 2 still needs real `0x3113` route construction/preservation rather than relying on a hidden Windows-like signal-32 seamless adoption path.

`KERNEL-READINESS-SM`: better final-design patch after `obs_11`, RE-15, and RE-16.

- The kernel now has a lightweight iMac 5K readiness state machine in `amdgpu_dm`: `off -> primary-seen -> secondary-route-seen -> secondary-aux-armed -> pair-ready -> secondary-stream-installed -> first-two-stream-commit`, with explicit `one-tile-deferred` diagnostics for rejected pre-handoff single-tile takeovers.
- The pair-ready state is asserted only when all of these are true: primary TILE shape is valid, secondary TILE shape is valid, the real secondary `0x3113` route proof succeeds, and the secondary AUX path has all three early setup markers (`0x10A`, source-DPCD, and `0x4F1`). After RE-15 this is deliberately named/logged as route/AUX readiness, not full handoff readiness.
- The new `secondary-stream-installed` / `stream_handoff_ready` layer requires the proposed or committed DC state to contain one tile stream on the Windows primary route (`0x3114` / DDC4 / UNIPHY_C) and one tile stream on the Windows secondary route (`0x3113` / DDC3 / UNIPHY_D). Merely seeing two `2560x2880` timings is no longer enough to arm the two-stream guard.
- Pair readiness is checked during boot detection, connector detection, get-modes, secondary TILE application, and secondary DPCD programming. This is meant to make the real two-tile shape available before Plymouth or a display manager sees the device, not merely to block a bad userspace commit late.
- DRM hotplug/resource notifications now run through an iMac-aware wrapper. If both heads are known but the pair is not ready, the event is deferred and logged instead of telling userspace to rescan a half-built topology.
- `atomic_check` has a pre-handoff safety barrier: if a proposed full DC state contains exactly one iMac tile stream while the real secondary route is already armed, the commit is rejected with a log line `IMAC5K: readiness barrier rejected pre-handoff single-tile takeover ...`. If the proposal contains both real route roles, it is accepted and logged as a two-real-tile handoff candidate.
- RE-16 adds the post-handoff safety barrier: once the current DC state already has both real role streams, a proposed state that would drop either the primary `0x3114` role or the secondary `0x3113` role is rejected in `atomic_check` before `commit_streams`. The log prefix is `IMAC5K: RE16 peer-preservation rejected destructive partial commit ...` and includes proposed/current role counts plus `missing_primary` and `missing_secondary`.
- The old tail-only `commit_streams_begin` observer was removed. We no longer knowingly allow a destructive one-tile state to reach hardware just to log it later; this matches the RE-16 Windows model where the unmentioned peer stream is carried forward instead of silently cleared.
- Stream logging now prints `primary_tile_streams`, `secondary_tile_streams`, `unknown_tile_streams`, per-stream `role=primary|secondary|unknown-tile|non-tile`, and `stream_handoff_ready`. This should make the next failed run tell us whether userspace offered one tile, two tiles with the wrong route identity, or two correct physical tile streams that failed later.
- The next run should answer two very specific questions:
  - Does dmesg show `route/AUX pair ready before stream handoff` before the first `plymouthd`/GNOME atomic state?
  - Does it then show `secondary stream installed` / `two-stream guard armed` with `primary_tile_streams=1 secondary_tile_streams=1`, or does userspace keep proposing a one-tile/incomplete route-role state?

`OBS-12`: result of the RE-15/RE-16 kernel patch.

- Result directory: `C:\proj\reverse\WT6A_INF\obs_12`.
- Kernel was the expected patched build: `7.0.1-1-imac-5k-g554c1da24d19`; local `linux-imac-5k` HEAD also resolves to `554c1da24d19`.
- The latest logging is present, so the run includes the RE-15/RE-16 role-aware state machine and barrier work: `stream_handoff_ready`, role counts, and `readiness barrier rejected pre-handoff single-tile takeover`.
- Early route/AUX setup works:
  - `route/AUX pair ready before stream handoff` appears at `6.525s`.
  - Secondary `0x3113` has real route proof, TILE metadata, `0x10A=1`, source-DPCD programmed, and `0x4F1=1` while HPD is still `0/1`.
- Plymouth first makes 249 invalid one-tile proposals. The barrier rejects them as designed, alternating between primary-only and secondary-only proposals:
  - primary-only example: `primary_tile_streams=1 secondary_tile_streams=0 missing_secondary=1`.
  - secondary-only example: `primary_tile_streams=0 secondary_tile_streams=1 missing_primary=1`.
- At `10.918s`, Plymouth finally proposes the correct two-real-tile atomic state and the barrier accepts it:
  - `readiness barrier accepted two real tile streams`.
  - `stream_count=2 tile_streams=2 primary_tile_streams=1 secondary_tile_streams=1`.
  - Both connectors have valid TILE blobs: eDP-1 `loc=0,0`, DP-1 `loc=1,0`.
  - The first accepted userspace commit uses one shared `5120x2880` framebuffer. The primary plane scans `src=0,0 2560x2880`; the secondary plane scans `src=2560,0 2560x2880`. This is exactly the logical 5K split shape we wanted Linux userspace to reach.
- The two-stream guard arms at `13.841s`, and later KMS activity keeps both real streams. There is no `RE16 peer-preservation rejected destructive partial commit` in this capture, which means userspace did not try to drop one tile after handoff.
- The remaining failure is lower-level link/AUX, not DRM tile grouping:
  - During the first accepted two-stream commit, secondary pretrain still has HPD/AUX and can read/write DPCD. Snapshot after source-DPCD shows `0x10A=0x01`, `0x4F1=0x01`, and valid DPCD reads.
  - Then normal DP link enable/training fails on the secondary: repeated `dpcd_set_link_settings` failures for `DP_DOWNSPREAD_CTRL`, `DP_LANE_COUNT_SET`, and `DP_LINK_BW_SET`, ending with `enabling link 1 failed: 15`.
  - After that failure, secondary HPD/AUX are gone (`hpd=0/0`, DPCD reads return `status=-1`) even though Linux keeps the secondary software stream and `link_state_valid=1`.
  - Later `0x4F1` latch reassertion also fails because AUX is already dead.
- OBS-12 conclusion: the RE-15/RE-16 kernel patch successfully forces Linux/Plymouth into the Windows-like two-real-stream/two-plane topology. The next problem is the secondary `0x3113` link-enable/training sequence. Future work should not focus on Mutter/TILE grouping unless visual/user-space captures contradict this; it should focus on the Windows DP stream-enable/link-training state machine, especially ordering around source-DPCD, `0x10A`, `0x316`, link settings, and `0x4F1`.

`RE-20 PLAN`: secondary `0x3113` link-enable/training failure after `OBS-12`.

Goal:

- Explain why Linux reaches the correct two-real-tile userspace topology, then loses the secondary during normal DP link-setting writes.
- Convert the answer into one targeted kernel patch rather than another broad experiment.
- Keep this pass scoped to Stage 1/probe boot. Plain-boot `0x3113` construction remains Stage 2 and should not distract this pass.

Starting observation:

- Before the first accepted two-tile commit, Linux has the real secondary `0x3113` route, valid TILE metadata, `0x10A=1`, source-DPCD published, `0x4F1=1`, and working DPCD reads.
- During normal link enable/training, the secondary fails on ordinary DP link-setting writes: `DP_DOWNSPREAD_CTRL`, `DP_LANE_COUNT_SET`, and `DP_LINK_BW_SET`.
- After those failures, HPD/AUX are gone and late `0x4F1` reassertion cannot recover the tile.
- Therefore the next RE target is not Gnome, Mutter, KDE, or TILE grouping. It is the Windows lower DP bring-up path for the exact `0x3113` route.

Notes-sync status before doing IDA work:

- `RE-6`, `RE-B`, `RE-LINK-ASSR-2`, and `RE-SEQUENCE` already establish the broad Windows order: optional `0x111`, source-DPCD, ASSR `0x10A`, link training/cached-link decision, optional `0x316`, stream-enable, then tag-gated `0x4F1`.
- `RE-7` already establishes that Windows preserves the real secondary object through a false-disconnect path; it does not solve this by inventing a fake sink.
- `RE-19` already establishes that the found seamless/preboot resource adoption path is type-`128`, not the direct signal-`32` `0x3113` route.
- Therefore `RE-20` should not rediscover those facts. It should answer the narrower unresolved question exposed by `OBS-12`: when the two-tile grouped commit arrives, does Windows fresh-train the `0x3113` secondary or classify it as already link-configured and avoid the destructive link-setting writes that Linux currently attempts?

`RE-20A`: exact fresh-training order for signal-32 `0x3113`.

- IDA anchors: `BringUpDpOrEdpWithLinkTraining` at `0x141B6CA70`, `ProgramDpPreTrainingAuxState` at `0x141B8E090`, `PerformLinkTrainingWithRetries` at `0x141B8D860`, `ProgramAssrThenTrainLinkSettings` at `0x1414E1570`.
- Do not re-prove the broad order already captured in `RE-6` / `RE-SEQUENCE`; instead trace whether the later grouped commit for an already-route-ready signal-`32` / low-byte-`0x13` object takes the fresh-training branch at all.
- If it does fresh-train, record the exact link-setting write order relative to `0x10A`, source-DPCD `0x300/0x303/0x310`, and any stream/source flag.
- Record whether Windows uses the same order for initial detect bring-up and later SetPathMode/stream-enable bring-up.
- Kernel decision this should drive: whether Linux should move any existing iMac source-DPCD/ASSR setup earlier, or avoid redoing it during the accepted two-stream commit.

`RE-20B`: cached-link fast path versus fresh retraining.

- IDA anchors: `TryEnableLinkWithHbr2Fallback` at `0x1414E5CB0`, cached match at `0x1414E5D1C`, fresh path at `0x1414E5D63`, `EnableValidatedStreamMaybeAssert4F1`.
- Confirm exactly what fields populate and compare the cached link settings at link-service `+0x8C/+0x90`.
- Determine whether Windows would skip fresh training for the secondary when the requested rate/lane tuple matches the already-live preboot/probe-trained state.
- Determine whether stream-enable state `1` is the "already link-configured" path for this case.
- Kernel decision this should drive: whether Linux should preserve/use the already-known secondary `cur_link_settings` and bypass a destructive fresh link-setting write sequence after the pair is route/AUX ready.

`RE-20C`: map the stream-enable state machine.

- IDA anchors: `StreamEnableWrite4F1AndPollSinkStatus` at `0x1414E7810`, adjacent state helper around `sub_1414E7760`, state writes around values `0`, `1`, `2`, `3`, and `5`.
- Produce a state table: value, meaning, entry condition, writes performed, next state.
- Special focus: state `1`, which previous notes show caches requested settings, programs `0x316`, calls stream-enable, promotes to `3`, and returns without full normal training.
- Kernel decision this should drive: whether our Linux state machine needs an explicit "secondary already link-configured, do stream finalization only" phase instead of allowing the generic DP training path to hammer AUX.

`RE-20D`: decide whether `0x316` is active for this panel.

- IDA anchor: `ProgramDpcd316_StreamSourceFlag` at `0x1414DD030`.
- Map stream mode values `10`, `11`, and `12` back to the selected path-mode/display object fields.
- Determine whether the iMac tiled `0x3113`/`0x3114` path can hit one of those values.
- If yes, record exact ordering relative to cached/fresh link enable and stream-enable vfunc.
- Kernel decision this should drive: either keep `0x316` diagnostic-only, or add a narrow real-secondary write at the Windows-equivalent point.

`RE-20E`: verify `0x111` is not the OBS-12 failure.

- IDA anchors: `SetDpcdMstCtrlEnableBit`, signal-`32` / class-`2` gate at `0x141B6CB5F`, `ClassifyDpLinkRateEncodingRange` at `0x141B8C880`, clear path in `StreamEnableWrite4F1AndPollSinkStatus`.
- Existing notes already show `obs_10` secondary rate `20` maps to class `1`, while the Windows `0x111` set is class-`2` gated and the stream-enable path can clear bit `0` again.
- Only revisit this if IDA shows the grouped-commit path recomputes a different link-setting class for the exact iMac secondary.
- Kernel decision this should drive: avoid adding a persistent `0x111` hack unless the exact secondary mode proves class `2`.

`RE-20F`: AUX write failure and retry behavior for link-setting writes.

- IDA anchors: `DpcdAuxWriteUpTo16Bytes` at `0x14154F020`, `DpcdAuxWriteRetryNakSpecialOui` at `0x14154EEB0`, lower training write sites reached from `PerformLinkTrainingWithRetries`.
- Trace how Windows handles failures for the same DPCD writes Linux loses on: `DP_DOWNSPREAD_CTRL`, `DP_LANE_COUNT_SET`, and `DP_LINK_BW_SET`.
- Identify whether Windows retries, delays, reopens AUX, reuses cached settings, or avoids those writes entirely on the already-configured secondary.
- Kernel decision this should drive: either implement a Windows-like cached-link skip, or a very narrow retry/order change if Windows really does fresh writes.

Expected RE outputs before the next kernel edit:

- A concrete sequence diagram for the Windows `0x3113` path from route-ready to stream-enabled.
- A yes/no answer for "does Windows fresh-train the secondary during the userspace grouped commit, or does it use an already-link-configured/cached-link path?"
- A yes/no answer for "does Windows write `0x316` for this exact iMac tile mode?"
- A yes/no answer for "is `0x111` active for the exact secondary mode, or just a class-2 path we should leave alone?"
- A kernel patch recommendation phrased as one of:
  - skip fresh secondary training and only finalize stream/private DPCD when cached settings match;
  - reorder Linux setup so `0x10A`/source-DPCD/optional `0x316` happen before link-setting writes;
  - add a narrow Windows-equivalent AUX retry/order fix for `0x3113`;
  - or document that static RE is exhausted and the next kernel run must instrument the remaining unknown.

`RE-20A RESULT`: initial bring-up and grouped stream-enable are not the same path.

- IDA checkpoint: `C:\proj\reverse\WT6A_INF\B416406\amdkmdag.sys.i64`.
- IDA comments added at:
  - `0x141B6CC3F`: initial lower DP/eDP source-DPCD publication before `PerformLinkTrainingWithRetries`.
  - `0x141446EAB`: later stream-enable skips fresh training when `LinkServiceHasCachedLinkSettings(+0x8C)` is true.
  - `0x1414E78F4`: stream-enable state `1` is an already-link-configured path.
  - `0x1414E7BC0`: normal stream-enable reaches `TryEnableLinkWithHbr2Fallback`.
  - `0x1414E5D1C`: cached-link tuple match skips `ProgramAssrThenTrainLinkSettings`.
  - `0x1414E5D63`: cache mismatch/absence enters the fresh ASSR-before-training path.
- Initial lower bring-up path:
  - `BringUpDpOrEdpWithLinkTraining` at `0x141B6CA70` calls optional signal-`32`/class-`2` `0x111`, optional signal-`128` HPD/AUX callbacks, then `ProgramDpPreTrainingAuxState`, then `PerformLinkTrainingWithRetries`.
  - This is the source-DPCD-before-training path. `ProgramDpPreTrainingAuxState` publishes `0x300`, `0x303`, `0x310`, and optional `0x340` before the lower bring-up training starts.
- Later grouped stream-enable path:
  - `EnableValidatedStreamMaybeAssert4F1` at `0x141446B40` first checks `LinkServiceHasCachedLinkSettings`.
  - If cached link settings exist, Windows does not call the fresh link-enable/training vfunc in that outer path; it proceeds to stream setup.
  - If cached settings are absent, it calls the link-enable vfunc, which reaches `TryEnableLinkWithHbr2Fallback`.
  - `StreamEnableWrite4F1AndPollSinkStatus` at `0x1414E7810` has an even stronger already-configured path: state `1` caches requested settings, programs `0x316`, calls stream-enable, promotes state to `3`, and returns without calling `TryEnableLinkWithHbr2Fallback`.
  - The normal stream-enable path calls `TryEnableLinkWithHbr2Fallback`, but that helper first compares requested link settings against cached settings at link-service `+0x8C/+0x90`. If they match, Windows returns success without `ProgramAssrThenTrainLinkSettings`.
  - Only if the tuple does not match, Windows disables/reprograms through the link-service vfunc and calls `ProgramAssrThenTrainLinkSettings`.
- Fresh stream-enable training path:
  - `ProgramAssrThenTrainLinkSettings` writes/cache-programs the requested settings, computes the `0x3113`-aware ASSR policy, writes DPCD `0x10A`, then calls the actual training helper.
  - The later stream-enable fresh path does not visibly call `ProgramDpPreTrainingAuxState`; source-DPCD publication is a lower initial bring-up/caps path, while stream-enable fresh training is mainly requested-settings/cache plus ASSR `0x10A` plus training.
- RE-20A answer:
  - Windows has a real fresh-training path, but it is not the only grouped stream-enable path.
  - For an already-route-ready/probe-trained secondary, Windows has two ways to avoid destructive fresh training: stream state `1` and cached link settings matching requested settings.
  - Therefore `OBS-12` is suspicious because Linux has already proven the secondary route/AUX state, then still enters normal DP link-setting writes and loses AUX. The Windows analogue would likely avoid that if the stream/link state is classified as already configured or cached-link-compatible.
- Kernel implication:
  - The next kernel change should not blindly add more DPCD writes before the same failing training sequence.
  - The higher-confidence direction is to teach the iMac `0x3113` path to recognize the OBS-12 route/AUX-ready state as an already-configured/cached-link handoff and avoid the destructive fresh link-setting write sequence during the first accepted two-tile commit, while still finalizing stream/private state.
  - `RE-20B` should now be narrowed to exactly what populates and invalidates the cached link settings and how Windows decides state `1` versus normal state before stream-enable.

`RE-20B RESULT`: cached link settings are a selected/requested tuple used to skip fresh training.

- IDA checkpoint: `C:\proj\reverse\WT6A_INF\B416406\amdkmdag.sys.i64`.
- IDA comments added at:
  - `0x1414D1D1F`: `ProgramRequestedLinkSettingsAndCache` stores the 12-byte tuple at link-service `+0x8C/+0x90/+0x94`.
  - `0x141443B7B`: `LinkServiceHasCachedLinkSettings` is only `*(dword *)(service + 0x8C) != 0`.
  - `0x1414E5D1C`: `TryEnableLinkWithHbr2Fallback` cached tuple match skips fresh training.
  - `0x1414E5D94`: `TryEnableLinkWithHbr2Fallback` refreshes the cache after the selected path.
  - `0x141443B1C`: `TryEnableLinkWrapperVerifyCapsAndFallback` refreshes the cache from the selected link-setting table entry.
  - `0x1414DF5F8`: `VerifyLinkCapWithRetry` candidate path programs/caches a candidate before optional training.
  - `0x1414E58A8`: verify-link-cap failure resets selected link-setting index `+0x2E4` to `0`.
- Tuple layout:
  - The cache is 12 bytes starting at DisplayPortLinkService `+0x8C`.
  - The first dword is the lane count. This is supported by the Windows failure log path: `"Unexpected Link Training failure @ %d lane %d*0.27Gbps"` prints tuple dword `0` as lane count.
  - The second dword is link rate in units of `0.27Gbps`. This maps cleanly to observed rate `20` as `5.4Gbps` HBR2.
  - The third dword is copied and table-managed with the tuple, but the top-level cached-link fast path only compares dwords `0` and `1`.
- Cache presence:
  - `LinkServiceHasCachedLinkSettings` does not check HPD, AUX readiness, `link_active`, or training status.
  - It only checks whether dword `0` of the cached tuple is nonzero.
  - Therefore the cache is a lifecycle/selection state used by Windows, not a direct live-HW proof.
- Cache population:
  - `ProgramRequestedLinkSettingsAndCache` calls a lower vfunc at service field `+0x40`, vtable slot `+0xB0`, with a 0x78-byte request block containing the stream/display object and optional 12-byte link tuple.
  - If a tuple argument is present, it then copies that tuple to service `+0x8C`.
  - `ProgramAssrThenTrainLinkSettings` calls this helper before writing `0x10A` and before actual training.
  - `StreamEnableWrite4F1AndPollSinkStatus` state `1` calls this helper, then `0x316`, then stream-enable, and returns without calling the fresh-training helper.
  - `VerifyLinkCapWithRetry` and `TryEnableLinkWithHbr2Fallback` also update the same tuple when choosing a candidate or fallback link setting.
  - `TryEnableLinkWrapperVerifyCapsAndFallback` refreshes `+0x8C` from the selected link-setting table entry indexed by service `+0x2E4` after a successful enable.
- Cache invalidation / fallback behavior:
  - In the traced stream-enable path, Windows does not wipe the cache merely because HPD is low or because it is entering grouped stream-enable.
  - The visible fallback logic overwrites the selected tuple or selected index when verify/training fails, rather than treating HPD instability as an immediate reason to force fresh training.
  - On cache mismatch or absence, `TryEnableLinkWithHbr2Fallback` calls the service vfunc at slot `+0x10` to disable/reprogram before `ProgramAssrThenTrainLinkSettings`.
  - On cache match, that disable/reprogram call and the ASSR-before-training helper are skipped.
- RE-20B answer:
  - Yes, Windows can skip fresh training for a later stream-enable/grouped commit when the requested lane/rate tuple matches cached service state.
  - The cache is not guaranteed proof of an actively trained link by itself. Windows combines it with the display/link lifecycle around a real display object; this matches our Linux requirement to gate only on the proven real `0x3113` route with prior AUX evidence.
  - The exact `state == 1` provenance is still better left to `RE-20C`, but its behavior is already clear enough: it is an already-link-configured stream-enable path that avoids `TryEnableLinkWithHbr2Fallback` entirely.
- Kernel implication:
  - Linux should not emulate this as a broad "ignore HPD and pretend connected" rule.
  - The Windows-shaped Linux condition is narrower: if the iMac secondary `0x3113` route is route/AUX-ready, has prior successful `0x10A`, source-DPCD, and `0x4F1`, and the current/proposed link settings match the preserved `cur_link_settings`, then the first accepted two-tile commit should be allowed to use a cached-link handoff path instead of issuing destructive fresh `dpcd_set_link_settings` writes.
  - The next kernel patch should log the decision as either `cached-link-handoff` or `fresh-retrain-required`, including lane count, rate, current cache validity, route proof, and prior AUX evidence.
  - If we still need more static confidence before coding, `RE-20C` should trace who sets stream state `1`; otherwise RE-20A/B are already enough to justify a targeted Linux cached-link handoff experiment.

Near-term RE, only if more static work is needed before another kernel test:

- `RE-A` now maps the Windows object-table/AUX construction path to Linux's BIOS parser, `dc_link`, DDC, HPD, and link-encoder construction path.
- `RE-B` confirms Windows has a conditional signal-`32` / class-`2` `0x111` bit0 write before source-DPCD and training. Remaining uncertainty is whether the runtime iMac secondary link settings hit class `2` in every relevant boot, not whether the helper exists.
- `RE-CAP` is now complete enough for kernel work: the capability provenance for tags `0x3A`, `0x3B`, and `0x40` is known, and there is no remaining reason to treat `0x205` as mandatory without runtime `0x40` evidence.
- `RE-12` is complete enough for Stage 1: the final grouped transaction point is `ValidateMultiDisplayViewContextAddStreamsPlanesFinalize`, after both real tile streams have been given per-tile source/timing state and before final DC resource/viewport rebuild.
- `RE-13` is complete enough for Stage 1: Windows expects per-endpoint DisplayID/EDID tiled topology; the only observed parser fallback walks the same endpoint's extension chain, and missing endpoint tile metadata is cleared rather than reconstructed from the primary tile.
- `RE-LINK` adds the missing lower-level distinction: Windows can skip fresh stream link training when cached matching link settings exist, then performs the stream-enable `0x4F1` path without an explicit HPD-high/link-active precondition. This is the best current match for Linux's `kernel_obs_9` HPD-low but `link_state_valid=1` secondary state.
- `RE-LINK-ASSR` and `RE-LINK-ASSR-2` add a concrete missed setup candidate: Windows can set DPCD `0x10A` bit `0` for object low byte `0x13` before training and feeds the same policy into training-pattern programming. `obs_10` proves Linux currently skips this because `panel_mode_edp=0`. The next kernel change should remove that advertised-`panel_mode_edp` dependency for the exact proven `0x3113` / DDC3 / UNIPHY_D route and force the Windows-like ASSR policy before source-DPCD/training.
- `RE-LINK-CLASSIFY` narrows the next kernel work further: implement the exact `0x3113` ASSR/panel policy first; keep `0x111`, `0x205`, `0x316`, and `0x32F` as diagnostics unless a later run proves the corresponding Windows gate is active on this panel.
- `RE-19` is now complete enough for Stage 1: the only found Windows boot/preboot pipe-resource reuse path is the type-128 seamless path. No separate signal-32 `0x3113` boot-resource adoption path was found, so the Linux fix should not depend on hidden seamless adoption for the secondary DP tile.
- `RE-16` is now complete enough for Stage 1: Windows protects partial one-tile SetPathMode/ResetPathMode operations by carrying any unmentioned peer display object that is not disabled and still has `display+0x38 != NULL` into the final commit stream array. This should guide Linux peer-preservation once `secondary-stream-installed` is reached.

Plain-boot Stage 2:

- Use the known AMD object ids and route mapping to find where Linux loses or never constructs `0x3113`.
- Make Linux create or expose the real secondary route before normal userspace modeset.
- Feed that route into the same preservation and TILE grouping path proven in Stage 1.

## Evidence Map

Primary raw notebook:

- `C:\proj\reverse\WT6A_INF\RE_5K_iMac19_1.md`

Important result directories referenced by the raw notes:

- `C:\proj\reverse\WT6A_INF\Probe_results`
- `C:\proj\reverse\WT6A_INF\kernel_obs`
- `C:\proj\reverse\WT6A_INF\kernel_obs_2_working`
- `C:\proj\reverse\WT6A_INF\kernel_obs_3`
- `C:\proj\reverse\WT6A_INF\kernel_obs_4K`
- `C:\proj\reverse\WT6A_INF\kernel_obs_gnome`
- `C:\proj\reverse\WT6A_INF\kernel_obs_4_gnome`
- `C:\proj\reverse\WT6A_INF\kernel_obs_5_gnome`
- `C:\proj\reverse\WT6A_INF\kernel_obs_6_gnome`
- `C:\proj\reverse\WT6A_INF\kernel_obs_7_gnome`
- `C:\proj\reverse\WT6A_INF\kernel_obs_8_gnome`
- `C:\proj\reverse\WT6A_INF\kernel_obs_9`

Current Linux tree:

- `C:\proj\reverse\WT6A_INF\linux-imac-5k`

Main kernel files:

- `drivers/gpu/drm/amd/display/amdgpu_dm/amdgpu_dm.c`
- `drivers/gpu/drm/amd/display/amdgpu_dm/amdgpu_dm.h`
- `drivers/gpu/drm/amd/display/amdgpu_dm/amdgpu_dm_quirks.c`

IDA database currently used for Windows RE:

- `C:\proj\reverse\WT6A_INF\B416406\amdkmdag.sys.i64`

Recent IDA breadcrumbs:

- `0x1414B8706`: display-map interface creation from object-table ids.
- `0x1414B875E`: durable per-object DPCD/AUX transport stored at display-map entry `+0x18`.
- `0x141465AA0`: `AdapterServiceGetAuxHandleForObjectId`, object id to descriptor AUX selector/DDC A register plus source mask to AUX factory.
- `0x141465B5C`: per-object AUX handle created from descriptor AUX selector/DDC A register and source mask.
- `0x14152AE80`: `AuxFactoryCreateHandleForSourceMask`, creates dual-pin AUX transaction handle.
- `0x1415B0C20`: `ConstructDualPinAuxTransactionHandle`, opens concrete DDC DATA/CLOCK endpoints after selector-to-pin resolution.
- `0x141695B0A`: DCE11 AUX selector/DDC A register `0x4871` maps to AUX pin index `2` for secondary `0x3113`.
- `0x1415524B4`: class-3 object-id acceptance for `0x3114` and `0x3113`.
- `0x141552658`: low-byte skip policy excludes `0x14` / `0x0E`, not `0x13`.
- `0x1414A9850`: detect/live-state update gate that maps to Linux real-sink preservation.
- `0x1414B4106`: Apple/tiled descriptor byte `+6` bit `3` false-disconnect preservation test.
- `0x1414B417E`: false-disconnect branch restores `result+0x72 connected = 1`.
- `0x1414B418A`: false-disconnect branch clears `result+0x70 changed/rescan = 0`.
- `0x1414B3B45`: live display-object teardown point that clears the 104-byte detection record's `record+0x18`.
- `0x141ABC752`: main vendor/product capability table descriptor, `off_144C4E080 -> 0x144C4F720`, `361` records.
- `0x141A41CF7`: capability-table source id `0x38` maps to tag `0x3A`.
- `0x141A41D01`: capability-table source id `0x39` maps to tag `0x3B`.
- `0x141A41D33`: capability-table source id `0x3E` maps to tag `0x40`.
- `0x144C50070`: static table record `APP/AE25 -> source id 0x38 -> tag 0x3A`, value `0`.
- `0x144C50080`: static table record `APP/AE26 -> source id 0x39 -> tag 0x3B`, value `0`.
- `0x141A27A20`: `TryMarkCompatibleActiveDisplaySetForSpecialCommit`, requires exactly two active displays with stream pointers before grouped validation.
- `0x141A27FD0`: `ValidateActiveDisplaySetViaViewCommitContext`, builds the non-clone view commit context and feeds per-display source/timing state before finalization.
- `0x141ED0DB0`: `CollectActiveDcStreamsFromDisplayObjects`, SetPathMode/ResetPathMode final collector; predicate is non-disabled display object plus live `display+0x38` stream pointer.
- `0x141ED9760`: `DisplayObjectIsDisabledForActiveCommit`, reads byte `display+0x95256`; used by the final collector to skip disabled displays.
- `0x141A22FC7`: SetPathMode stream replacement destroys only the currently requested display object's old stream.
- `0x141A24F29`: ResetPathMode collection point after only explicitly listed display streams have been destroyed.
- `0x1419FB8C0`: `AllocateDcStreamForDisplayObject`, allocates a stream from a display object's selected/special mode block but does not make it live until `display+0x38` is written.
- `0x141A10094`: detect/mode-update temporary stream allocation inside `UpdateEndpointModesAndTiledMetadataAfterPinAttach`; freed at `0x141A114C5`, not installed as `display+0x38`.
- `0x141ED6880`: `BuildDcStreamFromSelectedModeObject`, fills the allocated stream from selected path-mode timing/source state; xrefs are SetPathMode-like paths only.
- `0x141A23045`: main `WindowsDM_SetPathMode_BuildStreamsAndCommit` live-stream birth point, `display_object+0x38 = newly built stream`.
- `0x141B61560`: `AddDcStreamToCommitContextArray`, adds an already-built stream to a pending commit context; it is not the stream builder.
- `0x141AAE3DF`: `DalWDDM_SetPathMode_BuildStreamsAndCommit` live-stream birth point.
- `0x141F0348A`: `Windows7DM_SetPathMode_BuildStreamsAndCommit` live-stream birth point.
- `0x141A87BF0`: `ValidateAndCommitExistingStreamPropertyChange`, clones/revalidates existing live streams only after `display+0x38` already exists.
- `0x141A0A670`: `RebalancePeerStreamsAfterNewStreamBuild`, synchronizes grouped peer streams after a new stream is built, but only across peers that already have live stream pointers.
- `0x141A0A380`: `RebalancePeerStreamsAfterStreamReset`, reset-side peer synchronization for already-live stream groups.
- `0x141A75A70`: `CreateOrGetDisplaySetViewCommitContext`, `a4 == 0` selects `ConstructMultiDisplayViewCommitContext`; `a4 != 0` selects clone/fake-sink construction.
- `0x141ADB980`: `ConstructMultiDisplayViewCommitContext`, creates one real stream/plane pair per active tile.
- `0x141ADB740`: `SetViewContextDisplaySourceSize`, writes per-tile source size into stream and plane state.
- `0x141ADAC70`: `ApplyPathModeTimingToViewContextStream`, applies selected timing to each matching tile stream.
- `0x141ADA460`: `ValidateMultiDisplayViewContextAddStreamsPlanesFinalize`, final grouped transaction over all tile streams/planes.
- `0x141BADD00`: `DcValidateContext_AddStreamViaResourceCallback`, adds each stream to one shared DC validation context.
- `0x141BADE20`: `AttachPlaneStateAndCloneSplitPipeLinks`, attaches each tile plane and preserves/clones split-pipe links.
- `0x141BAE330`: `DcValidateFinalize_RebuildViewportsAndResources`, final resource/viewport rebuild only after grouped stream/plane add succeeds.
- `0x141BA96A0`: `ComputeViewportForSplitPipe`, derives viewport placement from split-pipe count/index.
- `0x141BA9CC0`: `GetSplitPipeCountAndIndex`, walks linked pipe contexts to compute split count and index.
- `0x141A75760`: `ValidateDisplaysShareTiledGroupMetadata`, requires all active endpoints to share valid DisplayID tile group metadata and full grid occupancy.
- `0x141A206C0`: `PopulateEndpointTileMetadataFromDisplayIdTopology`, fills endpoint tile metadata from that display object's own DisplayID parser.
- `0x141A0F2F0`: `UpdateEndpointModesAndTiledMetadataAfterPinAttach`, updates modes and tile metadata after pin attach; clears endpoint tile block when parsing fails.
- `0x141A0FCF5`: failure path that zeroes `endpoint+0x98..0xCF`.
- `0x141A113AE`: peer scan reads peer endpoint tile token for representative election only.
- `0x141A114BA`: writes only the representative byte `endpoint_tile+0x34`.
- `0x141A388A0`: `QueryNextEdidExtensionForTiledTopology`, same-endpoint parser-chain fallback.
- `0x141AEF130`: `ParseDisplayId20TiledTopologyBlock40`, DisplayID 2.0 tiled topology parser for tag `0x28`.
- `0x141AFB700`: `ParseDisplayId13TiledTopologyBlock18`, DisplayID 1.x tiled topology parser for tag `0x12`.
- `0x141AEF176`: DisplayID 2.0 missing-block fallback to same-endpoint next extension parser.
- `0x141AFB746`: DisplayID 1.x missing-block fallback to same-endpoint next extension parser.
- `0x1414E1570`: `ProgramAssrThenTrainLinkSettings`, writes DPCD `0x10A` from ASSR policy before actual link training.
- `0x1414E15D3`: computes `ClassifyDpPanelModeForAssrPolicy` for the stream/display object.
- `0x1414E15E0`: calls `ProgramDpcd10A_AssrBitForPanelMode`, the pre-training `0x10A` RMW.
- `0x1414E5D1C`: cached-link fast path in `TryEnableLinkWithHbr2Fallback`, requested settings equal cached settings.
- `0x1414E5D63`: fresh link path calls `ProgramAssrThenTrainLinkSettings`.
- `0x14154F020`: `DpcdAuxWriteUpTo16Bytes`, normal per-link DPCD/AUX write vfunc.
- `0x14154EEB0`: `DpcdAuxWriteRetryNakSpecialOui`, special OUI `0x90CC24` retry-on-NAK wrapper.
- `0x1414DD030`: `ProgramDpcd316_StreamSourceFlag`, stream-enable DPCD `0x316` RMW; sets bit `0` only for stream mode values `10`, `11`, or `12`.
- `0x141B6CA70`: `BringUpDpOrEdpWithLinkTraining`, lower normal DP/eDP bring-up order.
- `0x141B6CB5F`: signal-`32` / class-`2` gate for DPCD `0x111` bit `0`.
- `0x141B6CC3F`: `ProgramDpPreTrainingAuxState` before actual link training.
- `0x141B6CCA5`: actual link training starts after source-DPCD publication.
- `0x141B8D860`: `PerformLinkTrainingWithRetries`, normal DP training loop.
- `0x141BB7090`: `PrepareLinkTrainingStateAndArmEdp128`, signal-`128` HPD/AUX arming path.
- `0x141B8C880`: `ClassifyDpLinkRateEncodingRange`, link-setting class helper.
- `0x141B8C700`: `WriteDpcd32FForDpEdpSignal`, post-training DPCD `0x32F` helper.
- `0x141BA6DF0`: `MarkSeamlessBootStreamIfResourcesMatch`, only marker for seamless/preboot resource reuse; sets `stream+0x933` only after the type-128 matcher succeeds.
- `0x141B7B830`: `IsType128StreamMatchingBootResources`, hard-gates boot resource matching on `stream->signal == 128`.
- `0x141B7B4E0`: `IsDpOrEdpSignalTypeForBootResourceMatch`, accepts `32/64/128` only after the outer type-128 stream gate has already passed.
- `0x141BA6E80`: `TryMapSeamlessBootPipeResources`, exact boot/preboot pipe-slot reuse path, reachable only when `stream+0x933` is set.
- `0x141BA7B50`: `MapFirstFreePipeResourcesForStream`, generic non-seamless pipe/resource assignment fallback.
- `0x141BA8610`: `RecyclePipeContextForSplitClone`, split/clone resource recycling fallback, not missing-secondary boot adoption.

## One-Sentence Current Model

The iMac19,1 5K solution is not a fake 5K connector and not a magic firmware opcode: it is two real physical 2560x2880 tile streams, a real secondary `0x3113` AUX/link route, Windows-style suppression of false secondary disconnect, correct DisplayID/DRM TILE grouping, and one logical 5120x2880 userspace monitor built from those two real streams.

## Kernel Change After RE-20B: Cached Secondary Link Handoff

Implemented a Windows-shaped cached-link handoff in `linux-imac-5k` after RE-20B clarified that Windows stores a selected/requested 12-byte link tuple and can skip fresh training when the requested tuple matches the cached tuple.

Files changed:

- `drivers/gpu/drm/amd/display/dc/dc.h`
- `drivers/gpu/drm/amd/display/amdgpu_dm/amdgpu_dm.c`
- `drivers/gpu/drm/amd/display/dc/link/link_dpms.c`

Behavior added:

- DM clears stale cached-handoff state before each commit.
- DM arms a one-shot cached handoff only for the real secondary `0x3113` tile stream.
- The arm gate requires two real tile streams, the Windows-equivalent secondary route, strict secondary AUX evidence (`0x10A`, source-DPCD, `0x4F1`), `link_state_valid`, and requested rate/lane matching preserved `cur_link_settings`.
- DC consumes the one-shot flag in `enable_link_dp()`.
- If accepted, DC preserves the already-programmed source-DPCD/cable-ID state, sets up the 8b/10b stream encoder, preserves the selected tuple, marks the link active through the normal caller, and skips the fresh DP training writes that failed in OBS-12 (`DP_DOWNSPREAD_CTRL`, `DP_LANE_COUNT_SET`, `DP_LINK_BW_SET`).
- If rejected, DC logs the exact blocker and falls back to the normal training path.

Expected decisive log strings:

- `IMAC5K: commit_streams_pretrain cached-link-handoff armed`
- `IMAC5K: cached-link-handoff preserving source-DPCD/cable-id writes`
- `IMAC5K: cached-link-handoff accepted`
- Or, if it cannot follow the Windows cached path, `IMAC5K: cached-link-handoff not armed` / `IMAC5K: cached-link-handoff rejected` with the concrete reason.

## RE-20C RESULT: Stream-Enable State Machine Values

Scope checked from notes first:

- `RE-20C` was not meant to rediscover source-DPCD, `0x10A`, cached-link tuple storage, or the broad `0x4F1` order.
- The narrow remaining question was the stream-enable state field at `DisplayPortLinkService+0x58`: what values `0/1/2/3/5` mean, who sets `1`, and whether `1` is a real already-link-configured state relevant to the iMac secondary.

IDA checkpoint:

- `C:\proj\reverse\WT6A_INF\B416406\amdkmdag.sys.i64`

IDA comments/renames added:

- `0x1414E1670`: `RecoverActiveDpLinkSettingsFromDpcd`.
- `0x1414E7680`: `SetState1IfDpcdReportsTrainedLink`.
- `0x1414D2070`: `SetState1DefaultLinkTuple`.
- `0x1414D2100`: `DisableStreamAndClearEnableState`.
- `0x1414E7760`: `DisableStreamIfStateActive`.
- `0x1414D3910`: `BlankStreamSetState4`.
- `0x1414D3990`: `UnblankStreamPromoteState2To3`.
- Comments added at `0x1414E1670`, `0x1414E76DF`, `0x1414E770F`, `0x1414D20AB`, `0x1414D1F23`, `0x1414D1F7C`, `0x1414D226D`, `0x1414D240E`, `0x1414D2472`, `0x1414468EF`, `0x1414D396D`, and `0x1414D3ACB`.

State field:

- The state field is `service+0x58`, decompiled as `a1+88` in `StreamEnableWrite4F1AndPollSinkStatus`.
- `state == 0`: inactive/off/cleared. `DisableStreamAndClearEnableState` writes `0` after stream-off teardown.
- `state == 1`: already-link-configured pending stream-enable. This is the special path that programs/caches the requested tuple, optionally writes `0x316`, calls the stream-enable vfunc, promotes to `3`, and returns without `TryEnableLinkWithHbr2Fallback`.
- `state == 2`: normal enabled/completed. `StreamEnableWrite4F1AndPollSinkStatus` writes `2` after the normal path: optional `0x111` clear, optional cap verify, cached-or-fresh link enable, `0x316`, stream-enable vfunc, `0x4F1`, optional `0x205`.
- `state == 3`: state-1 enabled/completed. The special state-1 path writes `3` after stream-enable. The function treats `2` and `3` as already handled and immediately returns success.
- `state == 4`: blank/quiesce transition. A separate helper writes `4` after the lower vfunc when the stream is active and not already `5`; this is not the already-link-configured path.
- `state == 5`: quiesced/off-ish state. A vfunc wrapper writes `5` after a lower vfunc call; disable helpers treat `0` and `5` as already safe/no-op, and an adjacent helper normalizes `5` back to `0` before another transition. It is not the already-link-configured state.

Important state-1 constructors:

- `SetState1DefaultLinkTuple` at `0x1414D2070` writes a fixed tuple `{lane_count=4, link_rate=10, flags=0}` into `+0x64/+0x68/+0x6C`, then sets state `1`.
- `SetState1IfDpcdReportsTrainedLink` at `0x1414E7680` is the important one for the iMac case:
  - It starts with the same default tuple `{4, 10, 0}`.
  - It calls `RecoverActiveDpLinkSettingsFromDpcd`.
  - It only sets state `1` if recovered lane count and link rate are nonzero.
  - It copies the recovered 12-byte tuple into `+0x64/+0x68/+0x6C`.

`RecoverActiveDpLinkSettingsFromDpcd` details:

- Reads DPCD `0x100..0x101`.
  - Byte `0x100` becomes the link rate.
  - Byte `0x101 & 0x1f` becomes lane count.
- Reads DPCD `0x003`.
  - Bit `0` becomes tuple spread flag `0x10`.
- Reads DPCD `0x202..0x203`.
  - Requires trained/locked lane-status bits for the reported lane count.
  - If the status bits do not prove an already trained link, the recovered tuple is zeroed and state `1` is not set.

RE-20C answer:

- Yes, Windows has a stronger already-link-configured state than just "cached tuple exists".
- State `1` can be created from actual DPCD evidence that the link is already trained and locked.
- Once state `1` is set, the stream-enable function avoids `TryEnableLinkWithHbr2Fallback` entirely and therefore avoids fresh link-setting writes.
- State `5` is not relevant to preserving 5K; it is a quiesced/off state in the disable side of the lifecycle.

Linux implication:

- The cached-link handoff patch we just added still matches Windows conceptually, but RE-20C shows an even better diagnostic/possible future gate: if Linux can still read DPCD `0x100`, `0x101`, `0x202`, and `0x203` on the secondary `0x3113` before the first two-tile commit, it can prove the state-1 equivalent instead of relying only on stored `cur_link_settings`.
- If the next kernel run fails despite `cached-link-handoff accepted`, the next targeted Linux change should log those DPCD bytes at the handoff point and, if they indicate a trained link, treat that as the Windows `state == 1` equivalent.

## Kernel Change After RE-20C: Windows-Like Stream-Enable State

Files changed:

- `C:\proj\reverse\WT6A_INF\linux-imac-5k\drivers\gpu\drm\amd\display\dc\dc.h`
- `C:\proj\reverse\WT6A_INF\linux-imac-5k\drivers\gpu\drm\amd\display\amdgpu_dm\amdgpu_dm.c`
- `C:\proj\reverse\WT6A_INF\linux-imac-5k\drivers\gpu\drm\amd\display\dc\link\link_dpms.c`

Behavior now aligned to RE-20C:

- Added a per-link `dc_imac5k_stream_enable_state` with Windows-equivalent values `0..5`.
- Before arming the secondary `0x3113` cached-link handoff, Linux now reads the same DPCD proof bytes that Windows uses for `state == 1`: `0x100`, `0x101`, `0x003`, `0x202`, and `0x203`.
- The handoff is armed only when the DPCD bytes prove an already trained/locked link and the recovered tuple matches both the requested stream tuple and Linux's current preserved tuple.
- The cached handoff now stores the DPCD-recovered tuple, not merely the previous Linux `cur_link_settings`.
- DC `enable_link_dp()` now accepts the handoff only from stream state `1` plus valid DPCD trained-link proof; on success it promotes the link to state `3`.
- If the normal training path is used on a tracked iMac5K link and succeeds, the state becomes `2`; if it fails, the state returns to `0`.
- If Linux disables an already tracked link before a new enable, it records state `4` before `disable_link()` and state `5` after disable completes.

New decisive log strings:

- `IMAC5K: ... Windows-state1 DPCD recovery ...`
- `IMAC5K: ... stream-enable-state ... -> 1:already-link-configured-pending-stream-enable ...`
- `IMAC5K: cached-link-handoff armed ... stream_state=1:already-link-configured-pending-stream-enable ...`
- `IMAC5K: cached-link-handoff accepted ... stream_state=3:state1-enabled-completed ...`
- Failure logs now distinguish `dpcd-trained-proof-missing`, `requested-dpcd-mismatch`, `current-dpcd-mismatch`, and `stream-state-not-1`.

## OBS-13 RESULT: RE-20C Gate Proved No Trained Secondary Link

Input log:

- `C:\proj\reverse\WT6A_INF\obs_13\dmesg.txt`

This run is from the RE-20C kernel:

- `Windows-state1 DPCD recovery` is present.
- `stream-enable-state` transition logs are absent, which means the Windows-state1 path never armed or promoted.
- `cached-link-handoff accepted` is absent.

What worked:

- The early route and metadata discovery still match the Windows map:
  - primary `0x3114`, signal `128`, DDC line/channel `3`, transmitter `2`.
  - secondary `0x3113`, signal `32`, DDC line/channel `2`, transmitter `3`.
- The DRM TILE blobs are correct and Mutter-shaped:
  - eDP-1: `1:1:2:1:0:0:2560:2880`.
  - DP-1: `1:1:2:1:1:0:2560:2880`.
- The readiness barrier rejected 251 pre-handoff one-tile commits, mostly from `plymouthd`.
- At `11.105s`, the barrier accepted one real two-tile commit:
  - two active connectors: eDP-1 and DP-1.
  - two active CRTCs, each `2560x2880`.
  - one shared framebuffer `5120x2880`, pitch `20480`.
  - left plane source `0,0 2560x2880`.
  - right plane source `2560,0 2560x2880`.
- The kernel/DC logical model at that moment is the desired Windows-like two-real-stream model.

What failed:

- Before normal link training, secondary AUX still worked and DPCD link settings were readable/programmed:
  - `0x100 = 0x14` (`HBR2` / rate 20).
  - `0x101 = 0x84` (4 lanes plus enhanced framing bit).
  - `0x003 = 0x01`.
  - `0x4F1 = 0x01`.
- But DPCD lane status was not trained:
  - `0x202 = 0x00`.
  - `0x203 = 0x00`.
  - `trained=0`.
- Therefore the RE-20C Windows-state1 gate correctly refused to arm cached handoff:
  - first reason: `link-state-invalid`, because Linux had no valid active link state yet.
  - later reason: `dpcd-trained-proof-missing`, after AUX/HPD were gone.
- Linux then fell back into the generic DP training path, which again failed writing:
  - `DP_DOWNSPREAD_CTRL`.
  - `DP_LANE_COUNT_SET`.
  - `DP_LINK_BW_SET`.
  - final failure: `enabling link 1 failed: 15`.
- After that failure, secondary AUX/HPD were effectively dead:
  - `hpd=0/0`.
  - DPCD reads return `-1`.
  - repeated `0x4F1` latch writes fail.

Conclusion:

- OBS-13 proves the current failure is no longer primarily a Mutter/TILE/topology problem.
- The kernel now reaches the correct two-stream logical display model.
- The missing piece is the physical secondary `0x3113` bring-up sequence between "DPCD settings are programmed/readable" and "lane status reports trained/locked".
- We should not loosen the state-1 handoff to ignore `0x202/0x203`; Windows RE says state `1` requires trained-link proof, and OBS-13 shows that proof is absent.
- The next target is the Windows normal/pre-training path that turns the secondary route into a trained link without the destructive Linux generic DP link-setting sequence.

Best next RE target:

- Trace the Windows path around `BringUpDpOrEdpWithLinkTraining`, `ProgramDpPreTrainingAuxState`, `PerformLinkTrainingWithRetries`, and the lower stream-enable/source-output calls for the `0x3113` signal-32 route.
- Specifically answer what Windows does after programming source-DPCD/ASSR/`0x4F1` when `0x100/0x101` are set but `0x202/0x203` are zero.
- Determine whether Windows enables the source PHY/stream encoder first, avoids rewriting link-setting DPCD bytes in this case, uses a special AUX retry/OUI wrapper, or calls an Apple-specific lower transport before normal DP training.

## RE-21 PLAN: Secondary 0x3113 Trained-Link Bring-Up

Goal:

- Explain the missing transition observed in OBS-13: secondary `0x3113` has readable/programmed DPCD settings (`0x100=0x14`, `0x101=0x84`, `0x003=0x01`) but no trained lane status (`0x202=0x00`, `0x203=0x00`).
- Convert Windows behavior into the next Linux patch with as little guessing as possible.

Passes:

- `RE-21A`: Map normal secondary stream-enable order from `StreamEnableWrite4F1AndPollSinkStatus` through cached/fresh link enable.
- `RE-21B`: Trace link-setting programming and determine whether Windows rewrites DPCD `0x100/0x101/0x107`, in what order, and with what chunking.
- `RE-21C`: Trace actual training sequence and lane-status polling when the link starts untrained.
- `RE-21D`: Trace AUX/HPD failure tolerance and retry policy.
- `RE-21E`: Identify lower source-output / PHY / training-pattern vfuncs that may be missing from Linux.
- `RE-21F`: Turn confirmed Windows sequence into a Linux patch checklist, separating safe changes from new log probes.

## RE-21A RESULT: Normal Stream-Enable Spine

IDA checkpoint:

- `C:\proj\reverse\WT6A_INF\B416406\amdkmdag.sys.i64`

Anchors:

- `0x1414E7810`: `StreamEnableWrite4F1AndPollSinkStatus`
- `0x1414E5CB0`: `TryEnableLinkWithHbr2Fallback`
- `0x1414E1570`: `ProgramAssrThenTrainLinkSettings`
- `0x141B6CA70`: `BringUpDpOrEdpWithLinkTraining`

Result:

- State `2` or `3` returns success immediately.
- State `1` is the already-link-configured path:
  - `ProgramRequestedLinkSettingsAndCache`
  - `ProgramDpcd316_StreamSourceFlag`
  - stream-enable helper
  - real stream-enable vfunc `(*service + 0x98)`
  - state promotes to `3`
  - no call to `TryEnableLinkWithHbr2Fallback`
- Normal state-0 path:
  - maybe clear DPCD `0x111`
  - maybe verify/retry caps
  - `TryEnableLinkWithHbr2Fallback`
  - `ProgramDpcd316_StreamSourceFlag`
  - stream-enable helper
  - real stream-enable vfunc `(*service + 0x98)`
  - tag-gated `0x4F1 = 1`
  - optional tag-gated `0x205` poll
  - state becomes `2`

Linux implication:

- OBS-13 did not satisfy state `1`, so Linux must either reproduce the normal state-0 training path correctly or get the secondary into real state-1 before the grouped commit.

## RE-21B RESULT: Windows Link-Settings DPCD Write Order

IDA names/comments added:

- `0x1414DD760`: `DpcdSetLinkSettingsWrite100101Then107`

Important result:

- Windows writes link rate and lane count as one two-byte DPCD transfer:
  - DPCD `0x100`, length `2`
  - payload byte 0: link rate
  - payload byte 1: lane count OR `0x80`
- Windows writes spread/downspread separately after that:
  - DPCD `0x107`, length `1`
- This is visible in the log string:
  - `DisplayPortLinkService::dpcdSetLinkSettings`
  - `DPCD W2 0X000100 ...`
  - `DPCD W1 0X000107 ...`

Why this matters:

- OBS-13 Linux failed in `dpcd_set_link_settings` on:
  - `DP_DOWNSPREAD_CTRL`
  - `DP_LANE_COUNT_SET`
  - `DP_LINK_BW_SET`
- The Linux failure order is spread/lane/bw, while Windows writes rate+lane first as one combined transaction and spread second.
- This is one of the best concrete Linux patch candidates from RE-21 so far.

Important caution:

- `ProgramRequestedLinkSettingsAndCache` caches the requested 12-byte tuple at link-service `+0x8C/+0x90/+0x94`.
- That cache is requested/selected state, not proof of a trained link. OBS-13 confirms why this distinction matters.

## RE-21C RESULT: Training Sequence After ASSR

IDA names/comments added:

- `0x1414D9FD0`: `TrainLinkWithAssrRetryOnceOnStatus6`
- `0x1414DCC10`: `SetHwTrainingPatternWithAssrPolicy`
- `0x1414DC560`: `DpcdSetTrainingPatternAndLaneSettings`

Anchors:

- `0x1414D9040`: `PerformLinkTrainingWithAssrPatternPolicy`
- `0x141B8D860`: `PerformLinkTrainingWithRetries`
- `0x141BB7090`: `PrepareLinkTrainingStateAndArmEdp128`

Result:

- `ProgramAssrThenTrainLinkSettings` order is:
  - `ProgramRequestedLinkSettingsAndCache`
  - `ClassifyDpPanelModeForAssrPolicy`
  - `ProgramDpcd10A_AssrBitForPanelMode`
  - `TrainLinkWithAssrRetryOnceOnStatus6`
- The retry wrapper calls `PerformLinkTrainingWithAssrPatternPolicy`; if the result is status `6`, it retries once.
- The lower initial bring-up path `BringUpDpOrEdpWithLinkTraining` is separate:
  - optional signal-32/class-2 `0x111`
  - signal-128-only HPD/AUX callbacks
  - `ProgramDpPreTrainingAuxState`
  - `PerformLinkTrainingWithRetries`
- `PrepareLinkTrainingStateAndArmEdp128` repeats HPD/AUX arm callbacks only for signal/type `128`.
- The iMac secondary `0x3113` is signal `32`, so it does not automatically get that eDP/type-128 HPD/AUX re-arm path.

Linux implication:

- We should not expect a hidden Windows signal-32 HPD re-arm equivalent inside `PrepareLinkTrainingStateAndArmEdp128`.
- For `0x3113`, the important lower differences are the link-setting write order, training-pattern backend request, ASSR policy propagation, and AUX transaction behavior.

## RE-21D RESULT: AUX Retry Scope

Anchors:

- `0x14154F020`: `DpcdAuxWriteUpTo16Bytes`
- `0x14154EEB0`: `DpcdAuxWriteRetryNakSpecialOui`
- `0x141EDFC70`: `LowerControllerDpcdAuxWriteAddress`
- `0x141EDDCD0`: `LowerControllerDpcdAuxReadAddress`

Result:

- Normal DPCD writes build and submit one AUX write transaction, then map the result.
- The special retry wrapper only triggers for OUI `0x90CC24`.
- That wrapper retries NAK/defer up to seven times.
- It does not contain a hidden HPD/AUX re-arm or wake sequence.
- The lower read/write wrappers dispatch to miniport AUX vfuncs:
  - write vfunc `+0x190`
  - read vfunc `+0x188`
  - both use the link/session id from the display/link object.

Linux implication:

- The special OUI retry is probably not enough to rescue OBS-13 after AUX becomes fully dead (`status=-1`, HPD `0/0`).
- If Linux is going to follow Windows here, it needs to avoid killing AUX in the first place, likely by changing the pre-training/link-settings sequence before the generic Linux failure point.

## RE-21E RESULT: ASSR/Panel Policy Also Reaches PHY Training

IDA names/comments added:

- `0x141BB7510`: `WriteDpSetPowerDpcd600`

Anchors:

- `0x1414D8EE0`: `SetDpPhyPatternWithAssrPolicy`
- `0x141BB7510`: `WriteDpSetPowerDpcd600`

Result:

- Windows does not only write ASSR policy into DPCD `0x10A`.
- `SetDpPhyPatternWithAssrPolicy` also calls `ClassifyDpPanelModeForAssrPolicy` and passes that result into the lower backend vfunc at link-service backend `+0xC8`.
- Training preparation writes DPCD `0x600` through `WriteDpSetPowerDpcd600` after source-side training state is prepared.

Linux implication:

- A Linux patch that only writes `0x10A` may still be incomplete.
- The source/PHY training-pattern setup may need an iMac5K-specific ASSR/panel-mode flag equivalent, or at least logging around the AMD DC calls that set PHY/training patterns for DP-1.

## RE-21F DRAFT: Current Linux Patch Candidates

Candidates that now look justified by RE plus OBS-13:

- For the iMac secondary `0x3113` route, before normal DP training, try the Windows link-setting DPCD order:
  - combined write to `0x100` length `2` for rate+lane.
  - then write `0x107` length `1` for spread.
  - avoid Linux's current spread-first separate write order for this quirk path.
- Keep the strict state-1 proof gate. Do not fake trained state when `0x202/0x203` are zero.
- Add logs around Linux PHY/training-pattern programming for the secondary route, including any ASSR/panel-mode inputs.
- Consider a quirk-specific training path that mirrors Windows order:
  - source-DPCD already programmed.
  - `0x10A` ASSR asserted.
  - Windows-order `0x100/0x101` combined write.
  - `0x107` spread write.
  - PHY training-pattern setup with ASSR/panel policy.
  - normal lane-status polling.

Candidates still needing more RE or code inspection before patching:

- Exact mapping of Windows backend vfunc `+0xB0` / `+0xC8` to Linux DC source/PHY programming calls.
- Whether Windows writes DPCD `0x600` in the same grouped stream-enable path used for the iMac secondary, or only in the lower initial bring-up path.
- Whether the Linux DP training helper can be safely given a pre-programmed link-setting state without immediately redoing the failing spread-first writes.

## KERNEL PATCH AFTER RE-21B: Secondary 0x3113 Windows-Order Link Settings

Implemented in `linux-imac-5k/drivers/gpu/drm/amd/display/dc/link/protocols/link_dp_training.c`.

What changed:

- Added a narrow helper for the proven iMac5K secondary route:
  - raw display object `0x3113`;
  - `SIGNAL_TYPE_DISPLAY_PORT`;
  - DDC hw instance `2`;
  - `TRANSMITTER_UNIPHY_D`;
  - 8b/10b link training only;
  - no `use_link_rate_set` path.
- The original helper was additionally gated on DM's iMac5K AUX evidence bits for both `0x10A` and source-DPCD.
- OBS-14 showed that this was too strict or placed at the wrong layer: DM logged evidence before training, but `dpcd_set_link_settings()` still skipped the Windows-order path.
- The follow-up patch removes AUX evidence as a hard gate for the exact route; evidence remains logged as diagnostics via `evidence=` and `missing_evidence=`.
- Inside `dpcd_set_link_settings()`, after AMD has built the normal lane-count/enhanced-framing byte, the quirk path bypasses the generic Linux order and writes:
  - DPCD `0x100`, length `2`: link rate byte plus lane-count/enhanced-framing byte;
  - DPCD `0x107`, length `1`: spread byte.
- The normal link-training code still runs afterward; this patch does not fake trained lane status and does not loosen the state-1 proof gate.

Expected runtime logs:

- `IMAC5K: secondary 0x3113 Windows-order link-settings begin ...`
- `IMAC5K: secondary 0x3113 Windows-order link-settings complete ... normal lane-status training follows`
- If the route ever falls through to generic Linux ordering, the skip must be explicit:
  - `IMAC5K: secondary 0x3113 Windows-order skipped before generic link-settings reason=...`
- When the exact route enters the helper, the helper now logs the real sink-side preconditions immediately before the Windows-order write:
  - `IMAC5K: secondary 0x3113 Windows-order precondition snapshot ...`
  - includes DPCD `0x10A`, `0x310..0x312`, `0x4F1`, `0x205`, `0x100/0x101`, and `0x202/0x203`.
- If AUX write fails, the failing write is explicit as either `0x100/0x101` or `0x107`.

Why this is the right next test:

- OBS-13 showed the secondary route had readable/programmed requested settings but untrained lanes (`0x202=0x00`, `0x203=0x00`).
- RE-21B showed Windows writes rate+lane as one transaction to `0x100` before spread at `0x107`.
- The previous Linux failure happened in the generic separate/spread-first `dpcd_set_link_settings()` sequence.

## OBS-14 RESULT: Windows-Order Helper Was Built But Did Not Fire

Input log:

- `obs_14/dmesg.txt`
- Kernel build: `7.0.1-1-imac-5k-gaf5cb65be3f5`, built `Mon, 11 May 2026 12:55:30 +0000`.

What OBS-14 confirmed:

- The build included the recent naming pass: logs use `ddc_a_reg=0x4871` for secondary and `ddc_a_reg=0x4875` for primary.
- The topology path is good before userspace:
  - primary `0x3114` / eDP-left and secondary `0x3113` / DP-right are both present;
  - both TILE halves are exposed;
  - two CRTCs scan `2560x2880` each from one shared `5120x2880` framebuffer;
  - the right tile uses source offset `2560,0`.
- Secondary AUX setup succeeds before training:
  - DPCD `0x10A` bit 0 asserted;
  - source-DPCD `0x310 = 04 1D 03` programmed;
  - private sink latch `0x4F1 = 1` asserted.
- The state-1 proof correctly refused to treat the link as already trained because lane status remained untrained:
  - `0x202 = 0x00`;
  - `0x203 = 0x00`.

Critical failure:

- No `IMAC5K: secondary 0x3113 Windows-order link-settings begin` line appeared.
- Linux still executed the generic spread-first/separate-write path:
  - `DP_DOWNSPREAD_CTRL` failed;
  - `DP_LANE_COUNT_SET` failed;
  - `DP_LINK_BW_SET` failed;
  - `enabling link 1 failed: 15`.

Conclusion:

- OBS-14 did not test the Windows-order training hypothesis because the helper did not execute.
- The next kernel patch should make the exact `0x3113` / DDC3 / UNIPHY_D / 8b10b / no-link-rate-set route enter the Windows-order path without requiring AUX evidence as a hard gate.
- If that exact route ever skips the helper again, the log must show the specific skip reason before generic DPCD writes.
- Because `0x10A` and source-DPCD are real Windows-sequence preconditions, not just bookkeeping, the helper should also log their actual DPCD state in the lower training layer. That is stronger than relying on Linux-side evidence-bit propagation.

## OBS-15 RESULT: Windows-Order Helper Runs, But Source-DPCD Is Too Late

Input log:

- `obs_15/dmesg.txt`
- Kernel build: `7.0.1-1-imac-5k-gba4b12918019`, built `Mon, 11 May 2026 15:47:05 +0000`.

What OBS-15 confirmed:

- The Windows-order helper now executes on the exact secondary route:
  - raw object `0x3113`;
  - DDC hw instance `2`;
  - transmitter `UNIPHY_D`;
  - 8b/10b;
  - no link-rate-set path.
- The first two Windows-order attempts happen while HPD/AUX are still alive and both complete successfully:
  - DPCD `0x100/0x101` is written as one two-byte transaction;
  - DPCD `0x107` follows;
  - no spread-first generic Linux sequence is used for this route.
- The precondition snapshot proves Linux-side evidence bits are not reliable at this lower layer:
  - `evidence=0x0`;
  - `missing_evidence=0x3`;
  - but the real sink state already has `0x10A=0x01`, `assr=1`, and `0x4F1=0x01`.
- The same snapshot proves source-DPCD is still not in place at the first low training writes:
  - `0x310=00 00 00`;
  - `source_like=0`.
- Shortly afterward, the DM pretrain path does program source-DPCD successfully:
  - `0x310=04 1d 03`;
  - `status=1`;
  - source-DPCD flag becomes programmed.
- The first atomic handoff still reaches the desired KMS topology:
  - two real tile streams;
  - eDP-left and DP-right;
  - one shared `5120x2880` framebuffer;
  - secondary/right viewport and plane source offset `2560,0`.
- The final secondary training/retry happens after the secondary AUX/HPD path has already collapsed:
  - `hpd=0/0`;
  - all DPCD reads return `status=-1`;
  - Windows-order `0x100/0x101` writes fail with `status=-1`;
  - `enabling link 1 failed: 15`.

Conclusion:

- OBS-15 did test the Windows-order write path.
- Windows-order link-setting order is not sufficient if source-DPCD is only programmed later by the DM pretrain hook.
- Re-adding Linux evidence bits as a hard gate would be the wrong fix: they are bookkeeping and were stale/missing in the lower helper.
- The next Linux direction should make the exact `0x3113` lower training path ensure the real preconditions directly before link-settings/training:
  - confirm or set DPCD `0x10A` bit 0;
  - write/source-DPCD `0x310 = 04 1D 03` when missing;
  - then run the Windows-order `0x100/0x101` combined write and `0x107` write while HPD/AUX are still alive.
- If that still leaves lane status `0x202/0x203 = 0`, then the remaining gap is lower PHY/training-pattern policy, not source-DPCD ordering.

## RE-22 RESULT: Windows Timing/Sequencing Around Source-DPCD And Training

Question:

- OBS-15 proved Linux can now run the Windows-order `0x100/0x101` plus `0x107` write path, but it also proved source-DPCD `0x310` was still `00 00 00` at the first lower training writes.
- The narrow RE question was whether Windows has the same timing gap, whether it relies on delays, or whether it avoids being early/late through a different ownership/state model.

IDA checkpoint:

- `C:\proj\reverse\WT6A_INF\B416406\amdkmdag.sys.i64`

Functions checked:

- `0x141B6CA70`: `BringUpDpOrEdpWithLinkTraining`
- `0x141B8E090`: `ProgramDpPreTrainingAuxState`
- `0x141B84040`: `RetrieveDpLinkCapsAfterSourceDpcdSetup`
- `0x1414E7810`: `StreamEnableWrite4F1AndPollSinkStatus`
- `0x1414E5CB0`: `TryEnableLinkWithHbr2Fallback`
- `0x1414E1570`: `ProgramAssrThenTrainLinkSettings`
- `0x1414D9040`: `PerformLinkTrainingWithAssrPatternPolicy`
- `0x1414D8EE0`: `SetDpPhyPatternWithAssrPolicy`

Static result:

- Windows does not publish source-DPCD from a late high-level grouped-view/desktop commit hook.
- `ProgramDpPreTrainingAuxState` writes the source-specific DPCD block before receiver-cap reads and before real link training:
  - `0x300`: source OUI, payload `00 00 1A` when needed;
  - `0x303`: 9-byte source descriptor block;
  - `0x310`: source table revision, normally `04 1D 03` for the DCE path we mapped;
  - `0x340`: optional source minimum-hblank byte when gated.
- Xrefs to `ProgramDpPreTrainingAuxState` are lower detect/caps/bring-up paths:
  - `BringUpDpOrEdpWithLinkTraining`;
  - `DcLinkDetectHelper` signal-128 fast path;
  - `RetrieveDpLinkCapsAfterSourceDpcdSetup`.
- No normal `StreamEnableWrite4F1AndPollSinkStatus` xref to `ProgramDpPreTrainingAuxState` was found. This is important: the stream-enable path assumes the source-DPCD publication already happened lower down.

Initial lower bring-up order:

- In `BringUpDpOrEdpWithLinkTraining`:
  - optional DPCD `0x111` bit0 write when signal is `32` and link-rate class is `2`;
  - signal-`128` only HPD/AUX readiness callbacks;
  - `ProgramDpPreTrainingAuxState`;
  - optional short delay when link flags request it;
  - `PerformLinkTrainingWithRetries`;
  - post-training helpers such as `WriteDpcd32FForDpEdpSignal` when gated.
- Therefore Windows is not early like our OBS-15 path: it does not perform the lower link-settings/training writes while `0x310` is still zero.

Detect/caps order:

- In `RetrieveDpLinkCapsAfterSourceDpcdSetup`:
  - Windows calls `ProgramDpPreTrainingAuxState`;
  - waits briefly;
  - then reads receiver caps from DPCD `0x000`.
- This means Windows tries to publish the AMD source identity before some capability discovery, not only before final training.

Fresh stream-enable order:

- In `StreamEnableWrite4F1AndPollSinkStatus`, the normal state-0 path is:
  - maybe clear DPCD `0x111` bit0;
  - optional link-cap verify/retry;
  - `TryEnableLinkWithHbr2Fallback`;
  - `ProgramDpcd316_StreamSourceFlag`;
  - stream-enable vfunc;
  - optional DPCD `0x4F1 = 1`, with one retry after `10 ms`;
  - optional DPCD `0x205` poll, up to `50` times with `1 ms` sleeps, only when the descriptor bit is present.
- `TryEnableLinkWithHbr2Fallback` first compares the requested tuple with cached link settings at the link service. If they match, it returns success without fresh training.
- If it must fresh-train, it calls `ProgramAssrThenTrainLinkSettings`.
- `ProgramAssrThenTrainLinkSettings` order is:
  - `ProgramRequestedLinkSettingsAndCache`;
  - `ClassifyDpPanelModeForAssrPolicy`;
  - `ProgramDpcd10A_AssrBitForPanelMode`;
  - `TrainLinkWithAssrRetryOnceOnStatus6`.
- This path writes ASSR DPCD `0x10A` before training, but does not republish source-DPCD `0x310`; it relies on the earlier lower detect/caps/bring-up publication.

Already-configured stream-enable order:

- `StreamEnableWrite4F1AndPollSinkStatus` state `1` is the Windows "already-link-configured, pending stream-enable" path:
  - program/cache requested link tuple;
  - optionally write `0x316`;
  - call the stream-enable vfunc;
  - promote state to `3`;
  - return success without `TryEnableLinkWithHbr2Fallback`.
- States `2` and `3` return success immediately.
- This is how Windows avoids being late/destructive: if the link was already configured, stream enable does not repeat fresh DPCD link-settings writes after the route may have become unstable.

Training-pattern policy:

- `SetDpPhyPatternWithAssrPolicy` recomputes `ClassifyDpPanelModeForAssrPolicy` and passes the policy into the lower training-pattern backend.
- For the iMac secondary object low byte `0x13`, previous RE showed that policy can become ASSR/panel-mode policy `1`.
- This means a correct Linux mirror probably needs the policy in both places:
  - DPCD `0x10A` bit0 before training;
  - lower PHY/training-pattern setup, not just a raw DPCD write.

Conclusion:

- Windows avoids the early/late race mostly through ordering and ownership, not by a large magic delay.
- Source-DPCD is a lower link/caps/training responsibility in Windows, not a high-level modeset afterthought.
- Later stream-enable is protected by state and cached-link paths so it usually does not destructively retrain an already configured route.
- For Linux, the next targeted patch should move/duplicate source-DPCD publication for the exact secondary `0x3113` lower route so it happens directly before Windows-order link-settings/training, while AUX is alive.
- If lanes still remain untrained after that, the next remaining gap is the lower PHY/training-pattern policy path that carries the same ASSR/panel classification into the backend.

## RE-23 RESULT: Source-DPCD Payload Ownership Versus Linux

Question:

- Can we reduce the next kernel attempt by confirming exactly which source-DPCD bytes Windows publishes, and whether Linux already has a lower hook where this belongs?

IDA anchors:

- `0x141B8E090`: `ProgramDpPreTrainingAuxState`
- `0x141B8413D`: caps path calls `ProgramDpPreTrainingAuxState`
- `0x141B6CC3F`: lower bring-up calls `ProgramDpPreTrainingAuxState`

Windows source-DPCD payload:

- `0x300`: source OUI, written as `00 00 1A` when not already present.
- `0x303`: 9-byte source/device descriptor block.
- `0x310`: 3-byte source table revision:
  - byte 0 is `4` for the DCE-era path we observed;
  - byte 1 is `0x1D`;
  - byte 2 is `0x03`.
- `0x340`: optional minimum-horizontal-blank byte when the gated source capability is nonzero.
- There is also a separate branch that can write a prebuilt 12-byte block at `0x300`; that looks like a precomputed replacement for the same source block, not a separate Apple-private operation.

Linux cross-check:

- Linux already has `dpcd_set_source_specific_data()` in `link_dp_capability.c`.
- That helper writes:
  - `DP_SOURCE_OUI` / `0x300`;
  - `DP_SOURCE_OUI + 0x03` / `0x303`;
  - optional `DP_SOURCE_MINIMUM_HBLANK_SUPPORTED` / `0x340`.
- It is called from lower link/caps and DPMS paths, including before training in `link_dpms.c`.
- It does **not** write the Windows `0x310 = 04 1D 03` source table revision.
- Our current iMac helper writes only `0x310`, and it lives in `amdgpu_dm.c`, above the lower training/link-settings owner.

Conclusion:

- This is now a very concrete kernel candidate:
  - keep Linux's generic `dpcd_set_source_specific_data()` as the lower owner of source-DPCD;
  - add a narrowly gated `0x3113` / DDC3 / UNIPHY_D write of `0x310 = 04 1D 03` in the same lower timing window;
  - stop relying on the higher DM pretrain hook as the only `0x310` writer.
- This directly addresses OBS-15: Windows-order link-settings ran while `0x310` was still zero.

## RE-24 RESULT: Training Pattern / PHY Policy Loop

Question:

- If `0x310` is fixed and lane status still remains zero, what is the next real Windows-side gap: a private Apple call, a PHY/training-pattern policy, or normal DP training?

IDA anchors:

- `0x1414D9040`: `PerformLinkTrainingWithAssrPatternPolicy`
- `0x1414DB6C0`: legacy 8b/10b clock-recovery phase
- `0x1414DABC0`: legacy 8b/10b channel-equalization phase
- `0x1414DCC10`: `SetHwTrainingPatternWithAssrPolicy`
- `0x1414D8EE0`: `SetDpPhyPatternWithAssrPolicy`
- `0x1414DC560`: `DpcdSetTrainingPatternAndLaneSettings`
- `0x1414DD300`: DPCD lane-settings write
- `0x1414DC080`: lane-status / adjust-request read

Static result:

- The service fresh-training path is still normal DP training in shape:
  - link settings are written as `0x100/0x101` together, then `0x107`;
  - clock recovery starts;
  - first iteration writes DPCD `0x102` training pattern plus `0x103...` lane settings as one block unless a split-write flag is set;
  - later iterations update lane settings;
  - lane status / adjust requests are read from the normal `0x202...` area;
  - channel equalization follows if CR succeeds.
- No Apple-private opcode or firmware transport is visible inside the CR/EQ loop.
- The interesting Windows-specific part is still policy propagation:
  - `ProgramAssrThenTrainLinkSettings` writes DPCD `0x10A` before training;
  - `SetDpPhyPatternWithAssrPolicy` also passes `ClassifyDpPanelModeForAssrPolicy` into the lower training-pattern backend.
- For object low byte `0x13`, that policy can become the Windows ASSR/panel policy.

Linux cross-check:

- Our kernel already has the `dp_get_panel_mode()` route quirk in `link_edp_panel_control.c`, forcing `DP_PANEL_MODE_EDP` for exact `0x3113` / DDC3 / UNIPHY_D.
- Linux training uses `dp_get_panel_mode()` in both:
  - `edp_set_panel_assr()` / `dp_set_panel_mode()` before training;
  - `dp_set_hw_test_pattern()` via `pattern_param.dp_panel_mode`.
- Therefore the most obvious Windows PHY/pattern policy hook is already wired conceptually in the current Linux tree.

Conclusion:

- Before adding more PHY guesses, the next kernel try should first fix the proven `0x310` timing gap in the lower source-DPCD path.
- If `0x310`-lower still leaves `0x202/0x203 = 0`, the next useful log target is not another random DPCD byte; it is whether `dp_set_hw_test_pattern()` actually sees `DP_PANEL_MODE_EDP` for DP-1 during CR/EQ and which TPS pattern/lane-setting write fails.

## RE-25 RESULT: DP_SET_POWER / Retry Timing

Question:

- Does Windows have a missing power/settle step that explains lane-status zero after link settings?

IDA anchors:

- `0x141BB7510`: `WriteDpSetPowerDpcd600`
- `0x141BB7090`: `PrepareLinkTrainingStateAndArmEdp128`
- `0x141BB85F0`: link-training cleanup/disable side
- `0x141B8D860`: `PerformLinkTrainingWithRetries`

Static result:

- Windows writes DPCD `0x600` through `WriteDpSetPowerDpcd600`.
- The value is:
  - `1` for on / D0-ish;
  - `2` for off / D3-ish.
- `PrepareLinkTrainingStateAndArmEdp128` calls `WriteDpSetPowerDpcd600(..., true)` before the training attempt proceeds.
- On cleanup/disable, `sub_141BB85F0` writes `0x600 = 2` unless skipped by a link flag.
- The signal-`128` eDP path also performs HPD/AUX arming callbacks before training attempts, but the iMac secondary `0x3113` is signal `32`, so that specific eDP re-arm path is not expected for it.

Linux cross-check:

- Linux already writes `DP_SET_POWER` in `dpcd_write_rx_power_ctrl()` from `dp_enable_link_phy()` before link training.
- Linux also writes D3 from `dp_disable_link_phy()`.

Conclusion:

- `0x600` is worth logging for DP-1, but it is not currently the strongest missing-action candidate.
- The evidence points more strongly at source-DPCD `0x310` being owned by the wrong layer/timing in Linux.

## Current Kernel Direction After RE-23/24/25

The next kernel patch should be narrowly scoped:

- Move/duplicate the iMac secondary `0x310 = 04 1D 03` write into the lower source-DPCD path, preferably adjacent to `dpcd_set_source_specific_data()` or immediately before `dpcd_set_link_settings()` for the exact route.
- Keep the existing exact-route `DP_PANEL_MODE_EDP` quirk, because it already maps to both ASSR and training-pattern policy in Linux.
- Add one decisive training-pattern log for DP-1:
  - route identity;
  - `dp_panel_mode`;
  - requested training pattern;
  - link rate/lane count;
  - DPCD `0x102/0x103` write status if available;
  - lane status `0x202/0x203` after the first CR wait.
- Add a read/log of DPCD `0x600` around the same lower training window, but do not change power policy yet.

## RE-26 RESULT: Source-DPCD `0x303` Descriptor Cross-Check

Question:

- Before moving the `0x310` write lower, verify whether Windows' adjacent `0x303` source descriptor differs from Linux's generic `dpcd_set_source_specific_data()` enough that we should also patch it.

Windows `0x303` construction:

- `ProgramDpPreTrainingAuxState` builds a 9-byte block at `0x303`.
- The layout is:
  - byte `0`: low byte of source/chip id;
  - byte `1`: high byte of source/chip id;
  - bytes `2..5`: zero;
  - byte `6`: DCE/display-core generation value;
  - bytes `7..8`: zero.
- This is written immediately after the `0x300` OUI block and before the `0x310` table revision block.

Linux comparison:

- Linux `struct dpcd_amd_device_id` in `dc_dp_types.h` has the same 9-byte layout:
  - `device_id_byte1`;
  - `device_id_byte2`;
  - `zero[4]`;
  - `dce_version`;
  - `dal_version_byte1`;
  - `dal_version_byte2`.
- Linux `dpcd_set_source_specific_data()` writes this struct to `DP_SOURCE_OUI + 0x03`, which is DPCD `0x303`.
- Linux's current iMac helper already chooses the Windows `0x310` first byte by DCE generation:
  - `0x05` for `DCE_VERSION_12_0+`;
  - `0x04` otherwise.
- The observed iMac logs report `dce=8`, so the expected runtime table revision is `04 1D 03`, matching OBS-14/OBS-15.

Conclusion:

- Do not add a separate special `0x303` iMac payload.
- The generic Linux `0x303` descriptor already mirrors the Windows layout closely enough.
- The targeted kernel change remains: ensure `0x310 = 04 1D 03` is written in the lower source-DPCD/training timing window, alongside the existing generic `0x300/0x303/0x340` source-DPCD publication.

## Kernel Patch Applied After RE-23/24/25/26

Patch intent:

- Follow the latest Windows RE ordering instead of the older high-level DM-only source-DPCD assumption.
- Keep the exact iMac secondary route gate:
  - raw object `0x3113`;
  - DDC hardware instance `2` / DDC3;
  - `TRANSMITTER_UNIPHY_D`;
  - `SIGNAL_TYPE_DISPLAY_PORT`.

Applied changes:

- `link_dp_capability.c` now writes the iMac secondary `DP_SOURCE_TABLE_REVISION` / DPCD `0x310` from `dpcd_set_source_specific_data()`, next to the generic Linux `0x300`/`0x303`/`0x340` AMD source-DPCD publication.
- The exact payload is still DCE-derived:
  - `04 1D 03` for the observed DCE8 iMac path;
  - `05 1D 03` only for `DCE_VERSION_12_0+`.
- `amdgpu_dm.c` no longer treats the higher DM grouped-view hook as the sole `0x310` writer. It calls the lower source-DPCD helper and verifies that `0x310` has the expected value.
- No special iMac `0x303` payload was added, because RE-26 showed Linux's generic `struct dpcd_amd_device_id` already matches the Windows descriptor layout.
- `link_dp_training.c` keeps the Windows-order `0x100/0x101` then `0x107` write path, but the evidence bits are now logged as expected observations, not hard preconditions.
- Added route-specific logs for:
  - lower source-DPCD `0x310` publication;
  - `DP_SET_POWER` / `0x600` writes and skips;
  - training-pattern DPCD `0x102/0x103` state;
  - hardware test-pattern panel mode;
  - lane status `0x202/0x203` and adjust requests after training polls.

What this should answer on the next boot:

- Does the Windows-order link-settings path now begin with `0x310 = 04 1D 03` instead of `00 00 00`?
- Is the route using `DP_PANEL_MODE_EDP` when the hardware training pattern is programmed?
- Is `DP_SET_POWER` already D0 (`0x600 = 1`) before training?
- Do lane status bytes change from zero after CR/EQ polls, or are we still failing before the sink reports any lane progress?

## OBS-16 + RE-27 RESULT: Post-Training Power-Off / Keepalive

Question:

- `obs_16` showed the secondary `0x3113` route briefly reaches trained-looking lane status, then Linux writes `DP_SET_POWER` / DPCD `0x600 = 2` and later loses AUX. Does Windows also power the trained secondary off, or does it preserve it?

OBS-16 runtime facts:

- The latest kernel did run: `7.0.1-1-imac-5k-gccd8d9b17a6e`.
- The exact secondary route is still correct: raw object `0x3113`, DDC3 / hw instance `2`, `UNIPHY_D`, signal `32`.
- Lower source-DPCD publication ran and reported success: `0x310 = 04 1D 03`.
- Readback of `0x310` still returned `00 00 00`, even after the successful lower write. Treating readable `0x310` as a hard readiness gate is therefore too strict for this sink.
- The Windows-order training path produced real lane progress:
  - early CR state: `0x202 = 0x11`, `0x203 = 0x11`, align `0x80`;
  - later EQ/locked-looking state: `0x202 = 0x77`, `0x203 = 0x77`, align `0x81`;
  - DPCD `0x600` was `0x01` during the successful-looking training window.
- Immediately after that, Linux wrote `DP_SET_POWER` off: `0x600 = 0x02` with current rate/lane still `20/4`.
- After this off write, later grouped-commit attempts repeatedly see AUX/HPD unavailable and cannot recover DPCD proof.

Windows RE-27 findings:

- IDA renames/comments added:
  - `0x141B85200` -> `SetSkipPowerOffForBranchSinkOui`;
  - `0x141BB85F0` -> `DisableDpLinkMaybeD3PowerOff`;
  - `0x141B87410` -> `ProbeTrainDpLinkThenCleanupMaybeD3`.
- `WriteDpSetPowerDpcd600` at `0x141BB7510` writes DPCD `0x600 = 1` for D0/on and `0x600 = 2` for off/D3-ish.
- `DisableDpLinkMaybeD3PowerOff` only calls `WriteDpSetPowerDpcd600(..., false)` when link-service byte `+0x2DC` is clear.
- `SetSkipPowerOffForBranchSinkOui` sets that `+0x2DC` skip-power-off byte when:
  - downstream/branch type field `+0x218` equals `1`;
  - DPCD `0x500..0x502` sink/branch OUI, stored at link-service `+0x258`, is one of `00:10:FA`, `00:80:E1`, or `00:E0:4C`.
- `ProbeTrainDpLinkThenCleanupMaybeD3` does call disable/cleanup after a probe-training loop, even on the success path, but whether this actually writes `0x600 = 2` depends on the skip-power-off byte above.
- `PerformLinkTrainingWithRetries` calls `DisableDpLinkMaybeD3PowerOff` only after a failed training attempt before retry. A successful training attempt returns before the D3 cleanup.
- The stream-enable state-1 path in `StreamEnableWrite4F1AndPollSinkStatus` does not call the D3/disable helper. It programs the requested cached tuple, programs `0x316`, calls the stream-enable vfunc, promotes state `1 -> 3`, and returns.

Conclusion:

- Windows does not appear to blindly power off a successfully trained stream-enable path.
- Windows can perform detect/probe cleanup after successful probe training, but it has a sink-derived skip-power-off mechanism specifically preventing `0x600 = 2` for selected branch/OUI sinks.
- This maps directly to `obs_16`: Linux's destructive off write happens after the exact kind of early/probe training success where Windows has a conditional skip-power-off guard.
- The next kernel change should preserve the trained secondary `0x3113` route through first two-tile handoff by skipping DPCD `0x600 = 2` for this route once lane status proves trained, at least until the stream handoff reaches the Windows-like state-1/state-3 equivalent.
- Add one log/read before deciding how broad the guard should be: dump DPCD `0x005`, `0x080`, and `0x500..0x508` on the secondary `0x3113` route while AUX is alive, so we can see whether the iMac sink matches Windows' branch/OUI skip-power-off condition.

## Kernel Patch Applied After RE-27

Patch intent:

- Avoid repeating the `obs_16` failure where Linux reached trained-looking lane status on secondary `0x3113`, then immediately wrote DPCD `0x600 = 2` and lost AUX/HPD.
- Follow the Windows behavior found in RE-27: once the secondary link has real trained proof, do not blindly power the sink down on the cleanup/disable side.
- Stop treating readable `0x310` echo as mandatory proof, because `obs_16` showed the lower write returned `DC_OK` while readback still returned zeros.

Applied changes:

- `dc.h` now carries iMac-specific trained-link preservation evidence:
  - trained proof flag;
  - skip-D3-after-trained flag;
  - DPCD `0x005`, `0x080`, `0x600`;
  - raw `0x500..0x508` branch/OUI block.
- `link_dp_training.c` now arms this preservation state only for the exact Windows secondary route after lane status proves:
  - CR done;
  - channel EQ done;
  - symbol lock;
  - interlane align.
- When arming preservation, Linux records the trained DPCD tuple:
  - `0x100` rate;
  - `0x101` lane-count/enhanced-framing;
  - `0x003` downspread-derived proof;
  - `0x202/0x203` lane status.
- `link_dp_phy.c` now skips the destructive `DP_SET_POWER` D3 write only when:
  - route is exact secondary `0x3113` / DDC3 / UNIPHY_D;
  - trained-link preservation has already been armed;
  - the write is an off/D3 request.
- The skip log prints the preserved DPCD proof and branch/OUI bytes so the next boot can confirm whether the iMac sink matches Windows' skip-power-off OUI pattern.
- `amdgpu_dm.c` now treats successful lower-owner source-DPCD write evidence as sufficient even if `0x310` readback is zero.

Expected `obs_17` checks:

- Look for `trained-link-preserve armed`.
- Look for `DP_SET_POWER skipped ... reason=trained-link-preserve` instead of a post-training `value=0x02` write.
- Confirm whether `branch500_508` begins with one of the Windows skip-power-off OUIs:
  - `00 10 FA`;
  - `00 80 E1`;
  - `00 E0 4C`.
- Confirm whether `source-DPCD verify` reports `lower_write_evidence=1 programmed=1` even when `readback_ok=0`.
- If the next failure still occurs, the key distinction is whether it happens after preserved sink power with a fresh training failure, or because the grouped handoff still rejects the preserved trained proof.

## OBS-17 RESULT: D3 Skip Worked, Source Output Disable Still Broke Handoff

Runtime source:

- `C:\proj\reverse\WT6A_INF\obs_17\dmesg.txt`.
- Kernel identified itself as `7.0.1-1-imac-5k-gcc299f41669c`, so this was the RE-27 skip-D3 patch generation.

What improved:

- The secondary Windows route is still exact:
  - raw object `0x3113`;
  - DDC3 / hardware instance `2`;
  - `TRANSMITTER_UNIPHY_D`;
  - DisplayPort signal `32`.
- Training reached real proof on the secondary route:
  - `0x202 = 0x77`;
  - `0x203 = 0x77`;
  - align `0x81`;
  - `0x100 = 0x14`;
  - `0x101 = 0x84`;
  - `0x003 = 0x01`;
  - `0x600 = 0x01`.
- The new D3 guard fired:
  - `DP_SET_POWER skipped ... requested=0x02 reason=trained-link-preserve`.
- The lower source-DPCD write evidence is now accepted correctly:
  - `lower_write_evidence=1`;
  - `treating_programmed=1`;
  - readback of `0x310` still returns `00 00 00`, so readback must remain non-authoritative for this sink.
- The pre-userspace route becomes pair-ready:
  - `route/AUX pair ready before stream handoff`;
  - primary tile and secondary tile both present;
  - route OK;
  - AUX armed;
  - `0x10A`, source-DPCD, and `0x4F1` evidence set.
- The first real Mutter/GNOME atomic commit now has the shape we wanted:
  - two real tile streams;
  - eDP primary and DP secondary;
  - secondary plane uses the right half of the shared framebuffer:
    - `src=2560,0 2560x2880`;
    - `viewport=2560x2880+2560+0`.

What still failed:

- Skipping only DPCD `0x600 = 2` was not enough.
- Linux still entered the generic `dp_disable_link_phy()` cleanup path after the successful-looking secondary training.
- That path called the hardware source-output disable and then cleared `cur_link_settings`.
- At the first two-tile stream handoff, the secondary had lost the in-memory tuple:
  - `rate=0`;
  - `lanes=0`;
  - `link_state_valid=0`.
- Live DPCD lane-status reads at handoff returned:
  - `0x100 = 0x14`;
  - `0x101 = 0x84`;
  - `0x003 = 0x01`;
  - but `0x202 = 0x00`;
  - `0x203 = 0x00`.
- The recovery helper then erased the earlier trained proof and rejected the handoff:
  - `reason=link-state-invalid`;
  - later attempts degrade to AUX read/write failures and `dpcd-trained-proof-missing`.

Conclusion:

- OBS-17 proves the remaining failure is lower than Mutter/GNOME.
- Userspace supplied a valid two-tile/right-half commit.
- The kernel problem is preserving and consuming the already-trained secondary `0x3113` state through the first stream-enable handoff.
- Windows RE-27 maps to this: the state-1 path is an already-link-configured path. It does not require destructive retraining or a source-output reset before stream enable.

## Kernel Patch Applied After OBS-17

Patch intent:

- Preserve the trained secondary source-output path, not just sink DPCD power, until the first cached/two-tile handoff consumes it.
- Treat the stored trained proof from `0x202/0x203 = 0x77/0x77` as authoritative if live lane-status reads have already gone quiet but the requested tuple still matches.
- Allow the Windows-like state-1 handoff to proceed from `imac5k_trained_link_preserved` even when generic Linux `link_state_valid` has not yet been re-promoted.

Applied changes:

- `link_dp_phy.c` now skips the hardware `disable_link_output()` and the `cur_link_settings` clear for the exact secondary route when:
  - trained-link preservation is armed;
  - skip-D3-after-trained is armed;
  - cached handoff has not yet been consumed.
- This path logs:
  - `source-output disable skipped`;
  - current rate/lane tuple;
  - stored proof rate/lane tuple;
  - stored lane-status proof;
  - link active/state-valid flags;
  - stream-enable state;
  - handoff allowed/consumed flags.
- `amdgpu_dm.c` no longer erases stored trained proof merely because a later live DPCD lane-status read returns zero after source quiesce.
- The Windows-state1 DPCD recovery log now prints:
  - `live_trained`;
  - `cached_proof_valid`;
  - `used_cached_proof`;
  - stored proof bytes separately from live raw DPCD bytes;
  - preserved-link flag.
- `amdgpu_dm.c` and `link_dpms.c` now allow the handoff blocker to pass the generic `link_state_valid` check if `imac5k_trained_link_preserved` is set.
- The stricter tuple checks remain:
  - current tuple must be known;
  - cached/proof tuple must be known;
  - requested tuple must be known;
  - current, cached/proof, and requested rate/lane tuples must match.

Expected `obs_18` checks:

- Confirm the new log appears before first stream handoff:
  - `IMAC5K: secondary 0x3113 source-output disable skipped ... reason=trained-link-preserve`.
- At `commit_streams_pretrain`, the secondary route should no longer show:
  - `rate=0 lanes=0`.
- `Windows-state1 DPCD recovery` should show either:
  - `live_trained=1`; or
  - `cached_proof_valid=1 used_cached_proof=1`.
- The handoff should arm:
  - `cached-link-handoff armed on DP-1`.
- The lower link path should accept it:
  - `cached-link-handoff accepted on link 1`.
- If it still fails, the next important distinction is whether:
  - source output was still disabled by another path;
  - the tuple mismatch blockers reject the handoff;
  - or handoff is accepted and a later stream-enable / plane / timing step fails.

## RE-28: OBS-17 Follow-Up, Windows State-1 Live-Proof And Cached Stream Enable

Question:

- Before another Linux test, can static Windows RE tell us whether our post-OBS-17 Linux patch is still missing a lower stream-enable step or relying too much on cached proof?

New/confirmed IDA anchors:

- `0x1414E7680`: `SetState1IfDpcdReportsTrainedLink`.
- `0x1414E1670`: `RecoverActiveDpLinkSettingsFromDpcd`.
- `0x141443B50`: `LinkServiceHasCachedLinkSettings`.
- `0x1414431B0`: renamed/commented as `ProgramCachedLinkStreamSourceThenEnableStream`.
- `0x1414D2710`: renamed as `ProgramStreamSourceWithLinkSettings`.
- `0x141442470`: renamed as `ProgramPeerSourceUpdateBeforeStreamEnable`.
- `0x1414E7810`: `StreamEnableWrite4F1AndPollSinkStatus`.

Findings:

- Windows has an explicit state-1 constructor:
  - `SetState1IfDpcdReportsTrainedLink` resets a local requested tuple to rate/lane defaults;
  - calls `RecoverActiveDpLinkSettingsFromDpcd`;
  - only sets state `1` if the recovered tuple has nonzero lane count and link rate.
- `RecoverActiveDpLinkSettingsFromDpcd` reads:
  - DPCD `0x100/0x101`;
  - DPCD `0x003`;
  - DPCD `0x202/0x203`.
- It returns a zero tuple unless lane status proves a trained/locked link:
  - CR done;
  - channel EQ done;
  - symbol lock;
  - required lane bits for the active lane count.
- Therefore Windows' explicit state-1 path does not treat a stale cached tuple alone as trained proof if live DPCD lane status is zero.
- `LinkServiceHasCachedLinkSettings` is much weaker:
  - it only checks whether cached tuple dword `+0x8C` is nonzero;
  - there is no HPD-high or generic link-active test in that predicate.
- The broader enable path uses that cached tuple to skip the fresh-training vfunc:
  - if cached settings exist, `EnableValidatedStreamMaybeAssert4F1` does not call the fresh link-training vfunc `+0x50`;
  - it continues with source/stream setup.
- The cached stream path is not a no-op:
  - `ProgramCachedLinkStreamSourceThenEnableStream` first calls `ProgramPeerSourceUpdateBeforeStreamEnable`;
  - calls `ProgramStreamSourceWithLinkSettings` using the cached tuple at link-service `+0x8C`;
  - marks the display object enabled through display-object vfunc `+0x38`;
  - calls the real stream-enable vfunc at link-service vtable `+0x98`.
- The `StreamEnableWrite4F1AndPollSinkStatus` state-1 branch remains:
  - refresh/cache requested link tuple;
  - RMW DPCD `0x316`;
  - call `ProgramStreamSourceWithLinkSettings`;
  - call real stream-enable vfunc `+0x98`;
  - promote state `1 -> 3`;
  - return without fresh training or D3 cleanup.

Kernel interpretation:

- The post-OBS-17 source-output preservation patch is still the right next test because Windows expects the already-trained link to remain usable long enough for stream-enable.
- However, if Linux's next boot only succeeds through `used_cached_proof=1` while `live_trained=0`, that is a Linux workaround, not a precise Windows state-1 match.
- The most Windows-like success signature is:
  - source output disable skipped;
  - `cur_link_settings` still `20x4`;
  - live DPCD `0x202/0x203` still trained at handoff;
  - `cached-link-handoff armed`;
  - `cached-link-handoff accepted`.
- `0x316` is confirmed as part of the Windows stream-enable sequence, but it remains conditional:
  - bit 0 is set only when stream object field `+0x88` is `10`, `11`, or `12`;
  - current runtime observations show `0x316 = 0`;
  - do not force-write `0x316` yet unless we map the exact iMac stream mode or runtime evidence shows Windows would set it.

What this avoids:

- Do not loosen the Linux model into "cached proof is always enough" and call that Windows-equivalent.
- Do not add a broad `0x316` write yet.
- Do not return success from cached handoff without ensuring the normal Linux stream/encoder/attribute programming still occurs after link-training is skipped.

Best next RE target if obs_18 still fails:

- Identify the concrete implementation behind the Windows real stream-enable vfunc `+0x98` for this DisplayPort link-service object and map it to Linux DC concepts:
  - stream encoder setup;
  - stream attribute / MSA programming;
  - OTG/encoder mux;
  - any source-side enable bit that is separate from link training.

## RE-29: Lower Windows Stream-Enable Vfunc

Question:

- Does Windows' state-1 cached stream-enable do more than "skip training and return success"?
- If yes, what is the lower operation after `ProgramStreamSourceWithLinkSettings`?

New IDA anchors:

- `0x1414E8140`: `ConstructDpStreamObjectWithDpcdInterface`.
- `0x1414E75A0`: renamed as `StreamEnableMaybeDpcd170ThenSourceEnable`.
- `0x1414D88E0`: renamed as `ProgramDpcd170StreamEnableFlags`.
- `0x1414D89D0`: renamed as `ComputeDpcd170TimingFlag`.

Findings:

- `ConstructDpStreamObjectWithDpcdInterface` installs a DP stream subobject vtable at stream object `+0x28`:
  - `stream+0x28` vtable is `off_141FA2F60`.
- In that DP stream subobject vtable:
  - offset `+0x80` is `StreamEnableWrite4F1AndPollSinkStatus`;
  - offset `+0x88` is `DisableStreamIfStateActive`;
  - offset `+0x90` is `SetState1IfDpcdReportsTrainedLink`;
  - offset `+0x98` is `StreamEnableMaybeDpcd170ThenSourceEnable`.
- The state-1 branch calls this `+0x98` vfunc after:
  - caching requested settings;
  - optional `0x316` RMW;
  - `ProgramStreamSourceWithLinkSettings`.
- `StreamEnableMaybeDpcd170ThenSourceEnable` is not just a success return:
  - if per-stream flag `+0x36C` is clear, it delegates to another vfunc path;
  - if `+0x36C` is set, it computes a timing-derived flag;
  - writes one byte to DPCD `0x170`;
  - then calls a lower source-enable vfunc at object `+0x18` vtable offset `+0x3B0`, passing the stream and source-side state.
- `ProgramDpcd170StreamEnableFlags` builds the DPCD `0x170` byte:
  - bit `2` is always set;
  - bit `3` reflects the timing-derived flag from `ComputeDpcd170TimingFlag`;
  - bit `5` is set when stream-side byte/field `+0x394` equals `2`.
- `ComputeDpcd170TimingFlag` computes this from the selected stream timing fields and stores the result in the DP stream object side state.

Kernel interpretation:

- Windows state-1 cached handoff is a real stream-enable sequence:
  - preserve already-trained link;
  - program/refresh stream-source state;
  - optionally RMW source-specific DPCD such as `0x316`;
  - optionally write DPCD `0x170`;
  - call lower source-enable.
- Our Linux cached-handoff must therefore not bypass normal stream encoder / stream attribute / OTG source programming. It may skip fresh DP link training, but the stream still needs the normal source-side programming.
- The current Linux cached-handoff still returns through the normal DC commit machinery after `enable_link_dp()`, so this may already be covered by Linux's normal stream/pipe programming. `obs_18` should tell us:
  - if handoff is accepted and stream programming continues, the visual state may improve;
  - if handoff is accepted but the secondary remains dark or wrong, the next patch should instrument Linux's stream encoder / stream attribute path rather than the AUX/link-training path.
- Do not add a blind `0x170` write yet:
  - we have not proven the `+0x36C` flag is set on the exact iMac route;
  - Linux does not currently log `0x170`;
  - if obs_18 accepts cached handoff but fails later, logging DPCD `0x170` around handoff becomes a good next targeted kernel diagnostic.

Updated expected `obs_18` distinction:

- Failure before `cached-link-handoff accepted` is still a link-preservation / trained-proof issue.
- Failure after `cached-link-handoff accepted` moves the investigation from AUX/link training to source-side stream enable:
  - stream encoder setup;
  - stream attributes/MSA;
  - OTG/encoder mux;
  - possible source-specific DPCD `0x170`/`0x316`.

## RE-30: Stream-Enable Converter Branch Versus DPCD 0x107 Fallback

Question:

- Is the lower stream-enable vfunc found in RE-29 always part of the iMac 5K state-1 path?
- Or is it only used for a narrower DP converter-capability path?

New IDA anchors:

- `0x141442130`: renamed as `ConstructHWSequenceServiceForAsic`.
- `0x14155BFA0`: renamed as `HwSeqProgramStreamSourceWithLinkSettings`.
- `0x14155B460`: renamed as `HwSeqProgramStreamTimingAndSourceCommon`.
- `0x1415563E0`: renamed as `HwSeqEnableConverterStreamSourcePacket`.
- `0x141555F70`: renamed as `HwSeqProgramTimingGeneratorForStream`.
- `0x1414E8000`: renamed as `DpStreamHasConverterCapabilityData`.
- `0x1414E22D0`: renamed as `DpStreamSetMsaTimingIgnoreOnDpcd107`.
- `0x1414D8AC0`: renamed as `RefreshDpConverterCapabilityData`.
- `0x1414F5E70`: renamed as `ParseDpConverterCapabilityData`.
- `0x1414E7250`: renamed as `RefreshDpStreamCapsAndConverterData`.

Findings:

- The HW sequence service selected by `ConstructHWSequenceServiceForAsic` returns an interface pointer at object `+0x28`.
- For all observed HW sequence service variants, vtable offset `+0x3B0` resolves to the same function, now named `HwSeqEnableConverterStreamSourcePacket`.
- That `+0x3B0` function:
  - reads a lower object from the stream/context at `+0x178`;
  - checks a lower readiness method;
  - builds a 40-byte source-enable packet through `sub_141561A50`;
  - sends that packet to a lower hardware sequencer vfunc at offset `+0x30`.
- However, `StreamEnableMaybeDpcd170ThenSourceEnable` does not always call this lower packet path.
- Its first branch is `DpStreamHasConverterCapabilityData`, implemented as a simple check of stream-side byte `+0x36C`.
- If `+0x36C` is set:
  - Windows computes the `0x170` timing flag;
  - writes DPCD `0x170`;
  - calls HWSequenceService `+0x3B0` (`HwSeqEnableConverterStreamSourcePacket`).
- If `+0x36C` is clear:
  - Windows calls `DpStreamSetMsaTimingIgnoreOnDpcd107`;
  - this reads DPCD `0x107`;
  - sets bit `7` if stream fields `+0x5C` or `+0x60` are nonzero;
  - clears bit `7` otherwise;
  - writes DPCD `0x107` back only if the byte changed.

Converter-capability provenance:

- `RefreshDpStreamCapsAndConverterData` refreshes the DP stream capability side state before stream enable.
- It calls `RefreshDpConverterCapabilityData`, which reads DP/core stream capability information and calls `ParseDpConverterCapabilityData`.
- `ParseDpConverterCapabilityData` only enables the converter branch if the input capability byte has bit `0` set and its mode field equals `2`.
- The secondary iMac Linux captures so far show:
  - `DPCD 0x005 = 0x00`;
  - `DPCD 0x080 = 0x00`;
  - no Linux logging for `DPCD 0x170` yet.
- That evidence makes the non-converter fallback path more likely for the exact iMac `0x3113` route, though runtime logging of `0x107`/`0x170` around accepted handoff would still be useful.

Kernel interpretation:

- `0x170` should no longer be treated as a generic missing iMac 5K write.
- The more likely missing stream-enable detail, if Linux reaches cached handoff but still fails visually, is the fallback `0x107` bit `7` update.
- Current Linux logs show `0x107 = 0x10`, which is only the normal downspread bit pattern we already handle.
- Windows may additionally set `0x107 |= 0x80` when stream fields `+0x5C/+0x60` are nonzero.
- We still need to map `+0x5C/+0x60` precisely, but they are in the stream timing/placement region and may correspond to nonzero stream placement or related tiled stream fields.
- This is a better next kernel diagnostic than blindly writing `0x170`:
  - log DPCD `0x107` before and after cached handoff;
  - log whether the Linux secondary stream has any equivalent nonzero placement/tile offset fields;
  - only then decide whether to set bit `7` for the secondary `0x3113` route.

Important correction to RE-29:

- The lower `+0x3B0` source-enable packet is real, but it is converter-branch-specific.
- For the likely iMac path, normal stream-source programming still happens through `HwSeqProgramStreamSourceWithLinkSettings` / `HwSeqProgramStreamTimingAndSourceCommon`, while the final `+0x98` branch may only adjust `DPCD 0x107` bit `7`.

## RE-31: DPCD 0x107 Bit 7 Maps To Linux ignore_msa_timing_param

Question:

- What does the Windows fallback `0x107` bit `7` write actually mean?
- Is there already a Linux DC concept for it?

Findings:

- DPCD `0x107` is standard `DP_DOWNSPREAD_CTRL`.
- Bit `7` is standard `DP_MSA_TIMING_PAR_IGNORE_EN`.
- Linux names the same bit in `include/drm/display/drm_dp.h` and models it in AMD DC as `stream->ignore_msa_timing_param`.
- Linux writes the bit in `enable_stream_features()` in `dc/link/link_dpms.c`:
  - read DPCD `0x107`;
  - set `IGNORE_MSA_TIMING_PARAM` from `stream->ignore_msa_timing_param`;
  - write DPCD `0x107` back only if changed.
- Windows has at least two paths that perform the equivalent update:
  - `DpStreamSetMsaTimingIgnoreOnDpcd107` at `0x1414E22D0`;
  - `LinkServiceSyncMsaTimingIgnoreOnDpcd107` at `0x141446450`.
- Both Windows paths use the same condition:
  - set `0x107 |= 0x80` when stream fields `+0x5C` or `+0x60` are nonzero;
  - clear `0x107 & ~0x80` otherwise.

Kernel interpretation:

- This is not a private Apple opcode and not another firmware mailbox command.
- If the next Linux change needs to emulate this Windows branch, the right Linux representation is not a raw magic write first; it is to make the iMac secondary stream set `stream->ignore_msa_timing_param = true` if the Windows-equivalent condition is met.
- The current Linux path only sets `ignore_msa_timing_param` for FreeSync/VRR in `amdgpu_dm.c`.
- Current iMac logs showing `0x107 = 0x10` mean Linux is only setting the normal spread bit and is not setting MSA timing ignore.
- We still need one more narrow RE mapping for Windows stream fields `+0x5C/+0x60` before deciding whether the exact iMac 5K secondary should force `ignore_msa_timing_param`.
- If those fields map to always-nonzero active timing fields for this route, then `ignore_msa_timing_param` becomes a strong candidate for the next kernel change.
- If those fields map to VRR/DRR-only state, then this is probably not the iMac 5K gap and should stay untouched.

## RE-32: Windows Stream Fields Behind DPCD 0x107 MSA-Ignore

Question:

- What exactly are the Windows stream fields `+0x5C` and `+0x60` that gate `DPCD 0x107` bit `7`?
- Are they VRR/DRR-only state, or normal per-display stream identity/state?

Findings:

- `DpStreamSetMsaTimingIgnoreOnDpcd107` at `0x1414E22D0` receives the stream-state object as its second argument.
- The non-converter stream finalizer path is:
  - `ProgramCachedLinkStreamSourceThenEnableStream` at `0x1414431B0`;
  - `ProgramPeerSourceUpdateBeforeStreamEnable`;
  - `ProgramStreamSourceWithLinkSettings`;
  - sink/source enable vfunc on `stream_state+0x178`;
  - DP stream finalizer vfunc `+0x98`, which is `StreamEnableMaybeDpcd170ThenSourceEnable`.
- For non-converter streams, `StreamEnableMaybeDpcd170ThenSourceEnable` calls `DpStreamSetMsaTimingIgnoreOnDpcd107`.
- `ApplyPathModeTimingToViewContextStream` at `0x141ADAC70` maps the path-mode record into the stream state:
  - path record `+0x34` / dword `[13]` -> stream state `+0x5C`;
  - path record `+0x38` / dword `[14]` -> stream state `+0x60`.
- The path-mode record builder `sub_141424DB0` fills path record dword `[13]` with `PackDisplayObjectIdCoreFields(display_object_id)`.
- `PackDisplayObjectIdCoreFields` at `0x14143BFC0` packs the display object id core fields:
  - low byte;
  - bits `8..11`;
  - bits `12..15`.
- Therefore stream state `+0x5C` is not VRR-only metadata. It is the packed display object id copied into the stream state.
- Path record dword `[14]` is the first dword from the display/path metadata block returned by a display manager vfunc at object offset `+0x3C8`; this is an additional gate input, but `+0x5C` alone should already be nonzero for a valid DP display stream.
- This makes the Windows condition in `DpStreamSetMsaTimingIgnoreOnDpcd107` much stronger than previously assumed:
  - for a normal constructed display stream, `stream_state+0x5C != 0`;
  - on the non-converter path, Windows should set `DPCD 0x107 |= 0x80`;
  - that is standard `DP_MSA_TIMING_PAR_IGNORE_EN`.

Kernel interpretation:

- The previous suspicion that this might be VRR/DRR-only was likely wrong.
- For the iMac 5K route, if the exact secondary stream is on the non-converter path, Windows likely ends stream enable with `DPCD 0x107 = 0x90` rather than Linux's observed `0x10`.
- The Linux-side representation is still `stream->ignore_msa_timing_param`, not a private opcode and not a raw Apple mailbox write.
- A targeted kernel change should prefer setting `ignore_msa_timing_param` for the exact iMac 5K tiled streams, or at least for the exact secondary `0x3113` route first, then verify that `enable_stream_features()` logs/writeback produce `0x107 bit7`.
- This should be logged explicitly in the next run:
  - stream role primary/secondary;
  - whether `ignore_msa_timing_param` is set;
  - DPCD `0x107` before and after `enable_stream_features`;
  - whether the secondary handoff path is converter or non-converter if Linux can infer it.

Remaining narrow RE uncertainty:

- `DpStreamSetMsaTimingIgnoreOnDpcd107` has one early guard through a hardware-sequence/service vfunc at offset `+0x3D8`.
- We have not yet proven that guard returns false for the exact iMac `0x3113` route.
- However, if it returns false, the Windows condition for setting bit `7` is now clear and appears to be normal display-stream identity, not dynamic-refresh metadata.

## RE-33: HWSequence `+0x3D8` Does Not Block The DPCD 0x107 Path

Question:

- Does the early HWSequenceService vfunc call in `DpStreamSetMsaTimingIgnoreOnDpcd107` skip the iMac route, or does it just perform a required programming step before the DPCD `0x107` update?

Findings:

- `ConstructDal2CoreAndServices` creates `HWSequenceService` through `ConstructHWSequenceServiceForAsic` at `0x141442130`.
- `ConstructHWSequenceServiceForAsic` chooses ASIC-specific HW sequence implementations for ASIC families `11`, `12`, and `13`.
- The topology manager passes this `HWSequenceService` into stream construction:
  - `ConstructDisplayTopologyManager` stores DAL init `+0x18` as topology manager `+0x58`;
  - `BuildStreamForDisplayEntryPath` copies topology manager `+0x58` into stream init data `+0x28`;
  - `ConstructBaseStreamObjectFromInit` stores stream init `+0x28` into stream object `+0x40`;
  - the DP stream subobject sees that as `a1+0x18`.
- `DpStreamSetMsaTimingIgnoreOnDpcd107` calls `HWSequenceService+0x3D8` before reading DPCD `0x107`.
- All inspected HWSequenceService vtables route `+0x3D8` to the same function:
  - `off_141FA0588 + 0x3D8 -> 0x141555F70`;
  - `off_141FA0B70 + 0x3D8 -> 0x141555F70`;
  - `off_141FA1010 + 0x3D8 -> 0x141555F70`;
  - `off_141FA15F8 + 0x3D8 -> 0x141555F70`.
- `0x141555F70` is `HwSeqProgramTimingGeneratorForStream`.
- `HwSeqProgramTimingGeneratorForStream`:
  - builds an 80-byte timing-generator payload from `stream_state+0x2C`;
  - obtains a lower object from `stream_state+0x178`;
  - calls lower vfunc `+0x150` with the generated timing data;
  - returns `0`.
- Because it returns `0`, it does not block `DpStreamSetMsaTimingIgnoreOnDpcd107`; it is a pre-step before the DPCD `0x107` read/modify/write.

Corrected Windows non-converter finalizer shape:

1. `StreamEnableMaybeDpcd170ThenSourceEnable` tests converter capability byte `+0x36C`.
2. If converter capability is absent, it calls `DpStreamSetMsaTimingIgnoreOnDpcd107`.
3. `DpStreamSetMsaTimingIgnoreOnDpcd107` first calls `HwSeqProgramTimingGeneratorForStream`.
4. `HwSeqProgramTimingGeneratorForStream` programs timing-generator state and returns `0`.
5. Windows reads DPCD `0x107`.
6. Windows sets bit `7` when stream state `+0x5C` or `+0x60` is nonzero.
7. Since `+0x5C` is the packed display object id for a constructed stream, the bit should be set for a normal DP display stream.
8. Windows writes DPCD `0x107` back if changed.

Kernel interpretation:

- The `0x107` bit `7` candidate is now stronger.
- It is not behind a likely iMac-specific skip gate in the inspected HWSequenceService variants.
- The next Linux change should treat this as a normal stream-enable detail for the iMac 5K tiled route:
  - set `stream->ignore_msa_timing_param = true` for the exact quirked tiled streams, or at minimum the exact secondary `0x3113` stream first;
  - log DPCD `0x107` before and after `enable_stream_features()`;
  - verify the resulting value changes from the observed `0x10` to `0x90`.
- If the visual failure persists after that, the next RE target should be the lower timing-generator payload built by `HwSeqProgramTimingGeneratorForStream`, not another mailbox/opcode path.

## RE-34: Lower Timing-Generator Payload Before The 0x107 Write

Question:

- What does Windows program immediately before the non-converter DPCD `0x107` MSA-ignore write?
- If setting `ignore_msa_timing_param` is not enough, what is the next concrete Windows sequence to compare?

Findings:

- `HwSeqProgramTimingGeneratorForStream` at `0x141555F70` builds an 80-byte timing-generator payload on the stack.
- It calls `BuildHwTimingGeneratorPayloadFromStreamTiming` at `0x141562BA0` with:
  - HW sequence object;
  - `stream_state+0x2C`;
  - output payload buffer.
- `BuildHwTimingGeneratorPayloadFromStreamTiming` is a pure field-copy/repack helper:
  - output `+0x00..+0x2C` copies the first 12 timing dwords from `stream_state+0x2C`;
  - output `+0x30` copies input `+0x44`;
  - output `+0x34` copies input `+0x30`;
  - output `+0x38` copies input `+0x34`;
  - output `+0x3C/+0x3D` copy input bytes `+0x38/+0x39`;
  - output `+0x40` copies input `+0x3C`;
  - output `+0x44` repacks many flags from input `+0x40`;
  - output `+0x48` repacks flags from input `+0x4C`;
  - output `+0x4C` is derived through `sub_14155FE40` from a small mode field inside input `+0x4C`.
- After building the payload, `HwSeqProgramTimingGeneratorForStream`:
  - obtains a lower source/stream object from `stream_state+0x178`;
  - calls a vfunc at `+0xB8` on that object;
  - calls the returned object's vfunc `+0x150` with `payload+0x34`;
  - returns `0`.

Sequence relation:

- The non-converter Windows finalizer order is now:
  - program source with cached link settings;
  - enable/arm the stream source object;
  - program lower timing-generator payload;
  - read/modify/write DPCD `0x107` bit `7`.
- This is all normal DP/DC-style stream programming. It is not the private `1265` opcode path.

Kernel interpretation:

- The immediate next Linux candidate remains `stream->ignore_msa_timing_param = true` for the quirked iMac 5K tiled route, because that maps directly to the confirmed final DPCD `0x107` write.
- If that fails, the next static target should be comparing Linux's equivalent timing-generator / MSA programming against the Windows payload built at `0x141562BA0`.
- The useful Linux diagnostics for that future comparison would be:
  - per-stream timing totals/sync/blanking/pixel clock for both tiles;
  - per-stream color depth / pixel encoding / timing flags;
  - whether the secondary tile reaches the equivalent HW timing-generator programming path before `enable_stream_features()`.

## RE-35: Converter-Capability Provenance For The 0x170 Branch

Question:

- Is the Windows stream-enable `0x170` branch applicable to the exact iMac secondary DP route?
- Or does the iMac route fail the converter-capability gate and therefore use the non-converter `0x107` finalizer path?

Findings:

- `DpStreamHasConverterCapabilityData` at `0x1414E8000` is a simple byte check:
  - it returns true only when stream byte `+0x36C` is nonzero.
- `RefreshDpStreamCapsAndConverterData` calls `RefreshDpConverterCapabilityData` at `0x1414D8AC0`.
- The caller immediately before this is now named `RefreshDpCoreDpcdAndDownstreamPortCaps` at `0x1414E0AB0`.
- `RefreshDpCoreDpcdAndDownstreamPortCaps` reads the core DPCD capability block and caches:
  - core DPCD byte `0x005` into stream `+0x34C`.
- `RefreshDpConverterCapabilityData` then reads:
  - DPCD `0x080..0x083` into stream `+0x34D..+0x350`.
- `RefreshDpConverterCapabilityData` calls `ParseDpConverterCapabilityData` with:
  - parser output at stream `+0x358`;
  - first input byte at stream `+0x34C`, which is core DPCD `0x005`;
  - second input byte at stream `+0x34D`, which is DPCD `0x080`.
- `ParseDpConverterCapabilityData` at `0x1414F5E70` rejects the converter path unless:
  - DPCD `0x005` bit `0` is set;
  - and `((DPCD[0x005] >> 1) & 3) == 2`.
- If that gate fails, the parser returns before populating the converter capability data used by the stream `+0x36C` check.
- The `+0x36C` byte is not a magic standalone iMac flag:
  - it is inside the parsed converter-capability structure at output base `stream+0x358`;
  - the parser writes a dword at output `+0x14`, which lands at stream `+0x36C`;
  - the stream-enable vfunc later tests the low byte of that parsed field as the "converter capability present" gate.

IDA updates:

- Renamed `sub_1414E0AB0` to `RefreshDpCoreDpcdAndDownstreamPortCaps`.
- Added comments at:
  - `0x1414E0BCE`: core DPCD read.
  - `0x1414E0E06`: stream `+0x34C` comes from DPCD `0x005`.
  - `0x1414D8B26`: DPCD `0x080..0x083` read into stream `+0x34D..`.
  - `0x1414D8D79`: converter parser input mapping.
  - `0x1414F5ED0`: converter parser reject gate.

Kernel interpretation:

- Our iMac captures have shown:
  - `DPCD 0x005 = 0x00`;
  - `DPCD 0x080 = 0x00`.
- That means the exact iMac secondary route should fail the Windows converter-capability gate.
- Therefore Windows should not use the `0x170` converter branch on this route.
- The Windows-like route for the iMac remains the non-converter finalizer:
  - program source/timing;
  - read DPCD `0x107`;
  - set/clear bit `7` according to stream state;
  - write DPCD `0x107` back if changed.
- This makes blind Linux writes to DPCD `0x170` the wrong next move unless a future runtime log proves a nonzero compatible `0x005`/`0x080` tuple.
- The next kernel change should stay focused on matching the confirmed non-converter behavior: ensure the exact iMac tiled streams reach `enable_stream_features()` with `ignore_msa_timing_param` set so Linux writes `DPCD 0x107 = 0x90` instead of the observed `0x10`.
- Quick local kernel audit after this RE pass:
  - `enable_stream_features()` already writes DPCD `0x107` bit `7` from `stream->ignore_msa_timing_param`;
  - the current tree only sets `ignore_msa_timing_param` in the normal FreeSync/VRR config path;
  - there is not yet an iMac 5K-specific assignment that forces it for the quirked tiled streams.

Kernel change after RE-35:

- Added an iMac 5K DM hook before `dc_commit_streams()`:
  - runs at `commit_streams_pretrain`;
  - requires the machine quirk;
  - requires both real Windows-role tile streams, primary `0x3114` route and secondary `0x3113` route;
  - sets `stream->ignore_msa_timing_param = true` on both paired 2560x2880 tile streams.
- Added stream-state logging so the iMac stream dump now prints `ignore_msa`.
- Added lower `enable_stream_features()` logging for 2560x2880 tile streams:
  - logs DPCD `0x107` read status;
  - logs old byte and intended new byte;
  - logs write status when the byte changes;
  - expected successful secondary result is old `0x10`, new `0x90`, changed `1`, write status `1` (`DC_OK`).
- This keeps the implementation in the normal Linux DP path and deliberately does not add a raw `0x170` write.

## OBS-18: MSA-Ignore Reaches The Streams, But Secondary AUX Dies Before Lower 0x107

Input:

- Runtime log folder: `C:\proj\reverse\WT6A_INF\obs_18`.
- Kernel: `7.0.1-1-imac-5k-g2f70220768ff`, build `#20`, dated `Tue, 12 May 2026 15:02:31 +0000`.

Confirmed progress:

- The RE-35 kernel hook did run.
- The first userspace two-tile handoff was structurally correct:
  - eDP-1 tile blob: `1:1:2:1:0:0:2560:2880`;
  - DP-1 tile blob: `1:1:2:1:1:0:2560:2880`;
  - both CRTCs active at `2560x2880`;
  - one shared `5120x2880` framebuffer;
  - primary plane source `0,0 2560x2880`;
  - secondary plane source `2560,0 2560x2880`.
- `commit_streams_pretrain` found the exact secondary route and recovered live trained-link proof:
  - `0x10A = 0x01`;
  - `0x4F1 = 0x01`;
  - `0x100 = 0x14`;
  - `0x101 = 0x84`;
  - `0x003 = 0x01`;
  - `0x202/0x203 = 0x77/0x77`;
  - `active_link_trained = 1`;
  - evidence became `0x3f`.
- The state machine correctly promoted the secondary to:
  - `1:already-link-configured-pending-stream-enable`.
- The MSA-ignore hook set:
  - primary `old_ignore_msa=0 -> new_ignore_msa=1`;
  - secondary `old_ignore_msa=0 -> new_ignore_msa=1`.
- The primary lower `enable_stream_features()` path successfully wrote DPCD `0x107`:
  - old `0x10`;
  - new `0x90`;
  - write status `1`.

Failure:

- The secondary lower `enable_stream_features()` path did not reach a live AUX channel:
  - read status `-1`;
  - write status `-1`;
  - intended new `0x80` from old zeroed read data.
- Before that, the cached handoff was rejected:
  - `cached-link-handoff rejected on link 1 reason=no-current-settings`;
  - the rejection occurred even though the preserved DPCD proof and requested settings were valid.
- After rejection, Linux fell back into repeated fresh training / source-DPCD writes on the secondary, but AUX was already gone:
  - repeated `DP_SET_POWER on=1` status `-1`;
  - repeated Windows-order `0x100/0x101` writes failed with status `-1`;
  - repeated training DPCD writes failed with status `-1`.

Interpretation:

- The compositor is not the first failure in OBS-18.
- Mutter/GNOME initially submitted the correct two-tile, shared-framebuffer shape.
- The lower DC link-enable path still tried to fresh-train the secondary after a valid state-1 proof, because the handoff gate required current link settings that were no longer trustworthy at the exact lower-layer decision point.
- By the time the generic lower stream-feature code tried to write DPCD `0x107` on DP-1, the secondary AUX path was already HPD-low / inaccessible.
- Later Mutter commits switched to per-tile `2560x2880` buffers only after the secondary link-enable had failed. That likely explains the visible half/stretched behavior; it is a consequence, not the root cause.

Next kernel target:

- Do not wait for lower `enable_stream_features()` to be the first secondary `0x107` writer.
- While `commit_streams_pretrain` still has live secondary AUX and trained-link proof, proactively RMW DPCD `0x107` bit 7 on the exact `0x3113` tile route.
- Keep the normal stream flag too, so lower `enable_stream_features()` remains consistent if AUX is still live later.
- Relax the lower cached-handoff gate for this exact iMac state-1 path:
  - require armed handoff;
  - require state `1`;
  - require DPCD trained proof;
  - require cached/requested/DPCD rate-lane match;
  - do not reject solely because `cur_link_settings` is unknown at the lower decision point.
- On acceptance, seed `cur_link_settings` from the requested or cached settings before stream encoder setup.
- This better matches the Windows state-1 interpretation: the link is already configured from proof, and stream-enable should consume that state instead of triggering fresh training.

Kernel change after OBS-18:

- Added a pretrain-time secondary MSA-ignore RMW:
  - runs from the existing paired-stream MSA-ignore hook;
  - only targets the exact secondary `0x3113` Windows-role tile stream;
  - reads DPCD `0x107`;
  - sets `DP_MSA_TIMING_PAR_IGNORE_EN` / bit 7;
  - verifies and logs old/new/verify bytes while AUX is still expected to be live.
- Relaxed the lower cached state-1 handoff gate:
  - still requires armed handoff, stream state `1`, valid DPCD trained proof, and cached/requested/DPCD rate-lane agreement;
  - no longer rejects solely because lower `cur_link_settings` is unknown at that moment;
  - treats the trained DPCD proof as the current state for this exact path.
- On cached-handoff acceptance, seeds `cur_link_settings` and `link_state_valid` before stream encoder setup.
- Expected next successful log shape:
  - `prewriting secondary non-converter MSA-ignore DPCD 0x107 ... old=0x10 new=0x90 verify=0x90`;
  - `cached-link-handoff accepted on link 1`;
  - no repeated secondary `Windows-order write 0x100/0x101 failed` loop during the first two-tile commit.

## Instrumentation Added Before OBS-19: PHY Enable/Disable Tracing

To find exactly when/where the secondary `0x3113` AUX dies between `commit_streams_pretrain` and the lower stream-enable, two diagnostic helpers were added in `dc/link/protocols/link_dp_phy.c` (exported via `link_dp_phy.h`):

- `dp_imac5k_log_phy_event()` logs one `IMAC5K: phy_trace func=... checkpoint=... role=primary|secondary ...` line with the link's own rate/lanes/HPD/symclk/preserve state. No-op on non-iMac links.
- `dp_imac5k_probe_peer_aux()` does a 1-byte DPCD read on the *peer* tile and logs `IMAC5K: aux_probe func=... checkpoint=... peer_role=... dpcd_rev=... status=...`. This is the AUX-death detector: the first `dpcd_rev=0xff status=-1` after a healthy `0x12/status=1` pins the killing call.

Instrumented call sites (entry/exit, always paired log + peer probe): `dp_enable_link_phy`, `dp_disable_link_phy`, `dpcd_write_rx_power_ctrl` (non-skip path), `dce110_enable_dp_link_output`, `dce110_disable_link_output`, `power_down_encoders` (per link iteration), `power_down_all_hw_blocks` (sequence-complete), `link_blank_dp_stream`, `disable_link_dp`, `disable_link`, `enable_link_dp`.

Caution learned from the trace: `phy_state.symclk_state` is only updated by `dce110_disable_link_output` / `dce110_enable_dp_link_output`, NOT by the raw `link_enc->funcs->disable_output()` used inside `power_down_encoders`. After `power_down_encoders` it reads a stale `on/on` even when the transmitter is off. Trust the `aux_probe` `dpcd_rev`/`status`/`hpd_hw` fields, not `symclk_state`.

## OBS-19 RESULT: Root Cause Found — power_down_encoders Kills The Secondary

Input log:

- `C:\proj\reverse\WT6A_INF\obs_19\dmesg.txt`.
- Kernel: `7.0.1-1-imac-5k-g3ae56688ab41`, build `#21`, dated `Thu, 14 May 2026 14:35:10 +0000`.
- This build contains both the OBS-18 patch and the new phy-trace/aux-probe instrumentation.

What the OBS-18 patch achieved:

- `enabling link N failed` count went from present in OBS-18 to **0** in OBS-19. The destructive fresh `dpcd_set_link_settings` storm is gone.
- `cached-link-handoff accepted on link 1` fired at `t=7.83s`, promoting the secondary to stream state `3:state1-enabled-completed`. The relaxed blocker and `cur_link_settings` seeding work as intended.

Root cause proven by the new instrumentation:

- `power_down_encoders()` in `dc/hwss/dce110/dce110_hwseq.c` is the AUX killer. It runs once at `t=7.043s` (from `power_down_all_hw_blocks()` <- `dce110_enable_accelerated_mode()` on the first commit), iterates every `dc->links[]`, and directly calls `link_enc->funcs->disable_output()` plus clears `link_active` / `cur_link_settings`.
- The iMac preservation guard `link_should_preserve_imac5k_secondary_source_output()` only lives inside `dp_disable_link_phy()`. `power_down_encoders` bypasses it entirely.
- Timeline from the trace:
  - `t=7.052244` primary iteration `after-link-enc-disable-output`: peer (secondary) still alive, `dpcd_rev=0x12 status=1`.
  - `t=7.052-7.053` secondary iteration: secondary's own `disable_output()` runs. Preservation flags were already armed (`preserve_trained=1 skip_d3=1 handoff_allowed=1 handoff_consumed=0 stream_state=1 evidence=0x3f`), but the path ignores them.
  - `t=7.109847` first peer probe of the secondary after that point: `dpcd_rev=0xff status=-1 peer_hpd_hw=0` — AUX dead. Nothing else touched the secondary in the gap.
- After this, the `enable_link_dp` cached-link-handoff path skips `perform_link_training_with_retries` (hence `dp_enable_link_phy`), so the transmitter `power_down_encoders` killed is never re-powered. The handoff is "accepted" but the link is physically dead, so every later GNOME commit re-arms the handoff against stale cached proof (`used_cached_proof=1`) while all DPCD ops return `status=-1`. End state: `stream_count=2` with both tiles in the commit but secondary `0x4F1` latch write fails — left tile works, right tile dead.

Two bugs, one fix:

- Bug A (root cause): `power_down_encoders` is unguarded.
- Bug B (exposed by the OBS-18 patch): the cached-link-handoff path skips `dp_enable_link_phy`. Once Bug A is fixed and the transmitter stays alive, Bug B becomes correct-by-design — Windows also does not retrain an already-up cached link.

## Kernel Change After OBS-19: Guard power_down_encoders

Files changed:

- `dc/link/protocols/link_dp_phy.c` / `.h`: added public `dp_imac5k_link_preserves_secondary_output()`, a thin wrapper over the existing static `link_should_preserve_imac5k_secondary_source_output()` predicate, so hw-sequencer paths that bypass `dp_disable_link_phy()` can honour the same preservation decision.
- `dc/hwss/dce110/dce110_hwseq.c`: `power_down_encoders()` now checks `dp_imac5k_link_preserves_secondary_output()` per link; when the trained iMac secondary `0x3113` route is armed for preservation, the whole iteration is skipped (`continue`) — no `blank_dp_stream`, no `disable_output`, no `link_active`/`cur_link_settings` clear. Logged as `IMAC5K: phy_trace func=power_down_encoders checkpoint=iter-skip-preserved`.
- `power_down_all_hw_blocks()` now emits a `sequence-complete` phy-trace/aux-probe checkpoint per link, so the next run confirms whether the secondary survives the full power-down sequence (encoders + controllers + clock sources) or dies in a still-unknown downstream path.

Expected `obs_20` checks:

- `IMAC5K: ... power_down_encoders checkpoint=iter-skip-preserved role=secondary` appears instead of the secondary's `before/after-link-enc-disable-output`.
- `power_down_all_hw_blocks checkpoint=sequence-complete` aux_probe of the secondary shows `dpcd_rev=0x12 status=1` (alive), not `0xff status=-1`.
- The secondary stays alive through to the first accepted two-tile commit; `cached-link-handoff accepted` is followed by a working right tile rather than a re-arm loop.
- If the secondary still dies, the new `sequence-complete` checkpoint plus the existing per-call probes will localise the next killer (likely candidates if any: `power_down_controllers`, `power_down_clock_sources`, or a post-commit path).

## OBS-20 RESULT: power_down_encoders Guard Works, Killer Moved To Clock-Source Teardown

Input log:

- `C:\proj\reverse\WT6A_INF\obs_20\dmesg.txt`.
- Kernel: `7.0.1-1-imac-5k-g4fd6b11f970c`, build `#22`, dated `Thu, 14 May 2026 16:07:20 +0000`.

What worked:

- The `power_down_encoders` guard fired correctly: `IMAC5K: phy_trace func=power_down_encoders checkpoint=iter-skip-preserved role=secondary` — the secondary `0x3113` iteration was skipped.
- `enabling link N failed` count stayed `0`. `cached-link-handoff accepted` fired once.
- The secondary survived `power_down_encoders`: at the primary's `power_down_encoders` iteration the peer probe still showed the secondary `dpcd_rev=0x12 status=1`. `cur_link_settings` was also preserved (`peer_rate=20 peer_lanes=4` instead of the obs_19 `0/0`).

What still failed:

- The secondary AUX was dead (`dpcd_rev=0xff status=-1 peer_hpd_hw=0`) by `power_down_all_hw_blocks:sequence-complete`.
- The death window is therefore between `power_down_encoders` finishing and `power_down_all_hw_blocks` returning — i.e. inside `power_down_controllers` or `power_down_clock_sources`. The single coarse `sequence-complete` checkpoint could not distinguish them.
- End state: `stream_count=2 tile_streams=2` (both tiles in the commit) but 54 failed `0x4F1` latch writes (`status=-1`) — left tile works, right tile dead.

Analysis:

- `power_down_controllers` only calls `disable_crtc()` on every timing generator. AUX lives on the DIG/UNIPHY block, not the CRTC, so disabling a CRTC should not kill AUX. Unlikely killer.
- `power_down_clock_sources` powers down `dp_clock_source` and every `clock_sources[]` PLL. The UNIPHY_D PHY needs a clock; powering down its clock source darkens the PHY and AUX with it. Prime suspect.

## Kernel Change After OBS-20: Skip Clock-Source Teardown + Exhaustive Early-Init Tracing

The fix attempt and the instrumentation are deliberately combined so `obs_21` is conclusive in every outcome — no more coarse checkpoints that need a follow-up run to narrow down.

Files changed:

- `dc/link/protocols/link_dp_phy.c` / `.h`: added `dp_imac5k_any_link_preserves_secondary_output(struct dc *dc)`, a dc-wide variant of the preservation predicate, for teardown steps that cannot make a per-link decision.
- `dc/hwss/dce110/dce110_hwseq.c`:
  - The fix: when any link needs the iMac secondary preserved, `power_down_clock_sources()` is skipped entirely (logged as `IMAC5K: power_down_all_hw_blocks skipping power_down_clock_sources` plus the `clock-sources-skipped-imac5k-preserve` checkpoint). `power_down_controllers` is left running — it is the unlikely killer (CRTC disable, AUX-independent) and skipping it would add primary-side regression risk for no expected benefit — but it is now per-TG instrumented so the run still proves whether it was innocent.
  - Generalised the checkpoint helper to `imac5k_hwseq_checkpoint(dc, func, checkpoint)` and instrumented the **entire** early-init/teardown path so one run localises any killer:
    - `dce110_enable_accelerated_mode`: `entry`, `after-init-pipes`, `full-teardown-path-entry`, `before-power-down-all-hw-blocks`, `after-power-down-all-hw-blocks`, `after-clean-up-dsc-blocks`, `after-disable-vga-and-power-gate`, `full-teardown-path-exit`, `exit`, plus a `dm_output_to_console` line dumping `can_apply_edp_fast_boot` / `can_apply_seamless_boot` / `keep_edp_vdd_on`.
    - `power_down_all_hw_blocks`: `entry`, `after-encoders`, `after-controllers`, `after-clock-sources`, `sequence-complete`.
    - `power_down_controllers`: `before-disable-crtc-tg<N>` / `after-disable-crtc-tg<N>` per timing generator.
    - `power_down_clock_sources`: `before/after-dp-clock-source` and `before/after-clock-source-<N>` per PLL (only runs when not skipped).
    - `disable_vga_and_power_gate_all_controllers`: `entry`, `after-disable-vga-tg<N>`, `before/after-disable-plane-pipe<N>`, `exit`.
- Skipping `power_down_clock_sources` is low-risk: it is a clean-slate measure, and the subsequent commit re-programs whatever clock source DC assigns; `dp_disable_link_phy`'s existing preservation path already leaves clock state alone for the preserved secondary without primary regressions.

Expected `obs_21` outcomes — each one is now self-evident from the log:

- Fix works: secondary `dpcd_rev=0x12 status=1` at every checkpoint through `enable_accelerated_mode exit`, then alive into the first accepted two-tile commit, `0x4F1` writes `status=1`, right tile lights up.
- Fix works through `power_down_all_hw_blocks` but secondary dies in `disable_vga_and_power_gate_all_controllers`: the exact `after-disable-vga-tg<N>` or `before/after-disable-plane-pipe<N>` checkpoint pins it.
- `power_down_controllers` was the real killer: the exact `after-disable-crtc-tg<N>` checkpoint pins which timing generator.
- Killer is downstream of `enable_accelerated_mode`: the existing per-call probes (`enable_link_dp`, `dp_enable_link_phy`, `dce110_enable_dp_link_output`, `disable_link`, etc.) catch it.
- Secondary survives everything but the right tile is still wrong: it is a stream-programming / `0x4F1` issue, covered by the existing dpcd-snapshot logging — and the problem has moved off the AUX-death track entirely.

## OBS-21 RESULT: Prime Suspect Was Wrong — power_down_controllers Is The Killer

Input log:

- `C:\proj\reverse\WT6A_INF\obs_21\dmesg.txt`.
- Kernel: `7.0.1-1-imac-5k-g6c282a68e0bc`, build `#23`.

What worked:

- The clock-source skip fired correctly: the `clock-sources-skipped-imac5k-preserve` checkpoint is present and the per-PLL `power_down_clock_sources` checkpoints are absent. (`dm_output_to_console` does not reach the dmesg capture, so its string was missing — use the `imac5k_hwseq_checkpoint` lines instead.)
- The `power_down_encoders` guard still holds (`iter-skip-preserved role=secondary`).
- `enabling link N failed` stayed `0`; `cached-link-handoff accepted` fired once.

What still failed — and where, exactly:

- The exhaustive per-sub-step instrumentation pinned the killer precisely. Secondary alive at `power_down_controllers:before-disable-crtc-tg0` (`dpcd_rev=0x12 status=1`), dead at `power_down_controllers:after-disable-crtc-tg0` (`dpcd_rev=0xff status=-1 peer_hpd_hw=0`). It stayed dead through every subsequent checkpoint.
- So `power_down_controllers` is the killer, not `power_down_clock_sources`. The earlier "AUX is independent of the CRTC" assumption was wrong: `dce110_timing_generator_disable_crtc()` issues an AtomBIOS `enable_crtc(bp, controller_id, false)` call. For tg0 — the controller OCLP left driving the secondary tile — that AtomBIOS script tears down the pipe→UNIPHY_D path and kills AUX.
- End state unchanged: `stream_count=2` with both tiles committed, 52 failed `0x4F1` writes, right tile dead.

## Kernel Change After OBS-21: Take The Seamless-Boot-Equivalent Path

The whack-a-mole pattern is now clear — guarding each teardown sub-step just moves the killer to the next (encoders -> controllers -> ...). The root issue is that `dce110_enable_accelerated_mode` runs a full destructive teardown that the OCLP-initialised secondary survives none of. DC already has a proven path that skips the whole teardown block: `can_apply_seamless_boot`.

Files changed:

- `dc/hwss/dce110/dce110_hwseq.c`:
  - `dce110_enable_accelerated_mode()` now skips the entire `if (!can_apply_edp_fast_boot && !can_apply_seamless_boot)` destructive teardown block when `dp_imac5k_any_link_preserves_secondary_output(dc)` is true — the seamless-boot-equivalent path. Logged with the `full-teardown-skipped-imac5k-preserve` checkpoint.
  - Defense-in-depth for the other `power_down_all_hw_blocks` caller (`dce110_power_down`): `power_down_all_hw_blocks()` now also skips `power_down_controllers()` when armed (it already skipped `power_down_clock_sources()`), and `disable_vga_and_power_gate_all_controllers()` early-returns when armed.
  - All per-sub-step instrumentation from OBS-21 is kept, so any unexpected outcome is still self-diagnosing.

Why this is safe for the primary:

- The primary (eDP) is not armed for preservation, so its `enable_link_dp` still takes the normal fresh-training path during the subsequent commit. It simply inherits OCLP state instead of being torn down first — which is exactly what the existing `can_apply_seamless_boot` path already does and is proven to boot.

Expected `obs_22` checks:

- `IMAC5K: ... enable_accelerated_mode checkpoint=full-teardown-skipped-imac5k-preserve` appears instead of `full-teardown-path-entry`.
- The secondary stays `dpcd_rev=0x12 status=1` from `enable_accelerated_mode entry` straight through to the first accepted two-tile commit.
- `cached-link-handoff accepted` is followed by `0x4F1` writes with `status=1` and a working right tile.
- Primary checkpoints (`role=primary` phy_trace lines) confirm the eDP link still trains and lights up normally.
- If the secondary still dies, the existing per-call probes localise the next killer downstream of `enable_accelerated_mode`; if the primary regresses, its own phy_trace lines show it.

## OBS-22 RESULT: STAGE 1 WORKS — 5K LIVE ON BOTH TILES

Input: `C:\proj\reverse\WT6A_INF\obs_22\obs\` (dmesg.txt + DRM state/modetest/framebuffer dumps). Kernel `7.0.1-1-imac-5k-g59fcbaea5930`, build `#24` (commit `59fcbaea5 "changes after obs 21"`). User confirmed visually and saved `obs_22/working_5k_1.zip`.

The fix worked. Every marker is green:

- `full-teardown-skipped-imac5k-preserve` fired — `dce110_enable_accelerated_mode` took the seamless-boot-equivalent path.
- The secondary `0x3113` AUX was **never** seen dead: zero `aux_probe peer_role=secondary dpcd_rev=0xff` lines in the entire run (every prior obs had the secondary dying mid-boot).
- `cached-link-handoff accepted` 1, `rejected` 0, `enabling link failed` 0.
- DPCD `0x4F1` Apple/tiled panel latch: 103 writes with `status=1`, 0 with `status=-1` (obs_18-21 had 52-54 failures each).
- 102 stable two-tile commits (`stream_count=2 tile_streams=2 primary_tile_streams=1 secondary_tile_streams=1`) spanning t=7.08s to t=265s — 4+ minutes of stable operation, not a momentary flash.
- DRM state dump: `crtc-0` -> `eDP-1`, `crtc-1` -> `DP-1`, both active at `2560x2880@60`, both planes scanning live `gnome-shell` framebuffers. `modetest` shows both `eDP-1` and `DP-1` connected.
- Secondary stream-enable-state lifecycle behaving: `0 -> 1 -> 3:state1-enabled-completed`, then re-armed to `1` for subsequent commits.

Solution summary (the chain that closed Stage 1):

- The blocker for obs_13 onward was never Mutter/TILE grouping — that was solved by obs_12. It was that Linux's own early-init teardown kept powering down the OCLP-trained secondary `0x3113` route before the cached-link handoff could consume it.
- obs_19 instrumentation pinned it to `power_down_encoders`; obs_20 to `power_down_controllers`/`power_down_clock_sources`; obs_21 to the AtomBIOS `enable_crtc(false)` inside `power_down_controllers`.
- obs_22 stopped the whack-a-mole: `dce110_enable_accelerated_mode` skips the entire destructive teardown block when the secondary route is armed for preservation, so the OCLP-trained link survives intact and the cached-link handoff consumes it without retraining — exactly the Windows-shaped "carry the live object forward" behaviour.

Known minor issue (non-fatal, not blocking): one `atomic remove_fb failed with -22` plus a `WARNING` at `drm_framebuffer.c:1176` at t=138s — a framebuffer-cleanup splat; the display kept running normally for 127s+ afterward. Worth a follow-up pass but it does not affect the 5K result.

Remaining work: Stage 2 (Apple-free plain boot — Linux discovering/constructing the real `0x3113` route without OCLP) is still open and unchanged. The non-fatal `remove_fb` WARN is a small cleanup candidate.
