# iMacPro1,1 IDA MCP Findings

Date: 2026-06-03

Input handoff: `RE_iMacPro1_1_Windows_Driver_IDA_MCP_Handoff.md`

IDA database: `B416406/amdkmdag.sys.i64`

## Short Answer

The handoff document is enough to proceed with static RE in IDA MCP. The actual
dmesg is not required for this phase because the handoff already provides the
important runtime facts:

- `APP/AE1D` root/compat EDID is seen.
- `APP/AE1E` slave/tiled identity is seen.
- root re-read changes from `AE1D` to `AE1E`.
- both tiles are exposed to userspace as a 2x1 tiled monitor.
- root and slave `0x4F1=1` at stream enable succeeded/read back, but the left
  tile is still stretched.
- the current Linux experiment forced source DPCD `0x310 = 04 1d 03`.

The next dmesg should be a targeted validation log after the patch candidates
below, not a prerequisite for this IDA pass.

## Pro1,1 Capability Table Result

Windows contains static vendor/product rows for the iMacPro1,1 panel IDs:

| Address | Manufacturer | Product | Source id | Meaning |
|---|---:|---:|---:|---|
| `0x144C50050` | `0x1006` (`APP`) | `0xAE1D` | `0x38` | root/odd Apple 5K capability family |
| `0x144C50060` | `0x1006` (`APP`) | `0xAE1E` | `0x39` | tiled/even Apple 5K capability family |

Nearby rows show the same odd/even split for the older 5K family:

- `APP/AE25 -> source id 0x38`
- `APP/AE26 -> source id 0x39`

`MapCapabilityTableIdToPinPropertyTag()` maps:

- source id `0x38` -> tag `0x3A`
- source id `0x39` -> tag `0x3B`
- source id `0x3E` -> tag `0x40`

`SetPinDescriptorCapabilityBitById()` maps those tags into descriptor bits:

- tag `0x3A` -> descriptor byte `+6` bit 2
- tag `0x3B` -> descriptor byte `+6` bit 3
- tag `0x40` -> descriptor byte `+6` bit 5

Implication: Pro1,1 is not missing from the Windows Apple 5K path. The Windows
model treats `AE1D/AE1E` as the same Apple dual-tile family as `AE25/AE26`.

## Source DPCD `0x310`

`ProgramDpPreTrainingAuxState()` at `0x141B8E090` publishes source DPCD before
link training:

- DPCD `0x300` source OUI
- DPCD `0x303` source descriptor block
- DPCD `0x310` source table revision
- optional DPCD `0x340`

The `0x310` value is selected as:

- if `*(int *)(*(a1 + 0x158) + 0x50) >= 12`: write `05 1d 03`
- otherwise: write `04 1d 03`

Correction from the third IDA pass: this field is an internal Windows
ASIC/display-core generation ordinal, not the Linux enum value named
`DCE_VERSION_12_0`.

`dc_context+0x50` is assigned by
`ClassifyDisplayCoreVersionFromAsicId(init_data)`. The Pro1,1 GPU id `0x6860`
appears in a static device-id group marked `0x8D`, and the classifier's `0x8D`
branch returns generation `7` or `8`, both below the `>= 12` selector threshold.

Implication: for the Windows path where Pro1,1's `0x6860` adapter resolves to
family `0x8D`, Windows writes `04 1d 03`. Ordinal `7` and ordinal `8` both stay
below the `>= 12` selector threshold. The current Linux experiment that forced
`04 1d 03` is therefore aligned with that Windows-shaped path, not the cause of
the remaining stretched-left-tile failure.

The remaining static uncertainty is narrower: revision `dc_context+0x34`
chooses ordinal `7` versus `8`. That affects the DCE12/Vega resource factory
subpath and invalid-pipe fuse handling, not the `04` versus `05` source-DPCD
byte.

## `0x4F1` Writers

There is no evidence in the inspected Windows paths for a `0x4F1=0` pulse or
for `1 -> 0 -> 1`. The visible writers use payload `1`.

`StreamEnableWrite4F1AndPollSinkStatus()` at `0x1414E7810`:

- normal branch order:
  - maybe clear DPCD `0x111` bit 0
  - verify link caps
  - train or reuse cached link
  - program DPCD `0x316`
  - program stream/source settings
  - call the real stream-enable vfunc
  - if descriptor byte `+6` bit 2 or bit 3 is set, write DPCD `0x4F1=1`
  - retry once after `10 ms` if the first write status is not success
- state `1` branch:
  - programs requested link/source settings and DPCD `0x316`
  - calls stream-enable
  - transitions state `1 -> 3`
  - returns without the later tag-gated `0x4F1` write

This means absence of a late `0x4F1` write is not automatically a mismatch if
the path is a cached/already-configured analogue. But when the normal stream
enable branch is used, `AE1E`/tag `0x3B` is enough to authorize the write.

`DetectDisplayMaybeAssert4F1ForCapableSink()` is weaker for the secondary:

- it checks descriptor byte `+6` bit 2 only, matching tag `0x3A`
- it is root/odd-panel biased

`WindowsDM_EnableSecondaryTileIfRequired()` is also narrower than it first
looks:

- it accepts bit 2 or bit 3
- but also requires selected link/display type `128`
- therefore it is not the main signal-32 secondary route by itself

## Adjacent DPCD Candidates

These are now the main remaining candidates, because the deeper classifier pass
makes `0x310 = 04 1d 03` look Windows-aligned for Pro1,1:

- DPCD `0x107`: `DpStreamSetMsaTimingIgnoreOnDpcd107()` sets bit 7 when the
  stream object has the relevant nonzero state fields. Verify Linux writes
  `0x90`-style MSA timing ignore when expected, not only `0x10`.
- DPCD `0x316`: `ProgramDpcd316_StreamSourceFlag()` sets bit 0 only for stream
  modes `10`, `11`, or `12`. Log the stream mode and old/new `0x316`.
- DPCD `0x10A`: `ProgramDpcd10A_AssrBitForPanelMode()` sets/clears ASSR bit 0
  before training according to the panel policy.
- DPCD `0x111`: Windows may set MST-control bit 0 earlier, then clear it before
  stream enable. Do not treat `0x111` as a simple one-way enable.
- DPCD `0x205`: polling is gated by descriptor byte `+6` bit 5 / tag `0x40`.
  Pro1,1 static rows use `0x38/0x39`, not `0x3E`, so `0x205` is diagnostic only
  unless a separate runtime path sets tag `0x40`.

## Recommended Linux Experiments

1. Keep Pro1,1 source DPCD `0x310 = 04 1d 03`, but log the selector basis.

   Do this at the same lifecycle point as Windows: source-specific DPCD
   publication before receiver caps/training, not only from a high-level stream
   hook. Log the selected value, write status, and Linux-side generation enum so
   this does not get confused with the Windows internal classifier ordinal.

2. Stop expanding blind `0x4F1` experiments for now.

   Linux already proved root and slave `0x4F1=1` at stream enable can succeed
   without fixing Pro1,1. The Windows evidence still supports `0x4F1=1`, retry
   once after `10 ms`, but not a `0` pulse or broad repeated root writes.

3. Add focused log markers for `0x107`, `0x316`, `0x10A`, and `0x111`.

   This should answer whether stream/source flags, ASSR, and MSA-ignore differ
   even though both planes and framebuffers are correct.

4. Treat `0x205` as non-blocking.

   There is no static Pro1,1 table evidence that `0x205` polling is required.

5. Preserve the existing two-tile userspace path.

   The handoff already shows Linux exposing both tiles, CRTCs, and plane
   rectangles correctly. The next work should stay in lower DC/AUX/link/stream
   sequencing.

## Suggested Log Markers

```text
APPLE5K: source DPCD 0x310 link[%d] selector=%u value=%02x 1d 03 status=%d phase=pretraining
APPLE5K: stream DPCD 0x107 link[%d] old=%02x new=%02x status=%d msa_ignore=%d
APPLE5K: stream DPCD 0x316 link[%d] mode=%u old=%02x new=%02x status=%d
APPLE5K: training ASSR 0x10A link[%d] policy=%u old=%02x new=%02x status=%d
APPLE5K: stream DPCD 0x111 link[%d] old=%02x new=%02x status=%d phase=%s
```

## Current Patch Priority

Highest priority: do not switch Pro1,1 blindly to `05 1d 03`. The deeper
classifier pass makes Windows-shaped Pro1,1 source DPCD look like `04 1d 03`.
Keep logging enough to prove the write happens before training.

