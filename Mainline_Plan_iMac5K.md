# Mainlining Plan: iMac 5K (dual-tile) AMDGPU/DC support

Status: draft plan. Working baseline = `clean_1` (kernel commit `8f9b866b1d9b`), where plain-boot Linux reaches native `5120x2880` on iMac19,1 with the AUX storm eliminated and no regressions.

This document is the plan to take the out-of-tree iMac 5K bring-up work and reshape it into a form that can be proposed for the mainline Linux kernel (amdgpu/DC).

---

## 1. Goal and scope

- **Goal:** upstream the ability to drive the Apple 27" 5K dual-tile internal panel as a genlocked `2560x2880 + 2560x2880` tile pair via amdgpu/DC, with no per-board hardcoding, no debug scaffolding, and a design AMD's display maintainers can accept.
- **In scope:** the *mechanism* (wake the tiled slave, keep it connected, train it, re-read the root EDID, validate the tile mode's bandwidth, prefer the tile mode, genlock the pair).
- **Explicitly out of scope for upstream:** the `force_*` module params, the verbose `IMAC5K:` / raw-EDID `DC_LOG_WARNING` tracing, and the DMI-string + hardcoded object-id gating. These are bring-up scaffolding and stay in a downstream/dev branch.

---

## 2. The core obstacle (read this first)

Almost all of the mechanism currently lives in **`drivers/gpu/drm/amd/display/dc/`** — AMD's **cross-OS shared Display Core**. AMD is highly protective of DC; external contributors essentially cannot land board-specific `dmi_match()` + hardcoded-object-id hacks in DC core. Any DC change must be:

1. driven by **panel/sink capability**, not by DMI strings or GPU connector object IDs, and
2. reviewed/blessed by the **AMD display team** (often via their internal review before it reaches `amd-gfx`).

The good news: **upstream already has the right extension point.** Panel-specific behavior is expressed through `struct dc_panel_patch` flags (`dc/dc_types.h`), set per panel by **`apply_edid_quirks()`** in `amdgpu_dm/amdgpu_dm_helpers.c`, keyed on the EDID `panel_id` (`drm_edid_encode_panel_id(mfg, product)`). DC core then gates behavior on `edid_caps.panel_patch.*` flags — this pattern is used all over DC today (e.g. `disable_fams`, `remove_sink_ext_caps`, `embedded_tiled_slave`). **Our work must flow through this mechanism.**

---

## 3. Current quirk inventory (what exists today, and where)

All of the following are gated on `dmi_match(DMI_PRODUCT_NAME, "iMac19,1")` plus hardcoded object IDs `0x3114` (root/eDP) / `0x3113` (slave/DP), DDC hw `3`/`2`, transmitters `UNIPHY_C`/`UNIPHY_D`. **All of that gating must be replaced (see §4).**

| # | File | Current quirk | Layer |
|---|------|---------------|-------|
| A | `amdgpu_dm/amdgpu_dm.c` | `amdgpu_dm_imac5k_reprobe_primary_after_secondary()` (Change A: forced root EDID re-read after slave is up); `amdgpu_dm_imac5k_make_tile_mode_preferred()`; `count_tile_modes`; `log_edid_raw`; route/tile helpers; call site in `amdgpu_dm_initialize_drm_device()` | amdgpu_dm (Linux-only) |
| B | `amdgpu_dm/amdgpu_dm_helpers.c` | secondary-route DMI helper; EDID/tile logging in `dm_helpers_read_local_edid()` | amdgpu_dm |
| C | `dc/link/protocols/link_dp_training.c` | `imac5k_lt_rewake_primary()`; pre-wake + re-wake-on-failure retry loop in `dpcd_set_link_settings()` | **DC core** |
| D | `dc/link/protocols/link_dp_capability.c` | `imac5k_cap_rewake_primary()`; early pre-wake in `dpcd_set_source_specific_data()`; `dpcd_set_imac5k_secondary_source_table_revision()` (source-DPCD `0x310`); reported-cap bridge in `dp_verify_link_cap_with_retries()` | **DC core** |
| E | `dc/link/protocols/link_edp_panel_control.c` | root `0x4F1` panel-latch / control quirk | **DC core** |
| F | `dc/link/link_detection.c` | root `0x4F1` post-edid wake; slave pre-detect AUX poll; override `dc_connection_none -> single`; force DP sink signal; `imac5k_find_primary_link()` | **DC core** |
| G | `dc/link/link_dpms.c` | stream-enable-time `0x4F1` latch / false-disconnect preservation | **DC core** |
| H | `dc/core/dc.c` | genlock: `imac5k_streams_syncable_ignoring_msa()` + force-primary-master in the timing-sync grouping | **DC core** |
| I | `amdgpu/amdgpu_drv.c` + `amdgpu.h` | `amdgpu_imac5k_force_genlock`, `amdgpu_imac5k_force_primary_master` module params | amdgpu |

Reality check: **most of the mechanism (C–H) is in DC core.** That is the part that needs AMD buy-in and the most reshaping.

---

## 4. Target architecture (the reshape)

### 4.1 Identify by panel + role, never by DMI/object-id

- **Match the panel by EDID,** via a new `apply_edid_quirks()` case for the Apple 5K panel: `drm_edid_encode_panel_id('A','P','P', 0xAE25)` (root / 4K-compat identity) and `0xAE26` (tile identity). This auto-covers *every* iMac that ships this panel (15,1 / 17,1 / 18,3 / 19,1 …) once verified — and removes the per-board object-id pinning entirely.
- **Identify role by `connector_signal` + panel id, not object IDs:**
  - root/primary tile = the `SIGNAL_TYPE_EDP` link presenting the Apple root identity (`AE25`);
  - slave/secondary tile = the `SIGNAL_TYPE_DISPLAY_PORT` link presenting the Apple tile identity (`AE26`).
- DDC hw instance / transmitter checks (`hw3`/`hw2`, `UNIPHY_C`/`UNIPHY_D`) are **deleted** — they are board-specific and unnecessary once role is signal+panel-derived.

### 4.2 Express every behavior as a `dc_panel_patch` flag

Extend `struct dc_panel_patch` (reuse `embedded_tiled_slave` where it fits) with focused flags, set in `apply_edid_quirks()`, consumed in DC core. Proposed flags (names to be agreed with AMD):

| Flag (proposed) | Replaces current quirk | Consumed in |
|---|---|---|
| `tiled_slave_root_wake` | root `0x4F1` pulse before slave AUX (detect / source-dpcd / train) | link_detection.c, link_dp_capability.c, link_dp_training.c |
| `tiled_slave_keep_connected` | `dc_connection_none -> single` override when HPD low | link_detection.c |
| `tiled_slave_source_table_rev` | publish source-DPCD `0x310` | link_dp_capability.c |
| `tiled_use_reported_link_cap` | reported-cap bridge for mode validation | link_dp_capability.c |
| `tiled_root_force_edid_reread` | Change A forced root re-read after slave up | amdgpu_dm.c |
| `tiled_pair_genlock_ignore_msa` | genlock despite `ignore_msa_timing_param` | dc.c |
| `tiled_stream_enable_latch` | stream-enable `0x4F1` write | link_dpms.c |

The root link needs its own marker (e.g. a `tiled_root` panel-patch flag set on the `AE25` entry) so the slave-side code can find the root eDP link to pulse, without object IDs.

### 4.3 Keep Linux-only orchestration in `amdgpu_dm`

The forced re-detect/re-read (Change A) and the tile-mode-preferred policy are DRM/amdgpu_dm concerns and should stay in `amdgpu_dm/`, which is far more acceptable to touch than DC core.

### 4.4 Delete scaffolding

- Remove `amdgpu_imac5k_force_genlock` / `force_primary_master` module params — behavior becomes automatic via the panel-patch flag. (Make the master/anchor choice deterministic for the tiled pair.)
- Remove all `IMAC5K:` `DC_LOG_WARNING` tracing and raw-EDID dumps; convert anything genuinely useful to `drm_dbg_kms()` at most.
- De-duplicate: the route-check helpers are currently copy-pasted into multiple DC files. Upstream wants single shared helpers.

---

## 5. The hard sells (anticipate maintainer pushback)

These two are the riskiest because they *bypass* DC's own logic rather than fixing it:

1. **Reported-cap bridge** (`dp_verify_link_cap_with_retries` sets `verified_link_cap = reported_link_cap`). Overriding the verified (training-proven) cap with the reported cap is exactly the kind of safety bypass maintainers reject. **Preferred resolution:** make the slave link *actually verify/train* at HBR2x4 (the wake/retry work now makes training succeed — `clean_1` shows real HBR2x4), so the verified cap is genuinely correct and no bridge is needed. Investigate whether the bridge is still required once the wake is in place; if it is only needed on the first HPD-low detect, scope it tightly or eliminate it.
2. **`dc_connection_none -> single` detection override.** Forcing "connected" when HPD reads low is also a bypass. **Preferred resolution:** frame it as "this embedded tiled-slave panel has no usable HPD on the secondary link; treat as always-connected when the root panel is present" — a property of the panel, expressed via a panel-patch flag, mirroring how eDP/embedded panels are already treated as hardwired-connected.

If either cannot be turned into a clean capability-driven behavior, expect that specific patch to be the one that stalls.

---

## 6. Proposed patch series (bisectable, smallest-first)

Targeting `amd-gfx` (then `drm-misc`/`amdgpu` trees). Each patch must build and not regress other hardware.

1. **drm/amd/display: add Apple 5K tiled-panel quirk + dc_panel_patch flags (infrastructure).** Adds the `apply_edid_quirks()` `APP/AE25`/`AE26` cases and the new `dc_panel_patch` fields (defaults off). No behavior change yet. Pure plumbing — easiest to review.
2. **drm/amd/display: treat Apple tiled-slave panel as hardwired-connected.** The `tiled_slave_keep_connected` behavior (replaces the `none->single` override), capability-gated.
3. **drm/amd/display: wake Apple tiled panel via root latch before slave AUX.** The `0x4F1` root pulse used at detect / source-DPCD / link-training (consolidate the three pre-wake sites; `tiled_slave_root_wake`).
4. **drm/amd/display: publish source DPCD table revision for Apple tiled slave.** (`0x310`; may fold into #3.)
5. **drm/amd/display: ensure Apple tiled-slave link trains at full cap.** The real fix replacing the reported-cap bridge (see §5.1). If a bridge remnant is unavoidable, this is where it lives, tightly scoped.
6. **drm/amd/dm: re-read root EDID after tiled slave brings panel up.** Change A in amdgpu_dm (`tiled_root_force_edid_reread`).
7. **drm/amd/dm: prefer tile-native mode on Apple tiled panel.** `make_tile_mode_preferred` reshaped (consider whether it can be a generic "tiled connector: prefer the tile-native DTD over a same-EDID non-tile preferred mode").
8. **drm/amd/display: genlock the Apple tile pair despite MSA-ignore.** The dc.c timing-sync change (`tiled_pair_genlock_ignore_msa`); deterministic master selection (no module param).

Cover letter (`[PATCH 0/N]`) explains the panel, the dual-tile bandwidth reason, the topology (root eDP `AE25` + slave DP `AE26`, DisplayID tile blocks, `2560x2880` x2), and links the evidence.

---

## 7. Process and sequencing

**Phase 0 — refactor in-tree-for-us first (do regardless of upstream):**
Re-architect the current branch onto the panel-patch/EDID model (§4). This is valuable on its own: it generalizes to all 5K iMacs and shrinks the diff dramatically. Re-test on iMac19,1 (must still reach `clean_1`'s result) before going further.

**Phase 1 — RFC early, before polishing the full series:**
Post an **`[RFC]`** to `amd-gfx@lists.freedesktop.org` (Cc `dri-devel@lists.freedesktop.org`). Use `scripts/get_maintainer.pl` for the exact Cc list; expect AMD DC maintainers (Harry Wentland, Alex Deucher, Rodrigo Siqueira, et al.). Describe the problem + the panel-patch approach + attach a dmesg and EDID decode. Goal: get a steer on (a) whether they'll take the DC bits or want to own/rework them, and (b) the §5 hard sells. **Do not invest in a polished 8-patch series before this signal.**

**Phase 2 — full series + iterate:**
Send the series per maintainer feedback; iterate through review rounds. Realistic outcome spread:
- Most likely to land: the `apply_edid_quirks` entry + amdgpu_dm bits (#1, #2, #6, #7).
- Needs AMD design agreement: the DC-core wake/train/genlock bits (#3, #5, #8).
- At-risk: the cap-bridge remnant if it can't be made clean.

---

## 8. Pre-submission checklist

- [ ] `scripts/checkpatch.pl --strict` clean on every patch.
- [ ] Each patch builds standalone; series bisects cleanly; no `IMAC5K`/debug leftovers.
- [ ] No DMI strings, no `0x3113`/`0x3114`/DDC/transmitter literals in DC core.
- [ ] Behaves as a no-op on all non-Apple-5K panels (the EDID gate guarantees this — call it out in commit messages).
- [ ] `Signed-off-by` (DCO) on every patch; real name/email.
- [ ] Commit messages describe **observed hardware/DPCD/EDID behavior**, not "the Windows driver does X." Constants like DPCD `0x4F1`, `0x310`, EDID ids `AE25/AE26` are interface facts (fine); keep provenance clean — no copied code/tables from the BootCamp driver.
- [ ] Reference the DisplayID tile metadata and DRM tile-group properties that already exist (`drm_connector` tile fields) so reviewers see it integrates with existing tiled-display infra.
- [ ] Tested-by from the iMac19,1; ideally Tested-by from a second model (17,1/18,3) to prove the EDID-keyed generalization.

---

## 9. Open questions / needs before/while upstreaming

1. **Is the reported-cap bridge still needed once the wake lands?** `clean_1` shows the slave reaching real verified HBR2x4. If the bridge is only needed on a transient first detect, it may be deletable (best outcome for §5.1).
2. **Second-model confirmation.** Booting on a 17,1/18,3 (see the per-model analysis in `Linux_plain_boot_5K_requirements_from_RE.md`) to confirm the same `AE25/AE26` + tile-block behavior, which validates the EDID-keyed design and gives a second `Tested-by`. Navi (iMac20,x) needs a DCN genlock variant — likely a follow-up, not blocking.
3. **Genlock master selection** must be deterministic without the module param.
4. **The userspace micro-tile scanout storm** (`Micro tile mode 1 not supported for scanout`, seen in new_boot_8 / clean_1) is a separate Mesa/compositor modifier-negotiation issue on the tiled head — *not* part of this kernel series, but worth a heads-up to reviewers/users.

---

## 10. Milestones

- **M0:** Phase-0 panel-patch refactor done; iMac19,1 still reaches the `clean_1` result. (Out-of-tree, but upstream-shaped.)
- **M1:** RFC posted to `amd-gfx`; maintainer direction received.
- **M2:** Series v1 posted.
- **M3:** amdgpu_dm + `apply_edid_quirks` patches accepted.
- **M4:** DC-core patches accepted (or AMD-owned equivalent merged).
- **M5:** Second-model `Tested-by` + (optional) DCN/Navi genlock follow-up.
