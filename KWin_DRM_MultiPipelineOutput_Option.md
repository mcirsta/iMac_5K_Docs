# KWin Tiled Display Option B: DRM Multi-Pipeline Output

## Summary

This is the more radical option.

Instead of representing a tiled display as multiple `BackendOutput` objects grouped above the backend, the DRM backend would expose one `DrmOutput` for the whole tiled display. Internally, that one output would be backed by multiple DRM connectors, CRTCs, planes, and pipelines.

In other words, KWin's upper layers would see one output from the beginning.

```text
DRM connector  DRM connector
     |              |
 DRM pipeline   DRM pipeline
      \            /
       \          /
      DrmOutput for tiled display
              |
        LogicalOutput
```

This is attractive because it hides tiled displays at the lowest possible layer. It is also much more invasive.

## Goal

Make tiled displays look like one physical output to the rest of KWin.

For an iMac 5K-style display, the DRM backend would expose one `DrmOutput` named something like `iMac Display`, even though the kernel reports two connectors.

Upper layers would not need to know that the display is tiled, except perhaps for diagnostics.

## Why This Is Attractive

This option gives the cleanest external model:

- one backend output
- one logical output
- one output-management object
- one KScreen display
- one config entry
- no secondary tile UUIDs leaking into user-facing APIs

It also matches how users think about the hardware. The display is one panel, not two monitors.

## Why This Is Hard In KWin

KWin's DRM backend is built around outputs, pipelines, layers, render loops, and commits that usually map one visible output to one scanout path.

A multi-pipeline `DrmOutput` would need to coordinate multiple scanout paths while presenting itself as one output.

That affects:

- output discovery
- output lifetime
- mode selection
- atomic commits
- output layers
- render loops
- cursor planes
- direct scanout
- color management
- brightness and backlight
- DPMS
- VRR
- presentation feedback
- failure handling

This is not just a tile grouping patch. It is a backend architecture change.

## High-Level Model

The DRM backend would detect a valid tile group during connector discovery.

If the tile group is complete and valid:

- create one multi-pipeline `DrmOutput`
- attach all tile connectors or pipelines to that output
- expose aggregate modes
- hide the individual tile connectors from the rest of KWin

If the tile group is invalid:

- fall back to individual `DrmOutput` objects
- do not create the aggregate output

## Tile Validation

Validation would still need to be Mutter-like:

- same DRM device
- same tile group id
- same grid dimensions
- same tile dimensions
- positions inside the grid
- no duplicate tile positions
- full grid coverage

This option does not remove the need for strict validation. It only changes where the validated group becomes one output.

## Mode Strategy

The multi-pipeline `DrmOutput` would expose aggregate modes.

For a 2x1 display:

```text
Exposed mode:
5120x2880@60

Internal pipeline modes:
left connector:  2560x2880@60
right connector: 2560x2880@60
```

The `DrmOutput` would need a mode object that maps:

```text
aggregate mode -> per-pipeline mode assignments
```

Untiled fallback modes could be represented as aggregate modes where only one internal pipeline is active.

## Rendering Strategy

There are two possible rendering models.

### One Big Render, Split At Scanout

KWin renders one aggregate 5120x2880 image.

The DRM backend then splits that image into per-pipeline buffers or source rectangles.

This is conceptually clean, but it may require:

- buffer slicing or multiple framebuffer views
- careful plane source rectangle handling
- damage splitting
- constraints around direct scanout

### Per-Tile Rendering Behind One Output

The multi-pipeline `DrmOutput` owns multiple internal output layers, one per tile.

Upper layers still see one output, but the DRM backend asks the compositor for tile-sized render targets internally.

This may fit existing KWin rendering pieces better, but it leaks complexity into the output/layer interface.

Either way, the backend must know each tile's source rectangle in aggregate output space and must handle transforms correctly.

## Atomic Commit Strategy

The biggest benefit of this option is also one of the hardest parts: commits for the tiled display should be coordinated.

For a real tiled panel, left and right halves should update together.

The multi-pipeline output would need to:

- test all pipeline changes together
- commit all tile pipelines together when possible
- roll back or fail as one logical output
- avoid one half updating while the other half fails

If KWin's existing atomic machinery already supports commits containing multiple pipelines, this may be manageable. If not, this option gets expensive quickly.

## Cursor Strategy

Hardware cursor handling becomes more complex.

Possible policies:

- use software cursor for tiled outputs initially
- use a cursor plane only on the tile that currently contains the cursor
- split cursor visibility across tiles when the cursor crosses a boundary

The first implementation should probably use software cursor for tiled outputs unless the existing cursor pipeline can handle this cleanly.

## Direct Scanout Strategy

Direct scanout should initially be conservative.