Second priority: verify `0x107` MSA timing ignore, `0x316` stream source flag,
and `0x10A` ASSR policy on both tile streams.

Lower priority: further `0x4F1` ordering experiments. The latest Linux negative
result makes another blind latch pass unlikely to be the primary fix.

## Addendum: Second IDA MCP Clarification Pass

### `0x310` selector provenance

`ConstructLegacyConnectorDisplayEntry()` stores the shared provider/context
pointer into each link object:

- `link+0x150` / `a1+336` = DC/root object
- `link+0x158` / `a1+344` = provider/context pointer from constructor input

`ProgramDpPreTrainingAuxState()` then uses that same provider/context pointer:

- `context+0x28` becomes source descriptor `0x303` bytes `0..1`
- `context+0x50` becomes source descriptor `0x303` byte `6`
- `context+0x50 >= 12` selects `0x310` byte `0 = 05`
- otherwise `0x310` byte `0 = 04`

This ties the `0x310` choice to the adapter/DC generation field used in the AMD
source descriptor, not to EDID product id or Apple capability-table rows.

Third-pass correction: this is an internal Windows classifier ordinal, not the
Linux enum value named `DCE_VERSION_12_0`.

`ConstructDcContextFromInitData()` assigns:

- `dc_context+0x50 = ClassifyDisplayCoreVersionFromAsicId(init_data)`

`PopulateDcInitDataFromWindowsDM()` shows the relevant classifier input:

- `init_data[0]` receives `adapter_info+4`
- `init_data[1]` receives `adapter_info+0` and is the classifier switch key
- `init_data[3]` receives `adapter_info+0x14` and is the revision/range field

The Pro1,1 PCI device id `0x6860` appears in a static device-id group marked
`0x8D`. The classifier branch for family `0x8D` returns generation `7` or `8`,
depending on revision. Both are below the `>= 12` selector threshold.

Corrected implication: for Pro1,1-as-family-`0x8D`, Windows writes
`0x310 = 04 1d 03`. The current Linux forced-`04` experiment is
Windows-aligned for that path.

### Stream-enable gate before late `0x4F1`

`EnableValidatedStreamMaybeAssert4F1()` at `0x141446B40` is the only direct
caller of `StreamEnableWrite4F1AndPollSinkStatus()`.

The wrapper first validates that the stream/VC entry has a mapped present sink.
Then `UpdateMode11ImmediateStreamEnableFlag()` at `0x141442720` queries the
display object stored in `stream_arg+0x178`:

- display-object mode `11` sets the immediate flag and enters
  `StreamEnableWrite4F1AndPollSinkStatus()`
- display-object mode `12` clears the immediate flag
- non-mode-11 paths continue through generic cached stream setup instead

The display object at `stream_arg+0x178` is populated by
`BuildStreamArgForDisplayIndex_SetObject178()` from a display-index lookup. So a
future Linux comparison should log the resolved stream/display-object mode or
branch analogue, not only EDID/DisplayID TILE contents.

The generic cached path is real work, not a no-op:

- `ProgramCachedLinkStreamSourceThenEnableStream()` programs peer/source state,
  programs stream source with the cached tuple, marks the display object enabled,
  then calls the real stream-enable vfunc.
- `LinkServiceHasCachedLinkSettings()` only checks cached tuple presence
  (`link_service+0x8C != 0`). It does not include HPD/link-active in that
  predicate.

Implication: late `0x4F1` is branch-specific. If Linux is intentionally following
a cached-link/finalizer analogue, lack of another late `0x4F1` is not itself a
Windows mismatch. The more important condition is whether source-DPCD, ASSR,
stream/source state, and cached tuple are all valid before the first two-tile
finalization.

### Pro1,1 tag values

The raw Pro1,1 table rows have value dword `0`.

That matters for the post-bring-up helper path where tag `0x3B`'s value is copied
into sink-info and can gate a later helper. But the stronger `0x4F1` paths check
descriptor-bit presence directly:

- detect/status helper: byte `+6` bit 2 only
- stream-enable helper: byte `+6` bit 2 or bit 3

So `AE1E` / tag `0x3B` with value `0` still qualifies for the stream-enable
descriptor-bit path. The zero value mostly explains why the separate
value-gated post-bring-up helper is not the main Pro1,1 target.

### Adjacent DPCD branch details

`0x107`:

- `DpStreamSetMsaTimingIgnoreOnDpcd107()` reads DPCD `0x107`.
- It sets bit 7 when stream fields `+0x5C` or `+0x60` are nonzero.
- Existing IDA notes tie `+0x5C` to packed display-object identity, so this is a
  strong diagnostic. Linux should log old/new `0x107`, especially whether it
  reaches `0x90` when starting from `0x10`.

`0x316`:

- `ProgramDpcd316_StreamSourceFlag()` runs in both state-1 and normal
  stream-enable paths.
- It sets bit 0 only when stream arg mode at `+0x88` is `10`, `11`, or `12`.
- This should be logged with the same stream/display-object mode that controls
  the immediate `0x4F1` wrapper.

`0x10A`:

- `ProgramAssrThenTrainLinkSettings()` computes panel policy, writes DPCD
  `0x10A`, then trains.
- `ClassifyDpPanelModeForAssrPolicy()` has a visible object-low-byte `0x13`
  path: for the iMac secondary `0x3113`, a display-object predicate can return
  policy `1`, which sets ASSR bit 0.
- This is a pre-training requirement, not a late stream-enable tweak.

### Revised next test

The narrow next Linux experiment should be:

1. Keep lower source DPCD `0x310 = 04 1d 03` for Pro1,1, and prove it happens
   before caps/training/link-setting writes.
2. Log the Linux generation enum and the Windows-analogue selector separately so
   nobody reinterprets Linux `DCE_VERSION_12_0` as Windows ordinal `>=12`.
3. Log the resolved stream/display-object mode used by the stream-enable path.
4. Log `0x10A`, `0x107`, and `0x316` old/new values around the same first
   accepted two-tile commit.
5. Do not add new broad `0x4F1` pulses unless those logs show Windows-equivalent
   setup is complete but the panel still needs a branch-specific latch.

### Third-pass `0x310` writer check

Searching immediate DPCD address `0x310` found one additional display-side
writer at `0x1414DD180`, called from `RefreshDpStreamCapsAndConverterData()`.
This writer uses a different payload shape (`02 04 <bit>`) and sits in the
stream-capability/converter-refresh path. It is not the lower
`ProgramDpPreTrainingAuxState()` `04/05 1d 03` publication that Linux has been
modelling.

Practical implication: keep the source-DPCD pretraining log focused on
`ProgramDpPreTrainingAuxState()` equivalence, but add a separate diagnostic if
we want to know whether Windows' stream-capability refresh has a Linux analogue.

## Phase 1 Resource-Pool Result

The first concrete Pro1,1/iMac19,1 split is now the resource-pool constructor:

- iMac19,1/Polaris baseline is ordinal `5/6` and selects
  `Dce112ResourceConstruct()`.
- Pro1,1/`0x6860`/Vega, once classified as family `0x8D`, is ordinal `7/8` and
  selects `Dce12VegaResourceConstruct()`.
- Ordinal `7` uses `DCE12_VEGA_FACTORY_ORD7_NO_INVALID_PIPE`.
- Ordinal `8` uses `DCE12_VEGA_FACTORY_ORD8_INVALID_PIPE` and reads pipe fuses
  via `Dce12ReadPipeFuseInvalidMask()`.
- Both DCE12/Vega subpaths use `DCE12_VEGA_RESOURCE_CAPS_TABLE`.

Important correction: HPO is not active in the Pro1,1 ordinal `7/8` constructor.
The consumed HPO factory slots are null and `DCE12_VEGA_RESOURCE_CAPS_TABLE`
sets HPO FRL/DP stream/link counts at indices `12`, `13`, and `14` to zero. The
nearby HPO helper table belongs to another constructor xref, not this Pro1,1
path.

The high-value remaining diff is therefore the DCE12/Vega ordinary stream
encoder register tables, final resource service variant, revision-gated
pipe/TG mapping, per-pipe defaults, and the downstream stream-source payload.

## Phase 2 Link/AUX Route Result

The Pro1,1 route appears to keep the same logical Apple 5K connector object
roles as iMac19,1, but not the same lower AUX/DDC namespace.

Logical object side:

- Pro1,1 handoff root link id `3:20:1` corresponds to low byte `0x14` /
  object `0x3114`.
