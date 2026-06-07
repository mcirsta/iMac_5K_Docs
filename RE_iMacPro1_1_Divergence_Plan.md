# iMacPro1,1 Divergence RE Plan

Date: 2026-06-03

IDA database: `B416406/amdkmdag.sys.i64`

## Goal

Find the Windows-driver branch where iMacPro1,1 / Vega / Japura diverges from
the working iMac19,1 / Polaris path.

Do not continue with broad DPCD trials. The Apple panel-capability layer is
already shared:

- Pro1,1 `APP/AE1D` maps to source id `0x38`, tag `0x3A`.
- Pro1,1 `APP/AE1E` maps to source id `0x39`, tag `0x3B`.
- `0x4F1=1` root and slave stream-enable latches have already run and read back
  on Pro1,1 without fixing the stretched-left-tile failure.
- Both `05 1d 03` and `04 1d 03` source-DPCD experiments appear not to fix the
  Pro1,1 failure, and the current IDA classifier evidence makes `04 1d 03` look
  Windows-shaped for Pro1,1 anyway.

The answer should therefore be below EDID/table matching and above blind sink
latches: resource-pool selection, link object construction, stream-mode branch,
ASSR/training policy, stream-source programming, MSA-ignore, or a stream-cap
refresh path.

## Principle

For every candidate branch, produce one of these outcomes:

- **Same as 19,1:** document and stop chasing it.
- **Different for Pro1,1:** identify the exact Windows condition, payload/order,
  and the matching Linux log or patch point.
- **Static RE cannot decide:** define one narrow runtime marker that resolves the
  branch, not a speculative behavior change.

## Phase 1: Build The ASIC/Resource-Pool Diff

Primary anchors:

- `ClassifyDisplayCoreVersionFromAsicId` at `0x141BAB280`
- `SelectResourcePoolForDisplayCoreOrdinal` at `0x141BAC910`
- `CreateDce112ResourcePool_Alloc` at `0x141C0F7D0`
- `Dce112ResourceConstruct` at `0x141C0D120`
- `CreateDce12VegaResourcePool_Alloc` at `0x141C04EA0`
- `Dce12VegaResourceConstruct` at `0x141C031A0`

Known fork:

- iMac19,1 / Polaris is expected to land in ordinal `5/6`, which selects
  `Dce112ResourceConstruct`.
- Pro1,1 / GPU `0x6860` is locally mapped by the driver data to family `0x8D`;
  family `0x8D` returns ordinal `7/8`, which selects
  `Dce12VegaResourceConstruct`.

Tasks:

1. Confirm the iMac19,1 baseline device/family/ordinal from Windows tables or
   Linux PCI id notes. Do not assume; write the device id and ordinal down.
2. Diff `Dce112ResourceConstruct` vs `Dce12VegaResourceConstruct`.
3. Build a field map for the 0x4E0-byte resource pool object, especially fields
   consumed later by link construction and stream enable:
   - resource-pool capability fields used by `CreateDisplayCoreFromInitData`
   - link encoder factory slots
   - panel-control factory slots
   - AUX/DDC/GPIO service hooks
   - HW sequence service construction
   - timing generator / stream encoder / link encoder service tables
4. Rename/comment every differing service constructor that can affect DP/eDP
   link or stream finalization.

Success criterion:

- A compact table: resource-pool field/slot, 19,1 constructor or vtable target,
  Pro1,1 constructor or vtable target, and whether it can affect 5K stream
  composition.

### Phase 1 Progress: Ordinal Path Is Now Local-RE Certain

The Pro1,1 resource-pool path is no longer just "likely" from the DCE label.
The local driver evidence is:

1. `PopulateDcInitDataFromWindowsDM()` copies adapter info into `dc_init_data`.
2. `ConstructDcContextFromInitData()` assigns:
   - `dc_context+0x50 = ClassifyDisplayCoreVersionFromAsicId(dc_init_data)`
3. A local packed device-family list contains sentinel `0xFFFFFFFE`, family
   marker `0x8D`, then includes device id `0x6860`.
4. `ClassifyDisplayCoreVersionFromAsicId()` maps family `0x8D` to ordinal `7`
   or `8`.
5. `SelectResourcePoolForDisplayCoreOrdinal()` maps ordinal `7/8` to
   `CreateDce12VegaResourcePool_Alloc()` / `Dce12VegaResourceConstruct()`.

Certainty boundary:

- The handoff's Pro1,1 GPU id is `1002:6860`.
- The local Linux tree maps `0x6860` to `CHIP_VEGA10`.
- This Windows driver contains a packed local device-family list where the
  `0x8D` group includes `0x6860`.
- The classifier result is certain once `adapter_info+0` is family `0x8D`:
  the only results are ordinal `7` or `8`, and both select
  `CreateDce12VegaResourcePool_Alloc()`.

Static caveat: IDA still does not expose a direct xref from the packed
`0x6860` list to the adapter-info population path, so a runtime marker can
confirm the adapter family/revision if needed. That marker resolves only the
ordinal `7` vs `8` subpath; it is no longer a broad "which resource pool?"
question. There is no alternate DisplayCore construction path in this IDB.

### Phase 1 Progress: First Constructor Diff

The DCE112/iMac19,1 baseline and DCE12/Vega/Pro1,1 constructor are structurally
similar but install different resources:

| Resource | DCE112 / iMac19,1 baseline | DCE12/Vega / Pro1,1 path | Why it matters |
|---|---|---|---|
| resource funcs at `pool+0x4C8` | `off_144B10880` | `off_144B10C00` | later resource callbacks can differ |
| resource caps at `pool+0x4D0` | `sub_141C0D990(...)` selects 5/6-pipe DCE112 caps | `DCE12_VEGA_RESOURCE_CAPS_TABLE` dwords begin `[6,0,0,7,6,6,0,6,...]` | Pro path exposes different counts/caps |
| consumed factory table | `DCE112_POLARIS_FACTORY_TABLE` | `DCE12_VEGA_FACTORY_ORD7_NO_INVALID_PIPE` or `DCE12_VEGA_FACTORY_ORD8_INVALID_PIPE` | exact Pro subpath depends on revision |
| ordinary stream encoder factory | `Dce112CreateStreamEncoderWithPolarisRegs` | `Dce12CreateStreamEncoderWithVegaRegs` | same concept, different register tables |
| stream encoder register tables | `unk_144B4CC70`, `unk_144B07720`, `unk_144B259A0` | `unk_144B4D230`, `unk_144B07840`, `unk_144B25FD0` | can change source/timing programming |
| HPO consumed slots | null; caps zero | null; caps `[12..14]` zero | HPO is not instantiated for Pro1,1 ordinal `7/8` in this constructor |
| adjacent HPO helpers | not in baseline table | present near `0x144B00A40`, xref from another constructor | adjacent data, not active in this Pro1,1 path |
| final resource service slot | `Dce112CreateFinalResourceService` | `Dce12CreateFinalResourceService_Ord7NoInvalidPipe` or `Dce12CreateFinalResourceService_Ord8InvalidPipe` | can affect stream-source/timing service behavior |
| invalid pipe handling | none visible | revision-gated invalid-pipe mask and skip loop | can alter pipe/TG mapping on Vega |
| per-pipe defaults | `unk_144AFFB90` | `unk_144AFFD10` | can alter stream/pipe defaults before final enable |
| DCE112 post hook | installs `Dce112EnableDisplayPowerGating` | no exact DCE12 call in this constructor | observed diff, but it is power-gating, not the main stream-role suspect |

Interpretation:

This is the first proven Windows path where Pro1,1 can behave differently while
the Apple 5K panel IDs remain shared. The next RE should follow the DCE12/Vega
ordinary stream encoder, final resource service, pipe/TG mapping, and
stream-source payload, not HPO DP and not more generic `0x4F1` or source-DPCD
`0x310` byte experiments.

## Phase 2: Map Pro1,1 Link Objects, Not Just Panel IDs

Primary anchors:

- `CreateLinksFromBiosObjectTable` at `0x141B7A8F0`
- `ConstructLegacyConnectorDisplayEntry` at `0x141B6D1F0`
- `BringUpDisplayBySignalType` at `0x141B6A720`
- `BringUpDpOrEdpWithLinkTraining` at `0x141B6CA70`

Known shared shape:

- connector low byte `0x13` -> signal `32`, secondary DP-style route
- connector low byte `0x14` -> signal `128`, root/eDP-style route

Tasks:

1. Confirm the Pro1,1 physical connector order, object ids, DDC hardware
   instances, HPD sources, and transmitters in the Windows model.
