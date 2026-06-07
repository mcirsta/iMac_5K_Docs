# iMacPro1,1 DPCD 0x425 and 0x4F1 RE Instructions

Date: 2026-06-08

## Mission

Use IDA Pro through MCP to determine how the Windows AMD/BootCamp display
driver handles DPCD `0x425` and `0x4F1` on iMacPro1,1.

Primary existing IDA database: `B416406/amdkmdag.sys.i64`.

The immediate questions are:

1. Does the Windows driver read `0x425`, directly or as part of a larger DPCD
   block read?
2. Does a normal iMacPro1,1 tiled modeset write `0x4F1` at any lifecycle
   phase?
3. Which exact Windows state and branch conditions permit a Pro1,1 handoff
   without a late `0x4F1` write?
4. What operation distinguishes a preserved already-native handoff from a
   destructive fresh takeover?
5. Is there any Windows sequence that changes a compatibility/stretched panel
   into native tiled mode?

The objective is evidence, not a speculative Linux patch. Record exact
functions, callers, gates, link objects, and ordering.

## New Runtime Evidence To Explain

An iMacPro1,1 user produced a confirmed working 5K Linux state after booting
through OCLP and preserving the already-active display links.

The user reports:

- Root-link DPCD `0x425` is read-only.
- Root `0x425 == 0x00` corresponds to native/non-stretched output.
- Root `0x425 == 0x02` corresponds to the stretched-left-tile failure.
- Writing `0x4F1` can make the stretched state persist across a warm reboot
  into EFI.
- Avoiding link takeover and `0x4F1` writes preserves OCLP's working native
  state.

Treat these as strong runtime observations, but do not silently promote them
to proven Windows-driver semantics.

Source material:

- `Pro1.1/AB_test/obs_5/`
- `Pro1.1/AB_test/obs_5/modetest-good.txt`
- `Pro1.1/AB_test/obs_5/drm-state-good.txt`
- `Pro1.1/AB_test/obs_5/macos_5k_investigation/`
- `cachyos-linux/`, branch `pro1-apple5k-logging2` as last analyzed
- `iMac_5K_Docs/RE_iMacPro1_1_OCLP_Native_Handoff_0x425.md`

The successful experimental Linux branch is cumulative and broad. Use it as
evidence about the destructive interval, not as a model of the Windows path.

## Facts Already Established By Static RE

### Pro1,1 identity and topology

- Root panel/compatibility EDID: `APP/AE1D`
- Tiled/post-wake identity: `APP/AE1E`
- Root display object: `0x3114`
- Slave display object: `0x3113`
- Root DCE12 source: `5`
- Slave DCE12 source: `4`
- GPU/DC family: Vega/Japura, DCE12
- Pro table tags:
  - `AE1D -> tag 0x3A`, descriptor byte `+6` bit 2
  - `AE1E -> tag 0x3B`, descriptor byte `+6` bit 3

### Known `0x4F1` stream-enable behavior

`StreamEnableWrite4F1AndPollSinkStatus()` at `0x1414E7810` has two important
branches:

- The normal branch can write DPCD `0x4F1 = 1` after link, source, and stream
  programming when descriptor bit 2 or bit 3 is present.
- The state-1/already-configured branch returns without the later tag-gated
  `0x4F1` write.

`EnableValidatedStreamMaybeAssert4F1()` at `0x141446B40` is the only known
direct caller of that worker.

`UpdateMode11ImmediateStreamEnableFlag()` at `0x141442720` selects the worker
only when the display-object mode resolves to `11`:

- natural mode `11`: enter the immediate worker
- natural mode `12`: clear the immediate flag and use the generic finalizer
- mode `12` can report as `11` only when the explicit coercion byte at
  display-object `+0x1E9` is set

`DisplayObject_GetCommittedSignalTypeForModeGate()` is at `0x141549190`.
`DisplayObject_SetSignal12ReportsAs11Flag()` is at `0x141547CB0`.

For a normal two-tile Apple 5K source split, path metadata bit 7 is expected to
keep DCE12/Vega in mode `12`, without the iMac19,1-style immediate
`0x316`/`0x4F1` worker.

