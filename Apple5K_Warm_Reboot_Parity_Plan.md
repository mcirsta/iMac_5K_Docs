# Apple 5K Warm-Reboot Firmware Handoff Plan

Date: 2026-06-07

## Objective

Reach the same practical handoff quality as Windows:

- rebooting from the Linux 5K kernel must not stretch the Apple boot animation;
- the next macOS, Windows, or Linux boot must not inherit a wedged left tile;
- recovery must not require removing AC power;
- suspend, poweroff, cold boot, and non-Apple displays must remain unchanged.

This is an outcome-parity target. We should not assume Windows clears the
panel. It may instead preserve a coherent paired state that survives reset.

## What is already established

The existing RE is sufficient to start targeted kernel experiments.

1. The failure is not a userspace composition problem.
   - The Pro and iMac19,1 traces submit correct 2560x2880 per-tile geometry.
   - The left and right source rectangles are split at X=0 and X=2560.
   - Both stream encoders enable and the screenshots remain correct.
   - The physical panel can remain stretched through a warm reboot into macOS.

2. The Apple boot animation is already affected before the next OS display
   driver runs.
   - A next-boot Linux reset is too late to fix the firmware animation.
   - The primary fix must run during Linux shutdown, while AUX and both tile
     routes are still usable.

3. Linux already performs a Windows-shaped generic display teardown.
   - `amdgpu_pci_shutdown()` calls `amdgpu_device_prepare()` and then
     `amdgpu_device_suspend()`.
   - `dm_suspend()` calls `drm_atomic_helper_suspend()`, whose
     `drm_atomic_helper_disable_all()` commit removes every active CRTC,
     connector, and plane before DC D3.
   - DCE resets removed pipes in ascending hardware-pipe order.
   - DP 1.x SST disables the link before the stream; the root eDP path takes
     the alternate stream-before-link branch.
   - Both paths normally send DPCD `0x600 = 2`.
   - The Apple additions assert `0x4F1 = 1`, but there is no Apple tile-pair
     reboot handoff.

4. The Windows generic power-down path is now mapped and closely matches
   Linux.
   - non-D0 `WindowsDM_SetPowerState_ManageDcSeamlessFlags()` destroys each
     display DC stream and commits an empty DC context;
   - DCE110 resets removed pipes in ascending hardware-pipe order `0..5`;
   - its DPMS-off signal branches and DPCD `0x600 = 2` behavior match the
     corresponding Linux DCE code;
   - the receiver-D3 skip flag is only set for three legacy active-converter
     OUIs, not the Apple AE25/AE26 or AE1D/AE1E tiled panels;
   - no `0x4F1 = 0` operation appears in the direct SetPowerState-to-link
     teardown chain.

5. Static ordering is pipe-based, not connector-role based.
   - Windows proves ascending hardware-pipe order.
   - The Linux captures place the normal root path on pipe/CRTC 0 and the
     sibling on pipe/CRTC 1, making root-first the likely consequence for the
     captured allocation.
   - Static RE does not prove a fixed root-first rule if allocation changes.

6. Apple CoreEG2 firmware provides the competing neutralization model.
   - It writes root `0x4F1 = 1`, waits 10 ms, and validates the sibling.
   - On paired-setup failure and object cleanup it writes root `0x4F1 = 0`.
   - Therefore `0x4F1 = 0` is a valid firmware-shaped experiment, but it is not
     yet proven Windows behavior.

## Machine-specific interpretation

### iMac19,1

This is the clean acceptance machine for the boot-animation goal because its
working 5K path and firmware replay behavior are already established.

### iMacPro1,1

The Pro firmware is reported to show a stretched image even after a clean cold
boot, while macOS can establish a correct tiled state. Therefore:

- first test whether Linux destroys a correct state inherited from macOS or
  Windows;
- use "macOS remains correct after Linux" as the decisive Pro acceptance test;
- do not require the cold Pro firmware logo to become correct solely from this
  shutdown fix unless Windows proves that it can leave a reusable good state.

The warm-reboot failure and the Pro runtime failure may share a panel/TCON
state-machine cause, but they are not yet proven to be identical.

## Earliest useful hook

The first implementation point is instrumentation inside the existing amdgpu
shutdown and atomic-disable path:

```text
amdgpu_pci_shutdown()
  -> amdgpu_device_prepare()
  -> amdgpu_device_suspend()
  -> dm_suspend()
  -> drm_atomic_helper_suspend()
  -> drm_atomic_helper_disable_all()
  -> dce110_reset_hw_ctx_wrap()
  -> link_set_dpms_off()
  -> dp_disable_link_phy()
```

Current source anchors:

- `linux-imac-5k/drivers/gpu/drm/amd/amdgpu/amdgpu_drv.c:2588`
- `linux-imac-5k/drivers/gpu/drm/amd/display/amdgpu_dm/amdgpu_dm.c:3215`
- `linux-imac-5k/drivers/gpu/drm/amd/display/dc/link/link_dpms.c:2333`
- `linux-imac-5k/drivers/gpu/drm/amd/display/dc/link/protocols/link_dp_phy.c:71`

The logging must prove that both active streams reach the zero-stream commit,
that both removed pipes are processed, and that both DPCD `0x600` writes
complete before reset. Add an explicit pair helper before normal suspend only
if those logs show the generic path is absent, aborted, or too late. Do not
duplicate a teardown that already completed.

## Safety gate

Every experimental action must require all of:

- `system_state == SYSTEM_RESTART`;
- exact Apple internal-panel DMI and panel fingerprints;
- two real links joined by `link->tiled_peer`;
- one 2x1 group with two 2560x2880 streams;
- both AUX routes previously proven writable;
- no GPU reset, suspend, hibernate, kexec, or ordinary DP connector;
- an explicit experimental mode parameter.

The operation must be idempotent and time bounded. A failed AUX read or write
must fall back to the normal shutdown path.

Suggested temporary parameter:

```text
amdgpu.imac5k_reboot_handoff=off|log|preserve_rx|paired_quiesce|firmware_reset
```

Default remains `off` until one branch passes the full matrix.

## Phase 0: Reproducible baseline

Run each case with the same kernel and record a phone video beginning before
the reboot command and ending after the Apple animation:

| Case | Transition |
|---|---|
| A | AC removal, cold boot |
| B | Windows reaches correct 5K, warm reboot |
| C | Linux reaches 5K, warm reboot into the same kernel |
| D | Linux shutdown, boot again without AC removal |
| E | Linux shutdown, AC removal, then boot |
| F | macOS correct 5K, Linux, warm reboot back to macOS |

For each case record:

- Apple animation: correct, left stretched, right missing, or other;
- first bootloader image;
- fbcon image before the compositor;
- desktop result;
- whether the next macOS display is correct;
- whether AC removal changes the result.

This matrix must be run once with the current kernel and again for each single
experimental mode.

## Phase 1: Persistent shutdown instrumentation

Normal dmesg is lost at reboot. Enable `pstore/ramoops` or an equivalent
persistent sink before interpreting any shutdown experiment.

### Snapshot both links

At entry and after every handoff step, log:

- DPCD `0x100..0x10A`;
- DPCD `0x111`;
- DPCD `0x202..0x207`;
- DPCD `0x300..0x317`;
- DPCD `0x4F0..0x4F2`;
- DPCD receiver power `0x600`;
- AUX transaction result and readback;
- link index, connector signal, sink identity, and tile position;
- `cur_link_settings` and lane count/rate;
- `dp_keep_receiver_powered` and implicit eDP power flags.

### Snapshot source state

For both tile paths log:

- stream and link enable state;
- `DP_VID_STREAM_CNTL`;
- DIG BE to FE source selection;
- OTG/TG enable and timing registers;
- timing-sync group and master assignment;
- source 5/4 on Pro or source 2/3 on iMac19,1;
- the inter-path source-signal registers identified by the Pro RE;
- the exact order in which each link is blanked, disabled, put in D3, and
  removed from DC state.

Add explicit markers at:

```text
reboot-handoff entry
before atomic suspend
before stream blank
after stream blank
before/after link disable
before/after DPCD 0x600
before DC D3
reboot-handoff exit
```

Do not infer a DPCD write from the requested value. Log status and readback.

## Phase 2: Single-variable experiments

### Experiment P0: Force completion of the existing zero-stream teardown

Run this only if persistent logs show that the ordinary atomic-disable commit
does not finish on reboot.

- issue one explicit, pair-gated zero-stream commit while both streams and AUX
  routes are still available;
