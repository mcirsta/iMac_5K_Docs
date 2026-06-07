# RE iMacPro1,1 0x425 / 0x4F1 IDA MCP Findings

Target IDB: `B416406/amdkmdag.sys.i64`

This report records the first IDA MCP pass requested by
`RE_iMacPro1_1_0x425_0x4F1_IDA_MCP_Instructions.md`. The instruction document
was read end to end before analysis. Existing 5K RE notes were scanned first so
this pass does not re-establish already-known Pro1,1 topology, AE1D/AE1E tags,
source pair, or failed Linux-side experiments.

Evidence labels used below:

- Proven static: directly shown in this IDB by literals, wrappers, or call graph.
- Supported inference: follows from static evidence plus prior project findings.
- Hypothesis: plausible but not yet proven by static or runtime evidence.
- Open runtime: needs AUX/runtime tracing or a live-machine capture.

## Prior notes used

The following existing documents were used as guardrails:

- `RE_iMacPro1_1_IDA_MCP_Findings.md`
- `RE_iMacPro1_1_Divergence_Plan.md`
- `RE_iMacPro1_1_Windows_Driver_IDA_MCP_Handoff.md`
- `RE_iMacPro1_1_TileSplit_SourceSignal_Handoff.md`
- `RE_iMacPro1_1_OCLP_Native_Handoff_0x425.md`
- `BootCamp_monitor_capability_table_diff.md`
- `Apple5K_Warm_Reboot_Parity_Plan.md`

Important inherited anchors, not rediscovered here: `APP/AE1D` is the
root/compat object, `APP/AE1E` is the tiled/post-wake object, root/slave object
ids are `0x3114`/`0x3113`, and Pro1,1 DCE12/Japura/Vega uses source pair `5/4`.
Existing notes also already warned that source-DPCD `0x310`, ASSR `0x10A`, MSA
`0x107`, and simple Linux `0x4F1=1` replay were insufficient.

## 0x425 access audit

### Literal search

Proven static:

- Raw byte pattern `25 04 00 00` appears 140 times.
- IDA metadata classifies those raw matches as 15 instruction-overlap locations
  and 125 non-code/unknown/data locations.
- Immediate-aware search for `0x425` / decimal `1061` finds only 3 code operands:

| Address | Function | Classification |
| --- | --- | --- |
| `0x14003554e` | `sub_1400353F0` | Trace/event id passed to `sub_1400751A0`, paired with `0x424`; not AUX/DPCD. |
| `0x1402d6c96` | `sub_1402D68D0` | Table/offset entry stored by `sub_1402D5510` into a local descriptor array; not AUX/DPCD. |
| `0x1417d9743` | `sub_1417D9698` | Vega20 hwmgr source-line number in a trace/assert path; not AUX/DPCD. |

No IDA-recognized code immediate uses `0x425` as a DPCD address.

### Known AUX read wrappers

Proven static:

- `LowerControllerDpcdAuxReadAddress` (`0x141EDDCD0`) wraps the lower miniport
  AUX-read vfunc. Its direct callers read known ranges such as `0x170` and
  `0x2006`; none pass `0x425`.
- `ChunkedLowerControllerDpcdReadWithRangeFixup` (`0x141BB9460`) was audited
  through all 44 caller functions reported by xrefs. Constant/ranged callers
  cover standard DP capability, link-status, LTTPR, HDCP, audio, and power
  ranges, but not `0x425`.
- The near misses are explicit:
  - `RetrieveDpLinkCapsAfterSourceDpcdSetup` reads `0x400` length 9 and `0x409`
    length 3, ending at `0x40B`.
  - HDCP helpers read high ranges such as `0x6802C`, `0x70000`, `0x70001`,
    `0x70010`, and `0x70810`.
  - The large DCN/DCE init routines that were unresolved in the earlier pass all
    call the chunked read wrapper with `0x600` length 1.
- Dynamic-looking chunked callers resolved to non-`0x425` ranges:
  - `sub_141B85DA0`: `0x271`, `0x272`, and `0x273 + i` for audio capability
    bytes.
  - `sub_141B88D90`: LTTPR high range `0xF0020 + 0x50 * (i - 1)`.
  - `LowerReadLaneStatusAndAdjustRequests`: `0x202` length 6 or LTTPR high
    equivalent.
  - `sub_141BB4B00`: HDCP helper over `0x6802C` or table-driven high HDCP
    ranges; table values inspected begin in `0x68000`/`0x69000` space.
- `DpcdAuxReadUpTo16Bytes` (`0x14154F320`) is a display-map AUX read interface
  with vtable/data xrefs rather than direct code xrefs. No direct `0x425`
  immediate or known range feeds it in this pass.

Conclusion, supported inference: this IDB does not show Windows reading DPCD
`0x425` by direct literal, by the lower chunked AUX read wrappers, or by the
direct lower AUX read wrapper. The strongest runtime proof would still be an AUX
trace/watchpoint, but the static evidence does not support a Windows
`0x425`-based handoff decision.

## 0x4F1 writer audit

### Literal search

Proven static:

- Raw byte pattern `F1 04 00 00` appears 19 times.
- Immediate-aware search for `0x4F1` / decimal `1265` finds 8 code operands.
- Real DPCD writers are confined to three helper families below. The remaining
  code immediates are unrelated structure/logging uses (`0x14009492a`,
  `0x140099b8f`, `0x141895ebd`).

### Writer table