2. Compare against the iMac19,1 known route:
   - primary/root `0x3114`
   - secondary `0x3113`
   - DDC/AUX instance
   - UNIPHY transmitter
   - HPD behavior
3. Check whether the Vega resource pool changes any callback used only for
   signal `128` or signal `32`, especially the pre-training HPD/AUX callbacks.

Success criterion:

- A Pro1,1 route table equivalent to the iMac19,1 route table. If the Pro1,1
  secondary is not the same object/DDC/transmitter shape, that becomes a first
  patch/log axis.

### Phase 2 Progress: Link Objects Same, Lower AUX Namespace Differs

The logical connector-object shape still matches the older iMac route:

- Pro1,1 handoff root link id `3:20:1` has low byte `0x14`, matching AMD object
  id `0x3114`.
- Pro1,1 handoff slave link id `3:19:1` has low byte `0x13`, matching AMD object
  id `0x3113`.
- `ConstructLegacyConnectorDisplayEntry()` maps low byte `0x14` to signal `128`
  and low byte `0x13` to signal `32`.
- The working Pro1,1 VBIOS confirms raw UNIPHY2 objects for the Apple 5K pair:
  root/eDP `0x3114 -> 0x2221 -> source 5`, slave/DP
  `0x3113 -> 0x2121 -> source 4`. iMac19,1's known `0x2120/0x2220` route maps
  to source `2/3`.

The lower AUX/DDC namespace is the new concrete divergence candidate:

| Item | iMac19,1 / DCE11 baseline | Pro1,1 / DCE12 candidate |
|---|---|---|
| AdapterService base-pin selector | CBasePin property `[2] == 274` -> AUX generation `11` | CBasePin property `[2] == 288` -> AUX generation `12` |
| AUX pin-map factory | `ConstructDce11AuxPinMapFactory()` | `ConstructDce12AuxPinMapFactory()` |
| selector resolver | `Dce11ResolveAuxSelectorToPinIndex()` | `Dce12ResolveAuxSelectorToPinIndex()` |
| simple selector ladder | `0x4871 -> pin 2`, `0x4875 -> pin 3` | `0x55A7 -> pin 0`, `0x55AB -> pin 1`, `0x55AF -> pin 2`, `0x55B3 -> pin 3`, ... |
| observed Pro1,1 handoff fit | not a fit: root/slave would look like pins `3/2` | fits naturally: slave `ddc_hw=0`, root `ddc_hw=1` |

What is certain from static IDA:

- `CBasePinGetAuxGenerationFromProperty2()` returns generation `11` for
  property `[2] == 274` and generation `12` for property `[2] == 288`.
- `CreateAuxPinMapFactoryByGeneration()` has only the generation `11` and `12`
  construction paths here.
- generation `12` deterministically enters the DCE12/Vega selector resolver.
- the DCE12 resolver's simple selector base is `0x34C0`, not the DCE11
  `0x4869..0x4899` register range.

What remains separate from the now-confirmed source ordinal:

- The Pro1,1 v1.4 object table gives raw display records
  `0x3114 I2C/AUX=01 04 91 00 HPD=02 04 02 00` and
  `0x3113 I2C/AUX=01 04 90 00 HPD=02 04 01 00`.
- Its external display connection table offset is `0`, unlike the iMac19,1
  v1.3 ROM, so the old descriptor decode that yielded selectors
  `0x4871/0x4875` does not directly apply.
- If AUX selector certainty becomes necessary, the next static target is the
  Windows v1.4 object-record parser that turns these raw records into DCE12 AUX
  selectors/pins. Do not use runtime logging for this unless static RE hits a
  hard boundary.

Runtime marker that resolves the uncertainty:

```text
APPLE5K: aux route link[%d] object=0x%04x base_prop2=%u aux_gen=%u selector=0x%x source_mask=0x%x group=%d pin=%d ddc_hw=%d hpd=%d tx=%d
```

Interpretation:

Do not blindly reuse the iMac19,1 constants `0x4871/0x4875` or
DDC3/DDC4/pins `2/3` for Pro1,1. The Windows binary has a DCE12/Vega AUX
selector namespace, and the Pro1,1 handoff's `ddc_hw=0/1` shape is consistent
with that DCE12 namespace. This does not by itself prove the final 5K symptom is
an AUX-pin bug, because early `0x310` and `0x4F1` writes can still succeed. It
does make the lower route a first-class Pro1,1-vs-19,1 difference to log before
more DPCD payload experiments.

## Phase 3: Prove Which Stream-Enable Branch Pro1,1 Should Use

Primary anchors:

- `BuildStreamArgForDisplayIndex_SetObject178` at `0x141508510`
- `EnableValidatedStreamMaybeAssert4F1` at `0x141446B40`
- `UpdateMode11ImmediateStreamEnableFlag` at `0x141442720`
- `ProgramCachedLinkStreamSourceThenEnableStream` at `0x1414431B0`
- `StreamEnableWrite4F1AndPollSinkStatus` at `0x1414E7810`

Known branch:

- display-object mode `11` sets the immediate flag and enters the late
  `0x4F1` worker.
- display-object mode `12` clears that flag.
- non-mode-11 generic/cached path still programs stream source and calls the
  real stream-enable vfunc.

Tasks:

1. Trace the display-object vfunc at `stream_arg+0x178`, offset `+0x160`, back
   to the display object class selected for Apple 5K internal tiles.
2. Determine whether the 19,1 and Pro1,1 display objects can return different
   mode values (`10`, `11`, `12`) for the same tiled panel shape.
3. Map where stream arg `+0x88` is populated, because `0x316` bit 0 depends on
   this value.
4. Decide whether Pro1,1 should use:
   - immediate mode-11 worker;
   - generic cached/finalizer worker;
   - fresh-training worker;
   - a converter-capability worker.

Success criterion:

- One branch statement: "For Pro1,1 Apple 5K secondary, Windows expects stream
  mode X and worker Y." If static RE cannot prove it, add one Linux marker:

```text
APPLE5K: stream branch link[%d] display_mode=%u committed_mode=%u coerce12to11=%d mode_gate=%u path_bit7=%d cached=%d immediate=%d
```

### Phase 3 Progress: Mode 11 Is SetMode-Owned, Not An EDID Constant

The mode-11 stream-enable gate is now mapped:

- `BuildStreamArgForDisplayIndex_SetObject178()` stores the selected display
  object at `stream_arg+0x178`.
- `UpdateMode11ImmediateStreamEnableFlag()` calls that display object's vfunc
  `+0x160`.
- for full display objects, vfunc `+0x160` is
  `DisplayObject_GetCommittedSignalTypeForModeGate()`.
- that method reads the committed signal type from display interface `+0x48`.
- if display interface byte `+0x1E9` is set and committed type is `12`, it
  reports `11`; otherwise it reports the committed type unchanged.
- mode-gate result `11` sets the link-service immediate byte and calls
  `StreamEnableWrite4F1AndPollSinkStatus()`.
- mode-gate result `12` clears the immediate byte and uses the generic/cached
  stream-source finalizer instead.

The producer of byte `+0x1E9` is the SetMode MST-to-SST optimization path:

- `DisplayObject_SetSignal12ReportsAs11Flag()` is display-object vfunc
  `+0x318`.
- `SetMode_ApplyMstToSstOptimizationSignalCoercion()` calls vfunc `+0x318`.
- it first requires `SetMode_AllDisplaysHaveAtMostTwoTiles()`.
- it then finds an MST-to-SST optimization anchor display.
- anchor path timing/payload metadata bit `7` at `path_entry+0x18 -> +0x1C`
  selects the direction:
  - bit `7` clear: call vfunc `+0x318(1)`, setting byte `+0x1E9`;
  - bit `7` set: call vfunc `+0x318(0)`, clearing byte `+0x1E9`.

Important adjacent behavior:

- `BuildStreamsForDisplayObjectBySignalType()` has a special path for signal
  type `0x0B` and class-3 low byte `0x13` (`0x3113`): it builds extra stream
  interface variants before the main kind-1 stream.
- signal type `0x0C` does not build a stream in that switch.
- `ProgramDpcd316_StreamSourceFlag()` sets DPCD `0x316` bit `0` for stream
  payload mode values `10`, `11`, or `12`, but this helper is only called inside
  `StreamEnableWrite4F1AndPollSinkStatus()`.
- Therefore raw display-object mode `12` does not reach `0x316` in the generic
  cached finalizer unless SetMode first coerces `12 -> 11` via byte `+0x1E9`
  and selects the immediate worker.

Branch statement:

For Pro1,1 Apple 5K, Windows expects the immediate mode-11 worker only if the
display object either commits signal type `11`, or commits signal type `12` with
the SetMode coercion byte `+0x1E9` set. If it commits plain type `12` and the
coercion byte is clear, the generic/cached finalizer is Windows-shaped and the
absence of this wrapper's late `0x4F1` write is not automatically a mismatch.

The remaining static uncertainty is whether the Pro1,1 two-tile SetMode state
selects the same MST-to-SST optimization anchor/path-bit outcome as the working
iMac19,1 path. That should be logged with the stream branch marker above.

## Phase 4: ASSR And Training Policy

Primary anchors:

- `ClassifyDpPanelModeForAssrPolicy` at `0x1414E1460`
- `ProgramAssrThenTrainLinkSettings` at `0x1414E1570`
- `ProgramDpcd10A_AssrBitForPanelMode` at `0x1414E1370`
- `TryEnableLinkWithHbr2Fallback` at `0x1414E5CB0`

Known branch:

- object low byte `0x13` can return ASSR policy `1` through display-object
  vfunc `+0x1F0`.
- policy `1` or `2` sets DPCD `0x10A` bit 0 before training.
- cached-link match can skip fresh training.

Tasks:

1. Resolve the display-object vfunc `+0x1F0` implementation for the iMac tile
   display object.
2. Check whether the Vega/Pro1,1 object class changes this predicate relative to
   the iMac19,1 object class.
3. Confirm whether the Pro1,1 failure path is fresh-training or cached-link
   finalization.
4. If cached, check whether Windows still requires prior ASSR evidence before
   accepting cached finalization.

Success criterion:

- State whether Pro1,1 should definitely have `0x10A bit0 = 1` before final
  stream/source enable, and whether Linux currently proves that at the right
  lifecycle point.

## Phase 5: Stream-Source Role And MSA-Ignore

Primary anchors:

- `DpStreamSetMsaTimingIgnoreOnDpcd107` at `0x1414E22D0`
- `ProgramDpcd316_StreamSourceFlag` at `0x1414DD030`
- `ProgramStreamSourceWithLinkSettings`
- `HwSeqProgramStreamSourceWithLinkSettings` at `0x14155BFA0`
- `HwSeqProgramTimingGeneratorForStream` at `0x141555F70`
- `HwSeqProgramStreamTimingAndSourceCommon` at `0x14155B460`
- timing payload builder `sub_141562BA0`
- `DisplayMapResolveAvailableSourceIndex` at `0x1414AB040`
- `DisplayMapAssignStreamSourceIndex` at `0x1414AADD0`
- `BuildStreamSourceRolePayloadFromDisplayObject` at `0x141560E60`
- `CollectActivePathObjectsForSourceRolePayload` at `0x1415653E0`
- `ProgramPeerSourceUpdateBeforeStreamEnable` at `0x141442470`
- `HwSeqProgramSourceRolePayloadViaPath78And170` at `0x14155BE80`

Known branch:

- non-converter finalizer reads DPCD `0x107` and sets bit 7 when stream fields
  `+0x5C/+0x60` are nonzero.
- `0x316` bit 0 is set only inside the immediate worker; within that worker the
  payload mode values `10/11/12` set the bit.
- `ProgramStreamSourceWithLinkSettings` builds a 0x70-byte packet from
  `stream_arg+0x178`, a source/link index at link object `+0x64`, the optional
  cached link tuple, and the stream timing block at `stream_arg+0x2C`.
- `HwSeqProgramStreamSourceWithLinkSettings` performs two lower source writes:
  first the selected stream/source interface vfunc `+0x70(pipe_or_tg,
  mode_value, packet+0x10)`, then the active path object's vfunc
  `+0x70(role_payload)`.
- source index is resolved earlier by `DisplayMapResolveAvailableSourceIndex`.
  It uses pending signal type from display-object vfunc `+0x288`; pending type
  `12` uses resource priority cutoff `6`, and other pending types use cutoff
  `7`.
- `DisplayMapAssignStreamSourceIndex` writes that index through display-object
  vfunc `+0x480`; `BuildStreamSourceRolePayloadFromDisplayObject` reads it back
  through vfunc `+0x98` as `role_payload[0]`.
- the role payload contains source index, base object fields, committed/mode-gated
  signal, current object id, peer/fallback object id, display index, and flags.
- source-index allocation uses the raw stream path getter (`+0x450`), but final
  role programming uses active-gated current/peer getters (`+0x78/+0x80`).
  Active bit 0 is set during detect retain by display-object vfunc `+0x488`.
- HWSeq `+0x88` sends the common role payload to active path vfunc `+0x70`.
  HWSeq `+0x90` sends the same role payload to selected path vfuncs `+0x170`
  and `+0x78`.
- peer refresh runs before cached source programming and after the non-cached
  HWSeq `+0x90` source-packet path. It walks stream VC entries whose
  `entry+0x338` has sink-present bit 0 and payload-allocated bit 2 set.
  The display record first dword at `entry+0x350` is a display index passed to
  the stream full `+0x98` topology-notification interface, not to HWSeq and not
  to a source-role manager.

Why this matters:

- The visible Pro1,1 symptom is not "missing right tile" in userspace. Linux has
  both streams and correct rectangles, but the panel still composes the left tile
  incorrectly. That smells like stream-source role, timing-source, or MSA/source
  flag state rather than EDID layout.
- It is now plausible for Linux to match the raw source-index choice while still
  missing the Windows final role state: the allocator can see the raw path, but
  HWSeq role programming is gated by active stream state and peer/current path
  selection. The peer-refresh pass is still order-sensitive, but it has now been
  devirtualized as topology disable/enable notifications plus a PPLIB command-38
  power-policy refresh, not source-role programming.

Static RE tasks before asking a Pro1,1 user for anything else:

1. Devirtualize the inner DCE path object behind wrapper field `+0x58`.
   Resolve inner vfuncs `+0x68`, `+0x78`, and `+0x170`, because these consume
   the final role payload from HWSeq `+0x88/+0x90`.
2. Resolve the same inner object's source-allocation helpers:
   inner vfunc `+0x120` for preferred source index and inner vfunc `+0x128`
   for source mask. This is the static route to confirming whether Pro1,1/DCE12
   expects a different source index/mask model than iMac19,1/DCE11.
3. Closed: devirtualize the object used at stream/link context `+0x98`.
   It is the topology-manager full `+0x38` notification interface. The resolved
   vfuncs are `+0x40` PreStreamDisable, `+0x48` PostStreamDisable, `+0x50`
   optional PPLIB command-38 refresh, and `+0x58` PostStreamEnable.
4. Trace the construction/population of the stream VC table at context
   `+0x3F0`, especially `entry+0x338` and display record `entry+0x350`, to
   determine statically which root/slave display records feed the peer-source
   refresh.
5. Trace display-object path construction order into
   `DisplayObject_AddPathInterface()`, then forward to active-gated getters
   `+0x78/+0x80`. The static question is whether DCE12/Vega can select a
   different "first active" or peer path than the older iMac19,1 model.
6. Only after the above is exhausted, revisit whether a single narrow user-side
   observation is unavoidable.

Success criterion:

- Identify the lower write or state transition that consumes
  `src_index/current_obj/peer_obj/mode_gate`, or prove that the static Windows
  path requires only fields Linux already programs. The immediate objective is
  to avoid more user-run logs/probes unless static RE truly bottoms out.

### Phase 5 Progress: DisplayTransportPath Is The Real Active Path

The active path object stored in `DisplayObject` is now mapped more accurately.
The DisplayObject does not store the DCE12 controller/timing-generator interface
directly. It stores the display-map transport path constructed by
`CreateDisplayTransportInterface()`:

| Layer | Address | Role |
|---|---:|---|
| transport constructor | `0x1415C8440` | `DisplayTransportPath_Construct()` |
| transport vtable | `off_141FB2538` | DisplayObject active/raw path interface |
| lower binder | `0x1415C6600` | selects generation-specific lower path by transport-object low byte, path kind, and AUX generation |
| lower setter | `0x1415C7200` | stores selected lower object at transport full `+0x98` |

For DP/eDP-style transport-object low bytes `0x1E`, `0x20`, `0x21`, and
`0x25`, the binder's main path fork is:

| AUX generation | Lower constructor | Lower vtable | Meaning |
|---:|---:|---:|---|
| `11` | `0x14161E610` | `off_141FB7AB8` | DCE11/iMac19,1-style lower path |
| `12` | `0x141621AA0` | `off_141FB8E18` | DCE12/Vega/Pro1,1-style lower path |

