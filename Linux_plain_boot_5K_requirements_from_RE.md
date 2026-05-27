# Linux Plain-Boot 5K Requirements From RE

Date: 2026-05-26

This is the current Linux implementation summary distilled from the latest RE notes, especially `RE_5K_iMac19_1_CURRENT.md`, plus the recent `new_boot_3` kernel result. Treat this as the short working spec for the plain-boot Linux path.

## Target State

Plain Linux boot should reach the same real two-tile topology that the good OCLP/diagnostics-style boot can hand off:

- primary/left tile: AMD object `0x3114`, Linux `eDP-1`, DDC4 / hw instance `3`, `UNIPHY_C`, product `APP/AE25`;
- secondary/right tile: AMD object `0x3113`, Linux `DP-1`, DDC3 / hw instance `2`, `UNIPHY_D`, product `APP/AE26`;
- both tiles expose real EDID/DisplayID TILE metadata;
- both tiles run real `2560x2880` streams;
- DRM/userspace sees one logical `5120x2880` tiled internal monitor;
- no OCLP, diagnostics EFI, Product.efi, boot shim, or firmware-side helper is required.

The target is not a fake connector, not a synthetic single `5120x2880` stream, and not copied TILE metadata without a live secondary AUX/link route.

Two identity details below need verification against captures, because they affect EDID/capability gating:

- The primary is listed as product `APP/AE25`, but both captured EDIDs (`primary_edid.txt` and `secondary_edid.txt`) and the `new_boot_3` secondary log show base EDID model `0xAE26` (`APP/AE26`). The `AE25 -> tag 0x3A` vs `AE26 -> tag 0x3B` split is a Windows capability-table mapping; the RE itself says the base EDID can be `APP/AE25` **or** `APP/AE26`. Confirm the primary's real base model before relying on `AE25` for the primary; if it is also `AE26`, correct this line.
- Older notes that mention raw capability `0x36` should not be reused without re-checking the exact old-driver table. In the modern BootCamp driver, the relevant APP rows are source id `0x38` (`AE25` -> tag `0x3A`) and `0x39` (`AE26` -> tag `0x3B`), while the separate physical EDID refresh gate checks capability `0x25`.

## What The Firmware RE Says

The Apple firmware success path is useful mostly as a reference model.

- `CoreEG2` and the AMD GOP build a two-endpoint replay/topology record for the internal panel.
- The successful topology has endpoint 0 at viewport `0,0` and endpoint 1 at `tile_width,0`.
- The replay record describes topology and viewports; it does not itself do all link training.
- After replay succeeds, `CoreEG2` trains/applies root and sibling as two real connectors through GOP backend slot `+0x28`.
- Decimal `1265` is DPCD address `0x4F1`, not a low-byte mailbox opcode.
- The immediate firmware 5K-vs-4K fork is whether the sibling connector/backend `0x200` record becomes valid after a root-side `0x4F1 = 1` pulse.
- Plain boot does not fail because `GOP_BOOTCAMP_SUPPORT` is absent. Probe results showed the protocol can be present while replay state is empty.

So for Linux, the useful lesson is:

- primary `0x4F1 = 1` can wake or expose the secondary tile route;
- the final runtime solution still needs a real secondary `0x3113` AUX/link/sink path, not only primary-side firmware state.

## What The Windows RE Says

Windows' mechanism is driver-owned once the relevant route exists. It is not a single hidden BootCamp EFI callback.

### Object and AUX Route

Windows constructs the real secondary route from provider/object-info data:

- object id `0x3113`;
- selector/register `0x4871`;
- AUX pin index `2`;
- DDC3;
- `UNIPHY_D`.

The primary route maps separately:

- object id `0x3114`;
- selector/register `0x4875`;
- AUX pin index `3`;
- DDC4;
- `UNIPHY_C`.

