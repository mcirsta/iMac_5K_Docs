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

### macOS IORegistry Cross-Map

The public iMacPro1,1 IORegistry comparison in the OCLP/Khronokernel 5K UEFI article matches this model even though the machine uses Vega/Japura rather than the iMac19,1 Polaris path:

- Good/supported path: `ATY,Japura@0` and `ATY,Japura@1` are both attached to the internal 5K panel.
- Unsupported/generic UEFI path: `ATY,Japura@0` remains the internal panel, while `ATY,Japura@1` becomes `Empty`.
- The external Thunderbolt framebuffers remain DisplayPort entries in both cases, so the collapse is specifically the second internal tile route disappearing, not a global GPU/framebuffer failure.
- The macOS connector-personality label `LVDS` / `0x2` should be read as Apple's internal-panel personality label here, not as proof that the physical transport is legacy LVDS. Our route model remains two embedded DP-family paths: primary `0x3114` and secondary `0x3113`.
- This is the macOS-side analogue of our probe split: diagnostics/OCLP-style handoff has two replay endpoints (`0x100` and `0x200`), while plain EFI has only the primary endpoint (`0x100`) live.

The same article's iMac20,x note is also consistent with our scope boundary. iMac20,x/Navi can use DSC for a single-stream 5K internal path; pre-2020 5K iMacs and iMacPro1,1 need the dual-stream/two-framebuffer model. Do not use iMac20,x as the implementation model for iMac19,1 cold-boot enablement.

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
- CoreEG2's generic restore path is separate from the complex-display replay path. It reads a per-device-path `SavedConfig` blob through `qword_E668` and then calls GOP device-protocol slot `+0x38`, identified as `GopApplySavedDisplayConfiguration`.
- Provider naming correction from the 2026-05-20 IDA pass: `qword_E668` is the Device Path Property Database provider (`91BD12FE-F6C3-44FB-A5B7-5122AB303AE0`), while `qword_E660` is the Platform Info Database provider (`AC5E4829-A8FD-440B-AF33-9FFE013B12D8`).
- `GopApplySavedDisplayConfiguration` copies the `SavedConfig` blob into GOP state/properties, probes connector-record caches through CoreEG2's shared graphics protocol if needed, then builds the GOP display-connect mask. For an internal DP route, masks in `0xF00` classify as connector type `11`.
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
- That provider read is through `qword_E660`, the Platform Info Database protocol `AC5E4829-A8FD-440B-AF33-9FFE013B12D8`, published by `ApplePlatformInfoDatabaseDxe.efi`; the misleading old CoreEG2 callback label was corrected in IDA to `CoreEg2RegisterPlatformInfoDatabaseProvider`.
- `SavedConfig`, `ComplexDisplaySetup`, AUX power, backlight/gamma, and failure records use `qword_E668`, the Device Path Property Database provider. Do not confuse this property provider with the `dword_E670` platform-policy provider.
- The raw platform database maps selector `1823A1A6-...` to `j139 -> 5` and `j138 -> 1`; the same database maps `j139` to `iMac19,2` and `j138` to `iMac19,1`.
- Raw firmware recheck: the selector GUID byte pattern is not present in the PE32 module set. It appears in the platform-info raw freeform file `5E7BE016-33CF-2D42-8758-C69FA5CDBB2F`, raw section offset `0x78`, whose value block at `0x7A0` contains exactly the `j139 = 5` and `j138 = 1` rows. The AC5E provider GUID appears in many PE modules because it is the database service protocol; the selector value itself is static platform database data.
- Therefore, for the target `iMac19,1`, the best current evidence is `dword_E670 = 1`: bit0 should be set. Treat "plain boot has bit0 clear" as unproven and probably wrong unless a direct firmware trace/log proves it.
- The grouped `0xF00` path is real on the GOP side, not just an Apple-side label.
- The lower connector backend has a grouped/live subtype `20` / `0x14`.
- `CoreEG2`'s paired-root success path correlates with that grouped subtype.
- `CoreEG2` reaches GOP's `GopGraphicsConnectorBackendDispatchOpcode` for full-opcode connector backend operations.
- GOP backend mode `a4 == 1` preserves full opcodes up to `0xFFFFF`; CoreEG2 decimal `1265` is therefore sent as full `0x4F1`, not truncated to `0xF1`.
- Selector `5` clarification: CoreEG2 uses selector/opcode `5` as a one-byte status gate and only checks `(reply[0] & 6) == 2`. The later root class byte comes from the opcode/window `0` 16-byte classification block; `display+0x12C` is the first byte of that block and is accepted in the `0x10..0x14` range.
- GOP grouped-live clarification: for grouped/internal mask `0xF00`, GOP marks the grouped-live latch from the grouped state slot byte `+25`. That latch upgrades outgoing grouped packet subtype `19 -> 20` and is also required for special grouped custom-rate validation.
- 2026-05-20 grouped-slot source clarification: no GOP C-code writer for grouped state-slot byte `+25` has been found. `GopRefreshConnectorStateSlotFromExtendedWindows` reads extended opcode/window `0x000`, length `16`, into `slot+12..slot+27`; therefore `slot+25` is byte `13` returned by the lower connector mailbox after the grouped `0xF00` backend state is selected.
- The grouped-live byte is sampled only on refresh paths. The known refresh callers are:
  - `GopPrepareDpTrainingStateForSelectedMask`, reached from CoreEG2 backend `+0x28` training/apply calls.
  - `GopTrainDetectedDpOutputByMask` and `GopTrainConnectorMaskWithDpTrainingLoop`, reached during detection/training.
  - `GopValidateAndRefreshConnectorStateSlot`, reached during connector-record cache population.
- Connector-record cache population probes ordinary DP-family masks (`0x100..0x400`) and does not directly populate the grouped `0xF00` state slot. Grouped `0xF00` becomes active later when saved state or CoreEG2 training selects the grouped/internal family.
- `GopSelectConnectorMaskFromSavedState` is a framebuffer-protocol-table callback that copies a saved connector field, then masks it by saved mode type. Saved mode type `0xB` keeps only `0xF00`, so a valid Apple saved display state can intentionally route GOP into the grouped internal DP family. This still does not by itself prove grouped-live; the lower mailbox byte still has to become nonzero.
- GOP's extended opcode transport is also used for normal DP DPCD training addresses (`0x100`, `0x101`, `0x102`, `0x103`, `0x107`, `0x108`) and training-status readback (`0x202`).
- GOP has an optional extended `0x10A` write path gated by internal Apple/panel flags, which reinforces the current secondary-route ASSR/panel-policy direction.
- CoreEG2's own training-status checker reads DPCD `0x202`, length `2`, through the same backend transfer and expects `0x77/0x77` for four lanes, `0x77` for two lanes, or `0x07` for one lane.
- GOP connector id `1` maps to backend mask `0x100` / record-cache index `4`; connector id `2` maps to backend mask `0x200` / record-cache index `3`. CoreEG2's sibling rule is `root connector id + 1`, so this is the firmware-side root/sibling pair.
- 2026-05-20 focused collapse-fork correction: CoreEG2's ordinary root/sibling complex-display path does not start by selecting grouped `0xF00`. The GOP connector object for connector id `1` has supported mask `0x120`, which dispatches as `0x100`; connector id `2` has supported mask `0x220`, which dispatches as `0x200`. Grouped `0xF00` is still real for saved-state/grouped training paths, but the immediate 5K-vs-fallback fork is whether the individual sibling `0x200` connector record becomes valid after the root-side `0x4F1 = 1` pulse.
- CoreEG2 connector-record fetch details: `CoreEg2FetchConnectorRecordSet(connector, 1)` first writes extended opcode `0x600 = 1`, polls `0x600` until `(status & 3) == 1`, then reads paged opcode `0xA0` into 128-byte record pages. GOP's dispatch path for `0xA0` fills the selected per-index cache and marks it valid; if the paged read returns no data, CoreEG2 sees no connector record for that root or sibling.
- GOP has a lower readiness gate before that `0xA0` read: `GopGraphicsConnectorBackendDispatchOpcode` calls `GopWaitForConnectorMaskReady(state, connector_supported_mask)`. For the iMac pair, root connector id `1` has supported mask `0x120` and waits on MMIO `0x1223C` bit `0x100`; sibling connector id `2` has supported mask `0x220` and waits on MMIO `0x1223C` bit `0x00000001`. Both masks carry bit `0x20`, so GOP polls for up to about `210 ms` before giving up. This is now the earliest concrete lower-level gate for the sibling `0x200` record becoming readable.
- GOP immediate search found `0x1223C` only in the readiness/transaction code, with no GOP C-side writer. Current interpretation: the readiness bits are lower hardware/mailbox status produced by the selected connector bank/panel/TCON state, not a software flag that BootPicker or AppleBds directly stores.
- CoreEG2 has a root pre-pass that can write `0x4F1 = 0` if the root connector record already looks like the Apple internal-panel record, then it refetches the root before creating the display object. The actual second-tile enable pulse remains the later `0x4F1 = 1` in `SetupSecondConIntAppleCD`.
- CoreEG2's display-object destroy path also writes `0x4F1 = 0` when a complex-display block exists and the cached root record still has the Apple internal-panel signature. That is cleanup/unlatch behavior, not a second enable path.
- After the sibling `0x200` record passes identity checks, CoreEG2 refetches the root `0x100` record and requires the root to still match the same Apple internal-panel identity. Therefore the collapse can happen in three concrete places after the platform bit: no sibling connector object, no/invalid sibling `0xA0` record after `0x4F1 = 1`, or root no longer validates after sibling validation.
- CoreEG2's final paired-display decision still depends on live connector-record validation after `0x4F1 = 1`: sibling record bytes `+8/+9 == 0x0610`, byte `+0x0B == 0xAE`, and `(flags at +0x0A & 3) == 2`.
- The final firmware-side 5K commit point is not OCLP code replacing CoreEG2; it is CoreEG2 reaching `GopReplaySavedDisplayConfiguration(record)` during DXE driver-binding connect.
- `display + 0x25C` / `display + 604` is a key live flag/gate for whether complex-display data is consumed later.
- `CoreEG2ConfigureAndActivateDisplayPipeline` is a hard consumer of the success-side complex-display record.
- The strongest static boundary reached was that selector/class bytes come from raw lower-transport results.
- 2026-05-20 `boot.efi` pass: `boot.efi` is now mostly a negative/directing result for 5K init. It does connect/open/query preboot graphics state, but it does not contain the CoreEG2/GOP DPCD/tile operations (`0x4F1`, `0x202`, grouped `0xF00` training, etc.).
- `ConnectGraphicsDevices` locates Apple graphics-connect protocol `8ECE08D8-A6D4-430B-A7B0-2DF318E7884A`, calls slot `+0x18` for `ConnectAll` and slot `+0x08` for `ConnectDisplay`, and then reads Apple NVRAM variable `gfx-saved-config-restore-status`. In this IDB that variable is only referenced here, so `boot.efi` appears to consume/report restore status rather than construct the saved display config blob.
- Binary search across `RE_Files/FirmwarePE32_All` finds the `8ECE08D8-...` graphics-connect GUID in `AppleBds.efi` and `SlingShot.efi`, not in CoreEG2/GOP. `AppleBds.efi` also contains strings `START:ConnectGfxDevs`, `START:ConnectController`, `END:ConnectController`, `END:ConnectGfxDevs`, and `CoreEG2`. Local disassembly shows the published AppleBds protocol table has slot `+0x08` at `AppleBds.efi+0x11159`, which calls a `ConnectGfxDevs`-looking routine at `+0x712F`; that routine eventually calls UEFI `BootServices->ConnectController(..., Recursive=TRUE)` for selected graphics handles. This makes Apple BDS / boot-manager connection order the stronger suspect for the OCLP/plain difference.
- 2026-05-20 `AppleBds.efi` IDA pass:
  - `AppleBds.efi+0x1210` installs `APPLE_BDS_GRAPHICS_CONNECT_PROTOCOL_GUID = 8ECE08D8-A6D4-430B-A7B0-2DF318E7884A` with interface table `AppleBdsGraphicsConnectProtocol` at `0x21A30`.
  - `AppleBdsGraphicsConnectProtocol+0x08` is now named `AppleBdsGraphicsConnectDisplaySlot`; this is the slot `boot.efi` labels `ConnectDisplay`.
  - `AppleBdsGraphicsConnectProtocol+0x18` is now named `AppleBdsGraphicsConnectAllSlot`; this is the slot `boot.efi` labels `ConnectAll`.
  - `AppleBdsGraphicsConnectDisplaySlot` forwards to `AppleBdsConnectGfxDevsAndBootUiPath(0, 0)`.
  - The first display-relevant phase of `AppleBdsConnectGfxDevsAndBootUiPath` calls `LocateHandleBuffer(ByProtocol, EFI_PCI_IO_PROTOCOL_GUID)`, reads PCI config space through `EFI_PCI_IO.Pci.Read(width=Uint32, offset=0, count=16)`, and filters for PCI base class `0x03` display controllers.
  - It creates candidate records containing the PCI controller handle and a priority byte. The priority is `1` when the read PCI config data contains Apple vendor/subsystem value `0x106B`, otherwise `2`.
  - The `START:ConnectGfxDevs` loop only enters the `START:ConnectController` path for priority-`1` records first; the call is `BootServices->ConnectController(controller, NULL, NULL, TRUE)`.
  - After successful connect, AppleBds enumerates GOP handles and matches their device path back to the same PCI controller. This confirms the recursive connect is expected to produce/refresh graphics output handles for that controller.
  - 2026-05-20 13339 MCP recheck: `AppleBdsBootManagerLoop` calls `AppleBdsConnectAllDispatchLoop` at multiple boot-manager/retry/selected-entry sites (`0x45CD`, `0x482A`, `0x4834`, `0x509F`, `0x51E5`, `0x5748`, `0x57FA`). `ConnectAll` immediately runs `AppleBdsConnectGfxDevsAndBootUiPath(0, 0)`. This strengthens the current model that Apple-mediated boots can perturb graphics connection order by BDS-side recursive graphics reconnects before `LoadImage`/`StartImage`, while BootPicker itself only classifies/labels Windows entries.
  - Correction/nuance from the same pass: `ConnectAll` only calls `ConnectGfxDevs` when `byte_2239C != 1`. `ConnectGfxDevs` sets `byte_2239C = 1` early, so later `ConnectAll` calls usually run `sub_DE3B` plus `sub_DEB4` instead. `sub_DE3B` refreshes console device-path variables (`ConOut`, `ConIn`, `ErrOut`); `sub_DEB4` does a broad `LocateHandleBuffer(AllHandles)` loop and calls `ConnectController(handle, NULL, NULL, TRUE)` for every handle. So AppleBds can perturb graphics either through the dedicated Apple display-controller pass or through broad recursive connect, but it is not a simple unconditional "run 5K init now" switch.
  - `AppleBdsLoadAndStartBootOption` performs `LoadImage`/`StartImage` without a direct call to `ConnectGfxDevs`; BootManagerLoop's `ConnectAll` calls are mostly earlier/fallback/retry paths around boot-option enumeration. This means AppleBds static RE alone probably cannot prove the OCLP/plain difference. We need to observe whether the final pre-Linux/OCLP path invokes the graphics-connect protocol or broad recursive connect after the plain path has already failed.
  - Comments were added and saved in the AppleBds IDB at the `ConnectGfxDevs` entry, its `START:ConnectController` log site, and the BootManagerLoop `ConnectAll` call sites so this does not get rediscovered as a BootCamp string branch.
- 2026-05-20 `AppleBds.efi` Apple-vs-generic boot classification pass:
  - No explicit `BootCamp`, `Windows`, or `Microsoft` string was found in the current AppleBds string table. The boot split appears to be metadata/protocol based, not a literal BootCamp-name branch.
  - `AppleBdsEvaluateExtendedBootPolicy` (`0x1C153`) resolves the selected boot device path to `EFI_SIMPLE_FILE_SYSTEM_PROTOCOL_GUID` through `AppleBdsResolveSimpleFsAndPath` (`0x1B9B2`) and then calls `AppleBdsGetAppleBootVolumeIdentity` (`0x1BBCE`).
  - `AppleBdsGetAppleBootVolumeIdentity` first probes root-file `GetInfo` GUID `900C7693-8C14-58BA-B44E-974515D27C78`; if present it copies a 16-byte Apple volume identity and sets the high Apple-identity output flag.
  - If that is absent, it probes root-file `GetInfo` GUID `FA99420C-88F1-11E7-95F6-B8E8562CBAFA`; if that fails too, it falls back to `APPLE_PARTITION_INFO_PROTOCOL_GUID = 68425EE5-1C43-4BAA-84F7-9AA8A4D8E11E`.
  - The Apple partition-info fallback accepts Apple partition type/name evidence: `Apple_Boot`, `Apple_HFS`, `Apple_HFSX`, `Apple_Recovery`, or matching Apple partition type GUID encodings. Accepted fallback paths set the low Apple-identity output flag.
  - `AppleBdsEvaluateExtendedBootPolicy` later tests the combined Apple-identity flags. If neither Apple volume metadata nor Apple partition evidence was found, it forces result type `1`, the normal generic boot path. If Apple identity exists, it preserves the richer Apple EFBS/multiboot result and may set Apple failed-boot variables, recovery, boot-picker, or bridgeOS handoff state.
  - `AppleBdsBootManagerLoop` (`0x3FE1`) and `AppleBdsLoadAndStartBootOption` (`0xBA39`) use `APPLE_BOOT_POLICY_PROTOCOL_GUID = 62257758-350C-4D0A-B0BD-F6BE2E1E272C` to synthesize real boot file paths for media-only / Apple-policy boot entries before `LoadImage`. Generic EFI entries with explicit file paths do not need this AppleBootPolicy synthesis.
  - Current interpretation: firmware probably does not "know BootCamp" as a positive string in this path. It recognizes Apple-blessed volumes positively; BootCamp/systemd-boot/plain EFI are the negative case that lacks AppleBootPolicy/ApplePartitionInfo/root-GetInfo identity and therefore follows generic EFI boot handling.
- 2026-05-20 AppleBds BootCamp false-lead cleanup:
  - The tempting `#[BDS|BC:SS]` log in AppleBds is not BootCamp. IDA context shows `BC` here is boot-chime/startup-sound handling: the function reads `StartupSound`, `StartupMute`, `MuteNextBoot`, `PanicMedic`, and `CriticalBattery`, then logs a `BootChimeOnLastBoot` data-hub record.
  - IDA comments were added at `AppleBds.efi+0xFA00`, `+0x1D52B`, `+0x2AF9`, and `+0x2ACC` so this path does not get rediscovered as BootCamp display logic.
  - `BootPicker.efi` does contain user-facing strings such as `Windows`, `WININSTALL`, and `Efi Boot`; unlike AppleBds, this remains a live RE target for how Apple's boot picker classifies or launches Windows/BootCamp entries before the OS loader starts.
- Important correction/nuance: this Apple-volume classifier is not sufficient to explain OCLP. OCLP/OpenCore can boot from a generic EFI partition and still perturb the graphics state. OpenCore has UEFI graphics/driver connection options such as `ConnectDrivers`, `ReconnectGraphicsOnConnect`, `ProvideConsoleGop`, `GopPassThrough`, and `DirectGopRendering`; upstream docs describe `ReconnectGraphicsOnConnect` as disconnecting/reconnecting graphics drivers during driver connection and requiring `ConnectDrivers`. So OCLP can plausibly trigger the AppleBds/CoreEG2/GOP graphics-connect path without being classified as an Apple partition.
- The RE consequence is that we should keep two candidate explanations separate:
  - Apple/macOS boot target classification: AppleBds positively recognizes Apple-blessed volumes through BootPolicy/ApplePartitionInfo/root-GetInfo metadata.
  - OCLP preboot graphics trigger: OpenCore may explicitly reconnect/provide GOP/console graphics and thereby cause CoreEG2/GOP driver-binding or saved-state replay to run in the good order.
- 2026-05-20 OCLP PR #777 / Khronokernel article checkpoint:
  - The public OCLP 5K workaround is explicitly a Hardware Diagnostics disguise, not a normal OpenCore graphics setting. The described chain is Apple `boot.efi` -> `/System/Library/CoreServices/.diagnostics/diags.efi` -> first hardware driver `.../.diagnostics/Drivers/HardwareDrivers/Product.efi` -> OpenCore/real loader.
  - The article states Apple firmware renders Boot Picker in 5120x2880 using the dual-DP stream, but if a non-Apple UEFI entry is loaded it forces compatibility mode: single DP stream, 3840x2160, product-id style `ae01` instead of full `ae03`.
  - It also states Apple appears to check the signature of every loaded UEFI application, so simply renaming a third-party loader to `boot.efi` is insufficient. The diagnostics path works because Hardware Diagnostics loads product drivers through an Apple-controlled/encrypted-file-buffer path, letting the loader appear as a native Apple diagnostics driver.
  - This should steer RE away from "OpenCore setting X sets 5K" and toward "which Apple firmware image-load / diagnostics product-driver path avoids or delays the unsupported-entry collapse until after CoreEG2/GOP paired replay is live."
  - Open question to resolve with RE: where is the unsupported-image collapse implemented? Candidates are AppleBds/BootPolicy/boot.efi diagnostics-load code that classifies image signatures and then calls into graphics-connect/CoreEG2/GOP cleanup, versus a lower Apple diagnostics policy path that prevents the collapse from being requested in the first place.
- 2026-05-20 BootCamp firmware RE checkpoint:
  - The Polaris GOP publishes a separate private BootCamp support protocol, `GOP_BOOTCAMP_SUPPORT_PROTOCOL_GUID = A3249A94-CB66-43FC-92F3-8BA1889FD6C7`.
  - GOP driver-binding start now installs this protocol after the GOP device, connector, and framebuffer protocol stack is present. The interface is embedded at GOP display-state `+0x320`; method slot `+0x08` is `GopBootCampPrepareLinkState`.
  - `GopBootCampPrepareLinkState(a1, a2, a3)` treats `a3 == 1` as a no-op success. For `a2 == 1 || a2 == 2`, it calls `GopApplyBootCampLinkRegisterSequence(state, 0xF4000000)`.
  - That helper is GPU MMIO handoff plumbing, not a connector-mailbox/DPCD transaction. It toggles CRTC blank-control bit `0x100` on either the current endpoint or all GOP replay endpoints, writes global registers `0x2024` and `0x2C04`, pokes DCP primary-surface address/high registers, then restores the CRTC blank bit.
  - The sequence uses GOP multi-display state (`state+2216`, `state+2212`, `state+2210`, endpoint bank bytes at `state+2276+i`), so if saved/replayed dual-tile state is already active, the BootCamp hook can touch both tile endpoints.
  - Static writer search now makes that dependency sharper: `state+2216` and `state+2212` are set by `GopReplaySavedDisplayConfiguration(non-NULL)` and cleared by the replay cleanup/`NULL` path. The BootCamp method consumes that replay state; it does not appear to create it.
  - GOP output-bank mapping is also decoded: backend mask `0x100` maps to output bank byte `0`, mask `0x200` maps to bank byte `1`, while grouped mask `0xF00` maps to bank byte `0`. A real two-tile BootCamp handoff therefore needs replay endpoints that resolve to the split `0x100/0x200` connector objects, not only one grouped `0xF00` endpoint.
  - PlatformInit caller side is now confirmed in `PlatformInitCallGopBootCampSupport` (`PlatformInit.efi+0x1170`). PlatformInit registers this routine as an `EVT_SIGNAL_EXIT_BOOT_SERVICES` callback from its module entry path, so this is an OS-handoff-time hook.
  - That callback calls `LocateHandleBuffer(ByProtocol, GOP_BOOTCAMP_SUPPORT_PROTOCOL_GUID, ...)`, walks `ProtocolsPerHandle`, opens the exact protocol with `OpenProtocol(..., EFI_OPEN_PROTOCOL_GET_PROTOCOL)`, checks interface version `0x10000`, then calls method slot `+0x08` as `(iface, 1, 2, 0, 0, 0)`.
  - Therefore the GOP sees `a2 == 1` and `a3 == 2`: this definitely takes the real `GopApplyBootCampLinkRegisterSequence(..., 0xF4000000)` path, not the `a3 == 1` no-op path.
  - One PlatformInit gate remains: `PlatformInitSkipBootCampSupportCall` suppresses this callback when set. The only writer found so far is `PlatformInitSetSkipBootCampSupportCall` (`PlatformInit.efi+0x13D1`), a protocol-notify callback for GUID `0DFD015E-A1BB-4E63-8372-89BD526DE956`.
  - The source of that skip notify is now clear: `boot.efi` locates `EFI_OS_INFO_PROTOCOL_GUID = C5C5DA95-7D5C-45E6-B2F1-3FD52BB10077`, calls slot `+0x08` with `"macOS 11.0"`, then slot `+0x10` with `"Apple Inc."`.
  - In `EfiOSInfo.efi`, slot `+0x10` stores the OS vendor string. When the stored vendor is exactly `"Apple Inc."`, EfiOSInfo transiently installs/uninstalls GUID `0DFD015E-...`; this pulse wakes PlatformInit's notify callback and sets `PlatformInitSkipBootCampSupportCall = 1`.
  - Therefore the PlatformInit/GOP BootCamp handoff is the non-Apple/generic OS handoff path. Apple `boot.efi` deliberately suppresses it by declaring OS vendor `"Apple Inc."`.
  - Historical driver checkpoint: the first public Boot Camp/Windows path that enabled full 5K on the 2014 5K iMac appears to be the Apple Software Update "Boot Camp Update 6.0 for iMac 5K" from November 2015, with AMD display driver `15.201.2001.0` dated `2015-10-05`, package string `15.201.2001-15005a-295390C`, `ProductVersion 6.0.6237`, and INF `C0295241.inf`. The older Boot Camp 6 package seen in reports used AMD driver `15.200.1060.0` dated `2015-07-15`, `ProductVersion 6.0.6133`, and did not provide 5K for the affected systems.
  - RE implication: `15.200.1060.0` -> `15.201.2001.0` is a high-value old/new Windows driver diff target because it is the smallest known public transition from 4K-limited Boot Camp behavior to Windows 5K behavior.
  - `AppleBcUpdate6.0.6171` / Apple product `031-55711` is a misleading bridge candidate. Public posts label it as Boot Camp `6.0.6171` from `2016-04-01`, but local inspection of the Apple CDN `AppleBcUpdate.exe` shows `Bootcamp.xml` reports BuildNumber/ProductVersion `6.0.6136` and the AMD display payload is still `C0186304.inf` / `DriverVer=07/15/2015,15.200.1060.0000`, not `C0295241.inf`. Therefore it is not the 5K-enabling AMD branch and should be treated as another old-driver reference point, not the target diff endpoint.
  - Local BootCamp diff checkpoint: the unpacked `BootCamp/4K/6.0.6136` tree contains AMD display INF `C0186304.inf` and branch subdir `B186909`; the unpacked `BootCamp/5K/BootCamp6.0 6237` tree contains AMD display INF `C0295241.inf` and branch subdir `B295223`. The 5K package targets Apple Tonga `DEV_6920` / subsystems `014C106B` and `014D106B` (`R9 M395/M395X`), while the older 4K package targets Apple Tonga `DEV_6938` / subsystem `013A106B` (`R9 M295X`).
  - Apple display-package check: both trees include `AppleDisplayInstaller64.exe`, but its embedded `aaplmonf64.inf` is the same generic Apple HID/display filter package (`DriverVer=01/23/2009,3.0.0.0`), not a 5K EDID or tile-enable payload. This makes the AMD display driver the useful diff target, not the Apple display installer.
  - AMD display-driver diff checkpoint: extracted local binaries are `tmp_bc_amd_compare/4K_atikmdag.sys` (`21622272` bytes, SHA256 prefix `B7FD68E6E3AA4605`) and `tmp_bc_amd_compare/5K_atikmdag.sys` (`21519872` bytes, SHA256 prefix `3C63756B65789549`), plus `atikmpag.sys` (`665088` -> `484864` bytes). Both `atikmdag.sys` binaries already contain generic strings for `DalEnable5kTiledMode`, `DalEnableTiledDisplay`, `DPCD`, `AUX`, `EDID`, and MST/SST handling, so the 5K change is likely a code/data behavior change behind existing AMD DAL paths, not a newly named feature. The newer binary adds visible strings such as `Forced EDID read.` and `DalSendDPMSNotification`, but no obvious Apple-only 5K opcode string.
  - Current interpretation: BootCamp firmware support is real and active for non-Apple OS handoff when the GOP protocol is present, but this GOP method looks like scanout/link handoff after display state exists. It does not by itself explain how the panel becomes grouped-live/tile-paired; that still depends on the earlier CoreEG2/GOP grouped `0xF00` state and whether the GOP BootCamp protocol is present by handoff time.
  - New probe implication: `Probe_results/new_22` proves the GOP BootCamp protocol is present even in the plain generic path, but replay is `live=0/count=0`. So the missing BootCamp-vs-plain answer is not "who publishes GOP_BOOTCAMP_SUPPORT"; it is who makes CoreEG2/GOP reach paired replay state before the BootCamp handoff consumes it.
  - Open BootCamp question: if real Windows BootCamp cold-boots at 5K, the paired state is likely created either by Apple BootPicker/AppleBootPolicy/graphics-connect ordering before `bootmgfw.efi`, or later by the Windows driver reconstructing the second tile through its own object-table/AUX route. Current firmware RE favors investigating BootPicker next.
- `QueryBootDisplayState` opens GOP/fallback framebuffer state, then locates `APPLE_SHARED_GRAPHICS_PROTOCOL_GUID = 63FAECF2-E7EE-4CB9-8A0C-11CE5E89E33C` and calls protocol slot `+0x30`. In CoreEG2 this slot is now named `CoreEg2SharedGetBootDisplayState`; it only maps current CoreEG2 display object state byte/dword `display+0x20C` (`+524`) into a small boot display-state code. It does not perform paired-display initialization.
- `BootGraphicsOpenAndDrawSplashPreview` is the renamed boot.efi wrapper that may call `ConnectGraphicsDevices(0, 1)`, `OpenBootGraphics`, `QueryBootDisplayState`, and `DrawBootGraphics`; current evidence says this is splash/UI plumbing, not the 5K tile bring-up path.
- Current interpretation: the OCLP/diagnostics difference is still likely in the Apple firmware connection/probe order or saved-state preservation before Linux, not in `boot.efi` directly programming the panel. The next narrow RE target is now the CoreEG2 sibling-record fork: why connector id `2` / backend mask `0x200` / record-cache index `3` returns a valid Apple internal-panel `0xA0` record after the root-side `0x4F1 = 1` pulse on the good path, but not on the plain path. The grouped `0xF00` state remains relevant for saved-state/grouped consumers, but it is no longer the first-order collapse point.

What firmware RE does not prove:

- It does not prove Linux needs Apple EFI runtime calls after ExitBootServices.
- It does not prove `1265` is a magical mailbox opcode that Linux must replay.
- It does not prove a one-sink fake 5K abstraction is the hardware model.
- It does not prove the PlatformInit/GOP BootCamp handoff creates the 5K grouped-live state from scratch; current evidence says it prepares/hands off scanout/link state once GOP display state already exists.
- It does not prove the OCLP/plain difference is `dword_E670` bit0. The stronger current suspect is later CoreEG2/GOP sibling connector discovery/validation after the root-side `0x4F1` latch sequence.

Next dynamic check added on 2026-05-20:

- The EDK II probe shim now logs `GOP_BOOTCAMP_SUPPORT_PROTOCOL_GUID`, `EFI_OS_INFO_PROTOCOL_GUID`, and the transient Apple-vendor notify GUID counts.
- Probe v1.3 also reads the GOP BootCamp support interface at `iface`, validates the GOP state signature at `iface - 0x320`, and logs the private replay fields `state+0x8A2`, `state+0x8A4`, `state+0x8A8`, endpoint masks at `state+0x8B0`, and output slot bytes at `state+0x8E0..0x8E7`.
- This is still read-only. The useful signal is whether a preboot path shows `live=1`, `count=2`, masks including `0x100/0x200`, and output slots including `00/01` before Linux starts.
- `Probe_results/new_21` result: probe v1.3 ran successfully. GOP BootCamp support is present (`count=1`), interface version is `0x10000`, GOP state signature is `0x30315652`, replay state is live with `count=2`, endpoint masks are `0x100` and `0x200`, and output slot bytes include `00 01` at the endpoint-bank positions. This proves the diagnostics/OCLP-style path has exactly the split two-endpoint GOP replay state that PlatformInit's non-Apple BootCamp handoff consumes.
- `Probe_results/new_22` plain EFI result: probe v1.3 also ran from the generic removable path. GOP BootCamp support is still present and valid, but GOP mode is only `3840x2160`, replay state is not live (`count=0`, `live=0`), endpoint masks contain only `0x100`, and output slot bytes show only bank `00` for the current endpoint (`... 00 FF FF FF`) rather than `00/01`. This pins the OCLP/plain difference: plain EFI does publish the BootCamp support protocol, but it lacks the paired GOP replay state that would let the handoff operate on both tile endpoints.
- Windows-starting-state checkpoint:
  - We do not yet have a probe result launched through the real Windows Boot Manager / BootPicker path, so we cannot honestly claim that Windows starts from the exact same state as `new_22`.
  - A lightweight static scan of `BootPicker.efi` found Windows-aware strings (`Windows`, `WININSTALL`, `Efi Boot`) and references to `APPLE_BOOT_POLICY` and `EFI_GRAPHICS_OUTPUT`, but no embedded references to `APPLE_BDS_GRAPHICS_CONNECT`, `GOP_BOOTCAMP_SUPPORT`, `EFI_OS_INFO`, `APPLE_SHARED_GRAPHICS`, `CoreEG2`, `DPCD`, `AUX`, or tiled-display strings.
  - BootPicker RE with efiXplorer confirms the Windows strings are in boot-entry classification/UI paths:
    - `BootPickerClassifyFsBootTarget` (`BootPicker.efi+0x3133`) probes Simple FS/Block IO, checks `\EFI\Microsoft\Boot\bootmgfw.efi`, and distinguishes `WININSTALL` media by reading the block device boot sector. It returns class codes only.
    - `BootPickerClassifyHandleDeviceKind` (`BootPicker.efi+0x2ED2`) classifies an EFI handle by device path/protocols and keeps a `0x80000` Windows marker only when `BootPickerClassifyFsBootTarget` returns the normal Windows class.
    - `BootPickerBuildChoiceUiEntry` (`BootPicker.efi+0x34F6`) builds the visible boot-choice entry; its `Windows`, `Efi Boot`, `Recovery HD`, and disk-label strings are labels/fallback names, not a display bring-up path.
    - `BootPickerResolveConsoleGopForUi` (`BootPicker.efi+0x105D0`) reads `UIScale`, resolves a GOP with non-zero dimensions, and falls back to another GOP handle if ConsoleOut GOP is unusable. It does not train links, call CoreEG2, or create paired GOP replay state.
  - IDA annotations were added for these routines and the key Windows/WININSTALL branches so we do not rediscover the same path.
  - This strengthens the read that BootPicker itself does not directly perform the missing 5K/CoreEG2/GOP replay initialization, but it still does not disprove an indirect boot-policy, BDS, or connection-order effect before Windows starts.
  - The decisive experiment is to launch the existing preboot probe from the exact Windows loader path (`\EFI\Microsoft\Boot\bootmgfw.efi` or the real Boot#### Windows Boot Manager entry). If that probe reports `live=1/count=2/0x100+0x200`, Windows receives the 5K-enabled firmware path. If it reports the `new_22` shape (`3840x2160`, `live=0/count=0`, only `0x100`), Windows starts from 4K fallback and the runtime driver rebuilds/revives 5K.

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

## BootCamp AMD Driver RE Checkpoint: Forced EDID Refresh Delta

IDA MCP mapping for the current Windows BootCamp comparison:

- `mcp__ida_pro_mcp__`: old `4k_atikmdag.sys` from BootCamp `6.0.6136`, SHA256 `b7fd68e6e3aa4605c4f847423ddf0e4c7510bb451729ef68e5e81868c2127b96`.
- `mcp__ida_pro_mcp_gop__`: newer `5k_atikmdag.sys` from BootCamp `6.0.6237`, SHA256 `3c63756b65789549abff1b38891e15f99d358178eca3f64365de4942d117971b`.

Useful names/comments added in IDA:

- Old driver:
  - `0xb4db84` -> `DalEdidRefreshIfNeeded`
  - `0xb4e53c` -> `DalReadEdidBlocks`
  - `0xb4e9e0` -> `DalReadOneEdidBlock`
  - table slot `0x13cdda8` commented as the old EDID refresh vtable/table slot.
- New driver:
  - `0xb63c34` -> `DalEdidRefreshIfNeededOrForced`
  - `0xb64630` -> `DalReadEdidBlocks`
  - `0xb64ad4` -> `DalReadOneEdidBlock`
  - table slot `0x13dfc30` commented as the new forced-capable EDID refresh vtable/table slot.

Observed delta:

- The old and new EDID clusters are almost identical: same EDID read-status helper size, same EDID block-reader shape, same EDID error path (`EDID read error: %i. Skipping EDID query.`).
- The newer 5K-capable driver adds a second `char` argument to the EDID refresh wrapper. When nonzero, it calls `DalReadEdidBlocks()` even if cached EDID flags are already set and logs `Forced EDID read.`.
- A string-level old/new diff confirmed that `Forced EDID read.` is the only meaningful new display/EDID/tile-related string seen so far.
- `DalEnable5kTiledMode` and `DalEnableTiledDisplay` are present in both drivers as registry/config-table strings, not direct code anchors.

Current interpretation:

- This does not yet prove that forced EDID refresh is the whole 5K enablement mechanism, but it is the cleanest real code delta found so far in the 5K-capable Windows driver.
- It fits our Linux/OCLP observations: the Windows driver may not need Apple firmware to leave the panel fully alive if it has a path that forces a fresh EDID/tile identity query after boot instead of trusting stale/cached state.

Indirect caller check:

- Several existing enumeration/dispatch callsites pass `EDX=1` to a returned object's vtable slot `+0x60`:
  - New driver: `0xaf32a`, `0xaf54b`, `0xd94d6`.
  - Old driver: `0x7c51a`, `0x7c73b`, `0xa8ee6`.
- Important correction: this is not yet proven to be the `DalEdidRefreshIfNeededOrForced` slot. Matching the slot offset alone is insufficient; one neighboring callsite slot shape does not fit the EDID table cleanly.
- IDA comments at these callsites were corrected to mark them as unresolved candidate force-refresh patterns, not confirmed EDID dispatches.
- The two main new-driver callsites (`0xaf280`, `0xaf4b0`) both acquire the target object through `sub_B49A0(..., 3)`. The old-driver equivalents (`0x7c470`, `0x7c6a0`) use matching `sub_81B50(..., 3)`. These lookup helpers call an object-manager vtable method at offset `+0x278` (`632`) with the type/id argument `3`, then the returned object is used for the `+0x60` forced-refresh call.
- The higher caller `sub_AF3E0` is a generic display-enumeration path. It calls `sub_AF4B0` directly for one target, or for every enabled target when `a2 == -3`. This still looks like normal Windows display enumeration rather than a string-obvious BootCamp-only path.

Next RE target:

- Identify the object returned by `sub_B49A0(..., 3)` / old `sub_81B50(..., 3)` at those callsites and confirm it is the EDID/monitor object using the `0x13dfbd0...0x13dfc60` table.
- Then walk the caller chain one layer up to determine whether this forced refresh is reached from normal Windows display enumeration, hotplug detect, tiled-display setup, or a BootCamp-specific startup path.

Follow-up correction from the object-table check:

- The only proven references to `DalEdidRefreshIfNeededOrForced` are the unwind/pdata entry and the method table slot `0x13dfc30`. No direct code caller and no direct data reference to the apparent table base `0x13dfbd0` has been found yet.
- Therefore the strong conclusion is narrower than the first pass: the 5K-capable driver contains a real forced EDID refresh callee that the old driver lacks, but we have not yet proven which runtime path reaches it.
- The generic enumeration path is still interesting because it acquires object type/id `3` and calls `+0x60` with `EDX=1`, but that may be a different object's `+0x60` method. Treat it as a lead, not evidence.
- Current next target is the object manager implementation behind the `+0x278` call in `sub_B49A0`: find which concrete vtable is stored at the display-manager field `+0xA88`, then resolve what type/id `3` returns.

Object-manager breadcrumbs:

- `sub_AFE30` initializes the display-manager field `+0xA88`. The value comes from the driver context, not from a local constructor:
  - `sub_9FDF0(this)` returns `this+0x180`.
  - `?precision@ios_base@std@@QEBA_JXZ` at `0x1a2100` is just a small field accessor returning `*(ctx + 0x20)`.
  - `sub_AF61F4()` returns `*(that + 8)`.
  - The result is stored at `this+0xA88` and later used by `sub_B49A0`.
- `sub_B49A0` itself is now best described as: per-display object lookup/acquire. It gets the per-display backend state through `sub_A5B60()` and `sub_A3BA60()`, then calls the `+0xA88` manager's method at vtable offset `+0x278` with `(backend + 0x28, type_id, 0)`.
- A quick caller scan found no obvious `sub_AF61F4()` caller that immediately resolved the same `+0x278` object-manager method. The manager object appears to be supplied by the shared context, so the next useful target is the context/provider construction path rather than the display-manager enumeration wrapper.

Confirmed forced-EDID caller path:

- The earlier `+0x60` lead was the wrong slot. `DalEdidRefreshIfNeededOrForced()` subtracts `0x20` from `this`, which proves it is installed on the DP-sink secondary interface, not the primary interface.
- New driver vtable:
  - Secondary DP-sink vtable starts at `0x13dfbe8`.
  - `0x13dfc30 = 0x13dfbe8 + 0x48` points to `DalEdidRefreshIfNeededOrForced()`.
- Old driver vtable:
  - Secondary DP-sink vtable starts at `0x13cdd60`.
  - `0x13cdda8 = 0x13cdd60 + 0x48` points to `DalEdidRefreshIfNeeded()`.
- So the real forced EDID dispatch slot is secondary-vtable `+0x48`.

The runtime path is now identified in `DisplayCapabilityService`:

- New driver names/comments added in IDA:
  - `0xbbf7f8` -> `DisplayCapabilityService_ctor`
  - `0xb6a5b8` -> `CreateDisplayCapabilityService`
  - `0xbc1588` -> `DisplayCapabilityService_RefreshCapsAndMaybeForceEdid`
  - comment at `0xbc166c`: this call invokes downstream slot `+0x48` with `force=((capability_byte6 >> 1) & 1)`.
- Old driver names/comments added in IDA:
  - `0xbaa93c` -> `DisplayCapabilityService_ctor`
  - `0xb54720` -> `CreateDisplayCapabilityService`
  - `0xbac8ec` -> `DisplayCapabilityService_RefreshCapsNoForceArg`
  - comment at `0xbac9ca`: old code calls the same downstream slot `+0x48`, but does not prepare/pass a force boolean.

Important old/new diff:

- New `DisplayCapabilityService_RefreshCapsAndMaybeForceEdid()` reads a capability blob through its own vtable slot `+0x1e0`; that tail-calls a provider stored at service base `+0x68`.
- It then computes the force flag from byte 6 bit 1 of that returned capability blob:
  - assembly: `mov dl, [capability_byte6] ; shr dl, 1 ; and dl, 1 ; call [downstream_vtable+0x48]`
  - decompiler: `force = (v13 & 2) != 0`.
- The downstream object used for this call is service base `+0x48`. `DisplayCapabilityService_ctor()` stores the factory's `a7` argument there.
- One construction path passes `a7 = 0`; the grouped/path-list path in `sub_B141C0()` passes `*(config + 0x20)` as `a7`. That is the next object to identify, because it is likely the DP-sink-like object whose EDID refresh gets forced in the 5K path.
- Old `DisplayCapabilityService_RefreshCapsNoForceArg()` has the same broad shape, but only keeps the earlier capability bytes and calls downstream slot `+0x48` without setting `DL/RDX` from byte 6. Combined with the old DP-sink callee lacking the force branch, this is a concrete 5K-capable-driver behavior change.

Current interpretation:

- BootCamp 6.0.6237 likely does not contain an obvious Apple-only "enable 5K" routine. Instead, it adds a small but meaningful recovery mechanism: when `DisplayCapabilityService` sees a capability bit on the grouped/path-list object, it forces the downstream DP sink to re-read EDID over AUX instead of trusting cached state.
- This fits the hypothesis that Windows can start from a less complete firmware handoff than OCLP/macOS and revive the tile identity by re-querying the sink. It does not yet prove the whole 5K path, but it is now a confirmed caller-to-callee chain rather than just a string/table diff.

Next RE target:

- Identify the service `a7` / config `+0x20` object in the grouped/path-list constructor path (`sub_B141C0`, especially the callers `sub_B149BC` and `sub_B14480`).
- Determine what provides byte 6 bit 1 in the capability blob returned through service slot `+0x1e0` / provider slot `+0x30`.
- If that bit corresponds to tiled/grouped internal DP capability, this becomes the best Windows-shaped model for a Linux implementation: force the secondary/tile sink EDID refresh only when the grouped path says it is needed, not as a blind global wake.

Capability-bit source for forced EDID:

- The `+0x1e0` call in `DisplayCapabilityService_RefreshCapsAndMaybeForceEdid()` is through the service's secondary vtable (`0x13ef1d8 + 0x1e0 = 0x13ef3b8`), not the primary vtable.
- New names/comments added in IDA:
  - `0xbc0720` -> `DisplayCapabilityService_GetCapabilityMask8`
  - `0xc321fc` -> `DisplayCapsProvider_GetCapabilityMask8`
  - `0xc63d98` -> `MonitorCaps_GetCapabilityMask8`
  - `0xc63dec` -> `MonitorCaps_RebuildFromEdidAndOverrideTable`
  - `0xc64ccc` -> `MonitorCaps_SetCapabilityBit`
  - `0xc649ec` -> `TranslateMonitorCapabilityCodeToBitIndex`
  - `0xc63ad4` -> `MonitorCapsProvider_ctor`
- Data flow:
  - `DisplayCapabilityService_GetCapabilityMask8()` obtains an 8-byte monitor capability mask from service base `+0x50` (`secondary this +0x30`).
  - `DisplayCapsProvider_GetCapabilityMask8()` forwards to a `MonitorCaps` object.
  - `MonitorCaps_GetCapabilityMask8()` returns/copies the qword at `MonitorCaps +0x38`.
  - `DisplayCapabilityService_RefreshCapsAndMaybeForceEdid()` then uses byte 6 bit 1 of that qword as the `force` argument for downstream DP-sink slot `+0x48`.
- The exact bit is now mapped:
  - `TranslateMonitorCapabilityCodeToBitIndex(54)` returns internal bit index `56`.
  - `MonitorCaps_SetCapabilityBit()` maps internal bit index `56` to `mask[6] |= 0x02`.
  - `mask[6] & 0x02` is exactly the bit consumed as `force=1` for `DalEdidRefreshIfNeededOrForced()`.
- Therefore the 5K-capable driver is not just "force EDID sometimes"; it forces EDID only when the monitor capability system marks bit index `56`. The unresolved question is the semantic name of raw capability code `54` / internal bit `56`.

Next RE target, refined:

- Find the monitor-specific override/capability table entry that emits raw code `54`, and identify which EDID/vendor/product path selects it.
- If that table entry matches the iMac 5K internal panel, BootCamp's behavior becomes very clear: detect this panel through the monitor capability table, set bit 56, then force a DP-sink EDID refresh during DisplayCapabilityService refresh.
- After that, compare with our Linux captured EDIDs to see whether the same identity can be recognized safely in-kernel without relying on Apple firmware/OCLP state.

Static table result:

- The newer 5K-capable `atikmdag.sys` contains a monitor capability table store built by `MonitorCapabilityTableStore_ctor` at `0xaf9540`.
- That constructor installs:
  - a small 5-entry table at `0x1416d60`, now named/commented in IDA as `MonitorCapabilityOverrideTable_5Entries`;
  - the main 118-entry table at `0x1416db0`, now named/commented in IDA as `MonitorCapabilityOverrideTable_118Entries`.
- The APP-panel force-EDID fragment starts at `0x1417490`, now named/commented in IDA as `ApplePanelForceEdidCapabilityEntries_code36`.
- Entries are 16 bytes: `manufacturer_id`, `product_id`, `raw_capability_code`, `value`.
- Entries found:
  - `APP/AE01 -> raw 0x36, value 0`
  - `APP/AE02 -> raw 0x36, value 0`
  - `APP/AE0D -> raw 0x36, value 0`
  - `APP/AE0E -> raw 0x36, value 0`
  - `APP/AE05 -> raw 0x36, value 0`
  - `APP/AE06 -> raw 0x36, value 0`
  - `APP/AE09 -> raw 0x36, value 0`
  - `APP/AE0A -> raw 0x36, value 0`
- No matching APP/AE entries were found in the older 4K driver with the same scan.
- No `APP/AE25` or `APP/AE26` raw-`0x36` entry was found in this BootCamp 6.0.6237 binary. That matters because our later Linux logs show iMac-panel IDs such as `AE25`/`AE26`; this BootCamp delta appears aimed at an older Apple 5K panel set, not necessarily the exact iMac19,1 panel table we later studied.
- A broader plausible-table scan found the main `0x1416db0..0x1417550` run as the only meaningful monitor-capability table with APP entries. The apparent raw-`0x36` hits elsewhere are not APP/product override rows and look like unrelated small numeric tables.
- `MonitorCapabilityTableStore_ctor` also wires in dynamic/config rows:
  - `0xaf9a68` -> `LoadDalPanelPatchByIdEntries`, reading `DALPanelPatchByID` and converting entries into the same 16-byte row format.
  - `0xaf9884` -> `LoadDalMonitorStereoSupportOverride`, reading `DALMonitorStereoSupport`.
  - These explain how non-static rows can be added, but the APP raw-`0x36` rows discussed above are static in the 118-entry table.

Interpretation of the BootCamp 6.0.6136 -> 6.0.6237 delta:

- It is a two-part change:
  - new static APP panel entries that produce raw capability `0x36`;
  - new runtime code that maps raw `0x36` -> internal bit `56` -> `mask[6] |= 0x02` -> force DP-sink EDID refresh.
- The older driver lacks the APP raw-`0x36` entries and also lacks the force-capable EDID callee/caller.
- So for the panel IDs in that table, BootCamp's 5K enablement likely starts by recognizing the Apple panel in the monitor capability table, then forcing a fresh EDID read through the DP sink during `DisplayCapabilityService` refresh.

Linux relevance:

- The exact table IDs from this 2016 BootCamp driver should not be blindly copied for iMac19,1. For iMac19,1 we already have stronger APP/AE25/AE26 DisplayID/tile evidence in Linux logs and later Windows RE notes.
- But the mechanism is valuable: Windows gates the recovery on panel identity/capability, then forces EDID refresh. A Linux analogue should likewise be narrowly gated on the internal Apple tiled panel identity/topology, not a global forced EDID read on every DP sink.

Second pass on the first 5K-capable BootCamp driver:

- The forced path was followed below `DalEdidRefreshIfNeededOrForced()` in the 6.0.6237 driver.
- The force branch calls `DalReadEdidBlocks()` again. That routine reads EDID block 0 and up to 3 extension blocks through the normal I2C-over-AUX machinery.
- Important IDA names/comments added in the 6.0.6237 database:
  - `0xb644f8` -> `DalEdidProbeCacheState`
  - `0xb64030` -> `DalDpAuxReadDpcdLimited16`
  - `0xb6427c` -> `DalDpAuxWriteDpcdLimited16`
  - `0xb64c4c` -> `DalDpAuxReadDpcdRetryNak`
  - `0xb65000` -> `DalDpAuxWriteDpcdRetryNak`
  - `0xb649b0` -> `DalDpAuxReadEdidBlock128`
  - `0xb64368` -> `DalDpAuxReadEdidBlock16Chunks`
  - `0xb64d38` -> `DalDpAuxReadBytesAfterOffsetWrite`
  - `0xb64f94` -> `DalDpComplianceRespondEdidTest`
- `DalReadEdidBlocks()` first tries DDC address range `0x50..0x52` until a 128-byte block is returned, then follows the extension count, capped at 3.
- The only DPCD write side path found under this EDID reader is `DalDpComplianceRespondEdidTest()`:
  - reads DPCD `0x218`;
  - if the EDID test bit is set, writes one-byte values around DPCD `0x260/0x261`;
  - this matches generic DisplayPort EDID compliance-test plumbing, not an Apple-panel-specific 5K wake sequence.
- No writes to Apple-panel-looking DPCD offsets such as `0x4f1` were found under the forced EDID branch.
- This makes the BootCamp 6.0.6237 mechanism more modest than a full panel mode switch: for the APP/AE01/AE02/AE05/AE06/AE09/AE0A/AE0D/AE0E panels, the driver recognizes the panel through the monitor capability table, sets the force-EDID bit, and re-reads EDID through the ordinary DP sink path.
- Practical implication: if Windows does a deeper Apple tiled-panel wake for newer iMacs, it is probably outside this first 5K driver's forced-EDID helper, or appears only in a later driver/table generation. The next high-value target remains the original modern BootCamp driver with the iMac19,1-era panel IDs.

Old-vs-new BootCamp extraction pass:

- Question: is the 6.0.6237 5K enablement a newly added tiled-display parser/mode-set path, or is it a smaller quirk that feeds the already-existing parser?
- Compared binaries:
  - old / 4K fallback sample: `tmp_bc_amd_compare/4k_atikmdag.sys`
  - first 5K-capable sample: `tmp_bc_amd_compare/5k_atikmdag.sys`
- Raw string scan result:
  - both binaries already contain SLS/topology/tiled-display vocabulary;
  - both binaries contain the trace/test string `Test1 5k SetMode`;
  - therefore those strings alone are not the 6.0.6237 enablement.
- Post-EDID refresh comparison:
  - new `DisplayCapabilityService_RefreshCapsAndMaybeForceEdid()` calls a post-refresh helper at `0xc323bc`; if the refresh result is `3`, it enters a mode/capability rebuild helper at `0xbc20a8`;
  - old `DisplayCapabilityService_RefreshCapsNoForceArg()` has the same broad post-refresh shape through `0xc1da18` and `0xbad480`;
  - the mode-list rebuild path is present in both old and new drivers. This is not the key delta.
- Capability translator comparison:
  - old `TranslateMonitorCapabilityCodeToBitIndex_OldNoCode36()` at `0xc4fdfc` does not recognize raw capability `0x36` / decimal `54`; that input returns `0`;
  - new `TranslateMonitorCapabilityCodeToBitIndex()` at `0xc649ec` recognizes raw capability `0x36` / decimal `54` and maps it to internal bit index `56`;
  - new `MonitorCaps_SetCapabilityBit()` then maps internal bit `56` to `capability_mask[6] |= 0x02`;
  - `DisplayCapabilityService_RefreshCapsAndMaybeForceEdid()` consumes exactly `capability_mask[6] bit 1` as the `force` argument to the downstream DP-sink EDID refresh.
- EDID / DisplayID parser comparison:
  - both drivers recognize DisplayID extension blocks via tag `0x70` and DisplayID versions in the `0x11..0x13` family;
  - both drivers have a `DisplayIdParser` constructor with the same object shape and branch structure;
  - both drivers have matching DisplayID sub-block parse/cache logic and timing/mode extraction logic;
  - IDA names/comments added around the parser helpers in both databases to mark this as existing generic infrastructure, not the first-5K-driver delta.
- Current conclusion:
  - 6.0.6237 did not appear to add the generic DisplayID/tiled-parser machinery needed for 5K;
  - it added the Apple-panel capability gate and forced-EDID rerun that makes the existing machinery see fresh panel data;
  - the concrete static part is the new APP/AE01/AE02/AE05/AE06/AE09/AE0A/AE0D/AE0E raw-`0x36` monitor-capability rows;
  - the concrete runtime part is raw `0x36` -> internal bit `56` -> `capability_mask[6] bit 1` -> forced downstream DP-sink EDID refresh.
- Interpretation:
  - for this first 5K-capable BootCamp driver, the "5K mechanism" is best described as panel-identity-gated forced EDID refresh through existing DisplayID/tile/SLS paths;
  - it is not, in this binary, an obvious Apple-specific DPCD wake sequence such as a write to `0x4f1`;
  - for iMac19,1 we still need to inspect the later/original modern BootCamp driver because its panel IDs differ (`AE25`/`AE26` in our Linux evidence), but this older comparison gives us the mechanism shape to look for.

Old-vs-new BootCamp property6 / object-enumeration check:

- Question: did the first 5K-capable BootCamp driver change the hard property6/object-enumeration policy that might suppress the secondary route, rather than only adding the forced-EDID/panel-capability path?
- Method note:
  - IDA batch export for the old/new IDBs was blocked by the local IDA batch-mode license prompt, so this pass used `objdump` structural disassembly over the already-extracted binaries;
  - confidence is good for immediate constants and local control-flow shape, but a live old/new IDA MCP pass would still be useful for nicer names/comments.
- Compared functions:
  - old `4k_atikmdag.sys`: raw adapter/source flag population around `0x6c9a0..0x6cc20`;
  - new `5k_atikmdag.sys`: equivalent raw adapter/source flag population around `0x9f190..0x9f410`.
- Property/raw-flag result:
  - both old and new clear the qword at the `+0x8c/+0x90` raw flag area, then set `0x140000` in the same branch shape;
  - old branch: runtime-cap vfunc `+0x488`, or feature `0x12c` plus root/display-core vfunc `+0x278`, then `or dword ptr [rbx+0x8c], 0x140000` at `0x6cba4`;
  - new branch: same checks, then `or dword ptr [rbx+0x8c], 0x140000` at `0x9f394`;
  - neither binary has a visible `0x940000` immediate in the disassembly scan, so the modern `0x940000`-style branch is not the first-5K-driver delta.
- Object-enumeration / route result:
  - old driver already updates object `0x3113` at `0xaa2e7a` and object `0x3114` at `0xaa2e87`;
  - new driver does the same at `0xabaac7` and `0xabaad4`;
  - both versions compute a grouped mask from per-object/backend state, write it to `0x3113`, then merge the count bits into `0x3114`;
  - both versions also contain the same AUX/DDC selector resolver vocabulary: index 2 maps to `0x4871`, index 3 maps to `0x4875` (`0xbb83f4..0xbb8457` old, `0xbcdc20..0xbcdc84` new).
- Conclusion:
  - the first 5K-capable BootCamp driver did not appear to add the `0x3113/0x3114` internal route concept, nor did it obviously change the property6 hard-suppressor raw branch;
  - those mechanisms already existed in the older 4K-only driver, which means the older driver likely failed to enable 5K because the existing route/parser machinery did not receive the right fresh Apple-panel capability/EDID data;
  - this reinforces the prior forced-EDID finding: the 6.0.6237 delta is best read as "feed the already-existing route/tile stack with fresh panel identity" rather than "invent a new secondary object route."
- Linux implication:
  - do not expect the old-vs-first-5K diff to reveal a newly added object-constructor recipe for `0x3113`;
  - the production Linux path should still focus on making the real `0x3113` route exist or survive, then applying the Apple-panel capability/EDID/DPCD policy once that route's AUX service is live;
  - if we want higher confidence on names, open the old and first-5K IDBs in two IDA MCP instances and annotate the equivalent raw-property and `0x3113/0x3114` route functions directly.

Modern/original BootCamp driver pass:

- IDA MCP target: `C:\proj\reverse\WT6A_INF\B416406\amdkmdag.sys.i64`, input `B416406\amdkmdag.sys`, SHA256 `ac79119d7a64fc2619391ed972febf6c740d0199fd6ebb2fabb4728775a2fc05`.
- This later driver keeps the same broad mechanism shape, but the iMac19,1-era path is stronger than the 6.0.6237 forced-EDID-only delta:
  - static main vendor/product capability table at `0x144C4F720`, `361` records;
  - `APP/AE25` row `149` at `0x144C50070`: source id `0x38`, value `0`;
  - `APP/AE26` row `150` at `0x144C50080`: source id `0x39`, value `0`;
  - source id `0x38` maps to pin property tag `0x3A`; source id `0x39` maps to tag `0x3B`;
  - tag `0x3A` sets pin descriptor byte `+6` bit `2`; tag `0x3B` sets byte `+6` bit `3`.
- The modern driver has two important consumers for these tags:
  - EDID side: `ApplyPinCapabilityEdidFixups()` dispatches tags `0x3A/0x3B` to `DoubleEdidPhysicalWidthForTiledCaps()`, which doubles EDID byte `21` when horizontal physical size is less than vertical and less than `0x80`, then recomputes block checksum. This is an Apple tiled-half geometry fixup.
  - AUX/DPCD side: `DetectDisplayMaybeAssert4F1ForCapableSink()`, `WindowsDM_EnableSecondaryTileIfRequired()`, and `StreamEnableWrite4F1AndPollSinkStatus()` all use descriptor byte `+6` bit `2/3` as Apple/tiled gates before writing sink DPCD `0x4F1 = 1`.
- `WindowsDM_EnableSecondaryTileIfRequired()` adds one more gate: after the `0x3A/0x3B` descriptor-bit check, the resolved lower display object type must be `128`; only then does it call `SendSecondaryTileOpcode1265Payload1()`.
- `SendSecondaryTileOpcode1265Payload1()` writes decimal `1265` (`0x4F1`) payload `1`, retries once after `10 ms` on failure, and optionally waits `100 ms` after success.
- `BringUpDisplayBySignalType()` has another post-bring-up call to the same `0x4F1` helper, but this one is downstream of successful bring-up and is gated by lower-object flag/value `+0x5B4`. For `APP/AE25/AE26`, the static value is `0`, so this is not the primary evidence for the iMac19,1 path.
- IDA annotations added in the modern IDB:
  - data comments at `0x144C50070` and `0x144C50080` for the `APP/AE25` and `APP/AE26` rows;
  - comments at `0x141A9F483`, `0x141A9F4B5`, `0x141A221FD`, `0x141A17548`, and `0x141B6A8C2`;
  - renamed `0x141A9F1F0` to `ConstructDisplayPinFromEdidApplyVendorCaps`;
  - renamed `0x1415E73C0` to `RefreshEdidCacheApplyPinCapabilityFixups`.
- Current BootCamp answer:
  - older 6.0.6237 shows the first public 5K enablement as panel-identity-gated forced EDID refresh through already-existing DisplayID/tile machinery;
  - the modern/original driver for the later iMac generation shows the fuller mechanism: panel-identity-derived Apple tiled tags `0x3A/0x3B`, EDID half-panel geometry correction, and multiple tag-gated `0x4F1 = 1` writers.
- Practical implication:
  - Windows probably does not need the firmware to hand it the exact same OCLP paired replay state. It needs enough of the real internal DP route/object graph to exist, then the driver can recognize the Apple panel from EDID and assert the Apple/tiled DPCD latch itself.
  - This does not prove Windows can revive a completely absent secondary connector. It does prove the modern Windows driver has an explicit Apple-panel path that Linux should mimic more closely than the early 6.0.6237 forced-EDID quirk.

Modern BootCamp route-construction pass:

- Notes recheck before this pass:
  - `new_21` proves the OCLP/diagnostics-style firmware path has paired GOP replay state before the loader: `live=1`, `count=2`, endpoint masks `0x100/0x200`, output banks `00/01`;
  - `new_22` proves the plain EFI path still has the GOP BootCamp support protocol, but no paired replay state: GOP `3840x2160`, replay `live=0`, `count=0`, only endpoint mask `0x100`;
  - prior Windows RE already warned that type-128/eDP readiness is an HPD-ready wait/settle path, not a new firmware mailbox opcode.
- `CreateLinksFromBiosObjectTable()` at `0x141B7A8F0` builds `dc->links[]` from the BIOS/ATOM object table, not from GOP paired replay state:
  - constructor input `+0x10` is the physical connector index from BIOS enumeration;
  - constructor input `+0x14` is the runtime `dc->links[]` index, later stored at `display_entry+0x30`;
  - this index is the one later used by lower DPCD/AUX wrappers to select the link/session.
- `ConstructLegacyConnectorDisplayEntry()` at `0x141B6D1F0` maps connector object ids into runtime signal classes:
  - object low byte `0x13` -> signal `32`;
  - object low byte `0x14` -> signal `128`;
  - object low byte `0x14` also has eDP-style panel-control creation and HPD-ready callbacks.
- This resolves the route split:
  - the Windows type-128 path is the root/eDP-style `0x14` connector path, likely the `0x3114` side;
  - the Linux/Windows secondary route we have been preserving is `0x3113`, object low byte `0x13`, signal `32`;
  - therefore the stricter type-128 gate in `WindowsDM_EnableSecondaryTileIfRequired()` is not the whole secondary story. It is a root/eDP-style helper, while signal-32 `0x3113` still has normal DP detect/bring-up and later `0x4F1` opportunities.
- This also sharpens the likely division between the APP panel ids:
  - `APP/AE25 -> tag 0x3A -> descriptor byte+6 bit2` can explain root/type-128 detect-time `0x4F1` behavior;
  - `APP/AE26 -> tag 0x3B -> descriptor byte+6 bit3` is more important for paired/secondary handling during WindowsDM and stream-enable paths;
  - `DetectDisplayMaybeAssert4F1ForCapableSink()` is only proven to check bit2/tag `0x3A`, while `WindowsDM_EnableSecondaryTileIfRequired()` and `StreamEnableWrite4F1AndPollSinkStatus()` check bit2 or bit3.
- Both routes depend on the normal per-link DDC/AUX service:
  - `ConstructLegacyConnectorDisplayEntry()` creates `display_entry+0x140 = CreateDceAuxDdcService(...)`;
  - `ConstructDceAuxDdcService()` gets firmware I2C/DDC line info and creates the DDC GPIO pin-pair service;
  - connector low byte `0x14` or `0x0E` sets AUX/DDC service flag bit `1`, confirming type-128 is ordinary AUX-capable display plumbing, not a side-channel.
- Detect behavior:
  - `QueryConnectorReadyAndArmEdp128()` runs the eDP HPD-ready and post-ready delay callbacks only when `link+0x38` / signal is `128`;
  - after the callbacks, readiness still comes from the connector-state/HPD provider for the same object id;
  - `DcLinkDetectHelper()` can create a `dc_sink` for signal `128` after this readiness gate;
  - the signal `32` path is separate normal DP detection. It does not run the type-128 HPD callbacks, but can still create/read a `dc_sink` through the same DDC/AUX service and can retry detect while connector-ready.
- IDA annotations added in the modern IDB:
  - comments at `0x141B6D6E2`, `0x141B6D727`, `0x141B6D84E`, `0x141B6E8CA`, `0x141B6EA1C`, `0x141B6EB28`, and `0x141B6F1B1`.
- Current answer refinement:
  - the Windows driver has enough static machinery to build both the root/type-128 and secondary/signal-32 link objects from BIOS object tables after ExitBootServices; this does not require GOP paired replay state as an input;
  - but it still requires the firmware/BIOS object table and connector-state/DDC/AUX providers to expose the `0x13` secondary route. If that route is truly absent, the driver cannot invent it from only the primary sink;
  - for Linux, the closest Windows-shaped production target is still: construct or preserve the real `0x3113` signal-32 route, process the APP/AE25/AE26 capability path, and assert `0x4F1` while that route's AUX service is live. Type-128 eDP readiness explains the root-side path; it is not a replacement for preserving `0x3113`.

Pre-test RE sanity pass: AE26/signal-32 expectations

- Goal before another kernel test: make sure we are not expecting the wrong Windows `0x4F1` writer on the secondary `0x3113` route.
- `ReadEdidIntoDcSinkThroughPin()` at `0x141EDDD50` is called only by `DcLinkDetectHelper()` after a link-specific `dc_sink` is created:
  - it reads EDID through the link's normal pin/DDC/AUX path;
  - constructs a capability-aware pin via `ConstructDisplayPinFromEdidApplyVendorCaps()`;
  - then calls WindowsDM vtable `+0x158`, which is `WindowsDM_EnableSecondaryTileIfRequired()`.
- That EDID-time WindowsDM helper is not the expected AE26 secondary writer:
  - `WindowsDM_EnableSecondaryTileIfRequired()` checks descriptor byte `+6` bit2 or bit3;
  - but then it requires the selected link/display type to be `128`;
  - therefore on the signal-32 `0x3113` secondary, this helper can receive an AE26/tag-`0x3B` pin and still skip the `0x4F1` write because the link is not type `128`.
- The signal-32 secondary still gets the APP/AE26 capability:
  - `ApplyVendorProductCapabilityTableToPin()` maps `APP/AE26` -> source id `0x39` -> tag `0x3B`;
  - `SetPinDescriptorCapabilityBitById()` sets descriptor byte `+6` bit3;
  - `ParseEdidAndPinCapsIntoDcSinkInfo()` copies tag `0x3B` value into sink info `+0xAC` / `dc_sink+0x5B4`, but AE26's static value is `0`, so this value-gated post-bring-up helper is not the main path either.
- The proven AE26 secondary writer is stream-enable:
  - `StreamEnableWrite4F1AndPollSinkStatus()` at `0x1414E7810` checks descriptor byte `+6` bit2 OR bit3;
  - it has no visible `signal == 128` gate at the `0x4F1` write site;
  - it calls the real stream-enable vfunc first, then writes DPCD `0x4F1 = 1`, retrying once after `10 ms`;
  - the optional `0x205` poll remains separately gated by descriptor byte `+6` bit5 / tag `0x40`, not proven for `APP/AE25/AE26`.
- Detect/status-time `WriteSinkDpcd4F1Payload1Retry()` remains root/AE25-biased:
  - `DetectDisplayMaybeAssert4F1ForCapableSink()` checks only descriptor byte `+6` bit2 / tag `0x3A` before the detect/status `0x4F1`;
  - it should not be used as proof that AE26/tag `0x3B` gets an early detect-time latch on the secondary.
- IDA annotations added in the modern IDB:
  - comments at `0x141EDDE5C`, `0x141A17548`, `0x1414E7DD5`, `0x1414E7E1A`, and `0x1414B2BF3`.
- Kernel-test implication:
  - do not judge the patch solely by whether the secondary gets an early detect-time `0x4F1`;
  - the key Windows-shaped success condition for AE26/secondary is reaching stream finalization with the real `0x3113` AUX route alive, then seeing the stream-enable-time `0x4F1 = 1` write succeed;
  - if logs show AE26/tag-`0x3B`, stream-enable attempted after both tile streams exist, and `0x4F1` status succeeds, then the RE alignment is good even if the earlier type-128/EDID helper skipped.

Extra pre-test RE pass: mode-11 gate before the stream-enable `0x4F1` writer

- Reason for this pass:
  - the previous wording "stream-enable writes `0x4F1`" was too broad;
  - the modern BootCamp stream-enable path has an additional mode/state gate before the `0x4F1` writer can run.
- New/updated IDA anchors in the modern BootCamp IDB:
  - `0x141442720`: renamed `UpdateMode11ImmediateStreamEnableFlag`;
  - `0x1414437B0`: renamed `SelectLinkSettingsRespectMode11Flag`;
  - `0x1414E59D0`: renamed `SelectFittingLinkSettingsForMode11Stream`;
  - comments added at `0x14144275A`, `0x1414427BB`, `0x14144284A`, `0x141446C34`, `0x141446C41`, `0x141446F6B`, `0x1414437CA`, `0x14144884B`, `0x1414E78F4`, and `0x1414E7DD5`.
- `UpdateMode11ImmediateStreamEnableFlag()` is called by `EnableValidatedStreamMaybeAssert4F1()` immediately before the decision to call `StreamEnableWrite4F1AndPollSinkStatus()`:
  - it queries the stream/display object through vtable offset `+0x160`;
  - return value `12` clears link-service byte `+0x40A`;
  - return value `11` sets link-service byte `+0x40A`;
  - because `EnableValidatedStreamMaybeAssert4F1()` receives `link_service+0x28`, the same byte appears there as `a1+0x3E2`.
- Therefore the wrapper only calls `StreamEnableWrite4F1AndPollSinkStatus()` when this mode-11 byte is set.
- If the byte is clear, the wrapper follows the generic validated/cached stream path:
  - optional fresh training only when cached link settings are absent;
  - `ProgramCachedLinkStreamSourceThenEnableStream()`;
  - lower stream finalizer, which our earlier RE mapped to the likely non-converter `0x107` MSA-ignore path for the iMac route;
  - no call to `StreamEnableWrite4F1AndPollSinkStatus()` from this wrapper.
- The same mode-11 byte also affects link-setting tuple selection:
  - when set, `SelectLinkSettingsRespectMode11Flag()` calls `SelectFittingLinkSettingsForMode11Stream()`;
  - when clear, it uses the indexed/default link-setting tuple path.
- The DP/MST-style stream constructor at `0x1414486E0` initializes the backing byte `link_service+0x40A` to `0`, so mode `11` must be observed later to enter the immediate `0x4F1` stream-enable wrapper path.
- Inside `StreamEnableWrite4F1AndPollSinkStatus()` itself:
  - states `2` and `3` still return success immediately;
  - state `1` programs/caches the requested tuple, programs `0x316`, calls the lower stream-enable vfunc, promotes state `1 -> 3`, and returns without the later tag-gated `0x4F1` write;
  - the tag-gated stream-enable `0x4F1` write is therefore on the non-state-1 normal branch.
- Refined conclusion:
  - the proven AE26/tag-`0x3B` `0x4F1` writer is real, but it is not unconditional for every stream-enable;
  - it requires reaching the mode-11/immediate wrapper path and not short-circuiting as state `1`, `2`, or `3`;
  - a Windows-like already-configured state-1 handoff may correctly perform stream/source finalization without a late `0x4F1` write.
- Kernel-test implication:
  - do not treat absence of a late stream-enable-time `0x4F1` as automatic RE mismatch if the Linux path is intentionally following the already-link-configured/state-1 analogue;
  - for the current kernel direction, the stronger success criteria are: trained secondary `0x3113` preserved, two real tile streams committed, lower stream programming reached, `0x107` bit7/`ignore_msa_timing_param` handled, and no destructive retraining/powerdown;
  - late `0x4F1` success is still useful evidence on the normal branch, but it is no longer the sole Windows-shape marker.
- Remaining narrow RE question:
  - map stream mode values `11` and `12` back to the selected path-mode/display-object fields for the exact iMac 5K pair;
  - this would tell us whether Windows expects the iMac secondary to use the immediate `0x4F1` branch or the generic cached/finalizer branch during the relevant commit.

Extra pre-test RE pass: mode-11 gate audit and EDID false-lead correction

- Follow-up result:
  - the EDID parser work is useful, but it does not prove the source of the mode-11 stream-enable gate;
  - the earlier idea that `UpdateMode11ImmediateStreamEnableFlag()` was directly reading EDID parser-kind constants was too strong.
- New/updated IDA anchors in the modern BootCamp IDB:
  - `0x141A9EAC0`: renamed `CreateEdidParserChainFromRawEdid`;
  - `0x141A9E440`: renamed `CreateEdidExtensionParserForBlock`;
  - `0x141AFC880`: renamed `IsDisplayIdExtensionBlock`;
  - `0x141AF02D0`: renamed `IsEdidExtensionTag40_DdcDisplay`;
  - `0x141AE0020`: renamed `IsEdidExtensionTag10`;
  - `0x141AF41F0`: renamed `DisplayIdParserVfunc160_MarkTimingFromBlock0D`;
  - `0x141A38840`: renamed `EdidParserVfunc160_DelegateToNextExtension`;
  - `0x1419F0C50`: renamed `EdidParserVfunc168_Tag40_Return11`;
  - `0x1419F0730`: renamed `EdidParserVfunc168_Tag10_Return12`;
  - `0x1419F0D30`: renamed `EdidParserVfunc168_DisplayId_Return1`;
  - mode-gate comments corrected at `0x1414427BB` and `0x141446C41`.
- EDID parser facts that remain solid:
  - `ConstructDisplayPinFromEdidApplyVendorCaps()` builds the parser chain at pin field `+0x30`;
  - `CBasePin::Name()` returns that field, so many display paths use the parser chain through the pin/name abstraction;
  - `CreateEdidExtensionParserForBlock()` selects DisplayID when an extension starts with `0x70` and version `0x11..0x13`;
  - the captured iMac19,1 primary and secondary EDIDs both have one extension block starting `70 13`;
  - the decoded captures show DisplayID tiled topology block `0x12`, with tile locations `0,0` and `1,0`.
- Slot audit correction:
  - `UpdateMode11ImmediateStreamEnableFlag()` calls vtable slot `+0x160` on `stream_arg+0x178`;
  - the EDID parser constant-return functions are at parser vtable slot `+0x168`, not `+0x160`;
  - DisplayID parser `+0x160` is `DisplayIdParserVfunc160_MarkTimingFromBlock0D()`, a timing/block helper that returns success, not the constant `1` parser-kind function;
  - several non-terminal EDID parser `+0x160` slots delegate to the next extension parser, which further confirms this is not a simple kind constant.
- `stream_arg+0x178` is also used by stream-enable code as a richer display/sink object:
  - `ProgramCachedLinkStreamSourceThenEnableStream()` calls its vfunc `+0x38` with argument `1`;
  - `StreamEnableWrite4F1AndPollSinkStatus()` calls its vfunc `+0x58` to obtain the descriptor/pin-capability object;
  - the normal stream-enable path also tests its vfunc `+0x198`;
  - this does not look like a bare EDID parser-chain pointer.
- Refined conclusion:
  - the mode-11 byte is still real, but its source is unresolved;
  - it probably comes from the higher display/mode object stored at `stream_arg+0x178`, not directly from the EDID extension parser-kind constants;
  - therefore do not infer the mode-11 late `0x4F1` wrapper expectation from the captured DisplayID `70 13` EDID alone.
- Kernel-test implication:
  - the current kernel direction does not need to pivot on this false lead;
  - the stronger Windows-equivalence target remains the generic/cached finalizer path: real secondary `0x3113`, two tile streams, source-DPCD/ASSR, `0x107` MSA timing ignore, cached-link handoff, and avoiding destructive retrain/powerdown;
  - keep logging late `0x4F1`, but do not force the mode-11 wrapper until the `stream_arg+0x178` vtable path is mapped.
- Next narrow RE target:
  - trace where the object stored at `stream_arg+0x178` is assigned;
  - identify that object's vtable implementations for slots `+0x38`, `+0x58`, `+0x160`, and `+0x198`;
  - only then decide whether the iMac secondary should reach `StreamEnableWrite4F1AndPollSinkStatus()` or stay on the generic cached/finalizer route.

Extra RE continuation: `stream_arg+0x178` assignment found

- Follow-up result:
  - the assignment site for the object consumed by `UpdateMode11ImmediateStreamEnableFlag()` was found;
  - it is not directly the EDID parser chain.
- New/updated IDA anchors:
  - `0x141508510`: renamed `BuildStreamArgForDisplayIndex_SetObject178`;
  - `0x14150896D`: commented as the `stream_arg+0x178` assignment;
  - `0x1414D2748`: commented where stream-source programming forwards `stream_arg+0x178`;
  - `0x1414E7989`: commented where the state-1 stream-enable packet forwards `stream_arg+0x178`.
- `BuildStreamArgForDisplayIndex_SetObject178()` does this:
  - receives an output stream/mode argument buffer at `a4`;
  - resolves `v9` from a display-object provider/list lookup using display index `*(a3+0x28)`;
  - stores `v9` into `a4+0x178`;
  - then fills timing/mode/source fields and calls the downstream helpers that consume the same object.
- This explains why stream-enable uses `stream_arg+0x178` as a richer object:
  - mode gate: vfunc `+0x160`, compared to `11` and `12`;
  - cached stream-source packet: forwards the object as packet field 0;
  - state-1 stream-enable packet: forwards the object as packet field 0;
  - descriptor/capability path: vfunc `+0x58` returns the object used for pin/property lookups.
- Refined conclusion:
  - the mode-11 branch is a property of the display object selected for this stream/mode argument;
  - EDID/DisplayID still feeds that display object, TILE metadata, and capability tags, but the immediate `11/12` decision is one layer above the raw EDID parser.
- Next narrow RE target:
  - identify the concrete vtable(s) for the object returned by the provider/list lookup in `BuildStreamArgForDisplayIndex_SetObject178()`;
  - map that object's `+0x160` implementations and determine what field/state makes it return `11` or `12`;
  - then re-evaluate whether the iMac secondary is supposed to use the mode-11 `0x4F1` wrapper.

Extra RE continuation: topology display object owns the mode-gate value

- Follow-up result:
  - the provider behind `stream_arg+0x178` has now been traced one layer further;
  - `ConstructDal2CoreAndServices()` creates `DisplayTopologyManager` through `CreateDisplayTopologyManagerInterface()` and stores the returned interface at `core+0x60`;
  - the DS/dispatch init bundle copies `core+0x60` into the provider slot later used by `BuildStreamArgForDisplayIndex_SetObject178()`;
  - therefore the object stored at `stream_arg+0x178` is selected by the topology manager's display-index lookup, not by an EDID parser directly.
- New/updated IDA anchors in the modern BootCamp IDB:
  - `0x14143B930`: renamed `CreateDisplayTopologyManagerInterface`;
  - `0x141438F50`: renamed `TopologyManager_GetDisplayObjectByIndex`;
  - `0x1414B7D70`: renamed `TopologyPathBuilder_GetDisplayObjectByIndex`;
  - `0x141549190`: renamed `DisplayObject_GetCommittedSignalTypeForModeGate`;
  - `0x141549120`: renamed `DisplayObject_GetPendingSignalType`;
  - `0x14154A9C0`: renamed `DisplayObject_CommitPendingStreamFields`;
  - `0x141547CB0`: renamed `DisplayObject_SetSignal12ReportsAs11Flag`;
  - `0x14154AD10`: renamed `DisplayObject_AddPathInterface`;
  - `0x14154AD70`: renamed `DisplayObject_ValidateAndSetRequestedSignalType`;
  - comments added at `0x14142A1B6`, `0x14143B14D`, `0x14143B1D2`, `0x141438F50`, `0x141549190`, `0x141547CB0`, `0x14154A9C0`, `0x14154AD70`, `0x14154144D`, `0x14154145A`, and `0x1415436F3`.
- Concrete topology-manager path:
  - `TopologyManager_GetDisplayObjectByIndex()` is vtable slot `0` on the topology-manager interface returned as `outer+0x28`;
  - it checks `index < outer+0xA4`;
  - it returns `outer+0xC0[index]`;
  - `ConstructDisplayTopologyManager()` allocates this display-object array and fills it from the topology display-path builder.
- Concrete display-object mode gate:
  - full display objects are created by `CreateFullDisplayObjectInterface()` / `ConstructFullDisplayObject()`;
  - their interface vtable is `off_141FAB4E8`;
  - vtable slot `+0x160` is now `DisplayObject_GetCommittedSignalTypeForModeGate()`;
  - this function reads the committed per-stream signal type at stream-record field `+0x48`;
  - if display-object byte `+0x1E9` is set and the committed signal type is `12`, it reports `11` instead;
  - otherwise it returns the committed signal type unchanged.
- Important stream-record detail:
  - `DisplayObject_GetPendingSignalType()` reads pending stream-record field `+0x40`;
  - `DisplayObject_CommitPendingStreamFields()` copies pending fields to committed fields:
    - `+0x3C -> +0x44`;
    - `+0x40 -> +0x48`;
    - `+0x20 -> +0x28`;
  - the mode gate reads the committed `+0x48` value after this snapshot.
- Signal-type mapping observed in the packet path:
  - `sub_141543610()` dispatches packet `0xD00040` into `sub_141542940()` / `sub_141540860()`;
  - inside `sub_141540860()`, packet case `7` maps subcase `11` to signal type `11`;
  - the same packet case maps subcase `4` to signal type `12`;
  - this is a higher-level display/mode programming route that uses the same `11`/`12` vocabulary; the exact final writer into pending stream-record field `+0x40` is still tracked separately below.
- Refined conclusion:
  - the "does this stream use the mode-11 immediate `0x4F1` wrapper?" decision is currently best described as:
    - topology manager chooses display object by display index;
    - display object has a pending signal type;
    - pending signal type is committed into stream-record field `+0x48`;
    - mode gate reads committed `+0x48`, with an optional `12 -> 11` coercion flag.
  - EDID/DisplayID is still upstream evidence for tile topology and capabilities, but the exact `11` versus `12` decision is a display-object stream state, not a raw EDID-parser kind return.
- Why this matters for our Windows/BootCamp question:
  - Windows can plausibly start from a firmware state that is not already equivalent to OCLP's 5K handoff and still build a two-stream/tiled display object in the driver;
  - the driver has an explicit topology/path construction layer and a later signal-type programming path that can produce signal type `11` or `12`;
  - this supports the idea that BootCamp can revive or construct the 5K path in driver space, rather than requiring the exact OCLP firmware branch to have already happened.
- Remaining narrow RE questions:
  - identify the writer that sets pending stream-record field `+0x40` for the iMac internal paths;
  - identify who calls `DisplayObject_SetSignal12ReportsAs11Flag()` and under what platform/display condition;
  - decide whether the iMac 5K pair should normally report committed signal `11`, committed signal `12`, or signal `12` with the coercion flag set.

Extra RE continuation: topology pass applies requested signal types 11 and 12

- Follow-up result:
  - one more topology-manager pass found an explicit writer for requested signal type `11`/`12`, but not yet the final pending stream-record field read by the mode gate;
  - this is a useful upstream signal-type assignment, not yet the last writer of committed stream-record field `+0x48`.
- New/updated IDA anchors:
  - `0x1414A6690`: renamed `TopologyManager_AssignDerivedDisplaySignalTypes`;
  - `0x141549360`: renamed `DisplayObject_ApplyRequestedSignalTypeToStreams`;
  - `0x14149FB60`: renamed `DeriveDisplayObjectFlag416FromSignalAndConfunctionalCount`;
  - comments added at `0x1414A6839`, `0x1414A690B`, `0x1414A6AFD`, `0x1414A6BCF`, `0x141549360`, and `0x14149FB60`.
- What `TopologyManager_AssignDerivedDisplaySignalTypes()` does:
  - iterates topology display objects from the `outer+0xC0` array;
  - reads each display object's current pending signal type through vfunc `+0x288`;
  - derives a field stored by display-object vfunc `+0x438`;
  - reads display-object flags through vfunc `+0x60`;
  - if flags bit6 is set, first pass calls display-object vfunc `+0x508` with requested signal type `12`;
  - a later pass, with the caller condition forced false, calls the same vfunc `+0x508` with requested signal type `11` when the same flags bit6 is set.
- What `DisplayObject_ApplyRequestedSignalTypeToStreams()` does:
  - validates the requested type against the display object's path capabilities;
  - calls the same helper used by `DisplayObject_ValidateAndSetRequestedSignalType()`;
  - writes per-stream requested/derived type fields (`+0x64` / `+0x68` relative to the stream-record base in the outer object view);
  - it does not directly write the pending field `+0x40` or committed field `+0x48` that `DisplayObject_GetCommittedSignalTypeForModeGate()` reads.
- Current status:
  - the explicit `11`/`12` requested-signal assignment strongly links topology/confunctional path construction to the later mode-gate vocabulary;
  - a later pass identified the pending-field writer: `DisplayObject_SetPendingSignalTypeForAllStreams()` writes outer `+0x68`, equivalent to full display interface `+0x40`;
  - `DisplayObject_CommitPendingStreamFields()` then snapshots interface `+0x40` into committed interface `+0x48`, which is the field read by the mode gate.

Extra RE continuation: BootCamp SetMode owns an MST-to-SST coercion path

- Follow-up result:
  - the modern BootCamp driver has an explicit `ApplyMSTtoSSTOptimization` path inside `DSDispatch::SetMode`;
  - this path toggles the same display-object byte that makes committed signal type `12` (`DisplayPortMst`) report as signal type `11` (`DisplayPort`) to the downstream mode gate;
  - this is now the best explanation for how Windows reaches the mode-11/4F1 branch without requiring the firmware to have made the exact same OCLP-style handoff.
- New/updated IDA anchors:
  - `0x1415001F0`: renamed `SetMode_ApplyMstToSstOptimizationSignalCoercion`;
  - `0x141500070`: renamed `SetMode_FindMstToSstOptimizationAnchorDisplay`;
  - `0x1415004A0`: renamed `SetMode_AllDisplaysHaveAtMostTwoTiles`;
  - `0x14150F0C0`: renamed `DSDispatch_SetMode`;
  - `0x14150E2F0`: renamed `DSDispatch_BuildApplySetModePaths`;
  - `0x1415475D0`: renamed `DisplayObject_SetPendingSignalTypeForAllStreams`;
  - `0x141547670`: renamed `DisplayObject_ValidateRequestedSignalTypeAcrossStreams`;
  - `0x141547730`: renamed `DisplayObject_DeriveCompatibleSignalTypeForStream`;
  - `0x14144B670`: renamed `SetModePathList_GetEntryByOrdinal`;
  - `0x14144B6B0`: renamed `SetModePathList_GetCount`;
  - `0x14144B5C0`: renamed `SetModePathList_FindEntryByDisplayIndex`;
  - `0x14150FDB0`: renamed `DSDispatch_LogSetModePathList`.
- Concrete SetMode chain:
  - `DSDispatch_SetMode()` logs `All paths will be disabled from ApplyMSTtoSSTOptimization` when `SetMode_ApplyMstToSstOptimizationSignalCoercion()` returns true;
  - the helper first gates the path list with `SetMode_AllDisplaysHaveAtMostTwoTiles()`;
  - it then finds an anchor display with `SetMode_FindMstToSstOptimizationAnchorDisplay()`;
  - for each display in the SetMode path list, it resolves the topology display object through the provider at `a1+0x58`;
  - if the anchor path metadata flag at timing/path payload `+0x1C` bit 7 is clear, it calls display-object vfunc `+0x318(1)`;
  - vfunc `+0x318(1)` sets the `12 -> 11` reporting/coercion flag at display interface `+0x1E9`;
  - if that bit is set, the helper clears the same flag with vfunc `+0x318(0)`.
- Concrete pending-field chain:
  - `DisplayObject_SetPendingSignalTypeForAllStreams()` uses the outer display-object base, not the interface base;
  - it writes the requested type to outer `+0x68`, which is full interface `+0x40`;
  - it writes the derived compatible type to outer `+0x64`, which is full interface `+0x3C`;
  - `DisplayObject_CommitPendingStreamFields()` commits `+0x3C -> +0x44`, `+0x40 -> +0x48`, and `+0x20 -> +0x28`;
  - `DisplayObject_GetCommittedSignalTypeForModeGate()` reads committed `+0x48` and optionally reports `12` as `11`.
- Detection interaction:
  - `HandleDisplayDetectionResultAndLiveState()` clears the `12 -> 11` coercion flag unless detection result byte `+0x72` is set;
  - `DSDispatch::SetMode` can set it again later based on the selected path list;
  - this explains why detection and SetMode can appear to disagree: detection normalizes live state, SetMode applies the MST-to-SST optimization for the actual mode request.

Extra RE continuation: Windows 5K is a layered driver recipe, not a single firmware call

- Follow-up result:
  - the modern driver still contains named 5K/tile policy strings and functions:
    - `DalEnable5kTiledMode`;
    - `Dal5K60PipeSplit`;
    - `WindowsDM_EnableSecondaryTileIfRequired`;
    - `ModeManagerAddDerivedModesIncluding5kTile`;
    - `CreateModeSolutionWith5KPipeSplitOptionFlags`;
    - `MarkTimingPipeSplitForHighResAnd5k60`.
- Mode-manager layer:
  - `ConstructDalModeManagerWith5kTiledOption()` reads `DalEnable5kTiledMode` into mode-manager byte `+0x80`;
  - `ModeManagerAddDerivedModesIncluding5kTile()` injects a real `2560x2880` half-tile mode from a `1920x2160` base timing only when the 5K gates pass;
  - the final `2560x2880` injection is gated by mode-manager flag bit 5 and `DalEnable5kTiledMode`;
  - `MarkTimingPipeSplitForHighResAnd5k60()` sets timing flag bit 7 when timing aspect matches display geometry and marks high-res/5K timings as pipe-split when `DalSupportMultiPipe` allows it;
  - `CreateModeSolutionWith5KPipeSplitOptionFlags()` folds mode-manager flag bit 9, derived from `Dal5K60PipeSplit`, into the mode-solution constructor flags.
- Detection/runtime layer:
  - `DetectMstDisplayMaybeAssert4F1()` can recover an MST display object, apply requested signal type `12` through display-object vfunc `+0x508`, or reuse an existing display entry and report signal type `11`;
  - `DetectDisplayAndApplyApple5kOverrides()` preserves an Apple/tiled false-disconnect case when the descriptor tag bit is present and the signal type is `11`;
  - this points to Windows keeping a real display object alive rather than replacing the second tile with a fake sink.
- DPCD/latch layer:
  - `WindowsDM_EnableSecondaryTileIfRequired()` emits DPCD `0x4F1 = 1` only for signal/display type `128` and pin-descriptor bits `+6[2]` or `+6[3]`;
  - `BringUpDisplayBySignalType()` has a separate post-bring-up route: after successful link/stream bring-up, lower display object flag `+0x5B4` can trigger `SendSecondaryTileOpcode1265Payload1()` with a 100 ms settle;
  - `ParseEdidAndPinCapsIntoDcSinkInfo()` copies pin property tag `0x3B`'s value field into parsed sink-info `+0xAC`; when the caller output is `dc_sink+0x508`, this becomes `dc_sink+0x5B4`;
  - therefore `dc_sink+0x5B4` is not a mysterious firmware flag. It is the value of tag `0x3B`, while several other `0x4F1` writers gate only on tag/descriptor-bit presence;
  - `SendSecondaryTileOpcode1265Payload1()` writes DPCD address `1265` / `0x4F1` payload `1`, retries once after 10 ms, and optionally sleeps 100 ms after success.
- Current interpretation:
  - Windows does not appear to require an EFI runtime service or Apple GOP call after the OS driver loads;
  - the visible Windows mechanism is driver-owned: preserve or recreate real route objects, derive a per-tile 5K mode, choose a 5K pipe-split/view solution, SetMode the real paths, and assert `0x4F1` only after the relevant AUX/link object exists;
  - this supports the "Windows can revive/build 5K from a plain-ish state" theory more than the "Windows simply inherits an OCLP-like full 5K firmware state" theory.
- Remaining narrow question:
  - for the iMac19,1 `APP/AE26` style route, the static tag `0x3B` value appears to be `0`, so the value-gated post-bring-up `dc_sink+0x5B4` path may not be the important one;
  - the next useful RE target is which presence-gated `0x4F1` writer actually runs for the Apple 5K internal panel: detect/status, WindowsDM type-128 helper, or stream-enable/post-enable path.

Extra RE continuation: classify the presence-gated `0x4F1` writers

- Follow-up result:
  - the candidate `0x4F1` writers are not equivalent;
  - they use different descriptor bits, different AUX objects, and different ordering relative to link/stream enable.
- Writer 1: detect/status-time path:
  - `ProbeDisplayStatusMaybeAssert4F1()` and `DetectDisplayStatusMaybeAssert4F1()` both call `DetectDisplayMaybeAssert4F1ForCapableSink()`;
  - `DetectDisplayMaybeAssert4F1ForCapableSink()` checks connected-pin descriptor byte `+6` bit 2 only;
  - this corresponds to tag `0x3A` presence;
  - it writes DPCD `0x4F1 = 1` through `WriteSinkDpcd4F1Payload1Retry()`;
  - `WriteSinkDpcd4F1Payload1Retry()` resolves the display object id, finds the display-map entry, and uses display-map entry `+0x18` as the AUX/DPCD interface;
  - therefore this writer is early/detect-side and is not proven to cover an AE26/tag-`0x3B`-only secondary path.
- Writer 2: `WindowsDM_EnableSecondaryTileIfRequired()`:
  - checks connected-pin descriptor byte `+6` bit 2 or bit 3;
  - therefore it accepts tag `0x3A` or tag `0x3B`;
  - it still requires the selected link/display type to be `128`;
  - then it calls `SendSecondaryTileOpcode1265Payload1()`;
  - this remains a root/eDP-style helper, not proof that the internal secondary `0x3113` route should use this exact entry point.
- Writer 3: post-bring-up `BringUpDisplayBySignalType()`:
  - runs after signal-type-specific bring-up succeeds;
  - checks lower display object `dc_sink+0x5B4`;
  - `dc_sink+0x5B4` is copied from tag `0x3B`'s value field;
  - for the static AE26-style record, tag `0x3B` is present but value appears to be `0`, so this value-gated helper may not run for our panel.
- Writer 4: stream-enable lifecycle path:
  - `EnableValidatedStreamMaybeAssert4F1()` reaches `StreamEnableWrite4F1AndPollSinkStatus()` only when `UpdateMode11ImmediateStreamEnableFlag()` sets the link-service immediate flag from committed signal type `11`;
  - the committed type can be naturally `11`, or `12` coerced to `11` by `SetMode_ApplyMstToSstOptimizationSignalCoercion()`;
  - `StreamEnableWrite4F1AndPollSinkStatus()` accepts connected-pin descriptor byte `+6` bit 2 or bit 3, so it accepts tag `0x3A` or `0x3B`;
  - the write happens on the non-state-1 normal stream-enable branch after link/stream programming and the real stream-enable vfunc call;
  - it writes DPCD `0x4F1 = 1` through the link-service AUX object at `this+0x90`, retries once after 10 ms, then transitions the stream-enable state to `2`;
  - this is currently the strongest Windows analogue for the AE26/tag-`0x3B` internal secondary route.
- Updated interpretation:
  - for Linux plain/probe boot work, the most faithful target is not an unconditional early primary `0x4F1` pulse;
  - it is preserving or creating the real secondary AUX/link route, reaching a mode-11-equivalent stream-enable path, programming the stream, then asserting `0x4F1` on the real secondary AUX at the same lifecycle point Windows uses.

Extra RE continuation: assumptions checked against IDA

- Scope:
  - this was a confirmation pass on the modern BootCamp `amdkmdag.sys` IDB at `B416406/amdkmdag.sys.i64`;
  - no kernel changes were made in this pass;
  - IDA names/comments were refreshed for the capability-table, SetMode coercion, and `0x4F1` writer functions.
- Confirmed: Apple panel capability provenance:
  - static row `0x144C50070` bytes decode as manufacturer `0x1006`, product `0xAE25`, source id `0x38`, value `0`;
  - static row `0x144C50080` bytes decode as manufacturer `0x1006`, product `0xAE26`, source id `0x39`, value `0`;
  - `MapCapabilityTableIdToPinPropertyTag()` maps source id `0x38` to pin property tag `0x3A`;
  - the same mapper maps source id `0x39` to pin property tag `0x3B`;
  - therefore `APP/AE25 -> tag 0x3A` and `APP/AE26 -> tag 0x3B` are no longer just notes-derived assumptions.
- Confirmed: presence bit and value field must be kept separate:
  - `ApplyVendorProductCapabilityTableToPin()` inserts the mapped pin property and sets the corresponding descriptor bit;
  - `ApplyPinCapabilityEdidFixups()` dispatches both tag `0x3A` and tag `0x3B` to `DoubleEdidPhysicalWidthForTiledCaps()`;
  - `ParseEdidAndPinCapsIntoDcSinkInfo()` copies tag `0x3B`'s value field into parsed sink info `+0xAC`, which becomes `dc_sink+0x5B4` for the known caller;
  - since the static AE26 row value is `0`, the `dc_sink+0x5B4` post-bring-up helper is weaker evidence than the descriptor-bit-gated writers.
- Confirmed: writer split:
  - `DetectDisplayMaybeAssert4F1ForCapableSink()` only checks descriptor byte `+6` bit 2, matching tag `0x3A`;
  - `WindowsDM_EnableSecondaryTileIfRequired()` accepts bit 2 or bit 3, but still requires selected link/display type `128`;
  - `StreamEnableWrite4F1AndPollSinkStatus()` accepts bit 2 or bit 3 and writes DPCD `0x4F1 = 1` through the link-service AUX object at `this+0x90`;
  - the stream-enable writer remains the strongest static match for an AE26/tag-`0x3B` secondary tile path.
- Confirmed: mode-gate bridge:
  - `StreamEnableWrite4F1AndPollSinkStatus()` has one direct code xref, from `EnableValidatedStreamMaybeAssert4F1()`;
  - that wrapper calls the stream-enable writer only after `UpdateMode11ImmediateStreamEnableFlag()` sets the immediate flag from reported signal type `11`;
  - `DSDispatch_SetMode()` calls `SetMode_ApplyMstToSstOptimizationSignalCoercion()`;
  - the SetMode helper toggles `DisplayObject_SetSignal12ReportsAs11Flag()`, and `DisplayObject_GetCommittedSignalTypeForModeGate()` then reports committed signal type `12` as `11`;
  - this supports the interpretation that Windows reaches the mode-11/`0x4F1` branch through normal SetMode/display-object state, not through a hidden EFI runtime callback.
- Still inferred:
  - static RE still cannot prove which branch executes on a live iMac19,1 without runtime trace;
  - the best current assumption is that the Windows path either reaches the normal mode-11 stream-enable writer for AE26, or reaches a cached/state-1 analogue where absence of a late `0x4F1` write is not itself a mismatch;
  - the Linux target should continue to be route-preservation plus Windows-shaped stream/link finalization on the real secondary `0x3113` AUX route, with logs distinguishing state-1/cached behavior from the normal writer path.

Extra RE continuation: probable Windows SetMode and stream-commit recipe

- Scope:
  - this pass tried to reduce the "probably" around what Windows does once SetPathMode/SetMode owns the panel;
  - the focus was not another `0x4F1` writer search, but the path-state machinery that decides whether Windows uses normal stream-enable, cached/state-1 stream-enable, or the MST-to-SST optimization path.
- Confirmed: `ApplyMSTtoSSTOptimization` is a real path-state transformation:
  - `SetMode_AllDisplaysHaveAtMostTwoTiles()` rejects only displays whose descriptor reports more than two tiles, so an Apple dual-tile 5K panel passes this gate;
  - `SetMode_FindMstToSstOptimizationAnchorDisplay()` returns an anchor display index; for multi-path lists it delegates anchor selection to the topology/optimization manager at `a1+0x38` vfunc `+0x40`;
  - `DSDispatch_SetMode()` calls `SetMode_ApplyMstToSstOptimizationSignalCoercion()` in the pre-pass;
  - `DSDispatch_BuildApplySetModePaths()` calls the same helper again while constructing live path-state records;
  - when the anchor exists and anchor timing/path metadata `+0x1C` bit 7 is clear, non-anchor paths are marked with skip-enable / force-disable bits `0x100000` and `0x80000`;
  - when the helper returns true, all built path-state records also receive the `0x80000` optimization/force-disable metadata bit.
- Interpretation:
  - Windows is probably not simply enabling two independent tile streams in the most direct way;
  - it has an explicit "MST-to-SST optimization" layer that can coerce a DisplayPortMST-style object to report as DisplayPort mode `11`, then suppress at least some non-anchor enable work;
  - this makes the mode-11 `0x4F1` writer path less mysterious: Windows can reach it from SetMode-selected display-object state even if the hardware topology originally looked MST/tiled.
- Confirmed: SetPathMode still builds real stream objects and a real commit context:
  - `WindowsDM_SetPathMode_BuildStreamsAndCommit()` allocates a candidate DC stream with `AllocateDcStreamForDisplayObject()`;
  - it populates the stream through `BuildDcStreamFromSelectedModeObject()`;
  - it then writes the new stream to `display_object+0x38` and calls `AddDcStreamToCommitContextArray()`;
  - final commit goes through `DcCommitStreams_BuildValidateAndCommit()`;
  - the parallel `DalWDDM_SetPathMode_BuildStreamsAndCommit()` path follows the same shape.
- Confirmed: peer/tile streams are actively rebalanced, not ignored:
  - after building a stream whose type field `stream+0x3D4` is `64`, SetPathMode calls `RebalancePeerStreamsAfterNewStreamBuild()`;
  - that helper finds peer display objects whose live stream points at the same lower object and compatible peer state;
  - it can clone/rebalance stream state, write the peer stream back to peer `display_object+0x38`, and add that peer stream to the same DC commit context;
  - this is another clue that Windows owns a real paired-display/peer-stream model and is not relying on a post-boot firmware service to do the display composition.
- Confirmed: state-1 reuse is strict trained-link reuse:
  - `SetState1IfDpcdReportsTrainedLink()` calls `RecoverActiveDpLinkSettingsFromDpcd()`;
  - that reads DPCD `0x100/0x101`, `0x003`, and lane-status bytes `0x202/0x203`;
  - it refuses to set state `1` unless recovered lane count and link rate are nonzero and lane-status bits look trained/locked;
  - therefore Windows state-1 is not just "cached tuple exists"; it is "the link is already trained according to live DPCD".
- Confirmed: mode-11 normal enable chooses link settings from stream demand:
  - `SelectLinkSettingsRespectMode11Flag()` switches to `SelectFittingLinkSettingsForMode11Stream()` only when the mode-11 immediate flag is set;
  - the fitting helper starts from the required stream payload/bandwidth, checks explicit link-setting override state if present, then walks available link tuples until one has enough capacity;
  - this supports treating the normal mode-11 branch as a real Windows training/enable path, not a blind HBR2/default tuple replay.
- Confirmed: the non-immediate cached path is not a no-op:
  - `ProgramCachedLinkStreamSourceThenEnableStream()` calls `ProgramPeerSourceUpdateBeforeStreamEnable()`;
  - it calls `ProgramStreamSourceWithLinkSettings()` using the cached tuple at link-service `+0x8C`;
  - it marks the display object enabled through vfunc `+0x38`;
  - it then calls the real stream-enable finalizer vfunc `+0x98`;
  - mapped DP finalizers can program source-side DPCD such as `0x170`, `0x107`, and `0x316` depending on branch and stream fields.
- Updated working model:
  - if Windows inherits a trained secondary link, it can use strict state-1 reuse and avoid destructive retraining;
  - if it does not inherit a trained secondary link, the mode-11 normal path has enough machinery to pick link settings, train/enable, program source-side DPCD, and finally assert tag-gated `0x4F1`;
  - the MST-to-SST optimization can make the selected display object report mode `11` and can suppress non-anchor enable work, so Linux should be careful not to model Windows as "enable both tile paths independently, then latch";
  - a closer Linux analogue may be: preserve or recreate the peer/secondary route, use one selected/anchor path to drive the mode-11 stream-enable decision, keep peer stream metadata coherent, and only then run the real secondary AUX/source finalization.
- Still unknown:
  - the exact runtime value of the anchor path's `+0x1C` bit 7 on iMac19,1;
  - whether a plain Windows BootCamp boot reaches state-1 reuse from firmware-trained DPCD or takes the normal mode-11 training branch;
  - whether the non-anchor skip-enable flags mean "do not enable the second tile at all" or "do not independently enable it because peer/anchor machinery will cover it";
  - these unknowns are now narrow enough that a Windows runtime trace of SetMode logs/DPCD would answer them, but static RE already points away from a hidden EFI runtime call.

Extra RE continuation: SetMode skip/disable bits are consumed for real

- Scope:
  - this pass followed the `0x80000` and `0x100000` path-state bits after `DSDispatch_BuildApplySetModePaths()` sets them;
  - this was meant to answer whether the MST-to-SST optimization bits are just metadata or whether they change the later enable/disable sequence.
- New/updated IDA anchors:
  - `0x141502C40`: renamed `DSDispatch_BlankDisableSetModePaths`;
  - `0x1415022D0`: renamed `DSDispatch_EnableLinksAndUnblankSetModePaths`;
  - `0x14150AB40`: renamed `DSDispatch_RunFullSetModeTransition`;
  - `0x14150BCB0`: renamed `DSDispatch_ApplySetModeTransition`;
  - `0x1414D9040`: `PerformLinkTrainingWithAssrPatternPolicy`.
- Confirmed: bit `0x80000` / bit 19 forces the blank/disable phase:
  - `DSDispatch_BlankDisableSetModePaths()` computes `DisableOutput` as existing bit 3 OR bit 19;
  - therefore the `0x80000` bit set by `ApplyMSTtoSSTOptimization` is not cosmetic;
  - it causes the path to enter the blank/disable handling before the lower SetMode apply step.
- Confirmed: bit `0x100000` / bit 20 is a real `SkipEnable` gate:
  - `DSDispatch_EnableLinksAndUnblankSetModePaths()` only performs `EnableLink` / `ChangeMode` / `UnblankStream` when bit 20 is clear;
  - if bit 20 is set, the function logs `SkipEnable` for that display path;
  - therefore the non-anchor path marked by the MST-to-SST optimization is not re-enabled through the ordinary enable loop.
- Confirmed: call order:
  - `DSDispatch_RunFullSetModeTransition()` calls `DSDispatch_BlankDisableSetModePaths()`;
  - it waits 100 ms;
  - it invokes the lower SetMode apply path;
  - it then calls `DSDispatch_EnableLinksAndUnblankSetModePaths()`;
  - the other transition helper, `DSDispatch_ApplySetModeTransition()`, also calls blank/disable first, but calls enable/unblank only when requested by caller or by path-state bit 22.
- Timing flag clarification:
  - `MarkTimingPipeSplitForHighResAnd5k60()` sets timing record dword 7 bit 7 when the timing aspect matches the display geometry;
  - the SetMode MST-to-SST helper tests this same bit through the path timing pointer at `+0x18`, dword `+0x1C`;
  - therefore the anchor bit is tied to selected timing / pipe-split eligibility, not a bootloader or firmware-mode flag.
- Link-training clarification:
  - normal mode-11 enable reaches `TryEnableLinkWithHbr2Fallback()`;
  - if requested link settings match the cached tuple, it can skip fresh training;
  - if they do not match, it calls `ProgramAssrThenTrainLinkSettings()`;
  - that sequence programs requested link settings/cache, classifies ASSR policy, writes DPCD `0x10A`, and enters link training;
  - `PerformLinkTrainingWithAssrPatternPolicy()` writes DPCD `0x100/0x101` as a two-byte transfer, then writes DPCD `0x107`, then performs the lane-training sequence.
- Updated working model:
  - Windows' MST-to-SST optimization deliberately disables/blankets outputs first and can skip the ordinary re-enable of the non-anchor path;
  - the second tile is therefore likely kept coherent through peer-stream / selected-stream state and lower DC commit machinery, not through a symmetric "enable both paths independently" loop;
  - for Linux, this makes a blind "force both tile streams through the same late link-enable path" look less Windows-shaped;
  - the better target remains preserving/recreating the real secondary route, ensuring source/link DPCD setup happens while AUX is alive, and avoiding destructive independent re-enable/retrain behavior when the route is already part of a paired optimized commit.
- Still unknown:
  - the exact anchor selected on live iMac19,1 Windows;
  - whether the skipped non-anchor path still receives all necessary source-side DPCD through peer-stream rebalance/lower commit callbacks;
  - whether plain BootCamp starts with DPCD lane status trained enough for strict state-1 or has to run the normal mode-11 training branch.

Extra RE continuation: skipped secondary still enters the lower SetMode/sync machinery

- Scope:
  - this pass focused on the non-anchor/secondary tile after `ApplyMSTtoSSTOptimization` marks it with bit `0x100000` / `SkipEnable`;
  - the target question was whether Windows drops that path entirely, or whether it remains in the commit/synchronization list and only skips the later DS-level enable loop.
- New/updated IDA anchors:
  - `0x141508E90`: renamed `BuildSetModeStreamArgListForPaths`;
  - `0x1414D5470`: renamed `AllocateSetModeStreamArgList`;
  - `0x1414D5370`: renamed `SetModeStreamArgList_Append`;
  - `0x1414D55C0`: renamed `SetModeStreamArgList_Get`;
  - `0x1414D5600`: renamed `SetModeStreamArgList_Count`;
  - `0x1414FCEF0`: renamed `SyncManager_ApplySetModeSynchronizationState`;
  - `0x1414F98C0`: renamed `SyncManager_ApplyGLSyncSynchronization`;
  - `0x1414FBB20`: renamed `SyncManager_ApplyInterPathSynchronization`;
  - `0x1414FB7E0`: renamed `SyncManager_FindInterPathPendingTimingServer`;
  - `0x1414FC570`: renamed `SyncManager_SetupInterPathSynchronization`;
  - `0x1414FEA50`: renamed `SyncManager_SetupTimingSynchronization`;
  - `0x1414F8560`: renamed `SyncManager_ValidateTimingSyncRequest`.
- Confirmed: the stream-arg list keeps every SetMode path:
  - `AllocateSetModeStreamArgList()` allocates a list with room for six entries;
  - `SetModeStreamArgList_Append()` copies each `0x298`-byte stream/mode argument into that list;
  - `BuildSetModeStreamArgListForPaths()` loops over the SetMode path count and appends one stream/mode argument per path;
  - each argument stores the resolved display object at `stream_arg+0x178`, then fills view/timing/source fields from the selected path.
- Confirmed: the full transition submits this list before the later skip-enable loop:
  - `DSDispatch_RunFullSetModeTransition()` builds the full stream-arg list;
  - it marks each stream-arg as active/change-mode style work, then calls `SyncManager_ApplySetModeSynchronizationState()` on the full list;
  - it runs `DSDispatch_BlankDisableSetModePaths()`;
  - it waits 100 ms;
  - it invokes the lower SetMode apply vfunc with the full stream-arg list;
  - only after that does it call `DSDispatch_EnableLinksAndUnblankSetModePaths()`, where bit `0x100000` can produce `SkipEnable` for the non-anchor path.
- Confirmed: the alternate transition path has the same shape:
  - `DSDispatch_ApplySetModeTransition()` also builds a full stream-arg list;
  - it runs the SyncManager pass, blank/disable, then submits the full list through a lower apply vfunc;
  - only after the lower apply does it optionally run the DS-level enable/unblank loop.
- Confirmed: inter-path synchronization is real state, not just logging:
  - `SyncManager_SetupTimingSynchronization()` validates requested timing sync and dispatches inter-path requests to `SyncManager_SetupInterPathSynchronization()`;
  - `SyncManager_SetupInterPathSynchronization()` copies the requested sync source/target tuple into a per-display sync-state record and marks it active/pending;
  - `SyncManager_FindInterPathPendingTimingServer()` can promote one pending inter-path client to a local timing server and point peer clients at it;
  - `SyncManager_ApplyInterPathSynchronization()` then emits `HWSyncRequest_Set_InterPath` for matching source/target pairs.
- Interpretation:
  - Windows is not simply discarding the non-anchor tile when it marks it `SkipEnable`;
  - the non-anchor still has a stream/mode argument, still participates in synchronization setup, and is still present when the lower mode-set apply is called;
  - `SkipEnable` appears to suppress the later DS-owned `EnableLink` / `ChangeMode` / `UnblankStream` loop, not the earlier lower commit/sync machinery;
  - for Linux this argues against a symmetric "late-enable both tiles independently" model;
  - a closer Windows-shaped bring-up should keep both tile paths in the atomic/commit state, preserve the secondary route/AUX object, apply paired timing/source metadata, and avoid retraining or re-enabling the secondary through a separate late path if the lower paired commit already owns it.
- Kernel-facing next question:
  - can we make Linux's tile-pair path keep the secondary stream/route alive while suppressing the destructive part of the independent secondary enable/retrain sequence;
  - if yes, the next kernel experiment should log and gate secondary link training/re-enable separately from secondary stream inclusion, because Windows treats those as different layers.

Extra RE continuation: lower apply resolves to HWSequence/HWSync, not hidden firmware

- Scope:
  - this pass followed the lower vfuncs that receive the complete stream-arg list before `DSDispatch_EnableLinksAndUnblankSetModePaths()` can skip the non-anchor path;
  - the goal was to answer whether this lower layer is another DP/firmware SetMode path or a Windows-owned synchronization/hardware-sequence layer.
- Corrected call target:
  - `DSDispatch_RunFullSetModeTransition()` calls `HWSequenceService` vfunc `+0x2B0`;
  - the concrete target for the HWSequence interface returned by `ConstructHWSequenceServiceForAsic()` is `HwSeq_ApplySetModeHWSyncStateList()`;
  - therefore the vfunc is not a hidden Apple firmware call and not directly a DPCD writer;
  - it is a Windows HWSequence/HWSync application pass over the full stream-arg list.
- New/updated IDA anchors:
  - `0x14155A050`: renamed `HwSeq_ApplySetModeHWSyncStateList`;
  - `0x14155A100`: renamed `HwSeq_BuildAffectedHWSyncPathMask`;
  - `0x14155D660`: renamed `HwSeq_RunHardwareSetModeAndApplyHWSync`;
  - `0x1415E2C60`: renamed `HWSync_ApplyInterPathPreWorkForSetModeList`;
  - `0x1415E2AD0`: renamed `HWSync_DispatchPerStreamSyncState`;
  - `0x1415E2D60`: renamed `HWSync_BuildAffectedPathMaskForSetModeList`;
  - `0x14156F180`: renamed `HWSync_AdjustDpMstEdpSourceSignalForInterPath`;
  - `0x14156FC40`: renamed `HWSync_ApplyModeMaskToEligibleStreams`;
  - `0x1415E0F00`: renamed `HWSync_StateMatchesModeMask`;
  - `0x1415E1F00`: renamed `HWSync_EnableGenlockForStream`;
  - `0x1415E1840`: renamed `HWSync_EnableGenlockTimingServerForStream`;
  - `0x1415E11A0`: renamed `HWSync_EnableFreerunForStream`;
  - `0x1415E10A0`: renamed `HWSync_DisableFreerunForStream`;
  - `0x14156F770`: renamed `HWSync_EnableShadowForStream`;
  - `0x14156FEE0`: renamed `HWSync_FinalizeSetModeSyncStateList`.
- Confirmed: full transition HWSync pass:
  - `HwSeq_ApplySetModeHWSyncStateList()` first calls `HWSync_ApplyInterPathPreWorkForSetModeList()` on the complete stream list;
  - it then calls `HWSync_DispatchPerStreamSyncState()`;
  - it finally calls the inner HWSync finalizer vfunc, which maps to `HWSync_FinalizeSetModeSyncStateList()`.
- Confirmed: hardware SetMode path also runs HWSync:
  - `DSDispatch_ApplySetModeTransition()` calls HWSequence vfunc `+0x30`;
  - the concrete target is `HwSeq_RunHardwareSetModeAndApplyHWSync()`;
  - this path does the main hardware set-mode sequence over the complete stream list;
  - near the end, it also calls `HWSync_ApplyInterPathPreWorkForSetModeList()` before returning to the DS-level optional enable/unblank loop.
- Confirmed: `stream_arg+0x168` / dword `+360` is the HWSync action state:
  - state `1` is inter-path sync;
  - state `2` dispatches to genlock;
  - state `3` dispatches to genlock timing-server;
  - state `4` dispatches to freerun;
  - state `5` dispatches to shadow;
  - state `6` dispatches to disable/reset.
- Confirmed: state `1` inter-path handling touches all paired entries:
  - `HWSync_ApplyInterPathPreWorkForSetModeList()` scans the full list for any state-`1` entry;
  - if any exists, it runs `HWSync_AdjustDpMstEdpSourceSignalForInterPath()`;
  - for state-`1` entries grouped by timing-server/source id, the helper finds DP-MST/eDP peers and can call a source-side vfunc at `+0xB0` / `176`;
  - when the sync group contains a DP-MST/eDP peer plus a non-DP peer with signal type `9..14`, it writes the peer signal type to the DP-MST/eDP source object;
  - when the group contains only DP-MST/eDP peers, it writes signal/type `8` to those source objects.
- Confirmed: state `1` then applies a mode mask:
  - after the source-signal adjustment, state-`1` prework calls `HWSync_ApplyModeMaskToEligibleStreams()` with mask `5`;
  - `HWSync_StateMatchesModeMask()` applies mask bit 0 to state `1`, so this pass covers all state-`1` inter-path entries;
  - this is separate from the later DS-level `EnableLink` / `ChangeMode` / `UnblankStream` loop where the non-anchor can still hit `SkipEnable`.
- Interpretation:
  - Windows' "skipped secondary" is not inert;
  - it still participates in a complete stream list, hardware set-mode/HWSync sequencing, inter-path timing grouping, and source-signal adjustment;
  - the later `SkipEnable` therefore means "do not run the ordinary DS per-path enable/unblank for this path", not "do not include this tile in paired timing/source setup";
  - this is the strongest static evidence so far that Linux should model tile-pair bring-up as a paired atomic commit plus selective suppression of destructive secondary late-enable/retrain work.
- Kernel-facing implication:
  - our current tile-pair failures may not be solved by one more DPCD wake byte alone;
  - Windows appears to establish a source/timing relationship between the two internal display paths before the skip-enable point;
  - the next Linux experiment should log whether the two CRTCs/connectors have an explicit timing-source / sync-source relationship at atomic commit time, and should separate "include secondary stream" from "train/re-enable secondary link".

Extra RE continuation: who creates state-1 inter-path HWSync

- Scope:
  - this pass followed the producer of `stream_arg+0x168 == 1`;
  - the goal was to answer whether state `1` is created by the HWSync layer itself, by a hidden firmware mailbox, or by earlier DS/SyncManager SetMode state.
- Confirmed writer:
  - state `1` is written into stream args by `SyncManager_ApplyInterPathSynchronization()` only after SyncManager already has accepted inter-path timing-sync records;
  - client stream args get `stream_arg+0x168 = 1` at `0x1414FBD15`;
  - the timing-server stream arg also gets `stream_arg+0x168 = 1` at `0x1414FBE4B` when at least one client is paired;
  - therefore HWSync consumes state `1`; it does not invent the pairing decision.
- New/updated IDA anchors:
  - `0x141506BD0`: renamed `DSDispatch_PrepareSetModeInterPathTimingSync`;
  - `0x141507350`: renamed `SetMode_TimingRecordsMatchForInterPathSync`;
  - `0x1414FE2E0`: renamed `SyncManager_GetTimingSyncStateForDisplay`;
  - `0x1414FE730`: renamed `SyncManager_ResetTimingSynchronizationForDisplay`;
  - `0x1414FC030`: renamed `SyncManager_ResetInterPathSynchronization`;
  - `0x1414FAD00`: renamed `SyncManager_ResetGLSyncSynchronization`;
  - `0x141501F30`: renamed `SetMode_TimingPayloadsDifferForMstSstOptimization`;
  - `0x141507F70`: renamed `SetMode_BuildTimingRecordForDisplay`.
- Confirmed request builder:
  - `DSDispatch_PrepareSetModeInterPathTimingSync()` scans the built SetMode path-state list;
  - path-state bit `0x8000` at `path_state+0x14` marks the timing-server/anchor path;
  - the first path with bit `0x8000` contributes its display index as the local timing server for later client requests;
  - before creating new requests it queries the existing SyncManager record with `SyncManager_GetTimingSyncStateForDisplay()`;
  - when it needs a fresh request, it first clears the old state through `SyncManager_ResetTimingSynchronizationForDisplay()`;
  - it then builds a request tuple with `request[0] = 1`, meaning inter-path timing sync;
  - `request[1] = 1` for the `0x8000` anchor path and `request[1] = 2` for client paths;
  - for a non-anchor path, `request[2] = 1` and `request[3] = anchor_display_index`, matching the `SyncSrcTgtType_LocalTimingServer` validation path already seen in `SyncManager_ValidateTimingSyncRequest()`;
  - if `SyncManager_SetupTimingSynchronization()` returns `3`, the path-state bit `0x20` is set to remember that accepted sync work exists for this path.
- Confirmed compatibility gate:
  - `SetMode_TimingRecordsMatchForInterPathSync()` compares the selected timing records before inter-path setup is kept or created;
  - the check requires matching timing dimensions and matching timing flag bits `0`, `1`, `2..5`, `6`, `7`, and `9`;
  - this means Windows is not pairing arbitrary outputs just because two paths exist;
  - it only creates the inter-path timing relationship when the selected modes/timing payloads are compatible enough to be phase-linked.
- Confirmed anchor creation:
  - `DSDispatch_BuildApplySetModePaths()` sets path-state bit `0x8000` while building the SetMode path-state list;
  - its preferred anchor is the first path whose lower display/descriptor object reports both vfunc `+0x2A8` and vfunc `+0x238` as true;
  - if no path passes that descriptor capability test, it falls back to marking the first built path with bit `0x8000`;
  - this prevents the later inter-path builder from running without any timing-server candidate.
- Confirmed pipe-split connection:
  - the same descriptor vfunc `+0x2A8` appears in `ModeListContainsPipeSplitTiming()`;
  - there it gates the scan that calls `MarkTimingPipeSplitForHighResAnd5k60()`;
  - `MarkTimingPipeSplitForHighResAnd5k60()` requires DAL option `0x587` / `DalSupportMultiPipe` before it marks timings as pipe-split;
  - the 5120x2880@60-style branch sets `timing_record[9]` low bits to `2`;
  - `ModeListContainsPipeSplitTiming()` treats `timing_record[9] & 3 > 1` as evidence that the mode list contains a pipe-split timing;
  - this makes vfunc `+0x2A8` look like a high-res / pipe-split-capable display signal, although I have not assigned a final method name to it.
- Confirmed MST-to-SST interaction:
  - the same SetMode build path calls `SetMode_ApplyMstToSstOptimizationSignalCoercion()` before path-state construction;
  - the optimization is gated by path timing/payload bit `7` at `path_entry+0x18 -> +0x1C`;
  - when that bit is clear on the anchor path, Windows sets a DisplayObject coerce flag through vfunc `+0x318`, making `DisplayPortMst` signal type `12` report as DisplayPort type `11` to downstream mode-gate code;
  - the later path-state builder can then mark non-anchor paths with `0x100000` skip-enable and `0x80000` force-disable/optimization flags;
  - this is still compatible with including those paths in the complete stream list and SyncManager/HWSync pairing.
- Interpretation:
  - Windows' 5K path is now a more specific recipe:
    1. normalize/coerce the internal DP-MST-style object when MST-to-SST optimization applies;
    2. build both SetMode path states;
    3. mark exactly one timing-server anchor with path-state bit `0x8000`;
    4. verify compatible timing payloads;
    5. create SyncManager inter-path requests pointing clients at that local timing server;
    6. let `SyncManager_ApplyInterPathSynchronization()` convert that state into `stream_arg+0x168 = 1`;
    7. run HWSequence/HWSync over the full stream list;
    8. optionally skip the later ordinary DS enable/unblank loop for the non-anchor path.
- Kernel-facing implication:
  - the closest Linux analogue is not "wake panel, then independently train two links";
  - it is "construct one paired 5K commit with a primary timing-server CRTC/path and a secondary timing-client CRTC/path";
  - Linux should log and, if possible, force the timing relationship explicitly before the first tile-pair commit;
  - useful log points are: chosen anchor CRTC/connector, secondary client mapping, selected timing equality, tile order, whether both paths share a timing source/sync source, and whether secondary link retrain/re-enable is being suppressed after the paired commit.

Extra RE continuation: tile-group anchor and timing bit7

- Scope:
  - this pass focused on two unknowns from the state-1/HWSync chain:
    - descriptor vfunc `+0x238`, used alongside `+0x2A8` when marking path-state bit `0x8000`;
    - timing payload bit `7` at `path_entry+0x18 -> +0x1C`, used by the MST-to-SST optimization and by split-mode setup.
- New/updated IDA anchors:
  - `0x141507AB0`: renamed `SetMode_FindTileGroupAnchorDisplay`;
  - `0x14140BA40`: renamed `BuildSourceModeForTileSplitPath`.
- Confirmed tile-group selection:
  - `SetMode_FindMstToSstOptimizationAnchorDisplay()` delegates multi-display path lists to a manager vfunc that resolves to `SetMode_FindTileGroupAnchorDisplay()`;
  - `SetMode_FindTileGroupAnchorDisplay()` first validates that each display index resolves to a live display object;
  - it then calls the descriptor vfunc `+0x2C8` to read tile-group metadata;
  - all candidates must share the same tile-group id;
  - the number of SetMode paths must match the tile-grid size returned by that metadata;
  - the helper tracks tile coordinates and only succeeds when every expected tile position is covered.
- Confirmed `+0x238` role:
  - inside a valid tile group, descriptor vfunc `+0x238` chooses the preferred anchor tile/display;
  - if the first display in the list has `+0x238 == true`, it remains the anchor;
  - if not, a later display with `+0x238 == true` replaces the anchor output;
  - this is not just "is tiled" or "is connected";
  - the best current interpretation is "preferred tile / anchor-capable tile inside the tile group";
  - I left the vfunc itself unnamed in IDA because the concrete method name is still not proven.
- Confirmed timing bit7 producers:
  - `MarkTimingPipeSplitForHighResAnd5k60()` sets timing flag bit7 when the timing aspect ratio matches the display geometry within the 90%-110% window;
  - it also marks 5120x2880@60 / high-res timings as pipe-split through `timing_record[9]` when `DalSupportMultiPipe` is enabled;
  - `MarkModeRecordIfTileAspectCompatible()` is another producer of the same timing bit7, used when endpoint/tiled metadata is updated after pin attach;
  - `sub_1415D5E40()` calls `MarkTimingPipeSplitForHighResAnd5k60()` while populating/inserting mode-list timings.
- Confirmed timing bit7 consumer:
  - `BuildSourceModeForTileSplitPath()` reads `path_entry+0x18 -> +0x1C` bit7;
  - when set, it treats the source mode as tile-split capable;
  - for a two-tile group it doubles the requested source width, then halves timing fields and programs each half against the two tile display objects;
  - this gives a direct SetMode consumer for the same bit that also gates MST-to-SST optimization.
- Confirmed SetMode path metadata:
  - `DSDispatch_LogSetModePathList()` logs `isPathMultiPipe` and `pipeCount` for every SetMode path;
  - `isPathMultiPipe` is read from path entry byte `+0x55`;
  - `pipeCount` is read from path entry dword `+0x58`;
  - this means Windows carries explicit per-path multi-pipe metadata in addition to timing bit7 and descriptor tile metadata.
- Interpretation:
  - Windows has two related but distinct concepts:
    - tile group identity/coverage from descriptor vfunc `+0x2C8`;
    - preferred timing anchor from descriptor vfunc `+0x238`;
  - timing bit7 is the mode/timing-side permission that says the selected timing can be treated as a tiled/pipe-split mode;
  - path-state bit `0x8000` then chooses a concrete timing-server path for SyncManager/HWSync.
- Kernel-facing implication:
  - Linux should not only check "two internal connectors exist";
  - it should also validate that the two selected timings form one complete tile group and that one connector/CRTC is chosen as the timing anchor;
  - our logs should include tile group id, tile coordinates/order, full tile coverage, selected timing dimensions, and chosen anchor;
  - if the kernel cannot infer an anchor from EDID/display topology, a hardcoded iMac19,1 anchor policy may be needed for the first plain-boot experiment.

Extra RE continuation: DisplayID tile metadata vtable proof and Apple cap rows

- Scope:
  - this pass tightened the producer/consumer proof for endpoint tile metadata and rechecked the Apple `APP/AE25` / `APP/AE26` capability rows against the current IDB;
  - the goal was to avoid treating a notes-level assumption as proven before using it to steer the Linux plain-boot path.
- New/updated IDA anchors:
  - `0x144CC6C28`: renamed `DisplayId20ParserVtable`;
  - `0x144CC70F0`: renamed `DisplayId13ParserVtable`;
  - `0x141AEEED0`: renamed `DisplayId20_EncodeManufacturerId3Char`;
  - `0x141AEEE70`: renamed `DisplayId20_ReadProductCode16`;
  - `0x141AEEDD0`: renamed `DisplayId20_ReadSerialNumber32`;
  - `0x141AFB3D0`: renamed `DisplayId13_EncodeManufacturerId3Char`;
  - `0x141AFB370`: renamed `DisplayId13_ReadProductCode16`;
  - `0x141AFB2D0`: renamed `DisplayId13_ReadSerialNumber32`;
  - `0x141A20850`: renamed `WindowsDM_CopyEndpointTileMetadataByDisplayId`;
  - `0x141ABC720`: renamed `InitMainVendorProductCapabilityTable`;
  - `0x141A44530`: renamed `GetPinCapabilityRecordCount`.
- Confirmed DisplayID parser vtable slot:
  - `PopulateEndpointTileMetadataFromDisplayIdTopology()` calls the display object's active EDID/DisplayID parser vtable slot `+0x158`;
  - for DisplayID 2.0, parser vtable slot `+0x158` at `0x144CC6D80` is `ParseDisplayId20TiledTopologyBlock40()`, which searches data-block tag `0x28`;
  - for DisplayID 1.x, parser vtable slot `+0x158` at `0x144CC7248` is `ParseDisplayId13TiledTopologyBlock18()`, which searches data-block tag `0x12`;
  - both parser variants fall back only to `QueryNextEdidExtensionForTiledTopology()`, meaning the next parser/extension in the same display object's chain.
- Confirmed endpoint tile metadata copy:
  - `WindowsDM_CopyEndpointTileMetadataByDisplayId()` is present as interface vtable slot `+0x2C8` at `0x144AF45D0` and `0x144AF6B08`;
  - it resolves the display id through the WindowsDM display-object table;
  - if `display_obj+0x90+0x98` has a nonzero token, it copies `0x38` bytes from `display_obj+0x90+0x98..+0xCF` to the caller;
  - if the endpoint block is absent, it returns false; there is no peer/primary tile metadata clone here.
- Confirmed tile group token components:
  - the parser copies/derives manufacturer, product, and serial identity fields from the DisplayID tiled-topology payload;
  - the manufacturer helper encodes the three-character ID into EDID/EISA-style bits;
  - the product helper reads the 16-bit product code;
  - the serial helper reads the 32-bit serial/unique field;
  - `PopulateEndpointTileMetadataFromDisplayIdTopology()` combines those fields into the endpoint tile group token consumed later by grouped validation.
- Confirmed Apple capability rows in the modern BootCamp driver:
  - main table pointer `off_144C4E080` points at `0x144C4F720`, a 361-record table of `{EDID manufacturer, EDID product, source capability id, value}`;
  - record `149` at `0x144C50070`: `APP/AE25`, source id `0x38`, value `0`;
  - record `150` at `0x144C50080`: `APP/AE26`, source id `0x39`, value `0`;
  - `MapCapabilityTableIdToPinPropertyTag()` maps source id `0x38` to pin tag `0x3A` and source id `0x39` to pin tag `0x3B`;
  - `SetPinDescriptorCapabilityBitById()` maps tag `0x3A` to pin descriptor byte `+6` bit `2`, and tag `0x3B` to byte `+6` bit `3`.
- Cross-check with our local Linux/OCLP evidence:
  - the captured 5K EDID reports base EDID model `0xAE26`;
  - its DisplayID tiled-topology block reports tiled product ID `44563` / `0xAE13`;
  - this means the base EDID product used for the Windows vendor/product capability table is not the same field as the DisplayID tiled product ID used for the tile group token.
- Interpretation:
  - the Windows path has two separate identity layers:
    - base EDID `APP/AE25` or `APP/AE26` selects Apple/tiled pin capabilities `0x3A` / `0x3B`;
    - DisplayID tiled-topology identity builds the shared tile group token used by SetMode/grouped validation.
  - detect/status-time `0x4F1` remains biased toward descriptor byte `+6` bit `2` / tag `0x3A`;
  - stream-enable-time `0x4F1` accepts byte `+6` bit `2` or bit `3`, so it remains the strongest static match for an `AE26` / tag-`0x3B` secondary path;
  - the post-bring-up `dc_sink+0x5B4` helper is weaker for `APP/AE25/AE26` because both static rows have value `0`;
  - no static `APP/AE25` or `APP/AE26` row proves tag `0x40`, so `DPCD 0x205` should remain diagnostic/read-only unless runtime evidence proves it for the exact boot state.
- Kernel-facing implication:
  - Linux should log base EDID product and DisplayID tiled product separately; mixing them can make the route look inconsistent when it is not;
  - the closest Windows-shaped behavior is still to preserve or reacquire the real secondary endpoint's EDID/DisplayID tile metadata, not to clone the primary tile's block as normal behavior;
  - for plain boot, the important success condition is reaching a paired stream-enable equivalent with the real secondary AUX/route alive and the `APP/AE26 -> tag 0x3B` capability recognized;
  - if a Linux fallback synthesizes TILE metadata, it should be labeled as a compatibility fallback after the secondary route was physically proven, not as behavior copied from Windows.

Extra RE continuation: false-disconnect / secondary preservation path

- Scope:
  - this pass rechecked the Apple/tiled false-disconnect path in the modern BootCamp AMD driver;
  - the goal was to distinguish three different things that are easy to blur together:
    - the 120-byte detection result returned by the detect wrapper;
    - the current live display object / detection-record live pointer;
    - the lower display-map entry and per-object AUX/DPCD transport.
- New/updated IDA anchors:
  - `0x1414A9E90`: renamed `IsDisplayMapDetectRefcountPass`;
  - `0x1414A9EE0`: renamed `IsDisplayMapDetectNotifyPass`;
  - `0x1414AE400`: renamed `RetainDisplayMapEntriesForDetectPass`;
  - `0x1414AE020`: renamed `ReleaseDisplayMapEntriesAfterDetectPass`;
  - `0x1414B3A20`: renamed `UpdateDetectionRecordLiveDisplayObject`;
  - `0x1414AFD20`: renamed `NotifyDetectionRecordDisconnected`;
  - `0x1414AFD70`: renamed `NotifyDetectionRecordConnected`.
- Confirmed detection-result layout:
  - `DetectDisplayMainMaybeApple5k()` allocates/clears a 120-byte result buffer;
  - result byte `+0x72` is the detected connected flag;
  - result byte `+0x73` is a special side-path flag that bypasses the normal live-state update path;
  - result byte `+0x70` is the changed/rescan flag consumed by `HandleDisplayDetectionResultAndLiveState()`;
  - `ApplyDetectionResultAndUpdateLiveDisplayObject()` later passes result byte `+0x72` and result byte `+0x70` into `UpdateDisplaySinkConnectionAndLiveAuxState()`.
- Confirmed false-disconnect gate:
  - `DetectDisplayAndApplyApple5kOverrides()` first seeds the result from the current display object state;
  - if `DetectDisplayEarlyExitForApple5kCaps()` does not short-circuit, the normal path runs status detection and can call the detect/status `0x4F1` path;
  - after that normal detection, the Apple/tiled false-disconnect branch checks:
    - connected pin descriptor byte `+6` bit `3` is set, matching the `0x3B` capability family;
    - the current display object still reports connected through vfunc `+0x258`;
    - the new detection result says disconnected at result byte `+0x72`;
    - current Windows signal type is `11`, which this driver logs as DisplayPort/SST in the local signal enum.
  - when all four are true, Windows writes:
    - result byte `+0x72 = 1`;
    - result byte `+0x70 = 0`.
- Confirmed teardown avoidance:
  - `HandleDisplayDetectionResultAndLiveState()` computes `changed = result.connected != current_connected`;
  - after the `0x3B` correction, `changed` is false and `result+0x70` is clear;
  - for this DP false-disconnect case, it therefore does not call `ApplyDetectionResultAndUpdateLiveDisplayObject()`;
  - because that function is skipped, `UpdateDisplaySinkConnectionAndLiveAuxState()` is skipped;
  - because that function is skipped, `UpdateDetectionRecordLiveDisplayObject()` is not allowed to clear the detection record's live object pointer.
- Confirmed what the skipped clear would do:
  - `UpdateDetectionRecordLiveDisplayObject()` finds the detection record for the current object id;
  - when the display object reports connected, it stores the live object pointer at record `+0x18`;
  - when the display object reports disconnected, it calls `NotifyDetectionRecordDisconnected()` and clears record `+0x18`;
  - this is the teardown that the `0x3B` false-disconnect path avoids.
- Confirmed lower display-map bookkeeping still exists:
  - `RetainDisplayMapEntriesForDetectPass()` runs before the normal detect/status path;
  - `ReleaseDisplayMapEntriesAfterDetectPass()` runs after the false-disconnect correction on the normal path;
  - `IsDisplayMapDetectRefcountPass()` returns true for detect passes `0`, `2`, and `3`;
  - on those passes the retain/release helpers increment/decrement display-map entry `+0x0C` and clear byte `+0x12` when the count reaches zero;
  - this is not the same as clearing the detection-record live object pointer at record `+0x18`.
- Interpretation:
  - Windows does not preserve the Apple 5K secondary by keeping a fake sink after disconnect;
  - it edits the detection result before the live-object update path can convert a transient disconnected result into object teardown;
  - it still has lower display-map pass bookkeeping around the same detect window, so the visible mechanism is more subtle than "never release anything";
  - the durable per-object AUX/DPCD transport remains the display-map entry path validated by `ProbeDisplayStatusMaybeAssert4F1()` from entry `+0x18/+0x20`.
- Kernel-facing implication:
  - the closest Linux analogue is to suppress or defer the exact false disconnect before generic detect code clears `dc_link->local_sink` or powers down the lower route;
  - restoring a previous sink pointer after the lower route has already been invalidated is weaker than the Windows behavior;
  - a correct quirk should require concrete proof that the real secondary `0x3113` route was live/AUX-writable earlier, then prevent the transient HPD/detect result from tearing down that real route;
  - preserving only DRM `TILE` metadata or an emulated sink is not enough if the lower DDC/AUX backend is allowed to die.

Extra RE continuation: exact secondary stream-enable conditions

- Scope:
  - this pass rechecked the modern BootCamp AMD driver's stream-enable path after the false-disconnect mechanism was clarified;
  - the goal was to separate the exact AE26/tag-`0x3B` secondary stream-enable route from the other `0x4F1` writers that look similar but have different gates.
- New/updated IDA anchors:
  - `0x141B70E60`: renamed `IsDpFamilySignal32_64_128`;
  - added `IMAC5K_RE_STREAM_COND` comments on the mode-11 gate, stream worker state branches, descriptor-bit checks, lower signal classifier, and the two weaker `0x4F1` helper routes.
- Confirmed mode-11 immediate gate:
  - `EnableValidatedStreamMaybeAssert4F1()` calls `UpdateMode11ImmediateStreamEnableFlag()` before deciding whether to call `StreamEnableWrite4F1AndPollSinkStatus()`;
  - `UpdateMode11ImmediateStreamEnableFlag()` calls vtable slot `+0x160` on the display object stored at `stream_arg+0x178`;
  - if that method returns stream mode `11`, it sets link-service byte `+0x40A`;
  - because the wrapper receives `link_service+0x28`, the same byte appears as `a1+994` in `EnableValidatedStreamMaybeAssert4F1()`;
  - if the method returns stream mode `12`, it clears byte `+0x40A` and resets the associated allocation/cache state if the byte had been set;
  - only the mode-11/immediate branch calls `StreamEnableWrite4F1AndPollSinkStatus()` from this wrapper.
- Confirmed stream-enable worker states:
  - `StreamEnableWrite4F1AndPollSinkStatus()` uses state field `this+0x58`;
  - state `2` or state `3` returns success immediately;
  - state `1` is an already-link-configured pending-stream-enable path:
    - it programs/caches the requested tuple;
    - writes/updates DPCD `0x316`;
    - programs stream source state;
    - calls the real stream-enable vfunc;
    - promotes state `1 -> 3`;
    - returns without the later tag-gated `0x4F1` write.
  - state `0` is the normal path:
    - maybe clears DPCD `0x111` bit `0`;
    - maybe verifies/retries link caps;
    - calls `TryEnableLinkWithHbr2Fallback()`;
    - writes/updates DPCD `0x316`;
    - programs stream source state;
    - calls the real stream-enable vfunc;
    - then attempts DPCD `0x4F1 = 1` only if the descriptor gate is true;
    - marks state `2`;
    - polls DPCD `0x205` only if the separate tag-`0x40` gate is true.
- Confirmed descriptor gates:
  - the stream-enable `0x4F1` write accepts connected pin descriptor byte `+6` bit `2` or bit `3`;
  - that means both tag `0x3A` and tag `0x3B` can reach the stream-enable latch;
  - the `0x205` poll is separately gated by descriptor byte `+6` bit `5` / tag `0x40`;
  - the static `APP/AE25` and `APP/AE26` rows prove tag `0x3A` / `0x3B`, not tag `0x40`, so `0x205` should stay diagnostic unless runtime evidence proves it.
- Confirmed cached vs fresh link conditions:
  - `LinkServiceHasCachedLinkSettings()` is only a cached-tuple presence check: dword `link_service+0x8C != 0`;
  - `TryEnableLinkWithHbr2Fallback()` compares the requested tuple dword0/dword1 against cached `+0x8C/+0x90`;
  - if they match, it returns link-enable success without fresh training;
  - if they do not match, it disables/reprograms and calls the ASSR/training path;
  - state `1` is stricter than cached tuple presence: `SetState1IfDpcdReportsTrainedLink()` calls `RecoverActiveDpLinkSettingsFromDpcd()` and only arms state `1` if DPCD `0x100/0x101` plus lane status `0x202/0x203` prove a live trained link.
- Confirmed other `0x4F1` routes are weaker for AE26 secondary bring-up:
  - `WindowsDM_EnableSecondaryTileIfRequired()` checks descriptor byte `+6` bit `2` or bit `3`, but then additionally requires lower display/link type `128` before calling `SendSecondaryTileOpcode1265Payload1()`;
  - this looks like the eDP/root helper, not the whole signal-`32` AE26 secondary path;
  - `BringUpDisplayBySignalType()` has a post-success helper that can call `SendSecondaryTileOpcode1265Payload1()` after lower bring-up, but it is gated by lower display object flag/value `+0x5B4`;
  - the static `APP/AE25` and `APP/AE26` table rows have value `0`, so this post-bring-up helper is weaker evidence than the stream-enable worker for our panel.
- Lower bring-up relationship:
  - `IsDpFamilySignal32_64_128()` returns true exactly for signal types `32`, `64`, and `128`;
  - in `BringUpDisplayBySignalType()`, signal `32` reaches `BringUpDpOrEdpWithLinkTraining()`, signal `64` takes the MST pre-enable path, and signal `128` takes the eDP/root path;
  - `BringUpDpOrEdpWithLinkTraining()` publishes DP source-DPCD `0x300/0x303/0x310` before link training;
  - therefore the stream-enable path assumes lower DP source-DPCD setup and training have already happened. It is not the place where the whole secondary AUX/backend route is created from zero.
- Kernel-facing implication:
  - the false-disconnect preservation path is now clear enough to move on from: Linux should prevent teardown of a physically proven secondary route before generic detect clears it;
  - the exact Windows-shaped secondary stream-enable checklist is:
    - real secondary object survives detect;
    - stream object resolves through `stream_arg+0x178`;
    - stream mode is `11`, arming link-service byte `+0x40A`;
    - the worker is not already state `2` or `3`;
    - state `1` is allowed only with live trained-link DPCD proof; otherwise state `0` runs normal cached-or-fresh link enable;
    - source/stream state is programmed and the real stream-enable vfunc runs;
    - only then does tag `0x3A` or tag `0x3B` permit DPCD `0x4F1 = 1`;
    - `0x205` polling is not part of the AE26/tag-`0x3B` requirement unless tag `0x40` appears at runtime.
  - for Linux testing, the next high-signal logs should capture whether the secondary route reaches the stream-enable equivalent with:
    - live trained-link proof (`0x100/0x101`, `0x202/0x203`);
    - source-DPCD `0x300/0x303/0x310` already programmed;
    - selected stream/tile mode equivalent to Windows mode `11`;
    - recognized `APP/AE26 -> tag 0x3B` capability;
    - 0x4F1 attempted only after stream/source programming, not as the first wake operation.

Extra RE continuation: secondary source-DPCD / training sequence

- Scope:
  - this pass rechecked the lower DP source-DPCD and training loops in the modern BootCamp AMD driver;
  - the goal was to distinguish lower initial bring-up from later DisplayPortLinkService fresh-training, because they share concepts but not exact DPCD write order.
- New/updated IDA anchors:
  - `0x1414DB6C0`: renamed `TrainClockRecoveryLoop`;
  - `0x1414DABC0`: renamed `TrainChannelEqualizationLoop`;
  - `0x1414DC080`: renamed `ReadLaneStatusAndAdjustRequests`;
  - `0x1414DD300`: renamed `DpcdSetLaneSettingsOnly`;
  - `0x1414DCD60`: renamed `MapTrainingPatternIdToDpcdValue`;
  - `0x141B87D70`: renamed `LowerPerformDpLinkTraining`;
  - `0x141B8B9D0`: renamed `LowerDpcdSetLinkSettings`;
  - `0x141B898D0`: renamed `LowerTrainClockRecoverySequence`;
  - `0x141B89C60`: renamed `LowerTrainChannelEqualizationSequence`;
  - `0x141B8A7D0`: renamed `LowerReadLaneStatusAndAdjustRequests`;
  - `0x141B8B200`: renamed `LowerDpcdSetLtPatternAndLaneSettings`;
  - `0x141B8A2F0`: renamed `LowerDpcdSetLaneSettingsOnly`;
  - `0x141B8C0E0`: renamed `LowerWaitTrainingAuxRdInterval`.
- Confirmed lower initial bring-up order:
  - `BringUpDisplayBySignalType()` routes signal `32` to `BringUpDpOrEdpWithLinkTraining()`;
  - `BringUpDpOrEdpWithLinkTraining()` order remains:
    - optional DPCD `0x111` bit0 only for signal `32` plus link-rate class `2`;
    - type-`128` HPD/AUX callbacks only for signal/display type `128`;
    - `ProgramDpPreTrainingAuxState()`;
    - `PerformLinkTrainingWithRetries()`;
    - optional post-training cleanup/state writes such as `0x32F`.
  - therefore the iMac secondary `0x3113` / signal-`32` route should not be treated as if it automatically receives the type-`128` HPD/AUX re-arm callbacks.
- Confirmed source-DPCD ownership:
  - `ProgramDpPreTrainingAuxState()` is the lower owner of source-DPCD publication before training;
  - it writes:
    - DPCD `0x300`: source OUI, payload `00 00 1A` when the existing OUI is not already AMD/ATI;
    - DPCD `0x303`: 9-byte source/device descriptor block;
    - DPCD `0x310`: source table revision, normally `04 1D 03` for the DCE path we mapped;
    - optional DPCD `0x340`: AMD minimum-hblank byte when the runtime field is nonzero.
  - a separate cached-block branch can write 12 bytes starting at `0x300`, but that covers `0x300..0x30B`; it does not cover `0x310`;
  - this supports the current Linux direction: keep `0x310` publication adjacent to the lower source-DPCD path, not only in the higher DM grouped-view hook.
- Confirmed lower training retry order:
  - `PerformLinkTrainingWithRetries()` runs up to the caller-provided retry count;
  - before each attempt it calls `PrepareLinkTrainingStateAndArmEdp128()`;
  - `PrepareLinkTrainingStateAndArmEdp128()` repeats HPD/AUX arm callbacks only when lower signal/type is `128`;
  - it writes DP_SET_POWER / DPCD `0x600` D0 after source-side training state is prepared;
  - a failed attempt calls `DisableDpLinkMaybeD3PowerOff()` before retry;
  - a successful attempt returns before the D3/disable cleanup.
- Confirmed lower link-setting write order:
  - `LowerDpcdSetLinkSettings()` is the lower training implementation's link-setting writer;
  - for the ordinary 8b/10b path it writes:
    - DPCD `0x107` spread/downspread first;
    - DPCD `0x101` lane count second;
    - DPCD `0x100` link rate third.
  - there is a separate branch for newer/link-specific cases that writes `0x100 = 0` and then `0x115`;
  - this differs from the later DisplayPortLinkService fresh-training helper `DpcdSetLinkSettingsWrite100101Then107()`, which writes `0x100/0x101` as a two-byte transfer and only then writes `0x107`.
- Confirmed lower CR/EQ loop:
  - `LowerPerformDpLinkTraining()` dispatches to `LowerTrainClockRecoverySequence()` and `LowerTrainChannelEqualizationSequence()` for link-rate class `1`;
  - `LowerDpcdSetLtPatternAndLaneSettings()` writes DPCD `0x102` training pattern plus lane settings at `0x103..` as one block unless split-write behavior is selected;
  - `LowerDpcdSetLaneSettingsOnly()` writes subsequent lane settings at `0x103..` after adjust requests change drive settings;
  - `LowerReadLaneStatusAndAdjustRequests()` reads DPCD `0x202..0x207` and extracts lane status plus adjust request bytes;
  - clock recovery loops until success, 100 total iterations, or five same-drive attempts;
  - channel equalization loops up to six iterations and requires CR, channel EQ, symbol lock, and interlane alignment.
- Confirmed later DisplayPortLinkService fresh-training path:
  - `TryEnableLinkWithHbr2Fallback()` only calls this path on cached tuple mismatch/absence;
  - `ProgramAssrThenTrainLinkSettings()` order is:
    - `ProgramRequestedLinkSettingsAndCache()`;
    - `ClassifyDpPanelModeForAssrPolicy()`;
    - `ProgramDpcd10A_AssrBitForPanelMode()`;
    - `TrainLinkWithAssrRetryOnceOnStatus6()`;
    - `PerformLinkTrainingWithAssrPatternPolicy()`.
  - this path writes ASSR DPCD `0x10A` before training;
  - it also passes the same ASSR/panel policy into PHY/training-pattern requests through `SetDpPhyPatternWithAssrPolicy()`;
  - it does not republish source-DPCD `0x300/0x303/0x310`; it assumes lower source-DPCD publication already happened.
- Interpretation:
  - Windows has two relevant training layers:
    - lower initial bring-up: source-DPCD owner, lower link-setting writer, lower CR/EQ loops;
    - later DisplayPortLinkService fresh training: cached/fresh decision, ASSR `0x10A`, ASSR-aware PHY/training-pattern policy, CR/EQ loops.
  - the iMac secondary route likely needs both pieces when it cannot use cached handoff:
    - lower source-DPCD must be published before any destructive training writes;
    - ASSR/panel policy must be visible before and during training;
    - if the preserved tuple is valid and matches, Windows can avoid fresh training entirely through the cached path.
- Kernel-facing implication:
  - do not use source-DPCD `0x310` readback as the only readiness proof; previous Linux tests showed successful lower writes can still read back as zeros on this sink;
  - do require write-success/evidence that the lower source-DPCD path ran before link-setting writes;
  - do not copy the type-`128` HPD/AUX arm callbacks into the signal-`32` secondary path without stronger evidence;
  - the next useful Linux logs should show, for the exact secondary `0x3113` route:
    - whether lower source-DPCD publication occurs before any `0x107/0x101/0x100` or `0x100/0x101/0x107` link-setting writes;
    - whether `DP_PANEL_MODE_EDP` / ASSR policy reaches both DPCD `0x10A` and training-pattern/PHY programming;
    - whether the first two-tile commit is using cached handoff or fresh training;
    - if fresh training happens, the exact first failing stage: `0x100/0x101/0x107`, `0x102/0x103`, `0x202..0x207`, or post-failure D3 cleanup.

Extra RE continuation: mode-11 source and MST-to-SST coercion gate

- Scope:
  - this pass rechecked the exact source of the stream mode `11` gate used by `UpdateMode11ImmediateStreamEnableFlag()`;
  - the goal was to connect the stream-enable decision back to the SetMode MST-to-SST optimization instead of treating mode `11` as an unexplained display-object result.
- New/updated IDA comments:
  - added `IMAC5K_RE_MODE11_SOURCE` comments on:
    - `DisplayObject_GetCommittedSignalTypeForModeGate()`;
    - `DisplayObject_SetSignal12ReportsAs11Flag()`;
    - `SetMode_ApplyMstToSstOptimizationSignalCoercion()`;
    - `DSDispatch_SetMode()`;
    - `DSDispatch_BuildApplySetModePaths()`;
    - tile-group anchor selection;
    - HWSync source-signal adjustment.
- Confirmed mode-gate field:
  - `DisplayObject_GetCommittedSignalTypeForModeGate()` reads the committed signal type from display interface `+0x48`;
  - if display interface byte `+0x1E9` is set and the committed type is `12`, it returns `11`;
  - otherwise it returns the committed type unchanged;
  - therefore the stream-enable mode-`11` gate can be reached either by a naturally committed type `11`, or by committed type `12` plus byte `+0x1E9`.
- Confirmed setter:
  - `DisplayObject_SetSignal12ReportsAs11Flag()` is the vtable slot `+0x318` setter for byte `+0x1E9`;
  - it only writes that byte and returns;
  - xrefs show this function is installed through the display-object vtable; direct callers reach it virtually.
- Confirmed SetMode producer:
  - `DSDispatch_SetMode()` calls `SetMode_ApplyMstToSstOptimizationSignalCoercion()` before building/applying the SetMode path state;
  - `DSDispatch_BuildApplySetModePaths()` calls the same helper again while building path-state records;
  - if the helper succeeds, the path builder also marks affected paths with the MST-to-SST optimization / force-disable metadata.
- Confirmed optimization gate:
  - `SetMode_ApplyMstToSstOptimizationSignalCoercion()` first requires `SetMode_AllDisplaysHaveAtMostTwoTiles()`;
  - it finds an MST-to-SST optimization anchor display;
  - it then reads anchor path timing/payload bit `7` at `path_entry+0x18 -> +0x1C`;
  - if anchor bit `7` is clear, it calls display-object vfunc `+0x318(1)` for each path display, setting byte `+0x1E9`;
  - if anchor bit `7` is set, it calls vfunc `+0x318(0)`, clearing byte `+0x1E9`.
- Confirmed anchor/tile relationship:
  - multi-display anchor selection goes through the tile-group anchor helper;
  - that helper validates live display objects, common tile-group id, full tile-grid coverage, and preferred anchor flag from descriptor vfunc `+0x238`;
  - this ties the mode-`11` coercion to the valid tiled SetMode group, not merely to "two connectors exist".
- Confirmed HWSync is separate:
  - `HWSync_AdjustDpMstEdpSourceSignalForInterPath()` can write source-side signal type `8` for DP-MST/eDP peers through a source-object vfunc;
  - that is HWSync/source state and is separate from display-object byte `+0x1E9`;
  - it does not explain the `UpdateMode11ImmediateStreamEnableFlag()` result by itself.
- Interpretation:
  - the mode-`11` immediate stream-enable branch is a Windows SetMode-owned state, not a firmware-only artifact;
  - for a type-`12` DisplayPortMST-style object, the branch is enabled when MST-to-SST optimization sets byte `+0x1E9`;
  - that optimization is gated by a valid tiled path list, anchor selection, and selected timing/path metadata.
- Kernel-facing implication:
  - Linux should not only preserve the secondary AUX/link and TILE metadata;
  - it also needs the atomic commit to be recognized as a paired tiled SetMode equivalent before expecting Windows-like mode-`11` stream behavior;
  - useful next logs are:
    - whether the first 5K commit has both tile paths in one grouped atomic state;
    - which path is chosen as timing anchor;
    - whether selected timing/path metadata corresponds to Windows' bit-`7` tiled/split-mode gate;
    - whether the secondary is being treated as a mode-`11`/SST stream equivalent or left in a mode-`12`/MST-like path that would clear the immediate stream-enable flag.

Extra RE continuation: object `0x3113` route construction and lifetime

- Scope:
  - this pass rechecked how the modern BootCamp AMD driver constructs the secondary route from the object table, how the DPCD/AUX backend is attached, and which later lifetime paths touch the display-map entry versus the detection record;
  - the goal was to make Stage 2 plain-boot Linux less vague: either Linux must discover the same real `0x3113` route, or it must construct an equivalent physical route early enough for normal detect/training/stream code to use it.
- New/updated IDA anchors:
  - `0x1414B5AE0`: renamed `BuildStreamsForDisplayObjectBySignalType`;
  - `0x1414AC570`: renamed `ReleaseStreamsForDisplayMapEntry`;
  - `0x1414B7810`: renamed `AttachChildInterfacesToDisplayObjectPath`;
  - added `IMAC5K_RE_3113_ROUTE` comments on:
    - `PopulateDisplayMapAuxObjectsForHotplugRange()`;
    - `CreateDpcdAuxInterfaceForObjectId()`;
    - `ConstructDpcdAuxInterfaceForObjectId()`;
    - `BuildStreamForDisplayEntryPath()`;
    - `BuildStreamsForDisplayObjectBySignalType()`;
    - `ConstructDpStreamObjectWithDpcdInterface()`;
    - `AttachStreamInterfaceToDisplayEntry()`;
    - `AddDisplayObjectToDetectionRecord()`;
    - `MarkDisplayMapEntryLive()`;
    - `DecrementDisplayMapEntryRefAndClearLiveByte()`.
- Confirmed construction path:
  - `ConstructDisplayTopologyManager()` calls `PopulateDisplayMapAuxObjectsForHotplugRange()`;
  - that function iterates firmware/object-table display object ids through the adapter provider, creates a display-map interface for each candidate object id, inserts a 64-byte display-map entry, and only then creates the object-specific DPCD/AUX interface;
  - `0x3113` passes the class-3 filter in `CreateDisplayMapInterfaceForObjectIdMaybeSkipInternal()`;
  - the property13 bit17 skip policy can drop low-byte `0x14` or `0x0E`, but it does not drop low-byte `0x13`, so the secondary `0x3113` route is intentionally allowed through this constructor path.
- Confirmed DPCD/AUX attachment:
  - `PopulateDisplayMapAuxObjectsForHotplugRange()` stores `CreateDpcdAuxInterfaceForObjectId()` at display-map entry `+0x18`;
  - `CreateDpcdAuxInterfaceForObjectId()` is only called from this object-table population path in the current IDB;
  - `ConstructDpcdAuxInterfaceForObjectId()` obtains the lower AUX handle through AdapterService vfunc `+0x170`;
  - AdapterService vfunc `+0x170` is `AdapterServiceGetAuxHandleForObjectId()`, which queries the exact object descriptor and passes descriptor selector/source-mask data into the AUX factory;
  - for secondary `0x3113`, the descriptor selector is `0x4871`, which `Dce11ResolveAuxSelectorToPinIndex()` maps to AUX pin index `2`;
  - the dual-pin AUX transaction handle then requires the lower pin-graph endpoints for type `3` and type `4`, matching the DDC/AUX data/clock backend for that pin.
- Confirmed `0x3113` DPCD-interface policy:
  - `ConstructDpcdAuxInterfaceForObjectId()` defaults `DPDelay4I2CoverAUXDEFER` to true when the class-3 object low byte is `0x13`;
  - the same constructor sets byte `+0x88C` only for low-byte `0x14` or `0x0E`, not for `0x13`;
  - so Windows treats `0x3113` as a real DP-family object with a special AUX defer delay, not as the same eDP/internal bucket used by low-byte `0x14`.
- Confirmed stream construction from the route:
  - after entry `+0x18` is populated, `ExpandDisplayPathAndConstructStreams()` recursively expands child/transport objects and eventually calls `ConstructDisplayObjectAndStreams()`;
  - `BuildStreamForDisplayEntryPath()` builds the stream init block from the display-map path and passes the display-map entry `+0x18` DPCD/AUX object through init offset `+0x18`;
  - `ConstructDpStreamObjectWithDpcdInterface()` stores that init offset at stream object `+0xB8`, so the resulting DP stream carries the same object-specific DPCD/AUX backend forward;
  - `AttachStreamInterfaceToDisplayEntry()` records stream interfaces in the display-map stream table. This makes the created stream interfaces durable table entries, not only local temporary allocations.
- Confirmed `0x3113` special stream-shape:
  - `BuildStreamsForDisplayObjectBySignalType()` has an explicit class-3 low-byte `0x13` branch when the display object's per-stream signal/type is `0x0B`;
  - on that branch Windows calls:
    - `BuildStreamForDisplayEntryPath(..., kind 3)`;
    - `BuildStreamForDisplayEntryPath(..., kind 2)`;
    - then `BuildStreamForDisplayEntryPath(..., kind 1)`.
  - the kind `3` and kind `2` results are attached through the stream table as side effects, while the kind `1` stream is stored back on the display object through the normal vfunc `+0x468` setter;
  - this is a concrete Windows lifetime detail we had not written down before: the secondary `0x3113` route can have more stream-interface state than just the one final DP stream pointer.
- Confirmed display-map liveness:
  - successful display-object construction calls `MarkDisplayMapEntryLive()` for the main display interface and for each stream/path peer interface;
  - `MarkDisplayMapEntryLive()` sets display-map entry byte `+0x10` and byte `+0x11`;
  - if the entry was already live, it ORs bit0 into the interface flags before re-marking it live;
  - this is separate from the 104-byte detection-record live object pointer at record `+0x18`.
- Confirmed detection-record relationship:
  - `ConstructDisplayObjectRecordTable()` allocates one 104-byte record per display object id from the provider;
  - `AddDisplayObjectToDetectionRecord()` stores up to two candidate display objects at record `+0x20`, with candidate count at record `+0x30`;
  - if the record's base/display interface has class-3 low byte `0x13`, it sets record byte `+0x13 = 1` and clears byte `+0x12 = 0` before candidate insertion/timer scheduling;
  - `UpdateDetectionRecordLiveDisplayObject()` later writes the currently connected display object to record `+0x18`, or clears record `+0x18` on a reported disconnect;
  - the previously traced `0x3B` false-disconnect override prevents that clear by editing the detection result before this updater runs.
- Confirmed what detect-pass release does and does not do:
  - `RetainDisplayMapEntriesForDetectPass()` increments display-map entry `+0x0C` for detect passes `0`, `2`, and `3`;
  - `ReleaseDisplayMapEntriesAfterDetectPass()` calls `DecrementDisplayMapEntryRefAndClearLiveByte()`;
  - that helper decrements entry `+0x0C` and clears byte `+0x12` when the count reaches zero;
  - it does not clear display-map entry `+0x18`, and it is not the same operation as clearing detection-record record `+0x18`.
- Interpretation:
  - Windows' `0x3113` route is object-table driven, not EDID/TILE-synthesized;
  - the critical physical chain is:
    - object id `0x3113`;
    - class-3 display-map interface accepted;
    - display-map entry inserted;
    - entry `+0x18` DPCD/AUX wrapper created;
    - AdapterService exact-object descriptor queried;
    - selector `0x4871` resolved to AUX pin index `2`;
    - dual pin endpoints type `3` and type `4` opened;
    - display/stream objects constructed from that route;
    - low-byte `0x13` stream side-interfaces attached;
    - display-map entries marked present/live;
    - detection record candidates/live pointer updated separately.
  - the route lifetime has at least three layers that must not be conflated:
    - display-map entry and its object-specific DPCD/AUX transport at entry `+0x18`;
    - stream-interface table entries attached from that display-map path;
    - detection-record candidate/live pointers in the 104-byte record table.
- Kernel-facing implication:
  - Stage 2 Linux should first log whether plain boot enumerates an AMD/VBIOS/object-table route equivalent to `0x3113` with DDC3/AUX pin index `2` and the expected transmitter/encoder pairing;
  - if `0x3113` is present but entry/AUX creation fails, the target is the AUX factory/backend path, not TILE metadata;
  - if entry/AUX creation succeeds but the route is later lost, the target is display-map/detect/stream lifetime preservation;
  - if `0x3113` is absent from plain boot enumeration, the kernel quirk must expose or construct the real physical secondary route before normal detection, rather than faking a second tile after detect;
  - useful Linux logs should mirror the Windows layers:
    - raw object id / low byte;
    - DDC line or AUX hw instance;
    - transmitter/encoder;
    - HPD source;
    - whether the route matches secondary `0x3113` / DDC3 / UNIPHY_D;
    - whether the DPCD/AUX backend exists before detect;
    - whether stream construction creates one or multiple stream-interface records;
    - exactly which later transition clears `local_sink`, powers off AUX/link, or drops the secondary stream.

Extra RE continuation: provider/object enumeration gate

- Scope:
  - this pass checked the provider-side gate above `PopulateDisplayMapAuxObjectsForHotplugRange()`;
  - the question was whether secondary `0x3113` can be created by a synthetic provider, a BootCamp policy provider, or whether it must be present in the real VBIOS/object-info enumeration before the AUX-backed display-map path runs.
- New/updated IDA anchors:
  - `0x1414AD690`: renamed `ResizeDisplayMapStreamTableCapacity`;
  - `0x141466EE0`: renamed `AdapterServiceGetTotalDisplayObjectCount`;
  - `0x1414666E0`: renamed `AdapterServiceGetDisplayObjectIdByIndex`;
  - `0x141460A80`: renamed `AdapterServiceGetSyntheticTailObjectCount`;
  - `0x141463C20`: renamed `AdapterServiceBuildSyntheticTailObjectId`;
  - `0x14152C9E0`: renamed `ObjectInfoDescriptorAdapterGetDisplayObjectCount`;
  - `0x14152C900`: renamed `ObjectInfoDescriptorAdapterGetDisplayObjectIdByIndex`;
  - `0x14152F990`: renamed `DisplayDescriptorCapsGetObjectCount`;
  - `0x14152F830`: renamed `DisplayDescriptorCapsBuildObjectIdByIndex`;
  - `0x1415AE800`: renamed `VbiosParserV2GetDisplayObjectCount`;
  - `0x1415AE6B0`: renamed `VbiosParserV2GetDisplayObjectIdByIndex`;
  - `0x1415A8640`: renamed `VbiosParserV1GetDisplayObjectCount`;
  - `0x1415A8450`: renamed `VbiosParserV1GetDisplayObjectIdByIndex`;
  - added `IMAC5K_RE_ENUM_GATE` comments around the count/id-by-index calls and the VBIOS descriptor fields.
- Confirmed non-gate:
  - `ResizeDisplayMapStreamTableCapacity(display_map, 100)` only resizes the display-map stream table;
  - it allocates/copies/free-replaces the table and sets capacity at `+0xB8`;
  - it is not a 5K-vs-4K policy decision and the constant `100` is just capacity.
- Confirmed provider ordering:
  - `AdapterServiceGetTotalDisplayObjectCount()` returns:
    - ObjectInfoDescriptorAdapter / VBIOS object-info count;
    - plus DisplayDescriptorCapabilityFlags count;
    - plus AdapterService synthetic-tail count.
  - `AdapterServiceGetDisplayObjectIdByIndex()` resolves ids in that same order:
    - VBIOS/object-info provider first;
    - display-descriptor capability provider second;
    - synthetic tail last.
  - `PopulateDisplayMapAuxObjectsForHotplugRange()` deliberately loops only `total_count - synthetic_tail_count`;
  - therefore the objects that receive display-map entries and object-specific DPCD/AUX backends exclude the synthetic tail.
- Confirmed synthetic-tail contents:
  - `AdapterServiceGetSyntheticTailObjectCount()` computes only the number of missing synthetic tail objects;
  - DAL/register id `0x441` can override the expected tail requirement, otherwise a base/signal-provider value of `13` can request one;
  - this affects tail count only, and that tail is subtracted before AUX population;
  - `AdapterServiceBuildSyntheticTailObjectId()` produces:
    - family `0`: class `3`, type/low-byte `0x18`, instance `1..7`;
    - family `1`: class `2`, type/low-byte `0x26`, instance `1..7`.
  - it cannot produce class-3 low-byte `0x13`, so synthetic enumeration is not the source of secondary `0x3113`.
- Confirmed display-descriptor capability provider contents:
  - `DisplayDescriptorCapsGetObjectCount()` returns up to two objects depending on capability bytes at `+0x30..+0x32`;
  - `DisplayDescriptorCapsBuildObjectIdByIndex()` produces:
    - class `3`, type/low-byte `0x16`, instance `1`;
    - optionally class `3`, type/low-byte `0x17`, instance `1`.
  - this provider also cannot publish `0x3113`.
- Confirmed ObjectInfoDescriptorAdapter role:
  - `ConstructObjectInfoDescriptorAdapter()` is a thin wrapper around the VBIOS object-info parser;
  - its count and id-by-index vfuncs delegate directly to the parser at object field `+0x30`;
  - because the capability provider and synthetic tail cannot produce `0x3113`, an AUX-backed secondary `0x3113` must come from this VBIOS/object-info provider.
- Confirmed VBIOS parser gates:
  - V2:
    - `VbiosParserV2GetDisplayObjectCount()` returns zero if the display-base signal path reports `13`;
    - otherwise it counts object-table entries whose object/encoder refs are both present;
    - `VbiosParserV2GetDisplayObjectIdByIndex()` decodes the object id from the VBIOS object-table record ref.
  - V1:
    - `VbiosParserV1GetDisplayObjectCount()` has the same display-base signal `13` zero-count gate;
    - otherwise it returns the object count from the V1 object table;
    - `VbiosParserV1GetDisplayObjectIdByIndex()` decodes object ids from the V1 object-table record ref.
- Confirmed descriptor source:
  - V2 descriptor parse fills descriptor field `+0x1C` from the GPIO/AUX selector table;
  - V2 descriptor parse fills descriptor field `+0x3C` with the source/index byte;
  - `AdapterServiceGetAuxHandleForObjectId()` later consumes those fields to build the object-specific AUX path;
  - this means the `0x3113 -> selector 0x4871 -> AUX pin 2` chain is VBIOS/object-table descriptor data, not a runtime synthetic object.
- Interpretation:
  - for a durable Windows-like secondary route, the decisive question is not "did the driver synthesize a second object?";
  - it is whether the real VBIOS/object-info provider exposes class-3 low-byte `0x13` before `PopulateDisplayMapAuxObjectsForHotplugRange()` runs;
  - if the parser is in the display-base signal `13` zero-count state, the VBIOS object list can be suppressed before `0x3113` ever reaches display-map/AUX construction;
  - if the VBIOS object list does expose `0x3113`, the later property13 bit17 skip does not filter it, and it should proceed toward the AUX factory path.
- Kernel-facing implication:
  - Linux logging should distinguish three cases early:
    - real VBIOS/object-info `0x3113` object exists and has descriptor/AUX data;
    - only synthetic/capability-tail objects exist, which is not enough for the secondary tile;
    - VBIOS object-info enumeration is zeroed or missing because the display-base signal path is in the `13` gate.
  - useful plain-vs-OCLP/BootCamp probes:
    - base-pin signal type and display-base signal return value;
    - VBIOS object-info table version and raw object count;
    - per-index object id and provider source before subtracting synthetic tail;
    - synthetic tail count and DAL/register `0x441` value;
    - property6 bit4, because it suppresses AUX factory creation;
    - property13 flags, especially bit17, even though it does not filter `0x3113`;
    - for `0x3113`, descriptor selector/source fields and whether the AUX handle resolves to the expected DDC3/AUX pin.

Extra RE continuation: base-pin signal `13` gate

- Scope:
  - this pass followed the zero-count gate inside `VbiosParserV1/V2GetDisplayObjectCount()`;
  - the key question was what "display-base signal returns `13`" really means and whether it is separate from the AUX-suppression path.
- New/updated IDA anchors:
  - `0x14152DCD0`: renamed `CBasePinGetSignalType`;
  - `0x141467250`: renamed `AdapterServiceInterfaceGetBasePinSignalType`;
  - added `IMAC5K_RE_SIGNAL13_GATE` comments on:
    - `VbiosParserV1/V2GetDisplayObjectCount()`;
    - `ConstructVbiosObjectInfoParserV2()`;
    - `AdapterServiceInterfaceGetBasePinSignalType()`;
    - `AdapterServiceGetBasePinSignalType()`;
    - `CBasePinGetSignalType()`;
    - `ConstructDfpCapabilityNameType82()`;
    - `ConstructDfpCapabilityNameType8D()`.
- Confirmed gate identity:
  - `VbiosParserV2GetDisplayObjectCount()` calls the parser-stored AdapterService interface;
  - the vfunc slot is AdapterService `+0x28`, implemented by `AdapterServiceInterfaceGetBasePinSignalType()`;
  - that thunk calls `AdapterServiceGetBasePinSignalType()`;
  - `AdapterServiceGetBasePinSignalType()` delegates to `CBasePinGetSignalType()`.
  - Therefore the VBIOS parser zero-count gate is directly the CBasePin/base-pin signal type, not a separate display-object policy layer.
- Confirmed `CBasePinGetSignalType()` mapping:
  - it reads capability property index `2` from the pin capability-name object;
  - property `2 == 274` maps to driver signal type `11`;
  - property `2 == 288` maps to driver signal type `12`;
  - otherwise, if base-pin property `6` bit `4` is set, it maps to signal type `13`;
  - otherwise it returns `0`.
- Confirmed capability constructors force the `13` fallback:
  - `ConstructDfpCapabilityNameType82()` normally writes property `2 = 274`;
  - if property6 bit4 is set, the same constructor writes property `2 = 240` instead;
  - `ConstructDfpCapabilityNameType8D()` normally writes property `2 = 288`;
  - if property6 bit4 is set, the same constructor writes property `2 = 240` instead;
  - because `240` is neither `274` nor `288`, `CBasePinGetSignalType()` falls through to the property6-bit4 branch and returns `13`.
- Confirmed double effect of property6 bit4:
  - in `InitializeAdapterServiceForConnector()`, property6 bit4 suppresses creation of the AUX transaction factory at AdapterService `+0x1A8`;
  - in `CBasePinGetSignalType()`, the same property6 bit4 produces base-pin signal type `13`;
  - VBIOS parser V1 and V2 then return zero display objects when that signal type is `13`;
  - VBIOS parser V2 also skips `VBiosHelper` creation when the constructor signal argument is `13`.
- Raw source chain:
  - `InitializeDalFromKmdAdapter()` builds the DAL init structure;
  - it passes display-interface field `+0x140` as the `adapter_init_source`;
  - inside `PopulateDalAdapterInitSourceFlags()`:
    - display-interface `+0x1AC` is `adapter_init_source+0x6C`;
    - display-interface `+0x1B0` is `adapter_init_source+0x70`;
    - `+0x6C` and `+0x70` are zeroed and then synthesized from KMD adapter, core/hardware, and DAL feature predicates.
- Property6 bit4 raw source:
  - `TranslateRawDceFlagsToPinProperty6()` maps raw `adapter_init_source+0x6C` bit `0x100000` to base-pin property6 bit4 / `0x10`;
  - `PopulateDalAdapterInitSourceFlags()` can set raw `+0x6C` bit `0x100000` through multiple paths:
    - core/hardware predicate vfunc `+0x220` plus feature-object test id `300` sets mask `0x100000`;
    - feature-object test id `300` plus core/hardware predicate vfunc `+0x2F8` sets mask `0x140000`;
    - feature-object vfunc `+0x520` sets mask `0x940000`.
  - all three masks include raw `0x100000`, so all three can trigger the property6 bit4 double gate.
- Property13 bit17 remains a different policy bit:
  - `TranslateRawDceFlagsToPinProperty13()` maps raw `adapter_init_source+0x70` bit `0x80000` to base-pin property13 bit17 / `0x20000`;
  - `PopulateDalAdapterInitSourceFlags()` sets that raw bit from the local Apple/internal-display policy byte at display-interface `+0x1D0`;
  - this is the bit that skips low-byte `0x14` / `0x0E` while leaving low-byte `0x13` alone;
  - it does not explain VBIOS object-count suppression by itself.
- Interpretation:
  - property6 bit4 is the hard "secondary route cannot be built normally" gate:
    - it disables the lower AUX factory;
    - it changes the base-pin signal to `13`;
    - it causes the VBIOS object-info parser to enumerate zero objects.
  - property13 bit17 is a softer Apple/internal display policy:
    - it changes which internal class-3 objects are skipped;
    - it preserves `0x3113` from that skip path;
    - it participates in detect scheduling;
    - it does not suppress VBIOS enumeration or AUX factory creation.
- Kernel-facing implication:
  - the first Linux plain-boot question should be whether the equivalent of property6 bit4 is effectively active:
    - if active, Linux may see no real secondary object-table route and no valid AUX backend;
    - if inactive, `0x3113` should be able to come from the VBIOS/object-info table and proceed toward DDC3/AUX pin creation.
  - the most useful instrumentation remains:
    - raw ATOM/VBIOS connector object ids and encoder refs;
    - whether connector `0x3113` is counted before link creation;
    - DDC/AUX line mapping for `0x3113`;
    - whether Linux suppresses or fails lower DDC/AUX construction for that route;
    - whether a platform quirk should force the property6-bit4 outcome to "clear" for the iMac 5K internal pair before normal connector enumeration.

Extra RE continuation: property6 raw `0x100000` sources

- Scope:
  - this pass stayed on the hard property6 bit4 gate;
  - the goal was to separate static feature-table lookup from the runtime adapter capability predicates that actually set raw `adapter_init_source+0x6C` bit `0x100000`.
- New/updated IDA anchors:
  - `0x1400239C0`: renamed `LookupStaticFeatureTableValueById`;
  - added `IMAC5K_RE_PROP6_SOURCE` comments on:
    - `LookupStaticFeatureTableValueById()`;
    - the `adapter->vfunc+0x38` runtime capability-object getter inside `PopulateDalAdapterInitSourceFlags()`;
    - the three property6 raw-bit source branches at `0x14019F5D0..0x14019F663`;
    - `sub_140015910()`, where feature id `300` appears as an inhibitor for a separate feature-`298` path.
- Confirmed static feature-table layout:
  - `LookupStaticFeatureTableValueById(table, id, optional_byte_out)` reads:
    - table `+0x30`: entry count;
    - table `+0x38`: pointer array start;
    - entry `+0x20`: feature id;
    - entry `+0x30`: returned value;
    - entry `+0x44`: optional byte returned to caller;
    - entry `+0x4C`: enabled byte.
  - this helper is used for the stack/init feature table passed as `a3`;
  - it is separate from the runtime capability object returned by adapter vfunc `+0x38`.
- Confirmed runtime capability-object source:
  - `PopulateDalAdapterInitSourceFlags()` calls adapter vfunc `+0x38` once at entry and stores the result in `v8/r14`;
  - that object is then used for runtime virtual predicates:
    - vfunc `+0x8(feature_id)`;
    - vfunc `+0x4D8()`;
    - vfunc `+0x520()`;
    - and other hardware/capability predicates.
- Confirmed exact property6-bit4 source branches:
  - Branch A:
    - display/core object from `DisplayInterface+0x10` via `sub_140021950()` calls vfunc `+0x220()`;
    - runtime capability object calls vfunc `+0x8(300)`;
    - if both are true, Windows ORs raw `0x100000` into `adapter_init_source+0x6C`.
  - Branch B:
    - runtime capability object calls vfunc `+0x8(300)`;
    - display/core object calls vfunc `+0x2F8()`;
    - if both are true, Windows ORs raw `0x140000` into `adapter_init_source+0x6C`.
    - this includes raw `0x100000`, so it also becomes property6 bit4.
  - Branch C:
    - runtime capability object calls vfunc `+0x520()` with no explicit feature-id argument;
    - if true, Windows ORs raw `0x940000` into `adapter_init_source+0x6C`.
    - this also includes raw `0x100000`.
- Feature id `300` context:
  - feature id `300` participates in two of the three hard-gate source branches;
  - a nearby helper, `sub_140015910()`, uses feature id `300` as an inhibitor: the helper only continues into its feature-`298` path when feature `300` is false;
  - this does not give a stable public name for feature `300`, but it argues against treating `300` as "5K enable";
  - in the property6 source path, feature `300` is on the side that can set the hard suppressor bit.
- Interpretation:
  - the raw `0x100000` source is not a direct VBIOS connector-object bit and not only a static registry/override feature table value;
  - it is synthesized from runtime adapter capability predicates plus core/hardware predicates;
  - when it is set, the downstream effect is hostile to the route we need:
    - property6 bit4;
    - base-pin signal type `13`;
    - VBIOS object-info count forced to zero;
    - AUX transaction factory suppressed.
- Kernel-facing implication:
  - for the Linux plain-boot solution, the useful emulation target is the outcome "property6 bit4 clear for the internal 5K pair before object enumeration";
  - chasing feature id `300` as an enable switch is likely the wrong direction;
  - better logs/tests:
    - confirm whether plain Linux has a real `0x3113` object-table connector before any tile synthesis;
    - if missing, compare against the Windows hard-gate outcome: equivalent of raw `0x100000` should be considered active;
    - if present but AUX fails, compare against the AUX-factory half of the same property6 gate;
    - when implementing a quirk, model the final allowed route state directly: keep `0x3113` enumerable and keep DDC3/AUX materialization enabled.

Extra RE continuation: certainty pass - property6 hard gate

- Scope:
  - this pass tightened the property6 bit4 conclusion from "likely hard gate" to a static code-chain proof for the normal Windows DAL route;
  - the question was whether `0x3113` could still be materialized through some normal fallback path after property6 bit4 suppresses the AUX factory and forces base-pin signal type `13`.
- New/updated IDA anchors:
  - added `IMAC5K_RE_CERTAINTY` comments on:
    - `AdapterServiceGetAuxHandleForObjectId()` at `0x141465B5C`;
    - `ConstructDpcdAuxInterfaceForObjectId()` at `0x1415511A1` and `0x1415511AA`;
    - `CreateDpcdAuxInterfaceForObjectId()` at `0x141551470`;
    - `PopulateDisplayMapAuxObjectsForHotplugRange()` at `0x1414B8754`;
    - `VbiosParserV2GetDisplayObjectCount()` at `0x1415AE838`;
    - `VbiosParserV1GetDisplayObjectCount()` at `0x1415A86AD`.
- Proven static chain:
  - `InitializeAdapterServiceForConnector()` leaves AdapterService object `+0x1A8` / interface slot `a1[48]` null when property6 bit4 is set;
  - the same property6 bit4 path makes `CBasePinGetSignalType()` return signal type `13`;
  - both `VbiosParserV2GetDisplayObjectCount()` and `VbiosParserV1GetDisplayObjectCount()` return zero when base-pin signal type is `13`, so the VBIOS/object-info provider cannot expose `0x3113` through its normal object-count/id-by-index interface;
  - `CreateDpcdAuxInterfaceForObjectId()` has one code caller in this IDB: `PopulateDisplayMapAuxObjectsForHotplugRange()` at `0x1414B8754`;
  - `CreateDpcdAuxInterfaceForObjectId()` allocates a DPCD/AUX wrapper and discards it if the wrapper is invalid;
  - `ConstructDpcdAuxInterfaceForObjectId()` asks AdapterService vfunc `+0x170` for a lower AUX handle for the exact object id;
  - if that lower AUX handle is null, the constructor immediately marks the DPCD/AUX wrapper invalid;
  - AdapterService vfunc `+0x170`, implemented by `AdapterServiceGetAuxHandleForObjectId()`, directly calls the AUX factory stored at interface slot `a1[48]` / object `+0x1A8`;
  - there is no EDID-only, TILE-only, or synthetic fake-AUX fallback inside this constructor path.
- Certainty statement:
  - if property6 bit4 is set before AdapterService and display-object construction, the normal Windows DAL route cannot produce a durable AUX-backed `0x3113`;
  - it is blocked on both sides:
    - enumeration side: VBIOS object count becomes zero through signal type `13`;
    - AUX side: the lower AUX factory is null, so object-specific DPCD/AUX wrapper construction fails.
- What this removes:
  - `0x3113` is not expected to appear later from the synthetic tail path;
  - `0x3113` is not expected to be rescued by the DPCD wrapper constructor after the AUX factory is suppressed;
  - feature id `300` should not be treated as a "turn 5K on" knob in this path, because its observed property6 branches feed the hard suppressor outcome.
- Remaining runtime unknown:
  - static RE proves the consequence of property6 bit4, but not whether a particular boot sets it on the iMac hardware;
  - to eliminate that last uncertainty we still need either:
    - a runtime probe/log showing whether plain/OCLP/BootCamp construction reaches VBIOS object `0x3113` with a valid lower AUX handle; or
    - deeper RE of the runtime capability object predicates behind vfunc `+0x8(300)` and vfunc `+0x520()`.
- Kernel-facing implication:
  - the Linux target should be framed as "make the internal 5K secondary route look like the property6-bit4-clear state before connector enumeration";
  - that means preserving or constructing both parts together:
    - object-table route for `0x3113`;
    - valid AUX/DDC backend for `0x3113`;
  - simply forcing a TILE EDID or late secondary connector is unlikely to match Windows if the lower AUX/backend lifetime was already lost.

Extra RE continuation: runtime caps predicates behind the property6 gate

- Scope:
  - this pass attacked the remaining ambiguity in the three raw `0x100000` sources:
    - runtime caps vfunc `+0x8(feature_id)`;
    - runtime caps vfunc `+0x520()`.
  - the goal was not to assign public AMD feature names, but to determine whether these are opaque "maybe 5K enable" predicates or whether their local logic can be proven.
- New/updated IDA anchors:
  - `0x140015910`: renamed `RuntimeCapsCheckFeature298PathWhen300Clear`;
  - `0x140018C80`: renamed `RuntimeCapsFeatureIdTest_StoreBacked`;
  - `0x14043E9D0`: renamed `RuntimeFeatureStoreProbeIdSideEffect`;
  - added `IMAC5K_RE_CERTAINTY` comments at:
    - `RuntimeCapsCheckFeature298PathWhen300Clear()` feature-300 test at `0x14001594B`;
    - `PopulateDalAdapterInitSourceFlags()` vfunc `+0x520` callsite at `0x14019F65A`;
    - `RuntimeCapsFeatureIdTest_StoreBacked()` feature-store probe at `0x140018CD0`;
    - `RuntimeFeatureStoreProbeIdSideEffect()` return at `0x14043EA21`.
- Confirmed vfunc `+0x520()` meaning in the observed runtime-caps vtable family:
  - the `+0x520` slot points to `RuntimeCapsCheckFeature298PathWhen300Clear()`;
  - that helper first checks feature id `300`;
  - if feature `300` is true, it returns false immediately;
  - only when feature `300` is false does it check feature id `298`;
  - then it also requires a hardware/ASIC-family predicate through `sub_14003C5F0()` and related runtime caps methods.
- Consequence for the third property6 raw source:
  - the raw `0x940000` branch in `PopulateDalAdapterInitSourceFlags()` is no longer opaque;
  - it is the "feature 298 path when feature 300 is clear, plus hardware predicate" path;
  - because `0x940000` contains `0x100000`, this path still feeds property6 bit4 and the hard signal13/AUX-suppression outcome.
- Confirmed vfunc `+0x8(feature_id)` store-backed variant:
  - in the same vtable family, slot `+0x8` points to `RuntimeCapsFeatureIdTest_StoreBacked()`;
  - it passes the requested feature id into `RuntimeFeatureStoreProbeIdSideEffect()`;
  - `RuntimeFeatureStoreProbeIdSideEffect()` returns `0` on all visible paths in this IDB;
  - therefore this store-backed vfunc variant returns true for the visible local logic;
  - the feature id still matters as a side-effect/accounting input to the lower helper, but not as a reject condition in the visible return value.
- Updated interpretation of the three property6 raw sources:
  - Branch A:
    - hardware/display-core predicate vfunc `+0x220`;
    - feature-test vfunc `+0x8(300)`;
    - sets raw `0x100000`.
  - Branch B:
    - feature-test vfunc `+0x8(300)`;
    - hardware/display-core predicate vfunc `+0x2F8`;
    - sets raw `0x140000`.
  - Branch C:
    - runtime caps vfunc `+0x520()`;
    - resolved locally as feature `298` true while feature `300` false, plus hardware predicate;
    - sets raw `0x940000`.
- Certainty gained:
  - vfunc `+0x520()` is not an unknown Apple/OCLP/5K-specific firmware trigger in this driver path;
  - it is another route into the same hard suppressor, guarded by feature `298` with feature `300` clear;
  - all three sources still converge on the same Linux-relevant outcome: raw `0x100000` set means property6 bit4 set, which blocks the normal `0x3113` enumeration/AUX route.
- Remaining uncertainty:
  - we still do not have public names for feature ids `298` and `300`;
  - we also have not proven which runtime-caps vtable variant the iMac path uses at runtime;
  - however, neither uncertainty weakens the main bring-up conclusion: the desired state is not "enable feature 298/300", but "avoid the raw `0x100000` / property6-bit4 suppressor for the internal 5K route".

Extra RE continuation: split the two halves of the property6 source predicate

- Scope:
  - this pass focused on removing ambiguity around the two different objects used by `PopulateDalAdapterInitSourceFlags()`:
    - the local KMD adapter context used for runtime feature/caps predicates;
    - the display/root services object used for the hardware/display-core predicates.
- New/updated IDA anchors:
  - `0x140040790`: renamed `CreateKmdAdapterContextForStartDevice`;
  - `0x1400423C0`: renamed `KmdAdapterContextGetRuntimeCapsInterface`;
  - `0x140042450`: renamed `KmdAdapterContextGetDeviceServicesObject`;
  - `0x1401A4180`: renamed `BuildDalDisplayInterfaceAndCoreServicesPair`;
  - added `IMAC5K_RE_CERTAINTY` comments at:
    - the StartDevice callsite to `BuildDalDisplayInterfaceAndCoreServicesPair()` at `0x14002A021`;
    - the local adapter-context constructor vtable store at `0x1400407DC`;
    - `KmdAdapterContextGetRuntimeCapsInterface()` at `0x1400423C0`;
    - the `PopulateDalAdapterInitSourceFlags()` runtime-caps getter at `0x14019F1C7`;
    - the display/root object predicate callsites at `0x14019F5D0` and `0x14019F628`.
- Confirmed StartDevice argument split:
  - `sub_141344960()` validates the dispatcher/root object and calls `sub_140029170()`;
  - inside `sub_140029170()`, `CreateKmdAdapterContextForStartDevice()` creates the local adapter context in `r15` with vtable `off_1404F4B60`;
  - later the call at `0x14002A021` is:
    - out pair pointer;
    - `dispatcher+8` root/display-services object;
    - local adapter context `r15`;
    - boot/start args `rsi`;
    - `dispatcher+0xE0`;
    - status pointer.
- Confirmed DAL display-interface construction:
  - `BuildDalDisplayInterfaceAndCoreServicesPair()` calls `CreateDisplayInterfaceAndInitializeDal(dispatcher+8, r15, rsi, dispatcher+0xE0, status_ptr)`;
  - `ConstructDisplayInterfaceCommon()` calls the common constructor that stores its second argument at display-interface `+0x10`;
  - therefore `sub_140021950(display_interface)` returns `dispatcher+8`, not `r15`.
- Confirmed runtime-caps source:
  - the object passed as `a2` to `PopulateDalAdapterInitSourceFlags()` is the local adapter context `r15`;
  - its vtable is `off_1404F4B60`;
  - vfunc `+0x38` is `KmdAdapterContextGetRuntimeCapsInterface()`;
  - that getter returns `*(ctx+0xAC8)+0x20` or null;
  - this returned interface is the object used for:
    - vfunc `+0x8(feature id)`;
    - feature id `300`;
    - feature id `298`;
    - vfunc `+0x520()`.
- Corrected interpretation of the property6 raw branches:
  - Branch A is a conjunction of two different objects:
    - `dispatcher+8` root/display-services vfunc `+0x220`;
    - runtime-caps interface vfunc `+0x8(300)`;
    - together they set raw `0x100000`.
  - Branch B is also split:
    - runtime-caps interface vfunc `+0x8(300)`;
    - `dispatcher+8` root/display-services vfunc `+0x2F8`;
    - together they set raw `0x140000`.
  - Branch C remains entirely on the runtime-caps side:
    - runtime-caps interface vfunc `+0x520()`;
    - resolved previously as feature `298` true while feature `300` false plus hardware predicate;
    - sets raw `0x940000`.
- Certainty gained:
  - feature id `300` is definitely tested through the local adapter context's runtime-caps interface, not through the root/display-services object;
  - the hardware/display-core predicates `+0x220` and `+0x2F8` are definitely on `dispatcher+8`, not on the local adapter context;
  - this prevents a bad future simplification where all three raw `0x100000` sources are attributed to one feature object.
- Remaining RE target:
  - the next uncertainty is now narrower:
    - identify the concrete runtime class/vtable behind `dispatcher+8` for the iMac path;
    - then decode its vfunc `+0x220` and `+0x2F8` implementations;
  - this is the right place to continue if we want to know exactly which hardware/display-core condition combines with feature `300` to set the property6 hard suppressor.

Extra RE continuation: concrete root-services vtable behind dispatcher+8

- Scope:
  - this pass continued from the previous remaining target:
    - identify the concrete object stored at `dispatcher/PSID context +0x8`;
    - resolve the root/display-services vfuncs `+0x220` and `+0x2F8`;
    - determine whether those predicates are still opaque hardware checks or can be named with local proof.
- New/updated IDA anchors:
  - `0x14133F010`: renamed `AtiAddDevice_CreatePsidContext`;
  - `0x14003C690`: renamed `InitializePsidContextAndRootServices`;
  - `0x1400767E0`: renamed `AllocateKmdRootServicesObject`;
  - `0x140075EA0`: renamed `ConstructKmdRootServicesObject`;
  - `0x140080320`: renamed `QueryWindowsVersionAndSetGlobalCode`;
  - `0x14007E5C0`: renamed `RootServicesIsWindows7`;
  - `0x14007E390`: renamed `RootServicesVirtualDalRuntimePolicyPredicate`;
  - `0x140082500`: renamed `RootServicesSetRuntimeCapsContainer`;
  - `0x140079C80`: renamed `RootServicesGetRuntimeCapsContainer`;
  - added `IMAC5K_RE_CERTAINTY` comments at:
    - PSID magic write `0x14133F240`;
    - root-services allocation/store in `InitializePsidContextAndRootServices()` at `0x14003C6BA` and `0x14003C6D5`;
    - root-services vtable assignment at `0x140075ED3`;
    - vtable slots `off_1404FEAB8 + 0x220` and `+0x2F8`;
    - OS-code writes in `QueryWindowsVersionAndSetGlobalCode()`;
    - static feature id 20 / `KMD_EnableVirtualDalSupport` table entry.
- Confirmed dispatcher+8 object construction:
  - `AtiAddDevice_CreatePsidContext()` allocates the PSID/dispatcher context and writes magic `0x44495350` at `0x14133F240`;
  - `InitializePsidContextAndRootServices()` allocates a 33608-byte root KMD services object through `AllocateKmdRootServicesObject()`;
  - `ConstructKmdRootServicesObject(root, DeviceObject)` sets the root object's vtable to `off_1404FEAB8`;
  - `InitializePsidContextAndRootServices()` stores the constructed root object at `PSID context +0x8`;
  - therefore the `dispatcher+8` object used by `PopulateDalAdapterInitSourceFlags()` is concretely the `off_1404FEAB8` root KMD services object.
- Resolved Branch A root predicate:
  - vtable `off_1404FEAB8 + 0x220` points to `RootServicesIsWindows7()`;
  - `RootServicesIsWindows7()` is exactly:
    - compare global OS code `qword_141258510` low dword against `0x610`;
    - return true only on equality.
  - `QueryWindowsVersionAndSetGlobalCode()` sets:
    - Windows 7 / version 6.1 -> `0x610`;
    - Windows 8 / version 6.2 -> `0x620`;
    - Windows 8.1 / version 6.3 -> `0x630`;
    - Windows 10 -> `0xA00`;
    - Windows Server 2016 -> `0xA01`.
  - conclusion:
    - Branch A is not a generic iMac/5K hardware predicate;
    - Branch A is `Windows 7 && runtime feature300`;
    - on normal Windows 10 BootCamp, this root predicate is false.
- Resolved Branch B root predicate:
  - vtable `off_1404FEAB8 + 0x2F8` points to `RootServicesVirtualDalRuntimePolicyPredicate()`;
  - its return condition is:
    - true if root flag `+0x7E9C` bit `0x100` is set, static feature id `20` equals `1`, and runtime feature id `340` is true;
    - otherwise true if runtime-caps vfunc `+0x520()` is true and either the low byte of root flag `+0x7E9C` is not `4` or static feature id `20` is nonzero;
    - otherwise true if root byte `+0x8340` is nonzero.
  - static feature id `20` is not anonymous:
    - the built-in table entry points to the wide string `KMD_EnableVirtualDalSupport`;
    - its built-in value is `0xFFFFFFFF` / `-1`;
    - therefore the strict `== 1` arm requires an explicit override to become true.
  - the vfunc `+0x520()` arm calls through `root +0x370 +0x20`;
    - the root object has paired setter/getter methods for `+0x370`;
    - in the decoded runtime-caps family, vfunc `+0x520()` is `RuntimeCapsCheckFeature298PathWhen300Clear()`;
    - follow-up tracing closed the assignment path:
      - StartDevice calls KMD adapter-context vfunc `+0x2B0` at `0x140029728`;
      - this resolves to `KmdAdapterContextCreateRuntimeCapsContainer()`;
      - `BindRuntimeCapsContainerToKmdContextAndRoot()` stores that returned container at `ctx+0xAC8`;
      - the same bind routine calls root vfunc `+0x668`, `RootServicesSetRuntimeCapsContainer(root, container)`, storing it at `root+0x370`;
      - therefore `ctx+0xAC8+0x20` and `root+0x370+0x20` are the same runtime-caps interface.
  - important consequence:
    - Branch B's outer test requires runtime feature id `300`;
    - the inner `+0x520()` arm is the same `RuntimeCapsCheckFeature298PathWhen300Clear()` family, which returns false when feature id `300` is true;
    - therefore, under stable runtime-caps state, Branch B's `+0x520()` sub-arm cannot be the reason Branch B passes;
    - Branch B can still pass through the strict Virtual-DAL arm (`KMD_EnableVirtualDalSupport == 1`, runtime feature `340`, root flag bit `0x100`) or through root override byte `+0x8340`.
- Updated interpretation of the three raw property6 sources:
  - Branch A:
    - `RootServicesIsWindows7()`;
    - runtime feature id `300`;
    - sets raw `0x100000`.
  - Branch B:
    - runtime feature id `300`;
    - `RootServicesVirtualDalRuntimePolicyPredicate()`, mostly gated by `KMD_EnableVirtualDalSupport`, runtime feature id `340`, the runtime-caps `+0x520()` path, or root override byte `+0x8340`;
    - sets raw `0x140000`.
  - Branch C:
    - runtime-caps vfunc `+0x520()`;
    - previously decoded as feature `298` true while feature `300` is clear, plus hardware predicate;
    - sets raw `0x940000`.
- Certainty gained:
  - `dispatcher+8` is no longer an unknown display/core object;
  - the vtable and the two property6-relevant root predicates are concretely resolved;
  - the first predicate is OS-version policy, not panel detection;
  - the second predicate is Virtual DAL/runtime-caps policy, not a direct 5K tile-pair initialization check;
  - the root `+0x370` runtime-caps container provenance is now proven and matches the local adapter context `+0xAC8` runtime-caps container.
- Remaining uncertainty:
  - public names for runtime feature ids `298`, `300`, and `340` are still unknown;
  - even with that naming gap, the Linux-relevant outcome is clearer: Windows' durable `0x3113` path is protected by avoiding these raw `0x100000` property6 suppressor branches, not by executing a hidden root-services 5K wake sequence here.

Extra RE continuation: Linux-implementation value of Branch-B inputs

- Scope:
  - this was a short pass to decide whether more RE can still help Linux implementation directly;
  - the target was Branch B's remaining root-side inputs:
    - root flag `+0x7E9C` bit `0x100`;
    - root override byte `+0x8340`.
- New/updated IDA anchors:
  - `0x140082B00`: renamed `RootServicesSetVirtualCpuFlagBit8`;
  - `0x140082AF0`: renamed `RootServicesSetVirtualDalOverrideByte`;
  - `0x1400826E0`: renamed `RootServicesSetFlagBit9`;
  - `0x140082870`: renamed `RootServicesSetFlagBit10`;
  - `0x140040F60`: renamed `KmdSystemConfigurationInitCpuInfo`;
  - added `IMAC5K_RE_LINUX_IMPL` comments at:
    - the Branch-B predicate input site;
    - the root bit8 setter;
    - the CPU-info check that drives it;
    - the `+0x8340` override setter.
- Confirmed root flag bit8 source:
  - `RootServicesVirtualDalRuntimePolicyPredicate()` tests root `+0x7E9C` bit `0x100`;
  - root vtable slot `+0x530` points to `RootServicesSetVirtualCpuFlagBit8()`;
  - `KmdAdapterContextInitialize()` calls this setter when local adapter-context CPU/system flag `+0xBD8` has bit `0x08000000` set;
  - `KmdSystemConfigurationInitCpuInfo()` logs that same flag as `CPU VIRTUAL`.
- Consequence:
  - the strict Branch-B arm is now:
    - `CPU VIRTUAL`;
    - `KMD_EnableVirtualDalSupport == 1`;
    - runtime feature id `340`;
  - this looks like a VM/Virtual-DAL policy path, not a normal bare-metal iMac 5K bring-up path.
- Confirmed root override byte:
  - root vtable slot `+0x300` points to `RootServicesSetVirtualDalOverrideByte()`;
  - the predicate returns true unconditionally when root byte `+0x8340` is nonzero;
  - this pass did not find a direct non-vtable code caller in the normal StartDevice path.
- Linux implementation value:
  - this further lowers the odds that Branch B encodes the Windows 5K mechanism we need to reproduce;
  - for bare-metal Linux on the iMac, the more useful target remains avoiding the property6 hard suppressor and preserving/constructing the durable `0x3113` secondary route.
- RE still worth doing:
  - trace public/source names or table origins for runtime feature ids `298`, `300`, and `340`;
  - compare old 4K-only vs first 5K-enabled BootCamp driver around the property6/object-enumeration path;
  - trace the provider/object-info route that yields `0x3113` when the suppressor is avoided;
  - trace the Windows false-disconnect/secondary-preservation path enough to translate it into a Linux connector-lifetime policy.

Extra RE continuation: BootCamp-named DAL option and feature-id naming pass

- Scope:
  - this pass continued from the property6 suppressor branch and asked whether the remaining runtime feature ids or BootCamp-named configuration strings identify a hidden Windows-only 5K enable path;
  - the result is useful mostly as a negative control: the only explicit `BootCamp` string found in the modern BootCamp AMD driver is real, but its decoded local effects do not create or wake the `0x3113` secondary route.
- New/updated IDA anchors:
  - `0x141B1EE70`: renamed `QueryDalConfigValueByName`;
  - `0x141A379D0`: renamed `CreateTimingServiceInterface`;
  - `0x141AB98B0`: renamed `ConstructTimingServiceAndQueryBootCampPlatform`;
  - `0x141AB7B70`: renamed `TimingServiceRemoveTimingsBySize`;
  - `0x144C4E008`: renamed `g_KmdBootCampPlatformOption`;
  - added `IMAC5K_RE_BOOTCAMP_OPTION` comments at:
    - `ConstructDal2CoreAndServices()` query of `KMD_BootCampPlatform`;
    - `ConstructTimingServiceAndQueryBootCampPlatform()` cached-global query;
    - the `g_KmdBootCampPlatformOption` timing-conversion use site;
    - the exact width/height match in `TimingServiceRemoveTimingsBySize()`.
- Confirmed `KMD_BootCampPlatform` effects:
  - `ConstructDal2CoreAndServices()` queries named config `KMD_BootCampPlatform`;
  - if the query succeeds and returns nonzero, it calls TimingService vfunc `+0x00` with:
    - width `0x1400` / `5120`;
    - height `0x870` / `2160`.
  - that vfunc is `TimingServiceRemoveTimingsBySize()`;
  - it iterates the timing table and removes entries whose first two dwords match the requested width/height;
  - therefore this BootCamp-named DAL-core effect removes `5120x2160` timing, not the iMac 5K tile timing `5120x2880` or half-tile timing `2560x2880`.
- Confirmed cached global effect:
  - `ConstructTimingServiceAndQueryBootCampPlatform()` also queries `KMD_BootCampPlatform` into `g_KmdBootCampPlatformOption`;
  - if the query fails, it forces the global to zero;
  - the only direct xref consumer found in this pass is the timing-conversion helper at `0x141A376F0`;
  - there, the BootCamp flag broadens a timing-standard branch for timing type `14`, but no `0x3113`, DPCD, AUX, DisplayID TILE, or `0x4F1` behavior is visible in that branch.
- Local package search:
  - `rg -a "KMD_BootCampPlatform|BootCampPlatform" BootCamp` found no hit in the unpacked local BootCamp packages;
  - the string is present in the modern `B416406/amdkmdag.sys` IDB/binary, but this pass did not find an INF/driver-package text setting for it.
- Runtime feature-id naming result:
  - immediate searches for `298`, `300`, and `340` produced many false positives:
    - `298` appears as a struct offset/copy length and in unrelated HDCP/graphics blocks;
    - `300` is too common as an offset/size literal;
    - `340` appears in HDMI/DPCD/timing code as a constant, including a timing divisor/default, not necessarily as runtime feature id `340`.
  - the meaningful property6 uses remain the previously decoded ones:
    - feature id `300` in Branch A/B;
    - feature id `298` inside `RuntimeCapsCheckFeature298PathWhen300Clear()`;
    - feature id `340` only inside the strict Virtual-DAL arm of `RootServicesVirtualDalRuntimePolicyPredicate()`.
  - no public/source string name for runtime feature ids `298`, `300`, or `340` was found in this IDB pass.
- Interpretation:
  - `KMD_BootCampPlatform` is real BootCamp policy, but it is not the missing internal-panel route constructor;
  - its decoded effects are timing-list policy, not CoreEG2/GOP replay, `0x3113` object enumeration, AUX factory construction, source-DPCD, ASSR, or Apple/tiled `0x4F1`;
  - this reduces the chance that the plain-vs-BootCamp 5K split is hidden behind a single named KMD BootCamp registry key in the modern driver.
- Linux implementation value:
  - do not model `KMD_BootCampPlatform` as a Linux 5K bring-up knob;
  - keep the Stage 2 Linux target outcome-based:
    - make the internal 5K route look like the property6-bit4-clear state before connector enumeration;
    - ensure the real VBIOS/object-info `0x3113` route reaches DDC3/AUX pin creation;
    - then reuse the already-proven Apple/tiled EDID/DisplayID capability, ASSR, source-DPCD, and `0x4F1` machinery.

Extra RE continuation: provider/object-info route that yields `0x3113`

- Scope:
  - this pass followed the positive route after the property6 hard suppressor is avoided;
  - the question was exactly how the provider/object-info stack yields packed AMD object id `0x3113`, and which fields must exist for the route to become AUX-backed.
- New/updated IDA anchors:
  - `0x1415A9B10`: renamed `VbiosV2DecodeConnectorTypeFromRecordLowByte`;
  - added `IMAC5K_RE_OBJECT_ROUTE` comments on:
    - `VbiosV2DecodeObjectIdFromRecordRef()` at `0x1415A9DB0`;
    - class decode at `0x1415A9D41`;
    - instance decode at `0x1415A9C61`;
    - connector-type low-byte decode at `0x1415A9B10` and case `0x13` at `0x1415A9BBC`;
    - V2 id-by-index decode at `0x1415AE73E`;
    - V2 count gate at `0x1415AE896`;
    - lookup-by-id at `0x1415AC490`;
    - object descriptor lookup at `0x1415AE3D4` and encoder-record parse at `0x1415AE491`;
    - descriptor AUX fields at `0x1415AC3A2` and `0x1415AC3B8`;
    - provider dispatch at `0x14152C936`, `0x141466844`, and create-links object-id consumption at `0x141B6D3E2`;
    - link AUX creation at `0x141B6D84E`;
    - final per-object AUX materialization at `0x141465B5C`.
- Provider dispatch chain:
  - `CreateLinksFromBiosObjectTable()` gets the BIOS/display object service from display-core context `+0x340 -> +0x58`;
  - it calls that service's count slot to determine physical connector count, then calls its id-by-index slot for each physical connector index;
  - the service is the AdapterService provider stack:
    - `AdapterServiceGetTotalDisplayObjectCount()` combines ObjectInfo/VBIOS count, synthetic-tail count, and DisplayDescriptorCapabilityFlags count;
    - `AdapterServiceGetDisplayObjectIdByIndex()` resolves in this order:
      - ObjectInfoDescriptorAdapter / VBIOS object-info provider first;
      - DisplayDescriptorCapabilityFlags provider second;
      - synthetic tail last.
  - the capability provider only builds low-byte `0x16/0x17`, and the synthetic tail only builds low-byte `0x18` / class-2 low-byte `0x26`;
  - therefore a real `0x3113` can only come from the first bucket, the ObjectInfoDescriptorAdapter wrapping the VBIOS parser.
- VBIOS parser object id path:
  - `ObjectInfoDescriptorAdapterGetDisplayObjectIdByIndex()` is a thin delegate to the parser vfunc `+0x30`;
  - `VbiosParserV2GetDisplayObjectCount()` returns zero if base-pin signal type is `13`; otherwise it walks the VBIOS object-info table:
    - entry count is byte `objectInfoTable + 0x06`;
    - entries are 16 bytes;
    - an entry is counted when its object-ref word and encoder-ref word are present.
  - `VbiosParserV2GetDisplayObjectIdByIndex()` reads the object-ref word from the selected 16-byte entry, shown in code as `objectInfoTable + 0x08 + 16*i`;
  - `VbiosV2DecodeObjectIdFromRecordRef()` decodes that 16-bit reference:
    - class = bits `14:12`, with value `3` meaning class-3 display connector;
    - instance = bits `10:8`, so `1` means connector instance 1;
    - connector type = low byte for class `3`; low byte `0x13` returns type `0x13`.
  - `ObjectIdBuildCoreFields()` packs those fields as:
    - low byte/type at bits `7:0`;
    - instance nibble at bits `11:8`;
    - class nibble at bits `15:12`.
  - therefore a VBIOS record ref shaped as `class=3, instance=1, type=0x13` yields packed object id `0x3113` directly. There is no later magic translation that invents the `0x13`.
- Descriptor/AUX route path:
  - `VbiosParserV2GetObjectDescriptor()` resolves the same packed object id through `VbiosV2FindObjectRecordByObjectId()`;
  - for class-3 ids such as `0x3113`, that lookup compares the requested object id against the decoded object-ref word of each 16-byte VBIOS object entry;
  - once it finds the record, it walks the record's encoder-record list until it finds record type `1`;
  - `ParseVbiosV2EncoderRecordToObjectDescriptor()` then matches encoder byte `record+2` against the GPIO/I2C selector table referenced by parser field `+0x6C`;
  - it writes:
    - descriptor `+0x1C` = AUX/DDC selector from the matching selector-table row, e.g. the iMac secondary DDC3/AUX selector path such as `0x4871`;
    - descriptor `+0x3C` = source/index byte from the same row.
  - `AdapterServiceGetAuxHandleForObjectId()` consumes that descriptor:
    - reads descriptor `+0x1C` selector;
    - turns descriptor `+0x3C` into `source_mask = 1 << value`;
    - calls the AdapterService AUX factory at object `+0x1A8`.
- Link construction path:
  - `ConstructLegacyConnectorDisplayEntry()` receives the packed object id from the BIOS object service;
  - if the low byte is `0x13`, it maps the link to runtime signal `32`;
  - if the low byte is `0x14`, it maps the link to runtime signal `128`;
  - for `0x3113`, Windows then creates the normal per-link DDC/AUX service with `CreateDceAuxDdcService()` before building the link encoder.
- Certainty gained:
  - `0x3113` is not synthesized by DisplayDescriptorCapabilityFlags, the synthetic tail, BootCamp timing policy, or a TILE parser;
  - the packed id is the direct decode of a VBIOS object-info record reference;
  - the durable route requires two independent pieces to survive:
    - the object-info table must expose the `0x3113` object-ref entry before the signal-13/property6 zero-count gate;
    - the descriptor parse must resolve the selector/source fields so the AUX factory can create the per-object AUX handle.
- Linux implementation value:
  - Stage 2 logging should happen before any Linux tile synthesis:
    - raw VBIOS object-info revision;
    - raw object-info entry count;
    - per-entry object-ref word, encoder-ref word, and decoded packed object id;
    - whether an entry decodes to `0x3113`;
    - selector/source fields for the decoded `0x3113`;
    - whether Linux resolves that selector to the expected DDC3/AUX channel and builds a valid AUX backend.
  - if plain boot lacks the raw `0x3113` entry, the kernel cannot reproduce Windows by only forcing EDID or TILE metadata later;
  - if the raw `0x3113` entry exists but descriptor/AUX materialization fails, the target is selector/source/AUX-factory handling, not panel mode selection.

Extra RE continuation: Windows false-disconnect path as Linux connector lifetime policy

- Scope:
  - this pass mapped the Windows false-disconnect preservation path into a concrete Linux connector/sink lifetime rule;
  - target question: where exactly does Windows stop a transient secondary disconnect from destroying the live 5K tile route, and what is the closest Linux analogue?
- New/updated IDA anchors:
  - added `IMAC5K_RE_LIFETIME_POLICY` comments on:
    - `DetectDisplayAndApplyApple5kOverrides()` false-disconnect gate at `0x1414B4174`;
    - result correction writes at `0x1414B417E` and `0x1414B418A`;
    - `HandleDisplayDetectionResultAndLiveState()` changed/apply gates at `0x1414A8D20` and `0x1414A8DD9`;
    - `ApplyDetectionResultAndUpdateLiveDisplayObject()` -> `UpdateDisplaySinkConnectionAndLiveAuxState()` boundary at `0x1414A84CA`;
    - `UpdateDisplaySinkConnectionAndLiveAuxState()` live-record update call at `0x1414A7E21`;
    - `UpdateDetectionRecordLiveDisplayObject()` record `+0x18` clear at `0x1414B3B45`;
    - display-map release helper at `0x1414AE020`;
    - display-map refcount byte clear at `0x1414ABD94`;
    - low-byte `0x13` detection-record policy/candidate insertion at `0x1414B4544` and `0x1414B4619`;
    - alternate direct live-state caller at `0x141430DBC`;
    - record notification sweep at `0x1414B4409`.
- Confirmed Windows lifetime boundary:
  - the false-disconnect decision is made while the 120-byte detection result is still mutable;
  - the gate requires all of:
    - connected-pin descriptor byte `+6` bit `3` set, matching the Apple/tiled `0x3B` family;
    - the current display object still reports connected through vfunc `+0x258`;
    - the fresh detection result says disconnected at result byte `+0x72`;
    - current Windows signal type is `11`, the local DP/SST signal value.
  - when those are true, Windows writes:
    - result byte `+0x72 = 1`;
    - result byte `+0x70 = 0`.
  - that makes `HandleDisplayDetectionResultAndLiveState()` compute `changed = false`;
  - because `changed` is false and the rescan byte is clear, it does not call `ApplyDetectionResultAndUpdateLiveDisplayObject()`;
  - because that call is skipped, `UpdateDisplaySinkConnectionAndLiveAuxState()` is skipped;
  - because that call is skipped, `UpdateDetectionRecordLiveDisplayObject()` cannot clear detection-record `+0x18`.
- Confirmed what would be destroyed without the correction:
  - `UpdateDetectionRecordLiveDisplayObject()` finds the 104-byte record for the object's packed id;
  - if the display object is connected, it stores the live display object pointer at record `+0x18`;
  - if the display object reports disconnected, it calls `NotifyDetectionRecordDisconnected()` and clears record `+0x18`;
  - this is the direct Windows analogue of losing the secondary connector's real live sink/object, not just losing a DRM TILE blob.
- Important distinction:
  - Windows still runs `ReleaseDisplayMapEntriesAfterDetectPass()` after the false-disconnect correction;
  - that path decrements display-map entry refcounts at entry `+0x0C` and clears byte `+0x12` when the refcount reaches zero;
  - it does not clear the detection-record live object pointer at record `+0x18`;
  - therefore the policy is selective:
    - preserve the already-live secondary object through a known transient detect loss;
    - do not globally pin every display-map entry, every stream interface, or every AUX reference forever.
- Why this matters for Linux:
  - Linux should not model this as "if the secondary disappears, create a fake sink";
  - the Windows-shaped behavior is "if the exact proven secondary route reports a transient disconnect, do not let that disconnected result replace the existing live sink state";
  - in amdgpu terms, the closest boundary is before the disconnected detect path releases `aconnector->dc_sink`, clears DRM EDID/TILE state, or lets `dc_link->local_sink` / `dpcd_sink_count` collapse for the internal secondary route.
- Proposed Linux connector lifetime policy:
  - only enable on the iMac 5K tiled-display quirk path;
  - only target the secondary internal route, not arbitrary DP connectors;
  - require route identity proof:
    - packed object id equivalent to Windows `0x3113`, or the Linux-decoded route that maps to the same physical object;
    - expected DDC/AUX selector, currently the DDC3/AUX pin path seen in the Windows route;
    - expected transmitter/encoder pairing for the secondary internal tile;
    - not an MST root or external DP connector.
  - require prior real-route proof before suppression:
    - a real non-emulated sink was observed earlier on the secondary; or
    - preserved EDID/TILE metadata was captured from a real secondary sink; or
    - concrete AUX/DPCD evidence exists, e.g. secondary source-DPCD programming, ASSR/DPCD `0x10A`, or panel/private `0x4F1` evidence.
  - require the current event shape to match Windows:
    - the fresh detection result is disconnected or sinkless;
    - the connector still has a real previous sink/object to preserve;
    - route identity still matches the internal secondary;
    - this is not a user-visible external hot-unplug path.
  - on suppression:
    - keep the existing `aconnector->dc_sink` and/or restore `dc_link->local_sink` from it before generic disconnect cleanup runs;
    - keep a nonzero `dpcd_sink_count` for the route while the paired 5K commit is being assembled;
    - keep/restage the real EDID and TILE state from the prior secondary snapshot;
    - mark/log the event as false-disconnect suppression, not as a new detection success;
    - continue normal mode/freesync/subconnector updates that do not destroy the live object.
  - clear preservation when:
    - the connector no longer matches the secondary route identity;
    - the primary/internal tile peer disappears or the quirk is disabled;
    - a full disable of both tiles is intentional;
    - repeated AUX/DPCD probes fail after the paired-commit window;
    - suspend/GPU reset causes the saved physical-route proof to become stale.
- Assessment of current Linux direction:
  - the current `amdgpu_dm_imac5k_should_suppress_secondary_false_disconnect()` shape is conceptually aligned with Windows:
    - it checks quirk enabled;
    - requires `imac5k_secondary_head`;
    - verifies the Windows-like route;
    - requires a disconnected fresh detect result;
    - requires a real live previous sink;
    - rejects MST root;
    - requires AUX/DPCD evidence.
  - the current suppression block in `amdgpu_dm_update_connector_after_detect()` is also at the right layer because it runs before the generic disconnected branch releases `aconnector->dc_sink` and clears EDID state;
  - what still deserves care in implementation/testing:
    - do not let `dc_link_detect()` or a lower DC helper clear `dc_link->local_sink` too early without restoring from the preserved real sink;
    - do not treat preserved EDID/TILE alone as sufficient if route identity and AUX evidence were never proven;
    - do not hold the preservation state forever after repeated AUX failure or a deliberate full modeset disable.
- Bottom line:
  - Windows preserves the secondary by preventing a known false disconnected result from becoming a live-object teardown;
  - the Linux policy should be a narrow detect-stage lifetime guard for the proven `0x3113` secondary route, with explicit proof and explicit expiration.

### RE continuation: Linux/Windows `0x3113` AUX route construction side by side

Scope note:
- This pass did not add new live IDA names/comments because the IDA MCP tool namespace was not exposed in this Codex turn.
- The Windows side below is therefore a synthesis of the earlier IDA-anchored notes in this file.
- The Linux side was checked directly against the current `linux-imac-5k` source and the plain/OCLP boot logs.

Windows route construction model:
- The secondary tile route starts from the real AMD/VBIOS display object `0x3113`; earlier RE found no evidence that Windows synthesizes this object out of GOP replay state.
- `ConstructDpcdAuxInterfaceForObjectId` asks the adapter service for the lower AUX handle for that exact object id.
- The adapter-service connector descriptor carries:
  - selector/register `+0x1C = 0x4871` for secondary `0x3113`, matching DCE DDC3 A / `mmDC_GPIO_DDC3_A`;
  - selector/register `+0x1C = 0x4875` for primary `0x3114`, matching DCE DDC4 A;
  - source-mask information at descriptor `+0x3C`.
- The AUX factory path resolves `0x4871 -> pin index 2` and builds a dual-pin AUX/DDC transaction handle using the DDC data/clock endpoints.
- Earlier Windows RE also found that the display-map path can populate a per-object DPCD/AUX interface at record `+0x18` before a final live-sink/detection result is committed.
- The important distinction is that Windows can have an object-specific DPCD/AUX route for `0x3113` even while later detection/lifetime logic decides whether the secondary is currently live.

Linux route construction model:
- `dc_create()` calls `create_links()`, which asks the BIOS parser for connector count and connector ids.
- For object-info table rev 4/5, `bios_parser_get_connectors_number()` counts display paths with a nonzero encoder object, and `bios_parser_get_connector_id()` returns the decoded connector object from `display_objid`.
- For a given connector, `construct_phy()`:
  - stores the decoded `link_id`;
  - calls `get_src_obj()` to recover the encoder object from the same display path;
  - translates internal UNIPHY1 enum 2 to `TRANSMITTER_UNIPHY_D`, which is the secondary route expected for `0x3113`;
  - calls `link_create_ddc_service()` before HPD-based detection decides whether a sink is present.
- `link_create_ddc_service()` calls `bios_parser_get_i2c_info()`, which walks the connector record list for `ATOM_I2C_RECORD_TYPE`, maps the I2C id through the GPIO pin LUT, and creates a DDC GPIO pin.
- Plain-boot logs already prove this Linux construction succeeds at the physical route level:
  - secondary raw object is `0x3113`;
  - DDC hardware instance is `2`, matching DDC3;
  - DDC line/channel match the DDC3 route;
  - transmitter is `UNIPHY_D`;
  - a `ddc_pin` object exists.

Where the side-by-side diverges:
- Linux constructs the `0x3113` link/DDC/encoder route, but the normal DP detect path refuses to use it when HPD is low.
- `link_detect_connection_type()` treats ordinary DisplayPort HPD-low as `dc_connection_none`.
- `detect_link_and_local_sink()` then disconnects any previous local sink and only enters the DP `detect_dp()` / DPCD / EDID path if the connection type is not `none`.
- `set_ddc_transaction_type(..., I2C_OVER_AUX)` and `link->aux_mode = link_is_in_aux_transaction_mode(link->ddc)` happen inside that connected path.
- Therefore, in plain boot, `DP-1` / `0x3113` can have the correct DDC3 pin and UNIPHY_D route while still reporting `aux_mode=0`; later DPCD reads fail because the AUX transaction mode was never armed.
- OCLP/richer boots show the same route later with `aux_mode=1`, HPD high, a local sink, EDID, and trained DPCD state, so the physical route identity is not the missing piece.

Linux implementation implication:
- Do not treat current plain-boot `aux_mode=0` as proof that the `0x3113` route was not constructed.
- Split the current route check into three concepts:
  - `route_identity_ok`: object `0x3113`, DDC3/hw instance 2, DDC pin present, transmitter `UNIPHY_D`;
  - `aux_transport_armed`: DDC transaction mode has been switched to AUX and a basic DPCD read can execute;
  - `live_sink_ok`: detection produced a real `local_sink`, EDID, TILE data, and nonzero sink count.
- The next kernel target should be an iMac5K secondary-only pre-detect AUX activation/probe:
  - first require `route_identity_ok` without requiring `aux_mode`;
  - temporarily arm or force the DDC transaction type for the exact `0x3113`/DDC3/UNIPHY_D route;
  - read a harmless DPCD byte such as `0x000` before the ordinary HPD-low DP detect gate returns `none`;
  - if the read succeeds, continue with the secondary source-DPCD/ASSR/`0x4F1`/training path and then retry or override secondary detection narrowly for the internal tile.
- The current `amdgpu_dm_imac5k_verify_windows_route()` shape should not require `aux_mode` for the earliest plain-boot probe, because that rejects the real route before Linux has a chance to arm AUX.

Open RE item for a future live IDA pass:
- Confirm whether Windows' `CreateDceAuxDdcService` explicitly switches the lower handle into AUX transaction mode before HPD/live-sink detection, or whether the Windows DPCD interface can issue AUX transactions independently of the connector-ready state.
- This would sharpen the exact Linux analogue, but it does not change the current conclusion that object/DDC/encoder construction already succeeds on Linux plain boot.

### Live IDA confirmation: Windows `0x3113` AUX route construction

Date: 2026-05-23.

Loaded binary:
- `C:\proj\reverse\WT6A_INF\B416406\amdkmdag.sys.i64`
- module `amdkmdag.sys`
- SHA-256 `ac79119d7a64fc2619391ed972febf6c740d0199fd6ebb2fabb4728775a2fc05`

Confirmed call chain:
- `ConstructDisplayTopologyManager` calls `PopulateDisplayMapAuxObjectsForHotplugRange` at `0x14143B03E`.
- `PopulateDisplayMapAuxObjectsForHotplugRange` at `0x1414B8600`:
  - asks AdapterService vfunc `+0x60` for total display-object count;
  - asks AdapterService vfunc `+0x258` for the synthetic-tail count;
  - loops only over `[0, total - synthetic_tail)`, so this AUX population path is for real/provider-backed objects, not synthetic tail objects;
  - enumerates object ids through AdapterService vfunc `+0xD0`;
  - calls `CreateDisplayMapInterfaceForObjectIdMaybeSkipInternal`;
  - calls `CreateDpcdAuxInterfaceForObjectId` at `0x1414B8754`;
  - stores the returned DPCD/AUX interface at display-map entry `+0x18`;
  - expands display paths and streams only after `entry+0x18` is populated.
- `CreateDpcdAuxInterfaceForObjectId` at `0x141551400` has exactly one code xref: the population site above.
- `ConstructDpcdAuxInterfaceForObjectId` at `0x141551090`:
  - calls AdapterService vfunc `+0x170` at `0x1415511A1`;
  - passes the original object id;
  - invalidates the wrapper if the lower handle is null.
- AdapterService vfunc `+0x170` resolves to `AdapterServiceGetAuxHandleForObjectId` at `0x141465AA0`.
- `AdapterServiceGetAuxHandleForObjectId`:
  - first calls AdapterService vfunc `+0x100`, resolved as `AdapterServiceQueryObjectDescriptor` at `0x1414663A0`;
  - derives the AUX selector and source mask from the descriptor;
  - calls the AdapterService AUX factory with selector/source mask at `0x141465B5C`.
- `AuxFactoryCreateHandleForSourceMask` at `0x14152AE80` creates a `ConstructDualPinAuxTransactionHandle` object.
- `ConstructDualPinAuxTransactionHandle` at `0x1415B0C20`:
  - resolves selector/source mask via `ResolveSourceMaskToPinIndex`;
  - opens endpoint type `3` for the resolved pin index;
  - opens endpoint type `4` for the same pin index;
  - invalidates the handle if either endpoint is missing.
- `Dce11ResolveAuxSelectorToPinIndex` at `0x141695990` confirms the exact literal selector mapping:
  - `0x4869 -> 0`;
  - `0x486D -> 1`;
  - `0x4871 -> 2`;
  - `0x4875 -> 3`;
  - `0x4879 -> 4`;
  - `0x487D -> 5`;
  - `0x4881 -> 6`;
  - `0x4899 -> 7`.

Direct Linux comparison:
- Windows secondary object `0x3113` uses selector `0x4871`, which resolves to AUX pin index `2`.
- Linux plain-boot logs show the same physical route as:
  - raw object `0x3113`;
  - DDC hardware instance `2`;
  - DDC3 line/channel;
  - transmitter `UNIPHY_D`;
  - non-null `ddc_pin`.
- This confirms that Linux already constructs the Windows-equivalent physical route.
- The Linux gap remains transport activation/detect policy, not route discovery:
  - Windows can construct a durable per-object DPCD/AUX backend at display-map entry `+0x18`;
  - Linux creates the DDC pin but ordinary DP detect leaves `aux_mode=0` when HPD is low, so the route is physically present but not yet usable for DPCD.

IDA annotations made:
- `0x1414B8754`: display-map population creates per-object DPCD/AUX before stream expansion.
- `0x1415511A1`: DPCD wrapper asks AdapterService vfunc `+0x170` for object-specific lower AUX handle.
- `0x141465AA0`: object id to descriptor/selector/source-mask/AUX-factory bridge.
- `0x1414663E9`: descriptor query provider feeding selector/source fields.
- `0x141465B5C`: final AUX factory call using descriptor selector/source mask.
- `0x141695B0A`: `0x4871 -> AUX pin index 2`.
- `0x1415B0D4A`: endpoint type `3` opens DDC data side.
- `0x1415B0DB7`: endpoint type `4` opens DDC clock side.
- `0x141466F0D`: total object count includes VBIOS/object-info provider count.
- `0x141460B52`: synthetic-tail logic is not the real `0x3113` 5K/4K route decision.

Updated certainty:
- Certain: Windows' `0x3113` route is real object-table/provider-backed route materialized through descriptor selector `0x4871` and dual DDC endpoints.
- Certain: Linux plain boot already has the same object/DDC/transmitter route identity.
- Still open: whether Windows' DPCD interface explicitly arms AUX before HPD/live-sink detection, or whether its lower DPCD object can transact independently of a live-sink state bit.

### Live IDA continuation: DPCD transaction boundary after `entry+0x18`

Date: 2026-05-23.

Target question:
- Once Windows has the `0x3113` lower AUX handle, do DPCD reads/writes require a live-sink/HPD state, or can they transact from the object-specific lower handle alone?

Confirmed DPCD interface vtable for display-map entry `+0x18`:
- `off_141FAC048 + 0x00 -> DpcdAuxReadUpTo16Bytes` at `0x14154F320`;
- `off_141FAC048 + 0x08 -> DpcdAuxWriteUpTo16Bytes` at `0x14154F020`.

`DpcdAuxReadUpTo16Bytes` / `DpcdAuxWriteUpTo16Bytes` behavior:
- length is limited to `<= 0x10`;
- the object-local lower AUX transaction handle is copied from the DPCD wrapper;
- a stack DPCD request is built:
  - read uses `BuildDpcdAuxReadRequest`;
  - write uses `BuildDpcdAuxWriteRequest`;
- the request is submitted through the copied lower transaction path via `SubmitDpcdAuxRequest`;
- status is read from the request object and mapped back to a driver status.

Important absence:
- No HPD check is visible in the DPCD read/write wrapper.
- No live-display-object or detection-record pointer is consulted in the DPCD read/write wrapper.
- The visible transport gate is the lower AUX handle already stored in the DPCD wrapper, plus request length.

Lower dual-pin AUX handle path:
- `ConstructDualPinAuxTransactionHandle` had already shown that the lower handle requires both endpoint type `3` and endpoint type `4` for the resolved pin.
- The vtable at `off_141FB08E0` includes:
  - `DualPinAuxHandleConfigureTransaction`;
  - `DualPinAuxHandleSubmitTransaction`;
  - `DualPinAuxHandleSetTransactionFlags`.
- `DualPinAuxHandleSubmitTransaction` forwards the transaction through both endpoint proxies.
- The endpoint factory opens endpoints by type/pin:
  - case `3` opens/creates DDC DATA by pin index;
  - case `4` opens/creates DDC CLOCK by pin index.
- The DCE11 endpoint transaction/register layer programs DDC GPIO/AUX registers; no display HPD/live-sink predicate is visible in this lower layer.

Generic object-id AUX helper:
- Renamed `0x1416B03B0` to `AdapterServiceAuxReadWriteForObjectId`.
- This helper is another object-id based path:
  - object owns/stores an AdapterService pointer at `+0x28`;
  - object id is stored at `+0x30`;
  - the helper calls AdapterService vfunc `+0x170` to obtain the lower AUX handle for that object id;
  - mode `2` builds DPCD AUX read/write requests;
  - mode `0` builds I2C-over-AUX style requests.
- Again, the visible gate is `AdapterServiceGetAuxHandleForObjectId` returning a lower handle for the object id, not a live-sink/HPD condition.

IDA annotations made in this continuation:
- `0x14154F400`: DPCD read copies lower AUX handle and submits a <=16-byte DPCD request; no HPD/live-sink predicate visible.
- `0x14154F100`: DPCD write mirrors read and uses the lower handle/request path.
- `0x1414707DF`: request submission dispatches through the copied lower transaction path.
- `0x1415B09A7`: dual-pin submit forwards through DATA endpoint.
- `0x1415B09CA`: dual-pin submit forwards through CLOCK endpoint.
- `0x14152A412`: endpoint factory case `3` opens/creates DDC DATA by pin index.
- `0x14152A432`: endpoint factory case `4` opens/creates DDC CLOCK by pin index.
- `0x141696040`: DCE11 DDC endpoint transaction programming touches DDC GPIO/AUX registers only.
- `0x1416B046E`: generic object-id helper obtains lower AUX handle through AdapterService vfunc `+0x170`.
- `0x1416B0610`: generic helper mode `2` DPCD write request.
- `0x1416B064C`: generic helper mode `2` DPCD read request.

Updated Linux implication:
- Windows' lower transport model now looks even closer to the Linux patch we want:
  - prove route identity first;
  - obtain or arm the lower AUX/DDC transaction path for that object;
  - issue a small DPCD probe before relying on HPD/live-sink state.
- Linux should not wait for ordinary DP HPD-high detection before it even tries to put the proven `0x3113` DDC3 route into AUX transaction mode.
- The first plain-boot probe should be intentionally tiny:
  - exact route only: `0x3113` / DDC3 / hw instance `2` / `UNIPHY_D`;
  - force or arm AUX transaction mode;
  - read DPCD `0x000`;
  - log HPD before/after but do not require HPD as a precondition.

Previously open:
- At this point we had not proven every higher-level caller policy above the DPCD object.
- The next live IDA pass below refines that: the lower transport remains object/pin-handle based, while stream enable is gated by Windows' logical stream/VC sink-present state.

### Live IDA continuation: higher-level caller policy above `entry+0x18`

Date: 2026-05-23.

Target question:
- After Windows builds the object-specific `0x3113` DPCD/AUX backend, do higher-level callers still require live HPD before they use it?
- Or are there paths that carry/probe the backend first and only later reconcile live-sink state?

Answer:
- The lower DPCD/AUX object does not have an HPD/live-sink predicate.
- Display topology construction also does not require HPD before carrying the backend upward:
  - `ConstructDisplayObjectAndStreams` passes display-map entry `+0x18` into `CreateDisplayCapabilityServiceInterface`;
  - `ConstructDisplayCapabilityService` stores that pointer at service `+0x50`;
  - `BuildStreamForDisplayEntryPath` copies display-map entry `+0x18` into stream init data `+0x18`;
  - `ConstructDpStreamObjectWithDpcdInterface` stores that init pointer into the DP stream object (`stream+0x50` and `stream+0xB8` path already annotated).
- The late stream-enable path does have a higher-level guard, but it is not the lower AUX route:
  - `EnableValidatedStreamMaybeAssert4F1` calls `ValidateDisplayIndexSinkMappedPresent`;
  - `ValidateDisplayIndexSinkMappedPresent` requires the stream/VC entry to be mapped and `StreamVcEntryIsSinkPresent(entry)` to be true;
  - `StreamVcEntryIsSinkPresent` is just `entry+0x338 bit0`.

Important distinction:
- Windows separates three concepts:
  - durable per-object DPCD/AUX route: display-map `entry+0x18`;
  - display-map live bookkeeping: entry bytes `+0x10/+0x11` and detect-pass refcount `+0x0C`;
  - stream/VC logical sink-present state: stream entry `+0x338 bit0`.
- The DPCD route can be created and stored before the final live-display update.
- Stream enable requires the logical stream/VC sink-present bit, but that bit can be preserved or re-marked by detect/recovery paths.

Caller-side evidence:
- `WriteSinkDpcd4F1Payload1Retry` writes DPCD `0x4F1=1` through display-map `entry+0x18` after resolving the object id; it does not use the detection record's live object slot.
- `ProbeDisplayStatusMaybeAssert4F1` validates the current status backend against display-map `entry+0x18/+0x20` before status handling and optional `0x4F1`.
- `DetectDisplayMaybeAssert4F1ForCapableSink` reaches that detect/status-time `0x4F1` writer when descriptor byte `+6 bit2` is observed.
- The separate stream-enable `0x4F1` path accepts descriptor byte `+6 bit2` or `+6 bit3`, which remains the stronger Apple internal-tile path for AE26/tag `0x3B`.

False-disconnect / preservation link:
- `DetectDisplayAndApplyApple5kOverrides` has the Apple/tiled false-disconnect gate:
  - descriptor byte `+6 bit3` set;
  - current display object still connected;
  - fresh detect result says disconnected;
  - current signal type is `11`;
  - then it forces `detection_result.connected = 1` and clears the changed/rescan flag.
- `HandleDisplayDetectionResultAndLiveState` only calls the destructive live-object update on changed/rescan/embedded-panel cases.
- Therefore the tag `0x3B` correction preserves the already-live logical sink/stream state instead of letting the disconnect path clear it.

New IDA names/comments from this pass:
- `0x141443B90` -> `ValidateDisplayIndexSinkMappedPresent`
- `0x1414D0580` -> `StreamVcEntryIsSinkPresent`
- `0x1414D0540` -> `SetStreamVcEntrySinkPresent`
- `0x1414D0310` -> `StreamVcEntryHasPayloadAllocated`
- `0x1414D0460` -> `GetStreamVcEntryDisplayRecord`
- `0x1414D0C70` -> `FindStreamVcEntryByDisplayIndex`
- `0x1414D0A90` -> `FindFreeStreamVcEntryForDetect`
- `0x1414D09F0` -> `FindPresentStreamVcEntryBySinkGuid`
- Added comments at the stream-present guard, setter, detect/re-detect present-bit transitions, and Apple/tag `0x3B` false-disconnect preservation gate.
- Added comments at the capability/stream construction handoff points where display-map `entry+0x18` is consumed before the display-map entry is marked live.

Linux implication:
- Do not model Windows as "wait for HPD, then create the secondary route".
- A closer Linux model is:
  - keep/build the proven `0x3113` DDC3/UNIPHY_D route even when ordinary HPD says disconnected;
  - arm AUX on that exact route and perform a tiny DPCD probe;
  - if the probe and Apple/tile predicates match, preserve/create the logical secondary sink state narrowly;
  - only then allow the tile-pair stream enable path to run.
- The missing Linux behavior is likely not only an AUX transaction primitive. It is also a lifetime policy: the secondary internal tile must not be torn down just because the ordinary DP detect pass briefly reports "not connected" on this exact Apple/tiled route.

### Live IDA certainty pass: why both Linux pieces are required

Date: 2026-05-23.

Question:
- Can we turn "Linux likely needs both AUX route activation and logical sink preservation" into a firmer statement?

Answer:
- Yes. Windows' stream-enable wrapper has explicit logical stream/VC gates before it calls the actual stream-enable worker.
- A durable `entry+0x18` DPCD/AUX route is necessary, but not sufficient.

Confirmed stream-enable gates:
- `EnableStreamValidatedWrapperMaybe4F1` first calls `StreamVcEntryEnablePrecheckByDisplayIndex` at `0x1414F4FC0`.
- `StreamVcEntryEnablePrecheckByDisplayIndex`:
  - finds the stream/VC display-record subobject by display index through `FindStreamVcEntryByDisplayIndex`;
  - follows display-record `+0x10`, which is a back-pointer to the owning stream/VC entry;
  - follows entry vfunc `+0xA0`, which returns the mapped emulated-sink object at entry `+0x3A8` / `+936`;
  - reads the emulated-sink object's `ConnectionStatus` flags through vfunc `+0x50`;
  - requires `ConnectionStatus bit0` set before the stream-enable wrapper proceeds.
- If that precheck passes, `EnableValidatedStreamMaybeAssert4F1` then calls `ValidateDisplayIndexSinkMappedPresent`.
- `ValidateDisplayIndexSinkMappedPresent` requires:
  - the stream/VC entry exists;
  - the display-record back-pointer is mapped to its owning stream/VC entry;
  - `StreamVcEntryIsSinkPresent(entry)` is true, which is `entry+0x338 bit0`.

Confirmed display-object disconnect propagation:
- The display object connected getter/setter were resolved from the full display-object vtable:
  - vfunc `+0x258` -> `FullDisplayObjectIsConnected` at `0x14154A560`;
  - vfunc `+0x418` -> `FullDisplayObjectSetConnected` at `0x14154A4D0`;
  - both operate on the connected byte at interface `+0x188`.
- `UpdateDisplaySinkConnectionAndLiveAuxState` writes the detection-result connected byte into the display object through this setter.
- `UpdateDetectionRecordLiveDisplayObject` clears detection-record `+0x18` when the updater runs and the display object reports disconnected.
- Therefore the Apple/tag `0x3B` false-disconnect correction is not cosmetic:
  - it prevents the connected byte from being written false;
  - it prevents the detection-record live display object pointer from being cleared;
  - it preserves the state that later stream/mode construction resolves through `stream_arg+0x178`.

New IDA names/comments from this pass:
- `0x14154A560` -> `FullDisplayObjectIsConnected`
- `0x14154A4D0` -> `FullDisplayObjectSetConnected`
- `0x1414F4FC0` -> `StreamVcEntryEnablePrecheckByDisplayIndex`
- Added comments at the stream-enable precheck, display-object connected setter/getter, and live-record clear.

Updated certainty:
- Certain: Windows separates "DPCD/AUX route exists" from "logical stream/VC sink is present and ready".
- Certain: stream enable will not run from the DPCD route alone; it needs the logical stream/VC gates to pass.
- Certain: the tag `0x3B` false-disconnect path preserves display-object/live-record state before the destructive updater clears it.
- Still not perfectly mapped one-to-one: the Windows stream/VC entry and backing-object readiness bit do not have a single obvious Linux struct equivalent. In Linux terms, the closest implementation rule is still:
  - keep the exact `0x3113` route and AUX transport usable;
  - preserve/create the real secondary `dc_sink` / connector sink state narrowly for the Apple/tiled internal route;
  - do not let a transient ordinary DP disconnect path clear the secondary before tile-pair stream enable.

### Live IDA continuation: stream/VC readiness object

Date: 2026-05-23.

Question:
- What is the backing object tested by `StreamVcEntryEnablePrecheckByDisplayIndex`, and can we turn the "both pieces required" conclusion into a more exact Windows state requirement?

Correction to the previous reading:
- `FindStreamVcEntryByDisplayIndex()` returns `GetStreamVcEntryDisplayRecord(entry)`, not the stream/VC entry base.
- `GetStreamVcEntryDisplayRecord(entry)` returns `entry+0x350`.
- Therefore the precheck's `+0x10` load is `display_record+0x10`, a back-pointer to the owning stream/VC entry.
- That back-pointer is initialized in both stream/VC construction paths:
  - `0x1414F18EE`: `display_record+0x10 = entry`;
  - `0x1414F1B7A`: `display_record+0x10 = entry`.

Resolved precheck chain:
- `StreamVcEntryEnablePrecheckByDisplayIndex` at `0x1414F4FC0`:
  - `vc_manager = *(manager+0x3F0)`;
  - `display_record = FindStreamVcEntryByDisplayIndex(vc_manager, display_index)`;
  - `entry = *(display_record+0x10)`;
  - `mapped_sink = entry->vfunc+0xA0()`;
  - `flags = mapped_sink->vfunc+0x50()`;
  - return `flags & 1`.
- Entry vfunc `+0xA0` is `StreamVcEntryGetMappedEmulatedSink` at `0x141573B10`; it returns `entry+0x3A8` / `+936`.
- Entry vfunc `+0xE8` is `StreamVcEntrySetMappedEmulatedSink` at `0x141573AB0`; it stores the mapped object into `entry+0x3A8` / `+936`.

Resolved backing object:
- `FindEmulatedSinkListEntryByGuid` at `0x1414F05D0` scans the manager's `All_MstDevices` collection at `manager+0x488` / `+1160`.
- The list entry layout used here is:
  - byte `+0`: valid/present marker used by init/redetect paths;
  - qword `+8`: emulated MST sink object;
  - bytes `+16...`: sink GUID / RAD-like identity compared by `sub_1414BB7A0`.
- Non-branch sink restore/list population is in `RestoreNonBranchMstSinkFromRegistry` at `0x1414EFC30`.
- The object stored at list entry `+8` is created by `CreateEmulatedMstSinkObject` at `0x141576700`, which returns the subobject at allocation `+0x28` / `+40` with vtable `off_141FAE160`.

Resolved object methods:
- Emulated sink vfunc `+0x30` -> `EmulatedMstSinkGetEmulationMode` at `0x141575740`; returns object `+0xC`.
- Emulated sink vfunc `+0x50` -> `EmulatedMstSinkGetConnectionStatus` at `0x141574DE0`; copies object `+0x8` into the caller's output flags.
- Emulated sink vfunc `+0x58` -> `EmulatedMstSinkHasDisconnectedEmulationData` at `0x141575E50`; only true when `ConnectionStatus bit0` is clear and saved/emulated data policy says the object can emulate data.
- Emulated sink vfunc `+0x60` -> `EmulatedMstSinkHasConnectedEmulationData` at `0x141575D50`; only true when `ConnectionStatus bit0` is set and saved/emulated data policy says the object can emulate data.
- Emulated sink vfunc `+0x70` -> `EmulatedMstSinkSetConnectionStatus` at `0x141575A80`; updates `ConnectionStatus bit0` from the caller argument and persists/recomputes derived state.
- Constructor/registry load `sub_141574E10` reads `EmulationMode`, `ConnectionStatus`, `ConnectionProperties`, and emulation data, but then clears `ConnectionStatus bit0` at `0x14157512B`. Restored emulation data alone therefore does not pass stream-enable precheck.

Why this matters:
- Windows has at least three separate states in this layer:
  - physical/lower route and AUX/DPCD transport;
  - stream/VC entry `sink_present` bit at `entry+0x338 bit0`;
  - mapped emulated-sink object `ConnectionStatus bit0`.
- The stream-enable wrapper requires the mapped emulated-sink object's `ConnectionStatus bit0` before it even reaches the later `StreamVcEntryIsSinkPresent` validation.
- The DPCD emulation helpers can skip/fake remote DPCD reads or compare against cached emulation data when the object reports emulation-data availability, but that is not enough for stream enable because the precheck tests `ConnectionStatus bit0` directly.

Updated certainty:
- Certain: the earlier "entry+0x10 backing object" phrasing was wrong; it is a display-record back-pointer to the owning stream/VC entry.
- Certain: the stream-enable readiness bit is the emulated MST sink object's `ConnectionStatus bit0`, not raw HPD and not merely the stream/VC `sink_present` bit.
- Certain: a route/AUX object plus saved emulation data is insufficient unless the mapped emulated-sink object is also marked connected.
- Linux implication: the closest policy is now more exact:
  - keep/arm the real `0x3113` route and AUX transport;
  - preserve the secondary's real logical sink lifetime;
  - do not allow the exact Apple/tiled secondary to enter the tile-pair stream-enable path unless the Linux state is equivalent to Windows' mapped-sink connected bit, not merely "we still have a DDC route".

New IDA names/comments from this pass:
- `0x1414F05D0` -> `FindEmulatedSinkListEntryByGuid`
- `0x1414EFC30` -> `RestoreNonBranchMstSinkFromRegistry`
- `0x141576700` -> `CreateEmulatedMstSinkObject`
- `0x1415763D0` -> `EmulatedMstSinkObjectCtor`
- `0x141574DE0` -> `EmulatedMstSinkGetConnectionStatus`
- `0x141575740` -> `EmulatedMstSinkGetEmulationMode`
- `0x141575A80` -> `EmulatedMstSinkSetConnectionStatus`
- `0x141573B10` -> `StreamVcEntryGetMappedEmulatedSink`
- `0x141573AB0` -> `StreamVcEntrySetMappedEmulatedSink`
- `0x1414F0870` -> `FindPendingMstDeviceRecordByGuid`
- `0x141575E50` -> `EmulatedMstSinkHasDisconnectedEmulationData`
- `0x141575D50` -> `EmulatedMstSinkHasConnectedEmulationData`
- Added IDA comments at the display-record back-pointer loads/stores, entry mapped-sink getter/setter, mapped-sink install sites, `ConnectionStatus` getter, constructor bit0 clear, and setter bit0 update.

### Live IDA continuation: who asserts mapped-sink `ConnectionStatus bit0`

Date: 2026-05-24.

Question:
- The stream-enable precheck requires the mapped emulated-sink object's `ConnectionStatus bit0`. Who actually sets that bit, and does it correspond to HPD, AUX proof, stream/VC sink-present, or generic saved emulation data?

Answer:
- Static RE found two meaningful ways for the mapped emulated-sink object to become connected:
  - a detected-sink add path actively sets `ConnectionStatus bit0 = 1` before initializing/mapping the stream/VC entry;
  - a stream-entry sync path copies `StreamVcEntryIsSinkPresent(entry)` into the mapped object's `ConnectionStatus bit0`.
- Restore and redetect paths can preserve or clear the bit, but they do not by themselves turn saved emulation data into a durable connected state.

Confirmed active set path:
- `MstEmuTryAddDetectedSinkAndMarkConnected` at `0x1414F0FD0` is a vtable method on the MST/emulation manager subobject at manager `+0x3B8`.
- It refuses to proceed if the sink GUID is still in the pending-device record list.
- It resolves an emulated-sink object by GUID through the manager's object lookup subinterface.
- At `0x1414F10B5`, it calls emulated-sink vfunc `+0x70` with `(connected=1, mode=0)`, which sets `ConnectionStatus bit0`.
- It then requires either:
  - `EmulatedMstSinkHasConnectedEmulationData()` true; or
  - an already-present stream/VC entry plus either nonzero emulation mode or a higher policy/service flag.
- If that passes, it calls `MstEmuInitVcEntryForDetectedSink` at `0x1414F11F7`, which sets the stream/VC `sink_present` bit, maps the emulated-sink object into entry `+0x3A8`, and starts the remote DPCD probe state machine through `sub_1414D0120`.

Confirmed stream-entry sync path:
- `MstEmuSyncMappedSinkConnectionFromVcEntry` at `0x1414F0DD0` reconstructs/looks up the emulated-sink object for an existing stream/VC entry.
- At `0x1414F0ED7`, it reads `StreamVcEntryIsSinkPresent(entry)`.
- At `0x1414F0EF7`, it calls emulated-sink vfunc `+0x70` with that value, so the mapped object's `ConnectionStatus bit0` becomes a mirror of `entry+0x338 bit0`.
- It then maps the emulated-sink object back into `entry+0x3A8`.

Confirmed clear/preserve paths:
- `MstEmuRedetectSinkAndMapVcEntry` at `0x1414F1420`:
  - clears both stream/VC `sink_present` and mapped-sink `ConnectionStatus bit0` on failed redetect (`0x1414F1524` and `0x1414F153F`);
  - on an existing entry that still has usable connected/disconnected emulation data, it re-applies the existing `ConnectionStatus bit0` value at `0x1414F162B`;
  - for a newly-created VC entry, it first clears mapped-sink `ConnectionStatus bit0` at `0x1414F17AF`, then separately sets stream/VC `sink_present` at `0x1414F184C` and maps the object at `0x1414F18C4`.
- `MstEmuInitVcEntryForDetectedSink` at `0x1414F1960` marks the stream/VC entry sink-present and maps the emulated-sink object, but does not itself set mapped-sink `ConnectionStatus bit0`.
- `RestoreNonBranchMstSinkFromRegistry` may transiently call the setter with `connected=1`, but that branch immediately calls it again with `connected=0`; registry restore/list population does not leave the mapped sink connected.

Updated certainty:
- Certain: mapped-sink `ConnectionStatus bit0` is not just "saved emulation data exists".
- Certain: Windows can assert it from a detected-sink path before stream/VC init, and can later synchronize it from stream/VC `sink_present`.
- Certain: redetect/preservation paths are mostly about preventing destructive clear or re-applying an existing bit, not inventing connected state without a prior detect/sink-present basis.
- Linux implication:
  - preserving `dc_sink` is closest to preserving Windows stream/VC `sink_present`;
  - keeping the exact `0x3113` AUX route usable is still separate;
  - the "allow tile-pair stream enable" predicate should be equivalent to: exact route proven, logical secondary sink preserved/present, and no pending false-disconnect teardown. It should not be based on EDID cache alone.

New IDA names/comments from this pass:
- `0x1414F0FD0` -> `MstEmuTryAddDetectedSinkAndMarkConnected`
- `0x1414F0DD0` -> `MstEmuSyncMappedSinkConnectionFromVcEntry`
- `0x1414F1420` -> `MstEmuRedetectSinkAndMapVcEntry`
- `0x1414F1960` -> `MstEmuInitVcEntryForDetectedSink`
- Added comments at the connected setter call, stream-entry sync setter call, redetect clear/preserve calls, new-entry clear, and init/map sites.

### Live IDA continuation: primary tile metadata and `config+0x20`

Date: 2026-05-26.

Question:
- Does Windows make the iMac19,1 primary/root tile-aware by forcing a fresh eDP/root EDID read, or does it construct the primary tile metadata from a grouped/root object while only re-reading the secondary?
- What exactly is the `service a7` / `config+0x20` object previously flagged as the next object to identify?

Result:
- The loaded modern BootCamp driver is `B416406\amdkmdag.sys.i64`.
- `ConstructDisplayCapabilityService()` is present at `0x1415DC180`.
- `CreateDisplayCapabilityServiceInterface()` calls it from two display-construction paths:
  - `0x1414B5353` in `sub_1414B5210`, with final argument `0`;
  - `0x1414B749D` in `ConstructDisplayObjectAndStreams`, with final argument `*(path_config + 0x20)`.
- In the main display-object construction path, `path_config+0x20` is not a peer/group metadata object. It is filled from the final class-3 display-map entry:
  - `sub_1414B6380` reaches a class-3 object id, such as the leaf display endpoint;
  - it uses the matching display-map entry;
  - it stores `display_map_entry[3]` / `display_map_entry+0x18` into `path_config+0x20`.
- `ConstructDisplayCapabilityService()` then stores that final argument at `service+0x50`.
- Therefore `service a7` / `config+0x20` is the endpoint-local lower AUX/DPCD backend for the leaf display object. It can be the root/eDP endpoint or the secondary DP endpoint depending on which object is being constructed, but it is not a single grouped object that manufactures peer tile metadata.

Endpoint tile metadata correlation:
- `UpdateEndpointModesAndTiledMetadataAfterPinAttach()` still calls `PopulateEndpointTileMetadataFromDisplayIdTopology()` for the selected display endpoint.
- `PopulateEndpointTileMetadataFromDisplayIdTopology()` obtains tile topology through the display object's own EDID/DisplayID parser vtable slot `+0x158`.
- `WindowsDM_CopyEndpointTileMetadataByDisplayId()` copies the endpoint's own `display_obj+0x90+0x98` block and returns false if that endpoint token is absent.
- No peer-clone path was found in this pass.

EDID-payload update path:
- `WindowsDM_ApplyEdidPayloadAndRefreshDisplay()` at `0x141A1C2D0` applies an EDID payload to one display index, calls `ParseEdidAndPinCapsIntoDcSinkInfo()`, then calls `ApplyEdidPayloadAndRebuildDcSinkForDisplay()`.
- After that, it calls the normal endpoint update path again:
  - `RefreshDisplayTileActiveFromMetadataFlags()`;
  - `sub_141A07ED0(...)`;
  - `sub_141A064F0(...)`.
- This means Windows' explicit EDID-payload path rebuilds the sink and tile metadata for the selected display object itself. It is still endpoint-local.

Updated answer to the primary question:
- Static RE now leans strongly away from "Windows constructs the primary tile purely from the secondary/root group without primary endpoint tile metadata."
- The normal Windows grouped-validation model expects both endpoints to have their own DisplayID-derived tile metadata.
- The remaining unknown is runtime ordering: for iMac19,1 plain BootCamp boot, does Windows actually force a fresh primary/eDP EDID read so the primary endpoint obtains tile-aware DisplayID, or does the primary already have tile-aware EDID by the time this display-object update path runs?
- `config+0x20` no longer looks like the unresolved magic grouped object. It is the per-endpoint lower AUX/DPCD backend, so a forced refresh can in principle target either primary or secondary as separate endpoints.

Linux implication:
- Best target remains to get both Linux connectors into endpoint-local tile metadata:
  - primary `0x3114` must have real or narrowly reconstructed `0,0 / 2560x2880` tile metadata;
  - secondary `0x3113` must have real `1,0 / 2560x2880` tile metadata from its own route/EDID when possible;
  - any Linux synthesis of primary or secondary TILE should be labeled as a compatibility fallback, not as behavior proven in the Windows normal path.
- The cheapest empirical test is still valuable: after the primary `0x4F1` wake and successful secondary EDID read, force/retry a primary eDP EDID read and log whether the primary changes from the 4K/no-TILE `APP/AE25` style state to a tile-aware DisplayID state.

New IDA names/comments from this pass:
- `0x1415CFE30` -> `DisplayCapabilityService_GetCapabilityMask8`
- `0x1415D4C40` -> `DisplayCapabilityService_RefreshEdidOverrideAndCaps`
- `0x141A1C2D0` -> `WindowsDM_ApplyEdidPayloadAndRefreshDisplay`
- `0x141A06AE0` -> `WindowsDM_RemoveSinkFromMapByIndex`
- Added comments at `0x1414B6622`, `0x1414B749D`, `0x1415DC221`, `0x1415D4CA6`, `0x141A1C4AF`, and `0x141A1C61C`.

### Live IDA continuation: physical EDID read versus endpoint-local EDID fixups

Date: 2026-05-26.

Question:
- Does the modern BootCamp path prove that `APP/AE25` / `APP/AE26` force a fresh primary/root physical EDID read?
- Or do those Apple panel capabilities only affect the higher-level EDID/pin-cap/tile parsing path after an EDID buffer already exists?

Result:
- The string `Forced EDID read.` is in `EdidReader_RefreshMaybeForced()` at `0x141550DC0`.
- That function has two ways to perform a physical EDID refresh:
  - capability check `HasCapability(37)` / `0x25` succeeds, so it reads EDID even without the explicit force argument;
  - explicit force argument `a2` is set, so it calls the hardware EDID reader and logs `Forced EDID read.`.
- The physical read helper is `EdidReader_ReadEdidBlocksFromHardware()` at `0x141550880`.
  - It first tries DDC address `0x52`.
  - If that does not produce a valid EDID-like block, it scans addresses `0x50..0x52`.
  - The block read path goes through `EdidReader_ReadOneEdidBlock()` at `0x14154FA30`, which performs the actual segmented EDID/I2C-style transactions.
- This physical-read path is endpoint-local: it acts on the EDID reader/backend object it is called on. It is not a primary-from-secondary or secondary-from-primary TILE reconstruction path.

Important split from the Apple capability path:
- `APP/AE25` and `APP/AE26` are still proven vendor/product capability records, but they map through `MapCapabilityTableIdToPinPropertyTag()` as:
  - source id `0x38` -> pin property tag `0x3A`;
  - source id `0x39` -> pin property tag `0x3B`.
- The forced physical EDID-reader capability check is for id/tag `37` / `0x25`.
- This pass did not find static evidence that the `APP/AE25` or `APP/AE26` rows themselves set capability `0x25`.
- Therefore, the `APP/AE25` / `APP/AE26` rows are proven to drive the endpoint-local pin-cap/EDID-fixup/tile parsing path, but they are not yet proof that Windows physically re-reads the primary/root EDID.

Higher-level endpoint-local fixup paths:
- `ConstructDisplayPinFromEdidApplyVendorCaps()` constructs a pin from a supplied EDID buffer, applies the vendor/product capability table, and immediately runs `ApplyPinCapabilityEdidFixups()` if any pin-cap records exist.
- `ApplyPinCapabilityEdidFixups()` dispatches tags `0x3A` and `0x3B` to `DoubleEdidPhysicalWidthForTiledCaps()`. This is the known Apple/tiled EDID geometry fixup, not a hardware re-read.
- `WindowsDM_ReadType128EdidWithPinCapabilityFixups()` applies the same pin-cap EDID fixup path to the selected display object's type-128/internal EDID buffer.
- `DisplayCapabilityCache_RefreshBaseEdidWithFixups()` / `RefreshEdidCacheApplyPinCapabilityFixups()` refresh cached parsed EDID objects and applies fixups to that endpoint's raw/cached EDID buffer; it also does not by itself prove a new physical EDID transaction occurred.
- `UpdateEndpointModesAndTiledMetadataAfterPinAttach()` constructs a pin from this display object's current EDID/mode block, then calls `PopulateEndpointTileMetadataFromDisplayIdTopology()` for the same display index. No peer clone was found here either.

Updated answer:
- More certain: Windows' normal TILE metadata path is endpoint-local. The primary/root tile is not visibly manufactured from the secondary tile's DisplayID block.
- More certain: AE25/AE26 explain the Apple/tiled EDID fixup and pin-cap path, but do not by themselves prove the physical primary re-read trigger.
- Still unresolved: at runtime on iMac19,1, the primary may become tile-aware because:
  - the panel/firmware changes what the primary EDID reader returns after the root `0x4F1` wake;
  - another policy sets the explicit-force or capability-`0x25` physical EDID-read path for the primary;
  - the primary was already represented by a tile-aware/type-128 internal EDID buffer before the mode/tile update runs.

Linux implication:
- Keep the forced primary EDID re-read experiment. It is now clearly Windows-shaped because Windows has a real per-endpoint physical EDID reader that can be forced, even though AE25/AE26 do not prove when it is invoked.
- If the primary re-read flips to tile-aware `0,0 / 2560x2880`, Linux should use the real primary EDID/TILE.
- If the primary re-read remains 4K/no-TILE while the secondary has real `1,0 / 2560x2880`, Linux will need a narrowly gated primary TILE reconstruction fallback. That fallback should be documented as a Linux compatibility step, not as behavior proven to be Windows' normal path.

New IDA names/comments from this pass:
- `0x141550DC0` -> `EdidReader_RefreshMaybeForced`
- `0x141550880` -> `EdidReader_ReadEdidBlocksFromHardware`
- `0x14154FA30` -> `EdidReader_ReadOneEdidBlock`
- `0x1415E8780` -> `DisplayCapabilityCache_RefreshBaseEdidWithFixups`
- Added comments at `0x141550E0F`, `0x141550F13`, `0x1415508DC`, `0x1415E87B1`, `0x141A0FAC0`, and `0x141A2227D`.

### Live IDA continuation: EDID-reader call surface and pin-cap pipeline integration

Date: 2026-05-26.

Questions:
- Does Windows call the `Forced EDID read.` / `EdidReader_RefreshMaybeForced()` path from the iMac 5K detect/update path, and if so when and for which tile?
- How is the pin-cap EDID fixup path integrated, and what should Linux copy from it?

Result: two EDID readers must be kept distinct.

Physical/forced EDID reader:
- `EdidReader_RefreshMaybeForced()` at `0x141550DC0` is a vfunc on the per-object DPCD/AUX wrapper vtable `off_141FAC048`, slot `+0x60`.
- It can perform a hardware EDID refresh when:
  - capability `37` / `0x25` is present; or
  - its explicit force argument is set, which logs `Forced EDID read.`.
- The wrapper is endpoint-local. `CreateDpcdAuxInterfaceForObjectId()` constructs it for a concrete AMD object id and returns the `+0x28` subobject stored at display-map entry `+0x18`.
- Therefore, if this vfunc is called on an iMac 5K endpoint, it would target whichever concrete wrapper is selected: primary/root `0x3114` or secondary `0x3113`. It is not intrinsically a grouped/two-tile operation.
- Static xrefs still show no direct code caller for `EdidReader_RefreshMaybeForced()`; its only meaningful reference is the vtable entry. A byte/instruction scan for direct virtual-call slot `+0x60` candidates did not find a concrete 5K detect/update caller.

Normal DC detect EDID reader:
- The actual `DcLinkDetectHelper` path reaches:
  - `ReadEdidIntoDcSinkThroughPin()` at `0x141EDDD50`;
  - `sub_1419FF3A0()`;
  - `ReadEdidAndPopulateDcSink()` at `0x141BB2E70`;
  - `DdcService_ReadEdidBlocksForDetect()` at `0x141BB1E60`;
  - `DdcService_ReadOneEdidBlockWithRetries()` at `0x141BB2590`.
- This normal path first tries address `0x52`, then falls back to `0x50` and extension reads.
- The low-level block read chooses:
  - `DdcService_ReadEdidBlockAux()` at `0x141BB2860`; or
  - `DdcService_ReadEdidBlockI2c()` at `0x141BB2650`;
  - up to seven retries per EDID block.
- After the EDID bytes are obtained, `ReadEdidAndPopulateDcSink()` calls `ParseEdidAndPinCapsIntoDcSinkInfo()`, which constructs the same capability-aware pin/parser path used elsewhere.

Answer to "when and which tile":
- For the traced Windows 5K-relevant detect path, the EDID read is the normal DDC-service read, not the `Forced EDID read.` vfunc.
- The read is endpoint-local: when the active link/display object is `0x3113`, it reads the secondary endpoint; when it is `0x3114`, it reads the primary endpoint.
- This pass did not prove that Windows forces a physical primary/root EDID re-read during iMac19,1 5K bring-up. The forced reader remains a real Windows-shaped mechanism, but it is not the normal `DcLinkDetectHelper` EDID path we traced.

Pin-cap/fixup pipeline integration:
- `ConstructDisplayPinFromEdidApplyVendorCaps()` is the central path:
  1. copy supplied EDID bytes into the pin object;
  2. run `ApplyVendorProductCapabilityTableToPin()`;
  3. convert table source ids through `MapCapabilityTableIdToPinPropertyTag()`;
  4. insert pin-cap records and set descriptor bits through `SetPinDescriptorCapabilityBitById()`;
  5. run `ApplyPinCapabilityEdidFixups()` if any records exist;
  6. create the EDID parser chain from the already-fixed EDID bytes.
- For modern Apple panel ids:
  - `APP/AE25` maps source id `0x38` to pin property tag `0x3A`;
  - `APP/AE26` maps source id `0x39` to pin property tag `0x3B`.
- `SetPinDescriptorCapabilityBitById()` maps those tags to descriptor byte `+6`:
  - tag `0x3A` -> byte `+6`, bit `2`;
  - tag `0x3B` -> byte `+6`, bit `3`.
- Those same descriptor bits are later consumed by the `0x4F1` paths:
  - detect/status-time writer accepts tag `0x3A` / byte `+6 bit2`;
  - stream-enable writer accepts tag `0x3A` or `0x3B` / byte `+6 bit2` or `bit3`.
- The concrete EDID mutation for tags `0x3A`/`0x3B` is only `DoubleEdidPhysicalWidthForTiledCaps()`:
  - if EDID byte `21` horizontal physical size is less than byte `22` vertical size and less than `0x80`, double byte `21`;
  - recompute EDID block-0 checksum;
  - do not alter modes, preferred timing, or DisplayID TILE payload.
- `WindowsDM_ReadType128EdidWithPinCapabilityFixups()` and `DisplayCapabilityCache_RefreshBaseEdidWithFixups()` reuse this same endpoint-local fixup model for cached/type-128 EDID buffers.

Linux implication:
- Linux should apply any Apple/tiled pin-cap-equivalent EDID fixup before EDID/DisplayID parser use, not after TILE metadata has already been derived.
- For AE25/AE26, the Windows-proven fixup is narrow: physical-width byte correction plus checksum only.
- Linux should keep the forced primary EDID re-read experiment as an empirical probe, but should not treat it as proven Windows runtime behavior from this RE pass.
- If the primary EDID does not become tile-aware after wake/re-read, a Linux primary TILE reconstruction fallback is still plausible, but it should be documented as a compatibility fallback rather than claimed as Windows' normal mechanism.

New IDA names/comments from this pass:
- `0x141BB1E60` -> `DdcService_ReadEdidBlocksForDetect`
- `0x141BB2590` -> `DdcService_ReadOneEdidBlockWithRetries`
- `0x141BB2650` -> `DdcService_ReadEdidBlockI2c`
- `0x141BB2860` -> `DdcService_ReadEdidBlockAux`
- Added comments at `0x141BB2EC4`, `0x141BB1EE2`, `0x141BB2143`, `0x141BB25C7`, `0x141BB25F0`, `0x141BB2613`, `0x141550DC0`, `0x141A9F483`, `0x141A9F539`, and `0x141A45922`.

### Live IDA continuation: what replaces primary 4K/no-TILE state

Date: 2026-05-26.

Question:
- Can more RE tell us whether Linux needs a real new primary EDID, or whether it can construct the primary tile state when the primary still reports the plain 4K/no-TILE EDID?

Result:
- The downstream Windows SetMode/tile-split consumers make the primary requirement clearer.
- Windows does **not** appear to have a downstream repair path that says "if primary has no TILE, borrow the secondary TILE and continue."
- But Windows also does **not** require the raw EDID mode list to literally contain `2560x2880`. It has a derived-mode path that can inject the physical half-tile mode when the endpoint has the right tile/timing flags.

Primary endpoint TILE is required:
- `PopulateEndpointTileMetadataFromDisplayIdTopology()` is called from `UpdateEndpointModesAndTiledMetadataAfterPinAttach()` for the selected display endpoint.
- If that per-endpoint DisplayID tiled-topology parse succeeds, Windows writes the endpoint tile block at `display_obj+0x90+0x98`.
- If it fails, `UpdateEndpointModesAndTiledMetadataAfterPinAttach()` zeroes the endpoint tile block.
- `WindowsDM_CopyEndpointTileMetadataByDisplayId()` is the concrete vfunc `+0x2C8` implementation. It returns false if that endpoint tile block token is missing.
- `SetMode_FindTileGroupAnchorDisplay()` requires vfunc `+0x2C8` to return tile metadata for the first endpoint and for every peer endpoint.
- `ValidateDisplaysShareTiledGroupMetadata()` also requires:
  - a nonzero tile metadata token on each endpoint;
  - matching group token across endpoints;
  - grid product equal to the active display count;
  - coverage of all tile slots.
- `BuildSourceModeForTileSplitPath()` disables its tile-split branch when it cannot find a peer display with matching tile-group metadata.

Mode can be derived:
- `ModeManagerAddDerivedModesIncluding5kTile()` can inject the `2560x2880` physical half-tile mode.
- The path depends on:
  - mode-manager pipe/tile option flags;
  - a timing/mode record flag bit7 set by `MarkTimingPipeSplitForHighResAnd5k60()` or `MarkModeRecordIfTileAspectCompatible()`;
  - base timing shape `1920x2160`;
  - `DalEnable5kTiledMode` for the final `2560x2880` injection.
- Therefore Windows can get the half-tile mode through derived-mode logic, not necessarily because the raw EDID directly advertised `2560x2880`.

Updated certainty:
- Certain: the primary tile group token cannot be missing by the time Windows grouped SetMode validates the pair.
- Certain: Windows does not downstream-copy the secondary tile token into a missing primary tile block in the traced path.
- Certain: raw EDID modes and endpoint TILE metadata are separate problems. The half-tile mode can be derived, but the tile metadata still has to exist per endpoint.
- Still unresolved: at runtime, how the iMac19,1 primary endpoint obtains the tile metadata if a plain physical eDP EDID read returns 4K/no-TILE:
  - physical primary EDID changes after root `0x4F1` wake and a re-read;
  - a type-128/internal mode/EDID buffer already contains primary tile metadata;
  - or Windows gets a firmware/driver-provided primary tile block before this update path.

Linux implication:
- A forced primary EDID re-read after wake is still the cleanest empirical test. If it returns real primary DisplayID TILE, Linux should use it.
- If the primary still returns only 4K/no-TILE, Linux should not wait for EDID fixups to create TILE. The Windows-proven pin-cap fixup does not do that.
- The likely Linux fallback shape is:
  - keep the real primary route `0x3114`;
  - keep the real secondary route `0x3113` and real secondary EDID/TILE when available;
  - synthesize only the missing primary endpoint TILE metadata as left tile `0,0 / 2560x2880`, same group token as the proven secondary, and only for the exact Apple internal route;
  - ensure the primary exposes/keeps a valid `2560x2880` mode, either from a real EDID re-read or as a narrow derived-mode/fallback mode;
  - label this as a Linux compatibility fallback unless a later runtime trace proves Windows also constructs primary TILE from a non-EDID source.

New IDA comments from this pass:
- Added comments at `0x141A0FBF1`, `0x141A0FC75`, `0x141A0FCDB`, `0x141A3C8A9`, `0x141A3C884`, `0x14140BD5B`, `0x141507BD0`, and `0x141A75931`.

### Kernel change: Change A — forced primary EDID re-read (the empirical test of ss5936)

Date: 2026-05-26.

This implements the "cleanest empirical test" called for above (ss5936, "A forced primary EDID re-read after wake"): determine at runtime whether the iMac19,1 primary endpoint `0x3114` obtains real DisplayID TILE when re-read after the secondary `0x3113` has latched the panel into tile mode, or whether it keeps returning plain 4K/no-TILE (`0xAE25`).

Implementation: `amdgpu_dm_imac5k_reprobe_primary_after_secondary()` in `amdgpu_dm.c`, called once at the end of `amdgpu_dm_initialize_drm_device()` after the boot detect loop. Gated to DMI `iMac19,1` + exact primary/secondary routes; only fires when the secondary actually carries the TILE; idempotent (skips if the primary is already tile-aware). Logs the primary `product/manufacturer_id`, `has_tile`, tile geometry, and the `2560x2880` mode count before and after, tagged `CHANGED`/`UNCHANGED`. We deliberately implemented only Change A (read what the panel returns) and **rejected** Change B (synthesizing the missing primary TILE), since Change B manufactures state Windows was not observed producing; if the probe shows the primary never flips, the per-endpoint-TILE question (ss5936 fallback shape) is revisited with that evidence rather than pre-emptively.

New code-side finding that shaped the mechanism (Linux DC constraint, not a Windows RE finding):
- DC's `detect_link_and_local_sink()` (link_detection.c) has an **eDP early-out**: when the link already has a `local_sink` and `link->dc->config.allow_edp_hotplug_detection` is false (the default), it returns the cached sink **without re-reading EDID**. So a second `dc_link_detect()` on the primary eDP is a silent no-op for the EDID — it cannot serve as the forced re-read.
- Forcing the full detect path by flipping `allow_edp_hotplug_detection` is unsafe: `link_detect_connection_type()` then runs the eDP power sequence and, if `link_get_hpd_state()` reads low at that instant, **powers the eDP panel down** (`edp_power_control(link, false)`).
- Therefore Change A re-reads via `dm_helpers_read_local_edid(link->ctx, link, link->local_sink)` directly. That is the exact reader DC uses inside detect: a real DDC/AUX EDID read that also calls `drm_edid_connector_update()` (refreshing the DRM connector EDID + `has_tile` + tile geometry) and refreshes the DC sink `dc_edid`/`edid_caps` in place — power-neutral, no connection-type/power sequence, cannot disconnect the panel. Change A then resyncs `aconnector->drm_edid` (the cache `amdgpu_dm_connector_get_modes()` consumes) from the freshly read bytes so the two EDID layers stay coherent.

What `new_boot_7` resolves (maps directly to the ss5936 "still unresolved" bullets):
- `post-reread ... CHANGED ... product=0xae26 has_tile=1` ⇒ bullet 1 is true: the physical primary EDID does change after the root `0x4F1` wake + re-read. Linux should use the real primary TILE; the tile pair can form with the existing pieces.
- `post-reread ... UNCHANGED ... product=0xae25 has_tile=0` ⇒ bullet 1 is false for a pure software re-read: the primary endpoint does not expose TILE over a plain eDP EDID re-read. Then the remaining options are the type-128/internal buffer or firmware-provided primary tile block (bullets 2/3), or the labelled compatibility fallback (ss5936 "likely Linux fallback shape") — to be decided with this evidence.

## new_boot_7 RESULT: STAGE 2 WORKS — NATIVE 5K ON PLAIN BOOT

Date: 2026-05-26. Kernel `7.0.1-1-imac-5k-g066b89c6f6da`.

**ss5936 bullet 1 is TRUE.** The forced primary re-read flips the primary endpoint exactly as hoped, and the full tile pair forms. Plain-boot Linux (systemd-boot, no OCLP/firmware helper) now brings up native 5K `5120x2880` with no manufactured state.

Proven chain, in boot order:
- Primary flip (Change A): `primary 0x3114 pre-reread product=0xae25 has_tile=0` ext `tag=0x02 (CEA-861)` ⇒ `post-reread product=0xae26 has_tile=1 tile=2560x2880 loc=0,0 grid=2x1 ... CHANGED` ext `tag=0x70 (DisplayID)`. Raw bytes confirm it on the wire: id8_23 `... 25 ae ...` → `... 26 ae ...`. `drm_edid_connector_update [eDP-1] tile cap 0x82 size 2560x2880 location 0x0`. So the panel really does hand the eDP endpoint a DisplayID-TILE EDID once the secondary has latched it; Linux did not synthesize anything.
- Both endpoints now have complementary TILE in the same group: primary eDP-1 = left `loc 0x0`, secondary DP-1 = right `loc 1x0`, grid `2x1`, `tile_group=...b364f8e1`.
- `make-tile-preferred` fires on both: `eDP-1 tile mode 2560x2880 set as sole preferred` + `DP-1 ... set as sole preferred` (4K kept, not preferred).
- DRM forms the tile group: `[eDP-1] found tiled mode: 2560x2880` (tile 0) + `[DP-1] found tiled mode: 2560x2880` (tile 1).
- Secondary link trains at HBR2 x4: after one AUX doze (`-5` burst + DC `*ERROR* dpcd_set_link_settings`), the re-wake retry recovers — `link-config writes OK after re-wake retry attempt=1 rate=0x14 lanes=4`; stream-enable `0x4F1` latch verified.
- Genlock: `timing-sync result ... group_size=2 master=1` for `0x3114`, `master=0` for `0x3113`; `GSL: Set-up complete`; `dc_commit_state {2560x2880, 2720x2962@483250Khz}` x2 pipes.
- Modeset: `crtc-0 desired mode 2560x2880 set (0,0)` + `crtc-1 ... set (2560,0)` ⇒ **`fbdev_probe: surface width(5120) height(2880) bpp(32)`**. Stable through every re-probe to t=221s (tail is only routine `dc_allow_idle_optimizations` idle ticks — no teardown, no AUX death, no collapse to 4K/640x480).

Remaining items were cosmetic only (do not block 5K) and are now ADDRESSED in code (pending next-boot confirmation of clean logs):
1. AUX `-5` retry storm at t≈7.8-8.06 before the first link-config write succeeded. Recovered by the existing re-wake-on-failure retry, but noisy (12x `Too many retries -5` + 3 DC `*ERROR*`). FIX (link_dp_training.c `dpcd_set_link_settings`): on the exact iMac secondary route, pre-wake the panel via primary `0x4F1` + 60ms settle BEFORE the first `0x107/0x101/0x100` write, so the first attempt lands cleanly; the re-wake-on-failure retry stays as the safety net. Success log reworked to `link-config writes OK ... (pre-wake done; re-wake retries=N)` (N=0 means the pre-wake sufficed). New first line to expect: `secondary 0x3113 pre-waking panel via primary 0x4F1 before first link-config write`.
2. Misleading log: `link-cap NOT bridged ... 2560x2880 will still be pruned` printed even when `verified rate=0x14 lanes=4` was already HBR2 x4 (real training succeeded, nothing pruned). FIX (link_dp_capability.c bridge `else`): split into (a) reported known but not better => `link-cap bridge not needed: verified cap already adequate ... 2560x2880 not pruned`; (b) reported unknown => `link-cap NOT bridged: reported caps unknown (AUX read failed) ... may still be pruned`. The actual bridge logic is unchanged.

This supersedes the older "Stage 2: 4K fallback" framing: plain boot no longer drops to 4K. The genlock/timing-sync, reported-cap bridge, make-tile-preferred, secondary wake/AUX/retry, and Change A primary re-read together produce the two-tile `2560x2880 + 2560x2880` topology = one logical `5120x2880`.