The previously suspected DCE wrapper slice is not the DisplayObject path itself.
It is still part of the lower DCE controller world, but DisplayObject calls reach
it through `DisplayTransportPath` delegation.

Devirtualized source-allocation slots:

| DisplayTransportPath slot | Delegates to | DCE11 target | DCE12 target | Result |
|---:|---:|---:|---:|---|
| `+0x138` | lower source-mask `+0x138` | `0x14161E4A0` | `0x141621970` | different |
| `+0x140` | lower preferred-source `+0x140` | `0x141622FC0` | `0x141622FC0` | common |

The source-mask divergence is concrete:

- DCE11 returns `1 << preferred_source`, plus source bit `5` when the connected
  pin reports state `6`.
- DCE12 returns only `1 << preferred_source`.

This turns the Pro1,1 path difference into a source-index allocation question:
if Pro1,1's DCE12 route does not expose the DCE11 source-5 fallback, Linux logic
that was tuned from iMac19,1 can choose or allow a source model Windows would not
allow on Vega.

Devirtualized final source-role slots:

| DisplayTransportPath slot | Lower target | Meaning |
|---:|---:|---|
| `+0x70` | `0x141622A20` | no-op `return 0` |
| `+0x78` | `0x141622A00` | no-op `return 0` |
| `+0x170` | `0x14169DB60` | forwards `role_payload[0]` and `role_payload[3]` to source object `+0x130` |

The real source-mode writer is therefore the `+0x170` route, not the HWSeq
`+0x88` active-path `+0x70` call. The source object is created lazily by
`DpPathLower_CreateSourceObjectByAuxGen()` at `0x14169D810`:

| AUX generation | Source constructor | Source writer `+0x130` | DP/eDP register basis |
|---:|---:|---:|---|
| `11` | `0x1416AFD60` | `0x1416A5740` | `dword_141F8CB38[source] + 19139` |
| `12` | `0x1416B7F00` | `0x1416B1D90` | `Dce12SourceRegisterBaseOffset + Dce12SourceRegisterBlockOffsetBySource[source] + 6465` |

Updated next static target:

1. Resolve which class-2 transport-object low-byte/path-kind cases feed
   `DisplayTransportPath_BindLowerByObjectAndAuxGen()` on the paths that end at
   class-3 endpoints `0x3113` and `0x3114`.
2. Resolve the constructor RealState input that becomes the preferred source
   index on both tiles.
3. Decide whether the DCE11 source-5 fallback exists on the working iMac19,1
   route and is absent on Pro1,1 by construction.
4. Then continue to the stream VC-table peer refresh, using the now-correct
   source-index and source-mode writer model.

### Phase 5 Progress: RealState/source ordinal bridge is now concrete

The class-2 source ordinal bridge has been decoded and saved in IDA:

| Address | Name | Role |
|---:|---|---|
| `0x141622180` | `TransportObjectIdToRealStateSourceOrdinal()` | raw class-2 object id -> RealState/source ordinal |
| `0x141623B20` | `CBaseRenderer_ConstructDecodeTransportObject()` | stores decoded RealState at full object `+0x30` |
| `0x1416225F0` | `DpPathLower_SetPreferredSourceIndex()` | writes preferred source at lower full `+0x48` |
| `0x14169D600` | `DpPathLower_GetSourceObject()` | returns the generation-specific source object used by lower `+0x170` |

The decoded raw object map is:

| Raw class-2 object shape | Source ordinal |
|---|---:|
| `0x?11E` / type `0x1E`, enum `1` | `0` |
| `0x?21E` / type `0x1E`, enum `2` | `1` |
| `0x?120` / type `0x20`, enum `1` | `2` |
| `0x?220` / type `0x20`, enum `2` | `3` |
| `0x?121` / type `0x21`, enum `1` | `4` |
| `0x?221` / type `0x21`, enum `2` | `5` |
| `0x?125` / type `0x25`, enum `1` | `6` |
| `0x?122` / type `0x22`, enum `1` | `7` |
| `0x?123` / type `0x23`, enum `1` | `8` |
| `0x?223` / type `0x23`, enum `2` | `9` |

The constructor-side path is now proven:

- `ExpandDisplayPathAndConstructStreams()` at `0x1414B7AC0` recursively walks
  the BIOS object-service child list.
- When a child object has class nibble `2`, it calls
  `CreateDisplayTransportInterface()` with that raw object id.
- The alternate path builder at `0x1414B6380` does the same for class-2 entries.
- Therefore `DisplayTransportPath` RealState is derived from the raw class-2
  VBIOS display-path object, not from the final class-3 connector object and not
  from Linux's normalized encoder enum.

This resolves the iMac19,1 baseline:

| Endpoint | Known raw class-2 object | Windows source ordinal |
|---:|---:|---:|
| `0x3114` primary/root | `0x2120` | `2` |
| `0x3113` secondary | `0x2220` | `3` |

The working Pro1,1 VBIOS resolves the Pro1,1 source ordinal:

- ROM: `imacpro11-vbios.rom`.
- Part number: `113-D05001A1XT-018`.
- SHA256:
  `73AE9942052F17F6E1201409C3C55C98962A5050362D55E3D8E378CE936F16F8`.
- ATOM display object info table: revision `1.4`, offset `0x83c`.

| Endpoint | Pro1,1 raw class-2 object | Windows source ordinal |
|---:|---:|---:|
| `0x3114` primary/root eDP | `0x2221` | `5` |
| `0x3113` secondary/slave DP | `0x2121` | `4` |

This matches the handoff's transmitter/engine `5/4` shape and proves that the
Pro1,1 Apple 5K pair is routed through UNIPHY2/source `5/4`, not the
iMac19,1 UNIPHY1/source `2/3` route.

The downstream consequence is certain. `role_payload[0]` is forwarded by
`DpPathLower_ProgramSourceModeViaSourceObject()` at `0x14169DB60` to the source
object vfunc `+0x130`. On DCE12, `Dce12SourceObject_ProgramDpSourceIndexMode()`
at `0x1416B1D90` uses
`Dce12SourceRegisterBlockOffsetBySource[source] = {0, 0x100, 0x200, 0x300,
0x400, 0x500}` and writes the Vega DP/eDP source-control register at
`Dce12SourceRegisterBaseOffset + table[source] + 6465`.

Updated interpretation:

- The Windows driver and the Pro1,1 VBIOS now agree on root source `5` and
  slave source `4`.
- More `0x310 = 04/05 1d 03` experiments are lower value than matching this
  source-role/source-control model.
- A Linux path that still assumes the iMac19,1 source pair `2/3`, DDC3/DDC4, or
  UNIPHY1 semantics is using the wrong model for Pro1,1 final stream/source
  programming.

### Phase 5 Progress: Peer-refresh +0x98 is topology notification

The stream/link context `+0x98` target called by
`ProgramPeerSourceUpdateBeforeStreamEnable()` is now resolved.

Construction chain:

- `ConstructDisplayTopologyManager()` exposes topology-manager full `+0x38`
  using vtable `off_141F978F8`.
- `ConstructTopologyDisplayPathBuilder()` stores that interface at builder
  `+0x60`.
- `BuildStreamForDisplayEntryPath()` copies builder `+0x60` into stream init
  `+0x10`.
- `ConstructBaseStreamObjectFromInit()` copies init `+0x10` to stream full
  `+0x98`.

The four peer-refresh vfuncs are:

| Slot | Target | Meaning |
|---:|---:|---|
| `+0x40` | `0x14142F180` / `TopologyNotifyPreStreamDisableForDisplay()` | event id `59`, `Notify PreStreamDisable for Display[%d]` |
| `+0x48` | `0x14142F070` / `TopologyNotifyPostStreamDisableForDisplay()` | event id `63`, `Notify PostStreamDisable for Display[%d]` |
| `+0x50` | `0x14142F370` / `TopologyMaybeIssuePplibCmd38ForDisplay()` | optional display-power-policy callback |
| `+0x58` | `0x14142EE40` / `TopologyNotifyPostStreamEnableForDisplay()` | event id `60`, `Notify PostStreamEnable for Display[%d]` |

The optional power-policy callback goes through
`DisplayPowerPolicy_IssuePplibCmd38()` at `0x141535850` and
`PplibCmd38_DisplayPowerRefresh()` at `0x1415B9B20`, which sends PPLIB command
`38` with no payload.

Interpretation:

- This pass is a topology notification and display-power-policy refresh around
  active VC entries.
- It does not consume source masks, preferred source indices, or
  `role_payload[0]`.
