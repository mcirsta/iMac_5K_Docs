# KWin TiledOutputGroup Implementation Plan

## Purpose

This document describes how to implement Option A from `KWin_TiledOutputGroup_Option.md`: introduce a first-class tiled output group in KWin and move tiled-display behavior behind that abstraction.

The intended result is that an iMac 5K-style tiled panel is treated as one user-visible output, while KWin still keeps access to the individual backend outputs that represent the physical tiles.

The plan is written for KWin's current architecture, where:

- `BackendOutput` represents a physical/backend output.
- `LogicalOutput` represents the output exposed to window management and normal Wayland output globals.
- `OutputConfiguration` and `OutputDeviceV2Interface` are still mostly `BackendOutput` based.
- The compositor renders one `SceneView` per backend output/render loop.

## Starting Branch

This plan assumes a fresh KWin branch from `master`, not continued development on Xaver Hugl's prototype branch.

The working branch has been created as:

```bash
cd /home/marius/proj/5K/kwin
git fetch origin master
git switch -c work/tiled-output-group origin/master
```

At the time this document was updated, the branch starts at:

```text
f26ad19268 Remove Window::checkUnrestrictedInteractiveMoveResize()
```

The prototype branch should remain available as reference material only:

```text
work/tiled-displays
```

Starting fresh means:

- do not keep the prototype's `Workspace::primaryTileGroupOutput()` API
- do not keep the prototype's global `int groupId` map
- do not keep the prototype's destructive connector mode filtering
- do not keep the prototype's compositor tile math
- do not keep tiled behavior in `OutputConfigurationStore::applyAdjustments()` (this method is `applyMirroring()` in current master)

Code may still be reimplemented from the prototype when the shape is right, especially TILE parsing and virtual backend test support.

## Design Target

```text
BackendOutput          BackendOutput
     |                      |
     +----------+-----------+
                |
        TiledOutputGroup
                |
          LogicalOutput
```

For a valid 2x1 tiled panel:

- KWin creates two physical `BackendOutput` objects.
- KWin creates one `TiledOutputGroup`.
- KWin creates one `LogicalOutput`.
- Wayland `wl_output` and `xdg_output` expose one output.
- Output-management/KScreen expose one configurable output.
- The compositor still renders to both physical backend outputs, but each backend output receives the correct viewport of the single logical output.

## Relationship To Mutter

Mutter already has this shape:

```text
MetaOutput + MetaOutput
        |
MetaMonitorTiled
        |
MetaLogicalMonitor
```

KWin does not have a direct `MetaMonitor` equivalent. `TiledOutputGroup` is the KWin-shaped replacement for that missing layer.

The implementation should copy these Mutter principles:

- Parse TILE metadata at the physical output layer.
- Group only outputs from the same GPU/backend namespace.
- Validate the full tile grid before grouping.
- Fall back to normal outputs if validation fails.
- Build aggregate modes without deleting physical connector modes.
- Expand one logical configuration into per-tile backend configuration centrally.
- Compute per-tile rendering rectangles with transform awareness.

## Non-Goals

This plan does not attempt to implement the radical DRM multi-pipeline output design.

It also does not require the first merge request to support every display-management feature. The first useful implementation can support native tiled mode only, with mirroring and untiled fallback modes added later behind the same abstraction.

## Prototype Problems To Avoid

The current prototype has the right instinct, but the wrong ownership model.

Problem areas:

- `Workspace` stores `std::unordered_map<int, BackendOutput *> m_tileGroupOutputs`.
- The tile key is only `groupId`, but DRM tile group IDs are scoped per DRM device.
- `Workspace::primaryTileGroupOutput()` is called from unrelated places.
- `OutputConfigurationStore::applyAdjustments()` contains tiled behavior (this method is `applyMirroring()` in current master).
- DRM connector mode filtering removes non-tile-sized modes destructively.
- `Compositor::assignOutputLayers()` contains ad hoc tile viewport logic and admits rotation is not handled.
- Output-management still exposes physical backend outputs too directly.
- Config application does not have a single "expand group config to member configs" step.

The new implementation should delete those patterns rather than keep layering fixes on top of them.

## New Files

Recommended new files:

```text
src/core/tiledoutputgroup.h
src/core/tiledoutputgroup.cpp
```

Add them to `src/CMakeLists.txt`.

If reviewers prefer a more general name, use:

```text
src/core/outputgroup.h
src/core/outputgroup.cpp
```

with `TiledOutputGroup` as the first concrete group type.

For a first implementation, `tiledoutputgroup.*` is more direct and easier to review.

## Backend Tile Metadata

`BackendOutput::TileInfo` exists in the prototype branch, but not in a fresh branch from `master` (confirmed absent on the `work/tiled-output-group` base).

The prototype shape is:

```cpp
struct TileInfo
{
    int groupId;
    QSize completeSizeInTiles;
    QPoint tileLocation;
    QSize tileSizeInPixels;
    QSize completeSizeInPixels;
};
```

The fresh implementation should introduce this metadata deliberately, with one additional concept: a group namespace.

DRM tile group IDs are per DRM device, not global. KWin must not group outputs from different GPUs just because they both report `groupId == 1`.

Recommended shape:

```cpp
struct TileInfo
{
    quint64 groupScopeId;
    int groupId;
    QSize completeSizeInTiles;
    QPoint tileLocation;
    QSize tileSizeInPixels;
    QSize completeSizeInPixels;
};
```

`groupScopeId` is runtime-only and assigned by the backend.

For DRM:

- assign one stable runtime scope per `DrmGpu` or DRM device
- all connectors from the same DRM device get the same scope
- connectors from different DRM devices get different scopes

For virtual tests:

- assign one scope per virtual backend instance, unless a test explicitly needs multiple fake GPUs

Do not store `groupScopeId` in persistent config.

## TiledOutputGroup API

Proposed header sketch:

```cpp
class TiledOutputGroup
{
public:
    struct Key
    {
        quint64 scopeId;
        int groupId;

        auto operator<=>(const Key &) const = default;
    };

    struct Member
    {
        BackendOutput *output;
        QPoint tileLocation;
        QSize tileSize;
        Rect nativeRect;
    };

    enum class ValidationError {
        Empty,
        MissingTileInfo,
        MixedGroupId,
        MixedScope,
        MixedGridSize,
        MixedTileSize,
        InvalidTileLocation,
        DuplicateTileLocation,
        IncompleteGrid,
        MissingPrimaryTile,
    };

    static std::expected<std::unique_ptr<TiledOutputGroup>, ValidationError>
    create(const QList<BackendOutput *> &outputs);

    Key key() const;
    const QList<Member> &members() const;
    QList<BackendOutput *> backendOutputs() const;

    bool contains(BackendOutput *output) const;
    BackendOutput *primaryOutput() const;
    bool isPrimary(BackendOutput *output) const;

    QSize gridSize() const;
    QSize tileSize() const;
    QSize nativeSize() const;

    Rect nativeTileRect(BackendOutput *output) const;
    Rect transformedTileRect(BackendOutput *output, OutputTransform transform) const;
    RectF logicalViewport(BackendOutput *output, const LogicalOutput *logicalOutput) const;

    QString groupUuid() const;
    QString groupName() const;
    QString groupDescription(const QList<BackendOutput *> &visibleOutputs) const;

    void expandConfiguration(OutputConfiguration &config) const;
    void normalizeReplicationSource(OutputConfiguration &config, const QList<BackendOutput *> &allOutputs) const;
};
```