This establishes that a Windows-shaped Pro1,1 path may legitimately have no
late stream-enable `0x4F1` write. It does not prove that Windows never writes
`0x4F1` on Pro1,1 through another writer or lifecycle phase.

### Other known `0x4F1` writers

The known writer classes are not equivalent:

- detect/status writer:
  - biased toward tag `0x3A` / descriptor bit 2
- `WindowsDM_EnableSecondaryTileIfRequired()`:
  - accepts descriptor bit 2 or bit 3
  - additionally requires selected display/link type `128`
- mode-11 immediate stream-enable writer:
  - accepts descriptor bit 2 or bit 3
  - writes through the link-service AUX object

Known inspected Windows paths write payload `1`. No inspected Windows path has
yet shown a `0x4F1 = 0` write or a `1 -> 0 -> 1` pulse.

### What is not established

- No prior RE note found a meaningful `0x425` reference.
- The macOS capture does not contain AUX transaction traces and cannot prove
  whether macOS writes `0x4F1`.
- Static RE has not proven that a live Pro1,1 always stays in natural mode
  `12`.
- Static RE has not proven that no detect-time or other `0x4F1` writer can run
  on Pro1,1.
- The operation that changes `0x425` from `0x00` to `0x02` is unknown.
- No compatibility-to-native sequence is known.

## Evidence Rules

Every conclusion must be labeled as one of:

- **Proven static:** exact code path, constants, gates, and callers are mapped.
- **Proven runtime:** captured on the relevant machine with a controlled test.
- **Supported inference:** follows from mapped code and known Pro1,1 state, but
  live branch execution has not been observed.
- **Hypothesis:** useful lead without sufficient proof.

Do not use these statements without qualification:

- "Windows never writes `0x4F1` on Pro1,1."
- "`0x425` is a command register."
- "`0x425` has the same meaning on other iMacs."
- "Removing `0x4F1` is correct for all Apple 5K panels."
- "The state-1 branch and the generic mode-12 finalizer are the same path."
- "The working OCLP result proves plain-boot support."

The iMac19,1 working OCLP path performed successful `0x4F1` writes. Any
no-`0x4F1` conclusion must remain Pro1,1-specific unless separate evidence
proves otherwise.

## Priority 1: Find Every Possible Read Of DPCD 0x425

Search for all literal forms:

- hexadecimal `0x425`
- decimal `1061`
- little-endian immediate bytes `25 04 00 00`
- constructions such as `0x400 + 0x25`

Do not stop if there is no literal xref. Find all DPCD block-read helpers and
identify calls whose address range covers `0x425`, including examples such as:

- a read beginning below `0x425` with sufficient length
- a cached DPCD snapshot covering the `0x400` private range
- a helper that computes the address from a base plus a field or index
- a loop that walks vendor-private DPCD registers

For every direct or covering read, record:

- function address and proposed name
- exact DPCD start address and length
- whether the address is constant or computed
- AUX/link object used
- root versus slave reachability
- all comparisons or masks applied to the byte corresponding to `0x425`
- every caller and caller phase
- behavior for observed values `0x00` and `0x02`
- whether the result is cached into sink, link, display, or policy state

If no read is found, document the negative result precisely:

- which literal searches were performed
- which DPCD read wrappers were enumerated
- which block-read ranges were checked
- whether dynamic/computed-address calls remain unresolved

Do not conclude that Windows ignores `0x425` merely because the literal does
not occur.

## Priority 2: Prove All Pro1,1-Reachable 0x4F1 Paths

Search for:

- hexadecimal `0x4F1`
- decimal `1265`
- little-endian immediate bytes `F1 04 00 00`
- computed or table-driven DPCD writes that can resolve to `0x4F1`

Build one complete table with a row for every writer:

| Writer | Payload | AUX/link object | Caller phase | Gate | AE1D reachable | AE1E reachable | Natural DCE12 reachable |
|---|---:|---|---|---|---|---|---|

For each writer, recover:

- payload and retry behavior
- delay before or after the write
- readback or status polling
- root/slave object selection
- descriptor/tag gates
- display/link type gates
- display-object mode gates
- cached/state gates
- DCE/family/version gates
- shutdown, disable, retry, and recovery callers

Explicitly answer:

1. Can the AE1D/root detect/status path write `0x4F1` before the mode-12
   finalizer decision?
2. Can the AE1E/slave reach any writer without mode-11 coercion?
3. Can type `128` occur on the normal Pro1,1 root or slave path?
4. Can a failure/retry path write `0x4F1` even when the successful path would
   not?
5. Does any disable, cleanup, resume, or recovery path write `0x4F1`?
6. Is there any Windows writer with payload `0`, or any write sequence other
   than payload `1`?

The final result must distinguish:

- no late stream-enable write
- no write during the first modeset
- no write during the entire driver lifetime

These are different claims.

## Priority 3: Prove The Normal Pro1,1 Branch

Trace the complete branch chain for a normal two-tile Pro1,1 modeset:

1. Pro1,1 table-row match for `AE1D` and `AE1E`.
2. Display objects `0x3114` and `0x3113`.
3. DCE12 source selection `5/4`.
4. SetMode tile-group anchor selection.
5. Path timing metadata bit 7.
6. `12 -> 11` coercion byte `+0x1E9`.
7. Committed mode returned by
   `DisplayObject_GetCommittedSignalTypeForModeGate()`.
8. Immediate-worker versus generic-finalizer selection.
9. State-1/live-trained-link decision.
10. Every `0x4F1` writer reachable before and after finalization.

Determine where path metadata bit 7 originates. It is not enough to show how
later code consumes it. Identify the producer, the conditions that set or
clear it, and whether a normal Pro1,1 two-tile mode has enough evidence to
guarantee the expected value.

Also determine whether the first driver takeover differs from later modesets,
resume, hotplug-style detection, or recovery. The OCLP/native handoff may
enter a different state than a clean Windows initialization.

## Priority 4: Separate State-1 From Generic Mode-12 Finalization

Do not merge these into a generic "cached path."

For state `1`, prove:

- who sets state `1`
- required live DPCD proof
- required cached tuple
- whether both Pro links can enter state `1`
- what link/source/stream programming still occurs
- why the path returns before late `0x4F1`

For the natural mode-12 generic finalizer, prove:

- whether it can run with or without a live inherited link
- whether it trains, reuses, disables, or reconfigures the link
- which source-DPCD, ASSR, panel-mode, and stream writes still occur
- whether it calls any other `0x4F1` writer

Then compare both paths to Erik's working preservation behavior. Identify the
smallest Windows path that adopts an already-native Pro1,1 link without
performing an operation suspected of changing `0x425`.

## Priority 5: Find The Destructive Transition Candidate

The successful OCLP experiment narrows the destructive interval to operations
performed during normal slave-link takeover. Trace the Windows equivalents of:

1. disabling an already-active link
2. PHY enable, reset, or reconfiguration
3. sink preparation and `DP_SET_POWER`
4. ASSR programming at DPCD `0x10A`
5. panel-mode programming
6. link-configuration writes
7. actual link training
8. source-DPCD programming
9. stream enable and unblank
10. `0x4F1` writes

For each operation, determine:

- whether state-1 skips it
- whether generic mode-12 finalization performs it
- whether the operation is root-only, slave-only, or paired
- whether it has a Pro/DCE12-specific branch
- whether failure or retry changes the sequence
- whether it is ordered differently from iMac19,1/DCE11

Look especially for a Windows branch that preserves an already-active
secondary instead of first disabling it.

The desired output is an ordered operation matrix, not only a list of helper
functions.

## Priority 6: Search For A Compatibility-To-Native Sequence

The preservation path only explains how to keep an OCLP-provided native state.
Plain boot requires discovering how native state is created.

Search for functions that:

- run only for Apple tiled panels, Pro1,1/Japura, or DCE12
- operate on both `0x3114` and `0x3113`
- change panel mode, MST/SST behavior, ASSR, link power, or stream topology
- run before the first full tiled SetMode
- reconnect, rediscover, or replay firmware display state
- issue vendor-private DPCD reads or writes in the `0x400` range