- Therefore it is less likely than the DCE12 source `5/4` role/control path to
  explain a Pro1-only wrong-tile composition by itself, although Linux should
  still preserve the ordering if it has an equivalent notification/power hook.

### Phase 5 Progress: Path insertion order is not currently Pro-only

The DisplayObject path-order target is also narrowed:

- `DisplayObject_AddPathInterface()` stores paths into a two-slot stream array;
  insertion order defines stream index.
- `ConstructDisplayObjectAndStreams()` attaches the accumulated BIOS path array
  in reverse order.
- `AttachChildInterfacesToDisplayObjectPath()` captures the old path count
  before `AddPathInterface()` and uses that index for associated child
  AUX/source interfaces.
- `CollectActivePathObjectsForSourceRolePayload()` scans active streams from
  index `0` upward and stops at the first active stream.
- `DisplayObject_GetActivePeerStreamPathObject()` returns only `index + 1`; it
  does not wrap. A one-path endpoint has no active peer path and falls back in
  `role_payload[5]`.

The VBIOS comparison does not make this a Pro1-only fork:

| Machine | Root/eDP path | Slave/DP path |
|---|---|---|
| Pro1,1 | `0x3114 -> 0x2221 -> source 5` | `0x3113 -> 0x2121 -> source 4` |
| iMac19,1 | `0x3114 -> 0x2120 -> source 2` | `0x3113 -> 0x2220 -> source 3` |

Both are direct one-encoder-per-endpoint paths. So the current/peer path-order
question is less suspicious than the source ordinal/register-block difference:
Pro1,1 differs in DCE12 source `5/4`, not in an obvious multi-path peer ordering
shape.

## Phase 6: Converter/Stream-Capability Branch

Primary anchors:

- `RefreshDpStreamCapsAndConverterData` at `0x1414E7250`
- `WriteDpStreamCapsSourceDpcd310_0204Flag` at `0x1414DD180`
- `RefreshDpConverterCapabilityData`
- `ParseDpConverterCapabilityData` at `0x1414F5E70`
- `StreamEnableMaybeDpcd170ThenSourceEnable` at `0x1414E75A0`

Known branch:

- `WriteDpStreamCapsSourceDpcd310_0204Flag` writes DPCD `0x300` and a separate
  DPCD `0x310` payload shape: `02 04 <bit>`.
- This is not the `04/05 1d 03` pre-training source-table revision.
- `<bit>` is `DalSupportExternalPanelDrr` option id `788` AND base pin signal
  type `>= 11`. The option table default is true and it is not version-gated, so
  the expected Windows payload for DCE11/DCE12 DP/eDP-style Apple internal
  paths is `02 04 01`.
- The companion `0x300` word comes from CBasePin capability-name property id
  `19`; it is not a source ordinal.
- Converter capability switches finalization to a `0x170` path; otherwise the
  no-converter path uses `0x107` MSA-ignore.
- The no-converter `0x107` path sets bit 7 when `stream+0x5C/+0x60` is nonzero.
  `stream+0x5C` is copied from `path_record[13]`, which is the packed display
  object id, so Pro1,1 objects `0x3114/0x3113` should make Windows set
  `0x107[7]`.
- `0x316` is not part of the generic cached finalizer. It is only written inside
  `StreamEnableWrite4F1AndPollSinkStatus`, reached by the mode-11 immediate
  branch.
- `UpdateMode11ImmediateStreamEnableFlag` sets immediate mode only for committed
  display-object mode `11`; raw mode `12` clears the immediate flag. The
  display-object mode getter only coerces `12 -> 11` if byte `+0x1E9` is set.
  DCE12/Vega should therefore naturally use the generic finalizer, while DCE11
  naturally enters the mode-11 immediate path.

Tasks:

1. Compare Linux final enable against the Windows DCE12/Vega branch: generic
   cached/no-converter finalizer, source-role/control source `5/4`, and DPCD
   `0x107[7]`.
2. Check whether any Linux Pro1,1 route accidentally follows the iMac19,1
   mode-11 immediate/latch model (`0x316`/`0x4F1`) instead of the DCE12 mode-12
   generic finalizer.
3. Keep `0x170` converter handling as a low-probability branch unless static
   DPCD/converter-capability evidence says Pro1,1 has parsed converter data.
4. Check ASSR `0x10A` ordering after the DCE12 finalizer/source path is aligned.

Success criterion:

- Treat `0x310 = 02 04 01` as Windows-shaped baseline behavior, not the primary
  fix candidate.
- Decide whether Linux needs a DCE12/Vega-specific final enable path: source
  `5/4`, generic no-converter `0x107[7]`, and no iMac19,1-only immediate-latch
  assumptions.

## Phase 6B: SetMode Tile Anchor And Bit7

Status: mostly resolved.

New Windows evidence:

- `SetMode_FindMstToSstOptimizationAnchorDisplay()` delegates multi-entry path
  lists through the embedded `DSDispatch+0x38` interface.
- That vtable slot resolves to `SetMode_FindTileGroupAnchorDisplay()` at
  `0x141507AB0`.
- The multi-entry anchor finder validates a tiled group by common tile-group id,
  complete grid coverage, and preferred anchor flag from descriptor vfunc
  `+0x238`.
- The incoming path timing metadata bit is
  `path_entry+0x18->+0x1C bit7`.
- `BuildSourceModeForTileSplitPath()` consumes that bit to enter the dual-tile
  source split path.
- `SetMode_ApplyMstToSstOptimizationSignalCoercion()` consumes the same bit to
  decide whether DCE12 signal `12` is left as mode `12` or coerced to mode `11`.

Updated conclusion:

- For the normal Apple 5K dual-tile split, bit7 set is the Windows-shaped path:
  it clears the `12 -> 11` coercion byte and leaves Pro1,1/DCE12 on the generic
  mode-12 finalizer.
- Bit7 clear is the MST-to-SST optimization path: it sets the coercion byte and
  can push DCE12 into the mode-11 immediate path.
- The anchor finder itself is no longer a strong uncertainty. It should collapse
  a valid two-entry Pro1,1 tile group deterministically.

Next RE target after Phase 6B:

1. Trace grouped timing/inter-path sync after `DSDispatch_BuildApplySetModePaths`.
2. Confirm final source programming order for source `5/4` on the DCE12/Vega
   path.
3. Only revisit `0x316`/`0x4F1` if static RE finds a Pro1,1 path that clears
   bit7 or explicitly sets the `12 -> 11` coercion byte.

## Phase 6C: Grouped Timing / HWSync

Status: actionable Windows behavior identified.

New Windows evidence:

- `DSDispatch_PrepareSetModeInterPathTimingSync()` at `0x141506BD0` walks the
  built path list and prepares inter-path timing-sync requests.
- It uses path-state bit `0x8000` as the timing-server/anchor marker.
- It compares candidate path timing records with
  `SetMode_TimingRecordsMatchForInterPathSync()` at `0x141507350`.
- Accepted setup marks path-state bit `0x20`.
- `BuildStreamArgForDisplayIndex_SetObject178()` maps that path-state bit into
  the stream arg.
- `DSDispatch_ApplySetModeTransition()` applies SyncManager state before
  blank/disable and link enable/unblank.
- `SyncManager_ApplyInterPathSynchronization()` sets explicit HWSync request
  fields: `stream_arg+0x168 = 1` on paired clients and on the timing server.
- The log string `HWSyncRequest_Set_InterPath setup` confirms the purpose.

Updated next target:

1. Compare Linux Pro1,1 two-tile commit against Windows inter-path/HWSync
   behavior, especially whether root source `5` and slave source `4` are tied
   by an equivalent timing-server/client relationship.
2. If Linux currently only performs independent stream enables, consider this a
   patch candidate after the source `5/4` and mode-12 generic-finalizer checks.
3. Keep ASSR `0x10A` ordering as the following target after grouped timing sync.

## Phase 6D: Enable / Unblank Order

Status: checked, no new Pro1-only fork found.

New Windows evidence:

- `DSDispatch_EnableLinksAndUnblankSetModePaths()` at `0x1415022D0` walks the
  built SetMode path list in ordinal order.
- Its child path-interface calls line up with `EnableLink`, `ChangeMode`, and
  `UnblankStream`.
- Path-state bit `0x100000` is the real `SkipEnable` gate and comes from the
  MST-to-SST optimization branch.
- `DSDispatch_BlankDisableSetModePaths()` at `0x141502C40` walks built paths in
  list order, with per-display child path interfaces processed in reverse child
  index order for blank/disable.

Updated conclusion:

- There is no current evidence that Pro1,1 requires a special enable-order fork
  after SetMode path construction.