- wait for completion and record the resulting DC stream count;
- then enter normal `amdgpu_device_suspend()`;
- do not change DPCD `0x4F1` or receiver-power policy.

This tests shutdown timing/completion, not a different Windows sequence.

### Experiment P1: Preserve receiver D0

This remains a useful diagnostic A/B, but it is not Windows parity: Windows
normally sends DPCD `0x600 = 2` to these Apple panel links.

For the exact Apple pair and reboot only:

- set the equivalent of `dp_keep_receiver_powered` on both links;
- allow normal source blank and source-output disable;
- do not send DPCD `0x600 = 2`;
- do not change `0x4F1`;
- record both links immediately before DC D3.

Interpretation:

- good next Apple animation means receiver D3 creates the damaging residual
  state;
- no change means either source disable ordering matters or the panel requires
  explicit neutralization.

### Experiment P2: Pair-aware quiesce

Run if the generic teardown completes on both links but the symptom persists.

- collect both tile streams before normal atomic suspend;
- blank both while timing synchronization is still intact;
- disable the two child paths in one deterministic transaction;
- initially use ascending hardware-pipe order, matching the proven Windows
  DCE110 rule;
- preserve normal DPCD `0x600 = 2` behavior unless testing P1 separately;
- release the pair to normal amdgpu suspend only after both sides reach the
  same stage.

This tests whether Linux wedges the TCON by independently and unevenly
tearing down the two streams.

### Experiment N1: Firmware-shaped neutralization

Run separately from P1 and P2.

While both AUX routes are still available:

1. snapshot both links;
2. blank the pair coherently;
3. write root DPCD `0x4F1 = 0`;
4. read it back;
5. wait 10 ms;
6. snapshot the sibling and receiver-power state;
7. continue through normal paired shutdown.

Interpretation:

- good next animation means firmware expects a neutral root latch before it
  rebuilds paired replay state;
- failure does not disprove cleanup, because CoreEG2 may also clear grouped
  replay or mailbox state in a specific order.

This branch must be described as CoreEG2-shaped, not Windows-shaped.

### Experiment N2: Exact firmware cleanup

Only pursue this if N1 is the only branch that changes the symptom.

Reverse the complete CoreEG2/GOP destroy order:

- root/sibling order;
- grouped `0xF00` state;
- replay endpoint invalidation;
- `0x4F1 = 0` placement;
- receiver D3 placement;
- delays and readiness checks;
- MMIO `0x1223C` readiness effects.

Do not implement this broader sequence from inference alone.

## Phase 3: Focused Windows RE

This phase is complete for the generic DC/link teardown.

- `DriverEntry` at `0x1400C1840` builds the WDDM initialization table.
- SetPowerState resolves through `Dispatcher_SetPowerState()` at
  `0x140034130`, `Dispatcher_SetPowerStateCore()` at `0x140028AB0`, and
  `Dispatcher_ApplyAdapterPowerTransition()` at `0x140027560`.
- The non-D0 DAL path destroys each display stream, commits zero streams, and
  applies the empty context through `Dce110ApplyContextToHardware()`.
- `Dce110ResetRemovedPipes()` at `0x141CEB430` processes removed pipes in
  ascending hardware index and calls `LinkSetDpmsOffForPipe()` before CRTC
  blank/disable and pipe power-gating.
- `DisableDpLinkMaybeD3PowerOff()` at `0x141BB85F0` normally writes DPCD
  `0x600 = 2`.
- `SetSkipPowerOffForBranchSinkOui()` at `0x141B85200` only recognizes branch
  OUIs `0010FA`, `0080E1`, and `00E04C`. This is the same active-converter
  workaround as Linux `dp_keep_receiver_powered`.
- The exact mapped DC/link path contains no panel-private `0x4F1` clear.

Deliverable:

```text
DxgkDdiSetPowerState
  -> WindowsDM non-D0 stream destruction
  -> zero-stream DC commit
  -> ascending removed-pipe reset
  -> DPMS-off signal-specific ordering
  -> DPCD 0x600 = D3 for Apple panel links
  -> no direct 0x4F1 cleanup
```

Remaining Windows RE should be conditional. Pursue higher DAL topology object
notifications or panel-private grouped state only after persistent Linux logs
prove that the shared generic teardown completed and still produced the
stretched next-boot animation.

## Phase 4: Firmware boundary capture

