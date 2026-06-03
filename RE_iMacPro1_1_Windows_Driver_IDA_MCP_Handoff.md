# iMacPro1,1 Windows Driver RE Handoff

Created: 2026-06-03

This document is a handoff for an AI agent doing the next reverse-engineering
pass for the iMacPro1,1 internal 5K panel failure. The primary target is the
Windows AMD/BootCamp display driver in IDA Pro, accessed through MCP.

The goal is not to rediscover the generic iMac 5K model. The goal is to explain
why iMacPro1,1 still shows a stretched left tile even after Linux detects both
tiles and feeds them correctly.

## Current Working Theory

iMacPro1,1 belongs to the same dual-stream internal 5K family as the pre-2020
27-inch 5K iMacs, but it uses Vega/Japura rather than the Polaris iMac19,1 path.

The Linux branch already supports the iMacPro1,1 panel IDs:

- Commit `67b707d88`: `drm/amd/display: add iMacPro1,1 Apple 5K panel IDs`
- Root/compat EDID product: `APP/AE1D`
- Tiled identity / post-wake product: `APP/AE1E`
- GPU identity from logs: `1002:6860`, subsystem `106b:017c`
- Board: `Mac-7BA5B2D9E42DDD94`
- VBIOS part: `113-D05001A1XT-018`
- DC/DCE: DC `3.2.369`, `DCE 12.0`

Important: do not treat "iMacPro1,1 is in the table" as proof that the panel
enable sequence is complete. The logs show ID detection works; the visible
failure appears to be later, in panel/tile-controller state.

## Evidence From Linux Logs

Use these captures first:

- `../Pro1.1/dmesg-sanitized.txt`
- `../Pro1.1/imacpro-2026-06-03-13-22/`
- `../Pro1.1/dmesg-apple5k.txt`
- `../17.1/dmesg-sanitized.txt`

The `../17.1/dmesg-sanitized.txt` path is misleading: the DMI inside that file
is iMacPro1,1. Always trust DMI/log contents over directory names.

### What Worked

The branch recognizes iMacPro1,1 and its panel IDs:

- Initial eDP/root read matches `APP/AE1D`.
- The DP/slave side wakes and reads `APP/AE1E`.
- The root eDP EDID re-read changes from `AE1D` to `AE1E`.
- Both connectors report tiled metadata after the wake/re-read sequence:
  - root/eDP tile location `0,0`
  - DP tile location `1,0`
  - grid `2x1`
  - tile size `2560x2880`

The latest bad Pro1,1 capture also proves userspace/KMS layout is not the
remaining problem:

- `drm-state-bad.txt` was correctly captured from amdgpu at debugfs `dri/1`.
  `dri/0` was simpledrm in that boot.
- Two CRTCs were active:
  - `eDP-1` on `crtc-0`
  - `DP-1` on `crtc-1`
- GNOME allocated two separate `2560x2880` primary framebuffers.
- The kernel plane trace showed each tile fed from the correct source/dest
  rectangle.

Therefore, when the user says "left tile stretched across screen", do not chase
Mutter/KWin first. The Linux display engine is feeding two correct tile streams.
The remaining fault is likely panel-side mode/state.

### Latest Noisy-Branch Failure

The `../Pro1.1/dmesg-sanitized.txt` capture is a current noisy-branch test:

- DMI: `Apple Inc. iMacPro1,1/Mac-7BA5B2D9E42DDD94`
- Kernel: `7.0.1-imac5k-g86194e981b23`, build `#21`, built
  `Wed Jun 3 16:01:01 CEST 2026`
- Kernel command line includes `drm.debug=0x1e log_buf_len=32M`
- The branch marker from commit `5ca584fa8` is present:
  - `APPLE5K: stream-enable root latch 0x4F1 ... status=1 value=0x01 read_status=1 readback=0x01`
- The DP/slave latch also ran and read back:
  - `APPLE5K: stream-enable latch 0x4F1 link[1] status=1 value=0x01 read_status=1 readback=0x01`
- The user still reported the left tile stretched across the whole display.