- Keep focus on the pre-enable decisions: DCE12 source `5/4`, mode-12 generic
  finalizer, and inter-path/HWSync setup.

## Phase 6E: HWSync Source-Signal Helper Chain

Status: Windows chain resolved through provider source-signal helper and into
DCE source-control register writes.

New Windows evidence:

- The DisplayObject stores a source-signal-control helper through
  `DisplayObject_SetSourceSignalControlHelper()` at `0x141549550`.
- `TopologyDetectBindPathSourceSignalAndSourceIndex()` at `0x141431F30` binds
  that helper by class-9 object id before assigning the display source index.
- The active stream path is the `DisplayTransportPath` secondary interface.
  Its `+0x1C0/+0x1C8` methods forward through the lower DCE path to the DCE
  source object.
- DCE12 source object `+0x1D0`
  (`Dce12SourceObject_MapPathKeyToSourceSignalObjectId()` at `0x1416B0DE0`)
  maps source/path keys `0..5` to class-9 helper ids `9..14`.
- DCE12 source object `+0x1D8`
  (`Dce12SourceObject_ReadCurrentSourceSelectIndex()` at `0x1416B0970`) maps
  source-select bits `0x01/0x02/0x04/0x08/0x10/0x20` to source indices `0..5`.
- Therefore Pro1,1 source `5/4` selects class-9 helper ids `14/13`; iMac19,1
  source `2/3` selects helper ids `11/12`.
- `HWSync_AdjustDpMstEdpSourceSignalForInterPath()` at `0x14156F180` consumes
  `stream_arg+0x168 == 1` and calls the provider source-signal helper vfunc
  `+0xB0`.
- The class-9 helper vfunc `+0xB0` is
  `ProviderSourceSignal_ApplyHWSyncSignalTypeViaEventHandle()` at
  `0x141628FF0`, which forwards the signal-type adjustment to the provider
  stream-event handle vfunc `+0xB8`.
- Type12/DCE12 helpers install a `Dce12SourceSignalEventHandle` at provider
  object `+0x70`; event-handle `+0xB8` is
  `Dce12SourceSignalEventHandle_ApplySignalTypeToRegs()` at `0x141626820`.
- The DCE12 `+0xB8` implementation writes source-control registers selected by
  `dword_141F8CFA8[source]`: `base+6475` for `a3 == 0`, or `base+6434` and
  `base+6465` for `a3 == 1`.
- Type11/DCE11 has the analogous event-handle writer at `0x14162B030`, using
  `dword_141F8CF28[source]` and offsets `19149`, `19108`, and `19139`.
- The HWSync call sites explicitly pass current source-select index in `EDX`
  and peer helper id `9..14` or sentinel `8` in `R8D`. The provider thunk also
  forwards `R9D`, but the inspected HWSync sites do not explicitly set it; treat
  the exact extra payload as the remaining small prototype uncertainty.

Updated conclusion:

- HWSync is no longer just a grouped timing marker. On the Windows path it can
  alter provider source-signal state for the paired streams and reach DCE
  source-control register writes.
- Matching only DPCD `0x310`, `0x107`, or the final DCE12 source-control write
  may still miss the Pro1,1 difference if Linux never creates the equivalent
  source `5/4` class-9 helper relationship and inter-path HWSync adjustment.
- The final provider stream-event handle implementation is no longer the main
  unknown. The next useful static target is Linux parity: source `5/4`, helper
  ids `14/13`, and an equivalent inter-path source-signal/HWSync adjustment.

## Phase 6F: Peer-Source Update Source-Manager Closeout

Status: `stream/link context +0x98` is resolved as a topology notification
interface, not the main DCE source-role payload consumer.

Windows evidence:

- `ProgramPeerSourceUpdateBeforeStreamEnable()` at `0x141442470` walks the
  stream VC table at stream object `+0x3F0`.
- It only processes entries where sink-present and payload-allocated state are
  both true.
- It passes `GetStreamVcEntryDisplayRecord(entry)[0]` to vfuncs on the object at
  stream object `+0x98`.
- The concrete targets are:
  - `+0x40`: `TopologyNotifyPreStreamDisableForDisplay()` at `0x14142F180`
  - `+0x48`: `TopologyNotifyPostStreamDisableForDisplay()` at `0x14142F070`
  - `+0x50`: `TopologyMaybeIssuePplibCmd38ForDisplay()` at `0x14142F370`
  - `+0x58`: `TopologyNotifyPostStreamEnableForDisplay()` at `0x14142EE40`

Conclusion:

- This branch brackets VC entries with topology notifications and a possible
  display-power/PPLIB command-38 refresh.
- It does not read or transform source `5/4`, class-9 helper ids `14/13`, or
  the DCE12 source-control register payload.
- Keep it documented as closed unless a later Linux audit shows a missing
  topology notification or PPLIB refresh specific to the stretched-tile symptom.

## Phase 7: Minimal Runtime Markers If Static Inputs Are Missing

Do not request runtime probes for the source ordinal anymore: the Pro1,1 VBIOS
already proves root/eDP source `5` and slave/DP source `4`. Keep these markers
only as a last resort for later branches that static RE cannot close:

```text
APPLE5K: resource ordinal linux_dce=%u windows_ordinal_guess=%u route=%s
APPLE5K: aux route link[%d] object=0x%04x base_prop2=%u aux_gen=%u selector=0x%x source_mask=0x%x group=%d pin=%d ddc_hw=%d hpd=%d tx=%d
APPLE5K: stream branch link[%d] display_mode=%u committed_mode=%u coerce12to11=%d mode_gate=%u path_bit7=%d cached=%d immediate=%d
APPLE5K: ASSR 0x10A link[%d] phase=%s old=%02x new=%02x status=%d policy=%u
APPLE5K: MSA 0x107 link[%d] phase=%s old=%02x new=%02x status=%d ignore_msa=%d
APPLE5K: sourceflag 0x316 link[%d] mode=%u old=%02x new=%02x status=%d
APPLE5K: streamcap 0x310_0204 link[%d] attempted=%d payload=%02x %02x %02x status=%d
APPLE5K: final source role link[%d] obj=0x%04x peer_obj=0x%04x pending_mode=%u committed_mode=%u mode_gate=%u src_index=%d display_index=%u flags=%08x pipe_or_tg=%d mode_value=%u
```

The log is not a substitute for RE. It is only to resolve later branches that
static analysis cannot prove.

## Current Concrete Next Step

Status: superseded by the 2026-06-07 `Pro/new_1` reassessment below.

The next useful work is no longer another broad resource-pool diff and not
another `0x310 = 04/05 1d 03` experiment. The raw Pro1,1 source model is now
known: root/eDP source `5`, slave/DP source `4`.

The next target is to map the confirmed Windows model back onto Linux final
stream/source programming and HWSync/provider source-signal state:

1. Compare the Linux final two-tile commit path against Windows
   `role_payload[0]` and `role_payload[3]`: root must program source `5`, slave
   must program source `4`, and `role_payload[5]` should match Windows'
   one-path fallback behavior unless a separate Linux grouping layer explicitly
   creates a second active path.
2. Check whether any Pro1,1 quirk or inherited iMac19,1 path still hardcodes or
   derives source `2/3`, UNIPHY1, DDC3/DDC4, or old DCE11 AUX pin assumptions.
3. With source `5/4` aligned, compare the branch choice:
   - DCE11/iMac19,1 committed mode `11` can enter the immediate worker and write
     `0x316`/`0x4F1`.
   - DCE12/Pro1,1 committed mode `12` should clear the immediate flag and use
     the generic no-converter finalizer unless the display object explicitly
     coerces `12 -> 11`.
   - The generic no-converter finalizer should set DPCD `0x107[7]` because the
     stream path record carries packed object id `0x3114/0x3113`.
4. Treat stream-capability `0x310 = 02 04 01` as decoded baseline behavior:
   the third byte is `DalSupportExternalPanelDrr && base_signal_type >= 11`.
   It is lower-priority than the DCE12 source/mode/finalizer path.
5. Compare Linux grouped timing/inter-path HWSync against Windows:
   root/slave should become a source `5/4` timing-server/client group when the
   two-tile timing records are compatible.
6. Check whether Linux has an equivalent of the Windows class-9 source-signal
   helper adjustment: Pro1,1 should use helper ids `14/13`, not iMac19,1
   helper ids `11/12`.
7. After HWSync/source-signal behavior is aligned, check ASSR `0x10A` ordering.

Local Linux-tree scan result:

- The current `linux-imac-5k` Apple 5K route helpers are still iMac19,1-shaped
  in the files checked: `amdgpu_dm.c`, `amdgpu_dm_helpers.c`,
  `link_dp_capability.c`, `link_dp_training.c`, `link_detection.c`,
  `link_dpms.c`, and `link_edp_panel_control.c`.
- Those helpers gate on secondary DDC `2` / `TRANSMITTER_UNIPHY_D` and primary
  DDC `3` / `TRANSMITTER_UNIPHY_C`, which matches iMac19,1 source `3/2`, not
  Pro1,1 source `4/5`.
- The Pro1,1 route variant should instead recognize root/eDP object `0x3114`
  with transmitter `TRANSMITTER_UNIPHY_F`/source `5`, and slave/DP object
  `0x3113` with transmitter `TRANSMITTER_UNIPHY_E`/source `4`. Its observed
  DDC raw shape is `1/0`, so do not reuse DDC `3/2` as a Pro1 gate.

## 2026-06-07 `Pro/new_1` Dmesg And Kernel Reassessment

Inputs:

- `Pro/new_1/dmesg_Pro.txt`
- `Pro/new_1/dmesg_19_1.txt`
- local kernel trees:
  - `linux-imac-5k` on `pro1-apple5k-logging` at `ee0918521`
  - `linux-imac-5k-pro1` on `codex/pro1-apple5k-logging` at `2f54c1281`

### Capture provenance

The captures do not come from the checked-out `2f54c1281` behavior:

- Pro reports `7.0.1-imac5k-gf7f5b1dd28ba`.
- iMac19,1 reports `7.0.1-1-imac-5k-gada7abe5ec64`.
- Neither reported source object exists in the local repository.
- Both logs contain the `APPLE5K-PROBE` source/MSA/blank markers from the
  `400f9441c..ee0918521` probe branch.
- Both logs omit the `force_sync` panel flag, `stream MSA-ignore`,
  `stream feature 0x107`, and `timing-sync candidate/result/apply` markers
  introduced by `2f54c1281`.

Therefore the logs are authoritative for the probe behavior, but they do not
test the force-sync/MSA-ignore patch at local HEAD.

### Runtime comparison

| Plan axis | iMacPro1,1 | iMac19,1 | Assessment |
|---|---|---|---|
| ASIC/resource generation | Vega10, DCE12, GPU `1002:6860` | Polaris, DCE11.2, GPU `1002:67df` | expected Windows resource-pool divergence confirmed |
| root route | link 0, DDC 1, encoder/engine 5 | link 0, DDC 3, encoder/engine 2 | Pro DCE12 route is correctly discovered |
| slave route | link 1, DDC 0, encoder/engine 4 | link 1, DDC 2, encoder/engine 3 | Pro DCE12 route is correctly discovered |
| FE source select | source 5 mask `0x20`; source 4 mask `0x10` | source 2 mask `0x04`; source 3 mask `0x08` | exact Windows/VBIOS source ordinals are already programmed |
| tiled EDID construction | root becomes tile `(0,0)`, slave tile `(1,0)`, common 2x1 group | same | no Pro-only tile-group failure |
| tile timing/MSA | both engines: active `2560x2880`, total `2720x2962`, start `112,79`, width `2560`, height `2880` | identical | no stretched-tile cause in per-stream MSA geometry |
| framebuffer split | left source X `0`, right source X `2560`; destination/clip X local `0` | identical | matches the Windows tile-split source-mode result |
| DP stream enable | engines 5 and 4 unblank with enable readback `1` | engines 2 and 3 do the same | no missing right-tile `DP_VID_STREAM_ENABLE` latch |
| source table `0x310` | forced `04 1d 03`, DCE auto-first byte would be `05` | forced `04 1d 03`, auto-first is `04` | not a fixing difference; already deprioritized |
| stream latch `0x4F1` | slave writes/readbacks `1` | same | successful latch is not sufficient to fix Pro |
| link training | succeeds without the long initial retry sequence | one initial `enabling link 1 failed: 15`, then recovers | training is not the Pro-only failure axis |

The low-level probe closes the following candidates:

1. Linux is not accidentally using iMac19,1 source `2/3` on Pro.
2. Linux is not generating a full-width or otherwise different MSA for one Pro
   tile.
3. Linux is not leaving the Pro right-tile stream encoder disabled.
4. Candidate C from the tile-split handoff is already represented by the
   right-plane source X `2560`.

### Timing-sync log correction

The existing stream trace labels this field:

```text
sync_enabled=0 master_link[0] event=0 delay=0
```

but the implementation prints `stream->triggered_crtc_reset.enabled` and its
event-source fields. It does not print
`pipe_ctx->stream_res.tg->timing_sync_info`.

Consequences:

- The new logs do not prove that DC timing synchronization was absent.
- The `7b4158862` revert rationale treated `sync_enabled=0` as evidence against
  timing grouping, but that specific field is not the Phase 6C HWSync state.
- Phase 6C remains unresolved at runtime until the real
  `timing_sync_info.group_id/group_size/master` state is logged.

### Phase 6E correction

The later static closeout in
`RE_iMacPro1_1_IDA_MCP_Findings.md` supersedes the earlier Phase 6E
interpretation:

- HWSync passes selector `8` for the all-DP/eDP pair, or `9..14` for the mixed
  helper path.
- The concrete DCE12 event writer accepts only selectors `0` and `1`.
- Provider `+0xB0` forwards the selector directly.
- Therefore the inspected Windows HWSync source-signal helper call performs no
  DCE12 MMIO for the Pro tile pair.

There is no missing Linux class-9 helper register operation to implement.
Grouped timing synchronization itself remains a separate question.

### Latest local kernel assessment

The currently checked-out `linux-imac-5k` tree is `ee0918521`. Its probe
formats and reverted force-sync/root-repulse behavior match the new captures,
although the packaged build hashes themselves are not present locally. This is
the latest tested behavior represented by `Pro/new_1`.

The separate `linux-imac-5k-pro1` tree is checked out at `2f54c1281`.
`2f54c1281` adds two behaviors together:

1. it sets `ignore_msa_timing_param` for every recognized Apple 5K tile and
   writes DPCD `0x107[7]`;
2. it forces the tiled pair into a timing-synchronization group when the
   ordinary synchronizability check rejects it.

This is aligned with two Windows RE observations:

- DCE12 mode-12 generic finalization should set MSA-ignore through `0x107[7]`;
- SetMode constructs an explicit inter-path HWSync group.

However, the patch is not yet validated by these captures and its scope is
broader than the proven divergence:

- it applies to all supported Apple 5K generations, including the working
  DCE11.2 iMac19,1;
- it combines MSA-ignore and forced grouping, so a successful result would not
  identify which operation matters;
- the branch containing the new low-level probes explicitly reverts it, so no
  single captured kernel currently contains both the probes and the real
  timing-sync result logs.

Treat `2f54c1281` as an untested experimental patch, not as a resolved fix and
not as the baseline represented by `Pro/new_1`.

### Updated priority

#### P0: resolve actual DC timing grouping

Build a logging-first kernel on the probe branch that reports the real
`timing_sync_info` without changing grouping. The marker must include:

```text
APPLE5K: timing baseline pipe=%d link[%u] group_id=%d group_size=%d master=%u
```

Also retain the existing source-selection, MSA, and blank/unblank probes.

Interpretation:

- If baseline Pro already has a two-pipe timing group, forced grouping is not
  the missing operation.
- If iMac19,1 groups but Pro does not, Phase 6C becomes the first patch axis.
- If neither groups, a Pro-only DCE12 grouping test remains justified by the
  Windows HWSync path, but it is a generation-specific experiment rather than
  a shared Apple-panel quirk.

#### P1: test the DCE12 generic-finalizer behavior

Add a Pro-only experiment for DPCD `0x107[7]` and log old/new/readback. Do not
use the shared Apple panel-ID switch as the final scope. Gate the experiment on
the iMacPro1,1/DCE12 identity or the `AE1D/AE1E` family.

Because current DC rejects ordinary timing synchronization when either stream
has `ignore_msa_timing_param`, combine MSA-ignore with a Pro-only timing-group
override only after the P0 baseline reveals whether the pair was grouped
before the flag changed.

The extra slave `0x4F1` latch can remain for the first test because it has
already proved ineffective but not harmful. Removing it is a later branch
parity cleanup, not the first experiment.

#### P2: ASSR ordering

If P0/P1 do not change the symptom, add the narrow `0x10A` phase marker already
defined in Phase 7. The new logs contain no ASSR ordering evidence.

### Deprioritized after `new_1`