The important Windows object is the durable per-object DPCD/AUX backend stored in the display-map entry at `entry+0x18`. Recent RE confirmed the DPCD read/write wrappers use this lower handle and do not visibly require HPD-high or a live detection object before submitting a DPCD transaction.

Linux implication:

- do not wait for ordinary DP HPD-high detection before trying the exact secondary route;
- prove route identity first;
- arm/force AUX transaction mode on `0x3113` / DDC3 / `UNIPHY_D`;
- issue a tiny DPCD probe such as `0x000`;
- if it succeeds, continue detection narrowly for this internal tile.

### Route Is Necessary But Not Sufficient

Windows separates at least three layers:

- lower physical route and DPCD/AUX transport;
- logical sink/stream presence;
- stream-enable readiness.

RE confirmed stream enable does not proceed from the DPCD route alone. The Windows stream-enable wrapper checks a mapped emulated-sink object's `ConnectionStatus bit0`, then checks stream/VC `sink_present`. False-disconnect handling for the Apple/tiled route prevents destructive teardown of this logical state.

Linux implication:

- preserving or creating `dc_sink` / connector sink state is a separate requirement from having the AUX route;
- a transient disconnected result on exact `0x3113` must not immediately clear the real secondary sink during tile-pair bring-up;
- do not allow the tile-pair stream-enable path based only on cached EDID/TILE. It should require route proof plus a real/preserved logical secondary sink.

### Windows DPCD / Training Shape

The latest RE supports this order as the Linux model:

1. Publish lower DP source-DPCD before destructive training writes.
2. For the mapped DCE path, write source-DPCD `0x310 = 04 1d 03`.
3. Apply the secondary `0x3113` ASSR policy:
   - set DPCD `0x10A` bit `0`;
   - make the same policy visible to the training-pattern / PHY request.
4. Treat `0x111` as observable/conditional, not an unconditional enable bit. Windows can clear bit `0` before stream enable.
5. Program/update DPCD `0x316` during stream-enable style finalization when the relevant Windows stream mode requires it.
6. Train or reuse cached link settings.
7. Run the real stream-enable path.
8. Write sink-private DPCD `0x4F1 = 1` after stream/source programming on the real secondary AUX route.

`0x205`, `0x32F`, and broad `0x111` writes remain diagnostics unless a specific runtime log proves they are active for this panel path.

### BootCamp Driver Comparison

The old-vs-first-5K BootCamp comparison found a real mechanism but not a standalone magic initializer. The latest modern-driver RE refines what we can safely claim:

- the relevant modern APP panel capability rows are source id `0x38` (`AE25` -> tag `0x3A`) and source id `0x39` (`AE26` -> tag `0x3B`);
- existing DisplayID/TILE/SLS logic then consumes that endpoint's EDID or patched EDID buffer;
- no new Apple-specific `0x4F1` writer appeared in that old-driver delta.

Important correction: the modern physical EDID-reader path has its own forced-refresh gate, `HasCapability(37)` / `0x25`, plus an explicit force argument. Current RE does **not** prove that the `APP/AE25` or `APP/AE26` rows set this `0x25` gate. So `AE25/AE26` are proven to drive endpoint-local pin-cap EDID fixups and TILE parsing, but they are not by themselves proof that Windows physically re-reads the primary.

Latest call-surface check: the traced `DcLinkDetectHelper` EDID path does **not** call the `Forced EDID read.` vfunc. It uses the normal DDC-service reader (`DdcService_ReadEdidBlocksForDetect` -> `DdcService_ReadOneEdidBlockWithRetries` -> AUX/I2C block read), then immediately builds the pin-cap-aware parser. Keep any forced primary re-read as an empirical Linux probe, not as proven Windows normal-path behavior.