For tiled outputs, direct scanout should only be enabled when:

- the buffer covers the full aggregate output
- source rectangles can be assigned correctly per tile
- transforms are compatible
- color pipeline state is compatible
- both tile pipelines can accept the scanout plan

Otherwise, fall back to composited rendering.

## Color, Brightness, And HDR

A multi-pipeline output must maintain visual consistency across tiles.

Policies need to be explicit:

- color transforms should be applied consistently to every tile pipeline
- HDR metadata should be attached consistently where supported
- brightness/backlight should be controlled as one logical display
- per-tile differences should either be hidden or treated as a backend limitation

Partial support can create very visible seams between tiles.

## Output Management And KScreen

This option gives the cleanest behavior here.

Because the DRM backend exposes one output, KWin's upper layers naturally expose one output too.

There should be:

- one UUID
- one display name
- one position
- one scale
- one transform
- one mode list
- one enabled state
- one DPMS state

Legacy config matching still needs care. If users previously had config entries for the separate tile connectors, KWin should migrate or ignore those entries once the aggregate output appears.

## Failure And Fallback

The backend should be able to fall back safely.

Fallback cases:

- missing tile
- duplicate tile location
- inconsistent tile metadata
- no matching per-tile modes
- failed atomic test for aggregate mode
- one connector disappears on hotplug

When fallback happens, KWin can expose individual physical outputs. That is not pretty, but it is debuggable and safer than exposing a broken aggregate display.

## Suggested Implementation Plan

### Step 1: Passive DRM Tile Group Discovery

Teach the DRM backend to collect connectors into candidate tile groups, scoped by DRM device.

Do not change exposed outputs yet.

### Step 2: Backend Validation

Validate complete tile groups inside the DRM backend.

Invalid groups continue through the current one-connector-one-output path.

### Step 3: Internal Multi-Pipeline Object

Create a private object that owns:

- member connectors
- member pipelines
- tile geometry
- aggregate modes
- per-mode pipeline assignments

At this stage, it can exist without being exposed as a public `DrmOutput`.

### Step 4: Expose One DrmOutput

Create one `DrmOutput` for a validated tile group.

Individual tile outputs should not be exposed to `Workspace`.

### Step 5: Basic Native Mode

Support only the primary tiled native mode first.

Disable direct scanout and hardware cursor for tiled outputs if needed.

The first success criterion is stable composited 5120x2880 output.

### Step 6: Transforms

Add transform-aware tile source rectangles.

Verify 90, 180, 270, and flipped transforms.

### Step 7: Aggregate Mode List

Add full aggregate mode generation:

- tiled modes
- untiled fallback modes
- preferred mode selection
- refresh matching

### Step 8: Re-enable Optimizations

Only after correctness:

- hardware cursor
- direct scanout
- damage optimization
- VRR, if meaningful

## Tests

The minimum test set should include:

- valid tile group produces one `DrmOutput`
- invalid tile group exposes individual outputs
- same tile group id on another DRM device does not group
- native tiled mode assigns correct per-pipeline modes
- untiled fallback mode activates only the intended pipeline
- atomic test failure prevents partial aggregate enable
- hot-unplug of one tile tears down the aggregate output safely
- software cursor works across tile boundary
- direct scanout is disabled or correctly split
- 90 degree transform maps source rectangles correctly
- 180 degree transform maps source rectangles correctly
- 270 degree transform maps source rectangles correctly
- output-management sees one output
- legacy secondary-tile config does not corrupt the aggregate output

## Advantages

- Cleanest user-visible model.
- Upper KWin layers mostly do not need tile-specific behavior.
- Avoids secondary tile leakage into KScreen and Wayland output-management.
- Potentially gives better atomicity for tiled panel updates.
- Keeps tile knowledge close to the DRM concepts that produce it.

## Risks

- Large backend refactor.
- High chance of touching fragile DRM paths.
- More difficult to review and merge incrementally.
- Requires careful handling of multiple pipelines under one output.
- Cursor and direct scanout become significantly more complex.
- Harder to reuse for non-DRM backends and virtual tests.
- Bugs may be harder to diagnose because the physical connectors are hidden earlier.

## Recommendation

This is probably not the best first implementation path.

It may be the cleanest end-state for the DRM backend, especially if KWin wants atomic multi-pipeline outputs as a general concept. But for tiled display support specifically, it has a much larger blast radius than the `TiledOutputGroup` option.

A practical path would be:

1. Implement Option A first.
2. Keep the group abstraction clean enough that the DRM backend can later collapse a valid group into one multi-pipeline output.
3. Revisit this option after the behavior, tests, and policy are clear.

That way KWin gets correct tiled display support sooner, while still leaving room for a deeper DRM backend redesign later.
