# KWin Tiled Display Option A: First-Class TiledOutputGroup

## Summary

This option keeps KWin's existing `BackendOutput` and `LogicalOutput` model, but adds an explicit intermediate object for tiled displays: a `TiledOutputGroup`.

The group would own all tile-specific behavior. Instead of spreading special cases through `Workspace`, `OutputConfigurationStore`, output-management, the DRM connector, and the compositor, KWin would build a validated group from multiple physical outputs and expose one logical output for it.

This is the conservative architecture. It borrows the important parts of Mutter's strategy without requiring KWin to fully redesign its monitor stack.

## Goal

Represent a tiled display as one user-visible output while still allowing the DRM backend and compositor to address each physical tile.

For an iMac 5K-style 2x1 panel, users and clients should see one display. Internally, KWin still knows that the display is backed by two connectors, two modes, two scanout paths, and two render targets.

## Why This Option Fits KWin

KWin currently has many APIs that assume:

- `BackendOutput` represents a physical output.
- `LogicalOutput` is the user-visible output abstraction.
- Many paths still expect a `LogicalOutput` to be backed by one primary `BackendOutput`.
- Output-management and KScreen operate on exposed outputs.
- The compositor can render different backend outputs separately.

A `TiledOutputGroup` works with those constraints. It does not require pretending that the DRM backend has only one output. It instead gives KWin a single place where multiple backend outputs become one logical display.

## High-Level Model

```text
BackendOutput  BackendOutput
     \          /
      \        /
   TiledOutputGroup
          |
    LogicalOutput
```

Normal displays would use either no group or a trivial one-member path.

Tiled displays would use a validated multi-member group.

## Core Objects

### BackendOutput

`BackendOutput` should expose passive tile metadata:

- tile group identity
- horizontal and vertical tile location
- maximum horizontal and vertical tile count
- tile width and height
- tile flags, if useful later
- backend or GPU identity

It should not decide whether the display is validly tiled. It should only report what the backend knows.

### TiledOutputGroup

The group should own:

- member backend outputs
- primary or main tile
- tile grid dimensions
- aggregate pixel size
- per-tile rectangles in logical output space
- aggregate modes
- mapping from one logical mode to per-member backend modes
- policy for enabling, disabling, DPMS, transform, scale, color, brightness, and mirroring

### LogicalOutput

`LogicalOutput` should represent the group as a single user-visible display.

For a first implementation, it can still keep a primary `BackendOutput` for compatibility, but code that cares about physical tile layout should ask the group.

Longer term, `LogicalOutput` should point to an output group abstraction instead of directly implying one backend output.

## Tile Group Identity

Do not key tile groups by raw `int groupId`.

The kernel's tile group IDs are scoped per DRM device, not globally. The grouping key should include backend or GPU identity:

```text
(backend identity, tile group id)
```

For the DRM backend, this should be based on the `DrmGpu` or DRM device object. For the virtual backend, the backend instance can act as the namespace.

This avoids accidental grouping when two GPUs both report tile group id `1`.

## Validation

KWin should only create a `TiledOutputGroup` when the tile set is complete and consistent.

Validation should require:

- nonzero tile group id
- all members belong to the same backend/GPU namespace
- all members have the same group id
- all members have the same grid dimensions
- all members have the same tile width and height
- tile locations are inside the declared grid
- no duplicate tile positions
- the number of members equals `max_h_tiles * max_v_tiles`
- the grid is fully covered

If validation fails, KWin should leave those outputs as independent normal outputs.

This follows Mutter's important safety behavior: a broken or partial tiled setup should degrade into visible independent outputs, not into a half-created logical monitor.

## Mode Strategy

KWin should avoid destructive mode filtering in the DRM connector.

The connector's raw mode list should stay intact. The tiled group should build aggregate modes on top of the member mode lists.

### Tiled Aggregate Modes

A tiled aggregate mode maps one logical mode to one mode per tile.

For a 2x1 display with two 2560x2880 tiles:

```text
Logical mode: 5120x2880@60

Left tile:    2560x2880@60
Right tile:   2560x2880@60
```

The group should match tile modes by:

- tile-sized resolution
- refresh rate
- flags, when relevant
- preferred status, when available

### Untiled Modes

Some tiled displays can also run in a single-link lower-resolution mode.

Those should be represented as logical modes where only the main tile has an active backend mode and the other tile outputs are disabled or have no CRTC assignment.

KWin should not delete modes from secondary tiles to make this work. The group should decide which member outputs participate in each aggregate mode.

## Configuration Strategy

Configuration should be accepted at the logical group level and expanded to backend output changes in one place.

For a tiled group:

- enabling the logical output enables all tile members needed by the selected aggregate mode
- disabling the logical output disables all tile members
- transform applies to the group and is translated into per-tile rendering/layout behavior
- scale applies once to the logical display
- position applies once to the aggregate display
- DPMS should apply to all active members
- color management should be applied consistently across active members
- brightness should either apply to all members or be blocked if the hardware cannot make that coherent

The important rule: secondary tile user configuration should not survive as independent policy.

If old config files contain entries for secondary tiles, KWin should normalize or ignore them after the group is recognized.

## Output Management And KScreen

Only the grouped logical output should be exposed as configurable.

Secondary tile outputs should not appear as normal independent displays in KScreen or Wayland output-management.

At minimum:

- secondary tile UUIDs should resolve to the group or primary tile when used as references
- mirroring from or to a secondary tile should be normalized or rejected
- clients should not be able to independently position, rotate, enable, or disable a secondary tile

The cleanest user-visible model is:

```text
iMac Display
5120x2880
```

not:

```text
iMac Display Left
iMac Display Right
```

## Compositor Strategy

KWin can keep rendering per backend output, but the viewport calculation must come from the group.

The compositor should ask:

```text
For this logical tiled output and this backend member, what source rectangle
of the logical output should be rendered into this backend output?
```

That helper must be transform-aware.

For normal orientation, a 2x1 display has:

```text
left tile:  x=0,    y=0, width=2560, height=2880
right tile: x=2560, y=0, width=2560, height=2880
```

For 90, 180, 270, and flipped transforms, the tile source rectangles must be recomputed according to the transformed aggregate layout.

This mirrors Mutter's `calculate_tile_coordinate()` idea, but adapts it to KWin's current per-output scene view and viewport model.

## Suggested Implementation Plan

### Step 1: Passive Tile Metadata

Add `TileInfo` to `BackendOutput` without changing behavior.

DRM and virtual backends fill it in. Existing output behavior remains unchanged.

### Step 2: Tile Group Builder

Create a grouping pass that collects backend outputs by `(backend/GPU, groupId)`.

Validated groups become `TiledOutputGroup` objects. Invalid groups are ignored and their members remain normal outputs.

### Step 3: Logical Output Creation

Create one `LogicalOutput` for each valid tiled group.

Normal outputs continue to create one logical output per backend output.

### Step 4: Config Expansion

Move tiled config behavior into the group.

The output configuration store should store logical group configuration and expand it to backend output changes through a single helper.

### Step 5: Output Management

Expose only the logical group output.

Normalize legacy or secondary tile references.

### Step 6: Compositor Viewports

Replace ad hoc tile division with group-provided per-member source rectangles.

Add transform-aware tests before enabling rotation as supported.

### Step 7: Aggregate Modes

Move from primary-tile modes to real aggregate modes.

This can come after basic grouping if the first target is only the iMac 5K native tiled mode, but it should be part of the final design.

## Tests

The minimum test set should include:

- valid 2x1 tiled display becomes one logical output
- incomplete 2x1 group falls back to independent outputs
- duplicate tile position falls back
- out-of-range tile position falls back
- same tile group id on different GPU/backend namespaces does not group
- disabling the logical output disables all active tile members
- enabling the logical output enables all needed tile members
- secondary tile is not independently exposed through output-management
- mirroring a secondary tile UUID normalizes to the group or is rejected
- 90 degree rotation renders both tiles in the correct logical positions
- 180 degree rotation renders both tiles in the correct logical positions
- 270 degree rotation renders both tiles in the correct logical positions
- flipped transforms do not swap or overlap tiles incorrectly
- tiled native mode maps to one mode per member
- untiled fallback mode activates only the main tile when supported

## Advantages

- Fits KWin's current architecture better than a full monitor-manager rewrite.
- Keeps physical backend outputs available for DRM, rendering, and diagnostics.
- Avoids special casing scattered across unrelated systems.
- Allows invalid tile groups to degrade safely.
- Avoids destructive connector mode filtering.
- Gives output-management a clean single-display model.
- Can be implemented incrementally.

## Risks

- Some existing code may still assume one logical output means one backend output.
- The transition period may require compatibility helpers.
- Aggregate modes need careful design to avoid duplicating too much of Mutter's monitor-mode machinery.
- Color, HDR, brightness, VRR, and presentation timing across tiles need explicit policy.
- Atomic commits across multiple tile members may still expose backend limitations.

## Recommendation

This is the best starting point for KWin.

It is a fresh architecture, but not a maximal rewrite. It gives KWin the missing abstraction that the current tiled display patches are trying to approximate indirectly.

The current Xaver Hugl branch is still useful as reconnaissance: it identifies where KWin currently lacks tile-aware behavior. But the final implementation should move those behaviors behind a real group object instead of continuing to add `primaryTileGroupOutput()`-style special cases.
