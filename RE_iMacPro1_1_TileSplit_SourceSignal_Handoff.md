# RE Handoff: iMacPro1,1 DCE12 Tile-Split Source Mode (Candidate C + B)

Date: 2026-06-07
IDA database: `B416406/amdkmdag.sys.i64`
Access: IDA Pro via MCP. Keep a written trail of every rename / xref / decompile
that supports a conclusion.

## Why this RE (read first)

We have exhaustively matched Linux amdgpu/DC against the working iMac19,1 and
against the Windows driver at the **value** level: source 5/4, per-tile MSA
(`hstart=112 hwidth=2560`), BE→FE routing, ASSR, DPCD `0x4F1`/`0x107`/`0x310`,
link training, and the eDP panel power sequence (after fixing a real DCE12 SOC15
register-addressing bug) **all match**, yet the iMacPro1,1 still composes the
left tile stretched across the whole panel. macOS — and, by the AMD BootCamp
driver, Windows — composite the same hardware correctly.

Conclusion: the gap is **not a wrong register value**, it is a **missing
operation** in Linux's DCE12 path. The leading hypothesis is that Windows drives
the two tiles as a **coordinated inter-path pair** (one logical 5120 source split
across two tile objects, with an inter-path source-signal adjustment), while
Linux drives **two independent 2560 streams that merely happen to be genlocked**.
The AE25/AE26 panel tolerates independent streams; the iMacPro AE1E panel may
require the coordinated-pair treatment.

Already implemented/tested (so do NOT re-RE these):
- Source ordinal 5/4, MSA, routing, ASSR, `0x4F1`, `0x310`, training — match.
- Forced panel power cycle — runs now, does **not** fix the stretch.
- Direct DPCD `0x107[7]` MSA-ignore (candidate A) — implemented separately,
  pending test.

## Goal

Produce a **concrete, replicable artifact**: the exact Vega MMIO register
write(s) (and/or DPCD writes), with values and the gating condition, that the
Windows DCE12 **tile-split / inter-path** path emits and that Linux's
independent-stream DCE12 path does **not**. Failing that, prove the difference is
purely structural (one 5120-split source vs two 2560 streams) and state exactly
which struct fields encode it.

## Candidate C: tile-split source mode (structural)

Primary anchors:

- `BuildSourceModeForTileSplitPath` at `0x14140BA40` — on `path_entry+0x18->+0x1C`
  bit7 set + valid two-tile group, "doubles the source width, then
  halves/programs the timing across both tile display objects."
- `SetMode_FindTileGroupAnchorDisplay` at `0x141507AB0` — validates the tile
  group (common tile-group id, full grid coverage, anchor flag).
- `SetMode_ApplyMstToSstOptimizationSignalCoercion` — consumes the same bit7.

Tasks:

1. Decompile `BuildSourceModeForTileSplitPath`. Determine **exactly what it
   writes into the per-tile source-mode / stream struct**, especially:
   - Does it set a per-tile **horizontal source OFFSET** or tile-position field
     for the second (right/DP) tile, vs the first? (Linux programs both tiles
     identically — `hstart=112` on both. If Windows offsets tile 1, that is a
     concrete difference.)
   - Does it set a "this stream is half of a wider source" / tile flag the lower
     layers consume?
2. Follow the tile-split output into final source programming:
   - `DpPathLower_ProgramSourceModeViaSourceObject` at `0x14169DB60`
     (forwards `role_payload[0]` source index, `role_payload[3]` mode-gated
     signal).
   - `Dce12SourceObject_ProgramDpSourceIndexMode` at `0x1416B1D90`
     (writes `Dce12SourceRegisterBaseOffset +
     Dce12SourceRegisterBlockOffsetBySource[source] + 6465`).
   Compare the register **values** written in the tile-split (bit7) case vs a
   plain single-display case for the same source. Is anything different beyond
   the source index?

## Candidate B: inter-path source-signal register (concrete — highest value)

This is the most likely place a **replicable register write** lives. Prior RE
(Phase 6E) found that when Windows forms the inter-path HWSync group for the two
tiles, it performs a source-signal adjustment that writes DCE12 DP source-control
registers — something Linux (which only does the GSL/OTG timing genlock) does
**not** do.

Primary anchors:

- `HWSync_AdjustDpMstEdpSourceSignalForInterPath` at `0x14156F180` — consumes
  `stream_arg+0x168 == 1` (inter-path group flag), calls provider source-signal
  `+0xB0` with current source-select index and peer class-9 helper id `9..14`
  (Pro source 5/4 → ids `14/13`).
- `ProviderSourceSignal_ApplyHWSyncSignalTypeViaEventHandle` at `0x141628FF0`
  → provider stream-event handle `+0xB8`.
- `Dce12SourceSignalEventHandle_ApplySignalTypeToRegs` at `0x141626820` — the
  actual writer. Per RE notes it writes, relative to base `unk_141F8CD78`:
  - `a3 == 0`: `base + offset + 6475` bit0.
  - `a3 == 1`: `base + offset + 6434` bit0, and `base + offset + 6465`
    with mask `0xEFFFFFFF`.
  where `offset = Dce12SourceRegisterBlockOffsetBySource[source]`,
  table `{0, 0x100, 0x200, 0x300, 0x400, 0x500}`.

Tasks (do these first — they yield the concrete deliverable):

1. **Resolve `unk_141F8CD78` (the source-register base) to an absolute Vega MMIO
   register address.** Then resolve `base + 0x400 + 6465` (source 4) and
   `base + 0x500 + 6465` (source 5) to absolute addresses.
2. **Map those absolute addresses to named registers** in
   `drivers/gpu/drm/amd/include/asic_reg/dce/dce_12_0_offset.h` (or the relevant
   Vega DP/DIG header). Note: amdgpu register defs are dword indices with a
   SOC15 segment base — convert carefully. Likely candidates are `DP*`/`DIG*`
   source-control / DP-symbol registers.
3. Determine the exact **value** written (bit0 set/clear, the `0xEFFFFFFF` mask
   = clearing bit 28) and what bit28/bit0 mean (decompile the writer fully).
4. Determine the **gating**: confirm this only runs for the inter-path group
   (`stream_arg+0x168 == 1`), i.e. it is tied to the two-tile timing-sync group,
   and what sets that flag (`SyncManager_ApplyInterPathSynchronization`).
5. Resolve the `a3`/`R8D` argument semantics (peer class-9 id `9..14` vs sentinel
   `8`) enough to know which branch the Pro's source 5/4 tile pair takes.

## Map to the Linux side (for whoever implements the result)

- Linux DCE12 DP source select today: `dce110_link_encoder_connect_dig_be_to_fe`
  (`DIG_BE_CNTL.DIG_FE_SOURCE_SELECT`) — confirm whether the Windows
  source-signal register from Candidate B is a **different** register than
  `DIG_BE_CNTL` (it almost certainly is — it carries a signal-type, not just the
  FE select).
- DCE12 source object regs in Linux: `dce/dce_stream_encoder.c`,
  `resource/dce120/dce120_resource.c`. Vega register headers under
  `include/asic_reg/dce/dce_12_0_*`.
- The two-tile group on Linux: `link->tiled_peer`, the `tiled_*` panel patches,
  and `program_timing_sync` (which already forms the GSL group — the timing half
  of what Windows does; Candidate B is the **source-signal half** Linux lacks).

## Success criteria

One of:

1. **Concrete (preferred):** "Windows writes Vega register `mmXXX` (offset
   `0x…`) = `…` for source 5 and source 4 when the two tiles form an inter-path
   group; Linux does not. Linux patch point: `<file:function>`." With the gating
   condition.
2. **Structural:** "The tile-split produces per-tile field `…` (e.g. a
   right-tile horizontal offset / tile flag) that Linux's independent-stream
   model does not set," with the exact field and where Linux would set it.

## Deliverable format (match the existing RE docs)

```text
Phase: <tile-split source mode | inter-path source-signal reg>
Windows evidence: <func+offset> writes <reg/field> = <value>, gated on <cond>
Linux current: <file:function> does <what> / omits <what>
Conclusion: <the missing operation>
Patch candidate: <exact register/DPCD write + Linux insertion point>
```

## Guardrails

- Do not restart value-level comparisons (source/MSA/routing/ASSR/`0x310`/`0x4F1`)
  — those are matched and confirmed.
- The target is a **missing operation**, specifically the source-signal /
  source-mode writes tied to the **inter-path two-tile group**, not another DPCD
  byte experiment.
- Prefer Candidate B's register mapping first (it yields an implementable write);
  use Candidate C to explain the gating / why the write only applies to a true
  tile pair.