- Pro1,1 handoff slave link id `3:19:1` corresponds to low byte `0x13` /
  object `0x3113`.
- `ConstructLegacyConnectorDisplayEntry()` maps `0x14` to signal `128` and
  `0x13` to signal `32`.
- Cross-checking Linux's `translate_encoder_to_transmitter()` mapping makes the
  handoff's transmitter/engine `4/5` shape most consistent with raw UNIPHY2
  objects `0x2121/0x2221`, rather than iMac19,1's known `0x2120/0x2220`
  transmitter `2/3` route.

Lower AUX/DDC side:

- `InitializeAdapterServiceForConnector()` passes `CBasePin` signal/generation
  into `CreateAuxTransactionPathFactory()`.
- `CBasePinGetAuxGenerationFromProperty2()` maps pin capability property `[2]`
  value `274` to AUX generation `11`, and value `288` to AUX generation `12`.
- `CreateAuxPinMapFactoryByGeneration()` maps generation `11` to the DCE11
  factory and generation `12` to the DCE12/Vega factory.
- DCE11 simple selectors match the old iMac19,1 baseline:
  `0x4871 -> AUX pin 2`, `0x4875 -> AUX pin 3`.
- DCE12 simple selectors are different:
  `0x55A7 -> pin 0`, `0x55AB -> pin 1`, `0x55AF -> pin 2`,
  `0x55B3 -> pin 3`, then `0x55B7/0x55BB/0x55BF/0x55D7 -> pins 4..7`.

This fits the handoff's Pro1,1 route better than the iMac19,1 constants:

- Pro1,1 root reports `ddc_hw=1`, `hpd_src=1`, encoder/engine `5`.
- Pro1,1 slave reports `ddc_hw=0`, `hpd_src=0`, encoder/engine `4`.
- iMac19,1 baseline was root DDC4/AUX pin `3`, secondary DDC3/AUX pin `2`.

Certainty boundary:

- Static IDA proves the generation-11 versus generation-12 fork and the two
  selector ladders.
- Static IDA does not contain the live Pro1,1 VBIOS descriptor contents, so it
  cannot prove the exact Pro1,1 selector values by itself.
- A narrow runtime marker should log base pin property `[2]`, selected AUX
  generation, raw selector/source mask, resolved group/pin, `ddc_hw`, HPD, and
  transmitter for `0x3113` and `0x3114`.

Practical implication:

Do not carry the iMac19,1 route constants `0x4871/0x4875`, DDC3/DDC4, or AUX
pins `2/3` into Pro1,1-specific logic without checking the generation-12 route.
This is now a better Pro1-vs-19,1 explanation candidate than another `04` versus
`05` source-DPCD attempt, although it still needs to be tied to the final
stream-source/role state before calling it the whole cause of the stretched
left-tile symptom.

## Phase 3 Stream-Mode Result

The mode-11 stream-enable branch is now tied to display-object SetMode state:

- `BuildStreamArgForDisplayIndex_SetObject178()` stores the selected display
  object at `stream_arg+0x178`.
- `UpdateMode11ImmediateStreamEnableFlag()` calls that object vfunc `+0x160`.
- full display-object vfunc `+0x160` is
  `DisplayObject_GetCommittedSignalTypeForModeGate()`.
- this method reads committed signal type from display interface `+0x48`.
- if byte `+0x1E9` is set and committed type is `12`, it reports `11`;
  otherwise it reports the committed type unchanged.

The `+0x1E9` byte is set or cleared by SetMode, not by EDID parsing directly:

- `DisplayObject_SetSignal12ReportsAs11Flag()` is vfunc `+0x318`.
- `SetMode_ApplyMstToSstOptimizationSignalCoercion()` calls vfunc `+0x318`.
- it requires all displays to have at most two tiles, then selects an
  MST-to-SST optimization anchor.
- anchor path metadata bit `7` at `path_entry+0x18 -> +0x1C` controls the byte:
  bit clear sets `+0x1E9`, bit set clears it.

Branch result:

- mode gate `11` sets the immediate byte and calls
  `StreamEnableWrite4F1AndPollSinkStatus()`.
- mode gate `12` clears the immediate byte and continues through the
  generic/cached stream-source finalizer.
- DPCD `0x316` bit `0` is broader: `ProgramDpcd316_StreamSourceFlag()` sets it
  for modes `10`, `11`, or `12`. So seeing `0x316` set does not prove the
  immediate mode-11 `0x4F1` worker ran.

Pro1,1 implication:

The next log should record committed mode, byte `+0x1E9`, resulting mode gate,
anchor path bit `7`, cached-link state, and immediate flag. If Pro1,1 is plain
mode `12` with byte `+0x1E9` clear, the absence of this wrapper's late `0x4F1`
write is Windows-shaped. If Pro1,1 is mode `11` or coerced `12->11`, then Linux
should reach the immediate worker analogue before judging the final stream role.

## Phase 4 Stream-Source Role Result

Track 2 found the upstream source of the final source-role payload.

Source-index allocation:

- `DisplayMapResolveAvailableSourceIndex()` at `0x1414AB040` resolves the
  per-stream source index.
- It uses full display-object vfunc `+0x288`,
  `DisplayObject_GetPendingSignalType()`, not the later committed/mode-gated
  vfunc `+0x160`.
- Pending type `12` uses a resource-priority cutoff of `6`; other pending types
  use cutoff `7`.
- It first asks the stream path object for a preferred source index through
  vfunc `+0x140`.
- If that is unavailable or unsuitable, it reads the path object's source mask
  through vfunc `+0x138`, scans display-map resource bucket `10`, and picks the
  available candidate with the lowest resource priority at entry `+0x18`.
- `DisplayMapAssignStreamSourceIndex()` at `0x1414AADD0` writes the chosen index
  into the display object through vfunc `+0x480(stream=0, index)`.
- That setter is now named `DisplayObject_SetStreamSourceIndex()` at
  `0x141549B60`; it stores the value at per-stream `+0x38`.
- The getter is `DisplayObject_GetStreamSourceIndex()` at `0x141549EC0`, vfunc
  `+0x98`.

Final role payload:

- `BuildStreamSourceRolePayloadFromDisplayObject()` at `0x141560E60` builds the
  payload sent to the selected path object's vfunc `+0x70`.
- Observed dword layout:
  - `[0]`: source index from `DisplayObject_GetStreamSourceIndex()`.
  - `[1..2]`: base object fields from the base display-object id interface.
  - `[3]`: committed/mode-gated signal from vfunc `+0x160`.
  - `[4]`: current display object core id.
  - `[5]`: peer/next path object core id, or base object core id as fallback.
  - `[6]`: display index.
  - `[7]`: flags; bit `8` comes from a capability byte.
- `CollectActivePathObjectsForSourceRolePayload()` at `0x1415653E0` selects the
  first active stream and collects current path, peer path, stream/source
  interface, auxiliary object, and source index.
- `HwSeqProgramStreamSourceWithLinkSettings()` performs two lower operations:
  first it calls the selected stream/source interface vfunc `+0x70(pipe_or_tg,
  mode_value, packet+0x10)`, then it sends the role payload above through the
  active path object's vfunc `+0x70(payload)`.
- All observed HW sequence vtables point at the same final source-program and
  timing-generator implementations, so the Pro1,1-vs-19,1 difference is more
  likely in the source index/object/mode data fed to the common code than in a
  separate DCE12-only payload builder.
- `NormalizeSourceProgramModeValue()` at `0x1415613D0` leaves 5K-relevant modes
  `10`, `11`, and `12` unchanged; its only observed rewrite is a legacy mode
  `3 -> 1` low-bandwidth/timing case.

Important divergence candidate:

Source-index allocation can be based on pending signal type `12`, while the
final role payload and mode-11 branch use committed/mode-gated signal type from
`DisplayObject_GetCommittedSignalTypeForModeGate()`. That does not prove a bug
by itself, because it is the Windows shape, but Linux should log both views.

Track 2 runtime marker should therefore record, for each tile:

```text
APPLE5K: final source role link[%d] obj=0x%04x peer_obj=0x%04x pending_mode=%u committed_mode=%u mode_gate=%u src_index=%d display_index=%u flags=%08x pipe_or_tg=%d mode_value=%u
```

This is now the best RE target for the "left tile stretched across whole panel"
symptom. More `04/05 1d 03` testing is lower value unless this final role state
already matches Windows.

### Phase 4 follow-up: active path and peer-source bridge

The next IDA pass made the source-role path more concrete.

Display-object path selection has two different views:

- Raw path getters are used by allocation/detect retain:
  - `DisplayObject_GetStreamPathObjectRaw()` at `0x141549E60`, vfunc `+0x450`,
    returns stream path `+0x10` without checking the active bit.
  - `DisplayObject_GetPeerStreamPathObjectRaw()` at `0x141549E00`, vfunc
    `+0x458`, returns the next raw path without checking the active bit.
  - `DisplayObject_GetStreamAuxObjectRaw()` at `0x141549D40`, vfunc `+0x460`,
    returns per-stream field `+0x18` without checking valid bit 2.
- Final role programming uses active-gated getters:
  - `DisplayObject_GetActiveStreamPathObject()` at `0x14154A090`, vfunc `+0x78`,
    requires per-stream active bit 0 at `+0x08`.
  - `DisplayObject_GetActivePeerStreamPathObject()` at `0x14154A010`, vfunc
    `+0x80`, requires the current stream active bit and then returns the next
    stream path.
  - `DisplayObject_GetValidStreamSourceInterface()` at `0x141549FA0`, vfunc
    `+0x88`, requires valid bit 1.
  - `DisplayObject_GetValidStreamAuxObject()` at `0x141549F20`, vfunc `+0x90`,
    requires valid bit 2.

`DisplayMapRetainPathEntriesAndSetActiveFlags()` at `0x1414ABDE0` is the detect
retain bridge between those two worlds. On notify passes it calls display-object
vfunc `+0x488` to set active bit 0, and vfunc `+0x498` to set auxiliary valid
bit 2. This means source-index allocation can succeed from raw stream 0 while
the final payload still uses the wrong current stream, no peer stream, or a
fallback peer object if the active bits or stream order differ.

The DCE path wrapper layer is now also mapped enough for track 2:

- DCE wrapper vtables `0x144AFBBC0` and `0x144AFBFE0` share the same source-role
  forwarding slots.
- wrapper `+0x70`, `DcePathWrapper_ForwardRolePayloadToInner()` at
  `0x1416E0080`, forwards the role payload to the inner object at wrapper
  `+0x58`, inner vfunc `+0x68`.
- wrapper `+0x140`, `DcePathWrapper_GetPreferredSourceIndexViaInner()` at
  `0x1416DF820`, forwards to inner vfunc `+0x120`.
- wrapper `+0x138`, `DcePathWrapper_GetSourceMaskViaInner()` at `0x1416DF860`,
  forwards to inner vfunc `+0x128`.
- the remaining static uncertainty is the exact inner implementation behind
  wrapper field `+0x58`, but the wrapper-level role/source-mask/preferred-index
  forwarding is certain.

Peer-source refresh is a real finalization step, not a side note:

- `ProgramPeerSourceUpdateBeforeStreamEnable()` at `0x141442470` runs before
  cached source programming in `ProgramCachedLinkStreamSourceThenEnableStream()`
  at `0x1414431B0`.
- The non-cached packet path, now named
  `ProgramStreamSourcePacketThenPeerSourceUpdate()` at `0x141442FD0`, programs
  HWSeq vfunc `+0x90` and then calls the same peer-source refresh.
- The peer refresh is skipped if the service flags byte returned through
  `a1+0x48` has bit `0x08` set at offset `+7`.
- Otherwise it walks the stream VC table: count at table `+0xB0`, entry pointer
  `entries_base(+0xA8) + 0x3B0 * index`.
- It only uses entries whose `entry+0x338` has sink-present bit 0 and
  payload-allocated bit 2 set.
- The first dword of the display record at `entry+0x350` is a display index.
  It is passed through the stream full `+0x98` interface, which is the
  topology-manager full `+0x38` notification interface, not HWSeq and not a
  source-role manager.
- The resolved vfunc targets are:
  - `+0x40` -> `TopologyNotifyPreStreamDisableForDisplay()` at `0x14142F180`
  - `+0x48` -> `TopologyNotifyPostStreamDisableForDisplay()` at `0x14142F070`
  - `+0x50` -> `TopologyMaybeIssuePplibCmd38ForDisplay()` at `0x14142F370`
  - `+0x58` -> `TopologyNotifyPostStreamEnableForDisplay()` at `0x14142EE40`

HWSeq has two relevant source-role consumers:

- HWSeq `+0x88`, `HwSeqProgramStreamSourceWithLinkSettings()` at `0x14155BFA0`,
  calls the selected stream/source interface vfunc `+0x70`, then builds the
  common role payload and sends it to the active path object's vfunc `+0x70`.
- HWSeq `+0x90`, now named
  `HwSeqProgramSourceRolePayloadViaPath78And170()` at `0x14155BE80`, selects a
  stream path through active-gated display-object vfunc `+0x78`, builds the same
  role payload, then sends it to path vfuncs `+0x170` and `+0x78`.

Superseded next target:

The following pass corrected the wrapper-level model. The active path stored in
the DisplayObject is a display-map transport path, and that object delegates to
generation-specific lower path/source objects. A still later pass corrected the
peer-refresh model as well: stream `+0x98` is the topology notification
interface, and the peer-refresh calls emit stream-disable/enable notifications
plus an optional PPLIB command-38 power-policy refresh. They do not consume
source masks, preferred source indices, or the source-role payload.

If Pro1,1 differs from iMac19,1 here, the failure can be explained without any
new `0x310` byte theory: Windows may be refreshing paired source state and
programming a role payload that Linux never quite matches.

### Phase 4 continuation: real active path class and DCE12 source split

The wrapper-slice hypothesis was corrected in the next static pass. The active
path object stored in the DisplayObject is not the DCE12 controller/timing
generator interface directly. Construction goes through the display-map
transport path:

- `CreateDisplayTransportInterface()` constructs `DisplayTransportPath_Construct()`
  at `0x1415C8440` and returns the `full+0x28` interface whose vtable is
  `off_141FB2538`.
- `DisplayObject_AddPathInterface()` stores that transport-path interface at the
  per-stream path field `+0x10`.
- `DisplayTransportPath_BindLowerByObjectAndAuxGen()` at `0x1415C6600` binds the
  transport path to a lower generation-specific path object.
- For DP/eDP-style transport-object low bytes `0x1E`, `0x20`, `0x21`, and
  `0x25`, AUX generation `11` selects `Dce11DpPathLower_Construct()` at
  `0x14161E610`, while AUX generation `12` selects
  `Dce12DpPathLower_Construct()` at `0x141621AA0`.

The relevant transport path vfuncs are now devirtualized:

- transport `+0x138`, `DisplayTransportPath_GetSourceMaskViaLower()`, delegates
  to lower `+0x138`.
- transport `+0x140`, `DisplayTransportPath_GetPreferredSourceViaLower()`,
  delegates to lower `+0x140`.
- transport `+0x70` and `+0x78` delegate to lower `+0x70/+0x78`; on these DP
  lower classes both targets are no-op `return 0` methods. Therefore the HWSeq
  `+0x88` active-path `+0x70(role_payload)` call is not the final source-role
  register writer on this route.
- transport `+0x170`, `DisplayTransportPath_ProgramSourceModeViaLower()`,
  delegates to lower `+0x170`, which forwards `role_payload[0]` source index and
  `role_payload[3]` mode-gated signal to a generation-specific source object.

The concrete DCE11-vs-DCE12 source-allocation divergence is the source mask:

- `Dce11DpPathLower_GetSourceMaskWithSrc5Fallback()` at `0x14161E4A0` returns
  `1 << preferred_source`, plus source bit `5` when the connected pin reports
  state `6`.
- `Dce12DpPathLower_GetSourceMaskPreferredOnly()` at `0x141621970` returns only
  `1 << preferred_source`.
- The preferred-source getter is common:
  `DpPathLower_GetPreferredSourceIndex()` at `0x141622FC0` returns the
  constructor's RealState-derived source field.

The final `+0x170` source-mode write also splits by generation:

- `DpPathLower_CreateSourceObjectByAuxGen()` at `0x14169D810` creates a DCE11
  source object with `Dce11SourceObject_Construct()` or a DCE12 source object
  with `Dce12SourceObject_Construct()`.
- `Dce11SourceObject_ProgramDpSourceIndexMode()` at `0x1416A5740` uses
  `dword_141F8CB38[source]` and writes the DCE11 DP/eDP source-control register
  at offset `+19139`.
- `Dce12SourceObject_ProgramDpSourceIndexMode()` at `0x1416B1D90` uses
  `Dce12SourceRegisterBlockOffsetBySource[source]` and writes the DCE12/Vega
  DP/eDP source-control register at
  `Dce12SourceRegisterBaseOffset + offset + 6465`.