The branch builds with C++23 (`CMAKE_CXX_STANDARD 23` in the top-level `CMakeLists.txt`), and `std::expected` is already used elsewhere in the tree (for example `iccprofile.h`, `session.h`, `drm_backend.cpp`). Use it directly; no fallback result struct is needed.

Consumers test `if (!result)`, read the group via `*result` / `result->`, and read the error via `result.error()`.

## Group Key

Group by:

```text
(TileInfo::groupScopeId, TileInfo::groupId)
```

Never group by `groupId` alone.

This directly replaces:

```cpp
std::unordered_map<int, BackendOutput *> m_tileGroupOutputs;
```

with:

```cpp
std::vector<std::unique_ptr<TiledOutputGroup>> m_tiledOutputGroups;
QHash<BackendOutput *, TiledOutputGroup *> m_tiledOutputGroupByOutput;
QHash<TiledOutputGroup::Key, TiledOutputGroup *> m_tiledOutputGroupByKey;
```

The exact containers can change. The important part is that lookup by member output is cheap and that the group key includes the backend/GPU namespace.

## Validation Rules

Only create a group if all of these are true:

- candidate list is not empty
- every candidate has `TileInfo`
- all candidates have the same `groupScopeId`
- all candidates have the same `groupId`
- all candidates have the same `completeSizeInTiles`
- all candidates have the same `tileSizeInPixels`
- every `tileLocation.x()` is in `[0, completeSizeInTiles.width())`
- every `tileLocation.y()` is in `[0, completeSizeInTiles.height())`
- there are no duplicate tile locations
- number of candidates equals `width * height`
- tile `(0, 0)` exists
- `completeSizeInPixels == QSize(tileWidth * gridWidth, tileHeight * gridHeight)`

If validation fails:

- log a useful debug message
- do not create a group
- leave those outputs as normal outputs

This should be implemented in `TiledOutputGroup::create()` rather than in `Workspace`.

## Primary Output

The primary output should be the member at tile location `(0, 0)`.

Reasons:

- stable UUID behavior
- matches Mutter's "main tile" idea
- avoids random ordering changes
- matches existing prototype expectations

The primary output is a compatibility anchor, not the owner of tiled behavior.

Code should avoid spelling this as "primary tile group output" outside the group implementation.

Use:

```cpp
group->primaryOutput()
group->isPrimary(output)
workspace()->tiledOutputGroup(output)
```

not:

```cpp
workspace()->primaryTileGroupOutput(...)
```

## LogicalOutput Changes

Current prototype:

```cpp
explicit LogicalOutput(BackendOutput *backendOutput, std::optional<int> tileGroupId = std::nullopt);
```

Replace with:

```cpp
explicit LogicalOutput(BackendOutput *backendOutput, TiledOutputGroup *tiledGroup = nullptr);
```

Add:

```cpp
TiledOutputGroup *tiledOutputGroup() const;
bool isTiled() const;
QList<BackendOutput *> backendOutputs() const;
```

Behavior:

- `backendOutput()` still returns the primary backend output for compatibility.
- `backendOutputs()` returns all group members for tiled outputs, otherwise one output.
- `uuid()` initially returns the primary backend output UUID.
- `name()` initially returns the primary backend output name.
- `modeSize()` for tiled output returns the aggregate native size.
- `pixelSize()` continues to return transformed aggregate size.
- `physicalSize()` can initially return primary physical size, but the better long-term value is the complete panel physical size if EDID exposes it consistently.

This keeps the patch smaller than replacing every `backendOutput()` user at once.

## Workspace Changes

### New Helpers

Add:

```cpp
TiledOutputGroup *Workspace::tiledOutputGroup(BackendOutput *output) const;
TiledOutputGroup *Workspace::tiledOutputGroup(const LogicalOutput *output) const;
bool Workspace::isNonPrimaryTiledOutput(BackendOutput *output) const;
BackendOutput *Workspace::representativeOutput(BackendOutput *output) const;
```

`representativeOutput()` returns:

- group primary output for a member of a valid tiled group
- the original output otherwise

### Rebuild Order

Current code (both master and the prototype) does:

```cpp
void Workspace::slotOutputBackendOutputsQueried()
{
    updateOutputConfiguration();
    updateOutputs();
}
```

This is backwards for groups because config generation needs current group knowledge.

Change to:

```cpp
void Workspace::slotOutputBackendOutputsQueried()
{
    rebuildTiledOutputGroups();
    updateOutputConfiguration();
    updateOutputs();
}
```

Also call `rebuildTiledOutputGroups()` before any path that applies a fresh backend-output list after hotplug.

### Group Rebuilding

Implement:

```cpp
void Workspace::rebuildTiledOutputGroups()
{
    m_tiledOutputGroups.clear();
    m_tiledOutputGroupByOutput.clear();

    const QList<BackendOutput *> outputs = kwinApp()->outputBackend()->outputs();

    QHash<TiledOutputGroup::Key, QList<BackendOutput *>> candidates;
    for (BackendOutput *output : outputs) {
        if (output->isNonDesktop() || !output->tileInfo()) {
            continue;
        }
        candidates[keyFor(output)].append(output);
    }

    for (const QList<BackendOutput *> &candidateOutputs : candidates) {
        auto result = TiledOutputGroup::create(candidateOutputs);
        if (!result) {
            qCDebug(KWIN_TILED_OUTPUTS) << "Ignoring invalid tiled output group" << result.error();
            continue;
        }

        TiledOutputGroup *group = result->get();
        for (BackendOutput *member : group->backendOutputs()) {
            m_tiledOutputGroupByOutput.insert(member, group);
        }
        m_tiledOutputGroups.push_back(std::move(*result));
    }
}
```

Use the actual Qt/KWin container style preferred by the surrounding code.

### updateOutputs()

`Workspace::updateOutputs()` should no longer validate tiles inline.

Instead:

1. Ask `rebuildTiledOutputGroups()` to maintain group state.
2. Build `newBackendOutputs` from:
   - all normal enabled managed outputs
   - only the primary output for each valid tiled group
3. Skip non-primary group members.
4. Create `LogicalOutput(primary, group)` for group primaries.
5. Use `group->nativeSize()` for logical mode size.
6. Use `group->groupDescription(...)` for display text when useful.

Pseudo-flow:

```cpp
for (BackendOutput *output : availableOutputs) {
    if (!wantsToManage(output)) {
        continue;
    }

    if (TiledOutputGroup *group = tiledOutputGroup(output)) {
        if (!group->isPrimary(output)) {
            continue;
        }
    }

    newBackendOutputs.append(output);
}
```

When setting logical output data:

```cpp
if (auto group = tiledOutputGroup(output)) {
    logical->setData(output->position() - topLeftMostCoordinates,
                     group->nativeSize(),
                     group->refreshRate(),
                     output->transform(),
                     output->scale(),
                     group->groupDescription(newBackendOutputs));
} else {
    logical->setData(... existing behavior ...);
}
```

`group->refreshRate()` can initially return the primary output refresh rate. Once aggregate modes exist, it should return the selected aggregate mode refresh rate.