This is an important negative result: simply re-pulsing root `0x4F1 = 1` at
stream enable, in addition to the existing slave stream latch, is not sufficient
for iMacPro1,1.

The same log still shows correct Linux-side tile/KMS state:

- Initial root panel match: `APP/AE1D`, panel ID `0x610AE1D`.
- Slave panel match: `APP/AE1E`, panel ID `0x610AE1E`.
- Root re-read changes from `AE1D` to `AE1E`.
- Root/eDP tile: grid `2x1`, location `0,0`, size `2560x2880`.
- DP/slave tile: grid `2x1`, location `1,0`, size `2560x2880`.
- `stream_count=2`, both streams are `2560x2880`, RGB, 10 bpc.
- Plane traces show the fbdev and compositor phases feeding each tile correctly.

One suspicious but not conclusive line appears during one DP-1 enable attempt:

- `enabling link 1 failed: 15`

In Linux DC this status is `DC_FAIL_DP_LINK_TRAINING`. Later commits still show
two active streams, so do not treat this alone as the whole explanation. Carry it
into RE as a clue about retry/order/timing around DP link training.

### What Failed Or Remains Unproven

The Windows-driver-derived source-DPCD experiment did not fix iMacPro1,1:

- Current branch forced source DPCD `0x310` to `04 1d 03`.
- Logs show this did run:
  - `source DPCD 0x310 ... value=04 1d 03 ... force_apple5k_legacy=1`
- The panel was still reported bad after this kernel.

The DP/slave stream latch is alive but insufficient by itself:

- Linux writes `0x4F1 = 1` on the DP/slave link at stream enable.
- The DP/slave readback is `0x01`.
- The visible failure still occurs.

The readback for source DPCD `0x310` is not useful yet:

- Linux sees write status success, but readback `00 00 00`.
- Treat `0x310` as possibly write-only or side-effect-only until the Windows
  driver proves otherwise.

The newest noisy-branch experiment failed on Pro1,1:

- Commit `5ca584fa8`: `drm/amd/display: re-pulse Apple 5K root latch at stream enable`
- Observed log marker:
  - `APPLE5K: stream-enable root latch 0x4F1 ...`
- The write status and readback were both good, but the user still reported the
  stretched-left-tile failure.
- Do not propose this same simple root-latch-at-stream-enable patch again unless
  Windows RE shows a different payload, pulse order, delay, or link object.

## Current Linux Sequence To Compare Against Windows

The current Linux branch does roughly this:

1. Detect root/eDP `AE1D`.
2. Wire `tiled_peer` between root link 0 and DP link 1.
3. Pulse root `0x4F1 = 1` after root EDID.
4. During slave pre-detect, pulse root `0x4F1 = 1`.
5. Poll slave AUX/DPCD revision.
6. Read slave EDID `AE1E`.
7. Re-read root EDID; root changes from `AE1D` to `AE1E`.
8. Force source DPCD `0x310 = 04 1d 03` on the slave.
9. Pulse root `0x4F1 = 1` before/around training.
10. Train both tile links.
11. At stream enable, write slave `0x4F1 = 1` and read it back.
12. After commit `5ca584fa8`, also pulse/read root `0x4F1` at stream enable.

Steps 11 and 12 are now confirmed to run on Pro1,1 and are not enough to fix the
visible stretched-tile failure.

The Windows RE task is to find what this sequence is missing or doing in the
wrong order for Vega/Japura.

## Primary RE Questions

Answer these with code evidence from the Windows driver. Avoid hunches.

1. Does the Windows driver write source DPCD `0x310` as `05 1d 03`,
   `04 1d 03`, or both on iMacPro1,1 / Vega / DCE 12?
2. If both are used, what decides the value and what is the exact retry/order?
3. Does Windows write `0x4F1` on the root link, the slave link, or both?
4. Does Windows write `0x4F1 = 0` before `1`, or pulse `1 -> 0 -> 1`?
5. Does Windows repeat `0x4F1` after link training or immediately before stream
   unblank?
