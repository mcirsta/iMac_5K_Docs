# iMacPro1,1 OCLP Native 5K Handoff and DPCD 0x425 Findings

Date: 2026-06-07

## Executive Summary

An iMacPro1,1 user produced the first confirmed working Linux 5K tiled output
on this machine. The successful boot used OCLP and an experimental kernel that
preserved the display links inherited from firmware instead of performing the
normal AMD Display Core link takeover.

The result substantially narrows the problem:

- OCLP can initialize the iMacPro1,1 panel into its correct native tiled state.
- Linux DRM and KWin can inherit and use that state correctly.
- The stretched-left-tile failure is not caused by missing userspace tile
  support, incorrect EDID tile grouping, or incorrect DRM mode selection.
- Some operation during the normal AMD driver takeover of the already-active
  slave link changes the panel/TCON from native mode into stretched
  compatibility mode.
- Root-link DPCD address `0x425` appears to be a reliable read-only witness for
  the panel mode:
  - `0x00`: native/non-stretched mode
  - `0x02`: stretched compatibility mode
- Writing DPCD `0x4F1` on iMacPro1,1 can make the stretched state persist into
  an EFI boot. Avoiding that write allows EFI to restore native 5K.

The immediate engineering goal should be a narrowly gated iMacPro1,1 native
handoff preservation path. The next reverse-engineering goal should be finding
the exact operation that changes `0x425` from native to compatibility mode, and
eventually finding the sequence that changes it back without relying on OCLP.

## Source Material

Successful-state captures:

- `Pro1.1/AB_test/obs_5/modetest-good.txt`
- `Pro1.1/AB_test/obs_5/drm-state-good.txt`
- `Pro1.1/AB_test/obs_5/macos_5k_investigation/`

Earlier stretched-state comparison:

- `Pro1.1/imacpro-2026-06-03-13-22/drm-state-bad.txt`

Working experimental kernel repository:

- Repository: `https://github.com/erikolofsson/cachyos-linux`
- Branch: `pro1-apple5k-logging2`
- Last analyzed commit: `1c97035dacb08fcb7b50e30cb974b3b7404a8ebf`
- Commit subject: `Experiment 12`
- Local checkout: `cachyos-linux/`

The successful branch is a cumulative experimental branch based on common
commit `2cd9bd1daed5`. It contains a series of increasingly broad preservation
experiments and should not be treated as a merge-ready patch.

No successful-boot kernel `dmesg` was included in `obs_5`. The working-state
DRM captures and the user's experimental report establish the result, but a
successful-boot `dmesg` is still needed to independently audit the exact
`0x425` transitions and all paths taken by the driver.

## User-Reported Breakthrough

The user reported:

- DPCD `0x425` is read-only and indicates whether the display is stretched.
- A value of zero corresponds to the non-stretched state.
- Tracing this status allowed the user to identify when the working state was
  lost during boot.
- OCLP boot works when the kernel avoids changing the links during boot.
- Writing `0x4F1` makes the stretched state persist into EFI boot.
- If the kernel does not write `0x4F1`, EFI can restore native 5K mode.

These observations are consistent with the captured good DRM state and with
the progression of the experimental kernel commits.

## Proven Working DRM State

The good `modetest` capture proves that Linux sees both physical tiles and
their complete tiled-display metadata:

| Connector | Tile location | Active mode |
|---|---:|---:|
| eDP-1 / root | `0,0` | `2560x2880 @ 59.98 Hz` |
| DP-1 / slave | `1,0` | `2560x2880 @ 59.98 Hz` |

Both connectors report:

- matching Apple tiled-display topology
- two horizontal tiles and one vertical tile
- tile size `2560x2880`
- good link status
- exact matching tile timing with pixel clock `483250 kHz`

The good DRM state shows:

- CRTC 0 driving eDP-1
- CRTC 1 driving DP-1
- both CRTCs active with the same `2560x2880` tile timing
- two live KWin planes, one for each tile

Therefore, the successful Linux state is genuine two-link, two-tile
`5120x2880` output. It is not a scaled single-tile workaround.

## Userspace Is Not the Root Cause

The earlier stretched GNOME state and the successful KWin state have
essentially identical DRM topology:

- both expose the two tile connectors
- both configure two CRTCs
- both configure two `2560x2880` planes
- both use the correct tile positions and matching timings

The visible differences include compositor ownership, framebuffer identifiers,
pixel formats, and requested maximum bits per component. None explains why one
physical tile is stretched across the panel.

This rules out the following as the primary cause:

- GNOME lacking two-tile support
- KWin uniquely fixing the panel
- userspace failing to see the second tile
- incorrect DRM tile grouping
- incorrect `5120x2880` logical layout

The differentiating state exists below DRM userspace, inside the panel/TCON or
the link takeover sequence.

## DPCD State Witnesses

The experiments identify the following root-link state pattern:

| State | DPCD `0x41C` | DPCD `0x425` | DPCD `0x4F1` |
|---|---:|---:|---:|
| Native/non-stretched | `0x15` | `0x00` | `0x01` |
| Compatibility/stretched | `0x05` | `0x02` | `0x00` |

The most important new finding is `0x425`:

- It tracks the actual visible native-versus-stretched state.
- It is reported to be read-only.
- It gives the kernel and reverse-engineering work a precise success oracle.
- It should be probed before and after every candidate destructive operation.
- It must not be treated as a command register or written speculatively.

Earlier investigation of `0x42F` did not distinguish the states: both macOS and
Linux showed root `0x26` and slave `0x00`. `0x425` is therefore a much more
valuable state witness.

## DPCD 0x4F1 Is Dangerous on iMacPro1,1

The user's direct observation changes the interpretation of `0x4F1` for the
Vega/DCE12 iMacPro1,1:

- Writing `0x4F1` can latch or preserve the stretched compatibility state.
- The effect can persist across a warm reboot into EFI.
- Avoiding the write allows EFI to restore the native state.

The existing noisy Linux branch writes `0x4F1` from several places:

- post-EDID and slave polling in `link_detection.c`
- source-DPCD AUX readiness handling in `link_dp_capability.c`
- the pre-training wake path in `link_dp_training.c`
- stream-enable latch handling in `link_dpms.c`

Any iMacPro1,1 native-handoff preservation experiment must suppress all of
these writes while preserving a known-good native handoff.

This behavior must not be generalized to every Apple 5K iMac. Existing
Polaris/older-iMac evidence indicates that those machines tolerate or require
`0x4F1`. A clean implementation must be specifically gated for the
iMacPro1,1/DCE12 path and native handoff state.

## What the Working Experimental Branch Does

The cumulative experimental branch adds broad instrumentation and several
preservation controls:

- disables panel DPCD writes including `0x4F1`
- probes `0x425`, `0x41C`, and `0x4F1`
- adds broad DPCD dumps
- skips tiled-link boot cleanup blanking/power-down
- forces an eDP fast-boot-style path for the tiled root
- experiments with skipping slave retraining
- finally marks the tiled slave for seamless boot optimization

The final successful change causes the first slave `link_set_dpms_on()` to
return before the normal link-enable tail.

The successful path still performs some source-side setup and reads. It does
not literally do nothing to the links. More precisely, it avoids destructive
link state transitions, including:

- disabling an already-active slave link
- PHY enable/reset and reconfiguration
- sink preparation and power changes
- normal link training
- panel-mode and ASSR changes
- normal stream enable/unblank tail
- `0x4F1` writes

## Meaning of the Experiment Progression

The progression provides stronger evidence than the final patch alone:

| Experiment direction | Result | Meaning |
|---|---|---|
| Preserve only the root link | Did not fix | Root preservation alone is insufficient |
| Skip slave AUX training late in takeover | Did not fix | The destructive operation may occur before training |
| Skip the entire normal slave link-enable path | Works with OCLP | The destructive operation is inside the normal slave takeover path |

The failed slave-training experiment is not a clean proof that training is
innocent. Before skipping actual AUX training, the path still performs several
operations, including PHY enable, sink preparation, ASSR/panel-mode setup, and
possibly disabling the already-active link.

The strongest current candidate is the normal `enable_link()` behavior that
first calls `disable_link()` when the link is already active. This is a
high-value hypothesis, not yet a proven cause.

Other remaining candidates include:

- PHY reset or reconfiguration
- sink preparation or `DP_SET_POWER`
- ASSR programming
- panel-mode programming
- link configuration DPCD writes
- actual link training
- stream enable/unblank ordering

## What Is Proven

- OCLP can hand Linux a native, already-running two-link iMacPro1,1 state.
- Linux DRM can represent the two-tile panel correctly.
- Linux userspace can drive the two tiles correctly.
- Preserving both links during initial takeover can retain working 5K.
- The normal Linux AMD link takeover destroys or changes a required
  panel/TCON state.
- `0x425` is a useful visible-state witness according to the user's experiments.
- `0x4F1` writes are destructive or state-latching on the iMacPro1,1 according
  to the user's experiments.

## What Is Not Yet Proven