| Writer | Payload | Transport | Gate / caller evidence |
| --- | --- | --- | --- |
| `WriteSinkDpcd4F1Payload1Retry` (`0x1414B03F0`) | `1` | Display-map entry `+0x18` AUX interface | Called by `DetectDisplayMaybeAssert4F1ForCapableSink`; retries after 10 ms on failure. |
| `StreamEnableWrite4F1AndPollSinkStatus` (`0x1414E7810`) | `1` | Link-service AUX object at `this+0x90` | Reached only through `EnableValidatedStreamMaybeAssert4F1` when the mode-11 immediate flag is set; retries after 10 ms. |
| `SendSecondaryTileOpcode1265Payload1` (`0x141B716F0`) | `1` | `LowerControllerDpcdAuxWriteAddress` | Called by `WindowsDM_EnableSecondaryTileIfRequired` and `BringUpDisplayBySignalType`; retries after 10 ms and may sleep 100 ms after success. |

No direct `0x4F1=0` writer was found in this direct-immediate pass. All real
static DPCD `0x4F1` writers found here send payload `1`.

### Reachability and gates

Proven static:

- Detect/status writer:
  - `DetectDisplayMaybeAssert4F1ForCapableSink` gates the early writer on pin
    descriptor byte `+6` bit 2 only.
  - It does not accept bit 3 by itself, so a pure AE1E/tag-`0x3B` case is not
    proven to use this route.
  - After the write it sleeps 100 ms and rechecks sink status.
- Stream-enable writer:
  - `UpdateMode11ImmediateStreamEnableFlag` sets the immediate flag only for
    display object mode `11`.
  - Display object mode `12` clears the immediate flag.
  - Therefore `EnableValidatedStreamMaybeAssert4F1` calls
    `StreamEnableWrite4F1AndPollSinkStatus` only for the mode-11 immediate path.
  - Inside the writer, descriptor byte `+6` bit 2 or bit 3 gates `0x4F1=1`.
  - The real stream-enable vfunc is called before the optional `0x4F1` write.
  - State field `this+0x58` values `2`/`3` return success immediately.
  - State field value `1` programs link/source and stream-enable, then promotes
    state to `3`, returning before the later tag-gated `0x4F1` write.
- Generic/cached finalizer:
  - `ProgramCachedLinkStreamSourceThenEnableStream` still performs real work:
    peer/source update, stream-source programming with cached tuple, marks the
    display object enabled, then calls the real stream-enable vfunc.
  - That generic path does not call `StreamEnableWrite4F1AndPollSinkStatus`.
- Secondary-tile writer:
  - `WindowsDM_EnableSecondaryTileIfRequired` accepts descriptor byte `+6` bit 2
    or bit 3, but also requires selected link/display type `128`.
  - `BringUpDisplayBySignalType` can call the same helper after successful
    bring-up if lower display field `+0x5B4` is nonzero. Existing notes say this
    field is copied from tag `0x3B` value; static AE26 has tag `0x3B` present
    but value 0, so this route is not proven as the main AE26 path.

Supported inference: a normal Pro1,1 mode-12/generic tiled handoff is not proven
to write late `0x4F1`. The most explicit stream-enable `0x4F1` writer is
mode-11/immediate specific, while mode 12 is routed to the generic/cached
finalizer that still enables the stream without that late write.

## Answers to the instruction questions

1. Does Windows read DPCD `0x425`?
   - Proven static: no direct code immediate uses `0x425` as DPCD, and audited
     known lower AUX read wrappers do not read a range covering `0x425`.
   - Supported inference: Windows is not making the observed Pro1,1 handoff
     decision from DPCD `0x425`.

2. Does normal Pro1,1 tiled modeset write `0x4F1`?
   - Proven static: Windows contains gated `0x4F1=1` writers.
   - Supported inference: normal mode-12/generic Pro1,1 handoff is not shown to
     require or perform the late stream-enable `0x4F1` write. A runtime trace is
     needed to prove whether the secondary helper fires on a specific boot path.

3. What branches allow handoff without late `0x4F1`?
   - Proven static: mode 12 clears the immediate flag and enters the generic
     cached/finalizer path; state `1` in the immediate worker also skips the
     later `0x4F1` block; states `2`/`3` return success.
   - Supported inference: preserved native handoff likely depends more on
     display-object mode/state, mapped-sink presence, and cached link tuple than
     on replaying `0x4F1`.

4. What distinguishes preserved already-native handoff from destructive takeover?
   - Supported inference: preserved handoff is the path where Windows keeps a
     valid mapped sink/display entry and cached link tuple, then performs the
     generic stream finalizer without fresh destructive link bring-up.
   - Supported inference: destructive takeover is the path where validation or
     cached state is missing and Windows falls back into fresh link programming,
     source-DPCD setup, link training, or immediate-mode workers.

5. Is there a Windows compatibility-to-native sequence?
   - No static sequence was found in this pass that reads `0x425`, writes
     `0x4F1=0`, then transitions to native. Existing CoreEG2 firmware notes may
     still describe firmware behavior, but this Windows IDB pass did not prove a
     Windows `0x425` or `0x4F1=0` compatibility-to-native sequence.

## Follow-up runtime checks

Open runtime:

- AUX trace/watchpoint for `0x425` reads across cold boot, warm reboot, hotplug,
  and sleep/wake.
- AUX trace/watchpoint for `0x4F1` writes, including payload and selected link,
  while distinguishing detect-time, stream-enable, and secondary-tile callers.
- Runtime confirmation of display object mode `11` vs `12`, stream-enable state
  `this+0x58`, and cached-link flag before the generic finalizer runs.