## OutputConfiguration Expansion

This is the most important part.

Before sending config to `OutputBackend::applyOutputChanges()`, KWin must expand group-level changes to every group member.

Current path (master; simplified — the real function also adjusts auto-brightness curves for `Source::User`, and after apply calls `updateOutputs()` and `m_outputConfigStore->storeConfig(backendOutputs, ...)`):

```cpp
Workspace::applyOutputConfiguration(OutputConfiguration &config)
    assignBrightnessDevices(config)
    m_outputConfigStore->applyMirroring(config, backendOutputs)
    force DPMS
    outputBackend()->applyOutputChanges(config)
    updateOutputs()
    m_outputConfigStore->storeConfig(backendOutputs, ...)
```

Recommended path:

```cpp
Workspace::applyOutputConfiguration(OutputConfiguration &config)
    assignBrightnessDevices(config)
    normalizeTiledOutputConfiguration(config)
    m_outputConfigStore->applyMirroring(config, backendOutputs)
    expandTiledOutputConfiguration(config)
    force DPMS
    outputBackend()->applyOutputChanges(config)
    updateOutputs()
    m_outputConfigStore->storeConfig(backendOutputs, ...)
```

Note: master's method is `applyMirroring()`. The prototype renamed it to `applyAdjustments()` and folded tiled logic into it; that rename and the tile additions were never merged, so on the fresh branch the method is still `applyMirroring()` and that is the name to target. Because `storeConfig()` runs after apply on the full backend-output list, the per-group storing rules in the OutputConfigurationStore section must coordinate with that call.

The exact order with `applyMirroring()` may need tuning, but the final invariant must be:

```text
By the time OutputBackend receives OutputConfiguration,
every active tile member has an explicit coherent OutputChangeSet.
```

### normalizeTiledOutputConfiguration()

This step should:

- map secondary tile UUIDs to the group primary UUID
- reject or normalize attempts to configure secondary tiles directly
- ensure group-level config lives on the primary output changeset
- resolve mirroring references to grouped outputs

For example:

```text
normal output mirrors secondary tile UUID
    -> mirror primary/group UUID instead

secondary tile has position/scale/transform change
    -> ignore or copy to primary changeset, depending on source

tiled group member mirrors another member of same group
    -> reject
```

### expandTiledOutputConfiguration()

For each `TiledOutputGroup`:

1. Read the primary output changeset.
2. Determine group desired state:
   - enabled
   - position
   - scale
   - transform
   - mode
   - DPMS
   - replication source
   - color/HDR/brightness policy
3. Write explicit changesets for every member output.

Basic expansion:

```cpp
void TiledOutputGroup::expandConfiguration(OutputConfiguration &config) const
{
    auto primary = config.changeSet(primaryOutput());

    const bool enabled = primary->enabled.value_or(primaryOutput()->isEnabled());
    const QPoint pos = primary->pos.value_or(primaryOutput()->position());
    const double scale = primary->scaleSetting.value_or(primaryOutput()->scaleSetting());
    const OutputTransform transform = primary->transform.value_or(primaryOutput()->transform());

    for (const Member &member : members()) {
        auto change = config.changeSet(member.output);

        change->enabled = enabled;
        change->pos = pos;
        change->scaleSetting = scale;
        change->scale = scale;
        change->transform = transform;

        // DPMS, color, HDR, brightness, and mode handling described below.
    }
}
```

All members should get the same logical position. The compositor uses the group tile rectangle to decide which part of the logical output each member renders. Hidden secondary members should not contribute their own global layout.

### DPMS

DPMS should apply to every active member.

If the primary/group is off:

```cpp
memberChange->dpmsMode = BackendOutput::DpmsMode::Off;
```

If on:

```cpp
memberChange->dpmsMode = BackendOutput::DpmsMode::On;
```

### Scale

All members should get the same effective scale.

The group scale is the logical display scale, not a per-tile scale.

### Transform

All members should get the same transform at the backend state level.

The group is responsible for computing each member's transformed viewport. This avoids the bug where only the primary tile rotates.

### Mode

There are two stages.

#### Stage 1: Native Tiled Mode Only

For the first useful implementation:

- keep each member on its existing tile-sized current mode
- expose the aggregate logical size through `LogicalOutput`
- do not implement full aggregate mode selection yet
- reject mode changes that cannot map to the current tile modes

This is enough to prove grouping, output-management hiding, config expansion, and compositor rendering.

#### Stage 2: Aggregate Modes

Add a group-level mode model:

```cpp
class TiledOutputMode : public OutputMode
{
public:
    enum class Kind {
        Tiled,
        Untiled,
    };

    Kind kind() const;
    std::optional<OutputModeline> memberMode(BackendOutput *output) const;
};
```

For a tiled aggregate mode:

```text
group mode:       5120x2880@60
left member mode: 2560x2880@60
right member:     2560x2880@60
```

For an untiled fallback mode:

```text
group mode:       3840x2160@60
primary member:   3840x2160@60
secondary member: disabled or no mode assignment
```

KWin should not remove modes from physical outputs to achieve this.

Mode generation should:

- use tile-sized modes from the primary tile as references
- find matching tile-sized modes on every member by size, refresh, and flags
- mark an aggregate mode preferred only if every member's mode is preferred, or choose the best refresh fallback
- generate untiled modes from the main tile without requiring matching modes on other members

This mirrors Mutter's approach.

### Brightness And Color

Initial policy:

- color profile, HDR, wide gamut, EDR, max BPC, SDR luminance, and color power tradeoff should copy from the group primary to every active member
- brightness should copy to every member that has the same brightness device or independent brightness control
- if hardware exposes only one brightness device for the panel, assign that device to the group primary and avoid double-applying it

This area needs hardware testing. The implementation should prefer consistent output over exposing per-tile control.

### Replication/Mirroring

Mirroring should be implemented after native grouping works.

Rules:

- a group cannot mirror one of its own members
- a member cannot be configured to mirror another member directly
- if another output mirrors any member UUID, normalize to the group primary UUID
- if the tiled group mirrors another output, every active member mirrors the same source but receives a member-specific render offset
- if another output mirrors the tiled group, the source pixel size is the aggregate group pixel size

The helper should use group geometry:

```cpp
QSize TiledOutputGroup::sourcePixelSize(OutputConfiguration &config) const;
QPoint TiledOutputGroup::memberMirrorOffset(BackendOutput *member,
                                            QSize sourcePixelSize,
                                            double sourceScale,
                                            QSize destinationPixelSize,
                                            double destinationScale,
                                            OutputTransform transform) const;
```

Avoid preserving the current unrotated tile math. The mirror offset must use the transformed tile rectangle.

## OutputConfigurationStore

The config store should not treat secondary tiles as independent user-configurable outputs once a group is valid.

### Querying

When `queryConfig()` receives backend outputs:

- register all physical outputs so UUIDs exist
- build relevant config targets from:
  - normal outputs
  - primary outputs of valid tiled groups
- exclude secondary tiled members from setup matching

This avoids saved configs with two independent entries for the same physical panel.

### Storing

When `storeConfig()` stores a setup:

- store one entry for the group primary
- do not store secondary members as independent outputs
- preserve enough identity so the primary tile can be matched later