- The exact first operation that changes `0x425` from `0x00` to `0x02`.
- Whether `disable_link()` alone causes the transition.
- Whether any individual link-training operation is safe.
- How OCLP or Apple firmware changes the panel from compatibility mode into
  native mode.
- How to perform a compatibility-to-native transition from a plain firmware
  boot without OCLP.
- Whether the preserved state survives display blanking, suspend/resume, a
  mode change, VT switching, or later DPMS cycles.
- Whether `0x425` values or meanings are applicable to any non-Pro iMac.

## Limits of the Current Working Branch

The working branch is valuable experimental evidence but should not be merged
or cherry-picked wholesale:

- It contains many cumulative experiments rather than one minimal fix.
- Its Apple tiled-panel gates are broad and can affect older iMacs.
- It retains dead or disproven experimental code.
- It uses one-shot fast-boot and seamless-boot flags.
- Normal runtime DPMS-off behavior still disables links.
- It likely depends on OCLP having already initialized native mode.

The one-shot preservation flags are cleared after their first use. Therefore,
the following actions may still return the panel to stretched or blank output:

- display sleep and wake
- lock-screen blank and unblank
- suspend and resume
- mode changes
- VT transitions
- compositor restart

These must be tested explicitly.

## Recommended Clean Native-Handoff Implementation

Implement a small, Pro-specific preservation path rather than importing the
cumulative experiment branch.

### Detection and Gating

Require all of the following:

- DCE version is `DCE_VERSION_12_0`
- the links form an Apple 5K tiled root/slave pair
- an early root-link DPCD `0x425` read succeeds
- the native-state bit indicates non-stretched mode, currently observed as
  `0x425 == 0x00`

Cache this as an explicit `pro_native_handoff` state associated with the tiled
pair.

If `0x425` reports compatibility mode or cannot be read, do not claim native
handoff and do not blindly preserve the links. Fall back to the normal path
while retaining diagnostic probes.

### Behavior While Native Handoff Is Active

- suppress every iMacPro1,1 `0x4F1` write
- preserve the already-active root using a fast-boot-style path
- preserve the already-active slave using a seamless/cached-link handoff
- skip destructive boot cleanup and teardown for the pair
- retain targeted `0x425` probes before and after every relevant transition

### Safety Boundary

Do not globally change Apple tiled-panel behavior. In particular:

- do not suppress `0x4F1` on older/Polaris iMacs
- do not use panel IDs alone as the only gate
- do not assume a successful EDID tile match means native panel state
- do not write `0x425`
- do not attempt speculative writes to unknown DPCD addresses

## Operation-by-Operation Bisection Plan

Once a clean preservation baseline reliably works, restore one skipped
operation at a time. Read root `0x425` immediately before and after each
operation.

Recommended order:

1. The initial `disable_link()` on an already-active slave link.
2. `dp_enable_link_phy()` and any PHY reset/reconfiguration.
3. Sink preparation and `DP_SET_POWER`.
4. ASSR programming.
5. Panel-mode programming.
6. Link configuration DPCD writes.
7. Actual link training.
8. Stream enable and unblank.

For each operation, record:

- root and slave link identity
- DPCD read status
- root `0x425`, `0x41C`, and `0x4F1`
- slave `0x425`, `0x41C`, and `0x4F1`
- whether the operation changed either value
- whether the visible display became stretched

The first operation after which root `0x425` changes from native to
compatibility is the primary target for correction or replacement.

## Required Follow-Up Captures

From a successful OCLP/native-handoff boot, collect:

- full kernel `dmesg` containing all Apple 5K probes
- `0x425` immediately after driver probe
- `0x425` after the compositor displays its first frame
- `0x425` after display blank/unblank
- `0x425` after suspend/resume
- `0x425` after a mode change, if available
- behavior after warm reboot into EFI

Run the same non-destructive kernel from a direct firmware boot:

- If the first successful root `0x425` read is already `0x02`, that strongly
  confirms that direct boot begins in compatibility mode.
- If it begins at `0x00` and later changes to `0x02`, the probes should identify
  the destructive Linux operation.

Do not ask users to write either `0x425` or `0x4F1`.

## Plain-Boot Support Remains a Separate Stage

The preservation path can provide reliable OCLP handoff support, but it cannot
create native mode when the panel begins in compatibility mode.

Plain-boot support requires discovering the real compatibility-to-native
transition sequence. The most useful reverse-engineering targets remain:

- the Windows Boot Camp AMD driver
- Apple/OpenCore/OCLP GOP and display initialization behavior
- the DCE12/Vega-specific link and stream-source paths
- any firmware/CoreEG2 replay or reconnect behavior that OCLP triggers