Implication:

Pro1,1's DCE12 path is not just the iMac19,1 DCE11 path with different AUX pins.
For final source allocation, the DCE12 lower path is stricter: it exposes only
the preferred source bit, while the DCE11 path can expose a source-5 fallback.
For the later source-mode role write, both paths consume the same payload fields
but land in different generation-specific register tables. This is a concrete
Windows reason for Pro1,1 to behave differently while sharing the Apple 5K EDID
capability rows and the `0x4F1` latch logic.

The later Pro1,1 VBIOS decode below resolves the exact preferred-source
RealState mapping for the two Apple 5K paths. The later peer-refresh pass
resolved stream `+0x98` as topology notifications rather than source-role
selection, so the higher-value static target remains the final active
current/peer path and DCE12 source `5/4` role-control model.

### Phase 4 continuation: RealState/source ordinal bridge

The source ordinal bridge is now mapped far enough to remove one major
uncertainty, and to identify the exact remaining one.

IDA names/comments added and saved:

- `TransportObjectIdToRealStateSourceOrdinal()` at `0x141622180`
- `CBaseRenderer_ConstructDecodeTransportObject()` at `0x141623B20`
- `DpPathLower_SetPreferredSourceIndex()` at `0x1416225F0`
- `DpPathLower_GetSourceObject()` at `0x14169D600`

`TransportObjectIdToRealStateSourceOrdinal()` decodes the raw class-2 transport
object ID from the VBIOS/display-path object chain:

| Raw class-2 object | Resulting RealState/source ordinal |
|---:|---:|
| type `0x1E`, enum `1` | `0` |
| type `0x1E`, enum `2` | `1` |
| type `0x20`, enum `1` | `2` |
| type `0x20`, enum `2` | `3` |
| type `0x21`, enum `1` | `4` |
| type `0x21`, enum `2` | `5` |
| type `0x25`, enum `1` | `6` |
| type `0x22`, enum `1` | `7` |
| type `0x23`, enum `1` | `8` |
| type `0x23`, enum `2` | `9` |

`CBaseRenderer_ConstructDecodeTransportObject()` stores that value at full
object `+0x30`. The DP lower constructors then copy it to the lower-path
preferred source at full `+0x48`:

- DCE12 maps RealState `0..5` directly to preferred source `0..5`.
- DCE11 maps RealState `0..4` directly, but RealState `5` only becomes source
  `5` when the connected pin reports state `6`.

The constructor-side bridge is also proven. `ExpandDisplayPathAndConstructStreams()`
at `0x1414B7AC0` and the alternate builder at `0x1414B6380` ask the BIOS object
service for child objects; when a child object has class nibble `2`, they call
`CreateDisplayTransportInterface()` with that raw object ID. Therefore the
RealState mapping is fed by the raw class-2 VBIOS path object, not by the final
class-3 endpoint ID and not by Linux's normalized encoder enum.

Known-good iMac19,1 now decodes cleanly:

- `0x3114` primary/root path has raw class-2 chain object `0x2120`.
  - type `0x20`, enum `1` -> source ordinal `2`.
- `0x3113` secondary path has raw class-2 chain object `0x2220`.
  - type `0x20`, enum `2` -> source ordinal `3`.

The working Pro1,1 VBIOS now proves the previously suspected UNIPHY2 route:

- `0x3114` primary/root path uses raw class-2 object `0x2221`.
  - type `0x21`, enum `2` -> source ordinal `5`.
- `0x3113` secondary/slave path uses raw class-2 object `0x2121`.
  - type `0x21`, enum `1` -> source ordinal `4`.

DCE12 exposes only the single preferred-source bit, while the iMac19,1 DCE11
route has the source-mask fallback behavior described above.

The final consequence is also proven. `role_payload[0]` is the stream source
index, `DpPathLower_ProgramSourceModeViaSourceObject()` at `0x14169DB60`
forwards it with `role_payload[3]` to the generation-specific source object, and
`Dce12SourceObject_ProgramDpSourceIndexMode()` at `0x1416B1D90` indexes
`Dce12SourceRegisterBlockOffsetBySource[source]` before writing the DCE12/Vega
DP/eDP source-control register. That table is `{0, 0x100, 0x200, 0x300, 0x400,
0x500}`, so source ordinals `4/5` are first-class DCE12 register blocks, not an
error path. A wrong source index therefore changes the hardware register target.

### Phase 4 conclusion: Pro1,1 VBIOS resolves the source ordinal

The working Pro1,1 VBIOS artifact is now available as
`imacpro11-vbios.rom`.

- Part number: `113-D05001A1XT-018`.
- SHA256:
  `73AE9942052F17F6E1201409C3C55C98962A5050362D55E3D8E378CE936F16F8`.
- ATOM display object info table: revision `1.4`, offset `0x83c`.

The v1.4 inline display-path table removes the previous uncertainty:

- `0x3114` primary/root eDP path has raw class-2 encoder object `0x2221`.
  - type `0x21`, enum `2` -> Windows source ordinal `5`.
  - DCE12 source-control register block: `+0x500`.
- `0x3113` secondary/slave DisplayPort path has raw class-2 encoder object
  `0x2121`.
  - type `0x21`, enum `1` -> Windows source ordinal `4`.
  - DCE12 source-control register block: `+0x400`.

That is different from the known-good iMac19,1 ROM:

- iMac19,1 `0x3114 -> 0x2120` -> Windows source ordinal `2`.
- iMac19,1 `0x3113 -> 0x2220` -> Windows source ordinal `3`.

So the Pro1,1 final source-index fork is no longer hypothetical. Windows routes
the same Apple 5K logical pair through UNIPHY2/source `5/4` on DCE12/Vega,
while the iMac19,1 path uses UNIPHY1/source `2/3` on DCE11/Polaris.

## Current Answer: Why Pro1,1 Still Fails

The later IDA passes weaken the earlier `0x310 = 05 1d 03` theory. The current
best answer is:

- Pro1,1 is not failing because Windows lacks its panel IDs.
- Pro1,1 is not failing because Linux used `0x310 = 04 1d 03`; under the
  family-`0x8D` Windows path, that is the pre-training value for
  Vega/`0x6860`.
- The user's recollection that an earlier `05 1d 03` attempt also failed points
  the same way: the remaining bug is not explained by the `0x310` byte choice.
- Pro1,1 is not fixed by more simple `0x4F1=1` latches; the handoff already
  proves root and slave stream-enable latches can succeed and read back.
- Pro1,1 likely does differ from iMac19,1 in the lower AUX/DDC selector
  namespace: iMac19,1 uses DCE11 selectors/pins `0x4871/0x4875 -> 2/3`, while
  the Pro1,1 handoff's `ddc_hw=0/1` shape matches the DCE12/Vega
  `0x55A7/0x55AB -> 0/1` style route.
- Pro1,1 also differs in the DisplayTransportPath lower source model:
  DCE12 exposes only `1 << preferred_source` as its source mask, while the DCE11
  path can add source bit `5` as a fallback when the connected pin reports state
  `6`.
- The preferred source is decoded from the raw class-2 VBIOS path object. The
  working iMac19,1 ROM uses `0x2120/0x2220`, which Windows decodes to source
  `2/3`. The working Pro1,1 ROM uses `0x2221/0x2121` for the Apple 5K root/slave
  pair, which Windows decodes to source `5/4`.
- The final source-mode writer reached through path vfunc `+0x170` consumes
  `role_payload[0]` and `role_payload[3]`, then lands in DCE12/Vega register
  table `Dce12SourceRegisterBlockOffsetBySource` instead of the DCE11 table
  `dword_141F8CB38`.
- The peer-refresh helper at `0x141442470` is now devirtualized as topology
  Pre/Post stream notifications plus optional PPLIB command `38`. It is
  order-sensitive, but it does not choose or consume the source ordinal.
- The late stream-enable branch also depends on SetMode/tiled MST-to-SST state:
  committed mode `12` only behaves as mode `11` when display-object byte
  `+0x1E9` is set by the optimization path.
- The final stream-source role is more specific than either DPCD `0x316` or the
  `0x4F1` latch: Windows emits source index, current object id, peer/fallback
  object id, mode-gated signal, display index, and flags in one role payload.
  A mismatch in `src_index` is still the sharper Pro1,1 suspect; a
  `peer_obj`/fallback mismatch is lower probability now that both Pro1,1 and
  iMac19,1 VBIOS paths appear to be one-encoder-per-endpoint.