For first implementation, using the primary tile's existing EDID/UUID matching is acceptable.

Longer term, add a group identity:

```text
tiled-group:<primary-edid-hash>:<grid-width>x<grid-height>:<tile-size>
```

Do not include runtime `groupScopeId` in persistent config.

### Existing Secondary Tile Config

If an old config contains secondary tile entries:

- ignore them when a valid group exists
- optionally migrate useful settings to the primary/group entry
- never let a stale secondary position or transform override the group

## Wayland Output Exposure

KWin has two output exposure paths:

- `wl_output`/`xdg_output`, based on `LogicalOutput`
- KDE output-management devices, based on `BackendOutput`

`wl_output` is mostly solved once `Workspace::updateOutputs()` creates one `LogicalOutput` for the group.

Output-management needs explicit work.

## OutputDeviceV2 / KScreen Strategy

Current `OutputDeviceRegistryV2` offers `BackendOutput *` directly.

For tiled groups:

- offer only the group primary output
- withdraw or never offer secondary members
- make the primary output device report group-level geometry and modes
- accept configuration on the primary output and expand it to members before backend apply

### Practical First Patch

Add a sync function owned by `WaylandServer` or `Workspace`:

```cpp
void WaylandServer::syncOutputDeviceRegistry();
```

It should offer:

- all non-placeholder, non-non-desktop normal backend outputs
- only primary outputs for valid tiled groups

It should withdraw:

- secondary members of valid tiled groups
- outputs no longer present

This is better than blindly offering every backend output in `WaylandServer::handleOutputAdded()`.

### Group-Aware OutputDeviceV2Interface

For the first stage, `OutputDeviceV2Interface` can continue to hold a `BackendOutput *`, but it should consult a group override when the handle is a tiled group primary.

Add internal helpers:

```cpp
QSize outputDeviceModeSize(BackendOutput *handle);
QList<std::shared_ptr<OutputMode>> outputDeviceModes(BackendOutput *handle);
std::shared_ptr<OutputMode> outputDeviceCurrentMode(BackendOutput *handle);
QPoint outputDevicePosition(BackendOutput *handle);
QSize outputDevicePhysicalSize(BackendOutput *handle);
```

These helpers return group-level data for a group primary and normal backend data otherwise.

Avoid pulling `Workspace` too deeply into `wayland/outputdevice_v2.cpp`. If needed, add a small query API:

```cpp
OutputPresentationInfo Workspace::presentationInfoForOutputDevice(BackendOutput *output) const;
```

where `OutputPresentationInfo` contains:

- position
- physical size
- modes
- current mode
- transform
- scale
- name
- description

Longer term, replace `BackendOutput *` with a real output-management model object, but that is not required for the first implementation.

## OutputManagementV2 Validation

Update validation in `src/wayland/outputmanagement_v2.cpp`.

Rules:

- configurations for secondary tile members should not appear because they are not offered
- if a stale client somehow references a secondary member, reject it or normalize it before apply
- mirroring a group member to another member in the same group is invalid
- mirroring a secondary member UUID should normalize to the group primary UUID
- disabling the group primary disables the group
- prevent "all disabled" checks from counting hidden secondary members as separately enabled

The "all disabled" check should answer:

```text
Will there be at least one user-visible output after expansion?
```

not:

```text
Will at least one physical BackendOutput be enabled?
```

## Compositor Integration

Current prototype computes tile viewport in `Compositor::assignOutputLayers()` with unrotated division.

Replace that with:

```cpp
if (TiledOutputGroup *group = logicalOutput->tiledOutputGroup()) {
    view->setViewport(group->logicalViewport(backendOutput, logicalOutput));
} else {
    view->setViewport(logicalOutput->geometryF());
}
```

The actual code must preserve current snapping behavior. The helper should perform the snapping centrally.

### Viewport Algorithm

Inputs:

- logical output geometry
- logical output transform
- logical output scale
- group native size
- member native tile rectangle

Algorithm:

1. Compute the member's untransformed native rectangle:

   ```cpp
   Rect nativeRect(member.tileLocation * member.tileSize, member.tileSize);
   ```

2. Transform that rectangle inside the aggregate native bounds:

   ```cpp
   Rect transformed = logicalOutput->transform().map(nativeRect, group->nativeSize());
   ```

3. Convert from physical pixels to logical coordinates:

   ```cpp
   RectF localLogical = RectF(transformed) / logicalOutput->scale();
   ```

4. Translate to global logical coordinates:

   ```cpp
   RectF globalLogical = localLogical.translated(logicalOutput->geometryF().topLeft());
   ```

5. Snap consistently with the renderer:

   ```cpp
   Rect snapped = globalLogical.scaled(member.output->scale()).rounded();
   return snapped.scaled(1.0 / member.output->scale());
   ```

This uses KWin's existing `OutputTransform::map(rect, bounds)` instead of reimplementing Mutter's transform switch.

### Viewport-Relative Render Path

Computing the correct viewport rect is necessary but not sufficient. A tiled member renders a *sub-region* of the logical output into a tile-sized framebuffer, and several parts of the existing render path assume `viewport == full logical output geometry`. Those assumptions break for tiles and must be made viewport-relative. The prototype branch (`work/tiled-displays`) already solved this; reimplement these changes, since they are orthogonal to the group ownership model and are good cherry-pick/reimplement candidates:

- `Compositor::mapGlobalLogicalToOutputDeviceCoordinates()` and `mapItemToOutputDeviceCoordinates()` currently anchor on `logicalOutput->geometryF().topLeft()`. For a tile that is the *aggregate* origin, so content lands off the tile's framebuffer. Anchor on the member's own viewport top-left instead. The prototype threads the per-tile `SceneView` into these helpers and uses `primaryView->viewport().topLeft()`. This one change fixes ordinary item placement, **direct-scanout target rects** (`prepareDirectScanout`), normal rendering (`prepareRendering`), and **cursor-plane target rects** in one place.
- `createProjectionMatrix()` in `renderviewport.cpp` must handle a viewport rect smaller than the render target, not only the mirroring `renderOffset`. The prototype generalizes the projection so a sub-region viewport scales correctly into the full tile framebuffer.
- The scissor region in `itemrenderer_opengl.cpp` must use `viewport.deviceSize()` rather than `renderTarget.size() - 2 * bufferOffset`, so hardware clipping matches the tile's rendered sub-region.

Without these, `logicalViewport()` returns the right rectangle but the renderer still maps content relative to the aggregate output, producing off-screen or duplicated content per tile. Treat them as part of Phase 4, not a later cleanup. The prototype's draft rotation test exists precisely because its ad-hoc viewport math (the unrotated tile division) never made rotation work end-to-end; the transform-aware `logicalViewport()` plus this viewport-relative path is what finally closes that.

### Render Loop Scheduling

When scheduling repaints after config apply:

Current code schedules:

```cpp
for (LogicalOutput *output : m_outputs) {
    output->backendOutput()->renderLoop()->scheduleRepaint();
}
```

For tiled outputs, schedule every member render loop:

```cpp
for (LogicalOutput *output : m_outputs) {
    for (BackendOutput *backendOutput : output->backendOutputs()) {
        backendOutput->renderLoop()->scheduleRepaint();
    }
}
```

Same principle applies to output layer assignment and removal.