The one concrete EDID transform the RE found is `ApplyPinCapabilityEdidFixups()` -> `DoubleEdidPhysicalWidthForTiledCaps()` (gated by tags `0x3A`/`0x3B`): it doubles EDID byte `21` (horizontal physical size) when horizontal < vertical and < `0x80`, then recomputes the block checksum. This is a **physical-geometry** fixup for the tiled half — it does **not** remove modes and does **not** re-flag preferred modes. So Windows does not solve the 4K-vs-tile question by deleting the 4K mode; it leaves the mode list intact and selects the tile mode through tile-group SetMode logic (`BuildSourceModeForTileSplitPath`). Linux should likewise avoid force-removing 4K as the primary fix.

Latest primary-tile clarification: Windows can derive/inject the `2560x2880` half-tile mode, so the raw EDID does not have to literally advertise that mode. But Windows' grouped SetMode path still requires each endpoint, including primary `0x3114`, to have a nonzero TILE metadata block with the same group token and full grid coverage. If a forced primary EDID re-read still returns 4K/no-TILE, Linux should synthesize only the missing primary left-tile metadata/mode under the exact iMac route, and treat that as a compatibility fallback rather than a Windows-proven normal path.

For modern iMac19,1 `AE25/AE26`, the useful lesson is still outcome-based:

- force/redo EDID and TILE discovery only for a proven Apple internal tiled route, and log whether the primary physical re-read actually changes the EDID/TILE state;
- if a physical-geometry correction is needed, mirror the Windows `DoubleEdidPhysicalWidthForTiledCaps` shape rather than pruning modes;
- do not model `KMD_BootCampPlatform` as the 5K bring-up knob. Its decoded effects are timing-list policy, not `0x3113` route creation, AUX construction, ASSR, source-DPCD, or `0x4F1`.

## What Current Linux Already Proves

The recent kernel tests moved the plain-boot problem forward.

`new_boot_3` proves the AUX **read** path on the secondary:

- primary `0x3114` `0x4F1 = 1` wake succeeds (`status=1`, read-back `0x01`);
- secondary `0x3113` AUX becomes **readable** even with HPD low (`dpcd_rev=0x12` on the pre-detect probe, `reason=0` and `reason=2`);
- the secondary detection override can reach a real sink;
- secondary EDID is real, length `256`, read over AUX, with DisplayID TILE:
  - tile size `2560x2880`;
  - location `1x0`;
  - grid `2x1`;
  - product `APP/AE26`.

This means the current blocker is no longer "Linux cannot wake the secondary" or "Linux cannot read the secondary EDID".

Important scope limit: only the AUX **read** half is proven. AUX **writes** on the secondary are NOT reliable. In `new_boot_3` the secondary source-DPCD `0x310` write succeeded on some passes (`status=1`) and failed on others (`status=-1`), and the link-configuration writes failed outright (see below). So "AUX route proven" must be read as "AUX reads proven; AUX writes still marginal".

## Current Failure After `new_boot_3`

The blank screen in `new_boot_3` is caused by **failing secondary AUX writes**, not by a DRM/fbdev mode-pruning bug. This was misread initially; the verified chain is:

1. The secondary AUX is **read-capable but write-marginal**. DC's link-configuration writes on the secondary (`aux hw bus 1`) fail:
   - `*ERROR* dpcd_set_link_settings: core_link_write_dpcd (DP_LINK_BW_SET) failed`   (`0x100`)
   - `*ERROR* dpcd_set_link_settings: core_link_write_dpcd (DP_LANE_COUNT_SET) failed` (`0x101`)
   - `*ERROR* dpcd_set_link_settings: core_link_write_dpcd (DP_DOWNSPREAD_CTRL) failed` (`0x107`)
   - underlying transport error: `AMDGPU DM aux hw bus 1: Too many retries, giving up. First error: -5` (`-EIO`).