The remaining RE target versus the working iMac19,1 path is now narrower: prove
that Linux's final DCE12/Vega source-role/source-control path is using the
confirmed Pro1 source pair `5/4`, then continue the remaining branch checks:

- whether the secondary stream reaches the correct Windows branch analogue
  (`mode 11`, coerced `12->11`, plain `12`, cached finalizer, or immediate
  mode-11 path);
- whether `0x10A` ASSR is asserted before training and carried into training
  policy;
- whether non-converter MSA-ignore writes DPCD `0x107` bit 7 (`0x10 -> 0x90`);
- whether DPCD `0x316` bit 0 is set when Windows would see stream mode
  `10/11/12`;
- whether the secondary link remains physically alive through final stream
  enable, instead of using stale cached proof after AUX/HPD has already weakened.

The visible symptom, "left tile stretched across the whole panel", fits a panel
composition/source-role problem better than a userspace TILE layout problem:
Linux has both tile streams and correct CRTC/plane rectangles, but the panel/TCON
still appears to consume or compose the left stream as if the paired source state
is incomplete.

### Phase 5 continuation: peer refresh devirtualized

The stream/link-context `+0x98` target in
`ProgramPeerSourceUpdateBeforeStreamEnable()` is now devirtualized and the
previous "source-manager" label is superseded.

Construction chain:

- `BuildStreamForDisplayEntryPath()` writes stream init `+0x10` from
  `TopologyDisplayPathBuilder+0x60`.
- `ConstructTopologyDisplayPathBuilder()` sets builder `+0x60` from its init
  slot `+0x40`.
- `ConstructDisplayTopologyManager()` fills that builder init slot with the
  topology-manager full `+0x38` subinterface.
- `ConstructBaseStreamObjectFromInit()` copies init `+0x10` into stream full
  `+0x98`.
- Therefore `ProgramPeerSourceUpdateBeforeStreamEnable()` dispatches through
  topology-manager vtable `off_141F978F8`.

Resolved peer-refresh calls:

| Slot | Target | Behavior |
|---:|---:|---|
| `+0x40` | `0x14142F180` / `TopologyNotifyPreStreamDisableForDisplay()` | validates display index, builds event id `59`, logs `Notify PreStreamDisable`, dispatches it, and can notify display power policy |
| `+0x48` | `0x14142F070` / `TopologyNotifyPostStreamDisableForDisplay()` | validates display index, builds event id `63`, logs `Notify PostStreamDisable`, dispatches it |
| `+0x50` | `0x14142F370` / `TopologyMaybeIssuePplibCmd38ForDisplay()` | validates display index and, when the display sink policy says yes, calls core `+0x88` display-power-policy vfunc `+0x98` |
| `+0x58` | `0x14142EE40` / `TopologyNotifyPostStreamEnableForDisplay()` | validates display index, builds event id `60` with display payload, logs `Notify PostStreamEnable`, dispatches it, and can notify display power policy |

The display-power-policy callback is now named
`DisplayPowerPolicy_IssuePplibCmd38()` at `0x141535850`. It forwards to the
inner PPLIB table callback `PplibCmd38_DisplayPowerRefresh()` at `0x1415B9B20`,
which sends command `38` through `PplibSendCommand24()` at `0x1415B8D70` with
no payload.

Implication:

- The peer-refresh step is real and order-sensitive, but it is a topology event
  and power-policy notification sequence.
- It does not explain a wrong Pro1,1 source ordinal by itself, because it never
  consumes `role_payload[0]`, preferred source, or source mask.
- The source-role path remains the DCE12 transport/lower-path `+0x170` route
  already mapped above: Pro1,1 root/slave should still resolve to source `5/4`.

### Phase 5 continuation: path insertion and peer fallback

The DisplayObject path construction/order target is now narrowed.

Windows path insertion mechanics:

- `DisplayObject_AddPathInterface()` at `0x14154AD10` stores a path in the next
  of two per-display stream slots. The insertion order defines stream index.
- `ConstructDisplayObjectAndStreams()` attaches the accumulated BIOS path array
  in reverse order: `for (i = path_count; i > 1; --i)`.
- `AttachChildInterfacesToDisplayObjectPath()` captures the old path count
  before calling display-object vfunc `+0x3D8` /
  `DisplayObject_AddPathInterface()`. It then stores any associated child
  AUX/source interface at that same stream index.
- `CollectActivePathObjectsForSourceRolePayload()` scans stream indices from
  `0` upward and stops at the first active stream.
- `DisplayObject_GetActivePeerStreamPathObject()` is strict `index + 1` with no
  wraparound. A display object with only one active path has no peer path.

VBIOS comparison:

- The working Pro1,1 v1.4 ROM uses direct one-encoder paths:
  - `path[0]`: `0x3114 -> 0x2221 -> source 5`
  - `path[1]`: `0x3113 -> 0x2121 -> source 4`
- The known-good iMac19,1 v1.3 ROM has the same one-encoder-per-endpoint shape:
  - `path[0]`: `0x3114 -> 0x2120 -> source 2`
  - `path[1]`: `0x3113 -> 0x2220 -> source 3`

Implication:

- There is no current static evidence that Pro1,1 gets a different multi-path
  insertion order or a different active peer path shape than iMac19,1.
- For these direct one-encoder Apple endpoint paths, `role_payload[5]` is
  expected to fall back to the base display object id when no `index + 1` path
  exists. That fallback appears Windows-shaped for both machines, not a
  Pro1-only divergence.
- This leaves the DCE12 source `5/4` role/control path, stream branch, ASSR,
  MSA-ignore, `0x316`, and converter/stream-capability paths as higher-value
  remaining checks.

### Phase 6 continuation: stream-DPCD finalizer and mode split

The converter/stream-capability branch is now mostly decoded.

Stream-capability writer:

- `WriteDpStreamCapsSourceDpcd310_0204Flag()` at `0x1414DD180` writes DPCD
  `0x300` length `10` and then DPCD `0x310` length `3`.
- The `0x310` payload is little-endian `0x0402` plus one flag byte, i.e.
  `02 04 <flag>`. This is separate from the earlier pre-training
  `04/05 1d 03` source-table-revision payload.
- The third byte comes from adapter-service vfunc `+0x3F8`, now named
  `AdapterServiceShouldSetDpcd310ExternalPanelDrrFlag()` at `0x141460930`.
- That helper returns true only when DAL bool option `788` is enabled and base
  pin signal type is `>= 11`.
- The DAL option table entry for id `788` is named
  `DalSupportExternalPanelDrr`, has bool type marker `0`, and default high
  dword `1`. It is not version-gated by
  `IsDalOptionSupportedByDriverVersion()`.
- Therefore Windows should normally write `0x310 = 02 04 01` for both DCE11
  and DCE12 DP/eDP-style Apple internal 5K paths. This is not currently a
  strong Pro1-only suspect.

The companion `0x300` word is also decoded enough to classify it:

- Adapter-service vfunc `+0x410` at `0x141467160` forwards to
  `CBasePinGetCapabilityProperty19()` at `0x14152D8F0`.
- That fetches CBasePin capability-name property id `19`; it is a pin/capability
  announcement value, not a source ordinal.

Final stream-enable branch:

- `StreamEnableMaybeDpcd170ThenSourceEnable()` at `0x1414E75A0` checks
  `DpStreamHasConverterCapabilityData()` at `0x1414E8000`.
- If stream byte `+0x36C` is clear, the no-converter path calls
  `LinkServiceSyncMsaTimingIgnoreOnDpcd107()` at `0x141446450`.
- If stream byte `+0x36C` is set, the converter path computes/writes DPCD
  `0x170` before lower source enable.
- For the known Apple internal-panel captures with no converter capability, the
  expected Windows path is the no-converter `0x107` path.

DPCD `0x107` is now stronger than before:

- `DpStreamSetMsaTimingIgnoreOnDpcd107()` and
  `LinkServiceSyncMsaTimingIgnoreOnDpcd107()` set bit 7 when
  `stream_arg+0x5C` or `stream_arg+0x60` is nonzero.
- `ApplyPathModeTimingToViewContextStream()` copies `path_record[13]` to
  `stream+0x5C`.
- `sub_141424DB0()` fills `path_record[13]` with
  `PackDisplayObjectIdCoreFields(display object id)`.
- For Pro1,1 objects `0x3114` and `0x3113`, the packed value is nonzero, so
  Windows should set DPCD `0x107` bit 7 on real streams. This likely applies to
  iMac19,1 too, so it is required behavior but not obviously Pro-only.