## Output Layers

`Compositor::addOutput()` and `assignOutputLayers()` need to handle all group members.

The compositor should be called for each physical backend output, but with the same `LogicalOutput`:

```cpp
for (BackendOutput *backendOutput : logicalOutput->backendOutputs()) {
    assignOutputLayers(logicalOutput, backendOutput);
}
```

When a logical tiled output is removed:

- remove views for all member backend outputs
- disconnect all member render loops

Do not rely on `logicalOutput->backendOutput()` only.

## Visual Artifacts And Mitigations

The sections above make the *static* geometry correct: no seam gap or overlap, and transform-aware rotation. The remaining artifact risks are temporal and per-CRTC. A tiled panel is two DRM CRTCs, and KWin renders one `SceneView` per backend output and presents each through a per-CRTC `DrmCommitThread`. Mutter already solved the same problems for `MetaMonitorTiled`; the mitigations below follow its approach, adapted to KWin's classes.

Note on hardware: standard DRM/KMS has no portable primitive to hardware-lock two CRTC vblanks together. The best available tool is committing all of a group's member CRTCs in a single atomic `drmModeAtomicCommit`, which arms them in one kernel call. Whether they then complete on the same vblank depends on the transport: tiles carried over one DisplayPort MST link or a dual-link panel usually derive from a common clock and align well, while tiles on independent streams or cables may not. This implementation must not special-case any particular panel; it must keep the software path skew-free, rely on the single atomic commit, and accept that residual hardware skew varies by display. The iMac 5K is one motivating test case, not the design target.

### 1. Presentation Desync / Tear At The Seam

Problem: `Compositor::composite()` runs per `RenderLoop` (one per CRTC), and `DrmPipeline::present()` builds a `DrmAtomicCommit` for a single pipeline and hands it to that pipeline's own `DrmCommitThread`. The two tiles therefore page-flip in two independent commits and can land on different vblanks, tearing at the centerline during motion (video, scrolling, dragging a window across the seam).

How Mutter avoids it: Mutter accumulates all CRTC updates for a device into one `MetaKmsUpdate` (a `crtc_updates` list) via `meta_kms_update_merge_from()`, and issues a single atomic commit per device per frame (`queue_update()` / `meta_kms_impl_device_do_process_update()` in `meta-kms-impl-device.c`). It arms a per-CRTC deadline timer (`CrtcFrame.deadline`, a `timerfd` fired just before vblank) so all views in the commit render as late as possible and are gathered into the same commit. Both tile CRTCs are armed in one kernel call, targeting the same vblank cycle.

KWin mitigation (phased):