2. Because those writes fail, DC cannot train/program the secondary link, so its **verified link cap stays minimal** (well below HBR2 x4).
3. `create_validate_stream_for_sink` then computes bandwidth against that minimal cap and prunes the real tile mode for **`No DP link bandwidth`** — and it prunes it at **10-bpc, 8-bpc, AND 6-bpc**. Pruned even at 6-bpc is the tell: the link DC believes it has is far below what `2560x2880 @ 483250` needs, even though the primary drives `3840x2160 @ 533250` (more bandwidth) on its own link.
4. With `2560x2880` pruned, the fb/DRM client falls the secondary to `640x480`, the primary to `3840x2160`, and the genlock/timing-sync quirk correctly does not engage (`bypass=0`, `group_size=1`).

So the `640x480` fallback and the timing-sync miss are **downstream symptoms**. The mode prune is **correct** — DC genuinely cannot carry `2560x2880` on a link it could not program.

Two distinct rejection reasons appear in the logs and must not be conflated:

- `ERROR` = DC `mode_valid` / `create_validate_stream` returning `No DP link bandwidth` on the secondary (`DP-1`). This is the real blocker, rooted in the failed AUX writes.
- `VIRTUAL_Y` = `drm_mode_validate_size` height/`maxY` constraint on the other connector. This is a separate, lower-priority size prune and is not the cause of the blank.

Linux implication:

- the next fix must make the **secondary AUX usable for writes through link training**, so `0x100/0x101/0x107` succeed and DC verifies the real HBR2 x4 cap;
- only after the link is programmable does `2560x2880` survive validation; hiding the `640x480` fallback or force-injecting `2560x2880` would just produce a black/failed secondary, because the link still cannot be trained;
- likely cause of the write failures: the secondary AUX is only solid for a short window right after the wake (when the EDID read happens), then degrades. The detect-time re-wake fixed the *teardown*, but DC's link-config writes happen later, in the degraded window. Keeping the panel/AUX awake **through** the link-config writes (re-wake/hold immediately before training, and/or retry the writes under a wake) is the leading hypothesis.

### `new_boot_4` refinement: the prune is verified-cap-gating, not just write failures

`new_boot_4` sharpened this. That boot had **zero** secondary link-training activity and **zero** link-config write failures, yet `2560x2880` was **still** pruned for `No DP link bandwidth` at 10/8/6-bpc. So the prune is not (only) caused by failing writes. The deeper, consistent cause:

- DC validates modes against `link->verified_link_cap`, which is established by training during `dp_verify_link_cap`.
- For the secondary, `dp_verify_link_cap_with_retries` **bails at the HPD-low `dc_connection_none` check before training** (or the link-config writes fail), so `verified_link_cap` is left at fail-safe (minimal).
- `create_validate_stream` then prunes `2560x2880` against that minimal cap.

This is a **Linux-DC behavior, not a Windows one**: Windows enumerates modes against the sink's **reported** DPCD caps (which our working AUX *reads* provide as the real HBR2 x4) and only confirms the link by training at SetMode. DC's verified-cap gating is what produces the prune.

Fix implemented: a **reported-cap bridge** in `dp_verify_link_cap_with_retries` — for the exact iMac secondary route, if `reported_link_cap` (real DPCD-read) has more bandwidth than the fail-safe `verified_link_cap`, set `verified_link_cap = reported_link_cap`. This is not a fabricated cap; it is the sink's own reported capability, and it mirrors how Windows enumerates. It only unblocks mode validation; the link must still actually train at commit time (the re-wake/retry from requirement 8). If the bridge logs `reported` as not HBR2 x4, the next problem is the cap *read*, not verification.

## Linux Implementation Requirements

### 1. Exact Route Gate

Every quirk must be gated to the exact iMac internal tile route:

- DMI match for the target iMac family;
- object `0x3113` for secondary;
- DDC hw instance `2`;
- `UNIPHY_D`;
- Apple internal tiled EDID/DisplayID evidence when available.

Primary wake must be gated separately:

- object `0x3114`;
- DDC hw instance `3`;
- `UNIPHY_C`;
- real primary sink present.

### 2. Primary Wake For Plain Boot

Plain boot currently needs the firmware-shaped wake:

- write DPCD `0x4F1 = 1` on primary `0x3114`;
- wait/poll secondary AUX;
- then retry secondary DPCD `0x000` probe.

This should remain narrow and logged. The root-side wake is a way to expose the sibling route, not the final secondary stream-enable latch.

### 3. Secondary AUX Before HPD

For exact `0x3113`:

- arm/force AUX transaction mode even when HPD says low;
- perform a small DPCD proof read;
- if the read succeeds, allow the DP sink path to continue despite ordinary `none` detection.

This matches Windows' object-local DPCD backend model.

### 4. Real Secondary Sink Lifetime

Linux needs a narrow false-disconnect policy:

- preserve `dc_sink` / connector sink state for exact `0x3113` only after real route and AUX proof;
- do not preserve forever;
- expire on deliberate full disable, real route mismatch, repeated DPCD failure, or non-Apple/non-tiled identity;
- log the suppression as false-disconnect preservation, not as a fresh physical HPD success.

This corresponds to Windows preserving logical stream/sink readiness instead of letting a false disconnected result clear the live object.

### 5. Secondary Source / Training Policy

For exact `0x3113` when the route is proven:

- publish source-DPCD `0x310 = 04 1d 03`;
- force secondary ASSR policy so DPCD `0x10A` bit `0` is written before training;
- pass the same ASSR/panel policy into training-pattern programming where Linux has the equivalent hook;
- keep `0x111`, `0x205`, `0x316`, and `0x32F` logged unless we add a tightly gated experiment.

### 6. Stream-Enable Latch

For exact `0x3113`, write DPCD `0x4F1 = 1` at stream-enable/finalization time after the route, sink, source-DPCD, and stream state exist.

Avoid treating secondary `0x4F1` as the first wake operation. In Windows, the strongest match for `AE26` / tag `0x3B` is the stream-enable-time writer.

### 7. Correct Mode Shape Before Genlock

The genlock/timing-sync quirk only matters after the DRM state contains the correct pair:

- `0x3114`: `2560x2880`;
- `0x3113`: `2560x2880`;
- same tile group;
- complementary tile locations `0x0` and `1x0`.

The quirk should not engage for `3840x2160 + 640x480`, and `new_boot_3` correctly showed it did not.

### 8. Secondary AUX Write / Link-Training Reliability (current blocker)

This is the actual blocker exposed by `new_boot_3`. The secondary AUX is read-capable but write-marginal, so DC cannot program the link, so it prunes `2560x2880` for `No DP link bandwidth`. The fix is to make the secondary link programmable, not to mask the resulting `640x480` fallback.

Required:

- ensure the secondary `0x3113` AUX is solidly awake for **writes**, not just reads, through the link-configuration phase (`DP_LINK_BW_SET 0x100`, `DP_LANE_COUNT_SET 0x101`, `DP_DOWNSPREAD_CTRL 0x107`);
- leading approach: re-assert the primary `0x4F1 = 1` wake (and/or hold the panel awake) immediately before DC's link-config writes, and retry the `0x100/0x101/0x107` writes under that wake, since the AUX appears solid only briefly after a wake;
- success criterion: those three DPCD writes return success, DC verifies the secondary cap as HBR2 x4, and `create_validate_stream_for_sink` stops pruning `2560x2880` for `No DP link bandwidth`.

Explicitly do NOT rely on these alone (they are downstream band-aids that cannot work while the link is unprogrammable):

- hiding/rejecting the `640x480` fallback on `0x3113`;
- force-injecting or un-pruning `2560x2880`;
- "bypassing" the `ERROR` mode validation.

Keep `VIRTUAL_Y` separate: it is a `drm_mode_validate_size` (`maxY`) prune on the other connector, not the bandwidth blocker. Only revisit it if it still blocks the tile mode after the AUX-write/link-training fix lands.

## What Not To Reintroduce

Do not go back to older assumptions unless new evidence appears:

- no fake secondary sink without real route/AUX proof;
- no synthetic single-stream `5120x2880` model for this pre-2020 iMac path;
- no broad HPD override;
- no global eDP/MST forcing;
- no unconditional DPCD `0x111` setting;
- no broad primary or secondary `0x4F1` writes outside the exact route gates;
- no `KMD_BootCampPlatform`-style knob as the Linux model;
- no assumption that OCLP is the final mechanism. OCLP is useful because it creates or preserves a richer preboot state, but Linux must own the route/sink/stream sequence.

## Current Next Engineering Step

The next test kernel should keep the proven route/wake/AUX-read/EDID pieces, then make the secondary link **programmable**:

1. Keep exact primary wake and secondary AUX read poll.
2. Keep exact secondary source-DPCD, ASSR, EDID/TILE logging, and stream-enable `0x4F1`.
3. Make the secondary AUX usable for **writes** through link training: re-assert the primary `0x4F1 = 1` wake (and/or hold the panel awake) immediately before DC's link-config writes, and retry `DP_LINK_BW_SET 0x100` / `DP_LANE_COUNT_SET 0x101` / `DP_DOWNSPREAD_CTRL 0x107` under that wake.
4. Add logging around the secondary link-config writes and the verified link cap: log each `0x100/0x101/0x107` write status, the resulting verified rate/lane, and the `create_validate_stream` bandwidth verdict for `2560x2880`. The make-or-break line is whether those writes flip from `failed`/`-5` to success and whether `No DP link bandwidth` disappears.
5. Only after the secondary link is verified at HBR2 x4 and `2560x2880` survives validation on both tiles will the first committed pair be `2560x2880 + 2560x2880`. Then evaluate genlock:
   - expected success log should show `bypass=1` or normal timing sync;
   - resulting timing group should have `group_size=2`;
   - primary should be the timing master if the force-primary-master parameter is enabled.

The shortest version: Linux now knows how to wake the secondary, read its EDID, and see its TILE metadata. The remaining Linux problem is that the secondary AUX is only reliable for **reads**, so DC cannot write the link-config DPCD, cannot train the link, and therefore prunes the real `2560x2880` tile mode for `No DP link bandwidth`. Fix the secondary AUX-write/link-training reliability first; the `640x480` fallback, the mode prune, and the genlock miss all resolve once the link is programmable.

## Change A: Forced Primary EDID Re-read (implemented 2026-05-26, pending `new_boot_7`)

After `new_boot_4/5/6` and `RE_5K_iMac19_1_CURRENT.md` ss5808/5862, the secondary-side blockers were addressed in code (reported-cap bridge so `2560x2880` survives validation without proven training; `make-tile-preferred` so 4K is never the preferred mode). The **isolated remaining blocker** is that the **primary eDP-1 / `0x3114` stays in its 4K-compat identity** (product `0xAE25`, no DisplayID TILE) while only the secondary `0x3113` comes up tile-aware (`0xAE26`, TILE `2560x2880`). Without primary TILE metadata the grouped tile pair cannot form. The RE recommends the empirical probe noted above (line ~125): force the primary to physically re-read its EDID after the secondary is up, and **log whether the EDID/TILE state actually changes**.

Implemented as `amdgpu_dm_imac5k_reprobe_primary_after_secondary()` in `amdgpu_dm.c`, called once at the end of `amdgpu_dm_initialize_drm_device()` (after the boot detect loop, so the secondary has had its chance to latch the panel into tile mode). Tightly gated: DMI `iMac19,1` + exact primary/secondary routes; only runs when the secondary actually carries the TILE; idempotent (skips if the primary already carries the TILE identity). No-op on all other hardware.