Kernel logs cannot show the state at the first Apple animation. Extend the
existing UEFI probe only if the shutdown A/B result remains ambiguous.

Capture as early as the available firmware execution point permits:

- GOP paired replay `live` and endpoint count;
- endpoint masks `0x100/0x200`;
- output banks `0/1`;
- root and sibling connector records;
- root/sibling readiness at MMIO `0x1223C`;
- DPCD `0x4F1`, `0x600`, and source-private range;
- whether firmware accepted or discarded the inherited paired state.

The probe is diagnostic. If firmware draws the initial logo before the probe
runs, it cannot be the production repair.

## Phase 5: macOS A/B for the Pro

If the Pro remains the main target, complete the existing macOS capture runbook:

1. AC removal and direct macOS boot in the correct state.
2. Capture AGDCDiagnose, IORegistry, display logs, and available GPU state.
3. Boot the Linux 5K kernel.
4. Warm reboot to macOS without removing AC.
5. Capture the same data in the wedged state.
6. Diff only state that can plausibly survive the OS transition.

Highest-value differences are:

- tile-group or split-source mode;
- timing-sync master/group state;
- receiver D0/D3;
- panel-private latch state;
- source-signal/inter-path programming;
- whether one endpoint remains logically active while the other is reset.

## Production implementation shape

After one experiment wins:

- replace the mode switch with one pair-aware helper;
- keep exact DMI, panel, route, and tile-group gating;
- execute only for restart unless evidence supports poweroff too;
- preserve normal suspend/hibernate behavior;
- retain concise persistent failure logging;
- make a failed handoff fall back to standard amdgpu shutdown;
- keep iMac19,1 and iMacPro1,1 route metadata separate;
- document whether the final behavior is Windows preservation parity or
  CoreEG2 neutralization.

Do not fold this work into the Pro source-signal experiment until each has an
independent result. Otherwise inherited panel state can be mistaken for a
runtime modeset fix.

## Decision tree

```text
Baseline shows zero-stream teardown missing or incomplete
  -> run P0 explicit completion
  -> fix shutdown timing if P0 fixes the animation

Baseline proves both links fully tear down, P2 fixes animation
  -> implement coordinated pair shutdown
  -> retain normal receiver D3 unless a separate P1 matrix proves otherwise

P1 preserve receiver D0 fixes animation
  -> treat as a diagnostic panel-state result
  -> do not call it Windows parity
  -> investigate why Windows D3 leaves a coherent state while Linux D3 does not

N1 firmware reset fixes animation
  -> complete CoreEG2 cleanup-order RE
  -> implement the smallest proven neutralization sequence

No branch changes animation
  -> inspect UEFI inherited state
  -> reverse higher Windows DAL grouped-topology state
  -> compare macOS working vs wedged Pro state
  -> only then add a new experiment
```

## Acceptance criteria

For iMac19,1:

- 10 consecutive Linux warm reboots show a correct Apple animation;
- Linux to macOS and Linux to Windows do not inherit a stretched tile;
- Linux to Linux remains correct at fbcon and desktop;
- cold boot, poweroff, suspend/resume, and external DP remain unchanged.

For iMacPro1,1:

- a correct macOS- or Windows-established panel state is not destroyed by a
  Linux warm reboot;
- warm reboot back to macOS does not require AC removal to recover;
- the Linux runtime Pro experiment is assessed independently from handoff.

For both:

- no broad DPCD writes on unrelated hardware;
- no unbounded shutdown waits;
- persistent logs prove the selected branch ran on both tile links;
- the final sequence is reproducible from a clean AC-removed baseline.

## Immediate execution order

1. Add persistent shutdown logging and the strict pair detector.
2. Run the baseline predecessor matrix.
3. Confirm that atomic disable reaches both pipes and both DPCD `0x600` writes.
4. If it does not, implement and test P0.
5. If it does, test P2, P1, and N1 separately.
6. Perform deeper Windows grouped-state, firmware, or macOS RE only for the
   branch that changes the symptom.
7. Keep the Pro runtime modeset failure separate: a reboot cleanup cannot
   preserve a coherent pair if Linux never established one.

The focused Windows teardown call graph is now complete. The next highest-value
evidence is one persistent Linux shutdown trace proving whether the existing
Windows-shaped teardown actually completes on both Apple tile links.