Correlate any candidate sequence with:

- the first point both tiles become available
- a possible `0x425` read or state check
- absence or presence of `0x4F1`
- mode `12` versus coerced mode `11`
- root/slave ordering

If Windows contains no visible `0x425` use and no compatibility-to-native
sequence, state that clearly. The transition may belong to GOP, Apple
firmware, OCLP, or panel/TCON behavior rather than the Windows display driver.

## Comparison Target: iMac19,1

Use iMac19,1 only as a controlled contrast:

- iMac19,1/DCE11 naturally reaches mode `11`.
- Its working OCLP handoff tolerated successful `0x4F1` writes.
- Pro1,1/DCE12 naturally appears to remain mode `12`.

For every important Pro result, record whether the corresponding iMac19,1 path
differs because of:

- DCE generation
- display-object mode
- source pair
- panel/tag row
- path metadata bit 7
- cached/state-1 status
- different writer reachability

Do not import iMac19,1 behavior into the Pro path by analogy.

## IDA/MCP Working Procedure

1. Preserve existing resolved names and comments.
2. Rename newly proven functions with behavior-based names.
3. Add comments at every important branch with:
   - input object and relevant fields
   - exact gate
   - resulting path
   - confidence level
4. Enumerate direct and indirect callers before declaring reachability.
5. Follow vtable calls to their concrete implementations where possible.
6. Track the AUX/link object through each DPCD transaction.
7. Record unresolved dynamic dispatch instead of guessing.
8. Save the IDB after each coherent investigation phase.

When a constant search is negative, pivot to the DPCD read/write wrapper and
enumerate its callers. Private register accesses may be block-based,
table-driven, or computed.

## Required Deliverables

Produce or update a Markdown report containing:

### 1. `0x425` access map

- all direct accesses
- all block reads covering it
- computed-address candidates
- comparisons and resulting branches
- precise negative-search coverage if no access is found

### 2. Complete `0x4F1` writer map

- every writer and caller
- exact payloads
- root/slave selection
- Pro1,1 reachability
- first-modeset, runtime, resume, cleanup, and recovery reachability

### 3. Pro1,1 branch proof

- path metadata bit-7 producer
- mode `12` versus `12 -> 11` coercion
- state-1 versus generic-finalizer selection
- exact point where late `0x4F1` is skipped or permitted

### 4. Ordered takeover comparison

Compare:

- native inherited/state-1 path
- natural mode-12 generic path
- coerced mode-11 immediate path
- iMac19,1 mode-11 path

### 5. Patch-relevant conclusion

State only what the evidence supports:

- whether suppressing `0x4F1` is Windows-shaped for a native Pro handoff
- whether another Pro-reachable writer must also be suppressed
- which operation is the strongest destructive-transition candidate
- whether any Windows-derived plain-boot/native-entry sequence was found

Include exact function addresses, xrefs, structure offsets, constants, and
remaining unresolved calls.

## Stop Conditions

Stop and report rather than speculate when:

- the only remaining path is unresolved indirect dispatch
- a claimed Pro branch depends on an unproven runtime field value
- no Windows-driver access to `0x425` remains after complete wrapper/range
  analysis
- the native-entry sequence appears to live outside the Windows driver

A clean negative result is valuable. In particular, finding no `0x425` read in
the fully audited driver image would support treating it as a useful external
state oracle rather than a Windows policy input. Do not generalize that result
to other driver versions or firmware.

## Related Notes

- `iMac_5K_Docs/RE_iMacPro1_1_OCLP_Native_Handoff_0x425.md`
- `iMac_5K_Docs/RE_iMacPro1_1_IDA_MCP_Findings.md`
- `iMac_5K_Docs/RE_iMacPro1_1_Divergence_Plan.md`
- `iMac_5K_Docs/RE_iMacPro1_1_Windows_Driver_IDA_MCP_Handoff.md`
- `iMac_5K_Docs/RE_iMacPro1_1_TileSplit_SourceSignal_Handoff.md`
- `iMac_5K_Docs/BootCamp_monitor_capability_table_diff.md`
