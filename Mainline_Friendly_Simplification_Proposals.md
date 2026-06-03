# Mainline-friendly simplification proposals for Apple iMac 5K support

Status: discussion tracker. This assumes the current `mainline-prep` changes keep
working on iMac19,1 and are then tested on other Apple 5K machines such as
iMac15,1 and iMac17,1. The goal is to record the next round of simplifications
before preparing an RFC for `amd-staging-drm-next`.

## Goal

Make the iMac 5K support look like normal AMD Display Core panel handling:

- identify the panel once from EDID quirks;
- express behavior through `dc_panel_patch` flags;
- keep Apple-specific details isolated;
- avoid validation bypasses where a real readiness or capability fix can solve
  the underlying problem;
- avoid broad DC core changes unless they are generic enough to help other
  embedded or tiled panels.

The desired final shape is:

```text
Apple-specific identification:
  apply_edid_quirks()

Generic behavior flags:
  struct dc_panel_patch

Generic behavior:
  AUX readiness before link training
  tile-native preferred mode
  AUX-proven HPD-low embedded/tiled DP detection
  post-peer EDID refresh

Small Apple-specific primitive:
  root panel-latch DPCD 0x4F1 pulse
```

## Principles

1. Keep Apple panel identification local.

   `APP/AE25`, `APP/AE26`, iMac model names, and the role split between eDP
   root and DP slave should ideally live only in `apply_edid_quirks()` and
   commit messages.

2. Let call sites consume generic flags.

   Link training, detection, DPMS, and mode code should mostly see flags such as
   `aux_ready_before_link_training`, `tiled_slave_root_wake`,
   `tiled_root_force_edid_reread`, or `tiled_stream_enable_latch`, not Apple
   product IDs.

3. Prefer observed readiness over sleeps.

   When the panel needs time to wake, use a bounded DPCD/AUX proof such as
   `DP_SET_POWER` plus `DP_DPCD_REV`, with a small bounded retry count and
   cooldown. Avoid random settle delays.

4. Fix validation inputs before overriding validation results.

   Maintainers are more likely to accept code that makes DC see the real link
   state, real mode, and real tile metadata than code that says "allow this
   mode anyway."

5. Minimize changes in shared DC core.

   Linux-only orchestration belongs in `amdgpu_dm` when possible. DC core changes
   should be small, flag-driven, and framed as generic panel behavior.

## Proposed simplifications

### 1. Remove the slave tile mode-validation bypass if possible

Current concern:

The slave tile path contains a special allowance for the native `2560x2880`
mode even if normal DC stream validation fails.

Mainline-friendly direction:

If the current AUX readiness and link-cap behavior now makes the native tile
mode validate normally, delete the bypass. This would be a major cleanup because
it turns the story from "Apple mode is allowed despite validation" into "the
panel is prepared correctly, so normal validation succeeds."

Validation needed:

- cold boot on iMac19,1;
- suspend/resume on iMac19,1;
- plain boot on at least one other Apple 5K model;
- confirm no `No DP link bandwidth`, no `Pruned mode`, and no blank-tile fallback
  with the bypass removed.

### 2. Make tile-native preferred mode generic

Current concern:

The current logic is Apple-shaped: prefer `2560x2880` for this panel.

Mainline-friendly direction:

Prefer the mode matching DRM tile metadata:

```text
if connector has tile metadata:
  prefer modes matching tile_h_size x tile_v_size
```

This is easier to justify because it follows the DisplayID tile description
rather than an Apple-specific resolution. Apple 5K would just be the panel that
needs this policy to avoid userspace picking the 4K compatibility mode.

Open question:

Check whether this could affect unrelated tiled displays that intentionally
advertise a non-tile preferred mode. If that risk exists, gate the generic policy
behind a panel patch flag such as `prefer_tile_native_mode`.

### 3. Hide Apple root/slave helpers behind generic names

Current concern:

Helpers like `dc_link_is_apple_5k_root()` and
`dc_link_is_apple_5k_slave()` are practical but make Apple-specific logic more
visible in shared headers.

Mainline-friendly direction:

Rename or reshape call-site helpers around panel-patch semantics, for example:

```text
dc_link_has_tiled_root_panel_patch()
dc_link_has_tiled_slave_panel_patch()
dc_link_needs_pre_training_aux_ready()
```

The EDID quirk would remain Apple-specific; the rest of the code would read like
generic tiled-panel support.

### 4. Reconsider the persistent `tiled_peer` pointer

Current concern:

`struct dc_link *tiled_peer` solves a real ordering problem: the DP slave needs
root-derived panel knowledge before the slave has its own sink. But adding a
persistent peer pointer to `dc_link` is a visible structural change.

Mainline-friendly options:

- keep `tiled_peer`, but document why the pre-detect path needs it;
- narrow the concept to a root-wake peer pointer/state instead of a broad peer;
- try a second detect pass after root EDID parsing, avoiding persistent peer
  state if the boot order allows it;
- use DRM tile-group metadata if it becomes available early enough, though it is
  probably too late for slave pre-detect.

Current bias:

Keep the mechanism unless testing proves a simpler ordering works. It solves the
hard part cleanly, but it needs a careful explanation in the cover letter.

### 5. Keep `aux_ready_before_link_training` as a generic quirk

Current strength:

This is one of the most mainline-friendly pieces. It avoids fixed delays and
instead proves the sink is responsive before training.

Preferred shape:

```text
if panel_patch.aux_ready_before_link_training:
  panel-specific wake hook if needed
  write DP_SET_POWER = D0
  read DP_DPCD_REV
  retry a small bounded number of times with a small cooldown
```

Review framing:

Some embedded DP sinks can expose EDID/DPCD during detect but not be consistently
ready for link-training writes later. The quirk makes training conditional on an
observed AUX response, not on a blind delay.

### 6. Keep the Apple 0x4F1 latch tiny and isolated

Current reality:

The root panel latch at DPCD `0x4F1` is hardware-specific, so some Apple-specific
code probably has to remain.

Mainline-friendly direction:

Keep it as a small helper, used only by generic panel-patch behavior:

```text
link_apple_5k_root_panel_latch_pulse(root_link)
```

Avoid spreading `0x4F1`, panel IDs, object IDs, DDC instances, or transmitter
IDs through detection, DPMS, or link-training code.

### 7. Remove local planning-doc references from kernel comments

Current concern:

Internal references such as `iMac_5K_Docs/Mainline_Plan_iMac5K.md` are useful in
the development tree but should not appear in final kernel comments.

Mainline-friendly direction:

- move evidence and history into commit messages and the cover letter;
- keep code comments short and self-contained;
- use public `Link:` tags only if we have a public report, mailing-list thread,
  or public test artifact.

### 8. Keep source-DPCD retry accounting generic

Current strength:

The Apple-specific DPCD retry cleanup moves us away from "retry because Apple"
and toward "retry only for a panel that first requested AUX readiness."

Mainline-friendly direction:

Keep retry counts bounded and small. Make every retry tied to an explicit
readiness probe or a failed write that is safe to repeat. Do not loop tightly,
and do not add unbounded fallback behavior.

## Candidate RFC patch shape after simplification

This is the shape to aim for after the cleanup pass, not necessarily the current
branch's commit layout:

```text
[RFC PATCH 0/N] drm/amd/display: support Apple iMac 5K tiled panel

1. drm/amd/display: add panel-patch quirks for Apple 5K tiled panels
2. drm/amd/display: wire tiled panel peers for early slave AUX handling
3. drm/amd/display: detect HPD-low tiled DP slave using AUX proof
4. drm/amd/display: re-read tiled root EDID after slave wake
5. drm/amd/display: prefer tile-native modes for patched tiled panels
6. drm/amd/display: apply Apple 5K slave source-DPCD and stream latch quirks
7. drm/amd/display: prepare AUX before DP link training for patched panels
```

If the mode-validation bypass is removed and the tile-native preference becomes
generic enough, the series gets much easier to defend.

## Validation checklist before RFC

- iMac19,1 cold boot reaches native `5120x2880`.
- iMac19,1 suspend/resume keeps or recovers native `5120x2880`.
- At least one other Apple 5K model, ideally iMac17,1 or iMac15,1, boots with
  both tiles present.
- No AUX retry storm.
- No `No DP link bandwidth` or native tile-mode pruning.
- Slave link-training failure, if still logged transiently, does not prevent the
  final native tiled mode.
- No DMI, object-id, DDC-instance, or transmitter literals in final DC core
  behavior.
- No local planning-doc references in code comments.
- Retry loops are bounded, have cooldowns, and are gated by panel quirks.

## Current preferred direction

The most valuable cleanup before RFC is to remove the mode-validation bypass, if
testing confirms the readiness fixes make it unnecessary.

The most valuable thing to keep is the observed AUX readiness helper. It is
bounded, generic, and easier to review than blind delays or luck-based retries.