DPCD `0x425` now provides a precise success oracle for that work. It should be
used to identify the exact point at which a firmware or driver sequence enters
native mode, not treated as the mechanism itself.

## Comparison With the Earlier iMac19,1 OCLP Success

The earlier iMac19,1 OCLP success at commit
`59fcbaea5930d2eb50c383cd50ec62fdb5178ead` is an important independent
comparison. It was visually confirmed as full 5K and captured as `obs_22`.

That result and Erik's iMacPro1,1 result share the same fundamental model:

1. OCLP hands Linux two real, active, already-trained tile links.
2. Linux must not perform its normal clean-slate teardown of those links.
3. Linux must not disable and freshly retrain the already-working slave link.
4. Linux can adopt the inherited links and expose a real two-tile 5K desktop.

This is no longer a hypothesis based on one machine. The same general
preservation model produced confirmed OCLP 5K on both Polaris/iMac19,1 and
Vega/iMacPro1,1.

### What Commit 59fcbaea Proved on iMac19,1

The iMac19,1 work reached success through instrumented observations rather than
a single broad skip:

- `obs_19` proved that `power_down_encoders()` killed the preserved secondary
  route by directly disabling its output.
- `obs_20` proved that preserving only the encoder was insufficient.
- `obs_21` pinned the next fatal operation to
  `power_down_controllers()` calling AtomBIOS `enable_crtc(false)` for the
  controller OCLP left driving the secondary tile.
- Commit `59fcbaea` skipped the entire destructive accelerated-mode teardown
  whenever the proven secondary route was armed for preservation.
- The later cached-link handoff preserved the active secondary through
  `enable_link()` and skipped fresh source-DPCD/cable-ID programming and link
  training.

The iMac19,1 cached-link handoff was evidence gated. It required, among other
things:

- two real tile streams
- the expected secondary route
- working AUX evidence
- an active and valid link
- known current and requested link settings
- DPCD lane-status proof that the inherited link was trained
- matching current, requested, and DPCD-derived rate/lane settings

When accepted, the iMac19,1 path:

- did not disable the already-active slave link
- did not freshly train the slave link
- did not rewrite source-specific DPCD or cable ID
- did call `setup_stream_encoder()`
- returned success from link enable
- continued through normal stream enable and unblank

The root/eDP link was not preserved by that secondary-specific mechanism. It
was allowed to take the normal fresh-training path after the global teardown
was skipped.

The result was strong:

- visually confirmed working 5K
- both `2560x2880` tiles active
- 102 stable two-tile commits over more than four minutes
- no observed secondary AUX death
- one accepted cached-link handoff and no rejected handoff
- no failed link enables

### What Erik's iMacPro1,1 Experiments Added

Erik's Pro experiments independently rediscovered both preservation
boundaries, but the successful boundary is wider:

| Experiment | Relevant behavior | Result |
|---|---|---|
| Experiment 9 | Skip tiled root/slave boot-cleanup blanking | Not sufficient |
| Experiment 10 | Force root eDP fast boot, thereby skipping global accelerated-mode teardown | Not sufficient |
| Experiment 11 | Skip the slave's actual AUX training late in the normal enable path | Not sufficient |
| Experiment 12 | Mark the slave seamless and return before the entire normal slave link-enable tail | Working OCLP 5K |

Experiment 10 is especially useful when compared with `59fcbaea`: both skip
the same broad accelerated-mode teardown. Since that alone did not fix the
Pro, the Pro also requires protection during the later slave takeover.

Experiment 11 skipped actual AUX training too late to isolate training. Before
reaching that skip, the normal path can still:

- disable an already-active link
- enable or reset the PHY
- prepare or power the sink
- program ASSR or panel mode
- perform other DPCD writes

Experiment 12 avoids the whole normal slave-link enable path. This proves that
one or more operations in that wider interval are destructive on the Pro.

### Exact Shared and Divergent Behavior

| Behavior | iMac19,1 at `59fcbaea` | iMacPro1,1 Erik Experiment 12 |
|---|---|---|
| Requires an OCLP-initialized live pair | Yes | Yes |
| Skips global accelerated-mode teardown | Yes, when secondary preservation evidence is armed | Yes, by forced root fast boot and slave seamless flag |
| Preserves root/eDP link | No special cached preservation; root can be retrained | Yes, forced eDP fast boot |
| Preserves slave against initial disable | Yes, cached-link handoff | Yes, seamless early return avoids `enable_link()` entirely |
| Skips slave fresh training | Yes | Yes |
| Allows slave `setup_stream_encoder()` | Yes | Only earlier generic stream-attribute setup; skips cached-path encoder setup |
| Allows normal slave stream enable/unblank tail | Yes | No |
| Gate quality | Narrow, evidence-based secondary route proof | Broad Apple tiled-root/slave predicates |
| `0x4F1` behavior | 103 successful writes; working 5K | Writes disabled; user reports writes latch stretched state |
| Runtime evidence | Stable over many commits for more than four minutes | Successful state captured, but no successful-boot dmesg or runtime-cycle testing yet |

