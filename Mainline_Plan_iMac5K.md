# Mainlining Plan: iMac 5K (dual-tile) AMDGPU/DC support

Status: draft plan. Working baseline = `clean_1` (kernel commit `8f9b866b1d9b`), where plain-boot Linux reaches native `5120x2880` on iMac19,1 with the AUX storm eliminated and no regressions. Full evidence chain — including the observed primary `AE25 → AE26` EDID flip after the root-wake-driven re-read — is recorded in [Linux_plain_boot_5K_requirements_from_RE.md](Linux_plain_boot_5K_requirements_from_RE.md) (`new_boot_7 RESULT`); references throughout this doc are to that evidence, not to hypothesis.

**Phase 0 progress — M0 achieved + bonus refactors.** Boot 1 (`obs/0_1/dmesg.txt`) of the panel-patch refactor produced **zero `WARN_ON_ONCE` mismatches** across all 8 predicate sites, validating that the new EDID+`connector_signal`+`tiled_peer` gates are equivalent to the existing DMI+object-id+DDC+transmitter gates on iMac19,1. Boot 2 (`obs/0_2/dmesg.txt`) switched every consumer to the new gates, deleted DMI/object-id/DDC/transmitter literals from DC core, stripped `IMAC5K:` log spam, removed the `imac5k_force_genlock` / `imac5k_force_primary_master` module params, and re-gated genlock via the panel-patch flag — all in one tree. Zero WARN, zero `IMAC5K:` lines, full `clean_1` outcome. Boot 3 (`obs/0_3/dmesg.txt` + `dmesg_suspend.txt`) added four code-quality refactors — deduplicated the three root-panel-latch helpers into one shared `link_apple_5k_root_panel_latch_pulse()`, renamed the misleadingly-named `tiled_pair_genlock_ignore_msa` flag to `tiled_pair_force_sync_group`, consolidated the duplicate tile-dimension constants into `dc_types.h`, and **deleted the cap-bridge code entirely**. Cold boot reproduced cleanly, and a subsequent **suspend/resume cycle (S3, ~12s offline) also reproduced cleanly** with the cap-bridge gone, settling the §9.1 warm-path question empirically. Diff vs `clean_1`: **+314 / −947 (net −633 lines)**. All §7.4 acceptance criteria are met. The **`mainline-prep` branch at this point is M0**. Findings from each boot are interleaved through the doc below, tagged **[Boot 1]**, **[Boot 2]**, and **[Boot 3]**.

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

The good news: **upstream already has the right extension point.** Panel-specific behavior is expressed through `struct dc_panel_patch` flags (`dc/dc_types.h`), set per panel by **`apply_edid_quirks()`** in `amdgpu_dm/amdgpu_dm_helpers.c`, keyed on the EDID `panel_id` (`drm_edid_encode_panel_id(mfg, product)`). DC core then gates behavior on `edid_caps.panel_patch.*` flags — this pattern is used across DC today (e.g. `disable_fams`, `remove_sink_ext_caps`, `disable_colorimetry`, `wait_after_dpcd_poweroff_ms`). **Our work must flow through this mechanism.** (Note: `embedded_tiled_slave` is declared on the struct but has no current consumers — see §4.2.)

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

- **Match the panel-family by EDID,** via a new `apply_edid_quirks()` case that covers **both** identities the Apple 5K panel can present: `drm_edid_encode_panel_id('A','P','P', 0xAE25)` (4K-compat identity, observed on the primary pre-wake) **and** `drm_edid_encode_panel_id('A','P','P', 0xAE26)` (tile identity, observed on the secondary always, and on the primary after the root-wake forces a re-read — see `new_boot_7 RESULT`). Auto-covers every iMac that ships this panel (15,1 / 17,1 / 18,3 / 19,1 …) once verified — and removes per-board object-id pinning entirely.
- **Identify role by `connector_signal` alone, not by panel id.** Panel id cannot distinguish root from slave once the panel is in tile mode (the primary flips `AE25 → AE26` after the wake; the secondary is `AE26` from the start). Role rule:
  - root/primary tile = the `SIGNAL_TYPE_EDP` link in the panel-family;
  - slave/secondary tile = the `SIGNAL_TYPE_DISPLAY_PORT` link in the panel-family.
- DDC hw instance / transmitter checks (`hw3`/`hw2`, `UNIPHY_C`/`UNIPHY_D`) are **deleted** — they are board-specific and unnecessary once role is signal-derived.

