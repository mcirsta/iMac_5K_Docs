# Linux 5K Instrumentation Plan

## Goal

Determine which of these is true on the iMac19,1 5K panel path:

1. Linux never receives the firmware-built complex internal display state.
2. Linux receives enough of the same eDP state as Windows, but does not complete the later runtime bring-up that Windows completes.

This plan is intentionally Linux-first and static-RE-informed. It assumes:

- we have live Linux access on the target iMac
- we do not currently have live Windows tracing
- we do have the Windows and EFI reverse-engineering results preserved in `RE_5K_iMac19_1.md`

## Current Best Understanding

- Windows `object id 0x14 -> signal/type 128` is not a private Apple-only class.
- In Linux AMD DC, that normalizes to the ordinary eDP path:
  - `CONNECTOR_ID_EDP = 20 / 0x14`
  - `SIGNAL_TYPE_EDP = 128`
- Therefore Linux likely does get the same high-level connector classification as Windows.
- The remaining question is whether Linux is missing:
  - firmware-saved complex-display state on that eDP path
  - or the Windows runtime completion step on top of that eDP path

## Why This Plan

The latest RE no longer supports the older idea that Linux fails because it never even identifies the panel as the same class of connector.

The more likely split is now:

- firmware/GOP prepares extra internal-display state
- Windows `amdkmdag.sys` finishes the process with runtime link/control steps
- Linux either never sees the extra state, or ignores it and continues as ordinary single-path eDP

So the next highest-value move is instrumentation inside the Linux AMD display stack, not more broad firmware searching.

## Phase 1: Add Focused Linux Logging

Add temporary instrumentation with a single grep prefix:

- `IMAC5K:`

This keeps the logs easy to collect and compare.

### File 1

- `AMD_Linux/display/dc/core/dc.c`
- function:
  - `create_links()`

### Log here

- BIOS connector index
- connector object id
- connector enum id
- resulting `dc->link_count` / link index
- whether a link object was created or skipped

### Why

This confirms exactly what Linux thinks the VBIOS object table contains on this machine.

## File 2

- `AMD_Linux/display/dc/link/link_factory.c`
- function:
  - `construct_phy()`

### Log here

- `link->link_id.id`
- `link->link_id.enum_id`
- `link->connector_signal`
- `link->is_internal_display`
- source encoder object
- transmitter
- HPD source
- DDC line

### Why

This is the Linux equivalent of the Windows connector/display-entry construction pass. It will tell us whether Linux already gets:

- internal display classification
- eDP signal classification
- the expected physical path metadata

## File 3

- `AMD_Linux/display/dc/link/link_detection.c`
- eDP detect path around:
  - `case SIGNAL_TYPE_EDP`
  - `detect_edp_sink_caps()`
  - `read_current_link_settings_on_detect()`

### Log here

- DPCD revision
- lane count
- max link rate / current link rate
- `link_rate_set`
- `use_link_rate_set`
- supported link rates count and values
- MST capability bit
- sink signal chosen
- whether sink detect succeeds

### Why

This is where Linux should reveal whether it is seeing a plain single-path eDP sink or something richer that still gets flattened into normal eDP.

## File 4

- `AMD_Linux/display/dc/link/protocols/link_dp_capability.c`
- functions around:
  - eDP supported-link-rate parsing
  - rate selection / fallback

### Log here

- parsed `edp_supported_link_rates_count`
- all parsed supported-link-rate values
- chosen initial link setting
- fallback transitions
- whether the code uses:
  - `link_rate_set`
  - direct link-rate values

### Why

The current Linux AUX data showed all-zero supported link rates on plain Linux boot. This file tells us whether Linux is simply reflecting that, or transforming it in a way that hides a better preboot state.

## Optional File 5

- `AMD_Linux/display/amdgpu_dm/amdgpu_dm_debugfs.c`

### Possible addition

Expose one compact connector dump for the internal eDP link containing:

- connector id
- signal
- internal-display flag
- current link settings
- supported-link-rates info
- MST bit
- any tile-related properties visible to DRM

### Why

This is helpful if we want repeatable data collection without rebuilding more printk-heavy instrumentation later.

## Phase 2: Boot Patched Kernel On Plain Linux

Boot the instrumented kernel on the iMac with the normal plain Linux boot path.

Collect:

- `dmesg | grep IMAC5K`
- full `dmesg`
- existing AUX/DPCD probe output
- `drm_info`
- `modetest -M amdgpu`
- `/sys/class/drm/*`
- internal connector debugfs files such as:
  - `ilr_setting`
  - `internal_display`
  - `odm_combine_segments`

## Phase 3: Decide Which Branch We Are In

### Branch A: Missing Preboot State

Indicators:

- Linux sees ordinary internal eDP only
- no unusual supported-link-rate or connector-state data
- no sign of second-path / grouped internal state
- no tile property
- no richer current link settings than the earlier plain Linux probe

### Conclusion

The Apple/AMD firmware-built complex-display state is missing or not surviving the plain boot path.

### Next move

- test a minimal OpenCore/OCLP-style boot path and rerun the same Linux instrumentation
- compare whether the eDP state changes before the kernel takes over

## Branch B: Linux Gets Enough Preboot State But Does Not Finish It

Indicators:

- Linux sees meaningful internal eDP current-link data
- possibly nontrivial supported-link-rate data or stronger current settings
- internal-display flags line up cleanly
- but DRM still exposes only 4K and no final 5K/tile result

### Conclusion

Linux is likely dropping or failing to complete the same eDP path that Windows completes later.

### Next move

- instrument or patch the runtime Linux bring-up path
- compare against the Windows-side `1265` and post-training reassertion behavior

## Branch C: Mixed Case

Indicators:

- some state is richer than the current plain probe suggested
- but one key field is missing, such as:
  - no supported-link-rates table
  - no second-stage state after training
  - no tile/secondary-path publication

### Conclusion

The system may need both:

- a better preboot path
- and a Linux runtime completion patch

## Concrete Implementation Order

1. Add `IMAC5K:` logging in:
   - `dc/core/dc.c`
   - `dc/link/link_factory.c`
   - `dc/link/link_detection.c`
   - `dc/link/protocols/link_dp_capability.c`
2. Build and boot the patched kernel on plain Linux.
3. Collect the logs and compare them with:
   - current plain-Linux AUX/DPCD probe
   - Windows RE findings in `RE_5K_iMac19_1.md`
4. Decide:
   - preboot problem
   - runtime completion problem
   - or both
5. Only then choose between:
   - minimal EFI shim / OpenCore-style preboot experiment
   - Linux runtime quirk/patch work

## What We Should Not Do Yet

- do not start with modeline injection
- do not hard-force MST globally for eDP
- do not assume `MSLD` alone explains the failure
- do not assume opcode `1265` alone is sufficient to unlock 5K

## Success Criteria For This Phase

This phase is successful if it gives a confident answer to one of these:

- Linux never gets the richer preboot state.
- Linux gets the same basic eDP path as Windows and then fails later.
- Linux gets some, but not all, of the required preboot/runtime state.

That answer is more important than enabling 5K immediately, because it determines whether the next engineering step belongs in:

- EFI / preboot
- Linux AMDGPU/DC runtime
- or both