### The Most Important Difference

The iMac19,1 can tolerate the residual operations performed after its
cached-link handoff. The iMacPro1,1 currently cannot, or at least has not yet
been shown to tolerate them.

Relative to Erik's seamless early return, the iMac19,1 cached-link path still
permits operations including:

- clock-manager updates
- cached-path `setup_stream_encoder()`
- a second link-encoder setup
- `enable_stream()`
- `unblank_stream()`
- normal final stream setup

Erik's seamless path returns before these normal enable/unblank operations,
although it still performs earlier source-side stream-attribute and infoframe
setup and enables stream features/audio for the external-DP slave.

This comparison creates a much smaller Pro bisection problem: start from
Experiment 12's working seamless path and restore the iMac19,1 cached-handoff
operations one at a time while checking root `0x425`.

### 0x4F1 Is a Real Machine-Specific Divergence

The two successful handoffs give opposite evidence around `0x4F1`:

- iMac19,1/Polaris produced working 5K while making 103 successful `0x4F1`
  writes.
- The iMacPro1,1 user reports that writing `0x4F1` makes the stretched state
  persist even into EFI, while avoiding it lets EFI recover native mode.

Therefore, `0x4F1` must not be part of a generic Apple 5K preservation rule.
The Pro path must suppress it while native handoff is active. The existing
iMac19,1 behavior must remain available for machines where it is proven
working.

### Recommended Combined Architecture

Use the iMac19,1 implementation as the architectural model and Erik's working
path as the initial Pro preservation boundary:

1. Detect a real Apple tile pair and collect live-link evidence.
2. On DCE12/Vega, require root `0x425` native-state proof before arming Pro
   native handoff.
3. Skip the full accelerated-mode teardown while native handoff is armed.
4. Preserve the root using fast boot initially.
5. Preserve the slave using the seamless early return initially.
6. Suppress all Pro `0x4F1` writes while native handoff remains active.
7. Replace broad panel predicates with explicit, evidence-backed handoff state.
8. Restore operations between the Pro seamless boundary and the iMac19,1
   cached-link boundary one at a time, probing `0x425` after each.

The long-term ideal is an evidence-gated cached-link handoff shared in
structure with iMac19,1, but with Pro-specific state validation and DPCD policy.
It is not yet proven that the Pro can safely use the exact iMac19,1 residual
stream-enable sequence.

### Revised Pro Bisection Order

The cross-machine comparison suggests this more precise sequence:

1. Keep Erik's complete working preservation path and obtain a successful-boot
   dmesg proving root `0x425` remains native.
2. Replace broad tile gates with a cached `DCE12 + Apple pair + root 0x425
   native` handoff flag without changing behavior.
3. Keep teardown skipped and `0x4F1` suppressed, then replace slave seamless
   return with the iMac19,1-style cached-link handoff.
4. If that stretches the panel, restore the seamless path and reintroduce the
   residual cached-link operations individually:
   - cached-path `setup_stream_encoder()`
   - second link-encoder setup
   - `enable_stream()`
   - `unblank_stream()`
   - any clock update
5. Separately test whether avoiding only the initial `disable_link()` is
   sufficient before testing PHY, sink-power, panel-mode, and training steps.
6. Probe root `0x425` before and after every restored operation.

This path uses the proven 19,1 success to reduce the Pro investigation without
assuming that Polaris-safe operations are Vega-safe.

## Final Working Model

The current evidence supports this model:

1. OCLP initializes both iMacPro1,1 panel links and the panel/TCON into native
   tiled mode.
2. Root DPCD `0x425` reports the native state as zero.
3. Linux discovers correct EDIDs, tile metadata, modes, and connectors.
4. Normal AMD link takeover performs an operation that changes the panel/TCON
   into stretched compatibility mode.
5. DRM and userspace continue driving two apparently correct tile streams, but
   the panel internally stretches the left tile.
6. Preserving both inherited links avoids that destructive transition and
   produces correct 5K output.
7. Writing `0x4F1` can latch the bad compatibility state, including across a
   warm reboot.

This model explains why the stretched and working Linux DRM states can look
nearly identical while the physical display output differs dramatically.