- The batching primitive already exists: `DrmAtomicCommit(QList<DrmPipeline *>)` and `DrmPipeline::commitPipelines(QList<DrmPipeline *>, ...)`, used today for modeset (`DrmGpu::maybeModeset`). The per-frame `present()` path is the only per-pipeline part.
- Drive all members of a group from a single lead render loop (the primary's). Do not let each tile schedule independently; when the group repaints, render all member `SceneView`s in the same `composite()` pass.
- In the backend, present the group's member pipelines through one `DrmAtomicCommit` covering both CRTCs, committed on a single commit thread, instead of one `addCommit()` per pipeline. This is the KWin analogue of Mutter's merged per-device update.
- Pragmatic interim: ship native grouping first with independent per-tile flips, and treat the joined-commit work as a later MR. On displays whose tiles share a timing domain the seam will usually already be clean; where it is not, the joined commit is the fix. Add a trace that logs every member's page-flip timestamp so the per-display skew is measurable rather than guessed, and use it to decide whether the joined commit is needed on given hardware. Validate genericness through the virtual backend (multiple scopes, fake tile metadata) rather than tuning to any single panel.

### 2. Cursor Handoff At The Seam

Problem: hardware cursor/overlay layers are per-`RenderLoop` (`m_overlayViews` keyed by render loop). A cursor straddling the seam must appear on both CRTCs' cursor planes at once, each showing its half; if KWin assigns the cursor only to the tile it is "mostly" on, the cursor clips or vanishes at the seam.

How Mutter handles it: the native cursor renderer iterates over *every* view and realizes/sets the HW cursor sprite per CRTC (`meta-cursor-renderer-native.c`, the loop near `meta_renderer_get_views()` calling `realize_cursor_sprite_for_crtc()` per view). Each CRTC positions the same sprite in its own coordinate space; the hardware clips the half outside the CRTC, so the two halves join seamlessly. Mutter falls back to a software cursor for a view only when that CRTC cannot do HW cursors.

KWin mitigation:

- In layer assignment, set the cursor plane on every member whose tile viewport (grown by the cursor hotspot extent) intersects the cursor rectangle, not just the primary. Position each plane in that member's local device coordinates (partially negative or over-edge for the straddling tile). The prototype already provides the mechanism: it threads the per-tile `SceneView` into the cursor layer's `repaintScheduled` handler and computes the cursor target rect with `mapGlobalLogicalToOutputDeviceCoordinates(sceneView, ...)`, i.e. relative to that tile's viewport origin. Reuse that per member (this falls out of the *Viewport-Relative Render Path* change above).
- If a member's plane cannot represent a partially-offscreen cursor on the current hardware, fall back to compositing the cursor into that member's primary plane (software cursor) for frames where the cursor overlaps the seam.
- Test: move the cursor across `x = tileWidth` and assert both members show the cursor (HW plane set on both, or SW fallback) with no gap.

### 3. Damage Coverage Across The Seam

Problem: each `SceneView` intersects scene damage with its own viewport (`SceneView::mapToRenderTarget()` and the viewport intersection in `workspacescene.cpp`). An update spanning the seam must repaint both member views in the same logical frame, or one half goes stale.

How Mutter handles it: damage is tracked on the stage and each stage-view repaints the part intersecting its layout, so a region crossing two views dirties both; because Mutter gathers both into one device commit (point 1), they update together.

KWin mitigation:

- When damage is reported for a tiled `LogicalOutput`, schedule repaint on all member render loops (the plan already does this in the render-loop-scheduling section) and ensure each member `SceneView`'s damage is the global logical damage intersected with that member's viewport, not the whole logical geometry.
- Combined with point 1's single-pass render, a window moving across the seam then repaints both halves coherently.
- Test: damage a rectangle straddling the seam and assert both member framebuffers receive the corresponding sub-rectangle.

### 4. Refresh-Rate Beat Between Tiles (Phase 6)

Problem: if the two tiles run modes with slightly different refresh (for example 59.94 vs 60.00) or different VRR state, they drift and the seam shimmers.

How Mutter handles it: `find_tiled_crtc_mode()` (`meta-monitor.c`) only matches a tile mode whose `refresh_rate`, `refresh_rate_mode` (fixed vs VRR), and `flags` are all *equal* to the reference tile's mode, with no tolerance. `create_tiled_monitor_mode()` then requires one matching crtc mode per output, so an aggregate mode exists only if every tile can run the identical timing.

KWin mitigation (Phase 6):

- In `TiledOutputMode` generation, match member modes to the primary tile's reference mode by exact size, exact refresh, exact flags, and same VRR capability. Reject, do not round.
- Publish an aggregate tiled mode only if every member has an exact match (completeness check), mirroring Mutter's per-output crtc-mode array.
- Keep all members on the same VRR policy; never enable adaptive sync on one tile only.

### 5. Fractional-Scale Fragility And Scale Atomicity

Problem: the viewport helper divides by `logicalOutput->scale()` and re-multiplies by `member.output->scale()`; these are equal only because `expandConfiguration()` gives every member the group scale. If a scale change reaches one member a frame before the other, that frame gets a mis-sized viewport and a momentary seam. Fractional scales also stress the snapping most.

How Mutter handles it: a tiled monitor belongs to one `MetaLogicalMonitor` with a single scale; tiles never carry independent scales.

KWin mitigation:

- In `logicalViewport()` / `viewportDebug()`, assert `qFuzzyCompare(member.output->scale(), logicalOutput->scale())` and treat the group scale as the single source of truth; do not read per-member scale independently.
- Ensure `expandConfiguration()` writes the same scale to every member in the same `OutputConfiguration` so the backend applies them in one commit (ties into point 1), never per tile.
- Add a fractional-scale seam test (for example 1.5 and 1.25) asserting the left tile's right-edge device column equals the right tile's left-edge device column, with no gap or overlap.

## DRM Backend

The DRM backend should:

- parse TILE metadata
- include `groupScopeId`
- keep physical connector mode lists intact
- not delete non-tile-sized modes
- not decide user-visible grouping

Remove destructive mode filtering from `DrmConnector`.

The DRM backend remains responsible for applying per-member changes after `TiledOutputGroup::expandConfiguration()` has produced coherent changesets.

See *Visual Artifacts And Mitigations* point 1: the joined-commit work (presenting a group's member pipelines through one `DrmAtomicCommit` instead of one per-pipeline `addCommit()`) lives here. The batching primitive already exists for modeset; reuse it on the per-frame present path.

## Virtual Backend

The virtual backend should support test tile metadata.

Needed test helpers:

- create output with no tile info
- create output with tile group id, scope id, grid size, tile location, and tile pixel size
- create multiple fake group scopes to test same group id on different scopes

This can be added either to `VirtualOutput` directly or to the integration test helper API.

## Persistence And UUIDs

Initial policy:

- group UUID equals primary output UUID
- secondary output UUIDs remain physical UUIDs but are hidden from output-management when grouped
- stale references to secondary UUIDs normalize to the primary UUID

Longer-term optional policy:

- introduce a stable group UUID derived from EDID and tile topology
- keep primary UUID as compatibility alias

Do not block the first implementation on this. The primary UUID approach is simpler and matches the current prototype's goal of stable identity.

## Hotplug Behavior

Expected behavior:

### Full Group Appears

1. Physical backend outputs appear.
2. Workspace rebuilds groups.
3. Valid group is created.
4. Secondary output devices are hidden/withdrawn.
5. One logical output appears.
6. Config applies to the group.

### One Tile Missing

1. Group validation fails.
2. KWin exposes available outputs as normal physical outputs.
3. No aggregate logical output is created.
4. Saved group config is not applied as if the group were complete.

### Group Becomes Incomplete After Running

1. Rebuild groups.
2. Destroy the group.
3. Withdraw grouped output-management device if needed.
4. Expose remaining physical outputs normally.
5. Move windows using the normal output removal path.

This fallback is ugly but debuggable and safer than a half-functional group.

## Implementation Phases

### Phase 0: Prepare A Clean Branch

Start from current KWin master. This is already done locally:

```text
branch: work/tiled-output-group
base:   origin/master at f26ad19268
```

Use the prototype branch only as a reference. Cherry-pick only if the resulting diff still matches the new architecture; otherwise reimplement:

- passive TILE parsing in DRM
- passive TILE metadata in virtual backend
- the viewport-relative render-path changes (`mapGlobalLogicalToOutputDeviceCoordinates`/`mapItemToOutputDeviceCoordinates` anchoring on the member viewport, the `createProjectionMatrix` sub-region generalization, and the `itemrenderer_opengl.cpp` scissor) — these are independent of the group ownership model and are the highest-value reuse from the prototype; land them in Phase 4
- the pixel-sampling compositor test approach
- basic tests if they are easy to adapt
- small parser helpers if they are already clean and isolated

Do not carry over:

- `Workspace::primaryTileGroupOutput()`
- `m_tileGroupOutputs`
- compositor unrotated tile viewport logic
- destructive DRM mode filtering
- output-configuration tile adjustments

Acceptance criteria:

- branch contains no commits from `work/tiled-displays`
- branch builds before tiled-output work begins
- first implementation commit is easy to review against `origin/master`

### Phase 1: Passive Groups

Add:

- `KWIN_TILED_OUTPUTS` logging category
- `TiledOutputGroup`
- validation
- group rebuild in `Workspace`
- lookup helpers
- `debugDescription()`
- `debugJson()`
- `Workspace::tiledOutputGroupsDebugJson()`
- candidate/validation trace logging
- virtual backend test coverage

No behavior change yet except debug visibility.

Acceptance criteria:

- valid 2x1 group is detected
- incomplete group is rejected
- duplicate tile location is rejected
- same group id on different scopes does not group
- debug logs explain every accept/reject decision
- debug JSON includes group key, primary output, members, tile rects, and current modes

### Phase 2: One Logical Output

Change `Workspace::updateOutputs()` so valid tiled groups produce one `LogicalOutput`.

Add:

- `LogicalOutput::tiledOutputGroup()`
- `LogicalOutput::backendOutputs()`
- group aggregate mode size
- group description
- `Workspace::findOutput()` maps every group member to the same logical output
- trace logging for logical output creation and member-to-logical-output mapping

Acceptance criteria:

- `wl_output` exposes one output for a valid group
- secondary tile does not create a second logical output
- windows see one output geometry of full aggregate size
- incomplete groups still expose physical outputs normally
- logs show which logical output represents each physical tile

### Phase 3: Config Expansion

Add:

- group config normalization
- group config expansion
- secondary tile config suppression
- DPMS propagation
- transform propagation
- scale propagation
- position normalization
- config expansion trace logging

Acceptance criteria:

- changing scale on the grouped output changes every member
- rotating the grouped output changes every member
- disabling the grouped output disables every member
- enabling the grouped output enables every member
- old secondary-tile config cannot override the group
- logs show group-level input and per-member output changesets

See *Visual Artifacts And Mitigations* point 5: write the same scale to every member in one `OutputConfiguration` so scale is applied atomically across tiles, never per tile.

### Phase 4: Compositor Viewports

Replace ad hoc viewport math with `TiledOutputGroup::logicalViewport()`.

Add:

- `TiledOutputGroup::viewportDebug()`
- viewport-relative device-coordinate mapping in `mapGlobalLogicalToOutputDeviceCoordinates()` / `mapItemToOutputDeviceCoordinates()` (anchor on the member viewport top-left, not logical-output geometry) — reimplemented from the prototype
- sub-region viewport support in `createProjectionMatrix()` (`renderviewport.cpp`) and the `itemrenderer_opengl.cpp` scissor — reimplemented from the prototype
- transform-aware tests using pixel sampling (see *Compositor Tests*)
- viewport trace logging

See *Viewport-Relative Render Path* under Compositor Integration: the viewport rect alone does not render correctly until the device-coordinate mapping, projection matrix, and scissor are all made viewport-relative.

Acceptance criteria:

- normal orientation renders left and right halves correctly
- 90 degree rotation maps the two horizontal tiles into vertical source regions
- 180 degree rotation swaps tile source order correctly
- 270 degree rotation maps correctly
- flipped transforms do not overlap or leave gaps
- logs show native rect, transformed rect, and final logical viewport per member

See *Visual Artifacts And Mitigations* points 1–3 (render all members in one pass driven by a single lead render loop, set the cursor plane on every member the cursor crosses, and intersect damage with each member's viewport) and point 5 (assert member scale equals group scale in the viewport helper).

### Phase 5: Output-Management Hiding

Make KScreen/output-management expose one device for the group.

Add:

- output device registry sync
- group-aware output-device geometry
- group-aware output-device modes, at least current aggregate mode
- secondary member withdrawal
- validation for stale secondary references
- output-management offer/withdraw trace logging

Acceptance criteria:

- KScreen sees one display
- KScreen cannot move/rotate/disable the right tile independently
- disabling the grouped display from KScreen disables all members
- no secondary tile UUID is exposed as a configurable target
- logs show which physical output devices are offered, hidden, withdrawn, or normalized

### Phase 6: Aggregate Modes

Add:

- `TiledOutputMode`
- tiled aggregate mode generation
- current aggregate mode detection
- preferred aggregate mode selection
- per-member mode assignment in config expansion

Acceptance criteria:

- native tiled mode appears as one aggregate mode
- selecting aggregate mode selects matching per-tile modes
- connector modes are not deleted
- preferred mode is stable

See *Visual Artifacts And Mitigations* point 4: match member modes by exact size, refresh, flags, and VRR capability, and publish an aggregate mode only if every member matches exactly.

### Phase 7: Untiled Fallback Modes

Add:

- untiled aggregate mode generation from primary tile
- expansion where only primary member is active
- secondary members disabled for untiled mode

Acceptance criteria:

- lower-resolution single-link mode appears if hardware exposes it
- selecting it disables inactive tile members
- switching back to native tiled mode re-enables all members

### Phase 8: Mirroring

Add group-aware mirroring.

Acceptance criteria:

- normal output can mirror tiled group
- tiled group can mirror normal output
- mirror scale uses aggregate group size
- member render offsets are transform-aware
- group cannot mirror itself through a member UUID

## Test Plan

Recommended new test file:

```text
autotests/integration/tiled_output_test.cpp
```

It can also live in `outputchanges_test.cpp` initially, but a dedicated test file will be easier to read.

### Group Validation Tests

- valid 2x1 group creates group
- valid 1x2 group creates group
- missing tile rejects group
- duplicate tile position rejects group
- out-of-range tile position rejects group
- mixed tile size rejects group
- mixed grid size rejects group
- same group id on different scopes does not group

### Logical Output Tests

- valid 2x1 group produces one logical output
- logical output has aggregate mode size
- `Workspace::findOutput(left)` equals `Workspace::findOutput(right)`
- incomplete group produces separate logical outputs

### Config Tests

- disabling group disables all members
- enabling group enables all members
- transform propagates to all members
- scale propagates to all members
- DPMS off propagates to all members
- stale secondary config is ignored

### Compositor Tests

Prefer the prototype's pixel-sampling methodology over asserting only rectangle coordinates: render a known window (for example a white rect on black) onto the grouped output, read back each member's framebuffer, and assert the expected pixels at tile corners and across the seam. This catches the render-path bugs (wrong device-coordinate anchor, projection, scissor) that a pure geometry assertion would miss, and it is how the prototype's draft rotation test is written.

For a 2x1 group with tile size `2560x2880`:

- normal:
  - left viewport `0,0 2560x2880`
  - right viewport `2560,0 2560x2880`
  - a window straddling the seam appears continuous: sampled pixels match on both sides of the boundary, no gap or doubled column
- rotate 90:
  - tiles become top/bottom regions in `2880x5120`
- rotate 180:
  - left/right source order reverses
- rotate 270:
  - tiles become bottom/top regions
- flip variants:
  - no overlap
  - no gap
  - total covered region equals transformed aggregate output

For each transform, sample tile-corner pixels (as the prototype does) so the test fails on swapped or off-by-origin content, not just on mismatched rectangles.

### Output-Management Tests

- registry offers one output device for group
- registry does not offer secondary member
- KScreen-style config on primary expands to all members
- stale config for secondary is rejected or ignored
- all-disabled check counts group as one visible output

### Mode Tests

When aggregate modes are implemented:

- aggregate tiled mode exists
- aggregate current mode matches assigned member modes
- selecting aggregate tiled mode queues correct member modes
- untiled mode activates only primary member
- connector modes remain intact after TILE metadata changes

## Review Strategy

Split into several merge requests.

Recommended sequence:

1. Fresh branch sanity and passive TILE metadata.
2. Logging category, debug JSON, and passive group validation with tests.
3. Workspace logical output grouping.
4. Config expansion and secondary config suppression.
5. Compositor viewport helper and rotation tests.
6. Output-management hiding.
7. Aggregate tiled modes.
8. Untiled fallback modes.
9. Mirroring polish.

Each MR should leave KWin usable without requiring the full stack to be merged at once.

## Tracing And Debugging

Tracing is part of the implementation, not cleanup. The first phase should add enough instrumentation to answer these questions from logs:

- Which physical outputs reported TILE metadata?
- Which outputs were considered for each group?
- Why was a candidate group accepted or rejected?
- Which backend output became the primary/representative output?
- Which logical output was created for the group?
- What backend changes were generated from a group-level config?
- What viewport did each physical tile receive?
- What did output-management expose or hide?

### Logging Category

Add a dedicated logging category:

```cpp
Q_DECLARE_LOGGING_CATEGORY(KWIN_TILED_OUTPUTS)
```

Recommended location:

```text
src/utils/common.h
src/utils/common.cpp
```

Definition:

```cpp
Q_LOGGING_CATEGORY(KWIN_TILED_OUTPUTS, "kwin_tiled_outputs", QtWarningMsg)
```

This keeps tiled output traces separate from the already noisy `kwin_core` and `kwin_wayland_drm` categories.

Enable traces at runtime with:

```bash
QT_LOGGING_RULES="kwin_tiled_outputs.debug=true;kwin_wayland_drm.debug=true" kwin_wayland
```

For focused compositor viewport debugging:

```bash
QT_LOGGING_RULES="kwin_tiled_outputs.debug=true" kwin_wayland
```

Do not use unconditional `qWarning()` for normal tracing. Use:

```cpp
qCDebug(KWIN_TILED_OUTPUTS) << ...;
qCWarning(KWIN_TILED_OUTPUTS) << ...;
```

Use warnings only for unusual states that should be visible even without debug logging, for example a TILE blob that cannot be parsed or a group that looks complete but cannot be represented.

### Debug Formatting

Add stream operators or helper methods so logs stay readable:

```cpp
QDebug operator<<(QDebug debug, const TiledOutputGroup::Key &key);
QDebug operator<<(QDebug debug, const TiledOutputGroup::ValidationError &error);
QDebug operator<<(QDebug debug, const TiledOutputGroup::Member &member);
QDebug operator<<(QDebug debug, const TiledOutputGroup *group);
```

`TiledOutputGroup` should also provide:

```cpp
QString debugDescription() const;
QJsonObject debugJson() const;
```

`debugDescription()` is for compact logs. `debugJson()` is for dumps that can be attached to bug reports or compared in tests.

Example compact output:

```text
TiledOutputGroup(scope=1, group=7, grid=2x1, tile=2560x2880, native=5120x2880, primary=eDP-1, members=[eDP-1@(0,0), eDP-2@(1,0)])
```

Example JSON shape:

```json
{
  "scopeId": 1,
  "groupId": 7,
  "grid": [2, 1],
  "tileSize": [2560, 2880],
  "nativeSize": [5120, 2880],
  "primary": "eDP-1",
  "members": [
    {
      "name": "eDP-1",
      "uuid": "...",
      "tileLocation": [0, 0],
      "nativeRect": [0, 0, 2560, 2880],
      "enabled": true,
      "mode": "2560x2880@60000"
    },
    {
      "name": "eDP-2",
      "uuid": "...",
      "tileLocation": [1, 0],
      "nativeRect": [2560, 0, 2560, 2880],
      "enabled": true,
      "mode": "2560x2880@60000"
    }
  ]
}
```

### Workspace Dump Method

Add a debug-only or always-available dump helper:

```cpp
QJsonArray Workspace::tiledOutputGroupsDebugJson() const;
void Workspace::dumpTiledOutputGroups() const;
```

`dumpTiledOutputGroups()` should log:

- all physical outputs with TILE metadata
- all valid groups
- all rejected candidate groups with reason
- the logical output each group maps to, if any

This is very useful when a hotplug sequence creates surprising state.

### Validation Trace

During group creation, trace every candidate group:

```text
candidate key=(scope=1, group=7), outputs=[eDP-1,eDP-2]
validate eDP-1 tile=(0,0) grid=2x1 tileSize=2560x2880
validate eDP-2 tile=(1,0) grid=2x1 tileSize=2560x2880
accepted group key=(scope=1, group=7) primary=eDP-1 native=5120x2880
```

For rejection:

```text
rejected group key=(scope=1, group=7): IncompleteGrid, expected=2, found=1, occupied=[(0,0)]
```

Validation errors should include enough structured context to avoid rerunning with extra print statements.

### Config Expansion Trace

Every call to `TiledOutputGroup::expandConfiguration()` should be traceable.

Log one line for the group-level input:

```text
expand group key=(scope=1, group=7): enabled=true pos=0,0 scale=2 transform=Rotate90 mode=5120x2880@60000 replicationSource=<none>
```

Then one line per member:

```text
member eDP-1: enabled=true pos=0,0 scale=2 transform=Rotate90 mode=2560x2880@60000 dpms=On deviceOffset=0,0
member eDP-2: enabled=true pos=0,0 scale=2 transform=Rotate90 mode=2560x2880@60000 dpms=On deviceOffset=0,0
```

This makes it clear whether a bug is caused before backend apply or inside the backend.

### Compositor Viewport Trace

When `TiledOutputGroup::logicalViewport()` is used, log:

```text
viewport group=(scope=1, group=7) output=eDP-2 transform=Rotate90 nativeRect=2560,0 2560x2880 transformedRect=0,0 2880x2560 logicalViewport=0,0 1440x1280 scale=2
```

This is the trace that tells us whether rotation math works before staring at pixels.

Add a small helper:

```cpp
struct TiledOutputViewportDebug
{
    BackendOutput *output;
    Rect nativeRect;
    Rect transformedRect;
    RectF logicalViewport;
    OutputTransform transform;
    double scale;
};

TiledOutputViewportDebug TiledOutputGroup::viewportDebug(BackendOutput *output,
                                                         const LogicalOutput *logicalOutput) const;
```

`logicalViewport()` can call the same internal calculation and return only the final rectangle.

### Output-Management Trace

When syncing output-management devices, log:

```text
offer output device eDP-1 as tiled group primary key=(scope=1, group=7)
hide output device eDP-2 because it is non-primary member of key=(scope=1, group=7)
withdraw output device eDP-2 after group became valid
```

When rejecting stale or invalid config:

```text
reject output-management config: eDP-2 is a secondary member of key=(scope=1, group=7)
normalize replication source eDP-2 uuid -> eDP-1 uuid for key=(scope=1, group=7)
```

### DRM TILE Parse Trace

The DRM backend should log raw and parsed TILE metadata when debug logging is enabled:

```text
connector eDP-1 TILE raw="7:0:2:1:0:0:2560:2880" parsed scope=1 group=7 grid=2x1 loc=0,0 tile=2560x2880
connector eDP-2 TILE raw="7:0:2:1:1:0:2560:2880" parsed scope=1 group=7 grid=2x1 loc=1,0 tile=2560x2880
```

Parse failures should be warnings:

```text
connector eDP-1 TILE parse failed: raw="..."
```

### Runtime Debug Commands

KWin already has a debug console, but a first implementation does not need UI work.

Prefer one of these simple mechanisms:

- a DBus method on an existing debug object, if there is an obvious place
- a temporary-but-reviewable debug helper called from existing debug console plumbing
- a test-only helper for integration tests

Useful methods:

```text
dumpTiledOutputGroups()
dumpOutputConfigurationExpansion()
dumpTiledOutputViewports()
```

Do not block core implementation on a public UI. The important part is that developers can get structured state without adding new ad hoc print statements.

### Test Tracing

Integration tests should print group debug JSON on failure.

For example, when a viewport assertion fails:

```cpp
QVERIFY2(actual == expected,
         qPrintable(workspace()->tiledOutputGroupsDebugJson()));
```

This turns CI failures into actionable reports.

### Debugging Success Criteria

Before moving beyond Phase 1, we should be able to answer all of these from logs:

- Did KWin parse TILE on every expected connector?
- Did the group key include the GPU/backend namespace?
- Did validation accept or reject the group, and why?
- Which backend output is the group primary?
- Which logical output represents the group?
- Which physical outputs are hidden from output-management?
- What per-member configuration is generated?
- What viewport is assigned to each tile?

## Success Criteria

The implementation is successful when:

- a complete tiled panel is exposed as one logical display
- secondary tiles are not independently configurable
- all tile members receive coherent backend configuration
- rotation works without gaps, overlap, or swapped content
- connector modes are not destructively filtered
- invalid tile groups fall back safely
- same TILE group IDs on different GPUs do not collide
- tests cover the core grouping/config/compositor behavior

## Main Risk

The largest architectural risk is output-management, because it is still `BackendOutput` based while the desired user-visible object is group-shaped.

The plan handles this by keeping the primary backend output as a compatibility anchor and making the registry expose only that output with group-aware presentation data.

That is not the final perfect abstraction, but it is a practical bridge that keeps the first implementation reviewable.

## Recommendation

Implement Option A in phases, starting with passive group validation and one-logical-output behavior.

Do not start with aggregate modes or mirroring. Those are important, but they are easier and safer once the core group abstraction owns validation, member lookup, configuration expansion, and compositor viewports.

The guiding rule for every patch should be:

```text
Tile-specific policy belongs in TiledOutputGroup or a direct helper of it.
Other systems should ask the group what to do, not rediscover tile rules.
```
