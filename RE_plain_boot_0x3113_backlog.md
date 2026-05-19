# Plain-Boot 0x3113 Route Backlog

Date: 2026-05-09

This note intentionally parks the Apple-free plain-boot problem so the current work can stay focused on preserving the already-live 5K split/tiled state after the diagnostics/OCLP-style boot path.

Update 2026-05-15:

- Stage 1 is proven by `obs_22`: after the richer boot path, Linux can preserve the already-trained secondary `0x3113` route and present both `2560x2880` tiles as a working 5K desktop.
- The CoreEG2/GOP replay record is now decoded. The successful firmware shape is two real endpoints under one logical `5120x2880` timing: root connector at `0,0`, sibling connector at `tile_width,0`.
- GOP replay consumption does not itself train the secondary route; it publishes the replayed topology and programs viewport/window registers. CoreEG2 trains both halves immediately afterward through graphicsconnector backend slot `+0x28`, once for root and once for sibling, using the same `display+0x160` DP training request.
- Plain boot therefore still needs real `0x3113` AUX/HPD readiness before the replay-equivalent topology or training sequence can be useful.
- `obs_23` shows the plain-boot-shaped failure cleanly: object id, DDC3, and UNIPHY_D mapping for `0x3113` are present, but AUX/HPD is dead (`aux_mode=0`, `hpd=0/0`) before Linux can snapshot EDID, TILE, `0x10A`, or `0x4F1`.

## Goal

Make Linux reach the same real two-tile topology without `diags.efi`, `Product.efi`, OCLP, or any Apple preboot display binary:

- primary tile: object id `0x3114`, DDC4, UNIPHY_C, left `2560x2880`;
- secondary tile: object id `0x3113`, DDC3, UNIPHY_D, right `2560x2880`;
- one DRM tiled-monitor group exposing logical `5120x2880`;
- real AUX/DDC/link backing for both halves.

The target is not a fake connector, not a synthetic 5K-only mode, and not copied TILE metadata without a real secondary transport.

## Why Deferred

The current Stage 1 failure is still after the richer preboot path has already produced a useful split state. Therefore the immediate priority is to make Linux/GNOME keep and present that state correctly.

Plain boot is Stage 2. It should reuse the same Linux representation that works for Stage 1 rather than inventing a separate model.

## Windows RE Anchors Already Found

- `PopulateDisplayMapAuxObjectsForHotplugRange`: populates durable per-object AUX/DPCD display-map entries.
- `CreateDisplayMapInterfaceForObjectIdMaybeSkipInternal`: accepts class-3 display ids including `0x3113` and `0x3114`.
- `AdapterServiceGetAuxHandleForObjectId`: queries AMD object descriptors and builds AUX handles from selector/source-mask data.
- `AdapterServiceQueryObjectDescriptor`: provides descriptor fields used by the AUX route.
- `AuxFactoryCreateHandleForSourceMask`: creates the object-specific AUX transaction handle.
- `ConstructDualPinAuxTransactionHandle`: opens concrete DDC DATA/CLOCK backends.

Known Windows route mapping:

- secondary `0x3113`: selector `0x4871`, AUX pin index `2`, DDC3, UNIPHY_D;
- primary `0x3114`: selector `0x4875`, AUX pin index `3`, DDC4, UNIPHY_C.

## Linux Places To Revisit Later

- `drivers/gpu/drm/amd/display/dc/core/dc.c`: `create_links()`
- `drivers/gpu/drm/amd/display/dc/link/link_factory.c`: link construction from object ids
- `drivers/gpu/drm/amd/display/dc/link/link_detection.c`: initial detect and sink creation
- `drivers/gpu/drm/amd/display/dc/bios/bios_parser2.c`: object-table connector and I2C/AUX mapping
- `drivers/gpu/drm/amd/display/dc/link/protocols/link_ddc.c`: DDC service construction

## Questions For Later

- Does plain Linux enumerate a `dc_link` whose raw object id is `0x3113`?
- If yes, when is it dropped, hidden, or misclassified?
- If no, which BIOS/object-table filter prevents `0x3113` from becoming a Linux link?
- Can Linux construct the real DDC3/UNIPHY_D secondary route from the existing VBIOS/object-table data?
- Does the route need any Apple preboot-initialized state before ordinary AUX reads work?
- Once the route exists, can it feed the same false-disconnect suppression, DPCD, TILE, and GNOME-facing paths used by Stage 1?

## Acceptance Shape

Plain boot is solved only when a normal systemd-boot or Limine boot reaches the same effective topology as the successful probe/OCLP boot:

- real primary and secondary physical streams;
- real AUX/DPCD writes possible on `0x3113`;
- correct `2x1` TILE contract;
- GNOME/KDE see one usable 5K internal monitor;
- no Apple EFI binaries and no pre-systemd shim.