6. Are there required delays after `0x4F1`, after `0x310`, or between enabling
   the two tile streams?
7. Does Windows write any additional private DPCD address near this sequence
   (`0x316`, `0x10A`, `0x111`, `0x205`, or Apple/vendor ranges)?
8. Does Windows set ASSR / DP eDP config `0x10A` differently for the root and
   slave routes?
9. Does Windows treat iMacPro1,1/Japura differently from iMac19,1/Polaris once
   both tile EDIDs are visible?
10. Is the stream-enable order root-first, slave-first, or synchronized/grouped?
11. Is there a grouped-display/tile-controller state structure that must be
    committed before the planes/streams go live?
12. Does Windows use a special two-stream timing or timing-sync command even
    though each stream is nominally `2560x2880 @ 59.98`?

## IDA Pro + MCP Workflow

Use IDA MCP as a mechanical assistant. Keep a written trail of every rename,
constant, xref, and decompiled block that supports a conclusion.

### 1. Locate The DPCD/AUX Transfer Layer

Search constants and xrefs:

- `0x4F1`
- decimal `1265`
- `0x310`
- `0x316`
- `0x10A`
- `0x100`, `0x101`, `0x102`, `0x103`, `0x107`, `0x108`, `0x202`, `0x205`

Expected outcome:

- Identify the wrapper that performs DPCD/AUX reads.
- Identify the wrapper that performs DPCD/AUX writes.
- Rename them in IDA.
- Document argument order: link/connector object, DPCD address, buffer, length,
  transaction/result.

Do not stop at a constant hit. Follow xrefs up to the caller that knows the
display/link role.

### 2. Find Source-DPCD Table Revision Logic

Search byte/constant patterns:

- `05 1d 03`
- `04 1d 03`
- DPCD address `0x310`
- separate constants `0x1d` and `0x03` only after narrowing to DPCD callers

Expected outcome:

- Identify exactly where the Windows driver prepares source-DPCD bytes.
- Determine whether DCE version, ASIC family, panel ID, connector type, or
  firmware capability chooses byte 0.
- Record whether iMacPro1,1/Vega chooses `05` or `04`.

This is a critical question because Linux originally used a DCE heuristic
(`DCE_VERSION_12_0 ? 0x05 : 0x04`), then the noisy branch forced `0x04`. We need
Windows evidence for the correct iMacPro1,1 behavior.

### 3. Trace Apple Panel Latch `0x4F1`

Search `0x4F1` and decimal `1265`.

For every writer, document:

- Which link/connector object it writes through.
- Payload value.
- Whether it reads back.
- Whether it writes `0` on failure or shutdown.
- Delay after write.
- The caller phase:
  - EDID detect
  - slave AUX wake
  - source-DPCD setup
  - link training
  - stream enable/unblank
  - modeset disable

Important comparison:

- Linux currently writes root `0x4F1` during detect/training and slave `0x4F1`
  at stream enable.
- Commit `5ca584fa8` adds root `0x4F1` again at stream enable.
- If Windows does something else, that is likely the next patch target.

### 4. Map iMacPro1,1/Japura Link Objects

Do not assume the iMac19,1 DDC/encoder mapping is identical.

From Pro1,1 Linux logs:

- root link 0:
  - signal eDP
  - link id `3:20:1`
  - `ddc_hw=1`
  - `hpd_src=1`
  - encoder `5`
  - engine `5`
- slave link 1:
  - signal DP
  - link id `3:19:1`
  - `ddc_hw=0`
  - `hpd_src=0`
  - encoder `4`
  - engine `4`

Use Windows RE to determine the matching Windows connector/link objects for
Japura. If object IDs `0x3114` and `0x3113` still apply, confirm by xref or
data structure, not by analogy.

### 5. Find Panel-ID Or Apple-Tiled Branching

Search for EDID/product handling:

- `AE1D`
- `AE1E`
- little-endian bytes `1d ae` / `1e ae`
- APP manufacturer bytes as they appear in EDID/DAL code
- DisplayID tile block parsing paths
- strings or table entries for `iMac`, `Japura`, `Apple`, `APP`