The immediate stream-enable branch is the more interesting split:

- `UpdateMode11ImmediateStreamEnableFlag()` at `0x141442720` sets the immediate
  flag only when display-object mode is `11`.
- Raw mode `12` explicitly clears the immediate flag and falls through to the
  generic cached/finalizer path.
- `ProgramDpcd316_StreamSourceFlag()` at `0x1414DD030` is only called inside
  `StreamEnableWrite4F1AndPollSinkStatus()` at `0x1414E7810`, i.e. inside the
  immediate worker. Mode `12` by itself does not imply DPCD `0x316`.
- The display-object mode vfunc `+0x160` is
  `DisplayObject_GetCommittedSignalTypeForModeGate()` at `0x141549190`. It
  returns committed signal type, coercing `12 -> 11` only when display-object
  byte `+0x1E9` is set.
- The byte `+0x1E9` setter is
  `DisplayObject_SetSignal12ReportsAs11Flag()` at `0x141547CB0`, vtable slot
  `+0x318`. Earlier SetMode RE maps this to the MST-to-SST optimization path:
  if the optimization anchor/path bit selects coercion, DCE12 signal `12` can
  report as mode `11` and enter the immediate worker.

Implication:

- iMac19,1/DCE11 naturally maps to committed mode `11`, so it can enter the
  immediate worker that writes `0x316` and the old `0x4F1` path.
- Pro1,1/DCE12/Vega naturally maps to committed mode `12`; it uses the generic
  no-converter finalizer unless SetMode sets the explicit `12 -> 11` coercion
  byte.
- This mode split is a real Windows-driver divergence and explains why simply
  replaying the iMac19,1-style immediate/latch experiment on Pro1,1 can miss the
  actual Windows path.

### Phase 6B continuation: SetMode tile anchor and timing bit7

The SetMode MST-to-SST/tile split branch is now substantially clearer.

Resolved names/comments saved in IDA:

- `SetModePathList_CopyCtor_CloneEntries()` at `0x14144B960`
- `SetModePathList_AddUniqueEntryClone_PreserveAuxBuffers()` at `0x14144B6D0`
- `SetMode_FindTileGroupAnchorDisplay()` at `0x141507AB0`
- `SetMode_BuildTimingRecordForDisplay()` at `0x141507F70`
- `DSDispatch_BuildStreamArgForDisplayFromCurrentPathEntry()` at `0x141512870`
- `DSDispatch_RetrySetModeWithMatchingModeTimingCandidate()` at `0x141512EB0`
- `DSDispatch_PostApplySetModeNotifyBuiltPaths()` at `0x14150E020`

Path-list mechanics:

- A SetMode path-list entry is `0x68` bytes.
- Entry `+0x18` is the timing/path metadata pointer.
- The timing metadata flag consumed by the tile split and MST-to-SST coercion
  branches is dword `+0x1C`, bit 7.
- `SetModePathList_CopyCtor_CloneEntries()` clones each entry wholesale,
  including the entry `+0x18` timing metadata pointer.
- `SetModePathList_AddUniqueEntryClone_PreserveAuxBuffers()` also clones the
  incoming entry and preserves destination-owned auxiliary buffers. This means
  the important bit7 is not invented by the later DCE source-mode builder.

Multi-entry anchor behavior:

- `SetMode_FindMstToSstOptimizationAnchorDisplay()` uses the single-display
  capability predicate for a one-entry path list.
- For multiple entries it delegates through the embedded interface at
  `DSDispatch+0x38`, vtable slot `+0x40`.
- The vtable tail resolves that slot to
  `SetMode_FindTileGroupAnchorDisplay()` at `0x141507AB0`.
- `SetMode_FindTileGroupAnchorDisplay()` requires:
  - more than one display index,
  - every candidate display to be tiled,
  - a common tile-group id from descriptor/capability vfunc `+0x2C8`,
  - full coverage of the tile grid (`display_count == tile_width * tile_height`),
  - and then it prefers the display whose descriptor/capability vfunc `+0x238`
    is true as the anchor.

Implication for Pro1,1:

- If Windows represents the Apple 5K pair as two SetMode path entries, the
  anchor selection is deterministic for a valid two-tile group; it is not just
  "first active path wins".
- If Windows represents it as one logical dual-tile entry, the single-entry
  predicate still uses the same display capability/tile-info service.
- Therefore the remaining mode-branch uncertainty is no longer the anchor
  finder. It is whether the incoming/timing-candidate path metadata bit7 is set.

Bit7 meaning:

- `BuildSourceModeForTileSplitPath()` at `0x14140BA40` reads
  `path_entry+0x18->+0x1C bit7`.
- If that bit is set and the display pair validates as a two-tile group, Windows
  doubles the source width, then halves/programs the timing across both tile
  display objects.
- `SetMode_ApplyMstToSstOptimizationSignalCoercion()` uses the same anchor
  path bit:
  - bit7 set: clear the display-object `12 -> 11` coercion byte at `+0x1E9`;
    DCE12 remains mode `12` and stays on the generic finalizer.
  - bit7 clear: set `+0x1E9`, report signal `12` as mode `11`, and apply
    MST-to-SST skip/force-disable flags.

Conclusion:

- For a normal two-tile Apple 5K source-mode split, Windows' expected DCE12 path
  is source `5/4`, raw mode `12`, no mode-11 immediate worker, generic
  no-converter finalizer, and DPCD `0x107[7]`.
- The iMac19,1-style `0x316`/`0x4F1` immediate-latch path should not be assumed
  for Pro1,1 unless the SetMode path metadata bit7 is clear or another RE path
  proves explicit `12 -> 11` coercion.
- The best next static RE target is now downstream of this branch: grouped
  timing/inter-path sync and final source programming order for the two DCE12
  sources, not another source-DPCD `04/05 1d 03` attempt.

### Phase 6C continuation: grouped timing sync

The grouped timing/inter-path sync path is now decoded enough to make it a real
Linux comparison target.

Resolved names/comments saved in IDA:

- `DSDispatch_PrepareSetModeInterPathTimingSync()` at `0x141506BD0`
- `SetMode_TimingRecordsMatchForInterPathSync()` at `0x141507350`
- `SyncManager_ApplySetModeSynchronizationState()` at `0x1414FCEF0`
- `SyncManager_ApplyInterPathSynchronization()` at `0x1414FBB20`
- `SyncManager_FindInterPathPendingTimingServer()` at `0x1414FB7E0`
- `SyncManager_GetDisplayIndexFromStreamArg()` at `0x1414F8130`
- `SyncManager_IsStreamArgPendingSyncCancel()` at `0x1414F7A80`
- `SyncManager_CompareSignalTypeForInterPathSync()` at `0x1414F7B60`
- `SyncManager_TimingRecordsMatchForHWSync()` at `0x1414F7E90`

Ordering:

- `DSDispatch_BuildApplySetModePaths()` marks one built path-state entry as the
  anchor/timing server with bit `0x8000`.
- `DSDispatch_PrepareSetModeInterPathTimingSync()` runs over the built path list
  at `DSDispatch+0x1EF8`.
- It compares timing records between paths using
  `SetMode_TimingRecordsMatchForInterPathSync()`.
- If compatible, it builds SyncManager request type `1` for inter-path timing
  sync.
- Request role `1` is timing server for the path-state `0x8000` anchor; role
  `2` is timing client. Non-anchor paths store local timing-server target info
  with the anchor display index.
- Accepted setup marks path-state bit `0x20`.
- `BuildStreamArgForDisplayIndex_SetObject178()` maps path-state bit `0x20`
  into stream-arg flags.
- `DSDispatch_ApplySetModeTransition()` calls
  `SyncManager_ApplySetModeSynchronizationState()` before blank/disable and
  before link enable/unblank.
- `SyncManager_ApplyInterPathSynchronization()` converts accepted server/client
  pairs into explicit HWSync requests:
  - client stream arg: `stream_arg+0x168 = 1`, `stream_arg+0x170 = server id`
  - server stream arg: `stream_arg+0x168 = 1` when at least one client is paired
- The log string at `0x1414FBDB1` confirms this path as
  `HWSyncRequest_Set_InterPath`.

Implication:

- Windows is not merely enabling two independent DCE12 streams. For compatible
  two-path/tile cases it can install an inter-path HWSync relationship before
  link enable/unblank.
- This is now a stronger next Linux audit target than more `0x310` testing:
  check whether the Pro1,1 Linux route establishes an equivalent timing sync or
  grouped commit relationship between root source `5` and slave source `4`.