**Key Linux-specific constraint that shaped the implementation.** DC's `detect_link_and_local_sink()` **early-outs for eDP** when the link already has a `local_sink` and `allow_edp_hotplug_detection` is false (the default): it returns the cached sink **without re-reading EDID**. So a second `dc_link_detect()` on the primary is a silent no-op for the EDID. Bypassing the early-out via the global `allow_edp_hotplug_detection` flag is unsafe — it runs the eDP connection-type power sequence, which **powers the panel down** if HPD reads low at that instant. Therefore Change A calls `dm_helpers_read_local_edid()` directly — the exact reader DC uses during detect (DDC/AUX read + `drm_edid_connector_update`, which refreshes the connector EDID + `has_tile` + the DC sink caps) — which is power-neutral and cannot disconnect the panel. It then resyncs `aconnector->drm_edid` (the cache `get_modes` consumes) from the freshly read bytes.

This was a deliberate choice over the rejected Change B (synthesizing primary TILE metadata): Change A only *reads what the panel actually returns* and never manufactures state Windows was not seen producing.

Each `pre-reread`/`post-reread` pass also emits raw wire-EDID lines (`... raw-edid: ... mfg='APP' ... id8_23=<hex>` plus one `ext[N] tag=0x70 (DisplayID)` line per extension block), so the result can be judged on the literal bytes the panel returned — most importantly the product code in `id8_23` (bytes 10-11) and whether a `tag=0x70 (DisplayID)` extension (which carries TILE) is present — not only on parsed `product_id`/`has_tile`.

### `new_boot_7` RESULT: CONFIRMED — Change A flips the primary; native 5K works

`new_boot_7` (kernel `g066b89c6f6da`) hit the hoped-for `CHANGED` outcome and the full tile pair formed. Plain-boot Linux now brings up native `5120x2880`.

- `primary 0x3114 pre-reread product=0xae25 has_tile=0` ext `tag=0x02 (CEA-861)` ⇒ `post-reread product=0xae26 has_tile=1 tile=2560x2880 loc=0,0 grid=2x1 ... CHANGED` ext `tag=0x70 (DisplayID)`. Raw `id8_23` bytes show `25 ae` → `26 ae` on the wire. The panel **does** hand the primary a real DisplayID-TILE EDID once the secondary is up — nothing synthesized.
- Both endpoints then carry complementary TILE in the same group (primary left `loc 0x0`, secondary right `loc 1x0`, grid `2x1`); `make-tile-preferred` sets `2560x2880` sole-preferred on both; DRM forms the tile group (`found tiled mode: 2560x2880` on each); secondary trains HBR2 x4 (`link-config writes OK after re-wake retry`); genlock `group_size=2 master=1` (primary master); `crtc-0 (0,0)` + `crtc-1 (2560,0)` ⇒ `fbdev surface 5120x2880`. Stable to t=221s.

This validates the whole RE-derived chain end to end and confirms ss5936 "bullet 1": the physical primary EDID changes after the root `0x4F1` wake + re-read. See `RE_5K_iMac19_1_CURRENT.md` "new_boot_7 RESULT" for the full annotated trace.

Other outcomes (kept for reference, did NOT occur this boot):
- `post-reread ... product=0xae25 has_tile=0 ... UNCHANGED` ⇒ would mean the panel does not flip the primary on a software re-read; the only Windows-faithful lever left would be the per-endpoint TILE metadata the RE describes (line ~121) — **not** the rejected manufacture-everything Change B.
- `primary 0x3114 re-read skipped: secondary 0x3113 not in tile mode` ⇒ the secondary itself did not come up tiled; re-triage the secondary path first.

Remaining items were cosmetic only (do not block 5K) and are now fixed in code (pending next-boot confirmation): (1) the AUX `-5` retry storm before the first secondary link-config write — `dpcd_set_link_settings` now pre-wakes the panel via primary `0x4F1` + settle before the first write on the iMac secondary route, so the first attempt lands cleanly (retry loop kept as safety net); (2) the stale `link-cap NOT bridged ... will still be pruned` log — the bridge `else` now distinguishes "verified cap already adequate (not pruned)" from "reported caps unknown (may be pruned)". Neither change touches the bring-up logic.