Expected outcome:

- Determine whether Windows has a table for Apple 5K panels.
- Determine whether AE1D/AE1E share a generic Apple-tiled path or use a special
  iMacPro1,1 branch.
- Record any panel-family flags and how they affect DPCD writes or stream order.

### 6. Compare Against A Known-Good Family

Compare the Windows path for:

- iMac19,1 / Polaris if available
- iMacPro1,1 / Vega/Japura

The question is not "are both dual-tile?" They are. The question is whether
Vega/Japura changes:

- source-DPCD `0x310` byte 0
- root/slave `0x4F1` order
- stream-enable order
- required delay
- ASSR / `0x10A`
- timing-sync/grouped-display commit

## Deliverables From The RE Agent

The RE pass should produce a short report with:

1. IDA database name, driver version, file hash if available.
2. Renamed DPCD/AUX read/write functions and their addresses.
3. Renamed Apple/tiled-panel functions and their addresses.
4. A constant table for all relevant DPCD addresses and magic values.
5. A chronological Windows sequence for iMacPro1,1 from detect to stream
   unblank.
6. A comparison table: Windows sequence vs current Linux sequence.
7. A list of exact Linux patch candidates, ordered by evidence strength.
8. Any uncertainties that require runtime logging rather than guessing.

Preferred output style:

```text
Phase: source-DPCD setup
Windows evidence: function_name+0xNN writes 0x310 bytes XX XX XX via link object field +0x...
Linux current: writes 04 1d 03 in link_dp_capability.c
Conclusion: ...
Patch candidate: ...
```

## Guardrails

Do not chase these unless new evidence appears:

- Do not treat the Pro1,1 latest failure as a GNOME/Mutter layout bug. The
  debugfs state shows two correct tile framebuffers.
- Do not assume debugfs `dri/0` is amdgpu. In the Pro1,1 capture, simpledrm was
  minor 0 and amdgpu was minor 1.
- Do not use macOS IORegistry or OCLP as the primary source for Linux runtime
  DPCD order. They are useful for topology and boot-path context, but the
  actionable runtime sequence should come from the Windows AMD driver.
- Do not infer the source-DPCD first byte from DCE version without Windows
  evidence. That heuristic already failed to settle the Pro1,1 issue.
- Do not overinterpret source-DPCD `0x310` readback `00 00 00`; the AUX write
  can succeed while the register is not meaningful to read back.
- Do not assume folder names are machine IDs. Check DMI and GPU identity inside
  each log.

## Current Branch State To Know

Noisy test branch:

```text
imac15-1-test-logging
5ca584fa8 drm/amd/display: re-pulse Apple 5K root latch at stream enable
799104e95 drm/amd/display: trace Apple 5K tile plane state
a9ec042c3 drm/amd/display: avoid unexported DRM colorspace helper
248220e27 drm/amd/display: trace Apple 5K stream color state
d02fe6019 drm/amd/display: add iMac18,3 Apple 5K panel IDs
1636834ae drm/amd/display: add iMac20,x Apple 5K panel IDs
bc7ea5043 drm/amd/display: add iMac17,1 Apple 5K panel IDs
67b707d88 drm/amd/display: add iMacPro1,1 Apple 5K panel IDs
```

If a new Pro1,1 capture does not contain:

```text
APPLE5K: stream-enable root latch 0x4F1
```

then it did not test the latest stream-enable root-latch patch.

## Success Criteria

The RE effort succeeds when it can explain, with Windows-driver evidence, at
least one of these:

- Linux is using the wrong `0x310` bytes for iMacPro1,1.
- Linux writes `0x4F1` on the wrong link, at the wrong time, or with the wrong
  pulse value/order.
- Linux is missing another DPCD/private-register write before stream unblank.
- Linux enables the two tile streams in an order Windows avoids.
- Linux lacks a Vega/Japura-specific grouped/timing-sync step.

The final output should be precise enough to implement one small Linux patch
and one unambiguous log marker for the next iMacPro1,1 tester.