### 4.2 Express every behavior as a `dc_panel_patch` flag

Extend `struct dc_panel_patch` ([dc_types.h:165](linux-imac-5k/drivers/gpu/drm/amd/display/dc/dc_types.h#L165)) with focused flags, set in `apply_edid_quirks()`, consumed in DC core. Proposed flags (names to be agreed with AMD):

| Flag (current) | Replaces current quirk | Consumed in |
|---|---|---|
| `tiled_slave_root_wake` | root `0x4F1` pulse before slave AUX (detect / source-dpcd / train) | link_detection.c, link_dp_capability.c, link_dp_training.c — **all three now call the shared `link_apple_5k_root_panel_latch_pulse()` helper in link_dpcd.c** ([Boot 3]) |
| `tiled_slave_keep_connected` | `dc_connection_none -> single` override when HPD low | link_detection.c |
| `tiled_slave_source_table_rev` | publish source-DPCD `0x310 = 04 1d 03` | link_dp_capability.c |
| ~~`tiled_use_reported_link_cap`~~ | ~~reported-cap bridge~~ | **DELETED in Boot 3** — bridge demonstrably dead code on iMac19,1 across 3 cold boots + 1 suspend/resume cycle (see §5.1 / §9.1) |
| `tiled_root_force_edid_reread` | Change A forced root re-read after slave up | amdgpu_dm.c |
| `tiled_pair_force_sync_group` (renamed) | force timing-sync grouping of the tile pair (pairing keys off `drm_tile_group`, see below) | dc.c |
| `tiled_stream_enable_latch` | stream-enable secondary `0x4F1 = 1` write at finalization | link_dpms.c |

**Note on the existing `embedded_tiled_slave` field** (declared at [dc_types.h:175](linux-imac-5k/drivers/gpu/drm/amd/display/dc/dc_types.h#L175)): grep shows **zero consumers anywhere in DC core or amdgpu_dm**. It is a declared-but-unused slot — there is no existing behaviour to preserve. The plan therefore treats it as available for repurposing under one of the proposed names above (most naturally `tiled_slave_keep_connected`), or for outright rename. Either way, "reuse where it fits" should not be read as preserving any current effect.

**How flags get set on the right side.** `apply_edid_quirks()` currently takes only `(drm_device, edid, edid_caps)` ([amdgpu_dm_helpers.c:87](linux-imac-5k/drivers/gpu/drm/amd/display/amdgpu_dm/amdgpu_dm_helpers.c#L87)), so it has no per-link signal context. To set root-only vs slave-only flags correctly, extend its signature (or its caller `dm_helpers_parse_edid_caps()`, which already has `dc_link *`) to pass `connector_signal`: root-side flags (`tiled_root_*`) gated on `SIGNAL_TYPE_EDP`, slave-side flags (`tiled_slave_*`) on `SIGNAL_TYPE_DISPLAY_PORT`. Pair-level flags (`tiled_pair_*`) can be set on either side, or both.

**How the slave-side DC code finds the root link to pulse.** DC core should not walk DRM connectors. Instead, `amdgpu_dm` resolves the tile pair once — at the point where `drm_connector->has_tile` becomes true on two endpoints with the same `tile_group->id` ([drm_connector.h:2410-2422](linux-imac-5k/include/drm/drm_connector.h#L2410-L2422)) — and stashes a peer `struct dc_link *` on each side (e.g. `link->tiled_peer`, set symmetrically). DC consumers read that pointer directly; no lookup, no object-id literals. This replaces the current `imac5k_find_primary_link()` helper.

**[Boot 1] Peer-wiring timing.** The pre-Boot-1 concern was that the slave's pre-detect AUX path might run *before* the root's EDID parse — leaving `tiled_peer` unwired and the slave-side new gate evaluating to false. Observed timing on iMac19,1: root eDP EDID parses at t=6.368840 and `tiled_peer` wires at t=6.368842 (the same `apply_edid_quirks` call sequence), while the slave's first AUX activity is at t=6.374633 (~6 ms later). The race didn't materialize on this hardware; root-first parse ordering held. Phase 0's earlier-firing of peer wiring (immediate on root EDID parse, rather than waiting for both `has_tile` flags) is what made this work — keep that ordering in the upstream patch shape.

**[Boot 1] First-DP-sibling heuristic validated.** Boot 1 log shows `APPLE5K: tiled_peer wired: root link[0] <-> dp link[1]` — the heuristic ("first DP sibling in `dc->links[]`") picked the right slave on iMac19,1, with no external DP connected. Refining to "DP sibling whose DisplayID tile-group matches the root's" is a Phase 1.5 hardening item; the current heuristic is sufficient for M0 and for the 15,1/17,1/18,3 second-model test (§9.2).

**On genlock specifically.** The *pairing* decision (which two streams sync together) keys off `drm_connector->tile_group->id` — already populated by DisplayID TILE parsing when `has_tile == true` (confirmed in `new_boot_7`). The panel-patch flag `tiled_pair_genlock_ignore_msa` only enables the *override* of `ignore_msa_timing_param` for this panel-family; the *grouping* uses existing DRM tile-group infrastructure. That separation is much easier to defend to AMD: "we already had the metadata, we just weren't using tile-group membership for sync grouping."

**[Boot 1] Genlock flag-name discrepancy.** Boot 1 log shows the timing-sync forming as `group_size=2 master=0x3114(eDP) sync_type=1 force_genlock=1 ignore_msa=0/0`. The `ignore_msa=0/0` part means neither stream has `ignore_msa_timing_param` set at sync time — so `imac5k_streams_syncable_ignoring_msa()` ([dc.c:1701](linux-imac-5k/drivers/gpu/drm/amd/display/dc/core/dc.c#L1701)), which gates on that flag, did *not* fire. The actual code path that formed the pair is the `amdgpu_imac5k_force_genlock` branch at [dc.c:1836](linux-imac-5k/drivers/gpu/drm/amd/display/dc/core/dc.c#L1836), which force-groups the tile pair regardless of MSA state. So the proposed panel-patch flag name `tiled_pair_genlock_ignore_msa` is misleading — it describes a path that wasn't exercised.

**[Boot 3] Resolved:** flag renamed `tiled_pair_genlock_ignore_msa` → `tiled_pair_force_sync_group` across `dc_types.h`, `apply_edid_quirks()`, and the dc.c consumer. Boot 3 confirms timing-sync still forms correctly with the new name (GSL: Set-up complete cycles fire on both cold boot and post-resume). The `imac5k_streams_syncable_ignoring_msa()` defensive branch is kept for now; whether it can also be deleted is a Phase 1.5 cleanup question on top of the already-confirmed force-group path.

### 4.3 Keep Linux-only orchestration in `amdgpu_dm`

The forced re-detect/re-read (Change A) and the tile-mode-preferred policy are DRM/amdgpu_dm concerns and should stay in `amdgpu_dm/`, which is far more acceptable to touch than DC core.

### 4.4 Delete scaffolding

- Remove `amdgpu_imac5k_force_genlock` / `force_primary_master` module params — behavior becomes automatic via the panel-patch flag. **Master/anchor rule (decided):** the `SIGNAL_TYPE_EDP` tile (root) is always the timing-sync master; the `SIGNAL_TYPE_DISPLAY_PORT` tile (slave) follows. Deterministic from topology, no module param, no per-board literal. **[Boot 1] update:** the log shows `force_primary_master=0` (module param off) yet DC's *default* pipe-ordering still picked `master=0x3114` (eDP root). **[Boot 2] confirmed:** with the entire `force_primary_master` code block **deleted from `dc.c`** (no replacement, no flag, nothing), Boot 2 again picks the eDP root as master — `eDP-1 on [CRTC:63:crtc-0]` (pipe 0) and `DP-1 on [CRTC:66:crtc-1]` (pipe 1), with GSL set-up completing across three timing-sync cycles. So encoding the eDP-master rule needs **zero new code** — it is implicit in DC's existing pipe-ordering. §6 patch #8's cover-letter just states that observation rather than introducing a mechanism.
- Remove all `IMAC5K:` `DC_LOG_WARNING` tracing and raw-EDID dumps; convert anything genuinely useful to `drm_dbg_kms()` at most.
- De-duplicate: the route-check helpers are currently copy-pasted into multiple DC files. Upstream wants single shared helpers.

---

## 5. The hard sells (anticipate maintainer pushback)

These two are the riskiest because they *bypass* DC's own logic rather than fixing it:

1. **Reported-cap bridge** (`dp_verify_link_cap_with_retries` sets `verified_link_cap = reported_link_cap`). Overriding the verified (training-proven) cap with the reported cap looks like a safety bypass at first glance, but the actual bail point — verified against code — is the connection-type check at [link_dp_capability.c:2636](linux-imac-5k/drivers/gpu/drm/amd/display/dc/link/protocols/link_dp_capability.c#L2636), which runs *before* any training. With HPD low on the secondary's first detect, that check returns `dc_connection_none` and the function exits with `verified_link_cap = fail_safe_link_settings` — **training never even runs**, so "make training succeed" does not fix this path. (The follow-on link-config-write fixes from `new_boot_7` matter for the *commit-time* training, not for the pre-commit verify pass.) **Preferred framing for the maintainer-facing patch:** this is not "bypass the verified cap," it is "for a tiled-slave panel known to have no usable HPD on its link, fall back to the DPCD-read `reported_link_cap` as the pre-commit `verified_link_cap` when the connection-type check would otherwise leave it at fail-safe." That mirrors how Windows enumerates modes (against reported caps; see `new_boot_4` analysis in [Linux_plain_boot_5K_requirements_from_RE.md](Linux_plain_boot_5K_requirements_from_RE.md)). The link still has to actually train at SetMode commit; the bridge only unblocks DC's pre-commit mode validation. Gate on `tiled_use_reported_link_cap`. If review still objects, the next-best position is the same fix expressed as an extension of the existing connection-type bail: instead of falling through to fail-safe, consult panel-patch and use reported cap.
2. **`dc_connection_none -> single` detection override.** Forcing "connected" when HPD reads low is also a bypass. **Preferred resolution:** frame it as "this embedded tiled-slave panel has no usable HPD on the secondary link; treat as always-connected when the root panel is present" — a property of the panel, expressed via a panel-patch flag, mirroring how eDP/embedded panels are already treated as hardwired-connected.

**[Boot 1] Major finding on §5.1: the bridge does NOT fire on the cold-boot post-wake path.** Boot 1 log showed: `IMAC5K: secondary 0x3113 link-cap bridge not needed: verified cap already adequate (rate=0x14 lanes=4 bw=17280000 >= reported bw=17280000); 2560x2880 not pruned for bandwidth`. With the pre-wake landed before the first link-config write, the slave's actual training succeeds on the first attempt and `verified_link_cap` is set to real HBR2x4 by `dp_verify_link_cap()` — *before* the bridge code runs. The bridge then sees verified ≥ reported and does nothing.

**[Boot 2] Bridge code shrunk to ~5 silent defensive lines; cold-boot result identical.** Boot 2 stripped the bridge's diagnostic logging and the "needed/not-needed/unknown" three-way branch, retaining only the minimal `if (reported > verified) verified = reported` defensive remnant. Bring-up reproduced cleanly — two clean cold boots in a row now demonstrate the bridge is dead-code on this path.

**[Boot 3] Bridge code DELETED entirely; cold boot + suspend/resume both clean.** Boot 3 deleted the ~5-line defensive remnant *and* the `tiled_use_reported_link_cap` flag that gated it. Cold boot reproduced the `clean_1` outcome (5120×2880 + genlock); a subsequent S3 suspend/resume cycle (~12s offline) also reproduced cleanly — eDP root came back already tile-aware (no AE25→AE26 flip needed on resume), slave HPDRX-redetected and validated `2560x2880` mode, zero `WARN_ON_ONCE`, zero `No DP link bandwidth`, zero `Pruned mode`, zero AUX retry storms. **§6 patch #5 is now empirically deletable.** The maintainer-facing pitch is "we exercised both cold-boot and S3 warm-path with the bridge fully removed; verified cap reaches HBR2x4 in both cases via the wake+retry; the bridge was never needed." The §5.1 "hard sell" is fully retired.

If either §5.1 or §5.2 cannot be turned into a clean capability-driven behavior, expect that specific patch to be the one that stalls.

---

## 6. Proposed patch series (bisectable, smallest-first)

Targeting `amd-gfx` (then `drm-misc`/`amdgpu` trees). Each patch must build and not regress other hardware.

1. **drm/amd/display: add Apple 5K tiled-panel quirk + dc_panel_patch flags (infrastructure).** Adds the `apply_edid_quirks()` `APP/AE25`/`AE26` cases and the new `dc_panel_patch` fields (defaults off). No behavior change yet. Pure plumbing — easiest to review.
2. **drm/amd/display: treat Apple tiled-slave panel as hardwired-connected.** The `tiled_slave_keep_connected` behavior (replaces the `none->single` override), capability-gated.
3. **drm/amd/display: wake Apple tiled panel via root latch before slave AUX.** Adds the shared `link_apple_5k_root_panel_latch_pulse()` helper in `link_dpcd.c` and consolidates the three pre-wake sites (detect / source-DPCD / link-training); `tiled_slave_root_wake`.
4. **drm/amd/display: publish source DPCD table revision for Apple tiled slave.** (`0x310 = 04 1d 03`; may fold into #3.)
5. ~~**drm/amd/display: ensure Apple tiled-slave link trains at full cap.**~~ **REMOVED from series ([Boot 3]).** Empirically the wake+retry from #3 produces the right `verified_link_cap` on both cold-boot (×3) and S3 warm-path (×1); no bridge code is needed. Cover letter documents this as the result of the measurement rather than as a patch.
6. **drm/amd/dm: re-read root EDID after tiled slave brings panel up.** Change A in amdgpu_dm (`tiled_root_force_edid_reread`).
7. **drm/amd/dm: prefer tile-native mode on Apple tiled panel.** `make_tile_mode_preferred` reshaped (consider whether it can be a generic "tiled connector: prefer the tile-native DTD over a same-EDID non-tile preferred mode").
8. **drm/amd/display: force timing-sync grouping for the Apple tile pair.** The dc.c grouping change (`tiled_pair_force_sync_group`). **Master selection is *not* introduced** — Boot 2 + Boot 3 confirm DC's default pipe-ordering already picks the eDP root as master; cover letter notes this rather than introducing a mechanism.

**Effective patch count: 7 substantive patches** (was 8). The deleted #5 is the §5.1 "hard sell"; the un-introduced master-selection in #8 is the other one.

Cover letter (`[PATCH 0/N]`) explains the panel, the dual-tile bandwidth reason, the topology (root eDP + slave DP, both members of one DisplayID tile group, `2560x2880` x2), and that the primary's EDID flips `APP/AE25 → APP/AE26` after the root-wake-driven re-read (so role is identified by `connector_signal`, not by panel id). Links the evidence (`new_boot_7 RESULT` + dmesg + EDID decode of both endpoints).

---

## 7. Process and sequencing

**Phase 0 — refactor in-tree-for-us first (do regardless of upstream):**
Re-architect the current branch onto the panel-patch/EDID model (§4). This is valuable on its own: it generalizes to all 5K iMacs and shrinks the diff dramatically. End at a working `clean_1`-equivalent boot on iMac19,1; the bisectable upstream commit shape is reconstructed *after* Phase 0 (see Phase 1.5).

**Phase 0 is private and disposable.** The patch shape that goes upstream (§6) is reconstructed later via `git rebase -i` from the working post-0 tree — *not* built commit-by-commit during Phase 0. That distinction is the key cycle-saver: Phase 0 only has to *end* at a working tree, not bisect along the way. Techniques unacceptable in the upstream series (mega-commits, debug `WARN_ON_ONCE` cross-checks, dev-machine substitutes for the target) are entirely fair here.

### 7.1 The work to do in Phase 0

Merge into as few iMac boot cycles as possible:

- Add the new `dc_panel_patch` fields and the EDID-keyed `apply_edid_quirks()` case for `APP/AE25 + APP/AE26`; extend the function signature (or its caller `dm_helpers_parse_edid_caps()`) to carry `connector_signal` so root- vs slave-only flags can be set correctly (§4.2).
- Resolve the tile pair in `amdgpu_dm` and set `link->tiled_peer` symmetrically when `has_tile && same tile_group->id`.
- Switch every existing DMI/object-id-gated consumer (§3 table, items A–H) to the corresponding panel-patch flag / `connector_signal` / `tiled_peer` reference. Delete the old gating sites, the `imac5k_find_primary_link()` helper, and every `0x3113`/`0x3114`/DDC/transmitter literal in DC core.
- Strip `IMAC5K:` `DC_LOG_WARNING` spam and raw-EDID dumps; convert anything genuinely useful to a single `drm_dbg_kms()` at detect. Land the cosmetic fixes already noted at the end of [Linux_plain_boot_5K_requirements_from_RE.md](Linux_plain_boot_5K_requirements_from_RE.md) (AUX `-5` retry storm pre-clean, bridge "NOT bridged" log refinement).
- Remove `imac5k_force_genlock` / `force_primary_master`; encode the eDP-is-master rule (§4.4).
- Re-evaluate the cap-bridge frequency on the resulting kernel — input for §5.1 / §9.1 / Phase 1 RFC framing.

### 7.2 Compression techniques to keep iMac boot count low

The iMac boot is the bottleneck. The following collapse the naive ~13–40 boot iteration count down to **3–5**:

- **Verify the `apply_edid_quirks` gate as a ride-along of boot #1.** A single `pr_info` (or `drm_dbg_kms`) at the new `APP/AE25`/`APP/AE26` case is enough: it fires on every iMac boot the moment the real panel EDID is parsed. No dedicated 0a boot, no synthetic or injected EDID — the panel hands us the real EDID already, and that's the whole point of the doc's approach. The verification observes that real EDID; it does not fabricate one.
- **Verify gate equivalence in one boot via `WARN_ON_ONCE`.** For every consumer site whose gating you're moving from DMI/object-id to panel-patch/signal/peer, compute *both* gates and `WARN_ON_ONCE(old != new)` — while keeping the old gate driving behaviour. One iMac boot tells you whether every gate matches in real conditions; if no warning fires, switch them all to the new gate in one commit. Collapses N per-consumer verification boots into ~1. **[Boot 1] confirmed.** Zero `WARN_ON_ONCE` mismatches across all 8 predicate sites on iMac19,1 with the full `clean_1` outcome reproduced — the technique delivered the cycle compression as designed, and the new gate is verified equivalent to the old one for switch-over in Boot 2.
- **Batch independent pure-removal steps.** DMI/literal deletion, log cleanup, module-param removal, and master-rule encoding cannot affect functionality if the prior conversion was correct. One iMac boot covers all of them.
- **Try `kexec` early.** If `kexec -l && kexec -e` survives amdgpu re-init on iMac19,1, per-iteration wall-clock drops sharply (no firmware/grub/initramfs each time). If amdgpu doesn't re-init cleanly under kexec, fall back to normal reboots; the test takes one cycle to find out.
- **`ccache` + incremental builds.** Orthogonal to cycle count but cuts each compile from ~full-tree to ~2 minutes for typical refactor touches.
- **Always `make drivers/gpu/drm/amd/` before booting.** Catches ~90% of refactor mistakes (typos, struct-field renames, missing headers) on the dev box, for free.

### 7.3 Realistic iMac boot budget

| Boot | Purpose | Status |
|---|---|---|
| 1 | `WARN_ON_ONCE` cross-check: all new gates match all old gates across a normal `clean_1` boot. | ✅ **DONE** — 0 mismatches across all 8 predicate sites; `clean_1` reproduced; `obs/0_1/dmesg.txt`. |
| 2 | Switch all gates + delete DMI/literals + log cleanup + param removal + master rule, all in one tree. Confirm `clean_1` still holds and no warnings. | ✅ **DONE** — 0 WARNs, 0 `IMAC5K:` lines, full 5120×2880 + genlock; `obs/0_2/dmesg.txt`. **M0 functional bar met.** |
| 3 | Bonus refactor batch (Tier 1 + cap-bridge deletion) + suspend/resume warm-path verification. | ✅ **DONE** — dedup of 3 root-latch helpers → 1 shared, flag renamed, constants consolidated, cap-bridge code deleted entirely, S3 suspend/resume cycle clean; `obs/0_3/dmesg.txt` + `dmesg_suspend.txt`. **Net diff −633 lines vs `clean_1`.** |
| 4–5 | (Reserve) Not consumed. | — |

**Actual budget consumed: 3 iMac boots + 1 S3 cycle within Boot 3** — squarely in the 3–5 estimated range. The bonus Boot 3 refactors (Tier 1 dedup/rename/consolidate + cap-bridge delete) were not in the original M0 acceptance bar; they landed because Boots 1+2 were clean enough to spend the budget on code-quality wins. Without the compression techniques the equivalent work would have run ~13 boots in the doc's best case and 25–40 in practice. The trade is that each iMac boot does more work, so when one fails, diagnostic narrowing happens via the `WARN_ON_ONCE` / log channel rather than via cross-boot bisection — a fair trade for single-operator bring-up. (In this case, no boot failed.)

### 7.4 Acceptance bar for M0

`clean_1` result on iMac19,1 with: zero DMI strings, zero `0x3113`/`0x3114`/DDC/transmitter literals in DC core, no `IMAC5K:` log spam, no module params, no `WARN_ON_ONCE` firing across a normal boot, and a measurably smaller diff against upstream (today: ~1420 insertions across 10 files). At M0 the tree is one squashy commit (or a handful of pragmatic ones); Phase 1.5 turns that into the upstream-bisectable series.

**Phase 1 — RFC early, before polishing the full series:**
Post an **`[RFC]`** to `amd-gfx@lists.freedesktop.org` (Cc `dri-devel@lists.freedesktop.org`). Use `scripts/get_maintainer.pl` for the exact Cc list; expect AMD DC maintainers (Harry Wentland, Alex Deucher, Rodrigo Siqueira, et al.). Describe the problem + the panel-patch approach + attach a dmesg and EDID decode. The RFC does not need the full bisectable series — a description plus one or two representative patches (the `apply_edid_quirks` infrastructure, plus possibly one DC-core consumer to anchor the §5 discussion) is enough to draw maintainer feedback. **Goal:** get a steer on (a) whether they'll take the DC bits or want to own/rework them, and (b) the §5 hard sells. **Do not invest in a polished 8-patch series before this signal.**

**Phase 1.5 — reconstruct the upstream commit shape:**
With M0 in hand and RFC feedback received, split the Phase 0 squashy tree into the §6 bisectable 7-patch series via `git rebase -i` (or a fresh branch + `git checkout -p`). No iMac boots needed for the split itself; `make drivers/gpu/drm/amd/` per commit catches the easy mistakes. Run `scripts/checkpatch.pl --strict` per commit. **Strip Phase-0-only scaffolding before each commit lands:** the `WARN_ON_ONCE` cross-checks, any debug knobs, the temporary "alongside" gating — none of that survives into the upstream series. One final iMac boot of the *tip* commit confirms the rebase didn't break anything; intermediate commits are bisect-clean by construction (each adds one capability flag and one consumer) but are not boot-tested individually. That's a deliberate trade — the boot budget for Phase 1.5 is ~1 iMac boot, not 7.

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
- [ ] Behaves as a no-op on all non-Apple-5K panels. The `switch (panel_id)` in `apply_edid_quirks()` is the structural proof: every new code path is reached only via `case drm_edid_encode_panel_id('A','P','P', 0xAE25)` / `0xAE26`; any other panel id falls through `default:` and leaves the new `panel_patch.*` fields zero, so the gated DC consumers don't activate. Call this out in commit messages — it's a stronger argument than "we tried a few other monitors and nothing broke."
- [ ] `Signed-off-by` (DCO) on every patch; real name/email.
- [ ] Commit messages describe **observed hardware/DPCD/EDID behavior**, not "the Windows driver does X." Constants like DPCD `0x4F1`, `0x310`, EDID ids `AE25/AE26` are interface facts (fine); keep provenance clean — no copied code/tables from the BootCamp driver.
- [ ] Reference the DisplayID tile metadata and DRM tile-group properties that already exist (`drm_connector` tile fields) so reviewers see it integrates with existing tiled-display infra.
- [ ] Tested-by from the iMac19,1; ideally Tested-by from a second model (17,1/18,3) to prove the EDID-keyed generalization.

---

## 9. Open questions / needs before/while upstreaming

1. **How often is the reported-cap bridge actually hit at runtime?** *[Boot 1 + 2 + 3 — fully answered, closed.]* Cold boot does **not** hit the bridge: three clean cold boots in a row (Boots 1, 2, 3 — each with progressively smaller bridge code, ending with no bridge at all) confirm this. Warm path also clean: Boot 3's S3 suspend/resume cycle reproduced the `clean_1` outcome with the bridge entirely deleted — eDP root returned tile-aware (`tile cap 0x82 size 2560x2880`), DP slave HPDRX-redetected and validated 2560x2880, zero `No DP link bandwidth`, zero AUX retries. §6 patch #5 reduces to a cover-letter footnote: "we tested both cold-boot and S3 warm-path with no cap-bridge in place; the wake+retry path produces `verified_link_cap = HBR2x4` directly in both scenarios."
2. **Second-model confirmation.** Boot on a 15,1/17,1/18,3 (see the per-model analysis in [Linux_plain_boot_5K_requirements_from_RE.md](Linux_plain_boot_5K_requirements_from_RE.md)) to confirm: (a) both endpoints fall under the `APP/AE25` ∪ `APP/AE26` panel-family gate — *note the boot-time primary identity could differ on older models; if a third panel-id appears, add it to the `apply_edid_quirks` case rather than narrowing the gate*; (b) the same DisplayID tile-block topology forms; (c) the `connector_signal`-based role discrimination still holds. This validates the EDID-keyed design and gives a second `Tested-by`. **iMac20,x (Navi) is explicitly *out of scope* for this series**: that machine uses single-stream 5K over DSC on one DP link, not a dual-tile pair (see [RE_5K_iMac19_1_CURRENT.md:65](RE_5K_iMac19_1_CURRENT.md#L65)); none of the wake / keep-connected / genlock work here applies to it. Any iMac20,x issues are a separate DSC-negotiation problem and do not gate this work.
3. **Genlock master selection** — *fully resolved.* eDP root tile is always the timing-sync master, slave follows (see §4.4). **[Boot 1]** showed DC's default pipe-ordering already picks the eDP as master with `force_primary_master=0`. **[Boot 2]** physically removed the entire `force_primary_master` code block from `dc.c` — DC still picks eDP as master. The rule is implicit in DC's existing pipe-ordering; the upstream patch series needs no new master-selection code at all. (Save for the cover letter, where this observation is worth noting so reviewers don't expect a mechanism that isn't there.)
4. **The userspace micro-tile scanout storm** (`Micro tile mode 1 not supported for scanout`, seen in new_boot_8 / clean_1) is a separate Mesa/compositor modifier-negotiation issue on the tiled head — *not* part of this kernel series, but worth a heads-up to reviewers/users. The compositor-side angle — presenting the two DRM connectors as one logical tiled output — is being explored in [KWin_TiledOutputGroup_Option.md](KWin_TiledOutputGroup_Option.md), [KWin_TiledOutputGroup_Implementation_Plan.md](KWin_TiledOutputGroup_Implementation_Plan.md), and the more invasive [KWin_DRM_MultiPipelineOutput_Option.md](KWin_DRM_MultiPipelineOutput_Option.md). Mention these in the cover letter so reviewers see the userspace story isn't being ignored.

---

## 10. Milestones

- **M0:** ✅ **DONE** (2026-05-28 / 2026-05-29). Phase-0 panel-patch refactor + bonus code-quality refactors complete on `mainline-prep`; iMac19,1 reaches the `clean_1` result on cold boot AND on S3 suspend/resume with zero `WARN_ON_ONCE`, zero `IMAC5K:` log spam, zero DMI/object-id literals in DC core, no module params, three root-panel-latch helpers deduplicated into one shared helper, cap-bridge code entirely removed, flag rename applied (`tiled_pair_force_sync_group`), constants consolidated. Diff vs `clean_1`: **+314 / −947 (net −633 lines)**. Budget consumed: 3 iMac boots + 1 S3 cycle. Logs: `obs/0_1/dmesg.txt` (Boot 1: WARN cross-check), `obs/0_2/dmesg.txt` (Boot 2: gate switch + cleanup + M0 functional bar), `obs/0_3/dmesg.txt` + `dmesg_suspend.txt` (Boot 3: Tier-1 refactors + cap-bridge deletion + suspend/resume). Working tree, *not yet bisectable* — the upstream commit shape comes in Phase 1.5.
- **M1:** RFC posted to `amd-gfx`; maintainer direction received.
- **M1.5:** Phase-0 tree split into the §6 bisectable patch series via `git rebase -i`; Phase-0 scaffolding (`WARN_ON_ONCE`, debug knobs) stripped; one final iMac boot confirms the tip still reaches `clean_1`. (Budget: ~1 iMac boot.)
- **M2:** Series v1 posted.
- **M3:** amdgpu_dm + `apply_edid_quirks` patches accepted.
- **M4:** DC-core patches accepted (or AMD-owned equivalent merged).
- **M5:** Second-model `Tested-by` on another pre-2020 5K iMac (15,1 / 17,1 / 18,3) confirming the EDID-keyed generalization. (iMac20,x/Navi is single-stream DSC and is *not* part of this series — see §9.2.)