- If Linux has source `5/4` and the generic DCE12 finalizer aligned but still
  fails, missing inter-path/HWSync setup becomes a plausible remaining cause.

### Phase 6D continuation: enable/unblank order

Enable/unblank order was checked after the SyncManager handoff.

- `DSDispatch_EnableLinksAndUnblankSetModePaths()` at `0x1415022D0` walks the
  built SetMode path list in ordinal order.
- For paths that pass the enable predicate, it calls child path-interface
  methods that line up with:
  - vfunc `+0x80`: `EnableLink`
  - vfunc `+0xB8`: `ChangeMode`
  - vfunc `+0xA8`: `UnblankStream`
- If path-state bit `0x100000` is set by the MST-to-SST optimization branch,
  the function logs `SkipEnable` and does not call the enable/change/unblank
  methods for that path.
- `DSDispatch_BlankDisableSetModePaths()` at `0x141502C40` also walks built
  paths by list order, while per-display child path interfaces are blanked or
  disabled in reverse child index order.

Implication:

- No new Pro1-only alternate enable order was found here.
- The more important Pro1 decisions happen before this phase: source `5/4`,
  bit7 tile-split versus MST-to-SST coercion, generic mode-12 finalizer, and
  inter-path HWSync setup.
- For the normal bit7-set tile-split case, both built paths should reach the
  enable/unblank loop; for bit7-clear MST-to-SST optimization, non-anchor paths
  can be skipped/force-disabled before this loop.

### Phase 6E continuation: HWSync source-signal helper chain

The downstream consumer of `stream_arg+0x168 == 1` is now more concrete. It
does not only mark bookkeeping state; it reaches the source-signal helper stored
on the display object and can change the provider-side signal type for the
inter-path group.

Resolved chain:

- `TopologyDetectBindPathSourceSignalAndSourceIndex()` at `0x141431F30`
  computes a class-9 source-signal object id for the path, looks that object up
  in display-map bucket `9`, and calls
  `DisplayMapAssignSourceSignalControlHelper()` at `0x1414AB3C0`.
- `DisplayMapAssignSourceSignalControlHelper()` calls display vfunc `+0x4A8`,
  now named `DisplayObject_SetSourceSignalControlHelper()`, and writes the
  matched provider source-signal object into the display object's helper slot.
- The path object stored in the display is the `DisplayTransportPath` secondary
  interface returned by `CreateDisplayTransportInterface()`.
- `DisplayTransportPath_GetSourceSignalObjectIdViaLower()` at `0x1415C62C0`
  forwards wrapper `+0x1C0` to the lower DCE path `+0x1C0`.
- `DpPathLower_GetSourceSignalObjectIdViaSourceObject()` at `0x14169DAB0`
  forwards lower `+0x1C0` to DCE source-object `+0x1D0`.
- `Dce12SourceObject_MapPathKeyToSourceSignalObjectId()` at `0x1416B0DE0`
  maps source/path key `0..5` to class-9 ids `9..14`.
- `Dce12SourceObject_ReadCurrentSourceSelectIndex()` at `0x1416B0970` reads the
  DCE12 source-select register and maps bits `0x01/0x02/0x04/0x08/0x10/0x20`
  to source indices `0..5`.
- The DCE11 equivalents at `0x1416A4B20` and `0x1416A47A0` use the same logical
  source-key and bit mapping, with DCE11 register offsets.
- The class-9 provider source-signal helper vtable slots are now resolved:
  - `+0x58`: `ProviderSourceSignal_GetObjectIdForMapMatch()` at `0x1416291A0`
  - `+0xA8`: `ProviderSourceSignal_ForwardSlotA8ToEventHandleB0()` at
    `0x141629060`
  - `+0xB0`: `ProviderSourceSignal_ApplyHWSyncSignalTypeViaEventHandle()` at
    `0x141628FF0`
- `ProviderSourceSignal_SetEventHandle()` at `0x141628C10` stores the concrete
  provider stream-event handle at provider object `+0x70`.
- Type12/DCE12 provider helpers allocate
  `Dce12SourceSignalEventHandle_Construct()` at `0x1416284B0`; its vtable
  `off_141FB9840` has slot `+0xB8` =
  `Dce12SourceSignalEventHandle_ApplySignalTypeToRegs()` at `0x141626820`.
- Type11/DCE11 provider helpers allocate
  `Dce11SourceSignalEventHandle_Construct()` at `0x14162C4C0`; its vtable
  `off_141FB9AC0` has slot `+0xB8` =
  `Dce11SourceSignalEventHandle_ApplySignalTypeToRegs()` at `0x14162B030`.
- DCE12 event `+0xB8` writes hardware-facing source-control registers using
  `dword_141F8CFA8[source]`: for `a3 == 0`, it updates
  `unk_141F8CD78 + offset + 6475` bit0; for `a3 == 1`, it updates
  `unk_141F8CD78 + offset + 6434` bit0 and writes
  `unk_141F8CD78 + offset + 6465` with mask `0xEFFFFFFF`.
- DCE11 event `+0xB8` is analogous, using `dword_141F8CF28[source]` and
  register offsets `19149`, `19108`, and `19139`.

HWSync connection:

- `HWSync_AdjustDpMstEdpSourceSignalForInterPath()` at `0x14156F180` scans
  state-1 entries (`stream_arg+0x168 == 1`), groups them by timing-server/source
  id, and uses the display object's source-signal-control helper.
- For mixed peer signal groups it calls provider source-signal `+0xB0` with the
  current source-select index in `EDX` and peer class-9 helper id `9..14` in
  `R8D`.
- For groups containing only DP/MST/eDP peers it calls provider source-signal
  `+0xB0` with current source-select index in `EDX` and sentinel `8` in `R8D`.
- Provider source-signal `+0xB0` forwards the adjustment into the provider
  stream-event handle vfunc `+0xB8`.
- Small prototype caveat: the provider thunk also forwards `R9D` into the event
  handle, but the inspected HWSync call sites do not explicitly set `R9D`.
  Therefore the statically certain part is the helper binding and the DCE
  source-control register writer, while the exact meaning of that extra payload
  remains unresolved.

Implication:

- Pro1,1 source `5/4` is not only a final DCE12 register-block difference.
  Through this chain it also selects class-9 source-signal helper ids `14/13`.
- The working iMac19,1 source `2/3` path selects class-9 helper ids `11/12`.
- Therefore a Linux Pro1,1 path that only ports the iMac19,1 source `2/3`
  helper assumptions can be wrong in two places: source-control register block
  and HWSync/provider source-signal adjustment.
- The final provider event-handle target is now devirtualized through DCE12 and
  DCE11 register writes. The next useful comparison is whether the Linux Pro1,1
  path has an equivalent source `5/4` helper relationship and HWSync-driven
  source-control adjustment.

### Phase 6F continuation: peer-source update source-manager closeout

The older target "source-manager object at stream/link context `+0x98`" is now
resolved enough to stop treating it as the main source-role payload consumer.

Resolved chain:

- `ProgramPeerSourceUpdateBeforeStreamEnable()` at `0x141442470` checks a
  service flags byte and skips the peer-source refresh when bit `0x08` is set.
- Otherwise it walks the stream VC table at stream object `+0x3F0`.
- Each VC entry must have both sink-present and payload-allocated state before
  it participates.
- The only argument passed to the `stream+0x98` object vfuncs is
  `GetStreamVcEntryDisplayRecord(entry)[0]`, i.e. a display index.
- The `stream+0x98` object vtable resolves to topology notification callbacks:
  - `+0x40`: `TopologyNotifyPreStreamDisableForDisplay()` at `0x14142F180`
  - `+0x48`: `TopologyNotifyPostStreamDisableForDisplay()` at `0x14142F070`
  - `+0x50`: `TopologyMaybeIssuePplibCmd38ForDisplay()` at `0x14142F370`
  - `+0x58`: `TopologyNotifyPostStreamEnableForDisplay()` at `0x14142EE40`
- The vtable entries are contiguous at `0x141F97938..0x141F97950`.

Implication:

- This peer-source pre-enable hook brackets allocated VC entries with topology
  notifications and a display-power/PPLIB refresh opportunity.
- It does not consume the DCE12 source mask, preferred source index, class-9
  helper id, or final stream-source role payload.
- Therefore it is lower priority than the now-proven source `5/4` class-9
  helper/HWSync path for explaining why Pro1,1 differs from iMac19,1.