- source ordinal/register-block selection;
- AUX/DDC route discovery;
- tile descriptor construction;
- plane source split;
- per-tile MSA width/start/totals;
- `DP_VID_STREAM_ENABLE` persistence;
- further `0x310 = 04/05 1d 03` trials;
- root `0x4F1` re-pulsing;
- generic link-training retries.

## User-space exclusion assessment from `new_1`

The current logs strongly exclude a user-space **modesetting, topology, or
plane-geometry** error, but dmesg alone cannot completely exclude a
user-space **pixel-content** error.

### Kernel fbdev state before the desktop takes over

Both systems initially use the in-kernel DRM client with the same aggregate
layout:

| System | Aggregate FB | Left tile | Right tile |
|---|---:|---|---|
| iMacPro1,1 | `FB:108`, 5120x2880 | source `(0,0) 2560x2880` | source `(2560,0) 2560x2880` |
| iMac19,1 | `FB:96`, 5120x2880 | source `(0,0) 2560x2880` | source `(2560,0) 2560x2880` |

The kernel assigns the eDP connector to the left CRTC at `(0,0)` and the DP
connector to the right CRTC at `(2560,0)`. This state exists before a desktop
compositor has supplied its own framebuffers.

If the visible stretch is already present on the fbcon/text console during
this interval, desktop user space is conclusively excluded.

### Desktop atomic state

At compositor takeover, the Pro continues to use the correct structure:

- eDP remains on CRTC 70 and DP remains on CRTC 74.
- Both CRTCs run 2560x2880 with no scaling or rotation.
- Each output receives a separate 2560x2880 framebuffer with local source and
  destination rectangles of `(0,0) 2560x2880`.
- AMD DC maps the right tile into aggregate source X `2560`, while its local
  destination and clip start at X `0`.
- Subsequent modesets and page flips retain the same connector/CRTC mapping
  and geometry.

The working iMac19,1 follows the same per-output framebuffer model and the
same left-X-0/right-X-2560 DC mapping. The Pro log contains no rejected desktop
atomic check, duplicate plane source, 5120-to-2560 scaling, tile swap, missing
CRTC, or other state that would explain a stretched left tile.

The first desktop buffers differ in format/layout: the Pro uses 10-bit `AB30`
with an AMD modifier, while the iMac19,1 initially uses linear 10-bit `XR30`.
The Pro later also submits `XR30` while retaining correct geometry. This is a
remaining user-space-adjacent variable, but a modifier problem would more
typically produce corruption or incorrect addressing than a stable panel-side
tile stretch.

### Confidence boundary and decisive test

The atomic trace proves the geometry and topology submitted by user space and
accepted by DRM/DC. It does not contain framebuffer pixels or a client PID, so
it cannot prove that a compositor did not render duplicated image content into
two otherwise valid 2560x2880 buffers.

To close that final gap, boot the Pro to a text-only target with amdgpu and
fbcon active but without a display manager/compositor. If the stretch remains
on that console, the issue is below desktop user space. Based on the present
logs, the leading fault domain remains the Pro-specific kernel/DC-to-panel
handoff rather than user-space KMS configuration.

## Warm-reboot state and Windows predecessor effect

The report that a Linux-to-Linux reboot can leave the left tile stretched,
while a reboot after Windows has enabled 5K does not, materially strengthens a
persistent panel/TCON state hypothesis.

This does not yet prove that Windows performs cleanup. Two explanations fit:

1. Windows shuts the pair down into a neutral state that Linux can initialize.
2. Windows leaves a coherent, fully initialized 5K state that survives the warm
   reset, while Linux leaves a different or only partially paired state.

The second interpretation currently has stronger driver evidence.

### Linux teardown facts

The Linux reboot path reaches amdgpu suspend/atomic shutdown and normal DPMS
off. For DP 1.x, `link_set_dpms_off()` blanks the stream, disables the link,
disables the stream, and calls the panel ASSR-off path. `dp_disable_link_phy()`
normally writes receiver power D3 before disabling source output and clearing
the cached link settings.

The Apple-specific code in the current working and Pro test trees only asserts
the private latch with `0x4F1 = 1`. It has no matching Apple-pair shutdown
operation that clears the latch, resets both tile endpoints as a unit, or
verifies the state left for the next boot.

Both `new_1` logs also contain repeated early DPMS-off operations on link 1
after successful training. The following status samples still show
`0x4F1 = 1` and slave ASSR `0x10A = 1`:

| System | DPMS-off samples | Later private state |
|---|---:|---|
| iMacPro1,1 | 11.550690, 11.928978, 12.162462 s | `0x10A=01`, `0x4F1=01` |
| iMac19,1 | 8.102285, 8.724242, 9.033797 s | `0x10A=01`, `0x4F1=01` |

The trace does not currently log DPCD `0x600` or the exact source-register
state at these disable points, so it cannot tell whether the receiver was put
into D3 or whether the two source paths were disabled coherently.

### Windows and firmware evidence

A fresh immediate search of the BootCamp driver found eight literal `0x4F1`
references. The display-related writers all send payload `1`; no direct
driver-side literal `0x4F1 = 0` writer was found.

Windows does have behavior Linux does not model as an Apple tile-pair
lifecycle:

- ordered pre/post stream-disable topology notifications;
- an optional PPLIB command-38 display-power-policy refresh;
- `DisableDpLinkMaybeD3PowerOff()`, where DPCD `0x600 = 2` is conditional and
  can be skipped for selected sink/branch policy;
- reverse child-path processing during blank/disable;
- cached/already-configured stream paths that avoid destructive retraining or
  receiver power-down.

Apple firmware CoreEG2 does have explicit `0x4F1 = 0` pre-pass and
display-object-destroy paths. That makes latch clear a valid firmware-shaped
experiment, but it is not evidence that the Windows runtime driver performs
the same clear on reboot.

### Relationship to the Pro1,1 failure

This can plausibly be the same fault family as the Pro stretched-left-tile
failure:

- the submitted KMS geometry and per-tile MSA are correct;
- both stream encoders are enabled;
- the symptom is consistent with panel-side tile/split state;
- the state can survive a warm reboot;
- Windows uses paired lifecycle, timing, source, and power-state decisions that
  Linux currently handles as mostly independent DP links.

It is not yet proven to be the identical cause. The decisive observation is
whether the Pro symptom changes with predecessor state: Windows -> Linux,
Linux -> Linux, and true cold AC removal -> Linux.

### P0 experiment matrix

Run the same kernel and mode in this order, recording whether the defect is
visible on fbcon before the compositor:

| Case | Predecessor and transition |
|---|---|
| A | remove AC power long enough for panel state to decay, then boot Linux |
| B | Windows reaches 5K, then warm reboot directly into Linux |
| C | Linux reaches 5K, then warm reboot directly into the same Linux kernel |
| D | Linux reaches 5K, performs a full shutdown, then boot without removing AC |
| E | Linux reaches 5K, shuts down, remove AC, then boot Linux |

Before any Apple-specific boot write, snapshot both links:

- DPCD `0x100..0x10A`, especially `0x101`, `0x102`, `0x103`, and `0x107`;
- `0x111`;
- lane/sink status `0x202..0x207`;
- source/private range `0x300..0x317`, especially `0x310` and `0x316`;
- `0x4F0..0x4F2`;
- receiver power `0x600`.

At shutdown, log the same state before teardown and after each of: stream
blank, stream disable, source-output disable, receiver D3, and final amdgpu
suspend. Include source/TG enable and timing-sync registers for both paths.

### Narrow test branches

Test only one change at a time on the exact Apple tile pair:

1. **Shutdown preserve:** skip the destructive receiver-D3/source teardown only
   during reboot. If Linux -> Linux becomes good, teardown is creating the bad
   residual state.
2. **Firmware-shaped cleanup:** write/read back root `0x4F1 = 0`, wait, then
   perform the normal paired shutdown. This tests CoreEG2-style neutralization;
   do not describe it as Windows behavior.
3. **Deterministic boot reset:** if neither shutdown branch is decisive, reset
   both endpoints at boot and rebuild the pair from D0, source DPCD, ASSR,
   training, timing/source synchronization, and final latch in one controlled
   sequence.

Do not combine this with the Pro timing-sync or `0x107[7]` experiments. The
predecessor matrix and logging-only kernel should run first so inherited state
is not mistaken for the effect of another patch.

The implementation-ready execution plan, including the shutdown hook, safety
gate, persistent instrumentation, Windows callback anchors, and acceptance
matrix, is in `Apple5K_Warm_Reboot_Parity_Plan.md`.
